# 实现
## Item 26 尽可能延后变量定义式的出现时间
----
### 快速看点
+ 只要有可能就推迟变量的定义时间，这可以增加程序的可读性并且提升程序的性能（如果逻辑比较复杂的话尤为显著）
### 一个简单的例子
```C++
string encryptPassword(const string& password)
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
在这个例子中，我们使用了`std`的`namespace`，并且尝试加密一个东西，但是如果在`if`处发生了一个异常，那么encrypted就白定义了，而且这只是一个函数，如果设计的比较臭长的话，有可能其他人根本不知道encrypted怎么定义的，卒。

一个更好的建议是把encrypted丢到正好需要之前。具体代码就不写了。

甚至你还可以这么写：
```C++
void encrypt(string& s); // encrypts s in place
string encryptPassword(const string& password)
{
    ...                         // check length
    string encrypted(password); // define and initialize via copy constructor
    encrypt(encrypted);
    return encrypted;
}
```
首先用初始化函数跳过空白初始化，然后直接将自己传入进行encrypt，得到的结果就是需要的结果。虽然这样写可能有一个问题：encrypt的传入是一个`s`，肯定存在加密后面的需要前面的原始内容，这样还是要在字符串内复制一次，时间没有显著地减少。（不过可读性确实增加了）
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
具体情况具体处理，比如`vector`进行加速的时候还是要采用第一种比较舒服。
## Item 27 尽量少做转型动作
---
### 快速看点
+ `(T)expression`是`C-style`的转型。
+ `T(expression)`是`function-style`的转型。
+ `const_case<T>(expression)`能强制消除对象的`const`属性。
+ `dynamic_cast<T>(expression)`能进行类继承的向下转型，如果不是一个继承体系的就返回错误。
+ `reinterpret_cast<T>(expression)`用于底层的强制转型，导致实现依赖的结果，例如，将一个指针转型为一个整数。
+ `static_cast<T>(expression)`可以被用于强制隐形转换，`non-const`转型成`const`（注意不能反过来）。
+ 如果想在`derived class`调用基类的函数，请直接指定这个函数即可，不需要强制类型转换。
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
因为`int`和`double`的底层实现不一样，所以转换需要时间来修改表现形式。很自然的，`int`转`float`也不一样。

有的转换只是一个位移，比如`base class`和`derived class`的转换。
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
+ 这个函数可能修改了这个临时对象(`non-const`)，也有可能是针对修改产生的反应(js的同名函数)。
+ 在修改完之后(`onResize()`调用完成)，这个临时对象就会消失。

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
### 减少`dynamic_cast`的出现（需要补全）
+ 对指针进行`dynamic_cast`，失败返回`null`，成功返回正常cast后的对象指针；
+ 对引用进行`dynamic_cast`，失败抛出一个异常，成功返回正常cast后的对象引用。
+ 在类层次间进行上行转换时，`dynamic_cast`和`static_cast`的效果是一样的；
+ 在进行下行转换时，`dynamic_cast`具有类型检查的功能，比`static_cast`更安全。

`dynamic_cast`的很多实现相当慢。如果你在一个四层深的但继承体系中的对象上执行`dynamic_cast`，那么就会调用四次strcmp来比较类名字。但是这样做可以支持动态链接，所以不能再编译期完成。为了规避这样的性能浪费，通常会采取其他的方式来避免`dynamic_cast`的使用。

特别要注意的是避免所谓级联的`dynamic_cast`的设计。
## Item 28 避免返回handles指向对象内部成分
---
### 快速看点
+ 返回对象内部构件的句柄是相当危险的，也有可能让`const`成员函数的`const`效果消失。
### 这似乎在`const`那里[提到过](./Chapter1.md#Point)
有了RAII对象之后，我们可以做出如下修改：
```C++
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
    tr1::shared_ptr<RectData> pData;
};
```
即使我们定义了`const`的Rectangle，也可以因为`upperleft`或者`lowerright`来修改长方形的内部成员。实际上，假如`const`就可以解决这个问题：
```C++
class Rectancle
{
public:
    const Point& upperleft() const {return pData->ulhc;}
    const Point& lowerright() const {return pData->lrhc;}
private:
    tr1::shared_ptr<RectData> pData;
};
```
但是这有可能产生其他的问题，比如空悬句柄：引用了不再存在的对象的构件的句柄。这种小事的对象的最普通的来源就是函数返回值。

这就是为什么任何返回一个对象的内部构件的句柄的函数都是危险的。它与那个句柄是指针，引用，还是迭代器没什么关系。它与是否受到`const`的限制没什么关系。它与那个成员函数返回的句柄本身是否是`const`没什么关系。全部的问题在于一个句柄被返回了，因为一旦这样做了，你就面临着这个句柄比它引用的对象更长寿的风险。

但是这并不意味着永远不应该返回一个句柄，比如STL中的`operator[]`。
## Item 29 为“异常安全”而努力是值得的
---
### 快速看点
+ 写程序的核心是保护好所有你的资源。
+ 假设每一行比较复杂的操作都可以产生异常。
+ 可以通过`copy-and-swap`进行异常的强力保证。
+ 一个函数的保证不会强于这个函数调用的所有函数中最弱的保证。
### 异常安全
假设我们有一个类，代表带有背景图像的GUI菜单。这个类被设计成在多线程环境中使用，所以它有一个用于并行控制的互斥体：
```C++
class PrettyMenu {
public:
    ...
    void changeBackground(istream& imgSrc);
    ...

private:
    Mutex mutex;                    // mutex for this object
    Image *bgImage;                 // current background image
    int imageChanges;               // # of times image has been changed
};
```
考虑这个`PrettyMenu`的`changeBackground`函数的可能的实现：
```C++
void PrettyMenu::changeBackground(istream& imgSrc)
{
    lock(&mutex);                      // acquire mutex (as in Item 14)

    delete bgImage;                    // get rid of old background
    ++imageChanges;                    // update image change count
    bgImage = new Image(imgSrc);       // install new background

    unlock(&mutex);                    // release mutex
}
```
逻辑上似乎没什么问题，但是一旦发生异常就没这么简单了，就从`new`发生异常开始
+ 如果内存分配或者初始化发生异常，那么新的出不来，老的还删除了。除此以外，`imageChanges`还自增了。
+ 如果内存分配发生异常，那么东西就被锁上了，`unlock`没有运行。

规避资源泄露问题比较容易，因为[Item 13](./Chapter3.md#Item_13)解释了如何使用对象管理资源，而[Item 14](./Chapter3.md#Item_14)又引进了`Lock`类作为一种时尚的确保互斥体被释放的方法：
```C++
void PrettyMenu::changeBackground(istream& imgSrc)
{
  Lock ml(&mutex);                 // from Item 14: acquire mutex and
                                   // ensure its later release
  delete bgImage;
  ++imageChanges;
  bgImage = new Image(imgSrc);
}
```
引入资源管理类有一个好处：我们可以设置在析构的时候需要做什么，比如在这里可以设置成释放`lock`。这里解决了第二个问题，但是第一个没有解决。在这里我们有一个选择，但是在我们能选择之前，我们必须先面对定义我们的选择的术语。
### 异常安全函数提供的保证
异常安全函数提供下述三种保证之一，如果都不提供，那么就不是异常安全的。
+ 基本保证，允诺如果一个异常被抛出，程序中剩下的每一件东西都处于合法状态。没有对象或数据结构被破坏，而且所有的对象都处于内部调和状态（所有的类不变量都被满足），但是不保证程序状态是期望的。
+ 强力保证，允诺如果一个异常被抛出，程序的状态不会发生变化。调用这样的函数在感觉上是极其微弱的，如果它们成功了，它们就完全成功，如果它们失败了，程序的状态就像它们从没有被调用过一样。
+ 不抛出保证，允诺决不抛出异常，因为它们只做它们答应要做的。所有对内建类型（例如，ints，指针，等等）的操作都是不抛出(nothrow)的（也就是说，提供不抛出保证）。这是异常安全代码中必不可少的基础构件。

理论上我们应该尽力给一个更好的保证。但是如果不能达到更好的保证，也不要勉强。
### 解决问题的第一种方法
首先使用资源管理类对象来管理资源。然后保证只有在图像发生变化之后才修改变化次数。这保证了一个策略：仅仅是一个记录，而不是一个要求。

这就是修改之后的代码：
```C++
class PrettyMenu {
    ...
    tr1::shared_ptr<Image> bgImage;
    ...
};

void PrettyMenu::changeBackground(istream& imgSrc)
{
    Lock ml(&mutex);
    bgImage.reset(new Image(imgSrc));
    ++imageChanges;
}
```
`reset`可以保证手动删除旧的图像，因为没有东西指它就会自动析构掉。而且调用顺序显然是：
+ `new`开辟内存
+ `Image(src)`写入
+ `reset`
+ `delete`原来的内存（如果需要删除的话）

如果`new`不出来，就啥都没发生，也不会`delete`。而且修改指针的实际操作也非常简单，我们假设它也不会产生异常。最后是初始化写入失败，`istream`是不可回退的指针（注意参数表），所以会产生一些变化，所以整个函数只能提供基本异常安全保证。而如果参数表不是`istream`，那么可以认为是强异常安全保证。（但是为啥改图片的参数是`istream`。。。）
### 第二种方法
<span id='copy_and_swap'></span>

这种方法通常被称为[`copy-and-swap`](./Chapter2.md#copy_and_swap)，具体含义是先创建一个新变量的拷贝，然后在拷贝上做需要的全部操作，最后用不抛出异常的`swap`交换两者。

这种方法高明在哪里呢？如果出现异常，那么最初的对象保持不变，而如果前面没有异常，又由于`swap`没有异常，所以整个程序能正常运行。

这么牛皮的东西怎么进行实操呢？再包装一次。将需要改变的用一个类+资源管理的方式进行管理。这样三步都很容易实现：第一步`new`一个，第二步`new`出来一个`Image`之后改动掉，第三个就是交换指针嘛。
```C++
struct PMImpl {
    tr1::shared_ptr<Image> bgImage;
    int imageChanges;
};

class PrettyMenu {
    ...
private:
    Mutex mutex;
    tr1::shared_ptr<PMImpl> pImpl;
};

void PrettyMenu::changeBackground(istream& imgSrc)
{
    using swap;
    Lock ml(&mutex);
    tr1::shared_ptr<PMImpl> pNew(new PMImpl(*pImpl));
    pNew->bgImage.reset(new Image(imgSrc));
    ++pNew->imageChanges;
    swap(pImpl, pNew);
}
```
虽然这个方法没有解决第一个方法的问题，但是也提供了一个比较好的思路：更多的空间总是能够完成的比较好。
### 多个函数的异常安全
如果调用了一个函数，这个函数能够提供强力异常安全保证，但是并不能证明这个函数本身也是强力异常安全的。如果调用了两个函数，那这种保证会更加难以提供。因为你总是不能保证再第二个调用中发生的异常把第一个带来的影响完全抹去。

>四十年前，到处都是 goto 的代码被尊为最佳实践。现在我们为书写结构化控制流程而奋斗。二十年前，全局可访问数据被尊为最佳实践。现在我们为封装数据而奋斗，十年以前，写函数时不必考虑异常的影响被尊为最佳实践。现在我们为写异常安全的代码而奋斗。
>
>时光在流逝。我们生活着。我们学习着。

<span id='Item_30'></span>

## Item 30 彻底了解inlining的里里外外
---
### 快速看点
+ `inline`的原理是把每一个用到这个函数的地方用这个函数的函数体代替，而不是一个调用。
+ `inline`函数可以加速一些函数，代价是函数体积的膨胀，所以建议在比较短小的函数上面使用。
+ `inline`只是给编译器的优化建议，是否选择优化则由编译器自己决定。如果是成员函数，自动声明为`inline`。
### `inline`函数的基本内容
在典型情况下，编译器的优化是为了一段连续的没有函数调用的代码设计的，所以当你`inline`化一个函数，你可能就使得编译器能够对函数体实行上下文相关的特殊优化。然而，过分热衷于`inline`使得程序过分膨胀。即使使用了虚拟内存，`inline`引起的代码膨胀也会导致附加的分页调度，减少指令缓存命中率，以及随之而来的性能损失。在另一方面，如果一个`inline`函数本体很短，为函数本体生成的代码可能比为一个函数调用生成的代码还要小。如果是这种情况，`inline`化这个函数可以实际上导致更小的目标代码和更高的指令缓存命中率！

一定要记住，`inline`仅仅是一个请求，编译期可以选择拒绝（编译期：我太难了）。这个请求能够以显式的或隐式的方式提出，显式的方法显然是`inline`一个函数：
```C++
template<typename T>
inline const T& max(const T& a, const T& b)
{ return a < b ? b : a; }
```
这个函数来自于`std`的max函数。而隐式的方法就是在一个类定义的内部定义一个函数：
```C+++
class Person {
public:
    int age() const { return theAge; }

private:
    int theAge;
};
```
### `inline`函数的编译
在编译的时候同样需要知道这个函数是否需要用本体来替换掉用，因此大多数环境在编译期间进行`inline`。模板同样也在头文件内，因为编译期至少要知道这玩意看起来像什么东西才能进行实例化。但是这两个看起来没啥其他的关系，我也可以把这玩意的`inline`去掉，这都是实现的自由。只不过编译器在翻译的时候有可能和之前有点差异。

但是一定要注意一些事情：
+ `inline`可以引发代码膨胀，模板同样也可以，虽然原理不同，但是效果都十分显著。
+ 如果一个加入了`inline`的函数不能被编译器`inline`，那么你会收到一个Warning（我怎么没有，是不是我没用过）。
+ virtual函数除了最简单的都不会`inline`，因为编译器不知道动态类型怎么调用。
+ 就算`inline`成功，也有可能还是生成了函数本体，因为有些函数会被指。比如析构函数和构造函数，或者：
```C++
inline void f() {...}      // assume compilers are willing to inline calls to f
void (*pf)() = f;          // pf points to f
f();    // this call will be inlined, because it's a "normal" call
pf();   // this call probably won't be, because it's through a function pointer
```
+ 不要尝试对于`inline`函数进行debug，除非你能确保这玩意还有地址可以找到。
### 类中的`inline`
事实上，构造函数和析构函数对于 `inline` 化来说经常是一个比你在不经意的检查中所能显示出来的更加糟糕的候选者。例如，考虑下面这个类 Derived 的构造函数：
```C++
class Base {
public:
    ...
private:
   string bm1, bm2;
};
class Derived: public Base {
public:
    Derived() {}
    ...

private:
    string dm1, dm2, dm3;
};
```
有可能你觉得`inline`会在Derived中生效：看，我里面一行代码都没写。可是真的没写吗？
```C++
Derived::Derived()                       // conceptual implementation of
{                                        // "empty" Derived ctor
    Base::Base();                           // initialize Base part
    try { dm1.string::string(); }      // try to construct dm1
        catch (...) {                           // if it throws,
        Base::~Base();                        // destroy base class part and
        throw;                                // propagate the exception
    }
    try { dm2.string::string(); }      // try to construct dm2
    catch(...) {                            // if it throws,
        dm1.string::~string();           // destroy dm1,
        Base::~Base();                        // destroy base class part, and
        throw;                                // propagate the exception
    }
    try { dm3.string::string(); }      // construct dm3
    catch(...) {                            // if it throws,
        dm2.string::~string();           // destroy dm2,
        dm1.string::~string();           // destroy dm1,
        Base::~Base();                        // destroy base class part, and
        throw;                                // propagate the exception
    }
}
```
现在你还说一行代码都没有吗？这些代码都是编译器帮你写了。事实上，编译期采用的方法会更加复杂，因为他也不知道会发生什么神奇的情况，智能做更多的事情来尽可能覆盖。

这个函数中有很多对象，每个对象产生的时候要进行初始化，而产生失败的时候需要把前面的都析构掉。每一段代码都会影响`inline`的结果。甚至Base构造函数也是如此，如果Base进行构造，势必会有大量的代码加入进来，一共五个对象需要处理。天哪！这到底是有多少函数要进行`inline`，这个代码膨胀实在太恐怖了。

讲解到这个程度，请问你还会觉得看起来很简单的函数会进行`inline`吗？
### `inline`对发布的影响
如果`inline`函数发布以后需要进行修改，那么会发生一些不得不处理的情况：所有用到的代码都需要重新编译。如果项目非常巨大的话，就是很难受的结果。而相比于没有`inline`的只需要修改一下调用就行了。
## Item 31 将文件间的编译依存关系降至最低
---
### 快速看点
+ 最小化编译依赖的一个想法是用声明来替换定义。基于此想法的两个方法是`Handle`类和`Interface`类。
+ 库头文件应该以完整并且只有声明的方式存在。
### 一个实际情况
你进入到你的程序中，并对一个类的实现进行了细微的改变。提醒你一下，不是类的接口，只是实现，仅仅是 `private` 的东西。然后你重建(rebuild)这个程序，预计这个任务应该只花费几秒钟。毕竟只有一个类被改变。你在 Build 上点击或者键入 make（或者其它等价行为），接着你被惊呆了，继而被郁闷，就像你突然意识到整个世界都被重新编译和连接！当这样的事情发生的时候，你不讨厌它吗？（项目太大的苦衷）
### 编译依赖是如何产生的
我们先来看看这个类：
```C++
class Person {
public:
    Person(const string& name, const Date& birthday,
            const Address& addr);
    string name() const;
    string birthDate() const;
    string address() const;
    ...

private:
    string theName;        // implementation detail
    Date theBirthDate;          // implementation detail
    Address theAddress;         // implementation detail
};
```
涉及到了三个东西：`string`，`Date`，`Address`，对应的应该有三个头文件（至少应该满足有这些类吧）。但是会有一个问题：头文件发生变化以后，用到这个头文件的所有玩意都要重新编译。项目小还好，大了就要掀桌子了。
```C++
#include <string>
#include "date.h"
#include "address.h"
```
### 尝试将类的实现分离出来
你也许想知道 C++ 为什么坚持要将一个类的实现细节放在类定义中。例如，你为什么不能这样定义 `Person`，单独指定这个类的实现细节呢？
```C++
namespace std {
    class string;             // forward declaration (an incorrect
}                              // one — see below)

class Date;                    // forward declaration
class Address;                 // forward declaration

class Person {
public:
    Person(const string& name, const Date& birthday,
                const Address& addr);
    string name() const;
    string birthDate() const;
    string address() const;
    ...
};
```
别想了，这玩意过不了编译的。
+ `string`不是一个类，而是一个`basic_string<char>`，甚至要更加复杂的前置声明才能够合理地使用`string`。
+ 然而，这还不是要紧的，因为你不应该试着手动声明标准库的部件。作为替代，直接使用适当的`#include`s并让它去做。标准头文件不太可能成为编译的瓶颈，特别是在你的构建环境允许你利用预编译头文件时。如果解析标准头文件真的成为一个问题，你也许需要改变你的接口设计，避免使用导致不受欢迎的标准库部件。
+ 第二个难点是声明的每一个东西都要知道有多大，不然这个类咋定义。
### 使用指针替代对象
再来考虑一个东西：
```C++
int main()
{
    int x;                // define an int
    Person p(params);   // define a Person
    ...
}
```
再来看看这段代码：
```C++
int main()
{
    int x;               // define an int
    Person *p;           // define a pointer to a Person
    ...
}
```
是不是明朗多了？指针大小永远是固定的，但是类却不一定。Java大法好!

当然，也可以用在类中。一个比较好的方式是定义类中就一个指针，然后具体内容留到`Impl`里面。
```C++
#include <string>
#include <memory>                      // for tr1::shared_ptr; see below
class PersonImpl;                      // forward decl of Person impl. class
class Date;                            // forward decls of classes used in
class Address;                         // Person interface
class Person {
public:
    Person(const string& name, const Date& birthday,
            const Address& addr);
    string name() const;
    string birthDate() const;
    string address() const;
    ...

private:                                   // ptr to implementation;
    tr1::shared_ptr<PersonImpl> pImpl;  // see Item 13 for info on
};                                         // tr1::shared_ptr
```
这样一个设计经常被说成是使用了`pimpl`惯用法（指向实现的指针"pointer to implementation"），所以那个变量名交作`pImpl`。这种情况下就没有必要改一个东西就把所有东西都改了，顺便完成了[Item18](./Chapter4.md#Item_18)，因为只要包装起来，就很难发生依赖代码细节的代码了。
### 设计策略
这个分离的关键就是用对声明的依赖替代对定义的依赖。这就是最小化编译依赖的精髓：只要能实现，就让你的头文件独立自足，如果不能，就依赖其它文件中的声明，而不是定义。其它每一件事都从这个简单的设计策略产生。所以：
+ 当对象的引用和指针可以做到时就避免使用对象。
1. 仅需一个类型的声明，你就可以定义到这个类型的引用或指针。
2. 而定义一个类型的对象必须要存在这个类型的定义。
+ 只要你能做到，就用对类声明的依赖替代对类定义的依赖。

注意你声明一个使用一个类的函数时绝对不需要有这个类的定义，即使这个函数通过传值方式传递或返回这个类：
```C++
class Date;                        // class declaration
Date today();                      // fine — no definition
void clearAppointments(Date d);    // of Date is needed
```
### 声明+定义
我们可以考虑给每一个类单独提供声明，和定义文件，之后只需要依赖于声明就行了。为什么不总是提供定义？因为不是所有人都需要，你总归不能需要`string`的时候把`vector`也一路收拾了吧（#滑稽）。

```C++
#include "datefwd.h" // header file declaring (but not defining) class Date
Date today();        // as before
void clearAppointments(Date d);
```
这一套骚操作的模板在标准库就有<iosfwd>。其定义在很多个组件中。

除此以外，C++ 还提供了`export`允许将模板声明从模板定义中分离出来。不幸的是，支持`export`的编译器非常少，而与`export`打交道的实际经验就更少了。结果是，现在就说`export`在高效C++编程中扮演什么角色还为时尚早。
### 句柄类
像`Person`这样的使用`pimpl`惯用法的类经常被称为`Handle`类。为了避免你对这样的类实际上做什么事的好奇心，一种方法是将所有对他们的函数调用都转送给相应的实现类，而使用实现类来做真正的工作。例如，这就是两个`Person`的成员函数可以被如何实现的例子：
```C++
#include "Person.h"
#include "PersonImpl.h"
Person::Person(const string& name, const Date& birthday, const Address& addr)
    : pImpl(new PersonImpl(name, birthday, addr)) {}
string Person::name() const { return pImpl->name(); }
```
使`Person`成为一个`Handle`类不需要改变`Person`要做的事情，仅仅是改变了它做事的方法。
### 接口类
>@java的inferface

如果你熟悉java的话，肯定对于interface有印象。interface就是只有纯虚函数的类。就像：
```C++
class Person {
public:
    virtual ~Person();

    virtual string name() const = 0;
    virtual string birthDate() const = 0;
    virtual string address() const = 0;
    ...
};
```
和java不太一样的是，这玩意不需要继承就可以创建"实例"——实例的指针就能调用所有的东西。一开始对于这一点可能无法理解，但是根据之前的讲述还是能体会到C++有多么可怕的灵活性的。
+ `virtual`函数可以在类定义中`=0`，但是再实现中给出实现，即使是在这种情况下，继承对象也是不可能的。
+ 工厂模式返回指针。
```C++
#include <bits/stdc++.h>
using namespace std;
class RealPerson;
class Person
{
public:
    virtual ~Person(){};
    virtual string name() const = 0;
    static shared_ptr<Person> create(const string &name);
};

class RealPerson : public Person
{
public:
    RealPerson(const string &name) : theName(name) {}
    virtual string name() const { return theName; }
    virtual ~RealPerson() {}

private:
    string theName;
};
shared_ptr<Person> Person::create(const string &name)
{
    shared_ptr<Person> tmp(new RealPerson(name));
    return tmp;
}
int main()
{
    shared_ptr<Person> p = Person::create("123");
    cout << p->name();
    return 0;
}
```
`Person::create`的一个更现实的实现会创建不同派生类型的对象，依赖于诸如，其他函数的参数值，从文件或数据库读出的数据，环境变量等等。

`RealPerson`示范了两个最通用的实现一个Interface类机制之一：从Interface类(`Person`)继承它的接口规格，然后实现接口中的函数。实现一个Interface类的第二个方法包含多继承(multiple inheritance)，在[Item 40](./Chapter6.md#Item_40)中探讨这个话题。
### 代价
当然这样子做显然是有代价的：加一层包装对于编程而言不仅仅是增加了代码复杂度，而且增加了编译的复杂度，甚至会多分配一些空间才能完成相同的事情。但是为了彻底让接口与实现分离，这样子做是可以接受的。这复杂度真大。真香。

`Handle`类的代价：
+ 通过指针获得数据，这是一个间接层。
+ 动态内存分配的成本。
+ `bad_alloc`的可能性。

`Interface`类的代价：
+ 函数跳转的成本。
+ 虚函数的内存膨胀。

两者共同的问题：
+ `inline`难度大，这东西实在有点复杂，一个是各种虚函数+动态类型，一个是指针跳转。都在隐藏函数。