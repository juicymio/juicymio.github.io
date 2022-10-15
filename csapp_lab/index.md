# Csapp_lab

## 环境准备
- WSL2(Ubuntu22.04) + VSCode 
```
apt-get update
sudo apt-get install build-essential
sudo apt-get install gcc-multilib
sudo apt-get install gdb
```
## Data Lab
### 食用方法
- 阅读README获得完整信息
- 按照bits.c文件的注释修改bits.c
- 使用`./dlc bits.c` 检查代码是否符合标准, 若符合则无输出.  `./dlc -e bits.c`查看用了多少操作符
- 每次修改bits.c后 执行`make clean && make btest`编译测试工具
- 执行`./btest`检查正确性,`./btest -f foo`可以单独检查某个函数的正确性
- 运行`./driver.pl`打分
### 题目
#### bitXor
只使用~和&实现^
```c
a^b
= (a|b)&(~a|~b)
= ~(~a&~b)&(a&b)
= (a&~b)|(~a&b)
```
```c
//1

/*

 * bitXor - x^y using only ~ and &

 *   Example: bitXor(4, 5) = 1

 *   Legal ops: ~ &

 *   Max ops: 14

 *   Rating: 1

 */

int bitXor(int x, int y) {

  return ~(~(x & ~ y) & ~(~x & y));

}
```
#### tmin
```c
/*

 * tmin - return minimum two's complement integer

 *   Legal ops: ! ~ & ^ | + << >>

 *   Max ops: 4

 *   Rating: 1

 */

int tmin(void) {

  return 1 << 31;

}
```
#### isTmax
最大值x应该是第一位为0, 其它位都为1, 则`x+1 = ~x`. 但是由于所有位都为1的-1+1之后进位的0会溢出消失, 所以`-1 + 1 = ~(-1)`需要特判, 只需特判~x是否为0, 即`(!!~x)`
```c
//2

/*

 * isTmax - returns 1 if x is the maximum, two's complement number,

 *     and 0 otherwise

 *   Legal ops: ! ~ & ^ | +

 *   Max ops: 10

 *   Rating: 1

 */

int isTmax(int x) {
  return !((x+1)^(~x))&(!!~x); 
}
```
