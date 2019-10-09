# 设计与声明
<span id='Item_18'></span>

## Item 18 让接口容易被正确使用，不易被误用
### 快速看点
+ 如题，接口不止你一个人使用，但是仅仅通过代码的交流不能让编写者和使用者心意交通。
+ 接口的正确使用包括了限定合法输入。
+ 不易被误用甚至包括了同一系列的代码产品中的同类功能调用，例如STL。
+ shared_ptr是一个好东西，可以自动解锁互斥体，防止cross-DLL问题。
### 来看一个例子
```C++
class Date {
public:
  Date(int month, int day, int year);
  ...
};
```
这样设计很容易进入一个误区：
```C++
Date d(30, 3, 1995);//你可以这样
Date d(2, 20, 1995);//你也可以这样
```
虽然你希望输入的是1995.3.30和1995.2.20，但是谁会禁止你这么做呢？
### 修改一下实现
```C++
struct Day {            struct Month {                struct Year {
  explicit Day(int d)     explicit Month(int m)         explicit Year(int y)
  :val(d) {}              :val(m) {}                    :val(y){}

  int val;                int val;                      int val;
};                      };                            };

class Date {
public:
 Date(const Month& m, const Day& d, const Year& y);
 ...
};
```
再来看一下这次的调用
```C++
Date d(30, 3, 1995);                    // 编译不通过，因为explicit
Date d(Day(30), Month(3), Year(1995));  // 编译错误，因为类型不对
                                        // 而且没提供对应的初始化函数
Date d(Month(3), Day(30), Year(1995));  // 这样才能过去
```
但是
```C++
Date d(Month(13), Day(30), Year(1995));  // 黑人问号
```
### 合法性
于是乎我们再次修改一下
```C++
class Month {
public:
  static Month Jan() { return Month(1); }   // 这就是1月
  ...                                       // 以此类推
  ...                                       // 其他的方法

private:
  explicit Month(int m);                    // 阻止自定义月
  ...                                       // 其他需要的东西
};
```
这样是不是好多了？调用也显而易见。
### 之前的例子
```C++
Investment* createInvestment();
```
如果你还记得这个例子的话，当时引入了shared_ptr和auto_ptr，看上去像是给这两个东西打广告一样#滑稽。但是实际上在调用的时候，调用者有可能忘记把这玩意包装到一个shared_ptr里面，所以
```C++
shared_ptr<Investment*> createInvestment();
```
这下你还忘记？

除此以外，这样还有一个好处：我们在内部可以自定义一些内容，比如说加入自定义的析构函数，这样调用者也不可能说忘记导致的资源泄露。甚至根据这一点还可以继续改进：
```C++
std::tr1::shared_ptr<Investment> createInvestment()
{
    std::tr1::shared_ptr<Investment> retVal(
        static_cast<Investment*>(0),
        getRidOfInvestment
    );
    retVal = ... ;
    return retVal;
}

std::tr1::shared_ptr<Investment> createInvestment()
{
    std::tr1::shared_ptr<Investment> retVal(
        static_cast<Investment*>(...),
        getRidOfInvestment
    );
    return retVal;
}
```
这两个的差别在哪里呢？一个可能在初始化上面多花费一点时间，但是另外一个有可能会带来一些不好追踪的bug。只是

为什么不把return也给合并掉呢？这样不就又省下一行代码了？
### cross-DLL问题
具体定义就是一个对象在某个dll中产生，但是在另外一个dll中删除。shared_ptr可以比较好的解决这类问题。因为它的缺省的deleter只将delete用于这个shared_ptr被创建的DLL中
## Item 19 设计class犹如设计type
### 没啥快速看点的，都要注意
在设计一个class的时候应该足够用心，设计的时候考虑一下这些问题，一般来说这些问题的回答都会引导你的class设计。
+ 创建和销毁怎么实现？是否需要考虑重写operator new等等一系列函数？如果需要重写，又应该怎么重写？
+ 初始化和operator=有什么不同？一个大整数可以两者一样，但是对于一个auto_ptr就不一样了。注意不要混淆这两者。
+ pass-by-value对于这个类的对象意味着什么？和你的初始化函数相配吗？注意pass-by-value需要调用Constructor。
+ 存在合法值吗？比如只有12个月，天数最多31天。如果不合法应该抛出什么异常？
+ 新类型允许哪种转换？是自成一体，允许特定的类型还是只要有特定方法都可以初始化进来？
+ 有需要什么运算符吗？比如operator+。
+ 有什么函数是我不需要的但是已经有了？通过某些方法可以屏蔽掉。
+ 哪些成员（函数或者变量）可以访问？是public，protected，private还是friend？
+ 异常安全能提供哪些保证？
+ 这个类型有多大程度的通用性？如果你设计的和需要的差距太大，你可能需要一个template。
+ 这个类型你需要吗？如果不需要有什么办法可以减少这么多功夫来实现？
## Item 20 宁以pass-by-reference-to-const替换pass-by-value
### 快速看点
+ 用const T&代替T来作为传入参数。
+ 如果你知道你在干什么，建议按照你自己来。
+ STL不是很适用。
### 节约空间与时间
假如我们的大整数类有这样的实现
```C++
class BigInt
{
private:
    vector<int> a;
public:
    ...
    BigInt operator+(BigInt b);
}
```
那么在调用加法的时候，就会发现一个问题：这个b经过构造之后还要析构掉，对应的就要进行一个vector的生成与析构，这是很不必要的。

那么修改成
```C++
    BigInt operator+(const BigInt &b);
```
这样就可以把大多数问题给解决掉了。
### 指针实现-切断问题
假如定义了两个类，Window表示一个窗口，WindowWithScrollBars定义了一个带滚动条的窗口。
```C++
class Window {
public:
    ...
    std::string name() const;
    virtual void display() const;
};
class WindowWithScrollBars: public Window {
public:
    ...
    virtual void display() const;
};
```
如果我们进行
```C++
void printNameAndDisplay(Window w)
{
    std::cout << w.name();
    w.display();
}
```
这就会产生一个问题：传入一个WindowWithScrollBars对象以后，会调用Window的display方法，其virtual就无法体现。
但是修改成：
```C++
void printNameAndDisplay(const Window &w)
```
以后，问题就很神奇的解决掉了。其具体原因在编译器里：引用大多数采用指针实现，换句话说，传一个引用意味着传递一个指针。仔细想想这两个似乎本质上一样。

再仔细想想有没有pass-by-value比较适合的情况？内建类型(int, double之类)，STL类型都是值得pass-by-value的。但是一定要注意不是所有<I>小对象</I>都值得pass-by-value，一个double和一个只有一个double构成的对象就有可能采取不一样的措施，这会涉及到寄存器和内存的访问差异。
## Item 21 必须返回对象时，别妄想返回其reference
### 快速看点
+ 绝不要返回一个局部栈对象的指针或引用。
+ 绝不要返回一个被分配的堆对象的引用。
+ 如果存在需要一个以上这样的对象的可能性时，绝不要返回一个局部static对象的指针或引用。
<span id='Rational'></span>

### 一个例子
```C++
class Rational {
public:
    Rational(int numerator = 0, int denominator = 1);
    ...

private:
    int n, d;

public:
    friend const Rational operator*(const Rational& lhs, const Rational& rhs);
};
```
一个习以为常的例子。但是存在一个问题：为什么operator* 返回的是一个对象而不是一个引用？这样性能不是更好吗？
### const Rational&
如果修改成这样：
```C++
const Rational& operator*(const Rational& lhs, const Rational& rhs)
{
    Rational result(lhs.n * rhs.n, lhs.d * rhs.d);
    return result;
}
```
编译器会返回这一个Warning：reference to stack memory associated with local variable 'result' returned [-Wreturn-stack-address]
### 用指针和new在堆上开内存
```C++
const Rational& operator*(const Rational& lhs, const Rational& rhs)
{
    Rational *result = new Rational(lhs.n * rhs.n, lhs.d * rhs.d);
    return *result;
}
```
这里就有可能有一个问题，虽然实际编写没有遇到，欢迎指正：result没有经过delete，于是有可能发生内存泄漏。

但是这玩意的代码是不是有点长了。。。至少我一开始没想到这么写。
### static
```C++
const Rational& operator*(const Rational& lhs, const Rational& rhs)
{
    static Rational result(lhs.n * rhs.n, lhs.d * rhs.d);
    return result;
}
```
这是真的作死，万一
```C++
(a * b) == (c * d)
```
那永远返回的-1。

多个static？我有点烦，开这么多内存干啥？拿时间换空间？
## Item 22 将成员变量设计为private
### 这不需要多说什么吧
+ protected的封装性就比public强一点，对就一点（不允许外界轻易调用），因为这玩意可以继承以后就随便写了。
+ 把成员变量改成private。
+ 对于调用者而言，如果用上了非private成员变量，那么改动他们的代价就非常巨大。
## Item 23 宁以non-member、non-friend替换member函数
### 快速看点
+ 如果可能的话，用非成员非友元函数取代成员函数，这样可以提高封装性，包装弹性和机能扩充性
### 二选一
假如我们实现了一个web浏览器的类：
```C++
class WebBrowser {
public:
    ...
    void clearCache();
    void clearHistory();
    void removeCookies();
    ...
};
```
如果说这个时候加入一个新的函数用来包装所有这三个（一键清空），请问应该加在哪里呢？

大部分人会实现在类的内部：
```C++
class WebBrowser {
public:
    ...
    void clearEverything()
    {
        ... //把这三个函数都用一遍。
    }
    ...
};
```
然而实际上更好的方法是
```C++
namespace WebBrowserStuff
{
    class WebBrowser{...};
    void clearBroswer(WebBrowser& wb) {...}
}
```
### 为什么？
接下来这一段话值得好好看完。
>结合一个对象考虑数据。越少有代码能看到数据（也就是说，访问它），数据封装性就越强，我们改变对象的数据的特性的自由也就越大，比如，数据成员的数量，它们的类型，等等。作为多少代码能看到一块数据的粗糙的尺度，我们可以计数能访问那块数据的函数的数量：越多函数能访问它，数据的封装性就越弱。
>
>Item 22 说明了数据成员应该是 private 的，因为如果它们不是，就有无限量的函数能访问它们。它们根本就没有封装。对于 private 数据成员，能访问他们的函数的数量就是类的成员函数的数量加上友元函数的数量，因为只有成员和友元能访问 private 成员。假设在一个成员函数（能访问的不只是一个类的 private 数据，还有 private 函数，枚举，typedefs，等等）和一个提供同样功能的非成员非友元函数（不能访问上述那些东西）之间有一个选择，能获得更强封装性的选择是非成员非友元函数，因为它不会增加能访问类的 private 部分的函数的数量。这就解释了为什么 clearBrowser（非成员非友元函数）比 clearEverything（成员函数）更可取：它能为 WebBrowser 获得更强的封装性。
### namespace
在不同的文件中如果用到了namespace重复，会把两者合并掉，换句话说，两边都可以用，除非有完全一样的参数表和返回值的函数。

所以说我们可以做一些这样的事情：在一个文件中定义核心，在其他的文件中定义一系列方便操作的函数，并且分门别类地存放。
## Item 24 若所有参数皆需类型转换，请为此采用non-member函数
### 快速看点
+ operator的第一个参数是被隐藏的，因为默认是这个类型的一个变量进行的操作。
+ operator外置可以保证在面向对象情况下发生一些隐式转换。
### 调用与反过来调用
对于大整数类在内部包装的，应该会遇到一些神奇的问题
```C++
result = oneHalf * 2;
result = 2 * oneHalf;
```
通常来说这两个只有一个能通过编译，虽然实际意义是一样的。这是因为仅仅当参数列在参数列表中的时候才有资格进行隐式类型转换。

如果进行类似于Item 21的操作，把operator*外置，这样lhs和rhs都可以进行non-explicit转化，这样就能通过编译了。

关于这玩意应不应该成为友元的问题，我不是很同意书中的做法，而偏向于写成类内的友元方式。因为你写代码的时候还是要和template结合一下的，如果完全不考虑，最后才发现有问题就等着重构吧。
## Item 25 考虑写出一个不抛异常的swap函数
### 快速看点
+ swap函数可能会有性能问题
+ std认可完全特化的模板，但是不认可新增的模板。
+ STL中的swap通常由public swap和std中的swap特化两个版本共同实现。
### std中的swap函数
```C++
namespace std {
    template<typename T>
    void swap(T& a, T& b)
    {
        T temp(a);
        a = b;
        b = temp;
    }
}
```
从代码可以很明显的看出，只要类型T支持拷贝构造函数和operator=同类，那么swap就可以执行。由于拷贝函数是只要不提起构造函数就会出现的，所以默认情况下swap是被支持的。
### 效率问题及解决思路
但是拷贝函数不一定具有很好的效率，这取决于拷贝的实现，比如说我们定义了一个
```C++
class PQR
{
    int* x;
};
```
现在要进行交换的话就要进行三次拷贝构造和三次指针转移。
```C++
class PQR
{
    vector<int> x;
};
```
这个时候的swap。。。emmm 拷贝三次vector的感觉咋样？
这个时候就需要对于swap的性能进行一些优化：
```C++
namespace std
{
    template<>
    void swap<PQR>(PQR& a,PQR& b)
    {
        swap(a.x, b.x);
    }
};
```
### 解决编译问题
这样子写肯定过不了编译。首先要解决的问题是，x是PQR的private变量，无法从外部访问。这个问题的最好解决办法是在PQR内部实现一个swap。
```C++
class PQR
{
    vector<int> x;
public:
    void swap(PQR& other)
    {
        using std::swap;
        swap(x, other.x);
    }
}
namespace std
{
    template<>
    void swap<PQR>(PQR& a, PQR& b)
    {
        a.swap(b);
    }
};
```
碰巧的是，这种形式和STL中提供的大部分都是一样的。所有的STL容器都提供了public swap函数，并且在std中调用特化函数。
### 加入模板
可是这里就有一个问题，如果类的本身也是一个模板呢？那我改成下面的形式可以吗？
```C++
template<class T>
class PQR
{
    vector<T> x;
public:
    void swap(PQR& other)
    {
        using std::swap;
        swap(x, other.x);
    }
};
namespace std
{
    template<class T>
    void swap<PQR<T>>(PQR<T>& a, PQR<T>& b)
    {
        a.swap(b);
    }
}
```
看起来哪里有问题。。。如果你这样写的话，会得到function template partial specialization is not allowed的错误。
```C++
template<class T>
void swap(PQR<T>& a, PQR<T>& b)
```
即可，相当于这是一个对于swap的重载。
### 第二个问题
>警告：这个问题可能不存在，在llvm7编译环境下，这段代码能通过编译并且正确输出结果：
```C++
#include <bits/stdc++.h>
using namespace std;
template <class T>
class WidgetImpl
{
public:
    WidgetImpl(int _a = 0, int _b = 0) : a(_a), b(_b), v() {}
    WidgetImpl(const WidgetImpl &rhs) : a(rhs.a), b(rhs.b), v(rhs.v) {}
    WidgetImpl &operator=(const WidgetImpl &rhs)
    {
        a = rhs.a;
        b = rhs.b;
        v = rhs.v;
    }
    void print()
    {
        cout << a << " " << b << endl;
    }

private:
    int a, b;
    std::vector<double> v;
};

template <class T>
class Widget
{
public:
    Widget(int _a = 0, int _b = 0)
    {
        pImpl = new WidgetImpl<T>(_a, _b);
    }
    Widget(const Widget &rhs) { *pImpl = *(rhs.pImpl); }
    Widget &operator=(const Widget &rhs)
    {
        *pImpl = *(rhs.pImpl);
        return *this;
    }
    void swap(Widget &other)
    {
        using std::swap;
        swap(pImpl, other.pImpl);
    }
    void print() { pImpl->print(); }

private:
    WidgetImpl<T> *pImpl;
};
namespace std
{
template <class T>
void swap(Widget<T> &a, Widget<T> &b)
{
    a.swap(b);
}
}; // namespace std
int main()
{
    Widget<int> a(1, 2), b(4, 5);
    swap(a, b);
    a.print();
    return 0;
}
```
但是这段代码也有可能不能通过编译，因为std允许完全特化一个模板，但是不允许新增一个模板。那怎么办呢？写在自己的namespace里面即可。
### 多个namespace引发的问题
假如我们写了两个namespace，然后发现需要在外部调用swap，而且还是模板类型，请问会调用哪一个呢？你不知道，也有可能因为特化没有而选择了不期望的std特化。于是乎怎么办？
```C++
template<typename T>
void doSomething(T& obj1, T& obj2)
{
    using std::swap;
    ...
    swap(obj1, obj2);
    ...
}
```
用using指定不就是了？总之肯定会优先调用这一个的
### 等等 异常安全呢？
见Item 29。（这关子卖的）
## 扩展：C++的函数查找方式（以后更新）