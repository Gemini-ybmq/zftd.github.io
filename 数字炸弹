import random
import time

bomb = random.randint(1, 99)
print(bomb)
start = 0
end = 99
while 1 == 1:

    people = int(input('请输入{}到{}之间的数:'.format(start, end)))
    if people > bomb:
        print('大了')
        end = people
    elif people < bomb:
        print('小了')
        start = people
    else:
        print('BOOM!!!')
        break
    print('等待电脑了输入{}到{}之间的数:'.format(start, end))
    time.sleep(1)
    com = random.randint(start + 1, end - 1)
    print('电脑输入：{}'.format(com))
    if com > bomb:
        print('大了')
        end = com
    elif com < bomb:
        print('小了')
        start = com
    else:
        print('BOOM!!!')
        break
