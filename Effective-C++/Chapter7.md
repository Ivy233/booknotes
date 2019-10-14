# 模板与泛型编程
## Item 41 了解隐式接口与编译期多态
### 快速看点
+ 编译期多态和运行期多态是不一样的。虽然感觉上运行期多态做的事情在编译的时候就能够完成。
+ 隐式接口对于函数的调试会产生一定的麻烦，因为你不知道调用的是什么类型的玩意，而只有在编译的时候才能知道你写的到底对不对。
### 模板与面向对象的区别
我们先来看一个例子，这是一个比较简单的面向对象的类：
```C++
class Widget {
public:
    Widget();
    virtual ~Widget();
    virtual std::size_t size() const;
    virtual void normalize();
    void swap(Widget& other);                 // see Item 25
};
```
对应的，这是一个针对`Widget`进行调用的简单函数：
```C++
void doProcessing(Widget& w)
{
    if (w.size() > 10 && w != someNastyWidget) {
        Widget temp(w);
        temp.normalize();
        temp.swap(w);
    }
}
```
不知道你有没有想过为什么面向对象的`size()`能被这么清晰的解读出来，或者`!=`这个符号也能这么清晰的解读出来？因为他就在代码里面啊！编译期只需要用正确的打开方式就能找到我调用的东西，从而检查我的调用情况。
同样的，由于有很多`virtual`函数，因此需要进行很多运行时多态来确定要执行哪一个类的函数。

但是如果在换成模板之后，事情就都发生改变了：
```C++
template<typename T>
void doProcessing(T& w)
{
    if (w.size() > 10 && w != someNastyWidget) {
        T temp(w);
        temp.normalize();
        temp.swap(w);
    }
}
```
因为T的类型不确定，所以`size()`到底是什么东西在外面的lint是不知道的。当然有的比较牛皮一点可以知道简单的，但是根据我复刻STL的经历，总是F12不了的痛苦意味着对于稍微复杂一点的模板而言lint也无能为力。这就是编译期多态，只有在编译的时候才能真正确定到底调用的是什么函数，并且正确的展开函数进入内存以供调用。

这就是编译期多态，而由于不知道具体类型的对于接口的要求称之为隐式接口。
## Item 42 了解`typename`的双重含义
### 快速看点
+ `typename`由两种用法，第一种用法和`class`在`template<class T>`中是类似的，两者可以互换。
+ `typename`的另外一种用法可以强制认为一种不知道什么类型的东西是一种类型，虽然到底是什么类型依旧不清楚，但是至少可以嵌套依赖类型名的报错。
### 用法一
在了解过之前的内容之后，你可能会认为，那我简单点写`class`，就少了几个字母？实际上，这两者的区别更倾向于对于类模板初始化的倾向：如果对于所有类型都有效，采用`typename`，暗示不必要是一个`class`。
### 用法二 声明类型
我们来看一下这个函数，它有一个不是很明显的问题。
<span id='const_iterator'></span>

```C++
template<typename C>
void print2nd(const C& container){
    if(container.size() >= 2){
        C::const_iterator it(container.begin());
        ++it;
        int value = *it;
        cout<<value;
    }
}
```
你可能会问哪里有问题，建议你看完以后思考一下再给出答案。

问题出在代码的第4行，`C::const_iterator`并不是如你所想一样会直接产生一个类型，而是选择了报错。

发生这个奇怪的问题的原因就是编译期并不知道这是一个类型，虽然字面意义上来看这肯定不是一个类型，但是不能总是保证说没有一个代码狂人写了一个交作`const_iterator`的`static`变量或者`const`变量丢在里面。编译器不知道这玩意是不是一个类型，所以只好假定这玩意不是，除非你说这玩意是。这种情况被称之为"嵌套依赖类型名"

所以为了让编译器认为这是一个类型，我们选择给这玩意加一个`typename`：
```C++
typename C::const_iterator it(container.begin());
```
这个时候聚焦到代码的第2行，你又会发现一个问题。
### 用法二 嵌套从属类型名称
在上面的例子中，`C`是一个模板参数，而`C::const_iterator`依赖于`C`，这被称为从属名称，当一个从属名称嵌套在一个类里面的时候，成为嵌套从属名称。

`typename`对于嵌套从属名称是必须要有的，但是对于其他的则不允许有。
```C++
template<typename C>                   // typename allowed (as is "class")
void f(const C& container,             // typename not allowed
    typename C::iterator iter);       // typename required
```
### 例外
在类的初始化中，`typename`不允许出现，你可以理解为在里面出现的不管是啥东西默认为一个类型，除非它确实不是一个类型。
```C++
template<typename T>
class Derived: public Base<T>::Nested{  // 这里不加
public:
    explicit Derived(int x): Base<T>::Nested(x){    // 这里也不加
        typename Base<T>::Nested tmp;   // 但是这里要加
    }
};
```
### traits
C++中的STL简直如雷贯耳，但是对于其中更基本的`traits`（类型萃取机）则很多人都一无所知，后面对于这种会有更清晰地讲述，现在我们只是来看一下这一段最基本的应用。
```C++
template<typename IterT>
void workWithIterator(IterT iter)
{
    typedef typename std::iterator_traits<IterT>::value_type value_type;
    value_type temp(*iter);
}
```
如果我们知道`value_type`是什么东西以后，可以对于函数进行分门别类的处理。

只不过对于`typedef typename`看着很奇怪的，我觉得你可能又想来几遍#滑稽。
```C++
    typedef typename std::iterator_traits<IterT>::value_type value_type;
    typedef typename std::iterator_traits<IterT>::value_type value_type;
    typedef typename std::iterator_traits<IterT>::value_type value_type;
    typedef typename std::iterator_traits<IterT>::value_type value_type;
    typedef typename std::iterator_traits<IterT>::value_type value_type;
    typedef typename std::iterator_traits<IterT>::value_type value_type;
```
## Item 43 学习处理模板化基类内的名称
### 快速看点
+ 在模板的世界里面，大多数东西编译器都会假设他是不知道什么东西，所以你必须要通过某种手段让编译器认为是这个东西。
+ 通常来说模板调用错误会产生一大堆错误，这时不要心急，通过error来定位各种错误，并通过其他信息辅助判断或许是更好的选择。
### 为什么会假设不存在的第二个例子
你可能会问第一个例子在哪里？

其实就在前面，`typename C::const_iterator`的[位置](./Chapter7.md#const_iterator)。

既然你知道了，我们可以继续看下去了。假设有一系列公司需要传送信息，可以选择是明文还是密文发送，自然的就会有可以考虑这么写：
```C++
class CompanyA {
public:
    void sendCleartext(const std::string& msg);
    void sendEncrypted(const std::string& msg);
};
class CompanyB {
public:
    void sendCleartext(const std::string& msg);
    void sendEncrypted(const std::string& msg);
};
class MsgInfo { ... };
template<typename Company>
class MsgSender {
public:
    ...                                   // 构造，析构等等
    void sendClear(const MsgInfo& info)
    {
        std::string msg;
        create msg from info;

        Company c;
        c.sendCleartext(msg);
    }
    void sendSecret(const MsgInfo& info) { ... } //类似的
};
```
对于这种写法的想法建议先放一边，我们这个时候考虑一下更深一层的东西，比如写一个log进去：
```C++
template<typename Company>
class LoggingMsgSender: public MsgSender<Company> {
public:
    void sendClearMsg(const MsgInfo& info)
    {
        // write "before sending" info to the log;
        sendClear(info);
        // write "after sending" info to the log;
    }
};
```
这个时候编译器就会报出一个`error：use of undeclared identifier 'sendClear'; did you mean 'sendClearMsg'`。它甚至不知道你在调用那一个函数。

这个时候你可能就会进入一种黑人问号的阶段：我希望调用的是`MsgSender<Company>`中的`sendClear`，为什么编译器看不到？
### 为什么
下面来解释一下为什么编译期看不到这种东西。我们从前面一讲知道一个很重要的知识：如果编译器不知道这是不是一个类型，就假设不是一个类型，对于变量也是同理，所以编译器假设这玩意什么都不是，需要你强制指定才可以理解这是一个什么东西。

对于这样的函数就可以说：编译器不知道`MsgSender<Company>`是一个什么东西，所以假设这玩意的所有东西被隐藏（类似于之前讲过的名称遮蔽）以保证安全。

可是这玩意明明是一个类啊！已经不需要`typename`了。
```C++
template<>
class MsgSender<CompanyZ> {
public:
    void sendSecret(const MsgInfo& info) { ... }
};
```
在模板的世界中，没有什么是不可能发生的，特别是有偏特化这种东西存在的时候。这个公司没有`sendClear`这个函数，请问怎么调用呢？编译器考虑到这种情况，所以做出了这个举动。
### 解决
+ this指针
```C++
this->sendClear(info);
```
+ using
```C++
using MsgSender<Company>::sendClear;
```
+ 强制指定
```C++
MsgSender<Company>::sendClear(info);
```
子类模板无法访问父类模板中的名称是因为编译器不会搜索父类作用域，上述三个办法都是显式地让编译器去搜索父类作用域。 但如果父类中真的没有`sendClear`函数（比如模板参数是`CompanyZ`），在后续的编译中还是会抛出编译错误。
## Item 44 将与参数无关的代码抽离`template`s
### 快速看点
### 模板的好处与问题
模板是个好东西，能大大节约写代码的时间，至少一个`template`能解决的问题我至少不需要写第二份类似的代码进行处理，而且如果某一个类型是特殊的就需要单独实现，这也符合我们的期望。

但是模板存在一个问题，即使你的代码看起来非常短小，生成的二进制文件也可能会非常巨大，因为它会针对每类实例都会产生一个完整的副本。
### 代码重用
+ 在两个函数的代码差不多的时候，我们可以用第三个函数来实现两者，然后在两个函数中调用即可。这在operator=和复制构造函数里面用到很多。
+ 继承与复合。

这是我们在面向对象中的代码调用的可能情况中的两个例子。

在`template`s中，如果运用不当，可能会造成代码的过度膨胀，下面就举一个例子来说明一下。
### Square
假如你需要写一个矩阵类，你有可能会这样定义。
```C++
template<typename T>
class SquareMatrix {
public:
    ...
    void invert();
private:
    T* x;
    int n;
};
```
那么在矩阵乘法的时候就有可能遇到一个问题：如果大小不一样的话应该怎么处理，虽然你肯定不会这么想，但是肯定会有人这么写的，因为他不知道。通常的举措是加入异常，这是运行期的方法。

但是在`template`s中，这种情况可以完全杜绝。
```C++
template<typename T, std::size_t n>
class SquareMatrix {
public:
    ...
    void invert();
private:
    T* x;
};
```
这样就可以在编译期把两个不一样大的矩阵给报出来，因为`SquareMatrix<double, 10>`和`SquareMatrix<double, 5>`是不一样的类型。

但是这样可能会造成代码膨胀，特别是在有很多个这样的矩阵的时候，你会产生一大堆一样的代码却表示不同的类型。这个时候就可以采用这样的方式：
```C++
template<typename T>
class SquareMatrixBase {
protected:
    ...
    void invert(std::size_t matrixSize);
    ...
private:
    T* data;
    std::size_t n;
};
template<typename T, std::size_t n>
class SquareMatrix: private SquareMatrixBase<T> {
private:
    using SquareMatrixBase<T>::invert;
public:
    ...
    void invert() { this->invert(n); }
};
```
这样就可以减小总的代码大小。或者可以给一个更骚的实现：boost库。
```C++
boost::scoped_array<T> pData;
```
### 优异
注意这样的代码有几个点要注意：
+ `SquareMatrixBase::invert`是供子类用的，所以声明为`private`。
+ [调用父类`invert`的代价为零](./Chapter5.md#Item_30)，因为`SquareMatrix::invert`是隐式的`inline`函数。
+ 使用`this->`前缀是因为，`SquareMatrixBase`里的名称在子类模板`SquareMatrix`里是隐藏的。这个也在实现里面有。
+ 使用`private`继承是因为，`SquareMatrix`是根据`Base`类实现出来的，而不是就是`is-a`关系。

这里就有需要权衡的地方了。
+ 如果n是函数参数的话，传参总归还是有些困扰的。
+ 这样的结构相比于之前还是有些不直观，如果不嫌烦的话可以去看看现在STL的源码，被本书翻译骂了的那个#滑稽。

除此以外，我们只讨论了希望中非模板参数的代码膨胀，但是类型参数导致。
+ `int`和`long`
+ `list<int *>`, `list<const int *>`, `list<double *>`等等都是指针而已，但是还是会实例化成不同的模板。
## Item 45 运用成员函数模板接受所有兼容类型
### 快速看点
+ 成员函数模板可以帮助你兼容所有的类型的运算，但是请把控好尺度，否则回礼也很丰厚的。
+ 泛型赋值和初始化不能代替编译器自己生成的常规拷贝构造和拷贝赋值。
### 自己实现一个
shared_ptr这么智能，能不能自己实现一个，同时能得到裸指针的大部分特性？比如让这一段代码通过编译？
```C++
class Top { ... };
class Middle: public Top { ... };
class Bottom: public Middle { ... };
Top *pt1 = new Middle;                   // convert Middle* => Top*
Top *pt2 = new Bottom;                   // convert Bottom* => Top*
const Top *pct2 = pt1;                   // convert Top* => const Top*
```
这可能不是一个小问题，因为这涉及到不止一个类型的指针转化，而且要考虑到兼容和可扩展的问题，所以最好的选择还是模板。

首先我们完成从外界获取指针进行初始化的工作。
```C++
template<typename T>
class SmartPtr {
public:
    explicit SmartPtr(T *realPtr);
};
```
这样就能完成一些调用了。

然后加入初始化同类型指针的事情
```C++
    template<typename U>
    SmartPtr(const SmartPtr<U>& other);
```
这样就能完成大部分的初始化内容了（你也可以简单的理解为全部）。

然后我们从设计上考虑一些问题：我们设计这些指针最重要的目的有两个：
+ 模拟裸指针。
+ 在裸指针上面完成自己需要完成的安全性考虑。

从编译的角度考虑，我们只需要考虑第一点：裸指针。第二点考虑起来就不是一句两句话说的清楚的事情了。
+ 第一个声明为`explicit`，是因为我们希望使用者能知道这样的转化，至少是一个提醒。
+ 第二个不声明为`explicit`，与第一点正好相反。
+ 将初始化的方式从类的内部转换到类中裸指针的初始化，这样可以减少一些麻烦。

到这里就结束了吗？看上去是的，其实不是。
### 学习[shared_ptr](./Chapter2.md#Item_13)
这里东西就和原来的不一样了，我们先来看一下部分源码：
```C++
template<class T> class shared_ptr {
public:
    template<class Y>
        explicit shared_ptr(Y * p);
    template<class Y>
        shared_ptr(shared_ptr<Y> const& r);
    template<class Y>
        explicit shared_ptr(weak_ptr<Y> const& r);
    template<class Y>
        explicit shared_ptr(auto_ptr<Y>& r);
    template<class Y>
        shared_ptr& operator=(shared_ptr<Y> const& r);
    template<class Y>
        shared_ptr& operator=(auto_ptr<Y>& r);
};
```
看完源码你可能会询问一个问题：这里存在大量的模板，如果`T == Y`又会发生什么？

注意这里只出选了部分的源码，接着往下看另外一个角度看的一部分：
```C++
template<class T> class shared_ptr {
public:
    shared_ptr(shared_ptr const& r);

    template<class Y>
        shared_ptr(shared_ptr<Y> const& r);

    shared_ptr& operator=(shared_ptr const& r);

    template<class Y>
        shared_ptr& operator=(shared_ptr<Y> const& r);
};
```
于是真相大白了：
+ 根据`template`特化的规则，优先采用没有`template`的，之后采用全特化的，接着采用偏特化的，最后采用全部都是`template`的。这意味着如果两个`shared_ptr`进行互相转化，会优先调用自己。
+ 除此以外，还有一点：就算有了`template`的初始化函数，编译器也会给一个自己互相转化的初始化函数，为什么？看之前的重要规则：编译器不会把不能确定的东西当作类型的判据，也就是说，这些函数会被忽略掉，而在编译完成之后会有自己的类型。
## Item 46 需要类型转换时请为模板提供非成员函数
### 快速看点
+ `template`中的隐式转换和纯OO的转换不太一样，还是那句话，涉及到编译期不会把不能确定的东西当成一个可以用的东西。
### 之前的[Rational](./Chapter4.md#Rational)
我们把之前的`Rational`给翻出来看看
```C++
class Rational{
public:
    Rational(int n = 0, int d = 1);
    int numerator() const;
    int denominator() const;
}
const Rational operator*(const Rational& lhs, const Rational& rhs)
{
    return Rational(lhs.numerator() * rhs.numerator(), lhs.denominator() * rhs.denominator());
}
```
于是乎你突发奇想，想改成模板：
```C++
template<typename T>
class Rational {
public:
    Rational(const T& numerator = 0, const T& denominator = 1);
    const T numerator() const;
    const T denominator() const;
};

template<typename T>
const Rational<T> operator*(const Rational<T>& lhs, const Rational<T>& rhs); //省略掉，不写了。
```
然后你打算调用一下：
```C++
Rational<int> oneHalf(1, 2);
Rational<int> result = oneHalf * 2;
```
然后你拿到了一个编译错误：`invalid operands to binary expression ('Rational<int>' and 'int')`。

没找到那个函数？黑人问号.jpg。
```C++
    Rational<int> oneHalf(1, 2), Two(2);
    Rational<int> result = oneHalf * Two;
```
咋这能过编译呢？是不是编译器脑子有包？
### 参数推导？
我们来看一下我们希望编译器到底在干什么。
+ 对于`oneHalf`进行参数推导，发现`T = int`。
+ 希望第二个参数2也是`Rational<int>`。
+ 尝试把2转换成`Rational<int>`。
+ 转化成功，开始计算。

这是我们希望的编译过程，但是事与愿违，从前面两个程序的对比中你可以看出什么吗？

对！就是第三步2无法转化成`Rational`才导致的这一点。但是明明定义了转换函数，为什么没有转换呢？

因为编译器不会考虑这个。就这么简单。
>它们不这样做是因为在 template argument deduction（模板实参推演）过程中从不考虑 implicit type conversion functions（隐式类型转换函数）。从不。这样的转换可用于函数调用过程，这没错，但是在你可以调用一个函数之前，你必须知道哪个函数存在。为了知道这些，你必须为相关的 function templates（函数模板）推演出 parameter types（参数类型）（以便你可以实例化出合适的函数）。但是在 template argument deduction（模板实参推演）过程中不考虑经由 constructor（构造函数）调用的 implicit type conversion（隐式类型转换）。
### 类中定义
既然找不到我们就把这玩意丢进去喽。
```C++
template<typename T>
class Rational {
public:
    friend const Rational operator*(const Rational& lhs, const Rational& rhs);
};

template<typename T>
const Rational<T> operator*(const Rational<T>& lhs, const Rational<T>& rhs){}
```
继续尝试调用，发现能过编译了，但是不能过链接？难道非要把实现也丢在里面吗？

对！就要这样做。
+ 在oneHalf出现的时候，编译器能进行参数推导，并生成`Rational<int>`，然后初始化参数2，实验证明交换位置也可以完成同样的事情。
+ 在尝试链接的时候，发现`operator*`没有定义。这是因为其定义依旧不能推断出T是个什么东西。

那就没办法了，只能把定义也丢进去。
### 其他的手段
你可能觉得定义也丢进去实在太丑了，有没有比较好的手段呢？外面搞一个辅助函数就行了。
## Item 47 请使用`traits classes`表现类型信息
### 快速看点
+ STL提供了很多`trait`s和`iterator`s，以及很多tags，通过`template`s在编译期完成函数的确定和分类，最大化提升效率。
+ 关于这方面，译者自编了一本书，建议大家去看看，最少去看看网课，讲的真的不错。
### STL迭代器tag
```C++
struct input_iterator_tag {};
struct output_iterator_tag {};
struct forward_iterator_tag: public input_iterator_tag {};
struct bidirectional_iterator_tag: public forward_iterator_tag {};
struct random_access_iterator_tag: public bidirectional_iterator_tag {};
```
其中有这些区别：输入迭代器只能读取，并且只能读取一次，输出迭代器只能写入，并且只能写入一次，后面的基本就是字面意思。

这是最基本，最常用的区别，更多区别请看C++官网。
### `trait`s和tag的联合使用
由于`trait`s的实现实在太长了，这一块的具体实现还是自行查阅比较好。

通过`trait`s（萃取机）和tag，重载的联合使用，可以完成对于一些函数的分类，达到性能的最优化。这里给出一套比较经典的实现，不给注释，看看能否理解。
```C++
template <class _InputIter, class _Tp>
inline _InputIter find(_InputIter __first, _InputIter __last,
                       const _Tp& __val)
{
  __STL_REQUIRES(_InputIter, _InputIterator);
  __STL_REQUIRES_BINARY_OP(_OP_EQUAL, bool,
            typename iterator_traits<_InputIter>::value_type, _Tp);
  return find(__first, __last, __val, __ITERATOR_CATEGORY(__first));
}
template <class _InputIter, class _Tp>
inline _InputIter find(_InputIter __first, _InputIter __last,
                       const _Tp& __val,
                       input_iterator_tag)
{
  while (__first != __last && !(*__first == __val))
    ++__first;
  return __first;
}
template <class _RandomAccessIter, class _Tp>
_RandomAccessIter find(_RandomAccessIter __first, _RandomAccessIter __last,
                       const _Tp& __val,
                       random_access_iterator_tag)
{
  typename iterator_traits<_RandomAccessIter>::difference_type __trip_count
    = (__last - __first) >> 2;

  for ( ; __trip_count > 0 ; --__trip_count) {
    if (*__first == __val) return __first;
    ++__first;

    if (*__first == __val) return __first;
    ++__first;

    if (*__first == __val) return __first;
    ++__first;

    if (*__first == __val) return __first;
    ++__first;
  }

  switch(__last - __first) {
  case 3:
    if (*__first == __val) return __first;
    ++__first;
  case 2:
    if (*__first == __val) return __first;
    ++__first;
  case 1:
    if (*__first == __val) return __first;
    ++__first;
  case 0:
  default:
    return __last;
  }
}
```
看到这些奇怪的名字不要害怕，这些只是一系列变量而已。
### 如何定义`trait`s
```C++
template < ... >
class deque {
public:
    class iterator {
    public:
        typedef random_access_iterator_tag iterator_category;
    }:
};
```
使用的时候就是
```C++
deque::iterator
```
## Item 48 认识`template`s元编程
### 快速看点
+ 模板大法好
### 一个简单的小例子
假如你要写阶乘，你可能是这样写的
```C++
fac[0] = 1;
for(int i = 1; i < 20; i++)
    fac[i] = fac[i - 1] * i;
```
但是用了模板之后，可以这么写：
```C++
template<unsigned n>
struct Factorial{
    enum{ value = n * Factorial<n-1>::value };
};
template<>
struct Factorial<0>{
    enum{ value = 1 };
};

int main(){
    cout<<Factorial<5>::value;
}
```
黑人问号.jpg。利用偏特化理解递归终点，然后利用递归来得到结果，并且如果输入不合法立刻报错，编译都过不去。模板这么BT。
### 模板的真实水准
实际上，TMP(Template Metaprogramming)是被证明图灵完全的，也就是说只要能算的问题TMP都可以做，只不过可能样子你不认得而已。

再来看几个比较简单的例子吧，这些都是一旦错了连编译都过不去的#滑稽，异常都省下来了。
+ 量纲正确，通过`template`限制类型。
+ 表达式模板，消除运算过程中的临时对象，具体看[这个链接](https://blog.csdn.net/dbzhang800/article/details/6693454)
+ [策略模式](./Chapter6.md#Strategy)，这个自己回去翻。