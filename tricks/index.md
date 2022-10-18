# 奇技淫巧

收录遇到的各种奇技淫巧
- 位运算转换大小写
1. 转小写
```text
('a' | ' ') = 'a'
('A' | ' ') = 'a'
```
2. 转大写
```text
('b' & '_') = 'B'
('B' & '_') = 'B'
```
3. 大小写互换
```text
('c' ^ ' ') = 'C'
('C' ^ ' ') = 'C'
```
原理:
```text
' ' = 20 = 0b00100000
'_' = 95 = 0b01011111
'A' = 65 = 0b01000001
'a' = 97 = 0b01100001
'A' - 'a' = 20 = ' ' = 0b00100000
```
可以看到二进制ASCII码A与a差的就是`100000`, 其代表的是' ', 所以进行一个或运算可以将大写字母第5位的0变成1而不影响别的位, 小写字母不变. 
同理, 与'A'进行与运算可以将小写字母的第6-8位从011变成大写字母的010, 而由于'_'的0-5位全为1, 进行与运算不会影响这几位. 
又同理, 将字母与' '进行异或会将第5位的1变成0, 0变成1, 即可实现大小写互换. 
总之, 小写字母与大写字母相差32 = 2^5,只要操控第5位即可转换大小写. 大写字母与小写字母中间多的六个字符想必不是巧合, 而是设计时的有意为之. 