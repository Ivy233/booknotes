<style>
p{
    text-indent:2em;
}
</style>
# 基础议题
## Item 1 仔细区别pointers和references
-----
### 快速看点
+ pointers和references有不一样的形式，但是在C/C++中，处理方式基本是一样的，但是有一些细节需要注意。
+ pointers可以为空(nullptr)，但是references不可以为*(nullptr)。
+ 修改pointers的指向是允许的，但是不能修改references的指向，只能修改references捆绑对象的值。
+ operator[]最好返回一个reference，这样方便修改。
### 它们大致是一样的
在书籍<Effective-C++>中我们可以了解到C++编译器处理引用的方式都是指针（至少大部分是的），这样才让引用具有引用的特质，而且节约了代码量（毕竟偷懒才是进步的动力）。
```C++
void function(const int& x);
```
这个函数减少了一次复制，直接把东西传进去，用const保证其不会被修改。如果不信的话，可以观察编译出来的汇编代码。
### 这真的是一样的吗
但是既然差不多的话为什么还要引入这个符号呢？所以肯定有区别的。
1. pointers可以为空(nullptr)，但是references不可以为*(nullptr)。
```C++
char* pc = 0;   // 这是被允许的
char& rc = *pc; // 这是啥东西？
```
注意，这个结果是为定义。通过之前的实验可以知道：
+ 未定义的东西可以被编译器定义。
+ 不能指望每个编译器都是一样的定义。
+ 可以指望主流编译器定义这个，毕竟一只手都数的过来，而且主流编译器考虑的都是大众。
+ 建议严格要求自己。
由于有这一点保证，所以就会有这样的差别
```C++
void print(const double& rd)
{
    cout << rd;
}
void print(const double* pd)
{
    if(pd)
        cout << *pd;
}
```
这是因为rd必定绑定了一个double，但是pd不一定。

2. 修改pointers的指向是允许的，但是不能修改references的指向，只能修改references捆绑对象的值。这只需要一段代码就可以说明：
```C++
string s1("Nancy");
string s2("Clancy");
string &rs = s1;
string *ps = &s1;
rs = s2;    // 等价于s1 = s2
ps = &s2;   // 不等价于&s1 = s2
```
3. operator[]最好返回一个reference，这样方便修改，而不是返回一个指针。不服的看STL去，你想做玄冰400吗（笑）？
## Item 2 最好使用C++类型操作符
-----
### 快速看点
+ 推荐使用C++的操作符，因为类型更多而且会进行类型检查。
+
### C转型方式的问题
+ 几乎允许转换为任何类型，甚至允许把const去掉，或者把基类指针换到派生类指针。
+ 难以辨识，可能藏在任何地方。
### static_cast
```C++
int num1, num2;
// v1
double result = ((double) num1) / num2;
// v2
double result = static_cast<double>(num1) / num2;
```
很明显v2更容易看出强制转换，不论对于开发工具还是对于人类都是如此。

对于大部分的cast来说，都是采用的这种方式，最常见的就是这些情况：
+ 自有类型转换，如int->double
+ 继承体系中父类转换为子类

如果static_cast能应付所有情况的话，那也就失去了其意义，只是方便机器识别而已，完全没有安全性，所以其他三种转换便应运而生。
### const_cast
这种转换有以下几个用途，[cppreference](https://zh.cppreference.com/w/cpp/language/const_cast)：
+
### dynamic_cast
### reintepret_cast
## Item 3 绝对不要以多态(polymorphically)方式处理数组
----
### 快速看点
## Item 4 非必要不提供default constructors
---
### 快速看点