
+++
title = "C++性能优化总结二"
date = 2026-04-18
draft = false
categories = ["C++"]
tags = ["性能优化"]
+++

&emsp;书接上回, 这篇文章继续来谈谈C++性能优化相关的内容。参考文章: [https://boolan.com/](https://boolan.com/)

#### 一、返回对象
&emsp; 在函数输出变量值时尽量使用返回值的方式输出而非输出参数, 优点是代码直观方便阅读而且性能高, 没有不必要的拷贝发生。某些重载的运算符可以在一行里面写出来,比如：
```javascript 
std::string A, B, C;
std::string D = A+B+C;        
```
##### 1.1、返回值优化
&emsp;考虑如下代码:

```javascript 
struct A
{
    A(){std::cout<<"create obj"<<std::endl;}
    ~A(){std::cout<<"destory obj"<<std::endl;}
    A(const A& that){std::cout<<"copy create obj"<<std::endl;}
    A(A&& that){std::cout<<"move create obj"<<std::endl;}    
};
A GetObj()
{
    A a;
    return a;
}

int main()
{
    auto a = GetObj();
    return 0;
}       
```
此时输出为 "create obj" "destory obj"。我在刚毕业工作的时候以为是有拷贝构造要发生的, 当时还不晓得有返回值优化这个东东。直观上讲就是编译器在编译的时候直接就知道该在何处构造。但是当它无法确定的时候就会发生移动构造了, 考虑如下代码:
```javascript 
struct A
{
    A(){std::cout<<"create obj"<<std::endl;}
    ~A(){std::cout<<"destory obj"<<std::endl;}
    A(const A& that){std::cout<<"copy create obj"<<std::endl;}
    A(A&& that){std::cout<<"move create obj"<<std::endl;}    
};

A GetObjMove()
{
    A a1;
    A a2;
    if (rand()> 100){
        return a1;
    }else{
        return a2;
    }
}

int main()
{
    auto a = GetObjMove();
    return 0;
}       
```
此时输出为 "create obj" "create obj" "move create obj"  destory obj" destory obj" destory obj" 了。
所以通常情况下直接返回对象。有些例外情况不该返回值类型比如:
&emsp;  ①非值类型, 比如派生类对象, 这个时候就借助 std::unique_ptr 或者 std::shared_ptr来返回。
&emsp;  ②移动的开销很大, 考虑将其分配在堆上, 借助 std::unique_ptr 来返回。或者传递非const 的目标引用形参来修改。
&emsp;  ③函数内会重用一个带容量的对象, 这个时候也建议使用非const 的目标引用形参来修改。


#### 二、异常之得失
&emsp;使用C++异常机制它的优点是可以集中处理一些代码错误, 在不使用异常的时候写代码假如要返回错误类型,通常需要先在头文件里面用宏或者枚举定义各种错误值, 当出现异常的时候再一层一层的返回, 这期间通常有较多的 if_else。而且有时会因为疏忽导致某些错误类型被忽视, 在调试的时候假如没有发现, 就相当于代码埋雷了。所以用了异常机制后, 代码会更简洁, 易读, 可调式。

##### 2.1、避免不必要的 try...catch
&emsp; 在实现智能指针的时候可能会出现如下代码:
```javascript
    if (ptr){
        try{
            m_shared_cnt = new SharedCount();
        }
        catch(const std::bad_alloc& e){
            delete ptr;
            std::throw;
        }        
    }
```
假如引用计数模块内存申请失败了, 则可以需要把用户传进来的原生指针释放掉, 否则会内存泄漏。这个逻辑没有问题, 但是这里可以用更好的方式来保证避免内促泄漏, 比如C++最重要的特性RAII。
```javascript
    if (ptr){
        std::unique_ptr temp_ptr(ptr);
        m_shared_cnt = new SharedCount();
        temp_ptr.reset();      
    }
```
或者使用 defer, 这里有我之前写的文章链接:[https://blog.csdn.net/PX1525813502/article/details/127834364](https://blog.csdn.net/PX1525813502/article/details/127834364) 
当然引入异常机制也会带来一些问题, 比如会发生代码膨胀, 大约5%~15%。异常路径性能损失较大,也就是发生了异常后性能会降低, 比传错误码大很多。关于异常的详细的讨论参考吴老师的文章:[https://zhuanlan.zhihu.com/p/6170882594](https://zhuanlan.zhihu.com/p/617088259) 
总结: 使用异常机制时, 异常的概率最好低于1%, 频繁抛异常效率就太低了。根据我的理解只在某些关键地方使用, 比如使用系统API打开文件, 服务程序的重启机制等。

#### 三、错误码机制
&emsp; 关于错误码, 我深刻体会到它的一个问题就是不同模块之间定义的错误码不同, 而且通常都是 int 整数类型的错误码, 当在一个编译单元内使用了含有不同定义错误码的多个SDK时,就比较方了, 而且通常不可以转换成对应字符串, 日志里面看到易读性就不高。这里可以借助标准库的 std::system_error 来配合使用, 它的主要成员有:
&emsp;  ①枚举 errc, 几乎覆盖 POSIX Error 的错误码
&emsp;  ②error_code 和 error_condition 两个类分别代表错误和通用错误
&emsp;  ③支持特殊错误和一般错误的转换进行比较
所以可以把自定义的错误码转换成 std::system_error 里面的错误码, 用户直接将错误码和 std::system_error 标准库的错误码进行比较, 就避免了出现多套错误码的情形。下面结合直接上代码看看具体咋个用的。
##### 3.1、集成错误码一 标识错误码
&emsp; 假如自定义了一下错误码:
```javascript
enum class DivError
{
    kInvalidParam,
    kDivideZero,
};
```
通过特化向标准库标记其为错误码:
```javascript
template<>
struct std::is_error_code_enum<DivError>: true_type {};
```
##### 3.2、集成错误码二 错误类别和输出
&emsp;作用是将枚举类型通过继承 std::error_category 接口转成对应的字符串。
```javascript
class DivErrorCategory:public std::error_category
{
public:
    const char* name()const noexcept override 
    {
        return "Divide Error";
    }

    std::string message(int code) const override
    {
        switch (static_cast<DivError>(code))
        {
        case DivError::kSuccess:
            return "Success";
        
        case DivError::kInvalidParam:
            return "InvalidParam";
        
        case DivError::kDivideOverFlows:
            return "DivideOverFlow";
        }
         return "Unknown";
    }
};
```
覆写 std::error_category 基类的 const char* name() const noexcept 和 std::string message(int code) const 两个函数即可。

##### 3.3、集成错误码三 错误转换成标准库错误码
&emsp;这里和第二步的流程类似, 转换成对应的 std::error_condition
```javascript
    std::error_condition default_error_condition(int code) const noexcept override
    {
        switch (static_cast<DivError>(code))
        {
        case DivError::kSuccess:
            break;
        
        case DivError::kInvalidParam:
            return std::make_error_condition(std::errc::invalid_argument);

        case DivError::kDivideOverFlows:
            return std::make_error_condition(std::errc::value_too_large);  
        }

        return std::error_condition(code, *this);
    }   
```

##### 3.4、集成错误码四 构造 error_code
&emsp;需要提供一个转换成 std::error_condition 的通用函数:
```javascript
std::error_code make_error_code(DivError e)
{
    static DivErrorCategory diverror_category;
    return {static_cast<int>(e), diverror_category};
}
```
此时就可以把模块内定义的错误码与标准库的错误码关联起来了, 而且还能输出对应字符串。这个机制还可以参考 boost 库内的 boost::outcome。

####四 封装对象与返回值优化
&emsp; 考虑如下代码:
```javascript
struct BigObj
{
    BigObj(){std::cout<<"Big Obj Construct"<<std::endl;}
    ~BigObj(){std::cout<<"Big Obj deConstruct"<<std::endl;}
    BigObj(const BigObj& that){std::cout<<"Big Obj copy"<<std::endl;}
    void Op(){std::cout<<"Big Obj Op"<<std::endl;}
private:
   std::array<int, 1024> m_big_data;
};

BigObj GetBigObj(bool flag)
{
    if(!flag){
        std::runtime_error("BigObj bad flag");
    }
    BigObj obj;
    obj.Op();
    return obj;
}
```
根据第一节的知识, 假如有函数调用 GetBigObj(), 那么此时会有返回值优化, 期间 BigObj 只会构造一次, 但是当返回值类型被 std::optional 封装一次以后就不一定了。
 ```javascript
std::optional<BigObj> GetBigObj(bool flag)
{
    if(!flag){
        return std::nullopt;
    }

    BigObj obj;
    obj.Op();
    return obj;
}
```
此时如果调用 GetBigObj() 返回对象就会发生一次对象拷贝。因为函数内出现了返回两种不同类型的参数, 编译器就无法以此来进行返回值优化。此时就需要进行改进一下, 代码如下所示:
 ```javascript
std::optional<BigObj> GetBigObj(bool flag)
{
    std::optional<BigObj> result;
    if(!flag){
        return result;
    }

    result.emplace();
    result->Op();
    return result;
}
 ```
 此时, 就可以一如既往的拥有返回值优化的高性能了。