<style>
p{
    text-indent:2em;
}
</style>
# 让自己习惯C++
## Item 1 视C++为一个语言联邦
众所周知，C++是一门~~可以~~无法精通的语言，这门语言同时支持面向过程，面向对象，函数式编程，泛型以及元编程的形式，而这些组合起来其实没有这么多，一共就四种语言形式。

+ C
+ OO C++
+ Template C++
+ STL

但是神奇的是这些"语言"在相互切换的时候大部分规则都是适用的，或者只需要做出少量变化即可。而这些变化的核心只有一个：解决程序的二义性和明确调用的函数/方法是哪一个。
## Item 2 尽量以const, enum, inline替换 #define
### 导致无法预料的输出，甚至原因也无法预料
#define的形式大致如下
```C++
#define pi acos(-1)
#define f(x) ((x)*(x))
#define f(x) (x*x)
#define e 2.71828
```
大致常用的就这些形式把。但是一定要注意第二个和第三个的区别，如果你想调用
```C++
f(2*x+5)
```
在2-3行解释就是
```C++
f(x) => ((2*x+5)*(2*x+5))
f(x) => (2*x+5*2*x+5)
```
这直接造成了语义上的不同，而且这是在料想之外的。

就算是按照第二行来定义你期望的函数，也有可能会按照奇怪的形式运行，比如
```C++
f(x++) => ((x++)*(x++))
f(++x) => ((++x)*(++x))
```
请问这两个函数的输出会是什么？你可能在一个编译器中得到一个答案，在另外一个编译器中得到另外一个答案；也有可能在两个编译器中得到同一个答案，但是和你预计的不太一样。使用#define也有很多不可预料的行为。如果说这个例子还不足够说服你，那么我们再来看一个例子：
>取材自Effective C++ 3rd Edition
>```C++
>#define CALL_WITH_MAX(a,b) f((a) > (b) ? (a) : (b))
>int a = 5, b = 0;
>CALL_WITH_MAX(++a,b);
>CALL_WITH_MAX(++a,b+10);
>```

仔细观察就可以知道，传入f的参数具体大小甚至会因为比较结果而改变。第一个调用只++a了一次，但是第二个则是两次。
对于这种问题，最好的解决办法就是包装成一个函数。
```C++
template<class T>
inline T f(const T& x)
{
    return x * x;
}
```
在传入++x的时候，肯定是会先完成加法以后再进入函数，++x也是如此。
### 出错时很难查找原因
#define会在预处理期完成替换，从而把换掉的东西当成一个常量（函数太危险了）来完成计算。如果说这个时候出现了一个template导致类型不匹配，你去哪里查错？特别是在多人开发的时候，你对於这个常量从哪里来的都毫无头绪。所以这个时候建议使用const替换，关于const的指针，会在Item 3中讲解。

在类中有加入const，在每生成一个对象的同时也会产生一个const成员，并且在初次使用时初始化。这个时候就有可能const起不到const的作用，因为每一个类对象的同一个const成员含有不同的值。除非这是你期望的，否则建议你在const成员上加入static，并且在类生成的代码后面加入初始化，就像这样。
```C++
class Int
{
    static const int num = 5;   //只有int有可能可以在内部初始化
    static const double a;      //但是还是建议采用这一行表述方式
}
Int::a = 1.35;                  //在这里初始化
```
如果不允许在类内初始化static const int，并且需要用这玩意定义数组（只在STL的内存分配器中见过）的时候，建议采用enum来定义这个static const int。而且顺带提一句，enum变量取不出地址（笑）。
## Item 3 尽可能使用const
### const成员/变量
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
如果a是一个内置类型，那么就会编译错误。但如果是自定义类型呢？建议加一个const。

### const成员函数
如果你看过STL源码的话，可以看到有类似于这样的实现
```C++
template<class T>
class vector
{
    const T& operator[](std::size_t x) const {return *(begin() + x);}
    T& operator[](std::size_t x) {return *(begin() + x);}
}
```
完全一样的返回类型，完全一样的形参表，为什么会有两个这样的函数？实际上在const变量上面调用[]，会调用前者，反之会调用后者。这样就区分开两个函数了。

我们给一个函数附上const，是希望这个函数不允许修改任何成员变量，但是仅仅在形参表之后加入const不能解决这个问题，需要在返回值也加入const，举例：
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
这种情况下并没有做到
```C++
rec.getLeftUpside().setPoint(x, y);
```
因为它返回的是一个引用，这里建议给Point上面加入一个const。

这个时候又有另外一个问题出现了，如果带const和不带const的实现是一样的，而且代码比较长，如何提升代码的重用率，降低出错的可能？

可以考虑这样：
>```C++
>class TextBlock
>{
>    const char &operator[](const size_t &x) const {} //...原来的代码不变
>    char &operator[](const size_t &x)
>    {
>        return const_cast<char &>(
>            static_cast<const TextBlock &>(*this)
>                [x]
>        );
>    }
>}
>```
为什么不反过来调用，即写出char&的实现，const char &来调用它？C++有一个规定，即不允许const函数调用非const成员函数。这个规定其实非常合理，因为不能保证在非const中修改变量（从现在或者以后），所以干脆所有的都一刀切断。反过来显然是可以的。

特别要注意一点的是注意强制类型转换，这里用到了两个。如果内层的const不加上的话，会自己调用自己。如果没有外层的const_cast，那么会产生类型不匹配，char&不能拿到一个const的引用，不然怎么修改呢？
### 两种观念
1. bitwise const：成员函数只有在不更改对象的任何成员变量才可以说是const，static除外。
2. logical const：成员函数可以在客户端侦测不出的情况下修改对象也可以说是const。如果说想在const中修改成员变量呢？需要在成员变量中加入mutable。

## Item 4 确定对象被使用前已先被初始化
在任何一个class中，如果不提供初始化函数，那么对象有可能会不初始化。众所周知，不进行初始化是非常危险的，因为它很有可能引发不可预料的错误。在class中有两类初始化方式：
```C++
class Student
{
    std::string name;
    std::string stuid;
    int abc; //一时想不起来啥属性比较好
    Student(const std::string& _name, const std::string& _stuid, const int& _abc)
    {
        name = _name;
        stuid = _stuid;
        abc = _abc;
    }
}
```
```C++
class Student
{
    std::string name;
    std::string stuid;
    int abc; //一时想不起来啥属性比较好
    Student(const std::string& _name, const std::string& _stuid, const int& _abc)
      : name(_name), sutid(_stuid), abc(_abc) {}
}
```
两者在这里的结果是一样的，但是过程似乎不太一样，因为前者等价于
```C++
class Student
{
    std::string name;
    std::string stuid;
    int abc; //一时想不起来啥属性比较好
    Student(const std::string& _name, const std::string& _stuid, const int& _abc)
      : name(), sutid(), abc()
    {
        name = _name;
        stuid = _stuid;
        abc = _abc;
    }
}
```
为空意味着调用默认的初始化函数，具体初始化为什么由编译器和类的实现决定。显然后者比前者（忽略那个等价例子）少了一系列赋值操作。

除此以外，在STL中可见很多类采用了类似于这样的初始化方式：把所有初始化函数进行封装，形成一个成员函数，以private的形式体现，由各种初始化函数进行调用。这是因为有太多的初始化函数很类似，在不影响效率的情况下进行这样的操作是可以理解的，同时还能减少调试的复杂度。
### 跨编译单元的初始化顺序，用local static替换掉non-local static对象
假设有一个FileSystem的类，它让互联网上的文件看上去像是一个本机。由于这个class形成了一个单一文件系统，那么会产生一个对象在global作用域内
```C++
extern FileSystem tfs; // 多文件使用
```
如果在tfs构造完成之前调用它的成员函数那么会产生很严重的后果。比如在另外一个文件中，需要定义一个class交作Directory，用作目录
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
一旦这个出现不就滑稽了吗#滑稽。
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
这样只需要用tempDir()和tfs()指定这个FileSystem和临时Directory即可。

看过设计模式的一眼都能看出来：这是Singleton设计模式的常见实现手法。