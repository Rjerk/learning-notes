# Item 1：理解模板类型推导

## 函数模板传参：pointer、reference、forwarding reference 和 pass-by-value

假设有函数模板：

```
template <typename T>
void f(ParamType param);
```

调用：`f(expr);`

在编译器，编译器使用 expr 来推导两种类型：T 和 ParamType。但类型 T 的推导不仅依赖于 expr 的类型，还有 ParamType。

- ParamType 为 pointer 或 reference，但不是 forwarding reference
- ParamType 为 forwarding reference
- ParamType 既非 pointer 也非 reference

### case 1：ParamType 为 pointer 或 reference，但不是 forwarding reference

类型推导工作机制为：
1. 如果 expr 为 reference，忽略引用部分
2. 以 expr 的类型和 ParamType 来确定 T 的类型

如果有函数模板：

```
template <typename T>
void f(T& param); // param is a reference
```

三个变量声明为：

```
int x = 27;
const int cx = x;
const int& rx = x;
```

分别调用：

```
f(x);  // T is int, param's type is int&
f(cx); // T is const int, param's type is const int&
f(rx); // T is const int, param's type is const int&
```

当传递给 T& 参数一个 const 对象时，const 成为 T 的类型推导的一部分。

当传递给 T& 参数一个 reference 时，reference-ness 在 T 的类型推导的过程中被忽略。

在上面的例子中虽然都是 lvalue reference 参数，但对于 rvalue reference 参数类型推导的作用机制也是一样的。

如果我们改变一下模板参数，T& -> const T&：

```
template <typename T>
void f(const T& param);

int x = 27;
const int cx = x;
const int& rx = x;
```

对于 T 的推导会有所不同：

```
f(x);  // T is int, param's type is const int&
f(cx); // T is int, param's type is const int&
f(rx); // T is int, param's type is const int&
```

如果 param 是一个 pointer，推导过程也是一样：

```
template <typename T>
void f(T* param);

int x = 27;
const int* px = &x;
```

```
f(&x); // T is int, param's type is int*
f(px); // T is const int, param's type is const int*
```

### case 2：ParamType 为 forwarding reference

推导机制为：

- 如果 expr 是 lvalue，那么 T 和 ParamType 被推导为 lvalue reference。
  1. 模板类型推导中这是唯一一种情况 T 被推导为 reference。
  2. 即使 ParamType 被声明为 rvalue reference，它的推导类型也为一个 lvalue reference。
- 如果 expr 是 rvalue，使用 case 1 的推导规则。

如果有：

```
template <typename T>
void f(T&& param);

int x = 27;
const int cx = x;
const int& rx = x;
```

对其调用有类型推导：

```
f(x); // x is lvalue, so T is int&, param's type is also int&.
f(cx); // cx is lvalue, so T is const int&, param's type is also const int&.
f(rx); // cx is lvalue, so T is const int&, param's type is also const int&.
f(27); // 27 is rvalue, so T is int, param's type is int&&
```

### case 3：ParamType 既非 pointer 也非 reference

这种情况处理 pass-by-value。

```
template <typename T>
void f(T param);
```

意味着 param 是传入对象的一份拷贝，是一个新的对象，它的类型推导机制为：
1. 如果 expr 的类型是 reference，则忽略 reference 部分。
2. 忽略 expr 的 reference-ness 后，如果 expr 是 const, 忽略它。如果是 volatile，也忽略它。

例如：

```
int x = 27;
const int cx = x;
const int& rx = x;

f(x); // T's and param's type are both int.
f(cx); // T's and param's type are again both int.
f(rx); // T's and param's type are still both int.
```

理解 const 在 pass-by-value 的类型推导过程中被忽略掉很重要。假设 expr 是一个指向 const 对象的 const 指针，并 pass-by-value 时：

```
tempate <typename T>
void f(T param);

const char* const ptr = "fun with pointers";

f(ptr); // param's type is const char*
```

第二个 const 表明 ptr 不能被改变指向一个不同的位置，也不能设为 nullptr。当 ptr 传递给 f 时，ptr 被拷贝，根据类型推导机制，ptr 的常量性被去掉，所以 param 的类型被推导为 const char*。

## 以数组为参数

当一个数组作为 by-value 参数会怎样？

```
template <typename T>
void f(T param);

const char name[] = "J. P. Briggs"; // name's type is const char[13]

f(name);
```

我们知道函数参数中并没有数组这个东西。是的，下面这条语句合法：

```
void myFunc(int param[]);
```

但是数组声明被当做指针声明来对待，意味着 myFunc 等于：

```
void myFunc(int* param); // same function as above
```

所以当调用 `f(name)` 时，T 被推导为 `const char*`:

```
f(name); // name is array, but T deduced as const char*
```

但是，虽然函数不能声明参数为一个数组，但是可以声明参数为指向数组的引用。所以，当修改模板 f 以 by-reference 传参时：

```
template <typename T>
void f(T& param);

const char name[] = "J. P. Briggs"; // name's type is const char[13]

f(name);
```

类型 T 被推导为数组的类型，而且这个类型包含数组的大小，在上面例子中，T 被推导为 `const char[13]`，f 的参数类型则为 `const char(&)[]`。

有趣的是，借助指向数组的引用的声明，我们可以推导出数组元素的数量：

```
// return size of an array as a compile-time constant

template <typename T, std::size_t N>
constexpr std::size_t arraySize(T (&)[N]) noexcept
{
    return N;
}

int keyVals[] = { 1, 3, 5, 7, 9, 11, 13, 15 };

int mappedVals[arraySize(keyVals)]; // array mappedVals havs 7 elements 
```

## 以函数为参数

以函数作为参数时，函数类型能被转化为函数指针，而且对数组的类型推导也同样适用于这个过程：

```
void someFunc(int, double);

template <typename T>
void f1(T param);

template <typename T>
void f2(T& param);

f1(someFunc); // param deduced as ptr-to-func;
              // type is void (*)(int, double)

f2(someFunc); // param deduced as ref-to-func;
              // type is void (&)(int, double)
```