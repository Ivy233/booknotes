<style>
p{
    text-indent:2em;
}
</style>
# 设计与声明
## Item 18 让接口容易被正确使用，不易被误用
----
### 快速看点
+ 如题，接口不止你一个人使用，但是仅仅通过代码的交流不能让编写者和使用者心意交通。
+ 接口的正确使用包括了限定合法输入。
+ 不易被误用甚至包括了同一系列的代码产品中的同类功能调用，例如STL。
+ shared_ptr是一个好东西，可以自动解锁互斥体，防止cross-DLL问题。
### 来看一个例子
```C++
class Date {
public:
  Date(int month, int day, int year);
  ...
};
```
这样设计很容易进入一个误区：
```C++
Date d(30, 3, 1995);//你可以这样
Date d(2, 20, 1995);//你也可以这样
```
虽然你希望输入的是1995.3.30和1995.2.20，但是谁会禁止你这么做呢？
### 修改一下实现
```C++
struct Day {            struct Month {                struct Year {
  explicit Day(int d)     explicit Month(int m)         explicit Year(int y)
  :val(d) {}              :val(m) {}                    :val(y){}

  int val;                int val;                      int val;
};                      };                            };

class Date {
public:
 Date(const Month& m, const Day& d, const Year& y);
 ...
};
```
再来看一下这次的调用
```C++
Date d(30, 3, 1995);                    // 编译不通过，因为explicit
Date d(Day(30), Month(3), Year(1995));  // 编译错误，因为类型不对
                                        // 而且没提供对应的初始化函数
Date d(Month(3), Day(30), Year(1995));  // 这样才能过去
```
但是
```C++
Date d(Month(13), Day(30), Year(1995));  // 黑人问号
```
### 合法性
于是乎我们再次修改一下
```C++
class Month {
public:
  static Month Jan() { return Month(1); }   // 这就是1月
  ...                                       // 以此类推
  ...                                       // 其他的方法

private:
  explicit Month(int m);                    // 阻止自定义月
  ...                                       // 其他需要的东西
};
```
这样是不是好多了？调用也显而易见。
### 之前的例子
```C++
Investment* createInvestment();
```
如果你还记得这个例子的话，当时引入了shared_ptr和auto_ptr，看上去像是给这两个东西打广告一样#滑稽。但是实际上在调用的时候，调用者有可能忘记把这玩意包装到一个shared_ptr里面，所以
```C++
shared_ptr<Investment*> createInvestment();
```
这下你还忘记？

除此以外，这样还有一个好处：我们在内部可以自定义一些内容，比如说加入自定义的析构函数，这样调用者也不可能说忘记导致的资源泄露。甚至根据这一点还可以继续改进：
```C++
std::tr1::shared_ptr<Investment> createInvestment()
{
    std::tr1::shared_ptr<Investment> retVal(
        static_cast<Investment*>(0),
        getRidOfInvestment
    );
    retVal = ... ;
    return retVal;
}

std::tr1::shared_ptr<Investment> createInvestment()
{
    std::tr1::shared_ptr<Investment> retVal(
        static_cast<Investment*>(...),
        getRidOfInvestment
    );
    return retVal;
}
```
这两个的差别在哪里呢？一个可能在初始化上面多花费一点时间，但是另外一个有可能会带来一些不好追踪的bug。只是

为什么不把return也给合并掉呢？这样不就又省下一行代码了？
### cross-DLL问题
具体定义就是一个对象在某个dll中产生，但是在另外一个dll中删除。shared_ptr可以比较好的解决这类问题。因为它的缺省的deleter只将delete用于这个shared_ptr被创建的DLL中
## Item 19 设计class犹如设计type
---
### 快速看点
## Item 20 宁以pass-by-reference-to-const替换pass-by-value
---
### 快速看点
---
## Item 21 必须返回对象时，别妄想返回其reference
---
### 快速看点
---
## Item 22 将成员变量设计为private
---
### 快速看点
---
## Item 23 宁以non-member、non-friend替换member函数
---
### 快速看点
---
## Item 24 若所有参数皆需类型转换，请为此采用non-member函数
---
### 快速看点
---
## Item 25 考虑写出一个不抛异常的swap函数