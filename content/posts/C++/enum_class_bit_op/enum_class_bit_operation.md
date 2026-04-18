
+++
title = "C++枚举类型的 BitMask"
date = 2026-04-18
draft = false
categories = ["C++"]
tags = ["enum class"]
+++

&emsp; 枚举类型是 C/C++ 语言中的非常重要的一部分, 经常用于在软件中用来标识各种各样不同的状态。 C/C++98 语言中的枚举类型是个未知类型,  只要求该类型足够大来容下所有的枚举项目,  它没有作用域, 会造成全局命名空间的污染, 在被当做参数传递时会隐士的转换成 int. 
&emsp; C++11后, 引入了强枚举类型解决了上述问题, 可是某些场合下当需要使用   **BitMask** 掩码时就不方便了,  考虑如下代码:
```javascript 
enum class ColorScoped :uint8_t
{
    RED     = 1<<0, 
    BLUE    = 1<<1, 
    GREEN   = 1<<2, 
};

int main() {

 uint8_t mask = ColorScoped::RED | ColorScoped::BLUE;
 return 0;
}
```
编译直接报错
```
error: invalid operands to binary expression ('ColorScoped' and 'ColorScoped')      
```
那么该如何解决这个问题呢?

#### 一 、 使用类型转换
&emsp; 那就是直接使用 static_cast 类型转换。

```javascript 
uint8_t mask = static_cast<uint8_t>(ColorScoped::RED) | static_cast<uint8_t>(ColorScoped::BLUE);
```
这样掩码操作的问题就得到了解决, 但是这种方式有个问题, 在每个需要使用掩码的地方都需要进行一次类型类转换, 代码看起来就有点冗余,  重复代码太多了。所以更好的办法是重载运算符, 如下所示：

```javascript 
inline constexpr std::underlying_type_t<ColorScoped> operator | (ColorScoped lhs,  ColorScoped rhs)
{
    return static_cast<std::underlying_type_t<ColorScoped>>(lhs) | static_cast<std::underlying_type_t<ColorScoped>>(rhs);
}
```
于是就可以直接使用谓运算符了。
```javascript 
uint8_t mask = ColorScoped::RED | ColorScoped::BLUE;
```
#### 二 、使用宏结合类型强转

&emsp; 上述使用类型转换结合运算符重载的方式基本上就是我们所需要的。 如果我们希望代码更加灵活, 为这所有需要进行掩码操作的强枚举类型提供一组通用的操作符。 这个时候宏是一个合理的选择.
```javascript 
#define DEFINE_EnumClass_CLASS_BITWISE_OPERATORS(EnumClass)                   \
    inline constexpr EnumClass operator|(EnumClass lhs,  EnumClass rhs) {           \
        return static_cast<EnumClass>(                                   \
            static_cast<std::underlying_type_t<EnumClass>>(lhs) |        \
            static_cast<std::underlying_type_t<EnumClass>>(rhs));        \
    }                                                               \
    inline constexpr EnumClass operator&(EnumClass lhs,  EnumClass rhs) {           \
        return static_cast<EnumClass>(                                   \
            static_cast<std::underlying_type_t<EnumClass>>(lhs) &        \
            static_cast<std::underlying_type_t<EnumClass>>(rhs));        \
    }                                                               \
    inline constexpr EnumClass operator^(EnumClass lhs,  EnumClass rhs) {           \
        return static_cast<EnumClass>(                                   \
            static_cast<std::underlying_type_t<EnumClass>>(lhs) ^        \
            static_cast<std::underlying_type_t<EnumClass>>(rhs));        \
    }                                                               \
    inline constexpr EnumClass operator~(EnumClass E) {                       \
        return static_cast<EnumClass>(                                   \
            ~static_cast<std::underlying_type_t<EnumClass>>(E));         \
    }                                                               \
    inline EnumClass& operator|=(EnumClass& lhs,  EnumClass rhs) {                  \
        return lhs = static_cast<EnumClass>(                             \
                   static_cast<std::underlying_type_t<EnumClass>>(lhs) | \
                   static_cast<std::underlying_type_t<EnumClass>>(lhs)); \
    }                                                               \
    inline EnumClass& operator&=(EnumClass& lhs,  EnumClass rhs) {                  \
        return lhs = static_cast<EnumClass>(                             \
                   static_cast<std::underlying_type_t<EnumClass>>(lhs) & \
                   static_cast<std::underlying_type_t<EnumClass>>(lhs)); \
    }                                                               \
    inline EnumClass& operator^=(EnumClass& lhs,  EnumClass rhs) {                  \
        return lhs = static_cast<EnumClass>(                             \
                   static_cast<std::underlying_type_t<EnumClass>>(lhs) ^ \
                   static_cast<std::underlying_type_t<EnumClass>>(lhs)); \
    }

enum class ColorScoped :uint8_t
{
    None = 0, 
    RED = 1<<0, 
    BLUE = 1<<1, 
    GREEN = 1<<2, 
};
DEFINE_EnumClass_CLASS_BITWISE_OPERATORS(ColorScoped)

```
这样使用起来就很方便了,  当需要维操作需要使用掩码时, 直接使用**DEFINE_EnumClass_CLASS_BITWISE_OPERATORS** 宏进行声明即可。

#### 三 、使用模板特化结合类型强转
&emsp; 现代C++说尽量不使用宏,  使用模板来替代, 所有有没有更好的方案呢？ 答案是使用模板偏特化。
首先声明一个通用模板标识强枚举类型不支持掩码位操作, 然后使用 std::enable_if_t 进行模板参数约束。
```javascript
template <typename T>
struct EnableBitmaskOperators {
    static constexpr bool enable_bitmask {false};
};

template <typename T>
typename std::enable_if_t<EnableBitmaskOperators<T>::enable,  T> operator|(
    T lhs,  T rhs) {
    return static_cast<T>(static_cast<std::underlying_type_t<T>>(lhs) |
                          static_cast<std::underlying_type_t<T>>(rhs));
}

template <typename T>
typename std::enable_if_t<EnableBitmaskOperators<T>::enable,  T> operator &(
    T lhs,  T rhs) {
    return static_cast<T>(static_cast<std::underlying_type_t<T>>(lhs) &
                          static_cast<std::underlying_type_t<T>>(rhs));
}
...
```

当枚举类型需要进行位运算时, 直接使用枚举类型将上述模板特特化一个版本出来:
```javascript
enum class ColorScoped :uint8_t
{
    None    = 0, 
    RED     = 1<<0, 
    BLUE    = 1<<1, 
    GREEN   = 1<<2, 
    BROWN   = 5, 
};

template <>
struct EnableBitmaskOperators<ColorScoped> {
    static constexpr bool enable_bitmask {true};
};

int main() {

 //uint8_t mask = static_cast<uint8_t>(ColorScoped::RED) | static_cast<uint8_t>(ColorScoped::BLUE);
  ColorScoped mask = ColorScoped::RED | ColorScoped::GREEN;
  std::cout << (int)mask <<std::endl;

  return 0;
}
//输出：5
```
#### 总结
&emsp;目前综合来看应该使用第三种方案, 即模板偏特化结合类型转换的方案。