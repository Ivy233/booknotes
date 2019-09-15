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
## Item 9 绝不在构造和析构过程中调用virtual函数
## Item 10 令operator=返回一个<i>reference to</i> <b>*this</b>
## Item 11 在operator=中处理"自我赋值"
## Item 12 复制对象时勿忘其每一个成分