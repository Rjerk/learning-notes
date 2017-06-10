# Item 24：区分 forwarding references 和 rvalue references

## T&& 与 forwarding reference

声明一个类型 T 的 rvalue reference，可以这样写 `T&&`。所以在代码里看到 `T&&` 就说明那是一个 rvalue reference 吗？ 事情并非这么简单：

```
void f(Widget&& param); // rvalue reference

Widget&& var1 = Widget(); // rvalue reference

auto&& var2 = var1; // not rvalue reference

template <typename T>
void f(std::vector<T>&& param); // rvalue reference

template <typename T>
void f(T&& param); // not rvalue reference
```

`T&&` 有两个不同的意思。其一为 rvalue reference，它只绑定 rvalue，其存在的主要目的是标识对象可能被移动而来 (identify objects that may be moved from)。

另一个意思是 `T&&` 既不是 rvalue reference 也不是 lvalue reference，它在代码中看起来像 rvalue reference，但是它表现的却像个 lvalue reference。这种双重特性使它们即可绑定于 rvalue, 也可以绑定在 lvalue 上。而且，它们也可以绑定在 const 或 non-const 对象，volatile 或 non-volatile 对象，甚至同时是 const 和 volatile 对象。它们几乎可以绑定在任何东西上。我们称之为 forwarding reference。

## forwarding reference 的判断

forwarding reference 出现在两个地方，一般是在模板参数之中：

```
template <typename T>
void f(T&& param); // param is a forwarding reference.
```

或是在 auto 声明中：

```
auto&& var2 = var1; // var2 is a forwarding reference.
```

在这两个地方它们的共同之处就是类型推导的存在。

如果一个 reference 要成为 forwarding reference，要保证两点：
1. 类型推导发生了
2. 声明的形式必须是 `Type&&`

在下面例子中，T&& 没有类型推导，就说明是 rvalue reference，而非 forwarding reference：

```
void f(Widget&& param); // no type deduction; param is a rvalue reference.

Widget&& var1 = Widget(); // no type deduction; var1 is a rvalue reference.
```

因为 forwarding reference 是引用，所以它们必须被初始化。根据初始的结果，它可以表现为 lvalue reference 或 rvalue reference：
- 如果 initializer 为 rvalue，forwarding reference 就是一个 rvalue reference
- 如果 initializer 为 lvalue，forwarding reference 就是一个 lvalue reference

```
template <typename T>
void f(T&& param);

Widget w;
f(w); // lvalue passed to f; param's type is Widget&.

f(std::move(w)); // rvalue passed to f; param's type is Widget&&.
``` 

### 函数模板参数

如果有：

```
template <typename T>
void f(std::vector<T>&& param); // param is a rvalue reference.
```

param 的类型声明形式为 `std::vector<T>&&` 而不是 `T&&`，所以它是一个 rvalue reference 而非 forwarding reference。

甚至加上 const 也不行：

```
template <typename T>
void f(const T&& param); // param is a rvalue reference.
``` 

如果在一个模板里有个函数参数的类型为 `T&&`，所以它是 forwaring reference 吗？不，在模板里并不意味类型推导会存在：

```
template <class T, class Allocator = allocator<T>>
class vector {
public:
    void push_back(T&& x);
    ...
};
```

`void push_back(T&& x);` 中参数类型为 `T&&`，但是没有 vector 的实例它就不存在，实例的类型决定了 push_back 的声明：

```
std::vector<Widget> v;
```

这使得 std::vector 模板被实例化为：

```
class vector<Widget, allocator<Widget>> {
public:
    void push_back(Widget&& x); // rvalue reference
};
```

可见 push_back 中并未产生类型推导。

来个有类型推导的。在 std::vector 中的成员函数 emplace_back：

```
template <class T, class Allocator = allocator<T>>
class vector {
public:
    template <class... Args>
    void emplace_back(Args&&... args);
};
```

类型参数包 Args 独立于 vector 的类型参数 T，每次 emplace_back 被调用时 Args 就会产生类型推导。 

### auto&&

forwarding reference 在 auto 中的使用场合不像函数模板参数那么普遍，其 auto 用法在 C++11 中偶尔出现，在 C++14 中却出现得更多，因为 C++14 lambda 表达式可能声明 `auto&&` 参数：

```
auto timeFuncInvocation = 
    [](auto&& func, auto&&... params)
    {
        start timer;
        // invoke func on params
        std::forward<decltype(func)>(func) (
            std::forward<decltype(params)>(params)...
        );
        stop timer and record elapsed time;
    }
```

func 是一个可以绑定在任何可调用对象上的 forwarding reference，params 是 0 个或多个 forwarding reference，可以绑定在任意类型任意数量的对象上。因此 timeFuncInvocation 可以方便地为一切函数的执行计时了。