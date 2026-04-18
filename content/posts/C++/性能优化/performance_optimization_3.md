
+++
title = "C++性能优化总结三"
date = 2026-04-18
draft = false
categories = ["C++"]
tags = ["性能优化"]
+++
&emsp;书接上回, 这篇文章继续来谈谈C++ 并发编程性能优化相关的内容。

参考文章: [hhttps://boolan.com/](https://boolan.com/)
先形象的说明一下并发与并行:
**并发**：类似与足球场踢足球, 大家为了抢一个球(数据)可能会发生碰撞, 足球在某一时刻只能在一个球员手里, 所以为了避免恶意抢球就需要裁判(数据同步), 最后球在谁手里由上帝(CPU)决定。
**并行**：类似于刘翔跨栏时的场景, 大运动员之间各自拼命跑, 互相之间不共享东西(数据)。

#### 一、C++标准库线程间的通信
&emsp; 这里首先纠正自己之前存在的一个知识点误区, 就是关于 std::condition_variable_any 的。先看如下代码:
```javascript 
void WorkFunc(int& result, std::condition_variable_any& cv, std::mutex& cv_mut, bool& ready_flag)
{
    result = 100;
    {
        std::unique_lock w_lock(cv_mut);
        ready_flag = true;
    }
    
    cv.notify_one();
    return;
}

int main()
{
    std::condition_variable_any cv;
    std::mutex cv_mut;
    bool ready_flag{false};
    int result{0};

    std::thread workth(std::ref(result), std::ref(cv), std::ref(cv_mut), std::ref(ready_flag));
    std::unique_lock r_lock(cv_mut);
    cv.wait(r_lock, [&ready_flag](){return ready_flag;});
    std::cout<<"result is:"<< result <<std::endl;
    
    if (workth.joinable()){
        workth.join();
    }
    
    return 0;
}       
```
&emsp;在早期我一直认为线程函数 WorkFunc() 内将  std::condition_variable_any cv 条件变量需要的 ready_flag 标志置为 true 是不需要加锁的, 当时以为条件变量的 wait() 函数会先循环查询 ready_flag 这个标志位为真后再去判断是否有信号量的通知, 所以就不用加锁, 看来还是太年轻了, 理解不到位。
&emsp;而实际情况是 wait() 函数先判断一次标志位, 然后就直接调用实际的 wait()。在复杂时序的情况下可能会出现在检测完 ready_flag 标志为 false 后, 线程函数先执行完了, 主线程再去调用实际的 wait(), 这个时候因为错过了通知就死锁了。 标志位的作用: ①避免错过 notify_one()通知; ②避免假醒。
&emsp;还有就是标准库线程类的构造函数都是值传参, 不会因线程函数的形参都是引用就推导出引用类型, 所有这个需要加上 std::ref

#### 二、内存屏障、获得与释放语义
&emsp;为了保证多线程情况下对数据计算的正确性, 一方面当然是使用 std::mutex 来保证数据的同步, 此外, 某些数据较为简洁的应用场景则可以使用标准库提供的原子类型, 比如通知线程退出的标志位。 这里记录一下关于原子类型标准库提供的相关内存模型:

```javascript 
//保证当前语句之后的所有读写操作不乱序到当前语句之前
std::memory_order_acquire;

//保证当前语句之前的所有读写操作不乱序到当前语句之后
std::memory_order_release;

//松散类型, 只保证当前语句对数据操作的原子性, 没有任何内存屏障
std::memory_order_relaxed;

//同时拥有 std::memory_order_acquire 和 std::memory_order_release 的特点
std::memory_order_acq_rel;

//同时拥有 std::memory_order_acquire 和 std::memory_order_release 的特点外, 同时还拥有全局的内存顺序, 保证所有线程的执行顺序一致
std::memory_order_seq_cst   
```
关于内存序的典型应用场景就是单例模式里面创建实例时的双重检查锁定, 原始代码如下:
```javascript 
//线程非安全版本
Singleton* Singleton::getInstance() {
    if (m_instance == nullptr) {
        m_instance = new Singleton();
    }
    return m_instance;
}

//线程安全版本, 但锁的代价过高
//当对象被创建后, 其他所有线程其实就不需要再等待持有锁了
Singleton* Singleton::getInstance() {
    Lock lock;
    if (m_instance == nullptr) {
        m_instance = new Singleton();
    }
    return m_instance;
}


//双检查锁, 但由于内存读写reorder不安全
Singleton* Singleton::getInstance() {
    
    if(m_instance==nullptr){
        Lock lock;
        if (m_instance == nullptr) {
            m_instance = new Singleton();
        }
    }
    return m_instance;
}
```
双检查实现里面, 写代码时期望的执行顺序是先申请一块内存, 到后调用 Singleton() 构造函数, 最后将内存地址赋值给 m_instance。但是编译器出于优化的目的, 实际的顺序可能是 先申请一块内存, 然后将内存地址赋值给 m_instance, 最后再构造。 此时其他线程进函数后发现 m_instance 不为空, 然后直接返回, 此时这个实例是没有初始化的, 就可能会出现问题。所以安全的实现如下:
```javascript 
std::atomic<Singleton*> Singleton::m_instance;
std::mutex Singleton::m_mutex;

Singleton* Singleton::getInstance() {
    Singleton* tmp = m_instance.load(std::memory_order_relaxed);

    //获取内存fence
    std::atomic_thread_fence(std::memory_order_acquire);
    
    if (tmp == nullptr) {
        std::lock_guard<std::mutex> lock(m_mutex);
        tmp = m_instance.load(std::memory_order_relaxed);
        if (tmp == nullptr) {
            tmp = new Singleton;

            //释放内存fence
            std::atomic_thread_fence(std::memory_order_release);

            m_instance.store(tmp, std::memory_order_relaxed);
        }
    }
    return tmp;
}


//或者不用 fence 直接用获得释放语义实现
Singleton* Singleton::getInstance() {

    Singleton* tmp = m_instance.load(std::memory_order_acquire);
    if (tmp == nullptr) {
        std::lock_guard<std::mutex> lock(m_mutex);
        tmp = m_instance.load(std::memory_order_relaxed);
        if (tmp == nullptr) {
            tmp = new Singleton;
            m_instance.store(tmp, std::memory_order_release);
        }
    }
    return tmp;
}

```

这样就可以保证申请内存, 构造再赋值的执行顺序。

#### 三、多线程优化总结
&emsp;首先需要知道的是, 多线程加锁和数据竞争是性能杀手。有以下几点需要注意:
**①能用 std::atomic 原子类型就不要使用 std::mutex;**
**②如果多线程读比写多很多时, 优先考虑使用读写锁 std::shared_mutex, 其他情况还是使用 std::mutex;**
**③考虑使用 thread_local 变量, 这个相当于不需要加锁的全局变量, 当线程第一次访问的时候对象才会被创建, 线程退出时对象就会被销毁;**

**④能用标准库里面的高级接口就不要自己写, 比如 std::future, std::async等;**
