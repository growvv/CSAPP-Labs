# Data Lab 笔记

这一个 lab 主要涉及了位运算，补码和浮点数等内容。完成 lab 不仅要实现函数的功能，还要求仅用规定的操作符，操作符数目也在限定范围内，并且不能使用循环和条件语句，这一点比较坑，因为这样代码可读性不高，当然难度也大了。所有题目都限定在32位系统中。     

## 1.位操作

1. bitAnd

```C
/* 
 * bitAnd - x&y using only ~ and | 
 *   Example: bitAnd(6, 5) = 4
 *   Legal ops: ~ |
 *   Max ops: 8
 *   Rating: 1
 */
int bitAnd(int x, int y) {
    return ~(~x|~y);
}
```

第一题要求只用非运算 ~ 和或运算 | 实现和 & 运算，可以使用
[德摩根定律](https://zh.wikipedia.org/wiki/%E5%BE%B7%E6%91%A9%E6%A0%B9%E5%AE%9A%E5%BE%8B)       

2. getByte

```C
/* 
 * getByte - Extract byte n from word x
 *   Bytes numbered from 0 (LSB) to 3 (MSB)
 *   Examples: getByte(0x12345678,1) = 0x56
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 6
 *   Rating: 2
 */
int getByte(int x, int n) {
  return (x>>(n<<3))&0xff;
}
```

第二题是从 x 中提取出第 i 个字节（i=0,1，2,3），方法就是将那个字节移位至最低位，然后用屏蔽码 `0xff` 提取就可以了；       

3. logicalShift
```C
/* 
 * logicalShift - shift x to the right by n, using a logical shift
 *   Can assume that 0 <= n <= 31
 *   Examples: logicalShift(0x87654321,4) = 0x08765432
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 20
 *   Rating: 3 
 */
int logicshift(int x, int n)
{
    int temp = ~(1 << 31);
    temp = ((temp >> n) << 1) + 1;                //生成掩码0...01...1(前面为n个0)
    return (x >> n) & temp;
}
```

第三题要求实现逻辑右移，对于有符号的 `int` ，C 语言默认的移位方式是算术右移，就是右移时在高位扩展符号位，这里我们需要扩展的符号位都设置为 0 ,可以构造一个屏蔽码屏蔽 `x>>n` 中的非扩展的位，用 & 实现目的。   
但这里要注意 C 语言对移位位数超出自身长度的行为是未定义的，因此在这里构造屏蔽码时不能使得移位位数超过了32或是小于0，我这段代码为了避免这种情况的发生，将屏蔽码分了最高位和其他位两部分构造，直接使用 `((0x1<<(33+~n))+~0)` 构造的屏蔽码在 n=0 将会无法确定。    
这里 `32+~n` 表示了 31-n ，可以由补码的运算性质 `-x=~x+1` 得到，同时这里我在注释里写了两个我最初写的 bug 。      

4. bitCount

```C
/*
 * bitCount - returns count of number of 1's in word
 *   Examples: bitCount(5) = 2, bitCount(7) = 3
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 40
 *   Rating: 4
 */
int bitCount(int x)
{
    int result;
    int tmpmark1 = 0x55 + (0x55 << 8);                //最大0xff
    int mark1 = tmpmark1 + (tmpmark1 << 16);
    int tmpmark2 = 0x33 + (0x33 <<8);
    int mark2 = tmpmark2 + (tmpmark2 << 16);
    int tmpmark3 = 0x0f + (0x0f << 8);
    int mark3 = tmpmark3 + (tmpmark3 << 16);
    int mark4 = 0xff + (0xff << 16);
    int mark5 = 0xff + (0xff << 8);                  //以上生成5个掩码

    result = (x & mark1) + ((x >> 1) & mark1);
    result = (result & mark2) + ((result >> 2) & mark2);   //这两个由于进位问题，不能先加再与
    result = (result + (result >> 4)) & mark3;              //分治
    result = (result + (result >> 8)) & mark4;
    result = (result + (result >> 16)) & mark5;
    return result;
}
```

这题好难T_T，应该是这个 lab 里最难的了吧；

由于不能用循环，操作数限制在40，刚开始完全没有头绪。

题目意思就是要统计一个32位的 `int` 里的 `1` 的个数，但是只能使用40个操作符，直接扫一遍字的话操作符就大大超过规定数了；    
这里构造了五个常数，分别是 `0x55555555，0x33333333，0x0f0f0f0f，0x00ff00ff，0x0000ffff`，就是分别间隔了1个0,2个0,4个0,8个0和16个0,利用这五个常数就能依次计算出五个值，第一个值每两位的值为 x 的对应的两个位的和（即这两位中 `1` 的数目），第二个值每四位是第一个值对应的四位中两个两位的和（即原 x 中 `1`的数目），依次类推最后一个值就是结果了；    
怎么理解呢，可以看到这里构造的五个常数的间隔可以刚好使得只提取 n 位，移位之后再提取相邻 n 位（n=1,2,4,8,16），并且（考虑最大值可知）这两个 n 位加和后不会超出 n 位，使得 x 中的 `1` 一步步加和成最终的结果，可以举一个例子，若要求 1001 中 `1` 的数目，用`(1001&0101)+((1001>>1)&0101)`,就能将每相邻一位加和成一个两位，成 0101，再用`(0101&0011)+((0101>>2)&0011)`，就将每两位加和了，得到 0010 ,就是最终的结果。      

5. conditional

```C
/*
 * conditional - same as x ? y : z
 *   Example: conditional(2,4,5) = 4
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 16
 *   Rating: 3
 */
int conditional(int x, int y, int z) {
	  int temp=(~(!x))+1;
	  return (temp&z)|(~temp&y);
}
```

当x = 0时，temp = 1...1

当x != 0时，temp = 0...0 

## 2.补码

6. tmin

```C
/* 
 * tmin - return minimum two's complement integer 
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 4
 *   Rating: 1
 */
int tmin(void) {
    return 0x1<<31;/*tmin==~tmax*/
}
```

这题返回补码最小值，注意到 `tmin==~tmax`，补码负数表示部分和正数是不对称的，最小值的绝对值是最大值的绝对值加1。    

7. fitsBits

```C
/* 
 * fitsBits - return 1 if x can be represented as an 
 *  n-bit, two's complement integer.
 *   1 <= n <= 32
 *   Examples: fitsBits(5,3) = 0, fitsBits(-4,3) = 1
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 15
 *   Rating: 2
 */
int fitsBits(int x, int n) {
    int temp = 33+~n;
    return !(((x<<temp)>>temp)^x);
}
```

这题题目意思是判断 x 能否用一个 n 位的补码表示，能的话就返回 1,开始我没看懂题目…举个例子，101 是不能用 3 位补码表示的，因为 3 位最高位是符号位，最大只能表示 011,注意到这里 x 是32位的，不能直接右移的；   
要用 n 位的补码表示，x 只能是两种情况： `00…0|0|(n-1)位` 或是 `11…1|1|(n-1)位` ，这样 32 位的补码才会与 n 位的补码值相同，这里的方法就是将 x 左移（32-n）再右移回去，这样就能得到那两种情况的值，再判断这样操作之后是否与原来的 x 相等，就解决问题了；    
这里由补码性质，`33+~n` 等于 `32-n` 。      

8. divpwr2

```C
/* 
 * divpwr2 - Compute x/(2^n), for 0 <= n <= 30
 *  Round toward zero
 *   Examples: divpwr2(15,1) = 7, divpwr2(-33,4) = -2
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 15
 *   Rating: 2
 */
int divpwr2(int x, int n) {
    int bias = (1<<n)+(~0);
    return (x+((x>>31)&bias))>>n;
}
```

这题计算 x/(2^n) ，注意不能直接右移，直接右移是向下舍入的，题目要求是向零舍入，也就是正数向下舍入，负数向上舍入，这里参照 CS:APP 书上的做法，给负数加上一个偏正的因子 `(0x1<<n)+~0)` ，判断负数直接看符号位。    

9. negate

```C
/* 
 * negate - return -x 
 *   Example: negate(1) = -1.
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 5
 *   Rating: 2
 */
int negate(int x) {
  return ~x+1;
}
```

这题求 -x ，直接利用补码的性质 `-x=~x+1` 就可以了。   

10. isPositive

```C
/* 
 * isPositive - return 1 if x > 0, return 0 otherwise 
 *   Example: isPositive(-1) = 0.
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 8
 *   Rating: 3
 */
int isPositive(int x) {
  return !(!(x))&!((x>>31)&(0x1));
}
```

这里判断是否是正数，直接判断符号位，但是注意要排除 0 的情况！   

11. isLessOrEqual

```C
/* 
 * isLessOrEqual - if x <= y  then return 1, else return 0 
 *   Example: isLessOrEqual(4,5) = 1.
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 24
 *   Rating: 3
 */
int isLessOrEqual(int x, int y)
{
    int signx = x >> 31;
    int signy = y >> 31;
    int signEqual = (!(signx ^ signy) & ((x + (~y)) >> 31));//符号位不同时，做差
    int signDiffer = signx & (!signy);                      //符号位相同，直接比较符号位
    return signEqual | signDiffer;
}
```

```C
/*
    操作数更少的版本
*/
int isLessOrEqual_2(int x, int y)
{
    int not_y = ~y;
    return ((((x + not_y) & (x ^ not_y)) | (x & not_y)) >> 31) & 1;
    //        x-y-1<0 <----------x,y不同号------>x为负，y为正，才为正
}
```

这题比较两个数的大小，要求判断第一个数是否小于等于第二个数，这里考虑做减法然后判断符号，注意要考虑溢出的情况，这里 `((x+~y))` 表示了 `x-y-1` ，若其结果为负，则 x <= y ;    
这里先判断 x 与 y 的符号，如果 x 为负，y 为正直接返回 1 ,如果 x 为正，y 为正，直接返回 0；然后就是全正数和全负数的减法，这样不会溢出。   

12. ilog2

```C
/*
 * ilog2 - return floor(log base 2 of x), where x > 0
 *   Example: ilog2(16) = 4
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 90
 *   Rating: 4
 */
int ilog2(int x) {
    int bit_16, bit_8, bit_4, bit_2, bit_1;
    bit_16 = (!!(x >> 16)) << 4;                //如果x >> 16非零，则至少有16位
    x = x >> bit_16;
    bit_8 = (!!(x >> 8)) << 3;
    x = x >> bit_8;
    bit_4 = (!!(x >> 4)) << 2;
    x = x >> bit_4;
    bit_2 = (!!(x >> 2)) << 1;
    x = x >> bit_2;
    bit_1 = x >> 1;      //还剩两位时，直接判断首位。bit_1 == 1,剩两位；bit_1 == 0,剩一位
    return bit_16 + bit_8 + bit_4 + bit_2 + bit_1;
　　　　　//实际是（bit_1 + 1）-1；由于向下舍入，总位数减一
}
```

这题求 x 以 2 为底的对数，解法有点难想到，注意到 32 位数的对数最大也不会超过 32,可以写成是 `16*a+8*b+4*c+2*d+e` 这里 a，b，c，d，e 都是 0 或 1，然后通过向右移 16 位就可以判断符号就可以得到 a ，右移 16*a+8 位可得到 b，以此类推得到其他位。 

13. howManyBits

```C
/* howManyBits - return the minimum number of bits required to represent x in
 *             two's complement
 *  Examples: howManyBits(12) = 5
 *            howManyBits(298) = 10
 *            howManyBits(-5) = 4
 *            howManyBits(0)  = 1
 *            howManyBits(-1) = 1
 *            howManyBits(0x80000000) = 32
 *  Legal ops: ! ~ & ^ | + << >>
 *  Max ops: 90
 *  Rating: 4
 */
int howManyBits(int x) {
    int bit_16, bit_8, bit_4, bit_2, bit_1, result;
    int k = x >> 31;
    int temp = x ^ k;                      
　　　　　//x为正，temp = x;x为负，temp = ~x
    int isZero = (!!(temp << 31)) >> 31; 
　　　　 //x = 0或x = -1时，temp = 0,isZero = 0...0；否则isZero = 1...1
    bit_16 = (!!(temp >> 16)) << 4;
    temp = temp >> bit_16;
    bit_8 = (!!(temp << 8)) << 3;
    temp = temp >> bit_8;
    bit_4 = (!!(temp << 4)) << 2;
    temp = temp >> bit_4;
    bit_2 = (!!(temp << 2)) << 1;
    temp = temp >> bit_2;
    bit_1 = temp >> 1;
    result = bit_16 + bit_8 + bit_4 + bit_2 + bit_1 + 2;
　　　　　　//真正位数为bit_16 + ... + bit_1 + 1,再加符号位一位
    return (!isZero) | (result & isZero);
}
```

## 3.浮点数

以下三题是关于浮点数的，可以使用任何操作符和分支语句。   

务必先去学习IEEE 754标准

13. float_neg

```C
/* 
 * float_half - Return bit-level equivalent of expression 0.5*f for
 *   floating point argument f.
 *   Both the argument and result are passed as unsigned int's, but
 *   they are to be interpreted as the bit-level representation of
 *   single-precision floating point values.
 *   When argument is NaN, return argument
 *   Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
 *   Max ops: 30
 *   Rating: 4
 */
unsigned float_half(unsigned uf)
{
    unsigned s = uf & 0x80000000;            
    unsigned exp = uf & 0x7f800000;
    int lsb = ((uf & 3) == 3);                //判断frac最后两位是否为11
    if (exp == 0x7f800000)
        return uf;
    else if (exp <= 0x800000)
        return s | (((uf ^ s) + lsb) >> 1);  //uf^s将符号位置零，uf^s = frac + exp末位
    else
        return uf - 0x800000;                 //整体思路就是模拟
}
```     

14. float_i2f

```C
/* 
 * float_f2i - Return bit-level equivalent of expression (int) f
 *   for floating point argument f.
 *   Argument is passed as unsigned int, but
 *   it is to be interpreted as the bit-level representation of a
 *   single-precision floating point value.
 *   Anything out of range (including NaN and infinity) should return
 *   0x80000000u.
 *   Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
 *   Max ops: 30
 *   Rating: 4
*/
int float_f2i(unsigned uf)
{
    int abs;
    int sign = uf >> 31;
    int exp = (uf >> 23) & 0xff;
    int frac = uf & 0x007fffff;
    if (exp < 0x7f)        return 0;
    if (exp > 157)        return 0x80000000;                //Tmax = 2^31 -1

    abs = ((frac >> 23) + 1) << (exp - 127);            //模拟
    if (sign)
        return -abs;
    else
        return abs;
}
```

这题是将整型转化为浮点数的格式，坑点很多，耗时长。。   
整体思路就是依次计算符号位，阶码值和小数字段，符号位可以直接移位提取，阶码值就是除了符号位外最高位的位数减 1 再加上偏差 127，小数字段可以移位（负数可以化为正数操作）获得，但这问题没这么简单，有很多坑点：   
1. 特殊值 0 化为浮点数后是非规格化的，单独考虑；   
2. 特殊值 0x80000000 是 2 的整数倍，小数部分用移位的话因为舍入问题会溢出，单独考虑；   
3. 要仔细考虑移位过程，左移还是右移能得到 23 位的小数部分；   
4. 注意舍入问题，这里需要仔仔细细地考虑清楚，默认使用向偶数舍入，就是舍去部分小于中间值舍弃，大于中间值进位，为中间值如 100 就向偶数舍入：就是看前一位，进位或舍弃总使得前一位为 0；   

15. float_twice

```C
/* 
 * float_twice - Return bit-level equivalent of expression 2*f for
 *   floating point argument f.
 *   Both the argument and result are passed as unsigned int's, but
 *   they are to be interpreted as the bit-level representation of
 *   single-precision floating point values.
 *   When argument is NaN, return argument
 *   Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
 *   Max ops: 30
 *   Rating: 4
 */
unsigned float_twice(unsigned uf)
{
    int result;
    int exp = uf & 0x7f800000;
    int frac = uf & 0x7fffff;
    if (exp == 0x7f800000)
        return uf;
    else if (exp == 0)
        frac = frac << 1;              //frac也可用uf代替，因为此时frac = uf
    else
        exp = exp + 0x800000;
    result = (uf & 0x80000000) | exp | frac;
    return result;
}
```

这题计算浮点数的两倍，无穷大和 NaN 时直接返回，然后分规格化和非规格化两种讨论：    
规格化的情况，阶码值直接加 1 ，但是有个特殊情况就是加一后阶码为 255 时，应返回无穷大；   
非规格化的情况，排除符号位左移一位就可以了，因为这时阶码值为 0 ,两倍就相当于小数字段左移一位，不用担心溢出的情况，溢出时阶码值加 1,小数字段左移一位，相当于整体左移了。

## 小结

作为CSAPP配套实验中的第一个实验，整个 lab 总体难度较大，但相对后面的实验，对于新手还算友好。

我参考着 Google 花了一周的晚上才完成，真是太菜了。。还是要多学习，提高姿势水平。


 