# 信息的表示与处理
## 数据
### 数据大小
数据类型|在32位机器空间|在64位机器空间
:-----:|:-----------:|:-----------:
char/unsigned char|1|1
short/unsigned short|2|2
int/unsigned (int)|4|4
long/unsigned long|4|8
int32_t/uint32_t|4|4
int64_t/uint64_t|8|8
typename*|4|8
double|8|8
float|4|
### 寻址和字节顺序
表示方法|0x100|0x101|0x102|0x103|典型CPU
:-----:|:---:|:---:|:---:|:---:|:-----:
大端法|01|23|45|67|Intel
小端法|67|45|23|01|IBM,Oracle
双端法|两个都可以||||ARM,取决于系统
## 编码：公式及对字母的定义
补码：$B2T_w(\vec{x})=-x_{w-1}2^{w-1}+\sum_{i=0}^{w-2}{x_i2^i}$  
原码：$B2S_w(\vec{x})=(-1)^{x_{w-1}}(\sum_{i=0}^{w-2}x_i2^i)$  
反码：$B2O_w(\vec{x})=-x_{w-1}(2^{w-1}-1)+\sum_{i=0}^{w-2}{x_i2^i}$  
原码第一位是正负号，反码即原码取反，补码即+1

$U2T_w(u)=u    (u\leqslant TMax_w)$  
$U2T_w(u)=u-2^w(u\gtrdot TMax_w)$  
$T2U_w(x)=x+2^w(u\lessdot 0)$  
$T2U_w(x)=x    (u\geqslant 0)$  
$\Rightarrow U2T_w(u)=-u_{w-1}2^w+u$  
$\Rightarrow T2U_w(x)=x_{w-1}2^w+x$  

B：二进制数组  
T：补码结果  
U：无符号结果

## 运算
### 加减法
$x+_w^uy=x+y(x+y<2^w)$  
$x+_w^uy=x+y-2^w(x+y>=x^w)$

$x+_w^ty=x+y-2^w(2^{w-1} \leqslant x+y)$  
$x+_w^ty=x+y(-2^{w-1}\leqslant x+y \lessdot 2^{w-1})$  
$x+_w^ty=x+y+2^w(x+y \lessdot -2^{w-1})$

$\Rightarrow x+_wy=(x+y)\mod2^w$
### 乘法
$x*_w^uy=(x*y)\mod2^w$  
$x'*_w^ty'=[(x+x_{w-1}2^w)(y+y_{w-1}2^w)]\mod2^w=xy\mod2^w$

## 浮点数：IEEE
种类|符号位S|指数位exp|小数位frac
:-----:|:---:|:---:|:---:
双精度|63|62-52|51-0
单精度|31|30-23|22-0
$$res=(-1)^S(1+\frac{frac}{M})2^{exp-bias}$$  
$bias=2^{k-1}-1,k:bits\ of\ exp$  
$\Rightarrow 127(float)/1023(double)$
$M=2^p,p:bits\ of\ frac$

以单精度为例子：  
$* 00000000 ***********************$意味着这个东西太小了，此时令$exp=1$。  
$* 11111111 00000000000000000000000$意味着这个东西太大了，此时已经溢出。  
$* 11111111 ***********************$除了上一种情况之外就归入这一类，此时表示$NaN$，比如$\sqrt{-1}$，$\inf-\inf$  
其他都是正常值。