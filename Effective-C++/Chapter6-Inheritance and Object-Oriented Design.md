<style>
p{
    text-indent:2em;
}
</style>
# 资源管理
## Item 32 确定你的public继承塑模出is-a关系
----
### 快速看点
+ public继承意味着is-a，并且适用于base class地每一件事情同样也适用于derived class。
+ 有很多逻辑上的is-a不一定是代码中的is-a，除非你修改实现。
### public的含义
很容易就能看出下面的推断是正确的：因为Student始终是一个Person，但是反过来就不一样。
```C++
class Person {...};
class Student: public Person {...};
void eat(const Person& p);    // anyone can eat
void study(const Student& s); // only students study
Person p;                     // p is a Person
Student s;                    // s is a Student
eat(p);                       // fine, p is a Person
eat(s);                       // fine, s is a Student, and a Student is-a Person
study(s);                     // fine
study(p);                     // error! p isn't a Student
```
### public的误区
如果我们假设鸟能飞，但是企鹅也是一种鸟，它不能飞，这个时候该怎么办呢？
```C++
class Bird {
public:
    virtual void fly();                  // birds can fly
};
class Penguin:public Bird {            // penguins are birds
    ...
};
```
不如：
```C++
class Bird {
    ...                                       // no fly function is declared
};
class FlyingBird: public Bird {
public:
    virtual void fly();
    ...
};

class Penguin: public Bird {
    ...                                       // no fly function is declared
};
```
这样修改就能更好的体现会飞的鸟会飞但是企鹅虽然是鸟但不会飞这个性质。

如果你觉得这个例子不够生动形象的话，可以来看下面的例子：

请先回答：正方形是一种长方形吗？

基本上大多数人都会认为：正方形肯定是的。

那么在代码中呢？
```C++
class Rectangle {
public:
  virtual void setHeight(int newHeight);
  virtual void setWidth(int newWidth);

  virtual int height() const;               // return current values
  virtual int width() const;

  ...

};
class Square: public Rectangle {...};
Square s;
...
assert(s.width() == s.height());//正方形应该长宽相等
s.setHeight(s.height() + 10);
assert(s.width() == s.height());//正方形应该长宽相等
```
请问在修改的时候第二个assert改变了吗？

如你所见，就算是现实世界中应该正确的事情，再继承中都不一定成立。
### 其他关系
+ has-a：再类中带入一个其他类的成员。
+ is-implemented-in-terms-of：用这个东西实现出来。
## Item 33 避免遮掩继承而来的名称
---
### 快速看点
+ derived class会遮盖掉base class的名字，而且重载函数也会一起遮盖。
+ 让隐藏的名字重新可见有很多种方法，可以直接using和转调函数。
### 我们认识的名称遮掩
```C++
int x;
void func(){
    double x;
    cin>>x;     // read a new value for local x
}
```
这个在正常不过了对吧。再来看第二个例子：
```C++
class Base {
private:
    int x;
public:
    virtual void mf1() = 0;
    virtual void mf2();
    void mf3();
};

class Derived: public Base {
public:
    virtual void mf1();
    void mf4();
};
```
在这个例子中，derived的mf1遮掩了base的mf1，只有在动态类型为base的时候才可以直接找到（然后报错）。同样的，mf2-mf4都没有发生遮掩。如果在mf4中调用了mf2，那么会在Base中找到，如果假设base中没有，那么会在全局函数中寻找，在这之前还要找一下namespace。
### 我们意料之外的遮掩
来加几个东西：
```C++
class Base {
private:
    int x;
public:
    virtual void mf1() = 0;
    virtual void mf1(int);
    virtual void mf2();
    void mf3();
    void mf3(double);
};

class Derived: public Base {
public:
    virtual void mf1();
    void mf3();
    void mf4();
};
```
这个时候发生了什么呢？看上去什么都没有。然后我们写一下调用：
```C++
Derived d;
int x;
d.mf1();
d.mf1(x);
d.mf2();
d.mf3();
d.mf3(x);
```
你会发现两个调用x的都报了错：
>too many arguments to function call, expected 0, have 1; did you mean 'Base::mf1'?
甚至clang++给你提示了应当如何修改。
### 修改
+ 在Derived加入一行using Base::mf1，再次查看发现mf1的报错都消失了。这样我们就完成了对于Base::mf1的全部引入，而Derived修改的仅仅是不带参数的mf1。
+ 转发：在Derived中直接调用Base::mf1，这样可以屏蔽不想把所有mf1都带入进来的问题。
+ 如果像写几个但不是全部？一个一个来吧。
## Item 34 区分接口继承与实现继承
---
### 快速看点
+ 接口继承与实现继承不同。在public下，derived class总是继承基类接口。
+ pure virtual函数指定仅有接口被继承。
+ 其他的virtual函数指定接口继承和默认的继承。
+ non-virtual 函数，指定接口继承带上强制实现继承。
### 继承
```C++
class Shape{
public:
    virtual void draw() const = 0;
    virtual void error(const string& msg);
    int id() const;
};
class Rect: public Shape{...};
```
具体是啥请看快速看点。

不过快速看点没讲过的是，可以
```C++
Shape* a = new Rect();
a->draw(); //因为已经提供实现了，所以draw肯定被定义过。
```
### 一个危险的继承
我们假设有一座机场，里面飞两种飞机，于是可以考虑这样的实现：
```C++
class Airplane
{
public:
    virtual void fly() {}
};
class ModelA : public Airplane
{
};
class ModelB : public Airplane
{
};
```
这个时候调用ModelA和ModelB的fly是允许的。

但是这个时候加入了一种ModelC，他的飞行方式和AB都不一样，于是乎如果忘记重写的话，也会按照AB的代码进行飞行。卒。这个时候该怎么办才能隐藏掉AB的fly，进而在C没有自定义方法的时候要求强制定义？
### 一个比较安全的实现
```C++
class Airplane
{
public:
    virtual void fly()  = 0;
protected:
    void defaultFly() {}
};
class ModelA : public Airplane
{
public:
    virtual void fly() {defaultFly();}
};
class ModelB : public Airplane
{
public:
    virtual void fly() {defaultFly();}
};
```
这样就解决了默认的问题：如果C不重定义的话就等着报错吧。

这里需要注意一点：defaultFly是一个non-virtual函数，Item 36保证了这玩意不应该被重新实现。如果这玩意是一个virtual的话，是不是该有一个defaultdefaultFly呢？
### 一个优雅的实现
有人认为这样子写就是一大堆函数，于是乎想让defaultFly和fly函数进行合并，C++允许提供pure virtual函数之外提供一个实现，然后强制调用。
```C++
class Airplane
{
public:
    virtual void fly()  = 0;
};
void Airplane::fly() {}
class ModelA : public Airplane
{
public:
    virtual void fly() {Airplane::fly();}
};
class ModelB : public Airplane
{
public:
    virtual void fly() {Airplane::fly();}
};
```
如果直接调用fly的话，可是会循环的。
### 忠告
区分好三者（non-virtual，virtual，pure virtual）的关系和区别，不要滥用virtual或者极度防守的使用non-virtual，只有你知道什么你才能用的好。
## Item 35 考虑virtual函数以外的其他选择
---
### 快速看点
## Item 36 绝不重新定义继承而来的non-virtual函数
---
### 快速看点
+ 如题
### 一个例子
我们考虑有两个类，其中Derived <b>public</b>继承了Base，而且Base定义了一个func函数。那么这段代码应当两个func的调用用的是同一个func。
```C++
Derived d;
Base* pb = &d;
Derived* pd = &d;
pb->func();
pd->func();
```
但是如果Derived继承的func被重新定义了呢？
```C++
class Base{
public:
    void func(){}
};
class Derived: public Base{
public:
    void func(){}   // 隐藏了父类的名称func，见Item 33
};
```
这个时候两者就不一样了。

不一样就不一样了，有什么问题呢？
### 和设计模式相关的
+ 如果一个类public继承了另外一个，那么就认为Derived类也是一种Base类。如果这个时候要求两者的同一个函数进行的是不同的操作，那么这个时候就会是很难受的事情了：到底两个是否是is-a关系呢？
+ 除此以外，如果你用名称遮盖了base类的func，那也应该用virtual函数来指定，否则一旦派生类变成了子类，他就再也没有机会访问到自己的函数了。
## Item 37 绝不重新定义继承而来的缺省参数值
---
### 快速看点
+ 如题
+ 函数缺省参数值是静态绑定，但是virtual是动态绑定。
### 为什么会有这种问题？
假如我们按照颜色来画图画：
```C++
class Shape {
public:
    enum ShapeColor { Red, Green, Blue };
    virtual void draw(ShapeColor color = Red) const = 0;
};
class Rectangle: public Shape {
public:
    virtual void draw(ShapeColor color = Green) const;
};
class Circle: public Shape {
public:
    virtual void draw(ShapeColor color) const;
};
```
如果我们进行了如下的调用：
```C++
Shape *pc = new Circle;
Shape *pr = new Rectangle;
pc->draw();
pr->draw();
```
应该能很明显的发现调用的draw是什么东西——如果你不知道的话请回头看一下virtual的用法。但是今天讨论的问题是：那个参数color到底是什么。
### 问题
如果你把代码丢进去编译的话，发现代码是能编译的通过的。通过这句话你就应该察觉到不对：Circle的应该是要求输入参数的，为什么能通过编译呢？然后你在里面加入了输出color和类的特征字符串。结果得到了两个0。

多么神奇的结果！Base的指针获得了Derived的对象之后可以调用Derived的函数，但是传入的参数却是Base的。

这个不正常的问题就显而易见了：为什么采取了静态缺省参数和动态绑定函数名？这本书给出了这样的解释：
>为什么 C++ 要坚持按照这种不正常的方式动作？答案是为了运行时效率。如果 default parameter values（缺省参数值）是 dynamically bound（动态绑定），compilers（编译器）就必须提供一种方法在运行时确定 virtual functions（虚拟函数）的 parameters（参数）的 default value(s)（缺省值），这比目前在编译期确定它们的机制更慢而且更复杂。最终的决定偏向了速度和实现的简单这一边，而造成的结果就是你现在可以享受高效运行的乐趣，但是，如果你忘记留心本 Item 的建议，就会陷入困惑。
### 解决
这个和Item 35一样的，可以考虑使用NVI的实现方式，当然，其他的方法也有很多，比如不再重定义默认参数。
```C++
class Shape{
public:
    void draw(Color c = Red) const{
        doDraw(color);
    }
private:
    virtual void doDraw(Color c) const = 0;
};
class Rect: public Shape{
    ...
private:
    virtual void doDraw(Color c) const;
};
```
## Item 38 通过复合塑模出has-a或者“根据某物实现出”
---
### 快速看点
## Item 39 明智而审慎地使用private继承
---
### 快速看点
## Item 40 明智而审慎地使用多重继承
---
### 快速看点