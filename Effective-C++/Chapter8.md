# 定制`new`和`delete`
<span id='Item_49'></span>

## Item 49 了解 `new_handler` 的行为
-----
### 快速看点
+ `set_new_handler`允许用户替换掉这个类的构造出现问题的时候的应对方式，当然给了这么大的权力也应当学会使用。
+
### `set_new_handler`
在`operator new`出现问题的时候，它会抛出异常，也有可能返回一个`null`指针。在抛出异常之前，它会先调用一个用户指定的错误处理函数，这被称之为`set_new_handler`操作。在`std`中存在这样的声明：
```C++
namespace std{
    typedef void (*new_handler)();
    new_handler set_new_handler(new_handler p) throw();
}
```
对于替换掉的方式其实也很简单:
```C++
void outOfMem(){
    std::cout<<"Unable to alloc memory";
    std::abort();
}
int main(){
    std::set_new_handler(outOfMem);
    int *p = new int[100000000L];
}
```
在`new`没弄到足够的内存以满足需要的时候，它会不断调用`outOfMem`，这也被称为global handler。

一个设计良好的`set_new_handler`应当做以下事情的一件或者多件：
+ 使更多内存可用。
+ 安装另一个`set_new_handler`。
+ 卸载`new_handler`。这样下一次就可以让编译器知道无力应对，然后丢异常。
+ 抛出`bad_alloc`（或者相关子类）的异常。
+ 不返回，通常调用`abort`或者`exit`。
### class相关错误处理
对于外面的可以这么处理，那对于类里面的呢？
```C++
class Widget {
public:
    static std::new_handler set_new_handler(std::new_handler p) throw();
    static void * operator new(std::size_t size) throw(std::bad_alloc);
private:
    static std::new_handler current;
};

//你可以试试在里面定义#滑稽
std::new_handler Widget::current = 0;
std::new_handler Widget::set_new_handler(std::new_handler p) throw() {
    std::new_handler old = current;
    current = p;
    return old;
}
//但是这样还是没有使用过set_new_handler，所以需要重载一下new，并且在其中换掉这个handler
void * Widget::operator new(std::size_t size) throw(std::bad_alloc){
    NewHandlerHolder h(std::set_new_handler(current));
    return ::operator new(size);
}
// 同样的，如果函数在外面的话
void outOfMem();
Widget::set_new_handler(outOfMem);
```
### 每次写这么多好烦
那就包装一下。
```C++
template<typename T>
class NewHandlerSupport{
public:
    static std::new_handler set_new_handler(std::new_handler p) throw();
    static void * operator new(std::size_t size) throw(std::bad_alloc);
private:
    static std::new_handler current;
};

template<typename T>
std::new_handler NewHandlerSupport<T>::current = 0;

template<typename T>
std::new_handler NewHandlerSupport<T>::set_new_handler(std::new_handler p) throw(){
    std::new_handler old = current;
    current = p;
    return old;
}

template<typename T>
void * NewHandlerSupport<T>::operator new(std::size_t size) throw(std::bad_alloc){
    NewHandlerHolder h(std::set_new_handler(current));
    return ::operator new(size);
}
```
然后用奇怪的调用方式：
```C++
class Widget: public NewHandlerSupport<Widget>
{
    //该干啥干啥。
};
```
我定义自己#滑稽。但是实际上没啥大问题，因为里面都是静态成员，反正定义了也是覆盖，还都是一样的。
### no throw new
几乎所有的软硬件厂商都要面临一个问题：我开发了一套新的接口，速度更快更安全，但是老的我不能删掉，不然客户就要投诉了，还都是大客户或者大量小客户。所以C++还是要支持`nothrow`的`new`，于是乎就有了这些东西。
```C++
Widget *p1 = new Widget;
assert(p1 != 0);
Widget *p2 = new (std::nothrow) Widget;
if(p2 == 0) // 失败的时候进入if
{
    ...
}
```
## Item 50 领会何时替换 `new` 和 `delete` 才有意义
-----
### 快速看点
+ 有很多定制`new`和`delete`的理由，包括改进性能，调试错误，收集堆用法的信息。
+ 定制一个`new`很容易，但是定制一个好的`new`非常困难。在你有足够的理论和demo之前，最好不要随意触摸这个直接接触机器的领域，除非你想蓝屏。
### 一个定制的`new`
其实定制一个`new`不算很困难，比如我们可以随意实现一个边界检查的`new`。
```C++
static const int signature = 0xDEADBEEF;    // 边界符
typedef unsigned char Byte;

void* operator new(std::size_t size) throw(std::bad_alloc) {
    // 多申请一些内存来存放占位符
    size_t realSize = size + 2 * sizeof(int);

    // 申请内存
    void *pMem = malloc(realSize);
    if (!pMem) throw bad_alloc();

    // 写入边界符
    *(reinterpret_cast<int*>(static_cast<Byte*>(pMem)+realSize-sizeof(int)))
        = *(static_cast<int*>(pMem)) = signature;

    // 返回真正的内存区域
    return static_cast<Byte*>(pMem) + sizeof(int);
}
```
这个方法有这么几个操作：
+ 先弄出一片内存。如果弄不出来就报错了。
+ 然后写入边界符。
+ 最后返回真正的内存区域。
### 这个`new`的问题
这个`new`有问题吗？有，问题大了。
+ [Item 49](./Chapter8.md#Item_49)提到了这个`new`应当由`set_new_handler`的调用，不然出了问题连抢救都不抢救？
+ 针对硬件平台的特性要求太高，而且符合条件的太少。大多数情况下，`double`的起始地址都是8的倍数，在这种情况下，不能保证`double`的起始地址是8的倍数，说不定就是`% 8 = 4`了，对于amd64的CPU其实还好，大不了慢一点，但是对于某些对齐要求很高的呢？
### 更好的解决方案
+ 多阅读技术手册提升驾驭这玩意的能力。
+ 别人花了这么久的时间写的东西还是不得不用。
+ 开源的建议去看看。
### 我们为什么定制`new`和`delete`
+ 检测信息。这些信息包括了使用错误，内存分配的数据和使用的统计数据。
+ 提升速度与减小成本。和通用的`swap`类似，通用的不一定快。花费出去的力气总归还是要有点回报的。
+ 聚集类的对象。这是不太容易想到的一点，减小跳页的开销也可以提升速度。
+ 你知道你要干啥，那就去做，相信你有驾驭的能力。
## Item 51 编写 `new` 和 `delete` 时要遵守惯例
----
### 快速看点
+ `new`需要注意的细节很多，但是`delete`需要注意的细节比较少。
+ `new`的方式和`delete`的方式建议配套，除非你知道自带的干了些什么。
### `operator new`(non-member)
之前讲解的原因只是一个编写的动机，但是没有讲你可能遇到的坑。
+ 给出返回值很容易，返回申请到的合法地址，如果给不出来就返回空或者丢一个异常。
+ 每次失败以后都要调用`set_new_handler`，在function为空的时候，抛出一个异常。
+ 如果请求零字节，也要返回一个指针，来简化语言的其他部分。
```C++
void * operator new(std::size_t size) throw(std::bad_alloc){
    if(size == 0) size = 1;
    while(true){
        // 以下内容可以换成其他自定义的分配方式
        void *p = malloc(size);
        // 申请成功
        if(p) return p;
        // 申请失败，获得new handler
        new_handler h = set_new_handler(0);
        set_new_handler(h);
        // 如果能跑出来就运行，反之丢出一个异常。
        if(h) (*h)();
        else throw bad_alloc();
    }
}
```
只不过对于第三个的处理可能有点奇怪：我们把零字节的东西当作一个`char`来处理，说来也不是很亏，因为零字节的东西也不是很多，用这点空间来换编码时少考虑一种特殊情况时很值得的。

除此以外还需要注意的是这里的两个`set_new_handler`，这是因为你没办法直接得到handle函数的指针，所以必须调用`set_new_handler`来知道这到底是个啥。子多线程环境中，你可能需要把这个隐藏的更深，否则一个死锁就。。。

而且这里的while是不会停下的，除非throw或者return了，所以一定要对这里负责，小心的处理。
### `operator new`(member)
你可能会认为经过了non-member的话，难关就结束了，其实不是，还早得很。
+ 继承问题。你写的`operator new`考虑过继承了吗？或者你设计的就是为了这个类和所有子类用的。
+ 类对象的对齐问题。

解决办法其实也不难，但是一定要想清楚。
```C++
class Base{
public:
    static void* operator new(std::size_t size) throw(std::bad_alloc);
};
class Derived: public Base{...};

Derived *p = new Derived;

void *Base::operator new(std::size_t size) throw(std::bad_alloc){
    if(size != sizeof(Base)) return ::operator new(size);
    // 这里该干啥干啥
}
```
注意这里不需要检查`size == 0`。因为C++总是会给空对象插入一个`char`，所以你永远不会得到一个size为0的对象。

如果你想对一个类对象的数组处理，你可能需要花费更多的功夫。
+ 每一个对象的实际大小是多少？
+ 数组的大小是多少？
+ 对象之间的差距有多少？
### `operator delete`
相对于`new`而言，`delete`是很简单的，只需要保证删除空指针的问题就行了。

那么问题来了，为什么比较简单？
+ `new`是把一个不是属于自己的东西内化。
+ `delete`是把一个符合自己规范的东西扔掉。

你说是借东西容易还是还东西容易？
```C++
// delete(non-member)
void operator delete(void *rawMem) throw(){
    if(rawMem == 0) return;
    // 释放内存
}
// delete(member)
class Base{
public:
    static void * operator new(std::size_t size) throw(std::bad_alloc);
    static void operator delete(void *rawMem, std::size_t size) throw();
};
void Base::operator delete(void *rawMem, std::size_t size) throw(){
    if(rawMem == 0) return; // 这是不是空指针
    if(size != sizeof(Base)){
        ::operator delete(rawMem);  // 如果不是就和new一样的方法删除，
                                    // 如果new在这里和之前不一样，
                                    // 这里也要跟着改变
    }
    // 释放内存，并且这是一个Base对象。
}
```
## Item 52 如果编写了placement new，就要编写placement delete
----
### 快速看点
+ 在分配内存的时候，应当对于类对象附上初值，在C++中，这由两个函数完成，第一个为`new`，第二个placement new。
+ `new`和`delete`应当成对出现，并且
### 调用了哪些new？
```C++
Widget *p = new Widget;
```
这行代码很显然调用了两个，一个是缺省的构造函数，一个是placement new来分配内存。如果第二个出现了异常，那么第一个的结果应当回退掉，同时返回异常。但是有一个问题：客户不可能清理这个指针，因为他连这个指针指向哪里都不知道（异常发生的时候不会正常返回），所以这个事情智能落在`CRT`身上。
### 对应关系
在第一步中调用了缺省的构造函数应该是这样的：
```C++
void* operator new(std::size_t) throw(std::bad_alloc);
```
那么对应的operator `delete`是这样的：
```C++
void operator delete(void *rawMemory) throw(); // global中的典型声明
void operator delete(void *rawMemory, std::size_t size) throw(); // class中的典型声明
```
### placement new
理论上对于`new`而言只有一个函数，但是如果我们希望在`new`的同时对于其他东西进行操作（比如日志），这个时候你就需要placement new，与此同时，需要一个对应的placement delete。
除了这个之外呢，顺手了解一下placement这个名称的来历：
>`new` 的这个版本是 C++ 标准库的一部分，只要 #include \<new\> 你就可以访问它。需要指出，这个 `new` 用于 `vector` 内部，在 `vector` 的尚未使用的空间内创建 objects。它也是最初的 placement new。实际上，这就是这类函数被称为 placement new 的来历。这就意味着术语 "placement new" 被赋予了更多的含义。大多数情况下，当人们谈到 placement new，他们谈的就是这个特定的函数，持有一个 void* 类型的额外参数的 `operator new`。较少情况下，他们谈的是持有额外参数的 `operator new` 的任意版本。根据上下文通常可以搞清楚任何暧昧，重要的是要了解到通用术语 "placement new" 意味着持有额外参数的 `new` 的任意版本，因为短语 "placement delete"（过一会儿我们就会遇到它）直接起源于它。

但是如果我们把一个类写成这样的话：
```C++
class Widget {
public:
    ...
    static void* operator new(std::size_t size, std::ostream& logStream)
        throw(std::bad_alloc);
    static void operator delete(void *pMemory, std::size_t size) throw();
    ...
};
```
这个类其实是有问题的，那么为什么有问题呢？
我们来这样调用：
```C++
Widget *p = new(std::cerr) Widget;
```
这样会在`cerr`中记录错误信息。如果说发生异常的话，我们依旧不能调用`delete`，因为指针指向哪里都不知道。幸运的是`CRT`会帮助我们销毁这个东西，但是条件是必须找到和`new`持有除`size_t`参数以外所有参数类型都一样的函数。但是这个时候`Widget`并没有提供，于是乎存在一个这样的错误：`CRT`什么都不做，然丢了一个异常，于是乎内存发生了泄露。

解决方法呢？在解决这个问题之后顺便加上一个标准的
```C++
class Widget {
public:
    ...
    static void* operator new(std::size_t size, std::ostream& logStream)
        throw(std::bad_alloc);
    static void operator delete(void *pMemory) throw();
    static void operator delete(void *pMemory, std::ostream& logStream)
        throw();
    ...
};
```
### 名称隐藏
和[Item 33](./Chapter6.md#Item_33)类似，类中的名称会隐藏外部的名称，子类的名称会隐藏父类的名称，于是乎
```C++
class Base{
public:
    static void* operator new(std::size_t size, std::ostream& log) throw(std::bad_alloc);
};
Base *p = new Base;
Base *p = new (std::cerr) Base;
```
```C++
class Derived: public Base{
public:
    static void* operator new(std::size_t size) throw(std::bad_alloc);
};
Derived *p = new (std::clog) Derived;
Derived *p = new Derived;
```
这两行代码都会发生错误。
### 一种解决办法
行行行，类继承牛皮，我把东西都写在里面，想用的时候拿出来就行。
```C++
class StandardNewDeleteForms {
public:
    static void* operator new(std::size_t size) throw(std::bad_alloc)
    { return ::operator new(size); }
    static void operator delete(void *pMemory) throw()
    { ::operator delete(pMemory); }

    static void* operator new(std::size_t size, void *ptr) throw()
    { return ::operator new(size, ptr); }
    static void operator delete(void *pMemory, void *ptr) throw()
    { return ::operator delete(pMemory, ptr); }

    static void* operator new(std::size_t size, const std::nothrow_t& nt) throw()
    { return ::operator new(size, nt); }
    static void operator delete(void *pMemory, const std::nothrow_t&) throw()
    { ::operator delete(pMemory); }
};
```
然后就可以用`using`指明需要这个就可以用了，注意是所有的哦：
```C++
class Widget: public StandardNewDeleteForms {
public:
    using StandardNewDeleteForms::operator new;
    using StandardNewDeleteForms::operator delete;

    static void* operator new(std::size_t size, std::ostream& logStream)
        throw(std::bad_alloc);

    static void operator delete(void *pMemory, std::ostream& logStream)
        throw();
};
```