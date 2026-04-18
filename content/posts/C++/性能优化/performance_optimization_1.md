
+++
title = "C++性能优化总结一"
date = 2026-04-18
draft = false
categories = ["C++"]
tags = ["性能优化"]
+++

&emsp;本文为学习博览吴咏炜和李建忠老师 C++ 性能优化后的总结,本文主要总结模板相关的知识点, 感觉还是了解到了之前比较多没有意识到过或者已经忘了的知识点。

参考文章: [https://boolan.com/](https://boolan.com/)

#### 一、从模板的偏特化讲起
&emsp; 偏特化即对部分模板参数进行确定, 或者对部分模板参数进行约束。对于偏特化而言, 一个最经典的例子就是判断一个变量的类型是否为数组, 或者获取数组的 size, 如下所示

```javascript 
template<class T> struct IsArray
{
   static bool m_value{false};
};

template<class T> struct IsArray<T[]>
{
    static bool m_value{true};
};

template<class T, size_t N> struct IsArray<T[N]>
{
     static bool m_value{true};
};

//test
IsArray<int>::m_value    //false
IsArray<int[6]>::m_value //true         
```
先声明一个模板类让其默认非数组类型, 然后在特化两个数组类型形参的版本来判断数组类型。
同理还有判断是否为引用类型的,也都是先声明为一个默认模板, 然后再特化两个引用参数的返回 true。
还有一个关于模板特化的典型案例就是编译期求值,比如阶乘啥的, 如下所示:
```javascript 
template <int n>
struct Factorial
{
    static const int m_value = Factorial<n> * Factorial<n-1>::m_value;
};

template <>
struct Factorial<0>
{
    static const int m_value = 1;
};
//假如直接 std::cout<< Factorial<5> <<std::endl; 那么将直接输出 120          
```
再或者去除类型的 const
```javascript 
template<class T>
struct RemoveConst
{
    using type = T;
};

template<class T>
struct RemoveConst<const T>
{
    using type = T;
};

template<class T>
using RemoveConstT = typename RemoveConst<T>::type;
```

&emsp;函数模板是没有偏特化的, 假如要达到类似的效果可以借助函数重载或者函数对象模板的偏特化, 如下所示:
```javascript
template<class T> 
bool CheckIsArray(const T&)
{
    return false;
}

template<class T, size_t N> 
bool CheckIsArray(const T(&)[N] )
{
    return true;
}
//test
int array[6];
int value;
CheckIsArray(array); //return true
CheckIsArray(value); //return false 
```
函数模板整体上讲编译会优先匹配模板参数更复杂的那一个, 需要注意的是这里的形参是引用而不是指针, 假如是指针的话会类型推导的时候数组退化成指针。
#### 二、通透比较器
&emsp;C++标准库里面有比较大小的仿函数, std::less。它可能的实现如下所示:
```javascript
template<class T = void>
struct less
{
    constexpr bool operator()(const T& lhs, const T& rhs) const 
    {
        return lhs < rhs; 
    }
};
```
这个比较器可能存在一些问题, 比如将 std::string 与 const char* 进行比较的时候会就会出现类型转换, 就会多 std::string 的构造和析构, 此时效率就低了, 而 std::string 是直接可以和 const char* 进行比较的。所以, 这个时候就需要在次基础上特化出一个通透性比较器。它的实现如下:
```javascript
template<>
struct less<void>
{
    template<class T, class U>
    auto operator()(const T&& lhs, const U&& rhs) const 
    {
        return std::forward<T>(lhs) < std::forward<T>(rhs); 
    }

    //特殊标志, 类似于模板匹配
    typedef void is_transparent;
};
```
特化的比较器里面, operator() 是一个成员函数模板, 接受两个模板参数, 以转发引用的方式传递参数, 函数内通过完美转发保留参数值类别然后再进行比较。
##### 2.1 通透比较器的用途一关联容器
&emsp; 我自己目前为止使用关联容器的时候几乎没有指定过最后一个比较器参数, 全部都是使用标注库默认的, 考虑如下代码:
```javascript
std::map<std::string, int> name_age_mp
```
此时第三个默认模板参数是 std::less<std::string>. 考虑如下情况:
```javascript
 name_age_mp.find("Jim");
 name_age_mp.find("Xiaoming");
```
此时通过 find 成员函数查找对应键值时, 会先将 const char* 构造成 std::string, 效率就低了。如果按照如下声明:
```javascript
std::map< std::string, int, std::less<> > name_age_mp
```
此时 std::map::find 就会是一个通用性成员模板函数, 它可以接受所有可以直接和 std::string 进行比较的类型, 减少不必要的构造。
##### 2.2 通透比较器的用途一自带键的对象
&emsp; 这个的原理和前面的关联容器查找时降低不必要的构造类似, 比如试图放入集合 std::set 的对象自带键值,  考虑如下代码:
 ```javascript
struct Obj
{
    int id;
    const char* name;
};
std::set<Obj> s{{0, "zero"}, {1, "one"}, {2, "two"}};
```
假如此时直接使用 id 进行查找, 又会出不必要的构造, 所以可以实现一个通透性比较器来消除它, 如下所示:
 ```javascript
template <typename IdType>
struct id_compare 
{
    template <typename T, typename U>,
    bool operator()(const T& lhs, const U& rhs) const  { return lhs.id < rhs.id; },
    template <typename T>,
    bool operator()(const T& lhs, IdType rhs_id) const { return lhs.id < rhs_id; },
    template <typename T>,
    bool operator()(IdType lhs_id, const T& rhs) const { return lhs_id < rhs.id; },

    typedef void is_transparent;
};
```
可以看出重载了两个新的的 operator(), 分别用于自带的键值类型与 Obj 进行比较, 此时集合的声明如下:
 ```javascript
std::set<Obj, id_compare<int>> s{{0, "zero"}, {1, "one"}, {2, "two"}};
```
这里顺便记一下 C++ 新版的泛型lambda表达式, 它的原理就是前文提到的通用性比较器的模板类。泛型lambda比较器表达式如下:
 ```javascript
auto less_obj = [](auto&& lhs, auto&& rhs)
{
    return std::forward<decltype(lhs)>(lhs) < std::forward<decltype(rhs)>(rhs);
};
```
#### 三、降低模板二进制膨胀方面的优化
##### 3.1、模板类的优化
&emsp;模板二进制膨胀的原因是模板实例化后会可能会出现很多份相同的代码。比如模板类中存在与模板无关的成员函数, 此时每实例化一个类型就会产生一份相同的代码,类似于写了两个只有函数名不同，其余返回值,入参和实现完全相同的代码, 这就相当于在代码中重复写代码。
解决方案就是与模板无关的成员函数放到基类里面。如下所示:
 ```javascript
class BaseObj
{
public:
    void BufferOp();
};

class DerivedObj : private BaseObj
{
public:
    using BaseObj::BufferOp;
};
```
这里使用实现继承, 避免 BaseObj 指针可以转换成 DerivedObj 指针, 再使用 using 将其暴露出来。
##### 3.2、形参类型的优化
&emsp;当形参为 T&& 或者 auto&& 形式的转发引用时, 当传参为 const 左值, 非 const 左值和右值时会产生不同的实例话结果。通常而言, 出现 T&& 或者 auto&& 时, 都需要与完美转发 std::forward 一起配对出现。
所以, 当能确定传参是 T& 或者 const T& 时, 就不必使用转发应用。

#### 四、模板相关的一些知识点
##### 4.1、模板实例化机制
• 数据成员只要类型被使用，编译器会根据其数据成员、生成对应类型结构。
• 函数成员选择性实例化:
&emsp;  ①非虚函数，如果实际调用到，则会生成代码；如果没有调用到，则不生成。
&emsp;  ②虚函数，无论是否调用，总会生成代码。因为在运行时“有可能”调用到。
• 隐藏编译错误:
&emsp;  ①如果某些模板方法没有被调用，即使包含编译错误，也会被忽略。
&emsp;  ②强制实例化模板, 比如声明 template class MyTemplateClass<int>; 来强制要求编译所有 MyTemplateClass 的模板类函数成员, 排除所有编译错误，无论是否调用到。
• 优先使用using 而不是typedef
##### 4.2、using 和 typedef
&emsp;  ①两者都可以声明类型别名、成员类型别名
&emsp;  ②using 可以定义模板别名，而typedef不可以
&emsp;  ③可以免掉类型内typedef要求的typename前缀 , 和::type 后缀。
##### 4.3、模板参数类型自动推导
• C++模板编译时支持对类型参数的自动推导：
&emsp; ①普通函数参数; ②类成员函数参数; ③构造函数参数（C++ 17 支持），模板所有类型参数都有值。
• 模板类型推导时：
&emsp; ①引用性被忽略：引用类型实参被当作非引用类型处理。
&emsp; ②转发引用时，左值参数按左值、右值参数按右值。
&emsp; ③按值传递时，实参中const/volatile修饰会被去掉。
 ```javascript
    int data1=100;
    int& data2=data1;
    const int data3=data1;
    const int& data4=data1;

    //template<typename T> 
    //void process1(T& value);
    process1(data1);//  T是int, value 是int&
    process1(data2);//  T是int, value 是int&
    process1(data3);//  T是const int, value 是const int&
    process1(data4);//  T是const int, value 是const int&

    //template<typename T>
    //void process2(T value)
    process2(data1);// T 是int, value 是 int
    process2(data2);// T 是int, value 是 int **********
    process2(data3);// T 是int, value 是 int **********
    process2(data4);// T 是int, value 是 int **********
```
##### 4.4、模板特化
&emsp;模板类型的特化指的是允许模板类型定义者为模板根据实参的不同，而定义不同实现的能力。 ①特化版本可以和一般版本拥有不同的外部接口和实现。②模板偏特化：也可以对部分模板参数进行特化定义, 称为偏特化。③模板特化支持模板类、模板函数。