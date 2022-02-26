# 让自己习惯C++

## Item 1 视C++为一个语言联邦
众所周知，C++是一门无法精通的语言，这门语言同时支持面向过程，面向对象，函数式编程，泛型以及元编程的形式，而这些组合起来其实没有这么多，一共就四种语言形式。

+ C
+ OO C++
+ Template C++
+ STL

但是神奇的是这些"语言"在相互切换的时候大部分规则都是适用的，或者只需要做出少量变化即可。而这些变化的核心只有一个：解决程序的二义性和明确调用的函数/方法是哪一个。
## Item 2 尽量以const, enum, inline替换 #define
### 奇怪的输出
#define的形式大致如下
```C++
#define pi acos(-1)
#define f(x) ((x)*(x))
#define f(x) (x*x)
#define e 2.71828
```
大致常用的就这些形式把。有些习以为常的区别就不讲了（注意第二个和第三个）。

就算是按照第二行来定义你期望的函数，也有可能会按照奇怪的形式运行，比如
```C++
f(x++) => ((x++)*(x++))
f(++x) => ((++x)*(++x))
```
请问这两个函数的输出会是什么？你可能在一个编译器中得到一个答案，在另外一个编译器中得到另外一个答案；也有可能在两个编译器中得到同一个答案，但是和你预计的不太一样。使用#define也有很多不可预料的行为。如果说这个例子还不足够说服你，那么我们再来看一个例子：
```C++
#define CALL_WITH_MAX(a,b) f((a) > (b) ? (a) : (b))
int a = 5, b = 0;
CALL_WITH_MAX(++a,b);
CALL_WITH_MAX(++a,b+10);
```
仔细观察就可以知道，传入f的参数具体大小甚至会因为比较结果而改变。第一个调用只`++a`了一次，但是第二个则是两次。

对于这种问题，最好的解决办法就是包装成一个函数。
```C++
template<class T>
inline T f(const T& x)
{
    return x * x;
}
```
在传入++x的时候，肯定是会先完成加法以后再进入函数，++x也是如此。
### 这个数哪里来的
```C++
#define _________ }
#define ________ putchar
#define _______ main
#define _(a) ________(a);
#define ______ _______(){
#define __ ______ _(0x48)_(0x65)_(0x6C)_(0x6C)
#define ___ _(0x6F)_(0x2C)_(0x20)_(0x77)_(0x6F)
#define ____ _(0x72)_(0x6C)_(0x64)_(0x21)
#define _____ __ ___ ____ _________
#include<stdio.h>
_____
```
看到这段代码的时候，我的内心是黑人问号的。虽然这样玩有点过分，但是一定会有一种情况你遇到过。在写代码的时候，你调用了一个常量，但是在编译的时候说这块地方出了问题，然后一看信息，说`acos(-1)`没有定义。woc这个东西哪里来的？我没写过啊。

但是如果按照下面这么写呢？是不是好追踪很多了？
```C++
class Int
{
    static const int num = 5;   //只有int有可能可以在内部初始化
    static const double a;      //但是还是建议采用这一行表述方式
}
Int::a = 1.35;                  //在这里初始化
```
如果不允许在类内初始化`static const int`，并且需要用这玩意定义数组（只在`STL`的内存分配器中见过）的时候，建议采用`enum`来定义这个`static const int`。而且顺带提一句，`enum`变量取不出地址（笑）。
## Item 3 尽可能使用const
### 快速看点
+ `const`变量有若干种形式。但是特别注意区分`const int *`和`int *const`
+ 类中可以实现同名同参数表的函数两次，`const`函数会优先被`const`对象调用，反过来也是。
+ `const`函数如果返回引用，那么可能起不到`const`效果，因为这个东西可以在外部被改变。换句话说，`const`只能保证函数定义内部不修改成员变量。
+ 如果想要在`const`内部修改成员变量，可以考虑在对应变量上面加入`mutable`
### const的一些用法
const的用法应该基本熟悉，赋值以后不允许改变。对于指针而言则有几种：

+ int *a;               //指针
+ const int *a;         //指针指向的值不可以改，但是可以改变指针指向哪里
+ int *const a;         //指针指向不可以改，但是可以改变指向的值
+ const int *const a;   //两个都不可以改
+ const vector<int>::iterator;  //在STL中，类似于const int *
+ const_iterator                //在STL中，类似于int *const

之后有一处地方不太理解，如果
```C++
if(a * b = c) // 本意是一个比较 a * b == c
```
如果`a`是一个内置类型，那么就会编译错误。但如果是自定义类型呢？建议加一个`const`。

### const成员函数
如果你看过`STL`源码的话，可以看到有类似于这样的实现
```C++
template<class T>
class vector
{
    const T& operator[](std::size_t x) const {return *(begin() + x);}
    T& operator[](std::size_t x) {return *(begin() + x);}
}
```
完全一样的返回类型，完全一样的形参表，为什么会有两个这样的函数？实际上在`const`变量上面调用`[]`，会调用前者，反之会调用后者。这样就区分开两个函数了。

我们给一个函数附上`const`，是希望这个函数不允许修改任何成员变量，比如说：
<span id='Point'></span>

```C++
class Point
{
    double x, y;
    void setPoint(const double &_x, const double &_y)
    {
        x = _x;
        y = _y;
    }
};
class Rectangle
{
    Point LeftUpside, RightDownside;
    Point& getLeftUpside() const
    {
        return LeftUpside;
    }
} rec;
```
然后这么用
```C++
rec.getLeftUpside().setPoint(x, y);
```
于是`const`成员函数涉及到的变量就被修改了。其实这也正常，因为这根本就不是函数的范围之内，而是在对一个对象进行操作。
### 代码重用性
这个时候又有另外一个问题出现了，如果`const`带和不带的实现是一样的，而且代码比较长，如何提升代码的重用率，降低出错的可能？

可以考虑这样：
```C++
class TextBlock
{
public:
    const char &operator[](const size_t &x) const {} //...原来的代码不变
    char &operator[](const size_t &x)
    {
        return const_cast<char &>(
            static_cast<const TextBlock &>(*this)
                [x]
        );
    }
}
```
为什么不反过来调用，即写出`char&`的实现，`const char &`来调用它？C++有一个规定，即不允许const函数调用非`const`成员函数。这个规定其实非常合理，因为不能保证在非`const`中修改变量（从现在或者以后），所以干脆所有的都一刀切断。反过来显然是可以的。

特别要注意一点的是注意强制类型转换，这里用到了两个。如果内层的`const`不加上的话，会自己调用自己。如果没有外层的`const_cast`，那么会产生类型不匹配，`char &`不能拿到一个`const`的引用，不然怎么修改呢？
### 两种观念
1. bitwise const：成员函数只有在不更改对象的任何成员变量才可以说是`const`，`static`除外。
2. logical const：成员函数可以在客户端侦测不出的情况下修改对象也可以说是`const`。

编译器甚至采用了第一种方法，因为实现起来简单。甚至你无法想象什么叫做客户端侦测不出（至少我想不出来）。但是如果你希望实现`logic`呢？在涉及到的变量上面加入`mutable`，这样所有`const`函数都可以修改这玩意了（好危险啊）。

这个时候回头来看[这一段代码](#Point)
那这是不是意味着用的是logic？其实不是，因为修改`getLeftUpside`的返回值根本不是它自己的的事情，已经结束这个函数了才修改。

## Item 4 确定对象被使用前已先被初始化
### 别忘记初始化对象了
在任何一个`class`中，如果不提供初始化函数，那么对象有可能会不初始化。众所周知，不进行初始化是非常危险的，因为它很有可能引发不可预料的错误。在`class`中有两类初始化方式：
```C++
class Student
{
    std::string name;
    std::string stuid;
    int age;
    Student(const std::string& _name, const std::string& _stuid, const int& _age_)
    {
        name = _name;
        stuid = _stuid;
        age = _age;
    }
}
```
```C++
class Student
{
    std::string name;
    std::string stuid;
    int age;
    Student(const std::string& _name, const std::string& _stuid, const int& _abc)
      : name(_name), sutid(_stuid), age(_age) {}
}
```
两者在这里的结果是一样的，但是过程似乎不太一样，因为前者等价于
```C++
class Student
{
    std::string name;
    std::string stuid;
    int age;
    Student(const std::string& _name, const std::string& _stuid, const int& _age)
      : name(), sutid(), age()//默认的初始化
    {
        name = _name;
        stuid = _stuid;
        age = _age;
    }
}
```
默认初始化函数干了啥由编译器(built-in类型)和类的实现决定。显然后者比前者（忽略那个等价例子）少了一系列赋值操作。

除此以外，在`STL`中可见很多类采用了类似于这样的初始化方式：把所有初始化函数进行封装，形成一个成员函数，以`private`的形式体现，由各种初始化函数进行调用。这是因为有太多的初始化函数很类似，在不影响效率的情况下进行这样的操作是可以理解的，同时还能减少调试的复杂度。
### 跨编译单元的初始化顺序，用local static替换掉non-local static对象
假设有一个`FileSystem`的类，它让互联网上的文件看上去像是一个本机。由于这个`class`形成了一个单一文件系统，那么会产生一个对象在`global`作用域内
```C++
extern FileSystem tfs; // 多文件使用
```
如果在`tfs`构造完成之前调用它的成员函数那么会产生很严重的后果。比如在另外一个文件中，需要定义一个`class`交作`Directory`，用作目录
```C++
class Directory // 注意，这在另外一个文件中
{
public:
    Directory(params) // 这里的params省略
    {
        ...
        std::size_t disks = tfs.numDisks();
        ...
    }
}
```
如果`tfs`在`Directory`初始化的时候没有定义过，不就滑稽了吗#滑稽。
因为这两个东西可能是不同的人在不同的时间于不同的源码文件建立起来的，所以根本无法保证初始化顺序。这样的解决办法可以是
```C++
class FileSystem
{
public:
    std::size_t numDisks() const;
};
FileSystem& tfs()
{
    static FileSystem fs;
    return fs;
}
class Directory
{
public:
    Directory(params)
    {
        ...
        std::size_t disks = tfs().numDisks();
        ...
    }
}
```
如果需要一个文件夹来存放临时文件的话，就只需要
```C++
Directory& tempDir()
{
    static Directory td;
    return td;
}
```
这样只需要用`tempDir()`和`tfs()`指定这个`FileSystem`和临时`Directory`即可。

看过设计模式的一眼都能看出来：这是`Singleton设计模式`的常见实现手法。