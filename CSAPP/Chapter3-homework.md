#程序的机器级表示 家庭作业
### 建议：在做作业的时候注意查阅机器指令。
### 3.58
```C++
long decode2(long x,long y,long z)
{
    y-=z;x*=y;
    return x^(y<<63>>63);
}
```
### 3.59
在整个代码里面，%rcx,%rax,%rdx同时有几个作用，有很多变量都存储在这上面。这造成了比较大的难度。
编译器的解析和CSAPP第三章习题的提示是一样的。这段代码将x,y符合扩展成128位，此时本来的位置就是ull类型，高64位才具有符号位特性，然后
```
res =(x[1]*W+x[0])*(y[1]*W+y[0])
    =x[1]*y[1]*W*W+
     (x[1]*y[0]+x[0]*y[1])*W+
     x[0]*y[0]
```
注意到W\*W超出了128位，而且x[0]*y[0]会超出64位，所以以它的高位为基础，两边计算即可。
```C++
%rax=%rdx;                  # %rax=%rdx=y0
%rdx=(%rax>>63);            # %rdx=y1
%rcx=%rsi;                  # %rsi=%rcx=x0
%rcx>>=63;                  # %rcx=x1
%rcx*=%rax;                 # x1*=y0
%rdx*=%rsi;                 # y1*=x0
%rcx+=%rdx;                 # %rcx=x1y0+x0y1;
(%rdx:%rax)=%rax*%rsi;(u)   # resh=x0y0>>63,resl=x0y0&(unsigned)-1
%rdx+=%rcx;                 # resh+=%rcx
```
### 3.60
```C++
long loop(long %rdi,int %esi)
{
    %ecx=%esi;
    $edx=1;%eax=0;
L3:
    %r8=%rdi;
    %r8&=%rdx;
    %rax|=%r8;
    %rdx<<=%cl;
L2:
    if(%rdx)goto L3;
    return %rax;
}
```
这里注意：%cl是%rcx的8bit版本。从最后的rax可以定位到C语言result就是%rax；而mask初始值为1就可以定位为%rdx。根据最后的循环条件可以确认mask和rdx的关系。之后所有的都确定了。  
那么r8是干什么的？我们可以考虑一下去掉r8就可以发现%rax|=(%rdi&%rdx)根本无法实现。
### 3.61
```C++
long cread_alt(long* xp)
{
    long t=0;
    return (xp?*xp:t);
}
```
### 3.62
```C++
typedef enum{MODE_A,MODE_B,MODE_C,MODE_D,MODE_E}mode_t;
long switch3(long *p1,long *p2,mode_t action)
{
    long result=0;
    switch(action){
        case MODE_A:result=*p2,*p2=*p1;break;
        case MODE_B:result=*p1+*p2,*p1=result;break;
        case MODE_C:*p1=59,result=*p2;break;
        case MODE_D:result=*p2,*p1=result;break;
        case MDOE_E:result=27;break;
        default:result=12;
    }
    return result;
}
```
### 3.63
```C++
int switch_prob(int x, int n)
{
    int result = x;
    switch (n)
    {
    case 40:case 42:
        result <<= 3;
        break;
    case 43:
        result >>= 3;
        break;
    case 44:
        result <<= 3;
        result -= x;
    case 45:
        result *= result;
    case 41: default:
        result += 11;
        break;
    }
    return result;
}
```
### 3.64
从最后一行可以知道整个数组大小是3640/8=455=5*7*13，嗯看上去范围已经很小了，接下来就是确定每一维有多少个东西就行了。 
``` 
//i:%rdi,j:%rsi,k:%rdx
leaq (%rsi,%rsi,2), %rax    # %rax=3%rsi=3j
leaq (%rsi,%rax,4), %rax    # %rax=13%rsi=13j
movq %rdi, %rsi             # %rsi=i
salq $6, %rsi               # %rsi=64i
addq %rsi, %rdi             # $rdi=65i
addq %rax, %rdi             # %rdi=65i+13j
addq %rdi, %rdx             # %rdx=65i+13j+k
movq A(,%rdx,8), %rax
```
i++=>总地址+65\*8，j++=>总地址+13*8。这意味着RST分别为7，5，13.
### 3.65
看见这种题目第一反应就是关注常数。显然%rdx是A[i][j],%rax是A[j][i]，于是乎此题得解。
### 3.66
代码第一部分(到jle .L4为止)是一些常数和循环初始条件的判定。从这里可以看到%rax代表了NR(n)，那么NR(n)=3n。第二部分(到.L3为止)是循环初始的赋值。%rcx是result每一次加的位置的指针。第三部分(.L3~.L4)这一部分是循环体，其中的%rdx是i，%r8是到一行的增量。答案显而易见：NR(n)=3n,NC(n)=4n+1。
### 3.67 **
这个题存在大量的垃圾值，但是不知道为什么GCC这么做，是为了越位设置的缓冲区？  

A.调用process之前，%rsp-=104，而且从下到上分别放置了x,y,&z,z，其他不知道。  

B.传递了一个%rsp+64指针，但是实际上从process里面对于栈的利用可以看出，传递了整个strA。  

C.因为这是一个连续（至少是看起来的）过程，所以process自然可以直接使用eval里面放入栈甚至寄存器的值，只要两者商量好分别使用什么东西。从形式上来说，只是表面上传递了一个指针，其他都是放在栈里面的。  

D.在eval里面通过rdi划分了一个区域给process，之后%rdi+delta偏移进行访问。  

E.整个栈大小104，但是存在大量的垃圾。从下往上分别是：x,y,&z,z;一堆垃圾;(%rsp+64)u[0],u[1],q;一堆垃圾;栈顶。  

F.结构体本身很复杂，但是每一个东西都存在用处，寄存器放不下只能丢在内存里，通过栈进行访问。垃圾值非常多，原因不明，可能是缓冲区。
### 3.68
这个题注意常数即可，需要用到不等式。  
第一行告诉我们str2里面char结构的容量最多为8，最低为5，否则会被对齐规则舍去。  
第二行告诉我们short占据到了%rsi+26~%rsi+32区间的位置。而开始位置是%rsi+12，所以5<=B<=8,7<=A<=10。  
问题是这一就不能确定A,B分别是多少。这就需要第三行，%rsi+184说明str1结构里面的x占据了栈的%rdi+180~%rdi+184区间的位置。  
加上之前的不等式就可以解出来A=9,B=5。
### 3.69 **
```
mov 0x120(%rsi),%ecx    # ecx = bp->last
add (%rsi),%ecx         # ecx += bp->first
lea (%rdi,%rdi,4),%rax  # rax = 5i
lea (%rsi,%rax,8),%rax  # rax = bp + 40i
mov 0x8(%rax),%rdx      # rdx = M[bp+40i+8]
movslq %ecx,%rcx        # rcx = ecx(符号扩展)
mov %rcx,0x10(%rax,%rdx,8)   # rcx=8*(M[bp+40i+8])+bp
                             # +40i+16
retq
```
从前面两行可以看出来a这个数组和first变量一共占据了288字节。这里有一个暗示，ecx扩展到了rcx，这是为了方便把int赋值给long，也就是说a_struct里面的变量都是long类型的。那么换句话说a加起来一共35个long变量。

请注意i不是临时变量，而是一个传入参数。依靠这点就可以知道40i的意思是i个以前都是40个一个字节，即CNT=7。

剩下的就简单了：35/7=5即可。一个里面只有五个long，还要分一个给idx。

有点像寄存器的实现。
### 3.70
A.注意这玩意是一个union，两个内存空间是共用的。0 8 0 8。

B.16

C.
```
movq 8(%rdi),%rax
movq (%rax),%rdx
movq (%rdx),%rdx
subq 8(%rax),%rdx
movq %rdx,(%rdi)
```
代码应该不需要注释了吧。  
转换成C的时候需要注意下一开始给你的是指针。
```C++
void proc(union ele* up){
    up->e2.x=*(up->e2.next->e1.p)-(up->e2.next->e1.y);
}
```
### 3.71
首先吐槽下有输入没有输出是几个意思？#滑稽
```C++
void good_echo()
{
    char buffer[10];
    while (fgets(buffer, 10, stdin))
    {
        printf("%s", buffer);
        if (ferror(stdin))
        {
            printf("\nError\n");
            break;
        }
        else if (feof(stdin))
        {
            printf("\nError\n");
            break;
        }
    }
}
```
这个稍微改了改网上的版本[CSDN-CHOOOU](https://blog.csdn.net/one_of_a_kind/article/details/81738501)看起来没有问题，刚才一试发现输入100个字符原样输出。编译环境clang++ C++17标准。

然后我看到了这个版本：[CSND-maidou0921](https://blog.csdn.net/maidou0921/article/details/53907971)
```C++
void good_echo()
{
    int x = 0;
    while (x = getchar(), x != '\n' && x != EOF)
        putchar(x);
}
```
这个版本不错，直接针对每一个字符处理，虽然没用上fgets（当然不是直接用）
### 3.72
这个题极度类似p202的代码，重点考虑-16的二进制对齐意义。具体的还是省略把。
### 3.73
这个是引用asm来写入汇编代码。有几点需要注意的：  
第一是vucomiss是两个反过来比较的，这一点特别绕人。  
第二个是jp，[网上](https://blog.csdn.net/weixin_32589873/article/details/78207020)说是奇偶位有东西就会跳转，没有理解为什么NaN会导致奇偶位设置位为1。  
第三个是最后的:传入传出变量，因为有这个的存在，不需要专门指定寄存器，这部分是支持C语法的。
```C++
int find_range(float x)
{
    int result;
    float zero = 0.0;
    asm(
        "vucomiss %[x],%[z] \n\t"
        "jp .OTHER \n\t"
        "ja .NEG \n\t"
        "jb .POS \n\t"
        "mov $1,%[r] \n\t"
        "jmp .DONE \n\t"
        ".OTHER:mov $3,%[r] \n\t"
        "jmp .DONE \n\t"
        ".NEG:mov $0,%[r] \n\t"
        "jmp .DONE \n\t"
        ".POS:mov $2,%[r] \n\t"
        ".DONE:nop"
        : [r] "=r"(result)          /*output*/
        : [x] "x"(x), [z] "x"(zero) /*input*/
    );
    return result;
}
```
### 3.74
类似的原理，看得懂上一题的也不需要看这个题了。
```C++
int find_range(float x)
{
    int result;
    float zero = 0.0;
    asm(
        "movl $0, %%r8d\n\t"
        "movl $2, %%r9d\n\t"
        "movl $3, %%r10d\n\t"
        "vucomiss %[x], %[z] \n\t"
        "movl $1, %%eax \n\t"
        "cmovb $2, %[r] \n\t"
        "cmova %%r8d, %[r] \n\t"
        "cmovp %%r10d, %[r] \n\t"
        : [r] "=r"(result)
        : [x] "x"(x), [z] "x"(zero) /*input*/
    );
    return result;
}
```
这个版本看起来一点问题都没有，但是实际上，cmov系列指令要求寄存器。然后很容易修改成这个版本：
```C++
int find_range(float x)
{
    int result;
    float zero = 0.0;
    asm(
        "mov $0, %%r8\n\t"
        "mov $2, %%r9\n\t"
        "mov $3, %%r10\n\t"
        "vucomiss %[x], %[z] \n\t"
        "mov $1, %%rax \n\t"
        "cmovb $$r9d, %[r] \n\t"
        "cmova %%r8d, %[r] \n\t"
        "cmovp %%r10d, %[r] \n\t"
        : [r] "=r"(result)
        : [x] "x"(x), [z] "x"(zero) /*input*/
    );
    return result;
}
```
这个看起来更加没有问题，不过实际上result会被解析成放在eax，然后cmovb就很无奈了，一个64位寄存器要放在32位里面，放不进去啊。这里只有两种解决方法：一个是把result改成long long，一个是把所有的mov改成movl或者%%r8系列改成32位寄存器。
```C++
int find_range(float x)
{
    int result;
    float zero = 0.0;
    asm(
        "movl $0, %%r8d\n\t"
        "movl $2, %%r9d\n\t"
        "movl $3, %%r10d\n\t"
        "vucomiss %[x], %[z] \n\t"
        "movl $1, %%eax \n\t"
        "cmovb %%r9d, %[r] \n\t"
        "cmova %%r8d, %[r] \n\t"
        "cmovp %%r10d, %[r] \n\t"
        : [r] "=r"(result)
        : [x] "x"(x), [z] "x"(zero) /*input*/
    );
    return result;
}
```
### 3.75
从c_real的代码的其特性可以猜到肯定是什么特殊的结构导致了第一个complex变量的实部在%xmm0。  
然后从c_imag里面可以猜到虚部放在%xmm1。  
最后从c_sub里面可以猜到每一个complex占用两个寄存器，实部放在%xmm0，虚部%xmm1，一个一个往后面排。