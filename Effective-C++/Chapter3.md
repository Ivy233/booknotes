# 资源管理
<span id='Item_13'></span>

## Item 13 以对象管理资源
### 快速看点
+ 以对象管理资源常被称为资源取得时机就是初始化时机。
+ 不要总是觉得``new``出来的对象一定会有`delete`，特别是在别人接手以后。
+ `auto_ptr`和`shared_ptr`可以自动删除，其中`auto_ptr`指向的对象最多只被一个指针拥有，而`shared_ptr`可以计数。而且`shared_ptr`可以自定义deleter，但是`auto_ptr`不行
### 为什么需要使用RAII对象
假设有这样一段代码：
```C++
Investment *createInvestment(); //工厂模式
void f()
{
    Investment *pInv = createInvestment();
    ... // 使用这个指针
    delete pInv;
}
```
在编码阶段，由于这个函数肯定是由你负责，new出来的肯定会`delete`。但是有没有可能没有`delete`的时候？在代码接手以后需要特别注意这一点。
+ 提前`return`，无法触及到`delete`
+ `continue`/`break`跳出循环
+ `delete`之前发生异常

众所周知，源代码越少越好。为了能够防止手贱带来的影响，可以修改成这样
```C++
Investment *createInvestment() {...}
void f()
{
    std::auto_ptr<Investment> p(createInvestment());
    ...
}
```
这样在程序跳出的时候会自动析构`p`，也就会自动析构掉那个指针。存在异常？吞掉。

但是问题解决了吗？其实没有。详情见[Item 18](./Chapter4.md#Item_18)。
### `auto_ptr`和`shared_ptr`
如果一个普通指针和`shared_ptr`同时指向一个对象，`shared_ptr`不会把这个普通指针计入引用，在`shared_ptr`都不指向这个对象时，这个普通指针指向的对象也不存在。举例见下。
```C++
#include <bits/stdc++.h>
using namespace std;
class Obj
{
public:
    Obj(int x = 1) : x(x) { cout << "Obj Constructor" << endl; }
    ~Obj() { cout << "Obj Destructor" << endl; }
    int x;
};
int main()
{
    Obj *p = new Obj();
    cout << p->x << endl;
    shared_ptr<Obj> p2(p);
    cout << p2->x << endl;
    cout << p2.use_count() << endl;
    p2.reset();
    cout << p->x << endl;
    return 0;
}
```
除此以外，`shared_ptr`调用的析构函数是`delete`而不是`delete[]`，这可能造成内存泄露。
<span id='Item_14'></span>

## Item 14 在资源管理类中小心copy行为
### 快速看点
在资源管理类需要进行赋值的时候，建议做出以下一种选择之一，或者你知道自己在做什么的话可以另立门户。
+ 禁止复制。只需要`private`继承[`Uncopyable`](./Chapter2.md#Uncopyable)类即可，换句话说，让赋值运算符和复制初始化`private`。
+ 引用计数，类似于`shared_ptr`。注意`shared_ptr`可以再初始化或者reset成需要的析构函数，比如锁定资源的析构就是解锁。
+ 复制一份，换句话说，深拷贝。
+ 转移拥有权，比如`auto_ptr`。
### 除此以外
+ 事实上大部分类的复制都需要考虑这些问题，大部分对象都适用于这些解决方案，根据具体需要来决定选择哪一种解决方案。
+ 注意支持拷贝函数的泛型版本，并且注意模板化和特化的调用优先级，这不是简单的例子，而是混杂在很多初始化函数内部。
## Item 15 在资源管理类中提供对原始资源的访问
### 快速看点
+ 资源管理类应该提供取得对象对应资源的方法。
+ 对原始资源的访问可能需要转换。显式转换比较安全，而隐式转换比较方便。
### 具体一点
加入存在一个需要对类的指针访问的函数：
```C++
int f(Investment *p) {...}
```
然后你塞进去了一个`shared_ptr<Investment>`：
```C++
shared_ptr<Investment> p(createInvestment());
f(p);
```
这不久滑稽了吗？
其实修改很简单，加入一个get方法即可。
```C++
f(p.get());
```
这个时候你可能觉得我总是写一个`.get`太麻烦了，我自己定义一个。
然后你写了一个`Shared_ptr<Investment>`，并且加入了这个函数：
```C++
operator() const {return ptr;}
```
于是你就很开心的使用f(p)了。直到有一天……
```C++
Shared_ptr<Investment> p(createInvestment());
Investment* pp = p;
```
黑人问号.jpg。`.get`真香。实际上，`shared_ptr`重定义了`->`，可以直接调用`p->xxx()`或者`p->xxx`。
## Item 16 成对使用`new`和`delete`时要采取相同形式
### 快速看点
+ `new`了一个数组，`delete`就要带\[\]，`new`了一个元素，`delete`就不能带\[\]。
### 为什么
考虑一下C++的内存模型，假设一个Obj对象进行了密置排布，那么就会有这些东西：
+ 对于一个对象，就是一个Obj对象。
+ 对于一个对象数组，就是第一个元素是一个`(int)n`，表示数组的大小，之后才是一系列Obj对象。

<b>虽然C++没有强制要求编译器这么实现，但是很多都是这么做的。</b>

+ 对于一个对象调用`delete[]`，可能就会绕过前面四个字节，然后删除了很多很多个这样的元素，具体涉及到编译器是否优化，你的元素排布是怎么样的。

+ 对于一个数组调用`delete`，可能就只把前面`sizeof(Obj)`给析构了，然后内存泄漏。

实际上可以尝试一下，现代编译器比较牛批，对于简单的情况甚至会帮你析构掉，就算你忘了也没关系，不过还是要保持一个好习惯。
## Item 17 以独立语句将newed对象置入智能指针
### 快速看点
+ 对于资源管理类对象进行显式初始化，这不只是不能通过编译的问题，当这玩意在内存泄漏的时候，你还是得拆开来观察情况。
+ C++对于参数核算的方式在不同的编译器之间可能是不一样的。
+ 在能力范围之内，可以考虑合并语句，但是对于不知道的东西还是小心点为好。
### 除此以外
说句实话，这一个例子有点不明所以。

考虑一下本意相同的三个版本：
```C++
//声明
int priority();
void processWidget(shared_ptr<Widget> pw, int priority);
//版本1
processWidget(new Widget, priority());
//版本2
processWidget(shared_ptr<Widget>(new Widget), priority());
//版本3
shared_ptr<Widet> pw(new Widget);
procesWidget(pw, priority());
```
实际上版本1根本不能通过编译，因为`shared_ptr`是一个`explicit`，然后卒。