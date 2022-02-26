# 资源管理
## Item 32 确定你的`public`继承塑模出`is-a`关系
### 快速看点
+ `public`继承意味着`is-a`，并且适用于`base class`地每一件事情同样也适用于`derived class`。
+ 有很多逻辑上的`is-a`不一定是代码中的`is-a`，除非你修改实现。
### `public`的含义
很容易就能看出下面的推断是正确的：因为`Student`始终是一个`Person`，但是反过来就不一样。
<span id='Student'></span>

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
### `public`的误区
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
请问在修改的时候第二个`assert`改变了吗？

如你所见，就算是现实世界中应该正确的事情，再继承中都不一定成立。
### 其他关系
+ `has-a`：再类中带入一个其他类的成员。
+ `is-implemented-in-terms-of`：用这个东西实现出来。

<span id='Item_33'></span>

## Item 33 避免遮掩继承而来的名称
### 快速看点
+ `derived class`会遮盖掉`base class`的名字，而且重载函数也会一起遮盖。
+ 让隐藏的名字重新可见有很多种方法，可以直接`using`和转调函数。
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
在这个例子中，`Derived::mf1`遮掩了`Base::mf1`，只有在动态类型为base的时候才可以直接找到（然后报错）。同样的，`mf2`-`mf4`都没有发生遮掩。如果在`mf4`中调用了`mf2`，那么会在`Base`中找到，如果假设`Base`中没有，那么会在全局函数中寻找，在这之前还要找一下`namespace`。
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
你会发现两个调用x的都报了错：`too many arguments to function call, expected 0, have 1; did you mean 'Base::mf1'?`甚至clang++给你提示了应当如何修改。
### 修改
+ 在`Derived`加入一行`using Base::mf1`，再次查看发现`mf1`的报错都消失了。这样我们就完成了对于Base::mf1的全部引入，而`Derived`修改的仅仅是不带参数的`mf1`。
+ 转发：在`Derived`中直接调用`Base::mf1`，这样可以屏蔽不想把所有`mf1`都带入进来的问题。
+ 如果像写几个但不是全部？一个一个来吧。
## Item 34 区分接口继承与实现继承
### 快速看点
+ 接口继承与实现继承不同。在`public`下，`derived class`总是继承基类接口。
+ `pure virtual`函数指定仅有接口被继承。
+ 其他的`virtual`函数指定接口继承和默认的继承。
+ `non-virtual` 函数，指定接口继承带上强制实现继承。
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
这个时候调用`ModelA::fly`和`ModelB::fly`是允许的。

但是这个时候加入了一种`ModelC`，他的飞行方式和AB都不一样，于是乎如果忘记重写的话，也会按照AB的代码进行飞行。卒。这个时候该怎么办才能隐藏掉`fly`，进而在C没有自定义方法的时候要求强制定义？
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

这里需要注意一点：`defaultFly`是一个`non-virtual`函数，[Item 36](./Chapter6.md#Item_36)保证了这玩意不应该被重新实现。如果这玩意是一个`virtual`的话，是不是该有一个defaultdefaultFly呢？
### 一个优雅的实现
有人认为这样子写就是一大堆函数，于是乎想让`defaultFly`和`fly`函数进行合并，C++允许提供`pure virtual`函数之外提供一个实现，然后强制调用。
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
区分好三者(`non-virtual`，`virtual`，`pure virtual`)的关系和区别，不要滥用`virtual`或者极度防守的使用`non-virtual`，只有你知道什么你才能用的好。
## Item 35 考虑`virtual`函数以外的其他选择
### 快速看点
假设你在开发一个游戏，然后需要计算一个人物的健康值，那么你可以这么写。
```C++
class GameCharacter {
public:
    virtual int healthValue() const;
};
```
实现成`virtual`代表了这玩意仅仅是给出了默认实现，可以替换。但是有没有其他方法呢？
<span id='NVI'></span>

### NVI模式
把所有的东西交给一个函数有些时候过于让人不去关注，从而破坏了异常安全的部分。那么这个时候就可以考虑`public`函数提供一个强制实现，而`private`里面提供一个`virtual`，这样你希望不被修改的东西丢在`public`里面，`private`里面丢可调整的部分。
```C++
class GameCharacter{
public:
    int healthValue() const{
        // 你可以做些事情，比如异常安全
        int ret = doHealthValue();
        // 这里也可以做
        return ret;
    }
private:
    virtual int doHealthValue() const{
        // 默认实现
    }
}
```
NVI的好处已经在之前和代码里提到了。

但是C++有个奇怪的地方，`doHealthValue`是不可调用的，但是子类却可以重写。但是实际上仔细想想没啥大的问题：我定义我的，你调用你自己的，我又不调用你的`private`，在外面调用也可以调用`private`函数。

但是如果要求调用`base class`的东西的时候，就需要把这玩意声明为`protected`了。
<span id='Strategy'></span>

### 函数指针实现策略模式
NVI确实是不错的东西，但是还是在虚函数中做文章。有些更加夸张的主张：直接把函数替换掉，这样就不会卡在一个函数里面了。
```C++
class GameCharacter;
int defaultHealthCalc(const GameCharacter& gc);
class GameCharacter{
public:
    typedef int (*HealthCalcFunc)(const GameCharacter&);//函数指针
    //初始化
    explicit GameCharacter(HealthCalcFunc hcf = defaultHealthCalc)
        : healthFunc(hcf){}
    //调用
    int healthValue() const{
        return healthFunc(*this);
    }
private:
    HealthCalcFunc healthFunc;
}
```
这就是被称为策略模式(`Strategy`)，可以在运行时指定每个对象的生命值计算策略，比虚函数有更大的灵活性。
+ 同一角色类的不同对象可以有不同的`healthCalcFunc`，只需要在构造的时候传入即可。
+ 对于一个指定的角色健康值的计算函数可以在运行时改变。

但是实际上有一个巨大的问题：我们在外部定义了这个函数，于是无法访问`private`和`protected`，这很有可能会弱化封装。在弱化封装和便捷性之间做出一个平衡需要考虑的。
### function模板实现策略模式
如果我们加入模板的话
```C++
class GameCharacter;
int defaultHealthCalc(const GameCharacter& gc);
class GameCharacter{
public:
    //声明
    typedef std::function<int (const GameCharacter&)> HealthCalcFunc;
    //初始化
    explicit GameCaracter(HealthCalcFunc hcf = defaultHealthCalc)
        : healthCalcFunc(hcf){}
    //调用
    int healthValue() const{
        return healthFunc(*this);
    }
private:
    HealthCalcFunc healthFunc;
};
```
这样实现相对于纯指针而言有一些好处
+ 返回值只需要兼容`int`即可，比如`short`。
+ 输入函数可以为任意函数，仿函数与成员函数，只不过是有多复杂的问题。
```C++
//function
short calcHealth(const GameCharacter&);
xxx(calcHealth);
//functor
struct HealthCalculator{
    int operator()(const GameCharacter&) const{...}
};
xxx(HealthCalculator());
//member-function
class GameLevel{
public:
    float health(const GameCharacter&) const;
};
GameLevel pqr;
xxx(std::bind(&GameLevel::health, pqr, _1));
```
（说句实话第三个我时第一次见，C++11果然和C++是两门语言。）
（建议实现一下STL，就不会看不懂模板了）

这里有必要解释一下`bind`是怎么用的。对于这样的函数而言，`*this`是第一个参数，而众所周知_1是待填充的一个东西，这个`bind`的第二个参数才`GameLevel::health()`参数表的第一个参数。
### 经典的策略模式
看完了这么多可能看不懂的代码之后，我们来看一点简单的，不秀这么多神奇的语法，仅仅在继承上面做文章。

我们尝试把计算函数做成一个类，然后由角色进行has-a的引入。之后
```C++
class GameCharacter {
public:
    explicit GameCharacter(HealthCalcFunc *phcf = &defaultHealthCalc)
        : pHealthCalc(phcf) {}
    int healthValue() const { return pHealthCalc->calc(*this);}

private:
    HealthCalcFunc *pHealthCalc;
};
```
这样是不是结构要比之前明晰多了？虽然我没给出`HealthCalcFunc`的实现，但是也可以猜出一个大概。

这就是标准的策略模式。
<span id='Item_36'></span>

## Item 36 绝不重新定义继承而来的`non-virtual`函数
### 快速看点
+ 如题
### 一个例子
我们考虑有两个类，其中`Derived`类public继承了`Base`，而且`Base`定义了一个`func`函数。那么这段代码应当两个`func`的调用用的是同一个`func`。
```C++
Derived d;
Base* pb = &d;
Derived* pd = &d;
pb->func();
pd->func();
```
但是如果Derived继承的`func`被重新定义了呢？
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
+ 如果一个类`public`继承了另外一个，那么就认为`Derived`类也是一种`Base`类。如果这个时候要求两者的同一个函数进行的是不同的操作，那么这个时候就会是很难受的事情了：到底两个是否是`is-a`关系呢？
+ 除此以外，如果你用名称遮盖了`Base::func`，那也应该用`virtual`函数来指定，否则一旦派生类变成了子类，他就再也没有机会访问到自己的函数了。
## Item 37 绝不重新定义继承而来的缺省参数值
### 快速看点
+ 如题
+ 函数缺省参数值是静态绑定，但是`virtual`是动态绑定。
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
应该能很明显的发现调用的`draw`是什么东西——如果你不知道的话请回头看一下`virtual`的用法。但是今天讨论的问题是：那个参数`color`到底是什么。
### 问题
如果你把代码丢进去编译的话，发现代码是能编译的通过的。通过这句话你就应该察觉到不对：`Circle`的应该是要求输入参数的，为什么能通过编译呢？然后你在里面加入了输出`color`和类的特征字符串。结果得到了两个0。

多么神奇的结果！`Base`的指针获得了`Derived`的对象之后可以调用`Derived`的函数，但是传入的参数却是`Base`的。

这个不正常的问题就显而易见了：为什么采取了静态缺省参数和动态绑定函数名？这本书给出了这样的解释：
>为什么 C++ 要坚持按照这种不正常的方式动作？答案是为了运行时效率。如果 default parameter values（缺省参数值）是 dynamically bound（动态绑定），compilers（编译器）就必须提供一种方法在运行时确定 virtual functions（虚拟函数）的 parameters（参数）的 default value(s)（缺省值），这比目前在编译期确定它们的机制更慢而且更复杂。最终的决定偏向了速度和实现的简单这一边，而造成的结果就是你现在可以享受高效运行的乐趣，但是，如果你忘记留心本 Item 的建议，就会陷入困惑。
### 解决
这个和[Item 35](./Chapter6.md#NVI)一样的，可以考虑使用NVI的实现方式，当然，其他的方法也有很多，比如不再重定义默认参数。
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
## Item 38 通过复合塑模出`has-a`或者“根据某物实现出”
### 快速看点
+ `is-a`，`has-a`这三个分别对应`public`继承，在内部引入。（如果你想简单一点理解的话。
+ STL对象不建议`public`继承。这里是重提一遍。
+ 如果更具体一点说明的话，`has-a`指对象，而将方法转发出来的行为意味着根据某物实现出。
### 还需要讲什么吗
## Item 39 明智而审慎地使用`private`继承

### 快速看点
+ `private`继承意味着根据某物实现出，能掩盖掉从基类继承来的成员。
+ 除非绝对必要，否则不要用`private`继承。
+ 与复合不同，私有继承能使空基类优化有效，这对于致力于最小化对象大小的开发者来说极其重要。
### 一个奇怪的例子
我们实现了一个Person类，一个[`Student`](./Chapter6.md#Student)类，并且产生如下的调用：
```C++
class Person { ... };
class Student: private Person { ... };
void eat(const Person& p);

Person p;
Student s;
eat(p);
eat(s); //???
```
理论上学生能吃啊，结果你会在这里收到一条编译错误：`cannot cast 'const Student' to its private base class 'const Person'`。
### 复合与`private`继承的区别
加入我们写了一个`Widget`表示一个窗口，我们想实现一个`Timer`类记录时间，从而进一步体现出随着时间的调用次数变化。

这里很显然不是`public`继承，因为这两者不构成`is-a`关系，并且很容易被其他类调用。除此以外我们不希望被`derived class`调用，因此考虑只有两种选择：`private`继承和复合。

然后我们先写一个`Timer`的实现：
```C++
class Timer {
public:
   explicit Timer(int tickFrequency);
   // automatically called for each tick
   virtual void onTick() const;
};
```
+ private继承下，我们可以给出这样的实现：
```C++
class Widget: private Timer {
private:
    // look at Widget usage data, etc.
    virtual void onTick() const;
};
```
不过这一种有点问题：`friend`类可以访问，而且继承都可以重定义这玩意。
+ 在复合中可以给出一个这样的实现。
```C++
class Widget {
private:
    class WidgetTimer: public Timer {
    public:
        virtual void onTick() const;
    };
    WidgetTimer timer;
};
```
在很多情况下，大多数人都会选择第二种方式，具体原因可以仔细比较（提示：从继承和编译依赖考虑）。
### 什么时候可以使用private继承
+ 当一个将要成为`derived class`的类访问其`base class`的`protected`构建，但是关系却不是`is-a`，而是根据某物实现出。
+ 另一种情况就是空间最优化，这是一种极端情况。对于一个什么占用空间的成员都没有的类而言，其`sizeof`大多数为1，因为C++会给这玩意插入一个`char`。如果将其进行复合处理，会进行空间占用，进而出现空间膨胀；这种情况下进行`private`继承会好很多，因为可以合并空间，减小其`sizeof`。虽然这种情况在日常编程中很少发生，但是不要忽略掉嵌入式。
<span id='Item_40'></span>

## Item 40 明智而审慎地使用多重继承
### 快速看点
对于多继承，C++社区会有两个基本阵营。
+ 单继承有好处，多继承更有好处。
+ 单继承有好处，但是多继承的麻烦会导致得不偿失。

说句实话，我更支持第二种，因为多重继承会过多引入成员，更复杂的是对于一样的函数（比如`fstream`继承于`iftream`和`oftream`），应该继承谁的？相比之下，Java的单继承+多接口更加舒服，但是如果可以给默认实现就更能体现灵活性。
### 歧义问题
```C++
class A{
public:
    void func();
};
class B{
private:
    bool func() const;
};
class C: public A, public B{ ... };
C c;
c.func(); //???
```
请问最后一行的func调用的是谁的呢？对应错误内容为：`member 'func' found in multiple base classes of different types`。

在编译器中，两个func其实都得到了继承，所以`public A, public B`这里没有报错。也就是说，既然两者都在C中，那么我们只需要指定是谁的就可以解决问题。
```C++
c.A::func();
c.B::func(); // 'func' is a private member of 'B'
```
### 多继承菱形
我们来看一个经典例子：
```C++
class File { ... };
class InputFile: public File { ... };
class OutputFile: public File { ... };
class IOFile: public InputFile, public OutputFile
{ ... };
```
除了函数调用的歧义以外，还有一个明显的问题：你打开文件的时候总归需要一个文件名，那么这个时侯打开的文件名重复了就很尴尬。

对于这个问题，C++选择了普渡众生，虽然默认选择了复制，但是给出了另一个选择，就是在不需要的时候进行虚拟继承(virtual inheritance)。

可能有很多时候需要`virtual`继承，但总是会有代价的。
+ 虚继承类的对象会更大一些。
+ 虚继承类的成员访问会慢一些。
+ 虚继承类的初始化反直觉。继承层级的最底层负责虚基类的初始化，而且负责整个继承链上所有虚基类的初始化。

所以能不用就不用，能不丢数据就不要丢数据。
### 接口类
类似于Java的接口，没有实现的类交作接口类：
```C++
class IPerson {
public:
    virtual ~IPerson();
    virtual std::string name() const = 0;
    virtual std::string birthDate() const = 0;
};
```
同样的，类似于Java，可以同时实现接口类与实现类
```C++
class CPerson: public IPerson, private PersonInfo{}
```
由于由`virtual`继承，虚函数可以在里面重写并且继承。如果想要重写的话，还得是要`private`继承。