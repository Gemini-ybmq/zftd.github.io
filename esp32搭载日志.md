#节点代码

#include <LoRaNow.h>

//vspi for lora radio module
#define MISO 19
#define MOSI 23
#define SCK 18
#define SS 5

#define DIO0 4

//for 18b20
#include <OneWire.h>
#include <DallasTemperature.h>

// GPIO where the DS18B20 is connected to
const int oneWireBus= 25 ; 

// Setup a oneWire instance to communicate with any OneWire devices
OneWire oneWire(oneWireBus);

// Pass our oneWire reference to Dallas Temperature sensor 
DallasTemperature sensors(&oneWire);
double tmp;

void setup() {
  Serial.begin(115200);
  Serial.println("LoRaNow便携节点");
  pinMode(oneWireBus,INPUT);
  sensors.begin();
   LoRaNow.setFrequencyCN(); // Select the frequency 486.5 MHz - Used in China
  // LoRaNow.setFrequencyEU(); // Select the frequency 868.3 MHz - Used in Europe
  // LoRaNow.setFrequencyUS(); // Select the frequency 904.1 MHz - Used in USA, Canada and South America
  // LoRaNow.setFrequencyAU(); // Select the frequency 917.0 MHz - Used in Australia, Brazil and Chile

  // LoRaNow.setFrequency(frequency);
  // LoRaNow.setSpreadingFactor(sf);
  // LoRaNow.setPins(ss, dio0);

   LoRaNow.setPinsSPI(SCK, MISO, MOSI, SS, DIO0); // Only works with ESP32

  if (!LoRaNow.begin()) {
    Serial.println("LoRa无法加载，请检查你的连接。");
    while (true);
  }

  LoRaNow.onMessage(onMessage);
  LoRaNow.onSleep(onSleep);
  LoRaNow.showStatus(Serial);
}

void loop() {
  sensors.requestTemperatures(); 
  tmp = sensors.getTempCByIndex(0);
  LoRaNow.loop();
}


void onMessage(uint8_t *buffer, size_t size)
{
  Serial.print("收到信息: ");
  Serial.write(buffer, size);
  Serial.println();
  Serial.println();
}

void onSleep()
{
  Serial.println("sleep...");
  Serial.print("实时温度：");
  Serial.print(tmp);
  Serial.println("℃");
  delay(5000); // "kind of a sleep"
  Serial.println("发送信息：");
  LoRaNow.print("信息编号");
  LoRaNow.print(millis());
  LoRaNow.print("：");
  LoRaNow.print("实时温度：");
  //Serial.println("LoRaNow Message sended");
  LoRaNow.print(tmp);
  LoRaNow.print("℃");
  LoRaNow.send();
}

#网关代码

#include <LoRaNow.h>
#include <WiFi.h>
#include <WebServer.h>
#include <StreamString.h>
#include <PubSubClient.h>

//vspi
//#define MISO 19
//#define MOSI 23
//#define SCK 18
//#define SS 5

//hspi,only for ESP-32-Lora-Shield-Ra02
#define SCK 14
#define MISO 12
#define MOSI 13
#define SS 15

#define DIO0 4

const char *ssid = "tofly";
const char *password = "111222333";
const char *mqtt_server = "broker.hivemq.com";

WebServer server(80);
WiFiClient espClient;
PubSubClient mqttclient(espClient);

const char *script = "<script>function loop() {var resp = GET_NOW('loranow'); var area = document.getElementById('area').value; document.getElementById('area').value = area + resp; setTimeout('loop()', 1000);} function GET_NOW(get) { var xmlhttp; if (window.XMLHttpRequest) xmlhttp = new XMLHttpRequest(); else xmlhttp = new ActiveXObject('Microsoft.XMLHTTP'); xmlhttp.open('GET', get, false); xmlhttp.send(); return xmlhttp.responseText; }</script>";

unsigned long lastMsg = 0;
#define MSG_BUFFER_SIZE  (60)
char msg[MSG_BUFFER_SIZE];

int value = 0;

void handleRoot()
{
  String str = "";
  str += "<html>";
  str += "<head>";
  str += "<title>ESP32 - LoRaNow</title>";
  str += "<meta name='viewport' content='width=device-width, initial-scale=1'>";
  str += script;
  str += "</head>";
  str += "<body onload='loop()'>";
  str += "<center>";
  str += "<textarea id='area' style='width:800px; height:400px;'></textarea>";
  str += "</center>";
  str += "</body>";
  str += "</html>";
  server.send(200, "text/html", str);
}

static StreamString string;

void handleLoRaNow()
{
  server.send(200, "text/plain", string);
  while (string.available()) // clear
  {
    string.read();
  }
}

void callback(char* topic, byte* payload, unsigned int length) {
  Serial.print("Message arrived [");
  Serial.print(topic);
  Serial.print("] ");
  for (int i = 0; i < length; i++) {
    Serial.print((char)payload[i]);
  }
  Serial.println();

  // Switch on the LED if an 1 was received as first character
  if ((char)payload[0] == '1') {
    //digitalWrite(BUILTIN_LED, LOW);   // Turn the LED on (Note that LOW is the voltage level
    // but actually the LED is on; this is because
    // it is active low on the ESP-01)
  } else {
    //digitalWrite(BUILTIN_LED, HIGH);  // Turn the LED off by making the voltage HIGH
  }

}

void setup(void)
{

  Serial.begin(115200);

  WiFi.mode(WIFI_STA);
  //if (ssid != "")
    WiFi.begin(ssid, password);
  WiFi.begin();
  Serial.println("");

  // Wait for connection
  while (WiFi.status() != WL_CONNECTED)
  {
    delay(500);
    Serial.print(".");
  }

  Serial.println("");
  Serial.print("Connected to ");
  Serial.println(ssid);
  Serial.print("IP address: ");
  Serial.println(WiFi.localIP());

  server.on("/", handleRoot);
  server.on("/loranow", handleLoRaNow);
  server.begin();
  Serial.println("HTTP server started");

  LoRaNow.setFrequencyCN(); // Select the frequency 486.5 MHz - Used in China
  // LoRaNow.setFrequencyEU(); // Select the frequency 868.3 MHz - Used in Europe
  // LoRaNow.setFrequencyUS(); // Select the frequency 904.1 MHz - Used in USA, Canada and South America
  // LoRaNow.setFrequencyAU(); // Select the frequency 917.0 MHz - Used in Australia, Brazil and Chile

  // LoRaNow.setFrequency(frequency);
  // LoRaNow.setSpreadingFactor(sf);
  // LoRaNow.setPins(ss, dio0);

  LoRaNow.setPinsSPI(SCK, MISO, MOSI, SS, DIO0); // Only works with ESP32

  if (!LoRaNow.begin())
  {
    Serial.println("LoRa init failed. Check your connections.");
    while (true)
      ;
  }

  LoRaNow.onMessage(onMessage);
  LoRaNow.gateway();
  //mqtt
  mqttclient.setServer(mqtt_server, 1883);
  mqttclient.setCallback(callback);

}

void loop(void)
{
  LoRaNow.loop();
  server.handleClient();
  mqttloop();
}

void onMessage(uint8_t *buffer, size_t size)
{
  unsigned long id = LoRaNow.id();
  byte count = LoRaNow.count();

  Serial.print("Node Id: ");
  Serial.println(id, HEX);
  Serial.print("Count: ");
  Serial.println(count);
  Serial.print("Message: ");
  Serial.write(buffer, size);
  Serial.println();
  Serial.println();
  
   //此处通过mqtt发送信息。
   snprintf (msg, MSG_BUFFER_SIZE, "#%d#%s", count,buffer);
   Serial.print("Publish message: ");
   Serial.println(msg);
   mqttclient.publish("C2TLOutTopic", msg);

  if (string.available() > 512)
  {
    while (string.available())
    {
      string.read();
    }
  }

  string.print("Node Id: ");
  string.println(id, HEX);
  string.print("Count: ");
  string.println(count);
  string.print("Message: ");
  string.write(buffer, size);
  string.println();
  string.println();

  // Send data to the node
  LoRaNow.clear();
  LoRaNow.print("LoRaNow Gateway Message ");
  LoRaNow.print(millis());
  LoRaNow.send();
}

void reconnect() {
  // Loop until we're reconnected
  while (!mqttclient.connected()) {
    Serial.print("Attempting MQTT connection...");
    // Create a random mqttclient ID
    String clientId = "ESP8266Client-";
    clientId += String(random(0xffff), HEX);
    // Attempt to connect
    if (mqttclient.connect(clientId.c_str())) {
      Serial.println("connected");
      // Once connected, publish an announcement...
      mqttclient.publish("C2TLOutTopic", "hello world");
      // ... and resubscribe
      mqttclient.subscribe("C2TLInTopic");
    } else {
      Serial.print("failed, rc=");
      Serial.print(mqttclient.state());
      Serial.println(" try again in 5 seconds");
      // Wait 5 seconds before retrying
      delay(5000);
    }
  }
}

void mqttloop() {

  if (!mqttclient.connected()) {
    reconnect();
  }
  mqttclient.loop();

//  unsigned long now = millis();
//  if (now - lastMsg > 2000) {
//    lastMsg = now;
//    ++value;
//    snprintf (msg, MSG_BUFFER_SIZE, "hello world #%ld", value);
//    Serial.print("Publish message: ");
//    Serial.println(msg);
//    mqttclient.publish("C2TLOutTopic", msg);
//  }
}

#搭建过程中遇到的一些问题

1.esp32通过usb连接到电脑上后，arduino工具栏里的端口仍显示为灰色，无法上传代码至单片机。后经排查，发现缺少esp32的驱动程序。在安装完cp210xVCP驱动插件后，一切正常。

2.在接线时，发现排针焊反，导致引脚不够长。后来选择将母对母杜邦线的一端拆开以解决问题。

3.lora-node教程代码中，由于使用的是不同的温度传感器，温度计算公式也发生变化。在百度以及csdn社区中搜索，最后得到相关程序的具体计算公式。

#示图

![T67C})QRM 7F$3J8B%KM 8D](https://user-images.githubusercontent.com/80244129/114301080-ac1a4580-9af5-11eb-815c-638be8fcc850.jpg)

![BV$8~~GMY10R~TT_~40XD91](https://user-images.githubusercontent.com/80244129/114301090-b9373480-9af5-11eb-9a28-43ae3c531c8e.jpg)

![Z4T217945P Q68GWUFIY3RW](https://user-images.githubusercontent.com/80244129/114301097-c48a6000-9af5-11eb-8bca-0a7cf99a88e7.jpg)


