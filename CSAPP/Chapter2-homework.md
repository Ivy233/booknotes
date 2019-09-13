# 信息的表示与处理 家庭作业
### 2.56
```C++
typedef unsigned char *byte_pointer;
void show_bytes(byte_pointer start, size_t len)
{
    size_t i;
    for (i = 0; i < len; i++)
        printf(" %.2x", start[i]);
    printf("\n");
}
void show_int(int x)
{
    show_bytes((byte_pointer)&x, sizeof(int));
}
void show_float(float x)
{
    show_bytes((byte_pointer)&x, sizeof(float));
}
void show_pointer(void *x)
{
    show_bytes((byte_pointer)&x, sizeof(void *));
}
```
### 2.57
略：因为只需要把变量类型稍微改改就可以了。  
实践类型直接略过。
### 2.58
这个题有很多方法，假设弄一个前八个$bit$和后八个$bit$不同的$int$。  
如果这个机器是小端法的，那么前八个$bit$就是我们预设的，反之不是。  
可是肯定没有1这么方便#滑稽。
```C++
int is_little_endian(){
    int x = 1;
    return *((char *)&x);
}
```
### 2.59
0xff肯定是好东西。这题的二进制要求很明显了。
```C++
int tmp_2_59(int x, int y)
{
    return (x & 0xff) | (y & (~0xff));
}
```
### 2.61
这题比较有意思。最大的难点在不能用$==$和$!=$，所以!直接输出逻辑1是一个比较好的方式。  
第一问直接取反，如果不全是1，会导致$~x!=0$。  
第二问不用多说。  
第三问难点在只考虑低八位，^和!一起使用可以判断相等。  
第四问难点在只考虑高八位，如果不全是0，那么右移适当的位数过后会导致整个数不为0。
```C++
void tmp_2_60(int x)
{
    printf("%d\n", !(~x));
    printf("%d\n", !x);
    printf("%d\n", !((x & 0xff) ^ 0xff));
    printf("%d\n", !(x >> ((sizeof(int) - 1) << 3)));
}
```
### 2.62
```C++
int int_shifts_are_arithmetic()
{
    return (-1 >> 1) == -1;
}
```
### 2.63
这个题神题不解释。  
$xsra得到srl的方式就是得到主要部分然后位与需要的位数就可以了。当k=0的时候，-1是没有问题的。$  
$xsrl得到sra的方式需要注意k=0，如果没有w-k-1的-1，会导致left=-1，然后xsrl|-1，你懂的。$
```C++
unsigned srl(unsigned x,int k){
    unsigned xsra = (int)x >> k;
    int w = sizeof(int) << 3;
    return xsra & ((1 << w - k) - 1);
}
int sra(int x,int k){
    int xsrl = (unsigned)x >> k;
    int w = sizeof(int) << 3;
    int left=0;
    (x & INT_MIN) && (left = ~((1 << w - k - 1) - 1));
    return left | xsrl;
}
```
### 2.64
汉明质量的变体，具体的自行CSDN
```C++
int any_odd_one(unsigned x)
{
    x |= x >> 16;
    x |= x >> 8;
    x |= x >> 4;
    x |= x >> 2;
    x |= x >> 1;
    return x & 1;
}
```
### 2.65
把上一题的位或改成异或即可
### 2.66
```C++
int leftmost_one(unsigned x)
{
    x |= x >> 1;
    x |= x >> 2;
    x |= x >> 4;
    x |= x >> 8;
    x |= x >> 16;
    return (x >> 1) + 1;
}
```
### 2.67
$这题最主要的是绕开左移32位是不建议的着一个东西。题目给的算法思路没有问题。$  
$原题目给的1<<32是被当作1<<0=1处理了。$
```C++
int int_size_is_32_for_16bit()
{
    int set_msb = 1 << 15 << 15 << 1;
    int beyond_msb = set_msb << 1;
    return set_msb && !beyond_msb;
}
```
### 2.68
$给出了两个版本，第一个利用了1<<32的溢出。$  
另外一个版本是很舒服的右移思路。
```C++
int lower_one_mask(int n){
    return ((1 << (n - 1)) << 1) - 1;
    //int w = sizeof(int) << 3;
    //return (unsigned)-1 >> (w - n);
}
```
### 2.69
这个题出的不错。  
$一个是要考虑到unsigned化再进行处理。如果int处理，在负号为处理很难受。$  
$另外一个是注意到n=0时，t1=t。$
```C++
int rotate_left(unsigned x, int n)
{
    int w = sizeof(int) << 3;
    unsigned t = x << n;
    unsigned t1 = x >> (w - n); //n==0 <==> t1==t
    return t | t1;
}
```
### 2.70
$注意：n bits只能有n-1可以表示数字，所以正好是n-1位。$
```C++
int fits_bits(int x,int n)
{
    int z = x >> n - 1;
    return !(z >> 1) || (!~z);
}
```
### 2.71
$A：错误在没有扩展成int，而且直接扩展是0扩展而不是符号扩展$  
$B：见代码，核心在于先左移再右移，利用算术右移的符号扩展特性。但是减法数量超标。$$有另外一种思路就是~0xff看情况赋值，利用\&\&的短路条件实现if。$
```C++
int xbyte(unsigned word,int bytenum)
{
    int number = (word >> (bytenum << 3)) & 0xff;
    (number & 0x80) && (number|=~0xff);
    return number;
}
```
### 2.73
$注意不能用if，可以用\&\&和\|短路掉。$
```C++
int saturating_add(int x,int y)
{
    int sum = x + y;
    ((x & y & ~sum & INT_MIN) && (sum = INT_MIN)) || ((~x & ~y & sum & INT_MIN) && (sum = INT_MAX));
    return sum;
}
```
### 2.74
思路类似于上一个题目。  
给出一个网友的实现。
```C++
int tsub_ok(int x, int y)
{
    int sum = x - y;
    int w = (sizeof(int) << 3) - 1;
    x >>= w, y >>= w, sum >>= w;
    return (x != y) && (sum == y);
}
```
### 2.75
$为什么在res之外还要减去那么多奇怪的东西？$  
$考虑一下CSAPP P68最上面的处理。$
```C++
int signed_high_prod(int x, int y)
{
    return ((long long)x * (long long)y) >> ((sizeof(long long) - sizeof(int)) << 3);
}
unsigned unsigned_high_prod(unsigned x, unsigned y)
{
    unsigned t = signed_high_prod(x, y);
    int l = (sizeof(int) << 3) - 1;
    return t - ((x >> l) && x) - ((y >> l) && y);
}
```
### 2.76
$注意：这一段代码没有真实测验过，只能尽力保证没有bug。*涉及内存和野指针$  
$这个题目除了标准的malloc和memset以外，还要注意各种各样的防止溢出。$
```C++
void *calloc(size_t nmemb, size_t size)
{
    size_t sum = nmemb * size;
    if (!(nmemb && size && sum / size == nmemb))
        return NULL;
    void *p = malloc(sum);
    memset(p, 0, sizeof(sum));
    return p;
}
```
### 2.77
A:(x<<4)+x  
B:-(x<<3)+x
C:(x<<6)-(x<<2)
D:-(x<<7)+(x<<4)
### 2.78
注意正确的舍入方式有很多种理解。  
向零舍入。
```C++
int devide_power2(int x, int k)
{
    int bias = (x >> 31) & ((1 << k) - 1);
    return (x + bias) >> k;
}
```
四舍五入
```C++
int devide_power2(int x, int k)
{
    x += ((1 << k) >> 1);
    return x >> k;
}
```
### 2.81
听说左移32位等于没有左移？直接ull。
```C++
void tmp_2_81(int k, int j)
{
    int A = (-1) << k;
    int B = (~((unsigned long long)-1 << k + j)) >> j << j;
    printf("%x %x\n", A, B);
}
```
### 2.83
$$\sum_{i=1}^{+\infty}\frac{Y}{2^{ik}}$$  
$$\frac{5}{7}$$
$$\frac{2}{5}$$
### 2.84
给两个判断的版本。速度差不多。  
注意不要为了省空间用bool存储sx和sy，会拉慢90ms/100w个的速度。  
如果需要继续加速，把f2u放进函数比较
```C++
unsigned f2u(float x)
{
    return *(unsigned *)&x;
}
int float_le1(float x, float y)
{
    unsigned ux = f2u(x), uy = f2u(y);
    unsigned sx = ux >> 31, sy = uy >> 31;
    return (sx && !sy) || (sx == sy && (sx == (ux > uy) || ux == uy)) || !((ux << 1) || (uy << 1));
}
int float_le2(float x, float y)
{
    unsigned ux = f2u(x), uy = f2u(y);
    unsigned sx = ux >> 31, sy = uy >> 31;
    return (sx && !sy) || (!sx && !sy && ux <= uy) || (sx && sy && (ux >= uy)) || ((ux << 1) == 0 && (uy << 1) == 0);
}
```
### 2.85 略
请记住这个公式

$$res=(-1)^S(1+\frac{frac}{M})2^{exp-bias}$$  
### 2.86
这个题目里面$k=15,n=63,M=2^{63},bias=2^{k-1}-1$，此题可以考虑把整数位忽略掉。  
最小的正非规格化数即八十位除了最后一位为1，其他全0。对应十进制为$2^{-n+1-bias}$。
最小的正规格化数即exp最后一位为1，其他全0。对应十进制为$2^{1-bias}$
最大的规格化数即除了符号位为0，整数位为0，对应十进制位$(2-2^{-n})2^{bias}$
### 2.87
描述|Hex|M|E|V|D
:---:|:---:|:---:|:---:|:---:|:--:
-0|8000|0|-1|-0|-0.0
最小的>2的值|4001|$\frac{1}{1024}$|1|$\frac{1025}{512}$|2
512|6000|0|9|512|512.0
最大的非规格化数|03ff|$\frac{1023}{1024}$|-14|$\frac{1023}{1024}*2^{-14}$|0
$-\infty$|fc00|-|-|$-\infty$|$-\infty$
十六进制位3BB0的数|3bb0|$\frac{924}{1024}$|-1|$\frac{123}{128}$|0.961
### 2.88 略
### 2.89 略，考察的是基本的精度问题
### 2.90
这个题目需要注意的是：  
-149=1-bias-n  
-126=1-bias  
128=255-biasssss  
其他的就没什么好说的了
```C++
float fpwr2(int x)
{
    unsigned exp, frac;
    unsigned u;
    if (x < -149)
        exp = frac = 0;
    else if (x < -126)
        exp = 0, frac = (1 << (x + 149));
    else if (x < 128)
        exp = x + 127, frac = 0;
    else
        exp = 255, frac = 0;
    u = exp << 23 | frac;
    return *(float*)&u;
}
```
### 2.92
代码舍弃了$sign$的运算，因为本身没有用到过。  
针对NaN的简单运算，其实编译器(clang++)并没有$NaN$判定，直接简单的异或。  
系统内存转换效率真的可怕。
```C++
typedef unsigned float_bits;
float_bits float_negate(float_bits f)
{
    unsigned exp = f >> 23 & 0xff;
    unsigned frac = f & 0x7fffff;
    return exp == 0xff && frac ? f : f ^ INT_MIN;
}
```
### 2.93
同样的道理。时间没有测量，不过应该情况类似。
```C++
typedef unsigned float_bits;
float_bits float_absval(float_bits f)
{
    unsigned exp = f >> 23 & 0xff;
    unsigned frac = f & 0x7fffff;
    return exp == 0xff && frac ? f : f & INT_MAX;
}
```
### 2.94
如果是乘以2的话，基于位运算比较好处理。
当$|f|$是$NaN$或者$\infty$的时候，返回$f$。  
当$|f|$是实在太小的时候的时候，不处理$exp$，直接左移$frac$，当$frac$移出界的时候，就是$exp=1$的时候，这个时候，只需要$sign|exp|frac$放在合适位置然后位或即可。  
正常情况下，$exp$只需要加1就可以了。如果$exp$超界了，$frac=0$表示这玩意无穷大。  
系统效率还是真的可怕。
```C++
typedef unsigned float_bits;
float_bits float_twice(float_bits f)
{
    unsigned sign = f >> 31;
    unsigned exp = f >> 23 & 0xff;
    unsigned frac = f & 0x7fffff;
    if (exp == 0xff)
        return f;
    exp ? (++exp == 0xff ? frac = 0 : 0) : frac <<= 1;
    return (sign << 31) | (exp << 23) | frac;
}
```
### 2.95
这题主要还是多情况处理。如果$exp=0$，只需要对$frac$处理一下就可以了。如果$exp>1$，$--exp$就可以。最复杂的是$exp==1$的情况，不仅要处理$--exp$，还要$frac$的处理。就?:表达式而言，处理的越好，这玩意处理的越快。
```C++
typedef unsigned float_bits;
float_bits float_half1(float_bits f)
{
    unsigned sign = f >> 31;
    unsigned exp = f >> 23 & 0xff;
    unsigned frac = f & 0x7fffff;
    exp <= 1 ? ((exp ? frac |= 0x800000, --exp : 0), frac = (frac + ((frac & 3) == 3)) >> 1) : --exp;
    return (sign << 31) | (exp << 23) | frac;
}
float_bits float_half2(float_bits f)
{
    unsigned sign = f >> 31;
    unsigned exp = f >> 23 & 0xff;
    unsigned frac = f & 0x7fffff;
    exp ? (--exp == 0 ? frac |= 0x800000, frac = (frac + ((frac & 3) == 3)) >> 1 : 0) : frac = (frac + ((frac & 3) == 3)) >> 1;
    return (sign << 31) | (exp << 23) | frac;
}
```
### 2.96
这个题就是求$(1<<n|frac)2^{exp-0x7f-23}$，需要对$exp-0x7f$，$exp-0x7f-23$进行判定和越界判定。
```C++
typedef unsigned float_bits;
int float_f2i(float_bits f)
{
    unsigned exp = (f >> 23 & 0xff) - 0x7f;
    unsigned frac = (f & 0x7fffff) | (0x800000);
    if (exp & INT_MIN)
        return 0;
    if ((exp + 1) & 0xe0)
        return 0x80000000;
    exp > 23 ? frac <<= (exp - 23) : frac >>= (23 - exp);
    return f >> 31 ? (~frac + 1) : frac;
}
```
### 2.97
这个题未来还需要优化，虽然有花了一点，但是时间效率奇低。  
这个题需要注意的地方有很多。首先是要判断正负号，而且$0x80000000$的相反数不存在，直接转换成$0x7fffffff$即可。然后是判断出最高位，并且截取成23位长度。这里使用了循环展开进行加速。接着是四舍五入部分，如果$frac$后面的数字不是很小可以考虑进位，进位规则见代码。最后是$exp$的计算需要注意$frac$因为进位而导致$exp$进位。
```C++
typedef unsigned float_bits;
float_bits float_i2f(int i)
{
    if (i == 0)
        return 0;
    unsigned sign = 0, frac = i;
    i &INT_MIN ? (sign = INT_MIN, frac = (~frac) + (i != INT_MIN)) : 0; //get sign
    unsigned pos = 31;
    frac & 0xffff0000 ? 0 : (frac <<= 16, pos = 15);
    frac & 0xff000000 ? 0 : (frac <<= 8, pos ^= 8);
    frac & 0xf0000000 ? 0 : (frac <<= 4, pos ^= 4);
    frac & 0xc0000000 ? 0 : (frac <<= 2, pos ^= 2);
    frac & 0x80000000 ? 0 : (frac <<= 1, pos ^= 1);
    frac = frac << 1;
    frac = (frac >> 9) + ((frac & 0x100) && ((frac & 0x200) || (frac & 0xff)));
    unsigned exp = pos + 0x7f + ((frac & 0x800000) != 0);
    return sign | (exp << 23) | (frac & 0x7fffff);
}
```