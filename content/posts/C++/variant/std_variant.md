+++
title = "C++ std::variant 总结"
date = 2026-04-18
draft = false
categories = ["C++"]
tags = ["std::variant"]
+++

&emsp;本文来记录一下对C++标准库的 std::variant (带标签的联合体)用法的总结。参考文章: [hhttps://boolan.com/](https://boolan.com/)

#### 一、std::variant 的由来
&emsp;  根据 [cppreference](https://boolan.com/) 官网的概念, std::variant 是类型安全的共用体。 C语言里面共用体是把不同类型的数据成员存储在同一块内存区域上, 它与结构体不一样, 结构体的每个成员都存储在独立的地址空间内, 所以对任何成员的写操作都不改变其他成员的值, 访问的时候也是基于基址偏移的方式进行读操作。
&emsp; 共同体 **union** 改变任意一个成员的二进制值, 其余成员的值都会发生变化。考虑如下代码:
```javascript 

union TestUnion {
    int     m_int_value;
    float   m_float_value;
};

int main() {

    TestUnion test_un;
    test_un.m_int_value = 666;
    std::cout << "Integer Value: " << test_un.m_int_value << "\n";
    
    test_un.m_float_value = 3.14f;
    
    std::cout << "Float Value: " << test_un.m_float_value << "\n";
    std::cout << "Integer Value: " << test_un.m_int_value << "\n";
    
    return 0;
}   
```
&emsp;此时输出如下所示, 当对以上代码中的浮点数成员进行修改后, 对应整数成员的二进制发生了变化, 此时访问整数类型成员是无效的, 但是访问整数型成员的没有任何警告或者异常。

>Integer Value: 666
Float Value:  3.14
Integer Value: 1078523331


&emsp; 除此之外还有一个问题, 那就是 C++ 里面有很多复杂类型, 当我们向共用体里新增复杂类型后, 维护起来就比较困难了。考虑如下代码:

```javascript 
union TestUnion {
    int     m_int_value;
    float   m_float_value;
    std::string m_string_value;
};
```
当向共用体里面添加一个 std::string 类型时, 直接编译不通过. 编译器无法判断该如何构造 std::string, 析构的时候该不该析构 std::string 成员, 所以它就方了. 原先存的都是POD类型,它们是没有构造和析构调用的, 如果想要编译通过就得手动维护共用体的构造和析构函数, 这就复杂了。 

综上所述, 需要引入 std::variant.

#### 二、std::variant 用法简介
##### 2.1、类型安全保证
&emsp;先来看看它的类型安全保证, 代码如下:
```javascript 
int main() {
    
    std::variant<int, float> test_variant;

    test_variant = 10;
    std::cout << std::get<int>(test_variant) << std::endl;
    
    test_variant = 10.3f;
    std::cout << std::get<float>(test_variant) << std::endl;
    
    std::cout << std::get<int>(test_variant) << std::endl;
    return 0;
}
```
输出如下:
> 10
10.3
terminate called after throwing an instance of 'std::bad_variant_access'
  what():  std::get: wrong index for variant
Aborted

写入浮点数后再访问整数类型成员, 直接抛出了异常, 这提供了一个类型安全的保证。除此之外, 它能自动维护C++ 当中的复杂类型,也就是构造和析构函数的调用。
```javascript 
int main() {
    
    std::variant<int, float, std::string> test_variant;

    test_variant = 10;
    std::cout << std::get<int>(test_variant) << std::endl;
    
    test_variant = 10.3f;
    std::cout << std::get<float>(test_variant) << std::endl;
    
    test_variant = "test_variant";
    std::cout << std::get<std::string>(test_variant) << std::endl;
    return 0;
}
```
>10
10.3
test_variant

##### 2.2、访问 std::variant
&emsp;这里提供两种访问方式, 一种是预先的类型检测, 一种是采用指针的方式.
```javascript 
int main() {
    
    std::variant<int, float, std::string> test_variant;

    test_variant = "test_variant";
    if (std::holds_alternative<int>(test_variant) ){
        std::cout<<"hold int:" << std::get<int>(test_variant) <<std::endl;
    }else if(std::holds_alternative<float>(test_variant) ){
        std::cout<<"hold float:" << std::get<float>(test_variant) <<std::endl;
    }else if(std::holds_alternative<std::string>(test_variant) ){
        std::cout<<"hold std:;stirng:" << std::get<std::string>(test_variant) <<std::endl;
    }
}
```
>hold std:;stirng:test_variant

基于指针的方式,代码如下:
```javascript 
    std::variant<int, float, std::string> test_variant;
    test_variant = 100.f;
    if (auto ptr = std::get_if<int>(&test_variant) ){
        std::cout<<"hold int:" << *ptr <<std::endl;
    }else if(auto ptr = std::get_if<float>(&test_variant) ){
        std::cout<<"hold float:" << *ptr <<std::endl;
    }else if(auto ptr = std::get_if<std::string>(&test_variant) ){
        std::cout<<"hold std:;stirng:" << *ptr <<std::endl;
    }
```
>hold float:100

[cppreference](https://boolan.com/) 里面还提供了一种更牛逼的遍历方式, 代码如下:
```javascript 

using var_t = std::variant<int, long, double, std::string>;

/*
这里首先是声明一个可变模板参数的函数模板 overloaded, 再显示的推导为 overloaded 结构体
overloaded 也是一个可变模板参数的模板结构体, 通过继承所有模板参数, 再结合 using Ts::operator()...
实现对基类, 也就是模板参数中的 operator() 运算符的声明。 

此时就可以根据 vec 列表的成员类型调用合适的 lambda 表达式了
*/

template<class... Ts>
struct overloaded : Ts... { using Ts::operator()...; };
template<class... Ts>
overloaded(Ts...) -> overloaded<Ts...>;


int main()
{
    std::vector<var_t> vec = {10, 15l, 1.5, "hello"};
    for (auto& v: vec) {
        std::visit(overloaded{
            [](auto arg) { std::cout << arg << ' '; },
            [](double arg) { std::cout << std::fixed << arg << ' '; },
            [](const std::string& arg) { std::cout << std::quoted(arg) << ' '; }
        }, v);
    }
}

```

#### 三、std::variant vs OO 
&emsp; std::variant 在某种情况下可以作为多态的一种替换选择。

```javascript 

namespace{
  int sum_count = 0;  
  const int kShapeCount {50000};
};

class Shape
{
public:
    virtual ~Shape() =default;
    virtual void DoSomething() = 0;
};

class Cricle:public Shape
{
public:
    void DoSomething() override;
};

class Rectange:public Shape
{
public:
     void DoSomething()override;
};

class CricleEX
{
public:
    void DoSomething(){};
};

class RectangeEX
{
public:
     void DoSomething(){};
};

int main() {

    std::vector<std::variant<CricleEX, RectangeEX> > ex_shape_vecs;
     for(int i{0}; i<kShapeCount; i++){
        ex_shape_vecs.emplace_back(CricleEX() );
        ex_shape_vecs.emplace_back(RectangeEX() );
     }
    
    auto t1 = std::chrono::steady_clock::now();
    for(auto& itor: ex_shape_vecs){
        std::visit([](auto& obj){obj.DoSomething();}, itor);
    }
    
    auto t2 = std::chrono::steady_clock::now();
    std::cout << "std::varint cost:" <<std::chrono::duration_cast<std::chrono::duration<double, std::milli>>(t2-t1).count() <<"ms" <<std::endl;
    

    std::vector<std::unique_ptr<Shape> > shape_vecs;
    for(int i{0}; i<kShapeCount; i++){
        shape_vecs.emplace_back(std::make_unique<Cricle>() );
        shape_vecs.emplace_back(std::make_unique<Rectange>() );
    }
    
    
    auto t3 = std::chrono::steady_clock::now();
    for(const auto& itor: shape_vecs){
        itor->DoSomething();
    }
    auto t4 = std::chrono::steady_clock::now();
    
    std::cout << "OO cost:" <<std::chrono::duration_cast<std::chrono::duration<double, std::milli>>(t4-t3).count() <<"ms" <<std::endl;
    
    std::cout<<"sumcount:"<<sum_count<<std::endl;
    
    
    return 0;
}
```
>std::varint cost:7.94071ms
OO cost:4.19324ms
sumcount:500000

很奇怪，这里我得到的结论是面向对象的方式要比 std::variat 效率高, 这与吴咏伟老师在课堂上的当时出现了相反的结论, 这个难道是编译优化选项有啥不同哇, 大家觉得呢?