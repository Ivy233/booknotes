<style>
p{
    text-indent:2em;
}
</style>
# 实现
## Item 26 尽可能延后变量定义式的出现时间
----
### 快速看点
+ 只要有可能就推迟变量的定义时间，这可以增加程序的可读性并且提升程序的性能（如果逻辑比较复杂的话尤为显著）
### 一个简单的例子
```C++
std::string encryptPassword(const std::string& password)
{
    using namespace std;
    string encrypted;
    if (password.length() < MinimumPasswordLength) {
        throw logic_error("Password is too short");
    }
    ... // do whatever is necessary to place an
        // encrypted version of password in encrypted
    return encrypted;
}
```
在这个例子中，我们使用了std的namespace，并且尝试加密一个东西，但是如果在if处发生了一个异常，那么encrypted就白定义了，而且这只是一个函数，如果设计的比较臭长的话，有可能其他人根本不知道encrypted怎么定义的，卒。

一个更好的建议是把encrypted丢到正好需要之前。具体代码就不写了。

甚至你还可以这么写：
```C++
void encrypt(std::string& s); // encrypts s in place
std::string encryptPassword(const std::string& password)
{
    ...                         // check length
    string encrypted(password); // define and initialize via copy constructor
    encrypt(encrypted);
    return encrypted;
}
```
首先用初始化函数跳过空白初始化，然后直接将自己传入进行encrypt，得到的结果就是需要的结果。虽然这样写可能有一个问题：encrypt的传入是一个s，肯定存在加密后面的需要前面的原始内容，这样还是要在字符串内复制一次，时间没有显著地减少。（不过可读性确实增加了）
### 循环？
来看两个版本的代码：
```C++
//1
Widget w;
for (int i = 0; i < n; ++i){
    w = some value dependent on i;
    ...
}
//2
for (int i = 0; i < n; ++i) {
    Widget w(some value dependent on i);
    ...
}
```
说白了就是两种：
一个是一个构造函数+一个析构+n个赋值。
一个是n个析构函数+n个构造函数。
具体情况具体处理，比如vector进行加速的时候还是要采用第一种比较舒服。
## Item 27 尽量少做转型动作
---
### 快速看点
+ (T)expression是C-style的转型。
+ T(expression)是function-style的转型。
+ const_case<T>(expression)能强制消除对象的const属性。
+ dynamic_cast<T>(expression)能进行类继承的向下转型，如果不是一个继承体系的就返回错误。
+ reinterpret_cast<T>(expression)用于底层的强制转型，导致实现依赖的结果，例如，将一个指针转型为一个整数。
+ static_cast<T>(expression)可以被用于强制隐形转换，non-const转型成const（注意不能反过来）。
+ 如果想在derived class调用基类的函数，请直接指定这个函数即可，不需要强制类型转换。
### 跟着感觉走就行，没必要一定要用C++的
比如说类对象的初始化：
```C++
Widget(15);
static_cast<Widget>(15);
```
### 有的转换需要时间，有的转换只是一个位移
```C++
double d = static_cast<double>(x)/y;
```
因为int和double的底层实现不一样，所以转换需要时间来修改表现形式。很自然的，int转float也不一样。

有的转换只是一个位移，比如base class和derived class的转换。
```C++
Derived d;
Base *pb = &d;
```
### 有的时候强制类型转换是危险的
```C++
class Window {                      // base class
public:
    virtual void onResize() { ... } // base onResize impl
    ...
};

class SpecialWindow: public Window {// derived class
public:
    virtual void onResize() {       // derived onResize impl;
        // cast *this to Window,
        static_cast<Window>(*this).onResize();
                                // then call its onResize;
                                // this doesn't work!
        ...                     // do SpecialWindow-specific stuff
    }
    ...
};
```
这样的代码其实非常错误，因为没有考虑到一个事情：
+ 我正在用这个对象的临时的基类形式调用这个函数。
+ 这个函数可能修改了这个临时对象(non-const)，也有可能是针对修改产生的反应(js的同名函数)。
+ 在修改完之后（onResize()调用完成），这个临时对象就会消失。

换句话说，你啥都没做！

这样的解决办法其实非常简单，甚至比源代码还要简单，只是你有可能不知道而已：
```C++
class SpecialWindow: public Window {
public:
    virtual void onResize() {
        Window::onResize();
        ...
    }
    ...
};
```
直接指定不就是了，为啥要这么麻烦呢？
### 减少dynamic_cast的出现（需要补全）
+ 对指针进行dynamic_cast，失败返回null，成功返回正常cast后的对象指针；
+ 对引用进行dynamic_cast，失败抛出一个异常，成功返回正常cast后的对象引用。
+ 在类层次间进行上行转换时，dynamic_cast和static_cast的效果是一样的；
+ 在进行下行转换时，dynamic_cast具有类型检查的功能，比static_cast更安全。

dynamic_cast的很多实现相当慢。如果你在一个四层深的但继承体系中的对象上执行dynamic_cast，那么就会调用四次strcmp来比较类名字。但是这样做可以支持动态链接，所以不能再编译期完成。为了规避这样的性能浪费，通常会采取其他的方式来避免dynamic_cast的使用。

特别要注意的是避免所谓级联的dynamic_cast的设计。
## Item 28 避免返回handles指向对象内部成分
---
### 快速看点
+ 返回对象内部构件的句柄是相当危险的，也有可能让const成员函数的const效果消失。
### 这似乎在const那里提到过
我们假设有这样的长方形：
```C++
class Point
{
private:
    int x, y;
public:
    Point(int _x, int _y);
    void setX(int newX);
    void setY(int newY);
};
struct RectData
{
    Point ulhc;
    Point lrhc;
};
class Rectancle
{
public:
    Point& upperleft() const {return pData->ulhc;}
    Point& lowerright() const {return pData->lrhc;}
private:
    std::tr1::shared_ptr<RectData> pData;
};
```
即使我们定义了const的Rectangle，也可以因为upperleft或者lowerright来修改长方形的内部成员。实际上，假如const就可以解决这个问题：
```C++
class Rectancle
{
public:
    const Point& upperleft() const {return pData->ulhc;}
    const Point& lowerright() const {return pData->lrhc;}
private:
    std::tr1::shared_ptr<RectData> pData;
};
```
但是这有可能产生其他的问题，比如空悬句柄：引用了不再存在的对象的构件的句柄。这种小事的对象的最普通的来源就是函数返回值。

这就是为什么任何返回一个对象的内部构件的句柄的函数都是危险的。它与那个句柄是指针，引用，还是迭代器没什么关系。它与是否受到const的限制没什么关系。它与那个成员函数返回的句柄本身是否是const没什么关系。全部的问题在于一个句柄被返回了，因为一旦这样做了，你就面临着这个句柄比它引用的对象更长寿的风险。

但是这并不意味着永远不应该返回一个句柄，比如STL中的operator[]
## Item 29 为“异常安全”而努力是值得的
---
### 快速看点
## Item 30 彻底了解inlining的里里外外
---
### 快速看点
## Item 31 将文件间的编译依存关系降至最低
---
### 快速看点