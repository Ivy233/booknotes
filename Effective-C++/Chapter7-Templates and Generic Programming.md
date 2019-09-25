<style>
p{
    text-indent:2em;
}
</style>
# 模板与泛型编程
## Item 41 了解隐式接口与编译期多态
-----
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
对应的，这是一个针对Widget进行调用的简单函数：
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
不知道你有没有想过为什么面向对象的.size()能被这么清晰的解读出来，或者!=这个符号也能这么清晰的解读出来？因为他就在代码里面啊！编译期只需要用正确的打开方式就能找到我调用的东西，从而检查我的调用情况。
同样的，由于有很多virtual函数，因此需要进行很多运行时多态来确定要执行哪一个类的函数。

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
因为T的类型不确定，所以.size()到底是什么东西在外面的lint是不知道的。当然有的比较牛皮一点可以知道简单的，但是根据我复刻STL的经历，总是F12不了的痛苦意味着对于稍微复杂一点的模板而言lint也无能为力。这就是编译期多态，只有在编译的时候才能真正确定到底调用的是什么函数，并且正确的展开函数进入内存以供调用。

这就是编译期多态，而由于不知道具体类型的对于接口的要求称之为隐式接口。
## Item 42 了解typename的双重含义
-----
### 快速看点
+ typename由两种用法，第一种用法和class在template<class T>中是类似的，两者可以互换。
+ typename的另外一种用法可以强制认为一种不知道什么类型的东西是一种类型，虽然到底是什么类型依旧不清楚，但是至少可以嵌套依赖类型名的报错。
### 用法一
在了解过之前的内容之后，你可能会认为，那我简单点写class，就少了几个字母？实际上，这两者的区别更倾向于对于类模板初始化的倾向：如果对于所有类型都有效，采用typename，暗示不必要是一个class。
### 用法二 声明类型
我们来看一下这个函数，它有一个不是很明显的问题。
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

问题出在代码的第4行，C::const_iterator并不是如你所想一样会直接产生一个类型，而是选择了报错。

发生这个奇怪的问题的原因就是编译期并不知道这是一个类型，虽然字面意义上来看这肯定不是一个类型，但是不能总是保证说没有一个代码狂人写了一个交作const_iterator的static变量或者const变量丢在里面。编译器不知道这玩意是不是一个类型，所以只好假定这玩意不是，除非你说这玩意是。这种情况被称之为“嵌套依赖类型名”

所以为了让编译器认为这是一个类型，我们选择给这玩意加一个typename：
```C++
typename C::const_iterator it(container.begin());
```
这个时候聚焦到代码的第2行，你又会发现一个问题。
### 用法二 嵌套从属类型名称
在上面的例子中，C是一个模板参数，而C::const_iterator依赖于C，这被称为从属名称，当一个从属名称嵌套在一个类里面的时候，成为嵌套从属名称。

typename对于嵌套从属名称是必须要有的，但是对于其他的则不允许有。
```C++
template<typename C>                   // typename allowed (as is "class")
void f(const C& container,             // typename not allowed
    typename C::iterator iter);       // typename required
```
### 例外
在类的初始化中，typename不允许出现，你可以理解为在里面出现的不管是啥东西默认为一个类型，除非它确实不是一个类型。
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
C++中的STL简直如雷贯耳，但是对于其中更基本的traits（类型萃取机）则很多人都一无所知，后面对于这种会有更清晰地讲述，现在我们只是来看一下这一段最基本的应用。
```C++
template<typename IterT>
void workWithIterator(IterT iter)
{
    typedef typename std::iterator_traits<IterT>::value_type value_type;
    value_type temp(*iter);
}
```
如果我们知道value_type是什么东西以后，可以对于函数进行分门别类的处理。

只不过对于typedef typename看着很奇怪的，我觉得你可能又想来几遍#滑稽。
```C++
    typedef typename std::iterator_traits<IterT>::value_type value_type;
    typedef typename std::iterator_traits<IterT>::value_type value_type;
    typedef typename std::iterator_traits<IterT>::value_type value_type;
    typedef typename std::iterator_traits<IterT>::value_type value_type;
    typedef typename std::iterator_traits<IterT>::value_type value_type;
    typedef typename std::iterator_traits<IterT>::value_type value_type;
```
## Item 43 学习处理模板化基类内的名称
----
### 快速看点
+ 在模板的世界里面，大多数东西编译器都会假设他是不知道什么东西，所以你必须要通过某种手段让编译器认为是这个东西。
+ 通常来说模板调用错误会产生一大堆错误，这时不要心急，通过error来定位各种错误，并通过其他信息辅助判断或许是更好的选择。
### 为什么会假设不存在的第二个例子
你可能会问第一个例子在哪里？

其实就在前面，typename C::const_iterator的位置。

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
这个时候编译器就会报出一个error：use of undeclared identifier 'sendClear'; did you mean 'sendClearMsg'?他甚至不知道你在调用那一个函数。

这个时候你可能就会进入一种黑人问号的阶段：我希望调用的是MsgSender<Company>中的sendClear，为什么编译器看不到？
### 为什么
下面来解释一下为什么编译期看不到这种东西。我们从前面一讲知道一个很重要的知识：如果编译器不知道这是不是一个类型，就假设不是一个类型，对于变量也是同理，所以编译器假设这玩意什么都不是，需要你强制指定才可以理解这是一个什么东西。

对于这样的函数就可以说：编译器不知道MsgSender<Company>是一个什么东西，所以假设这玩意的所有东西被隐藏（类似于之前讲过的名称遮蔽）以保证安全。

可是这玩意明明是一个类啊！已经不需要typename了。
```C++
template<>
class MsgSender<CompanyZ> {
public:
    void sendSecret(const MsgInfo& info) { ... }
};
```
在模板的世界中，没有什么是不可能发生的，特别是有偏特化这种东西存在的时候。这个公司没有sendClear这个函数，请问怎么调用呢？编译器考虑到这种情况，所以做出了这个举动。
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
子类模板无法访问父类模板中的名称是因为编译器不会搜索父类作用域，上述三个办法都是显式地让编译器去搜索父类作用域。 但如果父类中真的没有sendClear函数（比如模板参数是CompanyZ），在后续的编译中还是会抛出编译错误。
## Item 44 将与参数无关的代码抽离templates
----
### 快速看点
### 模板的好处与问题
模板是个好东西，能大大节约写代码的时间，至少一个template能解决的问题我至少不需要写第二份类似的代码进行处理，而且如果某一个类型是特殊的就需要单独实现，这也符合我们的期望。

但是模板存在一个问题，即使你的代码看起来非常短小，生成的二进制文件也可能会非常巨大，因为它会针对每类实例都会产生一个完整的副本。
### 代码重用
+ 在两个函数的代码差不多的时候，我们可以用第三个函数来实现两者，然后在两个函数中调用即可。这在operator=和复制构造函数里面用到很多。
+ 继承与复合。

这是我们在面向对象中的代码调用的可能情况中的两个例子。

在templates中，如果运用不当，可能会造成代码的过度膨胀，下面就举一个例子来说明一下。
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

但是在templates中，这种情况可以完全杜绝。
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
这样就可以在编译期把两个不一样大的矩阵给报出来，因为SquareMatrix<double, 10>和SquareMatrix<double, 5>是不一样的类型。

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
+ SquareMatrixBase::invert是供子类用的，所以声明为private。
+ 调用父类invert的代价为零，因为SquareMatrix::invert是隐式的inline函数。如果你忘记这个的话建议往前翻一下。
+ 使用this->前缀是因为，SquareMatrixBase里的名称在子类模板SquareMatrix里是隐藏的。这个也在实现里面有。
+ 使用private继承是因为，SquareMatrix是根据Base类实现出来的，而不是就是is-a关系。

这里就有需要权衡的地方了。
+ 如果n是函数参数的话，传参总归还是有些困扰的。
+ 这样的结构相比于之前还是有些不直观，如果不嫌烦的话可以去看看现在STL的源码，被本书翻译骂了的那个#滑稽。

除此以外，我们只讨论了希望中非模板参数的代码膨胀，但是类型参数导致。
+ int和long
+ list<int*>, list<const int*>, list<double*>等等都是指针而已，但是还是会实例化成不同的模板。
## Item 45 运用成员函数模板接受所有兼容类型
----
### 快速看点
## Item 46 需要类型转换时请为模板提供非成员函数
----
### 快速看点
## Item 47 请使用traits classes表现类型信息
----
### 快速看点
## Item 48 认识templates元编程
----
### 快速看点
