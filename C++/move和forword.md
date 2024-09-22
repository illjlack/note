### std::move与std::forward

定义在头文件

```
#include <type_traits> // traits是特性   或者   #include <utility>
```

#### 作用

`std::move`：把左值引用或右值引用都转成右值引用。

```
C++11 引入了 引用折叠 的概念。在模板中，引用类型可能被进一步引用，当多重引用类型结合时，遵循一定规则来简化成最终的引用类型：
T& & => T&
T&& & => T&
T& && => T&
T&& && => T&&

万能引用：
template<class T>
void ff(T&& x){}
T可以被理解为 & 或者 && 引用类型

传参过程中左值被推导为左值引用,右值被推导为右值引用
例：有一个模板 A 和它的特化版本 <A> 以及 <A&>，当传递一个左值时，它会被匹配到 <A&>
```

`std::forward`：完美转发，强转成特例化的引用类型，用于传递万能引用的参数到调用函数的时候，需要原因是：**在 C++ 中，所有有名字的对象，无论它们的类型是左值引用还是右值引用，在它们的作用域内都会被当作左值对待。**

```cpp
//demo
#include <iostream>
#include <utility> 
using namespace std;
void f(int&& x) { cout << "====右值引用====" << endl; }

void f(int& x) { cout << "====左值引用====" << endl; }

template<class T>
void ff(T&& x)
{
    cout << "不转发都被理解为左值";
    f(x);
    cout << "转发";
    f(std::forward<T>(x));
}

int main()
{
    int x;
    int&& xx = static_cast<int&&>(x);

    ff(x);
    ff(xx); // 右值引用被理解为左值

    ff(std::forward<int&&>(x)); // 直接特例化成右值引用的时候和move相同作用
    ff(std::move(x));

    //ff(std::forward<int&>(static_cast<int&&>(x))); // 右值转为左值报错 
}
```



#### 函数原型

```cpp
// 特化后,参数是左值引用调用,因为 引用折叠 原理,会转成特化的类型
template <class _Ty>
_NODISCARD constexpr _Ty&& forward(remove_reference_t<_Ty>& _Arg) noexcept { 
    // forward an lvalue as either an lvalue or an rvalue
    return static_cast<_Ty&&>(_Arg);
}

// 特化后,参数是右值引用调用,因为 引用折叠 原理,会转成特化的类型
template <class _Ty>
_NODISCARD constexpr _Ty&& forward(remove_reference_t<_Ty>&& _Arg) noexcept {
    // forward an rvalue as an rvalue
    static_assert(!is_lvalue_reference_v<_Ty>, "bad forward call"); // 右值引用转为左值引用报错
    return static_cast<_Ty&&>(_Arg);
}

// 万能引用，然后统一强转为右值引用
template <class _Ty>
_NODISCARD constexpr remove_reference_t<_Ty>&& move(_Ty&& _Arg) noexcept { 
    // forward _Arg as movable
    return static_cast<remove_reference_t<_Ty>&&>(_Arg);
}
```

```cpp
_NODISCARD 是 Microsoft Visual C++ (MSVC) 中使用的实现方式，是编译器特定的宏，等价于标准 C++ 中的 [[nodiscard]]
是 C++17 引入的一个属性，用于告诉编译器，如果函数的返回值被忽略，编译器会生成一个警告
```

```cpp
constexpr 是 C++11 引入的一个关键字，用于指示一个变量或函数可以在编译时求值，从而提高程序的性能和优化能力。
```

```cpp
noexcept 是 C++11 引入的关键字，用于指示一个函数不会抛出任何异常。它主要用于优化和保证异常安全性。
使用 noexcept 来确保某些函数（如移动构造函数和移动赋值运算符）不抛出异常，这对于对象的移动操作非常关键，因为异常可能导致资源泄漏。
```

```cpp
/// remove_reference
  template<typename _Tp>
    struct remove_reference
    { typedef _Tp   type; };

  template<typename _Tp>
    struct remove_reference<_Tp&> // 模板特化，会优先匹配，这涉及到了C++模板推导
    { typedef _Tp   type; };

  template<typename _Tp>
    struct remove_reference<_Tp&&> 
    { typedef _Tp   type; };

```

