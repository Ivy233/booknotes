<style>
p{
    text-indent:2em;
}
</style>
# 构造/析构/赋值运算
## Item 5 了解C++默默编写并调用了哪些函数
众所周知，如果你不在类中声明某些函数，编译期会自动帮你书写一部分函数，比如说
```C++
class Empty {}
```
等价于
```C++
class Empty
{
public:
    Empty() {...}
    Empty(const Empty & rhs) {...}
    ~Empty() {...} //注意这里并非virtual
                   //也就是说需要多态继承的时候需要重新定义一个，覆盖掉
    Empty& operator=(const Empty &rhs) {...}
}
```
这个类
```C++
class BigInt
{
public:
    BigInt(const int &rhs) {...}
}
```
不会生成一个无实参构造函数，但是其他的copy构造函数，赋值运算符，和析构函数依旧会生成。如果意图不生成copy赋值函数，那么需要给这个class加入不允许copy的部分（指从实际意义上），比如：
>```C++
>template<class T>
>class NamedObject
>{
>public:
>   NamedObject(std::string &name, const T& value);
>private:
>   std::string &nameValue;
>   const T objectValue;
>};
>std::string newDog("Persephone");
>std::string oldDog("Satch");
>NamedObject p(newDog, 2);
>NamedObject s(oldDog, 36);
>p = s;
>```
看看这段代码会发生什么？编译器：我太难了。

除此以外还有一种情况也不会生成默认函数：private以后的derived calss，因为大多数情况需要调用base class的，但是因为这部分被private了，所以不允许调用。
## Item 6 若不想使用编译器自动生成的函数，就该明确拒绝
从上一个Item可以看到，private化一个构造函数可以屏蔽掉编译器默认的构造，赋值函数。事实上，private以后如果仅声明但是不定义，那么可以很好的屏蔽掉外部调用。

但是可能还有一个问题，friend和成员函数依旧可以访问。这时有一个比较好的解决办法就是把不希望被所有函数访问到的给包装起来，
```C++
class UnCopyable
{
protected:
    Uncopyable() {}
    ~Uncopyable() {}
private:
    Uncopyable(const Uncopyable &);
    Uncopyable &operator=(const Uncopyable &);
}
```
之后private继承即可，这样自己也无法访问父类定义的那些函数了。
## Item 7 为多态基类声明virtual析构函数
我们来看一个具体的例子：
```C++
class IPhone
{
public:
    IPhone();
    ~IPhone();
}
class IPhone7 {...}
class IPhone8 {...}
class IPhoneX {...}
```
这是典型的工厂模式，我们可以使用namespace包装以后外置一个函数来生成一个IPhone对象指针：
```C++
IPhone* getIPhone(const std::string &iphone) {...}
```
然后这个指针的静态类型就是IPhone，动态类型是不确定的，虽然变化不多。但是这种情况会有一个问题：析构是怎么析构的？

这里就会产生一个错误：以IPhone的形式析构一个IPhoneX对象（或者其他的同类对象）就会产生析构不完全的问题，造成内存泄露。所以这里需要给析构函数加入一个virtual。你甚至可以把析构函数变成一个pure virtual函数，用以强制要求定义析构函数，比如
```C++
    virtual ~IPhone() = 0;
```
但是还是要在外部进行一下定义，否则在真正析构的时候还是要调用到这里的，到时候就是Linker发出抱怨了。
## Item 8 别让异常逃离析构函数
如果析构函数发生了异常，那么会发生不可预料的后果，举个例子：
```C++
vector<BigInt> a;
```
这个数组中如果在之前产生了异常，那么发生异常之后的所有BigInt都无法得到析构。这个时候如果不希望异常逃离析构函数，应该用try-catch吞下这个异常。

在数据库中一个连接可能是连着的，也有可能已经关闭，这个时候建议加入一个是否已关闭的状态来表示，并且在析构的时候针对不同情况进行不同的处理要好很多。但是这并不意味着关闭连接函数会发生异常，但是我们能保证这不是重复关闭的人为产生的异常了，此时应该直接记录调用失败，try-catch语句少不了的。
## Item 9 绝不在构造和析构过程中调用virtual函数
我们试想一下，在构造函数中调用函数的对象身份是什么？应该是这个对象才对，而不是一个对象的指针。

我们构造或析构父类对象的时候，不可避免地会调用子类的对应函数。而此时如果在构造函数内调用一个virtual函数，那么显然调用的是父类的函数，此时virtual不再virtual。并不是说这绝对不允许使用，而是你可能期望这玩意调用子类的virtual函数，这样就不用传参数过去了。

C++为什么不允许你这么调用？因为在Derived class对象中，调用构造必然会优先调用Base class的，此时调用Derived意味着调用了没有初始化的东西。同理可以解释为什么不在析构函数中调用virtual函数。

解决方法自然也很简单：如果不能从Base往Derived调用，那么可以在Derived把Base需要的参数给传上去：
```C++
class Derived: public Base
{
    Derived(params): Base(params) {...}
};
```
## Item 10 令operator=返回一个<i>reference to</i> <b>*this</b>
对于赋值运算：
```C++
x = y = z = 15;
```
是显而易见的，实际上这是因为右结合律。
```C++
x = (y = (z = 15));
```
其中z = 15返回了一个int(15)，这样y才能按照我们希望的得到15的值。

对于其它类型的也很容易知道这个特点，其中最重要的原理是赋值运算operator=返回了这个数，否则无法继续赋值。

但是有一个地方很奇怪：在大多数的C++语法中，operator=返回了一个引用。此时我们来对比一下两段代码：
```C++
class Base
{
public:
    int x;
    Base() : x(0)
    {
        cout << "Constructor" << endl;
    }
    Base(const Base &a) : x(a.x)
    {
        cout << "Copy constructor" << endl;
    }
    Base &operator=(const Base &rhs)
    {
        cout << "Assign from Base" << endl;
        x = rhs.x;
        return *this;
    }
    Base &operator=(const int &_x)
    {
        cout << "Assign from int" << endl;
        x = _x;
        return *this;
    }
};
int main()
{
    Base a, b;
    b = b = a = 15;
    return 0;
}
```
对应的输出：
```
Constructor
Constructor
Assign from int
Assign from Base
Assign from Base
```
和去除引用的版本：
```C++
class Base
{
public:
    int x;
    Base() : x(0)
    {
        cout << "Constructor" << endl;
    }
    Base(const Base &a) : x(a.x)
    {
        cout << "Copy constructor" << endl;
    }
    Base &operator=(const Base &rhs)
    {
        cout << "Assign from Base" << endl;
        x = rhs.x;
        return *this;
    }
    Base &operator=(const int &_x)
    {
        cout << "Assign from int" << endl;
        x = _x;
        return *this;
    }
};
int main()
{
    Base a, b;
    b = b = a = 15;
    return 0;
}
```
对应的输出：
```
Constructor
Constructor
Assign from int
Copy constructor
Assign from Base
Copy constructor
Assign from Base
Copy constructor
```
差别就体现出来了：少了几个copy constructor减少了一定的时间复杂度。

那么综上所述，为什么返回了引用？
+ 对于返回Base而言，我们更希望返回引用，这是为了返回对象本身，而不是一个复制。这在operator+=之类的运算符上体现的更多。
+ 相比于返回对象，返回引用能减少一定的时间复杂度。
## Item 11 在operator=中处理"自我赋值"
说起这个想起了一段这样的swap的代码：
```C++
void swap(int &a, int &b)
{
    a ^= b;
    b ^= a;
    a ^= b;
}
```
写出这样的代码时，一定要注意a=b的情况，有可能这样的情况会让你直接加入彻夜调试大会#滑稽。

那么对于大多数类的copy函数，不得不考虑的是自己和自己拷贝。比如说
```C++
class Bitmap {...};
class Widget {
    ...
private:
    Bitmap *pb;
    ...
}
Widget& Widget::oeprator=(const Widget &rhs)
{
    delete pb;
    pb = new Bitmap(*rhs.pb);
    return *this;
}
```
如果这玩意发生了自我赋值，那么在删除pb的时候，rhs.pb也指向了不存在的内存地址。除此以外，如果new发生了异常但是delete没有，那么这个Widget的pb将会指向一块被删除的Bitmap。

同时解决这两个问题的方法其实很简单：只有在产生了需要的对象以后才删除之前的对象，之前的过程则保存在另外一个指针上。
```C++
Widget& Widget::operator=(const Widget& rhs)
{
    Bitmap *before = pb;
    pb = new Bitmap(*rhs.pb);
    delete before;
    return *this;
}
```
除此以外还有copy-and-swap技术，在保证一个没有异常的swap之上可以进行这样写：
```C++
Widget& Widget::operator=(Widget rhs)
{
    swap(*this, rhs);
    return *this;
}
```
可以说相当投机取巧，这个函数在传入形参的时候就会复制一份。有的时候不遵循一些建议也可以获得比较好的结果。

这里就可以看出作者比较厉害的地方了：他认为大部分代码都有可能抛出异常，这是其一；其二是他认为程序员（后续接手或者接口二次开发）才是C++的客户，而不是操作软件的用户进行非法操作。往往程序员的不理解才是更大的软件公敌。
## Item 12 复制对象时勿忘其每一个成分
在之前的基础上，似乎这里没有什么可以讲的了，只需要注意：
+ 深拷贝，或者你知道你在干什么（换一个指针接管之类的）
+ 注意Base class和Derived class的调用。
+ 不建议用copy构造函数实现copy assignment，或者反过来。如果有这方面需求，建议用第三个函数同时实现，然后将其private。