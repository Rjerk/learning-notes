# 核心语言特性

## 尽可能使用 `auto` 关键字

C++11 中，`auto` 能被用于声明局部变量和带有 trailing return type 的函数返回类型，C++14 中，`auto` 能被用于不指定 trailing return type 的函数返回类型和 lambda 表达式的参数声明。

```
// C++11
auto func1(int const i) -> int
{
    return 2 * i;
}

// C++14
auto func2(int const i)
{
    return 2 * i;
}
```

好处：
- 不会让变量处于未初始化的状态。
- 确保总是使用正确类型并且不会出现隐式类型转换。

    ```
    auto v = std::vector<int> {1, 2, 3};
    int size1 = v.size(); // 隐式类型转换，size_t -> int
    auto size2 = v.size(); // size_t
    auto size3 = int{ v.size() }; // error, narrowing conversion.
    ```

- 促进好的面向对象实践，比如说偏好接口而非实现，指定的类型数量越少，代码越泛化并对未来的改变更开放。
- 打字更少并无需关心实际的类型。
- 提供一致的代码风格，类型总出现在右边。

注意：

- `auto` 只是类型的占位符，不用于 `const`/`volatile` 和引用说明符。如果要用，则需自行指定，比如 `auto&`。
- 不可移动的类型不能用 `auto`。

    ```
    auto ai = std::atomic<int>(42); // error.
    ```

- 多字类型不能用 `auto`，比如 `long long`, `long double`, `struct foo`。但在第一个情况里可以用字面量或类型别名来说明。

    ```
    auto l1 = long long {32}; // error.
    auto l2 = llong {42}; // OK.
    auto l3 = 34LL; // OK.
    ```

- 如果你使用 `auto` 说明符，但仍需知道该类型，可以在一些 IDE 里用光标放在变量上进行查看，离开 IDE 则只能自行推导。

## 创建类型别名和别名模板

`typedef` 不能用于创建模板类型别名。

C++11 中，类型别名是另一个已声明类型的名字，别名模板是另一个已声明模板的名字。使用 `using` 来创建这些别名。

为了可读性和一致性：
- 不要把 `typedef` 和 `using` 混用
- 使用 `using` 创建函数指针类型的名字

创建别名模板：

```
template <class T>
class custom_allocator { ... };
 
template <typename T>
using vec_t = std::vector<T, custom_allocator<T>>;

vec_t<int> vi;
vec_t<std::string> vs;
```

注意：
- 别名模板不能为偏特化或全特化
- 当推导模板形参时别名模板别名模板不能被模板实参推导出来
- 当特化一个别名模板时产生的类型不被允许去直接或间接地用它自己的类型

## 理解统一初始化

C++11 中大括号初始化是统一初始数据的方法，因此它也叫统一初始化（uniform initialization）。

大括号初始化形式 `{}` 可以被用于直接初始化和拷贝初始化，我们分别称之为直接列表初始化和拷贝列表初始化。

```
T obj {other}; // direct list initialization.
T obj = {other}; // copy list initialization. 
```

1. C++11 里标准库容器的初始化可使用 `std::initializer_list<T>` 作为构造函数的参数。使用 `std::initializer_list` 初始化的的过程如下：
- 编译器解析初始化列表里的元素类型（所有元素必须是同一类型）
- 编译器为这些元素创建一个数组
- 编译器创建一个 `std::initializer_list<T>` 来包裹先前创建的数组
- `std::initializer_list<T>` 对象作为一个实参被传入构造函数

当大括号初始化被用到时，初始化列表优于其他构造函数被使用： 

```
class foo {
    int a_;
    int b_;
public:
    foo(): a_(0), b_(0) { }
    foo(int a, int b = 0): a_(a), b_(b) { }
    foo(std::initializer_list<int> l) { }
};

foo f{ 1, 2 }; // call constructor with initializer_list<int>.
```

该优先规则适用于任何函数，不仅仅是构造函数。

```
void func(int const a, int const b, int const c)
{
}

void func(std::initializer_list<int> const l)
{
}

func({ 1, 2, 3}); // call second overload.
```

2. 大括号初始化不允许 narrowing conversion。

```
int i { 1.2 }; // error.
double d = 23 / 13;
float f1 { d }; // error.
float f2 { 23/13 }; // OK.
```

必须显示类型转换：

```
int i { static_cast<int>(1.2) };
double d = 23 / 13;
float f1 { static_cast<float>(d) };
```

**注意**：大括号初始化列表不是一个表达式而且没有类型。因此，`decltype` 不能用于大括号初始化列表，而且模板类型推导也不能推导出它的匹配类型。

在 C++11 中对直接列表初始化和拷贝列表初始化的类型推导为 `std::initializer_list<T>`：

```
auto a = {23}; // std::initializer_list<int>
auto b {23}; // std::initializer_list<int>
auto c = {4, 2}; // std::initializer_list<int>
auto d {4, 2}; // std::initializer_list<int>
```

C++17 中改变了列表初始化的规则来区分直接和拷贝初始化：
- 对于拷贝列表初始化 `auto` 推导，如果所有元素类型相同，会推导为 `std::initializer_list<T>`，否则为 ill-formed。
- 对于直接列表初始化的 `auto` 推导，如果列表里有单个元素，会推导为 `T`，如果有多个元素则为 ill-formed.

```
auto a = {23}; // std::initializer_list<int>
auto b {23}; // int
auto c = {4, 2}; // std::initializer_list<int>
auto d {4, 2}; // error, too many.
```

## 理解非静态成员初始化的不同形式

为初始化类的非静态成员应该：
- 如果类的成员带有多个构造函数为之通过提供默认值来使用默认成员初始化 ([3] [4])
- 对于静态和非静态的常量，使用默认成员初始化 （[1] [2]）
- 依赖于构造函数的参数使用初始化列表来初始化成员 （[5] [6]）
- 当其他选项不能用时，在构造函数里使用赋值操作（比如用 `this` 指针初始化数据成员，检查构造函数参数值并抛出异常）

```
struct Control {
    const int DefaultHeigh = 14; // [1]
    const int DefaultWidth = 80; // [2]
    
    TextVAligment valign = TextVAligment::Middle; // [3]
    TextHAligment halign = TextHAligment::Left; // [4]
    
    std::string text;
    
    Control(std::string const & t) : text(t) // [5]
    {}
    
    Control(std::string const & t,
    
    TextVerticalAligment const va, TextHorizontalAligment const ha)
        : text(t), valign(va), halign(ha) // [6]
    {}
};
```

尽可能用构造函数初始化列表，但也有一些无法适用的情况：
- 成员必须被初始化为指向对象自己的引用或指针，使用 `this` 指针在初始化列表里可能引发编译器警告，说它在对象被构造前被使用。
- 有两个成员必须包含对彼此的引用。
- 测试输入参数在初始化一个非静态数据成员的值前用这个参数的值抛出一个异常。

从 C++11 开始，非静态数据成员可以在类中声明时被初始化，称为默认成员初始化。当成员为常量或不依赖于对象被构造的方式时，默认成员初始化就可以被使用。

当数据成员同时用默认成员初始化和构造函数初始化列表初始化时，则取后者构造的结果，默认值被丢弃。

## 控制和查找对象对齐

C++11 提供标准化方法来指定和查找类型对齐的需求。控制对齐对于在不同处理器中提升性能很重要并且能让只在特定对齐方式下才能处理数据的一些指令起作用。

使用 `alignas` 说明符来控制一个类型或对象的对齐，用 `alignof` 操作符来取得一个类型的对齐需求。

```
struct alignas(4) foo {
    char a;
    char b;
};

struct bar {
    alignas(2) char a;
    alignas(8) int  b;
};

alignas(8)   int a;
alignas(256) long b[4];

auto align = alignof(foo);
```

因为处理器访问内存不是一次一个字节，而是以 `2^n` （2, 4, 8...) 来访问。因此需要编译器在内存中对齐数据使得数据可以被处理器轻易地访问到，如果不对齐，编译器访问数据需要做一些额外的工作。

C++ 编译器对齐变量基于他们数据类型的大小：`bool` 和 `char` 为 1 字节，`short` 为 2 字节，`int`、`long` 和 `float` 为 4 字节，`double` 和 `long long` 为 8 字节等。当数据类型为 `struct` 或 `union` 时，对齐必须匹配为最大成员的大小来避免性能问题，比如说：

```
struct foo1 { // size = 1, alignment = 1.
    char a;
};

struct foo2 { // size = 2, alignment = 1.
    char a;
    char b;
};

struct foo3 { // size = 8, alignment = 4.
    char a;
    int  b;
};
```

为了进行对齐，编译器还会假如填充字节。如 foo3 实际上被转换为：

```
struct foo3_ {
    char a; // 1 byte.
    char _pad0[3]; // 3 bytes padding to put b on a 4-byte boundary.
    int  b; // 4 bytes.
};
```

C++11 中，`alignas` 说明符可以被用于表达式，type-id，或 parameter pack 上。它有一些限制：

- 有效对齐为 2^n，其他值都是不合法的。
- 0 对齐总是被忽略。
- 如果在一个声明上的最大 `alignas` 小于不带任何 `alignas` 的 natural 对齐，则程序被视为 ill-formed。

下例中，不带 `alignas` 的 natural 对齐为 1，带 `alignas(4)` 则为 4.

```
struct alignas(4) foo {
    char a;
    char b;
};
```

下例中，对齐需求为 4，小于 strictest 需要的对齐，于是被忽略，编译器会发出警告。

```
struct alignas(4) foo {
    alignas(2) char a;
    alignas(8) int  b;
};
```

上例实际上为：

```
struct foo {
    char a;
    char _pad0[7];
    int b;
    char _pad0[4];
};
```

对于变量：

```
alignas(8)   int a;
alignas(256) long b[4];

printf("%p\n", &a); // eg. 0x7ffe3b7212f8
printf("%p\n", &b); // eg. 0x7ffe3b721300
```

可以看到，`a` 的地址是 8 的倍数，`b` 的地址是 256 的倍数。


不像 `sizeof`，`alignof` 操作符只能用于 type-id，不能用于变量或类数据成员。所应用的类型必须为完整类型，一个数组类型，或一个引用类型。对于数组，返回值为元素类型的对齐；对于引用，返回值为引用类型的对齐。

```
alignof(char); // 1
alignof(int); // 4
alignof(int*); // 4 on 32-bit, 8 on 64-bit.
alignof(int[4]); // 4

struct alignas(4) foo {
    alignas(2) char a;
    alignas(8) int  b;
};
alignof(foo&); // 8
```

## 使用限定作用域的枚举类型

- 使用限定作用域的枚举类型而不是无限定域的枚举类型
- 通过 `enum class` 或 `enum struct` 来使用

无限定作用域的枚举类型带来的问题：
- 它们到处其枚举子到周围作用域中，有两个缺点：在相同作用域的两个枚举类型中有相同名字会导致名称冲突；使用枚举时不可能使用它的完整限制名。

    ```
    enum Status { Unknown, Created, Connected };
    enum Code { OK, Failure, Unknown }; // error.
    auto status = Status::Created; // error.
    ```

- 在 C++11 之前，它们无法指定需要成为整数类型的 underlying 类型，这个类型不能大于 `int`，除非枚举子类型不符合 `signed` 或 `unsigned` 整型。因为直到定义才知道 underlying 类型，这时才知道枚举类型的大小，所以枚举类型也不能前向声明。
- 枚举子的值隐式转换为 `int` 类型。可能会无意混用枚举类型和整型，而编译器却不发出警告。

    ```
    enum Code { OK, Failure };
    void includeOffset(int pixels) { ... }
    includeOffset(Failure);
    ```

限定作用域的枚举类型是强类型的枚举，不同于无限定作用域的枚举类型：
- 不会导出其枚举子到周围作用域。
- 可以指定 underlying 类型。

    ```
    enum class Codes : unsigned int;
    void printCode(Codes const code);
    enum class Codes : unsigned int {
        OK = 0,
        Failure = 1,
        Unknown = 0xFFFF0000U
    };
    ```
- 不会隐式转换为 `int`。

## 虚函数使用 `override` 和 `final` 关键字

为确保基类和派生类中的虚函数的正确声明，也为增加可读性，应这样：
- 当声明派生类中的虚函数应该覆盖基类中的虚函数时，总是使用 `virtual` 关键字，并且
- 在虚函数声明或定义后使用 `override` 的特殊标识。

```
class Base {
    virtual void foo() = 0;
    virtual void bar() {}
    virtual void foobar() = 0;
};

void Base::foobar() {}

class Derived1 : public Base {
    virtual void foo() override = 0;
    virtual void bar() override {}
    virtual void foobar() override {}
};

class Derived2 : public Dervied1 {
    virtual void foo() override {}
};
```

为确保函数不能被进一步覆盖或者类不能再被派生，使用 `final` 特殊标识：

- 在虚函数声明或定义的 declarator 部分之后，阻止在派生类中的覆写：

    ```
    class Derived2 : public Derived1 {
        virtual void foo() final {}
    };
    ```

- 在类声明的类名之后使用，阻止进一步的继承该类：

    ```
    class Derived5 final : public Derived1 {
        virtual void foo() override {}
    };    
    ```

`override` 和 `final` 关键字是特殊标识，使得一个成员函数的声明或定义具有特殊意义。 它们不是保留的关键字并且可以在程序的其他地方作为用户定义的标识符。

## 使用基于范围的 for 循环到迭代器上

```
std::vector<int> getRates()
{
    return std::vector<int> {1, 1, 2, 3, 5, 8, 13};
}

std::multimap<int, bool> getRates2()
{
    return std::multimap<int, bool> {
        { 1, true },
        { 1, true },
        { 2, false },
        { 3, true },
        { 5, true },
        { 8, false },
        { 13, true }
        };
}
```

基于范围的 for 循环可以用于不同的方法：
- 为序列容器提交特定的类型

    ```
    auto rates = getRates();
    for (int rate : rates)
        std::cout << rate << std::endl;
    for (int& rate : rates)
        rate *= 2;
    ```

- 不指定类型，让编译器推导它

    ```
    for (auto& rate : getRates())
        std::cout << rate << std::endl;
    for (auto& rate : rates)
        rate *= 2;
    for (auto const& rate : rates)
        std::cout << rate << std::endl;
    ```

- 在 C++17 中通过使用 structured bindings 和 decomposition declaration：

    ```
    for (auto&& [rate, flag] : getRates2())
        std::cout << rate << std::endl;
    ```

在 C++17 之前，基于范围的 for 循环

```
for (range_declaration : range_expression)
    loop_statement
```


被编译器转换为：

```
{
    auto&& __range = range_expression;
    for (auto __begin = begin_expr, __end = end_expr;
        __begin != __end; ++__begin) {
        range_declaration = *__begin;
        loop_statement
    }
}
```

`begin_expr` 和 `end_expr` 是什么取决于范围的类型：
- 对于 C-like 数组，`__range` 和 `__bound` 是数组中的元素数量。
- 对于带有 `begin()` 和 `end()` 成员的类类型来说是：`__range.begin()` 和 `__range.end()`。
- 对于其他类型来说是 `begin(__range)` 和 `end(__range)`，通过参数名称查找来决定。

在 C++17 中，编译器生成的代码有所不同：

```
{
    auto&& __range = range_expression;
    auto __begin = begin_expr;
    auto __end = end_expr;
    for (; __begin != __end; ++__begin) {
        range_declaration = *__begin;
        loop_statement
    }
}
```

新标准里面已经移除了 begin 表达式和 end 表达式必须是相同类型的限制，end 表达式不必是一个迭代器，但它必须具备和一个迭代器有不等性比较的能力。这样一个好处就是范围可以通过一个 predicate 来限定。

## 让基于范围的 for 循环在定制类型中起作用

需要做到：
- 为该类型创建 mutable 和 constant 迭代器并实现以下操作符：
    - `operator++` 用于递增迭代器
    - `operator*` 用于解引用迭代器并访问迭代器实际指向的元素
    - `operator!=` 用于比较两个迭代器的不等性
- 提供可用的 `begin()` 和 `end()` 函数

假如有自定义的简单数组：

```
template <typename T, size_t const Size>
class dummy_array {
    T data[Size] = {};
public:
    T const & GetAt(size_t const index) const
    {
        if (index < Size) return data[index];
        throw std::out_of_range("index out of range");
    }

    void SetAt(size_t const index, T const & value)
    {
        if (index < Size) data[index] = value;
        else throw std::out_of_range("index out of range");
    }
    
    size_t GetSize() const { return Size; }
};
```

希望可以这样使用：

```
dummy_array<int, 3> arr;
arr.SetAt(0, 1);
arr.SetAt(1, 2);
arr.SetAt(2, 3);

for(auto&& e : arr) {
    std::cout << e << std::endl;
}
```

实现最小的迭代器类:

```
template <typename T, typename C, size_t const Size>
class dummy_array_iterator_type {
public:
    dummy_array_iterator_type(C& collection, size_t const index)
        : index(index), collection(collection)
    { }

    bool operator!= (dummy_array_iterator_type const & other) const
    {
        return index != other.index;
    }

    T const & operator* () const
    {
        return collection.GetAt(index);
    }

    dummy_array_iterator_type const & operator++ ()
    {
        ++index;
        return *this;
    }
private:
    size_t index;
    C& collection;
};
```

mutable 和 constant 迭代器的别名模板：

```
template <typename T, size_t const Size>
using dummy_array_iterator = dummy_array_iterator_type<T, dummy_array<T, Size>, Size>;

template <typename T, size_t const Size>
using dummy_array_const_iterator = dummy_array_iterator_type<T, dummy_array<T, Size> const, Size>;
```

`begin()` 和 `end()` 函数返回对应的 begin 和 end 迭代器：

```
template <typename T, size_t const Size>
inline dummy_array_iterator<T, Size> begin(dummy_array<T, Size>& collection)
{
    return dummy_array_iterator<T, Size>(collection, 0);
}

template <typename T, size_t const Size>
inline dummy_array_iterator<T, Size> end(dummy_array<T, Size>& collection)
{
    return dummy_array_iterator<T, Size>(collection, collection.GetSize());
}

template <typename T, size_t const Size>
inline dummy_array_const_iterator<T, Size> begin(dummy_array<T, Size> const & collection)
{
    return dummy_array_const_iterator<T, Size>(collection, 0);
}

template <typename T, size_t const Size>
inline dummy_array_const_iterator<T, Size> end(dummy_array<T, Size> const & collection)
{
    return dummy_array_const_iterator<T, Size>(collection, collection.GetSize());
}
```

## 使用 explicit 构造函数和 conversion 操作符来避免隐式转换

C++11 之前，但一个参数的构造函数被认为是一个转换构造函数，在 C++11 中，没有 `explicit` 说明符的构造函数被认为是一个转换构造函数。这样一个构造函数定义为从参数类型到类类型的一个隐式类型转换。

```
struct handle_t {
    explicit handle_t(int const h): handle(h) { }
    explicit operator bool() const { return handle != 0; }
private:
    int handle;
};
```

## 使用未命名的名称空间而不是静态全局变量

当需要声明静态全局变量来避免链接时的名称碰撞问题，请使用未命名的名称空间：
- 在源文件中声明未命名的名称空间
- 将全局函数或变量的定义放在未命名的空间里而无需使之为 `static`

```
// file1.cpp
namespace {
    void print(std::string message)
    {
        std::cout << "[file1]" << message << std::endl;
    }
}

void file1_run()
{
    print("run");
}

// file2.cpp
namespace {
    void print(std::string message)
    {
        std::cout << "[file2]" << message << std::endl;
    }
}

void file2_run()
{
    print("run");
}
```

当一个函数在翻译单元里声明时，它就有了个外部链接。这就意味着来自两个文件的两个有相同名称的函数会产生一个链接错误。在 C 中的解决办法是，声明函数或变量为 `static` 并把它的链接从外部改为内部。这样，它的名称就不再被导出到翻译单元，链接问题就可以避免。

在 C++ 中合适的解决办法是使用未命名的名称空间，当如上定义一个名称空间时，编译器会把它转换为如下形式：

```
// file1.cpp
namespace _uniqie_name_ {}
using namespace _unique_name_;
namespace _uniqie_name_ {
    void print(std::string message)
    {
        std::cout << "[file1]" << message << std::endl;
    }
}

void file1_run()
{
    print("run");
}
```

首先，它用一个独一无二的名称声明了一个名称空间，该名称空间是空的，其目的只是建立一个名称空间。然后，一个 `using` 指令把所有 `_unique_name_` 的所有东西带到当前名称空间之中。最后，该名称空间在源代码中，带着编译生成的名称，被定义出来。

通过定义翻译单元局部 `print()` 函数在一个未命名的名称空间中，他们就有了局部可见性，那么它们的外部链接就不会产生一个链接错误，因为它们现在有了一个外部独特名称。

模板参数不能为带有内部链接的名称，因此使用静态变量是不可以的。因此，在匿名空间中的名称带有外部链接可以被用作模板参数中。

```
template <int const& Size>
class test { }

static int Size1 = 10;

namespace {
    int Size2 = 10;
}

test<Size1> t1; // 编译错误
test<Size2> t2; // OK.
```


## 为符号版本化使用内联名称空间

C++11 标准引入一个新类型的名称空间，称为内联名称空间（inline namespace），内联名称空间中的名字可以被外层名称空间直接使用，它通过在 `namespace` 前面添加 `inline` 来使用。这对于库版本化是一个有用的特性。

为了提供一个库的多个版本并且让用户来决定使用什么版本的库，我们可以这样做：
- 在一个名称空间的内部定义该库的内容
- 在它里面的内部内联空间中定义该库的每个版本
- 使用预处理器宏和 `#if` 指令来使用该库的一个版本

内联名称空间具有传递性，也就是说，如果名称空间 A 中如果有一个内联名称空间 B，B 中包含一个内联名称空间 C，那么 C 中的所有成员就会像是 A 和 B 的成员，B 中的成员就像是 A 的成员一样。

假如有这样一个库，它的第一个版本为：

```
namespace modernlib {
    template <typename T>
    int test(T value) { return 1; }
}
```

该库的用户可以通过如下调用返回值 1：

```
auto x = modernlib::test(42);
```

然而，用户可能决定特化该模板函数 `test()`：

```
struct foo { int a; }

namespace modernlib {
    template <>
    int test(foo value) { return value.a; }
}

auto y = modernlib::test(foo { 42 });
```

这种情况下，`y` 的值为 1 而不是 42，因为用户定义的特化版本得到调用。

如果库的开发者突然决定创建该库的第二个版本，提供的 `test()` 函数调用的返回值为 2，而不是 1。为了同时第一个第一个和第二个版本实现，
它们被放入一个嵌套的名称空间中 `version_1` 和 `version_2`，并且使用预处理器宏命令来可选地编译库。

```
namespace modernlib {

    namespace version_1 {
        template <T>
        int test(T value) { return 1; }
    }

    #ifndef LIB_VERSION_2
    using namespace version_1;
    #endif

    namespace version_2 {
        template <T>
        int test(T value) { return 2; }
    }

    #ifdef LIB_VERSION_2
    using namespace version_2;
    #endif

}
```

这样，用户的代码就不能使用了，因为 `test` 函数现在在一个嵌套的名称空间里，而且对于 `foo` 的特化在 `modernlib` 名称空间里，所以客户的代码就必须改变：

```
#define LIB_VERSION_2

#include "modernlib.h"

struct foo { int a; }

namespace modernlib {
    namespace version2 {
        template <>
        int test(foo value) { return value.a; }
    }
}
```

这就是一个问题，因为该库泄露了实现细节，而且客户为了模板特化需要注意这些。

如果使用内联名称空间，将该库定义如下：

```
namespace modernlib {
#ifndef LIB_VERSION_2
    inline namespace version_1 {
        template <typename T>
        int test(T value) { return 1; }
    }
#endif

#ifdef LIB_VERSION_2
    inline namespace version2 {
        template <typename T>
        int test(T value) { return 2; }
    }
#endif
}

#define LIB_VERSION_2

struct foo { int a; }

namespace modernlib {
    template <>
    int test(foo value) { return value.a; }
}

auto y = modernlib::test(foo { 42 });
```

客户的代码就不会失效，而且实现细节也被隐藏起来。

然而要记住：
- 名称空间 `std` 为标准保留而且不应该被内联
- 如果名称空间在第一次没有被定义为内联，再次进入时就不应该被定义为内联


## 使用结构化绑定来处理多个返回值

为从函数中返回多个返回值，可以通过引用一个函数的参数，定义一个结构体来包含多个返回值，或者返回一个 `std::pair` 或 `std::tuple`。前两个使用命名变量可以清楚地指定返回值的含义，但是缺点是不得不显示地定义它们。`std::pair` 有成员 `first` 和 `second`，`std::tuple` 有只能通过函数调用来取得的未命名成员，但是可以使用 `std::tie()` 被拷贝到命名变量指针。然而这些办法都不是理想的。

C++17 扩展了 `std::tie()` 的语义用法到第一级的核心语法特性中，可以取出一个 tuple 里面的值到命名变量之中。这个特性叫做结构化绑定（structured bindlings）。

为了从函数中返回多个值：
- 使用一个 `std::tuple` 作为返回类型：

    ```
    std::tuple<int, std::string, double> find()
    {
        return std::make_tuple(1, "marius", 123.4);
    }
    ```

- 使用结构化绑定来解压 tuple 里的值到命名变量之中：

    ```
    auto [id, name, score] = find(); 
    ```

- 在一个 `if` 或 `switch` 内部，使用分解声明（decomposition declaration）来绑定返回值到变量中：

    ```
    if (auto [id, name, score] = find(); score > 1000) {
        std::cout << name << std::endl;
    }
    ```

结构化绑定是一种类似 `std::tie()` 语言特性，除了我们无需为每个需要使用 `std::tie()`的被解压值定义命名变量。利用结构化绑定，我们使用 `auto` 在一个单独的定义里定义所有命名变量，这样编译器可以为每个变量推断出正确的类型。

```
std::map<int, std::string> m;

{
    auto[it, inserted] = m.insert({1, "one"});
    std::cout << "inserted = " << inserted << std::endl
              << "value = " << it->second << std::endl;    
}
{
    auto[it, inserted] = m.insert({1, "two"});
    std::cout << "inserted = " << inserted << std::endl
              << "value = " << it->second << std::endl;
}
```


C++17 中，可以声明变量在 `if(init; condition)` 和 `switch(init; condition)` 中，可以结合结构化绑定来简化代码。

```
if (auto [it, inserted] = m.insert({1, "two"}); inserted) {
    std::cout << it->second << std::endl;
}
```