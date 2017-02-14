---
layout: post
title: "popcount problem"
date: 2014-10-25 15:59:37 +0800
comments: true
categories: algorithm 
---
##背景
最近在看google 的[sparsehash](https://code.google.com/p/sparsehash) 实现。hash为了节省内存，使用稀疏矩阵（sparsehash table）实现，因为每次查找都会计算bitmap中1的个数。计算二进制数1的个数，专业说法是 [popcount](http://en.wikipedia.org/wiki/Hamming_weight)” 问题，就成了这个算法的快慢的关键。而popcount本身也多次出现在各种程序员面试题目中（至少我被面试过2次，我也面试过别人很多次）。这里深入学习和总结了一下，如果完全看懂这篇文章，以后面试这类问题，完全可以[SM面试官](http://book.douban.com/review/4713637/) :)
<!--more-->
##题目描述

求一个N位整数x的二进制表示中1的个数，越快越好。


实际上这个问题叫做Hamming weight[1]，或者叫做population count以及pop count。看到Hamming大家基本上可以恍然大悟一下了，这玩意可以用来计算海明距离。另外在信息论、编码学等也有很多应用。
这个问题网上一搜一大把。[这里](http://iafita.wordpress.com/2009/01/30/popcount-problem-%E6%B1%82%E4%BA%8C%E8%BF%9B%E5%88%B6%E6%95%B0%E4%B8%AD1%E7%9A%84%E4%B8%AA%E6%95%B0-%E4%B8%8A/) 和[这里](http://www.cnblogs.com/Martinium/articles/popcount.html)都写的不错，算法本身都很简单，就几行代码，但是要理解起来，要费一点时间。

###naive solution 
顺次移位，数1的个数。
算法复杂度：O(N)。
最最基本的算法（都叫naive了），不多描述

###less-naive solution
用到一个关键点 `x&(x-1)`可以将最右端（rightmost）1变为0。
``` c
int pop_count_lnaive(uint32 x)
{
	int count = 0;
	for (; x; ++count)
		x&=x-1;
	return count;
}
```
时间复杂度：O(N)如果1的个数很少，会提速很多
也没什么好多介绍的。

###shift_and_add
这个算法用到了分治和并行计算来优化，比较巧妙。
``` c
typedef unsigned __int64 uint64;  //assume this gives 64-bits
const uint64 m1  = 0x5555555555555555; //binary: 0101...
const uint64 m2  = 0x3333333333333333; //binary: 00110011..
const uint64 m4  = 0x0f0f0f0f0f0f0f0f; //binary:  4 zeros,  4 ones ...
const uint64 m8  = 0x00ff00ff00ff00ff; //binary:  8 zeros,  8 ones ...
const uint64 m16 = 0x0000ffff0000ffff; //binary: 16 zeros, 16 ones ...
const uint64 m32 = 0x00000000ffffffff; //binary: 32 zeros, 32 ones

int popcount_1(uint64 x) {
	x = (x & m1 ) + ((x >>  1) & m1 );  
	x = (x & m2 ) + ((x >>  2) & m2 ); 
	x = (x & m4 ) + ((x >>  4) & m4 ); 
	x = (x & m8 ) + ((x >>  8) & m8 ); 
	x = (x & m16) + ((x >> 16) & m16); 
	x = (x & m32) + ((x >> 32) & m32); 
	return x;
}
```
时间复杂度：O(logN) 总体需要6步，每一步是1次shift，1次and，一次add。所以一共是6次shift, 12次and, 6次add，共24次算术运算。
算法很简单，将n写成二进制，相邻位相加，并且存放在原地。然后继续这样操作下去。
举个例子

![](http://pic002.cnblogs.com/img/zdd/201006/2010060623161414.jpg)

比如8bit数[a][b][c][d][e][f][g][h]:

第一步就是计算 
```
  [0][b][0][d][0][f][0][h] 
+ [0][a][0][c][0][e][0][g] 
---------------------------
= [AB]  [CD]  [EF]  [GH] 
```
相加结果记为[AB][CD][EF][GH]. 其中[AB]=a+b的值。比如说[AB]就等于图中的2(10) [CD]就是1(01)...

第二步就是计算[AB]+[CD] 和 [EF]＋[GH]的结果

第三步就能计算出答案。因为例子是8bit，如果是32bit，就需要5步。

该算法很重要，必须理解透彻，它是后续所有算法的核心。

###shift_and_add优化(hacker_popcnt)
先放上代码，在慢慢解释
``` c
int popcount_2(uint64 x) {
	x -= (x >> 1) & m1; 
	x = (x & m2) + ((x >> 2) & m2); 
	x = (x + (x >> 4)) & m4; 
	x += x >>  8; 
	x += x >> 16; 
	x += x >> 32;  
	return x & 0x7f;
}
```
也叫hacker_popcnt，是因为该算法来自[hackersdelight](http://www.hackersdelight.org/)。这本书虽然不厚，但是看起来很慢，我用了一个下午也看不了几页，全都是这些烧脑的算法。

和优化前的代码步骤一样，也是6步。每一步的目的和结果都是一样的。因此我们每一步分析具体是怎么优化的。

+ 第一步 `(x & m1 ) + ((x >>  1) & m1 )`⇒ `x-(x >> 1) & m1`，省了一步and。对于2bit数 ab，需要计算1的个数。原始做法是直接计算0a+0b得到，实际上也可以通过ab-a得到，即`f(ab)=ab-a`这个证明很简单，因为二进制数，直接枚举所有情况（也就四种）就能证明。比如11, a=1,b=1,  f(ab) = ab -a = 11 -1 =2. 11有2个1 。

+ 第二步没有任何优化。

+ 第三步 `(x + (x >> 4)) & m4` => `(x & m4 ) + ((x >>  4) & m4 )` 省了一步and。即 先与后加=>先加后与。 之所以先与后加，是为了去掉不必要的干扰，但是在第三步开始实际上是不需要的。举个例子。

	比如8bit数[X][Y]，其中X和Y都是4bit数字。原始算法就是老老实实与，然后计算0Y+0X。我们知道X和Y虽然是4bit数，但是最大就是4(4bit最多就4个1)，X＋Y最大是8，所以4bit完全可以存上，不会进位。直接计算XY+0X,低位的X+Y的结果不会进位。

	然而第二步就不能这么优化， 因为4bit最大可以是4（100），存到2bit里面，可能会产生进位，所以必须先与后加。

+ 第四～六优化都是一样的，结果都不用与了，直接相加。之所以第三步最后要做一次与，是把高位置0，防止下一步计算会污染。但是从第四步开始，就不存在这种情况。举个例子。

	比如32bit数[A][B][C][D], 其中A～D都是8bit数。第四步结果 [A][B][C][D] + [0][A][B][C] = [...][A+B][...][C+D] A＋B表示两个8bit数中1的个数，最多16个，完全存在8bit不会溢出

	第五步，计算结果就是[...][...][...][A+B+C+D]，4个8bit的1个个数，最多32，也是完全可以存放到8bit。而前一步没有通过与清干净的[...]也不会对这一步造成污染。最后结果就是低8bit。

	如果64bit也是同理。最后与0x7F，是因为64bit最多就是7bit就能存下。

###shift_and_add优化深入之一
``` c
const uint64 h01 = 0x0101010101010101;
int popcount_3(uint64 x) {
    x -= (x >> 1) & m1;  
    x = (x & m2) + ((x >> 2) & m2);
    x = (x + (x >> 4)) & m4;    
    return (x * h01)>>56;
}
```
思路：参考下图，用乘法代替最后的4~6步加法。CPU有专门的乘法器，进一步减少指令数。总共需要 4次shift, 4次and, 2次加法，1次减法，1次乘法共12次算数运算。说明如下：
```
             0a 0b 0c 0d
*
             01 01 01 01
---------------------
             0a 0b 0c 0d
          0a 0b 0c 0d
       0a 0b 0c 0d
    0a 0b 0c 0d

```
###shift_and_add优化深入之二
``` c
int popcount_3(uint64 x) {
    x -= (x >> 1) & m1;  
    x = (x & m2) + ((x >> 2) & m2);
    x = (x + (x >> 4)) & m4;    
    return x % 255
}
```
仅仅最后一步，做了修改，直接％255，得到结果。这个解释清楚，需要了解一个数学定理：

K进制数B (BnBn-1...B1B0)，B=Bn*Kn-1+Bn-1*Kn-2+...+B1*K+B0 Ki≡1(mod K-1)

可以利用二项式Ki=((K-1)+1)i展开，或用数学归纳法证明此结论。
```
B (mod K-1)
=Bn*Kn-1+Bn-1*Kn-2+...+B1*K+B0 (mod K-1)
=Bn+Bn-1+...+B1+B0 (mod K-1)
```
结论：一个K进制数模(K-1)的结果，等于此K进制数的各位相加再模K-1

于是，popcnt(B)≡B (mod K-1)

当然，我们能肯定 Bn+Bn-1+...+B1+B0<K-1 的话，可以加强结论：popcnt(B)=B%(K-1)

其实这个结论，我们小时候就用到，比如要判断一个10进制数字除以9的余数，大家都会口算，直接各位相加除以9。类似推论，除以3也是一样。

上面每8位一组，相当于2^8=256进制，所以用了255这个数；为了使用上面的等式计算，必须至少得3次迭代。
2次迭代创造 2^4=16 进制，而对于一个64位整形，popcount 最大值为64，64>16-1；64<256-1

###MIT HAKMEM169
``` c
typedef unsigned __int32 uint32;  
const uint32 m1=033333333333;
const uint32 m2=011111111111;
const uint32 m3=030707070707;

int popcount_hakmem(uint32 x) {
    x -= (x>>1 & m1 + x>>2 & m2);
    x += x>>3 & m3;
    return x % 63;
}
```
这个是32bit版本。总共需要3次shift, 3次and, 2次add, 1次mul, 1次mod 共10次算数运算

基本原理就是先triple 一下 popcnt(abc)=popcnt(4a+2b+c)=(4a+2b+c)-(2a+b)-a=a+b+c，接着再 double 一下，最后模63。

第一步3位数一算，第二步骤2个三位数double，就是6位数。根据前面分析，2^6 = 64，而32位，最多只能有32个1，所以直接％（64-1）得到结果

###查表法
用空间换时间，所以速度可以非常快。
``` c
const int lut[256] =
{
    0,1,1,2,1,2,2,3,1,2,2,3,2,3,3,4,1,2,2,3,2,3,3,4,2,3,3,4,3,4,4,5,
    1,2,2,3,2,3,3,4,2,3,3,4,3,4,4,5,2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,
    1,2,2,3,2,3,3,4,2,3,3,4,3,4,4,5,2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,
    2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,3,4,4,5,4,5,5,6,4,5,5,6,5,6,6,7,
    1,2,2,3,2,3,3,4,2,3,3,4,3,4,4,5,2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,
    2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,3,4,4,5,4,5,5,6,4,5,5,6,5,6,6,7,
    2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,3,4,4,5,4,5,5,6,4,5,5,6,5,6,6,7,
    3,4,4,5,4,5,5,6,4,5,5,6,5,6,6,7,4,5,5,6,5,6,6,7,5,6,6,7,6,7,7,8
};

static inline int popcount_lut(uint64 x) {
  int i, ret = 0;

  for (i = 0; i < 64; i += 8)
    ret +=lut[(x >> i) & 0xff];
  return ret;
}
```
没啥好解释的
###汇编
``` c
int assembly_popcnt(unsigned int n)
{
#ifdef _MSC_VER /* Intel style assembly */
    __asm popcnt eax,n;
#elif __GNUC__ /* AT&T style assembly */
    __asm__("popcnt %0,%%eax"::"r"(n));
}
```
现在的CPU一般都有popcnt指令，硬件直接计算出结果。这也是Google sparsehash速度很快的原因

##性能对比
我没有测试，直接从[网上](http://www.dalkescientific.com/writings/diary/archive/2008/07/03/hakmem_and_other_popcounts.html)找到的资料：

基本结论：naive > less_naive > shift_and_add > shift_add_add_opt > HAKMEM169 > shift_and_add_mul > lut > asm

汇编最快，查表法也很快，而之前分析的复杂的算法，因为用到了mod等运算，速度没有那么快。

结论：KISS，最简单的算法，其实速度最快。


