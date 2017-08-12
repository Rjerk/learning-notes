# 鲁棒和性能

## 用异常处理错误

使用下面的方法处理异常：
- by value 捕获异常

    ```
    void throwing_fun()
    {
        throw std::system_error(
            std::make_error_cde(std::errc::timed_out));
    }
    ```

- by reference 捕获异常，或者大多数情况下，by constant reference

    ```
    try {
        throwing_func();
    } catch (std::exception const& e) {
        std::cout << e.what() << std::endl;
    } 
    ```

- 从类层次体系中捕获多个异常时，从最远派生类到基类的次序捕获

    ```
    auto exprint = [](std::exception const& e)
    {
        std::cout << e.what() << std::endl;
    }
    
    try {
        throwing_func();
    } catch (std::system_error const& e) {
        exprint(e);
    } catch (std::runtime_error const& e) {
        exprint(e);
    } catch (std::exception const& e) {
        exprint(e);
    }
    ```

- 用 `catch(...)` 捕获所有异常，无论类型如何

    ```
    try {
        throwing_func();
    } catch (std::exception const& e) {
        std::cout << e.what() << std::endl;
    } catch (...) {
        std::cout << "unknown exception" << std::endl;
    }
    ```

- 使用 `throw;` 重新抛出异常。在为多个异常创建一个异常处理函数时可以用到。当你想隐藏一个异常的起始位置时抛出异常对象：

    ```
    void handle_exception()
    {
        try {
            throw(); // throw current exception.
        } catch (const std::logic_error& e) {
            //
        } catch (const std::runtime_error& e) {
            //
        } catch (const std::exception& e) {
            //
        }
    }
    
    try {
        throwing_func();
    } catch (...) {
        handle_exception();
    }
    ```

异常在构造函数和析构函数中的处理：
- 用异常来指出出现在构造函数中的异常。
- 不要让异常逃离析构函数。

虽然可以抛出任何类型的异常，但是：
- 最好抛出标准库异常或从 `std::exception` 或其他标准库异常中派生出来的异常。
- 避免抛出内置类型的异常，比如整型。
- 当使用一个提供了自己的异常体系的库或框架，最好从这个体系中抛出异常或者你从它派生出来的异常，至少这部分的代码要和这个体系紧密关联以保持代码一致。 

C++ 标准库定义了异常的分类，出于以下目的：
- `std::logic_error` 表示程序中的逻辑错误，比如无效参数，索引越界等。它有多个派生类，比如 `std::invalid_argument`、`std::out_of_range` 和 `std::length_error`。
- `std::runtime_error` 表示超出程序作用域内或因各种因素不可预测的错误。有多个派生类，比如 `std::overflow_error`、`std::underflow_error` 或 `std::system_error`。
- 带有 `bad_` 的异常，比如 `std::bad_alloc`、`std::bad_cast` 和 `std::bad_function_call`，代表程序的各种错误，比如内存分配失败，不能动态类型转换，函数调用失败等。

所有这些异常来自 `std::exception`，它有个 non-throwing 的虚函数 `what()` ，该函数返回一个指向表示错误描述的字符数组的指针。定制异常时，可以派生它的分类异常，或直接派生自 `std::exception`，要注意：
- 派生自 `std::exception` 时要覆写虚函数 `what()` 提供错误描述

    ```
    class simple_error : public std::exception {
    public:
        virtual const char* what() const noexcept override
        {
            return "simple exception";
        }
    };
    ```

- 派生自 `std::logic_error` 或 `std::runtime_error` 等，你只需提供一个不依赖运行时数据的静态描述，然后传递给基类的构造函数

    ```
    class another_logic_error : public std::logic_error {
    public:
        another_logic_error():
            std::logic_error("simple logic exception") { }
    };
    ```

- 如果依赖于运行时数据，提供一个带有参数的构造函数并用它们来构建该描述信息。可以将它传递给该描述给基类构造函数，或者从覆盖的 `what()` 中返回

    ```
    class advanced_error : public std::runtime_error {
        int error_code;
        std::string make_message(int const e)
        {
            std::stringstream ss;
            ss << "error with code " << e;
            return ss.str();
        }
    public:
        advanced_error(int const s)
            : std::runtime_error(make_message(e).c_str()), error_code(s)
        {
        }
        int error() const noexcept
        {
            return error_code;
        } 
    };
    ```

## 对于不抛出异常的函数使用 `noexcept`

使用下面方法来指定或查询异常规格：
- 在函数声明中使用 `nothrow` 来表明该函数是不抛出任何异常的

    ```
    void func_no_throw() noexcept
    {
    }
    ``` 

- 在函数声明里使用 `nothrow(expr)`，比如在模板元编程里，表示基于一个计算出 `bool` 的条件，该函数可能或可能不抛出异常：

    ```
    template <typename T>
    T generic_func_1()
    noexcept(std::is_nothrow_constructible<T>::value)
    {
        return T;
    }
    ```

- 使用 `noexcept` 操作符，在编译时期检查一个表达式是否被声明为不抛出任何异常

    ```
    template <typename T>
    T generic_func_2() noexcept(noexcept(T{}))
    {
        return T{};
    }
    
    template <typename F, typename A>
    auto func(F&& f, A&& arg) noexcept
    {
        static_assert(!noexcept(f(arg)), "F is throwing!");
        return f(arg);
    }
    
    std::cout << noexcept(func_no_throw) << std::endl;
    ```

在 C++17 中，异常规格是函数类型的一部分，但不是函数签名的一部分，它可能作为任意函数装饰器的一部分出现，所以两个函数签名不会只因异常规格不同而有区别。在 C++17 之前，异常规格不是函数类型的一部分并且只能作为 lambda 装饰器或 top-level 函数装饰器的一部分出现，它们甚至不能出现在 `typedef` 或类型别名声明里。

注意：
- 如果没有异常规格出现，那么函数可能抛出异常。
- `noexcept(false)` 等于没有异常规格。
- `noexcept(true)` 和 `noexcept` 表明一个函数不抛出任何异常。
- `throw()` 等于 `noexcept(true)` 但已经被弃用。

一个指向不抛出任何异常的函数的指针可以被隐式地转换为指向可能抛出异常的函数的指针，但反过来不行。

如果一个虚函数有 no-throwing 异常规格，表明它所有覆写函数的所有声明一定保留这个异常规格，除非覆写函数被声明为删除的。

在编译时期，可以用 `operator noexcept` 来检查一个函数是否被声明为 no-throwing。

    ```
    int double_it(int const i) noexcept
    {
        return i + i;
    }
    int half_it(int const i)
    {
        throw std::runtime_error("not implemented!");
    }
    struct foo
    {
        foo() {}
    };
    std::cout << std::boolalpha
        << noexcept(func_no_throw()) << std::endl // true
        << noexcept(generic_func_1<int>()) << std::endl // true
        << noexcept(generic_func_1<std::string>()) << std::endl // true
        << noexcept(generic_func_2<int>()) << std::endl // true
        << noexcept(generic_func_2<std::string>()) << std::endl // true
        << noexcept(generic_func_2<foo>()) << std::endl // false
        << noexcept(double_it(42)) << std::endl // true
        << noexcept(half_it(42)) << std::endl // false
        << noexcept(func(double_it, 42)) << std::endl // true
        << noexcept(func(half_it, 42)) << std::endl; // true
    ```

一个带有 `noexcept` 的函数如果因为异常而退出则会导致程序的不正常终止，因此使用 `noexcept` 要小心。它的存在有利代码优化，保留强异常保证时帮助提升性能。

许多标准库容器中提供了带有强异常保证的一些操作，比如 `vector` 的 `push_back()` 方法。该方法可以替代拷贝构造函数或拷贝赋值运算符，使用移动构造函数或移动赋值运算符进行优化。然而，为了保留它的强异常保证，只有当移动构造函数和移动赋值运算符不抛出异常时才可以完成。如果抛出异常，拷贝构造函数和拷贝赋值运算符必须被使用。`std::move_if_noexcept()` 就起这种作用。

为了异常规格要遵循以下规则：
- 如果函数可能抛出异常，不要指定任何异常说明符
- 只有函数保证不会抛出异常时才用 `noexcept` 标记
- 只有基于某条件可能抛出异常的函数才用 `noexcept(expression)` 标记
- 除非提供了真实直接的好处，否则不要用 `noexcept` 或 `noexcept(expression)` 标记一个函数

## 保证程序的常量正确性

`T*` 可以被隐式地转换为 `T const *`。然而，`T**` 不能被隐式地转换为 `T const **`，因为这样可能导致常量对象被一个指向非常量对象的指针所修改。

```
int const c = 42;
int* x;
int const** p = &x; // this is an actual error.
*p = &c;
*x = 0; // this modifies c.
```

## 创建编译器常量表达式

常量表达式不仅可以是字面量（比如一个数字或字符串），还可以是函数指向的结果。如果函数的所有输入值都在编译时期知晓，那么编译器就可以执行函数并在在编译期获得结果。`constexpr` 可以做到这一点。

使用：
- 定义可以在编译期被执行的非成员函数

    ```
    constexpr unsigned int factorial(unsigned int const n)
    {
        return n > 1 ? n * factorial(n-1) : 1;
    }
    ```

- 定义可以在编译期执行初始化 `constexpr` 对象的构造函数，而且成员函数也在编译期被调用

    ```
    class point3d {
        double const x_;
        double const y_;
        double const z_;
    public:
        constexpr point3d(double const x = 0,
                          double const y = 0,
                          double const z = 0)
            : x_(x), y_(y), z_(z) { }
    
        constexpr double get_x() const { return x_; }
        constexpr double get_y() const { return x_; }
        constexpr double get_y() const { return z_; }
    };
    ```

- 定义变量使它们的值在编译期被计算

    ```
    constexpr unsigned int size = factorial(6);
    char buffer[size] {0};
    constexpr point3d p {0, 1, 2};
    constexpr auto x = p.get_x();
    ```

以 `constexpr` 声明一个函数不意味着它总被在编译期执行，它只是让可在编译期被计算的表达式中的函数被使用成为可能。仅当函数的所有输入值可以在编译期被计算时函数在编译期执行才会发生。函数可也能在运行时期被调用，例如：

```
constexpr unsigned int size = factorial(6); // compile time evaluation.

int n;
std::cin >> n;
auto result = factorial(n); // runtime evaluation.
```

`constexpr` 的使用有些限制：
- 对于变量，使用仅当：
    - 它的类型是字面量类型
    - 声明时初始化
    - 用来初始化它的表达式是一个常量表达式
- 对于函数，使用仅当：
    - 非 virtual
    - 返回类型和参数类型都是字面量类型
    - 对于函数的调用，至少有一组参数能产生一个常量表达式
    - 函数体必须是删除的或默认的；不应包含 `asm` 声明，`goto` 语句，标签，`try...catch` 块和未初始化或非字面量类型的变量，以及带有静态或线程存储时期的变量。
    - 如果函数是默认的拷贝或移动赋值操作符，那么不应该包含任何 mutable variant 成员。
- 对于构造函数，使用仅当：
    - 所有参数是字面量类型
    - 对于该类没有 virtual 基类
    - 不包含一个函数 try 块
    - 函数体必须是删除的，默认的或满足一些其他条件的。复合语句块必须满足常规函数的所有条件。所有构造函数初始化非静态成员的，包括基类，必须也是 `constexpr`。而且所有非静态数据成员必须被构造函数初始化。
    - 如果有默认拷贝或移动构造函数，那么该类不能包含任何 mutable  variant 成员

函数是 `constexpr` 不是隐式的 `const` （C++14），因此如果该函数不改变对象的逻辑状态就需要显式指定 `const`。函数是 `constexpr` 的，隐式为 `inline`。

另外，一个对象被声明为 `constexpr` 那么隐式地是 `const`，以下两式相等：

```
constexpr const unsigned int size = factorial(6);
constexpr unsigned int size = factorial(6);
```

当你需要同时用到 `constexpr` 和 `const` 声明时，需要注意它们指向声明的不同部分：

```
static constexpr int c = 42;
constexpr int const* p = &c; // p 是一个指向常量整数的 constexpr 指针
```

引用也可以是 `constexpr`，当且仅当，它们带静态存储时期或一个函数地引用一个对象 (they alias an object
with static storage duration or a function:)：

```
static constexpr int const & r = c;
```

## 运用正确的类型转换

进行类型转换时：
- 使用 `static_cast` 执行非多态类型的转换，包括整数到枚举，浮点数到整数或一种指针类型到另一种指针类型，比如从基类到派生类或从派生类到基类，但是没有任何运行时检查.

    ```
    enum options { one = 1, two, three };
    
    int value = 1;
    options op = static_cast<options>(value);
    
    int x = 42, y = 13;
    double d = static_cast<double>(x) / y;
    
    int n = static_cast<int>(d);
    ```

- 使用 `dynamic_cast` 执行从基类到派生类的多态类型的指针或引用的类型转换。在运行时期进行检查并需要 RTTI：

    ```
    struct base {
        virtual void run();
        virtual ~base();
    };
    
    struct derived : public base { };
    
    derived d;
    base b;
    
    base* pb = dynamic_cast<base*>(&d); // OK
    derived* pd = dynamic_cast<derived*>(&b); // fail
    
    try {
        base& rb = dynamic_cast<base&>(d); // OK
        derived& rd = dynamic_cast<derived&>(b); // fail
    } catch (std::bad_cast const& e) {
        std::cout << e.what() << std::endl;
    }
    ```

- 使用 `const_cast` 执行带有 `volatile` 和 `const` 说明符的类型和不带这些说明符的类型的转换。

    ```
    void old_api(char* str, unsigned int size)
    {
        // do something without changing the string.
    }
    
    std::string str{"sample"};
    old_api(const_cast<char*>(str.c_str()), static_cast<unsigned int>(str.size()));
    ```

- 使用 `reinterpret_cast` 执行 bit reinterpretation，比如从指针到整数，从一个指针类型到另一个指针类型，而不涉及任何运行时检查。

    ```
    class widget {
    public:
        using data_type = size_t;
        void set_data(data_type d) { data = d; }
        data_type get_data() const { return data; }
    private:
        data_type data;
    };
    
    widget w;
    user_data* ud = new user_data();
    // read
    w.set_data(reinterpret_cast<widget::data_type>(ud));
    // write
    user_data* ud2 = reinterpret_cast<user_data*>(w.get_data());
    ```

使用 C++ 提供的类型转换的好处：
- 能更好地表达用户的目的
- 对于不同的类型转换更安全（除了 `reinterpret_cast`）
- 容易在源代码中被搜索到

`static_cast` 在编译器实行转换，而且可以被用于隐式类型转化，反向的隐式类型转换，以及类层次体系中的指针转换。但是不能被用于无关类型的指针转换。比如从 `int *` 到 `double *` 会出现编译期错误，从 `base *` 到 `derived *` 不出现编译期错误但是出现运行时错误。

```
int* pi = new int(42);
double* pd = static_cast<double*>(pi); // compiler error.

base b;
derived* pd = static_cast<derived*>(&b); // compilers OK, runtime error.

int const c = 42;
int* p = static_cast<int*>(&c); // compiler error.
```

层次体系中的安全类型转换只能用 `dynamic_cast`。它只能用于指针或引用，如果转换指针失败，就返回一个 null 指针；如果转换一个引用失败时，就抛出 `std::bad_cast` 异常。因此，转换引用时要把 `dynamic_cast` 放在 `try...catch` 块中。如果试图用 `dynamic_cast` 去转换非多态类型，就会出现编译错误，因为它的实现机制是 RTTI，需要 type_info。

`reinterpret_cast` 会对二进制表示进行干预，应该小心使用。因为没有任何检查，所以它可以成功地将两个无关类型进行转换，比如 `int *` 到 `double *`：

```
int* pi = new int(42);
doule* pd = reinterpret_cast<double*>(pi);
```

`reinterpret_cast` 的一种典型用法：许多 API 以指针或整数的类型存储用户数据，然而如果你需要传递一个用户自定义的类型到这样的 API，你需要转换这种类型的指针为指针类型值或整型值，以方便存储。


`const_cast` 应该只用于当对象不以 `const` 或 `volatile` 声明时移去 `const` 或 `volatile`，其他情况会出现未定义的行为：

```
int const a = 42;
int const* p = &a; // p is non-const a pointer to a const object.
int* q = const_cast<int*>(p); // undefined behavior.
```

## 使用 `unique_ptr` 单独持有内存资源

使用 `std::unique_ptr` 的典型操作和注意事项：
- 使用可用的重载构造函数来创建 `unique_ptr` 来管理对象或者通过指针管理一个数组对象。默认构造函数创建的指针不会管理任何对象：

    ```
    std::unique_ptr<int> pnull;
    std::unique_ptr<int> pi(new int(42));
    std::unique_ptr<int[]> pa(new int[3] { 1, 2, 3 ]);
    std::unique_ptr<foo> pf(new foo(42, 42.0, "32"});
    ```

- 在 C++14 中可以用 `std::make_unique()` 函数模板创建一个 `unique_ptr` 对象：

    ```
    std::unique_ptr<int> pi = std::make_unique<int>(42);
    std::unique_ptr<int[]> pa = std::make_unique<int[]>(3);
    std::unique_ptr<foo> pf = std::make_unique<foo>(32, 23.0, "32");
    ```

- 如果默认删除操作对于管理的对象或数组不合适，可以使用删除器：

    ```
    struct foo_deleter {
        void operator()(foo* pf) const
        {
            std::cout << "deleting foo..." << std::endl;
            delete pf;
        }
    };
    
    std::unique_ptr<foo, foo_delete> pf(new foo(32, 32.0, "23"),
                                        foo_delete());
    ```

- 使用 `std::move` 转交对象的持有关系：

    ```
    auto pi = std::make_unique<int>(32);
    auto qi = std::move(pi);
    assert(pi.get() == nullptr);
    assert(qi.get() != nullptr);
    ```

- 返回对象的 raw pointer，使用 `get()` 可以继续持有对象，使用 `release()` 不会继续持有对象：

    ```
    void func(int* ptr)
    {
        if (ptr != nullptr) {
            std::cout << *ptr << std::endl;
        } else {
            std::cout << "null" << std::endl;
        }
    
        std::unique_ptr<int> pi;
        func(pi.get()); // prints null.
    
        pi = make_unique<int>(32);
        func(pi.release()); // print 32.
    }
    ```

- 可以使用 `operator*` 和 `operator->` 操作指向对象的指针：

    ```
    auto pi = std::make_unique<int>(32);
    *pi = 21;
    
    auoto pf = std::make_unique<foo>();
    pf->print();
    ```

- 如果管理的是一个数组对象，可以使用 `operator[]` 来访问数组内的单个元素：

    ```
    std::unique_ptr<int[]> pa = std::make_unique<int[]>(3);
    for (int i = 0; i < 3; ++i) {
        pa[i] = i + 1;
    }
    ```

- 为检查是否能管理一个对象，可以使用显式的 `operator bool` 或检查 `get() != nullptr`：

    ```
    std::unique_ptr<int> pi(new int(32));
    if (pi) {
        std::cout << "not null" << std::endl;
    }
    ```

- `unique_ptr` 可用于存储在容器中。使用 `make_unique` 返回的对象可以直接存储：

    ```
    std::vector<std::unique_ptr<foo>> data;
    for (int i = 0; i < 5; ++i) {
        data.push_back(std::make_unique<foo>(i, i, std::to_string(i)));
    }
    auto pf = make_unique<foo>(32, 32.0, "32");
    data.push_back(std::move(pf));
    ``` 

支持的定制删除器可以是函数对象，也可以是指向函数对象或函数的 lvalue reference，这个可调用对象必须有一个类型为 `unique_ptr<T, Deleter>::pointer` 的参数。

C++14 中添加了 `std::make_unique` 函数创建 `unique_ptr`，避免了一些情况下的内存泄露，但是有一些限制：
- 只能用于分配数组，但不能同时初始化它们：

    ```
    std::unique_ptr<int[]> pa(new int[3]{1, 2, 3});
    
    std::unique_ptr<int[]> pa = std::make_unique<int[]>(3);
    for (int i = 0; i < 3; ++i) {
        pa[i] = i + 1;
    }
    ```

- 不能用于带有用户定义删除器的 `unique_ptr` 对象。

常量 `unique_ptr` 对象不能转移管理对象或数组的拥有权给另一个 `unique_ptr` 对象。

管理一个 Dervied 对象的 `unique_ptr` 可以隐式转换为管理 Base 对象的 `unique_ptr` （Derived 派生自 Base），只有当 Base 有一个虚析构函数时该转换才是类型安全的，否则会出现未定义行为：

```
struct Base {
    virtual ~Base()
    {
        std::cout << "~Base()" << std::endl;
    }
};

struct Derived : public Base {
    virtual ~Derived()
    {
        std::cout << "~Derived" << std::endl;
    }
};

std::unique_ptr<Derived> pd = std::make_unique<Derived>();
std::unique_ptr<Base> pb = std::move(pd);
```


## 使用 `shared_ptr` 共享内存资源

使用 `shared_ptr` 和 `weak_ptr` 注意：
- 当管理一个数组对象时总要指定一个删除器。删除器可以是`std::default_delete` 的偏特化或者以指针作为模板类型的函数：

    ```
    std::shared_ptr<int> pa1(new int[3]{1, 2, 3},
                             std::default_delete<int[]>());
    
    std::shared_ptr<int> pa2(new int[3]{1, 2, 3},
                             [](auto p) { delete[] p; });
    ```

- 在 C++17 中，如果管理的是一个数组对象，那么可以用 `operator[]` 来访问数组内的单个元素：

    ```
    std::shared_ptr<int> pa1(new int[3]{1, 2, 3},
                             std::default_delete<int[]>());
    
    for (int i = 0; i < 3; ++i) {
        pa1[i] *= 2;
    }
    ```

- `weak_ptr` 不参与引用计数：

    ```
    auto sp1 = std::make_shared<int>(32);
    assert(sp1.use_count() == 1);
    
    std::meak_ptr<int> wp1 = sp1;
    assert(sp1.use_count() == 1);
    
    auto sp2 = wpi.lock();
    assert(sp1.use_count() == 2);
    assert(sp2.use_count() == 2);
    
    sp1.reset();
    assert(sp1.use_count() == 0);
    assert(sp2.use_count() == 1);
    ```

- 当你需要为一个已经被另一个 `shared_ptr` 对象所管理的实例创建 `shared_ptr` 对象时，使用 `std::enable_shared_from_this` 类模板作为一个类型的基类。

    ```
    struct Apprentice;
    
    struct Master : std::enable_shared_from_this<Master> {
        ~Master() { std::cout << "~Master" << std::endl; }
        void take_apprentice(std::shared_ptr<Apprentice> a);
    private:
        std::shared_ptr<Apprentice> apprentice;
    };
    
    struct Apprentice {
        ~Apprentice() { std::cout << "~Apprentice" << std::endl; }
        void take_master(std::weak_ptr<Master> m);
    private:
        std::weak_ptr<Master> master;
    };
    
    // shared_fron_this() return a shared_ptr.
    void Master::take_apprentice(std::shared_ptr<Apprentice> a)
    {
        apprentice = a;
        apprentice->take_master(shared_from_this());
    }
    
    void Apprentice::take_master(std::weak_ptr<Master> m)
    {
        master = m;
    }
    ```

C++17 中添加了一些新的非成员函数：`std::static_pointer_cast()`，`std::dynamic_pointer_cast()`，`std::const_pointer_cast`，`std::reinterpret_pointer_cast`，用来把 `shared_ptr` 为指定的类型:

```
std::shared_ptr<Derived> pd = std::make_shared<Derived>();
std::shared_ptr<Base> pb = pd;

std::static_pointer_cast<Derived>(pb)->print();
```

`shared_from_this()` 返回一个 `shared_ptr`。C++17 中可以用 `weak_from_this()` 返回一个对象的 non-owning 引用。这些方法只能在一个被现存的 shared_ptr 管理的对象上调用，否则 C++17 里会抛出 `std::bad_weak_ptr` 异常，C++17 前该行为是未定义的。 

如果不用 `std::enable_shared_from_this` 而是直接创建一个 `shared_ptr<T>(this)` 将会导致多个 `shared_ptr` 各自管理同一个对象，但不知道彼此之间的存在，这样对象会被销毁多次。

```
struct S {
    shared_ptr<S> dangerous()
    {
        return shared_ptr<S>(this); // don't do this!
    }
};

int main()
{
    shared_ptr<S> sp1(new S);
    shared_ptr<S> sp2 = sp1->dangerous();
}
```

上例中，sp1 拥有 S 对象，但是 `S::dangerous` 不知道它自己是  sp1 的 `shared_ptr` 对象，所以会为 sp2 创建一个 `shared_ptr` 管理该对象， 所以 sp1 和 sp2 同时管理同一个对象但是不知道彼此的存在，出了作用域时，对象会被它们释放两次。

## 实现移动语义

移动拷贝构造函数和移动赋值运算符的使用会带来性能的提升。

`std::move` 可以显式地把一个左值转换为对应的右值引用类型。
