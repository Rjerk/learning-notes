# 使用 Boost.Test 进行单元测试

## 安装

测试 Boost 的安装完成：

```
// g++ test.cpp -o test -lboost_unit_test_framework

#define BOOST_TEST_MODULE My first test module.
#include <boost/test/included/unit_test.hpp>

BOOST_AUTO_TEST_CASE(first_test_fcuntion)
{
    BOOST_TEST(true);               
}
```

输出：

```
Running 1 test case...

*** No errors detected
```

所有关于该文的代码：[这里](https://github.com/Rjerk/snippets/tree/master/boost-prac/boost-test)

## Test 库内容

Test 库包含三个变体：
- 单独的头文件：让你写测试程序时无需构建该库；你只需要 include 以下一个单独的头文件。它的限制是对该模块你只有一个翻译单元；然而，你仍可以分离该模块到多个头文件中，这样你就能在不同的文件中分隔不同的测试组件。
- 静态库：使你分离跨不同翻译单元的一个模块，但是该库需要先作为一个静态库被构建出来。
- 共享库：和静态库的使用场景一样，但是它的优点是，对于有许多测试模块的程序，这个库只被链接一次，生成一个小得多的二进制块。然而，共享库必须在运行时获得。

一些术语和概念：
- **测试模块**是一个执行测试的程序。有两种类型的模块：单独文件（single-file）（当你使用单独头文件变体时）和多文件（multifile）（当你使用静态或共享变体）。
- **测试断言**是一个被测试模块检查的条件。
- **测试用例**是一组或多组测试断言，分别被测试模块用于执行和检查，如果某个失败或泄露未捕获的异常，其他测试的执行不会被停止。
- **测试组件**是一个或多个测试或测试组件的集合。
- **测试单元**是测试用例或测试组件。
- **测试树**是测试单元的分层结构。在该结构中，测试用例是叶结点，测试组件是非叶结点。
- **测试运行器**是这样一个构件，给它一个测试树，就会执行必要的初始化，测试的执行，并产生报告。
- **测试日志**是所有发生在测试模块执行过程中的所有事件的记录。
- **测试设置**是测试模块的一部分，负责框架的初始化，测试树的构建和独立测试用例的设置。
- **测试清理**是测试模块的一部分，负责清理操作。
- **测试设施**是一对设置和清理操作，被多个测试单元调用，来避免重复代码。

## 书写测试

在上面的代码实例中：
1. `#define BOOST_TEST_MODULE My first test moduel` 为模块初始化定义一个存根并为主测试组件设置一个名称。必须定义在任何库头文件之前。
2. `#include <boost/test/included/unit_test.hpp>` 一个 single-header 库，包含了所有其他必要的头文件。
3. `BOOST_AUTO_TEST_CASE(first_test_function)` 声明一个无形参的测试用例并自动注册它，使它作为封闭测试组件中测试树的一部分。该例中测试组件是被 `BOOST_TEST_MODULE` 定义的主测试组件。
4. `BOOST(true);` 执行测试断言。

### 自己写 main()

如果不想让库生成 `main()` 函数，而是想自己写。需要添加宏，`BOOST_TEST_NO_MAIN` 和 `BOOST_TEST_ALTERNATIVE_INIT_API` 到任何库头文件之前。那么，在 `main()` 函数中，就要提供称为 `init_unit_test()` 默认初始化函数作为参数来调用称为 `unit_test_main()` 的默认测试运行器：

```
#define BOOST_TEST_MODULE My first test module
#define BOOST_TEST_NO_MAIN
#define BOOST_TEST_ALTERNATIVE_INIT_API
#include <boost/test/included/unit_test.hpp>

BOOST_AUTO_TEST_CASE(first_test_function)
{
    BOOST_TEST(true);
}

int main(int argc, char* argv[])
{
    return boost::unit_test::unit_test_main(init_unit_test, argc, argv);
}
```

### 自己写初始化函数

也可以定制测试运行器的初始化函数。在这种情况中，必须移除 `BOOST_TEST_MODULE` 宏的定义并写一个无任何参数和返回 `bool` 值的初始化函数：

```
#define BOOST_TEST_NO_MAIN
#define BOOST_TEST_ALTERNATIVE_INIT_API
#include <boost/test/included/unit_test.hpp>

BOOST_AUTO_TEST_CASE(first_test_function)
{
    BOOST_TEST(true);
}

bool custom_init_unit_test()
{
    std::cout << "test runner custom init" << std::endl;
    return true;
}

int main(int argc, char* argv[])
{
    return boost::unit_test::unit_test_main(custom_init_unit_test, argc, argv);
}
```

### 写测试用例和测试组件

Boost.Test 提供了自动的和手动的方法来登记测试用例和测试组件，给测试运行器执行。

```
class point3d
{
    int x_;
    int y_;
    int z_;
public:
    point3d(int const x = 0, int const y = 0, int const z = 0)
        :x_(x), y_(y), z_(z) {}
    int x() const { return x_; }
    point3d& x(int const x) { x_ = x; return *this; }

    int y() const { return y_; }
    point3d& y(int const y) { y_ = y; return *this; }

    int z() const { return z_; }
    point3d& z(int const z) { z_ = z; return *this; }

    bool operator==(point3d const & pt) const
    {
        return x_ == pt.x_ && y_ == pt.y_ && z_ == pt.z_;
    }

    bool operator!=(point3d const & pt) const
    {
        return !(*this == pt);
    }

    bool operator<(point3d const & pt) const
    {
        return x_ < pt.x_ || y_ < pt.y_ || z_ < pt.z_;
    }

    friend std::ostream& operator<<(std::ostream& stream,
    point3d const & pt)
    {
        stream << "(" << pt.x_ << "," << pt.y_ << "," << pt.z_ << ")";
        return stream;
    }
    
    void offset(int const offsetx, int const offsety, int const offsetz)
    {
        x_ += offsetx;
        y_ += offsety;
        z_ += offsetz;
    }

    static point3d origin() { return point3d{}; }
};
```

使用下列宏来创建测试单元：
- 创建一个测试组件，使用 `BOOST_AUTO_TEST_SUITE(name)` 和 `BOOST_AUTO_TEST_SUITE_END()`：

    ```
    BOOST_AUTO_TEST_SUITE(test_contruction)
    // test cases
    BOOST_AUTO_TEST_SUITE_END()
    ```

- 创建一个测试用例，使用 `BOOST_AUTO_TEST(name)`，测试用例定义在 `BOOST_AUTO_TEST_SUITE(name)` 和 `BOOST_AUTO_TEST_SUITE_END()` 之间：

    ```
    BOOST_AUTO_TEST_CASE(test_construction)
    {
        auto p = point3d{ 1, 2, 3 };
        BOOST_TEST(p.x() == 1);
        BOOST_TEST(p.y() == 2);
        BOOST_TEST(p.z() == 4); // will fail.
    }

    BOOST_AUTO_TEST_CASE(test_origin)
    {
        auto p = point3d::origin();
        BOOST_TEST(p.x() == 0);
        BOOST_TEST(p.y() == 0);
        BOOST_TEST(p.z() == 0);
    }
    ```

- 创建一个嵌套测试组件，定义一个测试组件在另一个测试组件中：

    ```
    BOOST_AUTO_TEST_SUITE(test_operations)
    BOOST_AUTO_TEST_SUITE(test_methods)

    BOOST_AUTO_TEST_CASE(test_offset)
    {
        auto p = point3d{ 1, 2, 3 };
        p.offset(1, 1, 1);
        BOOST_TEST(p.x() == 2);
        BOOST_TEST(p.y() == 3);
        BOOST_TEST(p.z() == 3); // will fail.
    }

    BOOST_AUTO_TEST_SUITE_END()
    BOOST_AUTO_TEST_SUITE_END()
    ```

- 为添加装饰器到一个测试单元中，加入另一个形参到测试单元的宏中。装饰器可以是 description, label, precondition, dependency, fixture 等。

    ```
    BOOST_AUTO_TEST_SUITE(test_operations)
    BOOST_AUTO_TEST_SUITE(test_operators)

    BOOST_AUTO_TEST_CASE(
        test_equal,
        *boost::unit_test::description("test operator==")
        *boost::unit_test::label("opeq"))
    {
        auto p1 = point3d{ 1, 2, 3 };
        auto p2 = point3d{ 1, 2, 3 };
        auto p3 = point3d{ 3, 2, 1 };
        BOOST_TEST(p1 == p2);
        BOOST_TEST(p1 == p3); // will fail.
    }

    BOOST_AUTO_TEST_CASE(
        test_not_equal,
        *boost::unit_test::description("test operator!=")
        *boost::unit_test::label("opeq")
        *boost::unit_test::depends_on(
            "test_operations/test_operators/test_equal"))
    {
        auto p1 = point3d{ 1, 2, 3 };
        auto p2 = point3d{ 3, 2, 1 };
        BOOST_TEST(p1 != p2);
    }

    BOOST_AUTO_TEST_CASE(test_less)
    {
        auto p1 = point3d{ 1, 2, 3 };
        auto p2 = point3d{ 1, 2, 3 };
        auto p3 = point3d{ 3, 2, 1 };
        BOOST_TEST(!(p1 < p2));
        BOOST_TEST(p1 < p3);
    }

    BOOST_AUTO_TEST_SUITE_END()
    BOOST_AUTO_TEST_SUITE_END()
    ```

### 运行测试

运行测试时：
- 执行整个测试树，不带任何参数地运行程序。
- 执行一个单独的测试组件，运行程序时使用参数 `run_test` 来指定测试组件的路径：

    ```
    ./test03 --run_test=test_construction
    ```

- 执行一个单独的测试用例，运行程序时使用参数 `run_test` 来指定测试用例的路径：

    ```
    ./test03 --run_test=test_constrution/test_origin
    ```

- 执行定义在相同 label 下的一组测试组件和测试用例，运行程序时使用参数 `run_test` 指定以 `@` 作为前置的标签名：

    ```
    ./test03 --run_test=@opeq
    ```

### 内部机制

测试树由测试组件和测试用例构成。测试组件包含一个或多个测试用例和其他嵌套的测试组件。测试组件有点像名称空间，可以在同一个文件或不同文件多次出入。

测试组件的自动注册用 `BOOST_AUTO_TEST_SUITE` 完成，需要一个名字，以及用 `BOOST_AUTO_TEST_SUITE_END` 进行结束。

测试用例的自动注册用 `BOOST_AUTO_TEST_CASE` 完成。

测试单元成为最近测试组件的成员，可以是用例或组件。如果测试单元定义在文件域级中，则是主测试组件（用 `BOOST_TEST_MODULE` 声明来隐式创建的测试组件）的成员。

测试组件和测试用例可以用一系列属性来进行装饰，用来影响在测试模块的执行过程中，测试单元如何被处理。有下列装饰器：
- `depends_on`: 指示当前测试单元和一个指派的测试单元之间的依赖
- `descrition`: 提供测试单元的描述
- `enabled/disabled`: 设置测试单元的默认运行状态，true 或 false
- `enable_if`: 根据编译器表达式的计算结果设置测试单元的默认运行状态
- `fixture`: 指定一对函数（启动和清理）在测试单元执行之前和之后进行调用
- `label`: 给测试单元关联一个标签；相同标签可用于多个测试单元；一个测试单元可以有多个标签
- `precondition`: 给测试单元关联一个断言，用于运行时期决定测试单元的运行状态

如果测试用例的执行导致一个未处理的异常，该框架会捕获异常并终止测试用例的进行。然而，该框架也提供了一些宏来测试是否抛出或不抛出异常。

构成模块的测试树的测试单元可以被全部或部分执行。用 `--run_test` 命令行选项可以过滤掉测试单元并指定路径或标签。

## 断言

`Boost.Test` 库提供了一些以宏为形式的 API 来写测试。
- `BOOST_TEST`，最简单的形式，用于大多数测试：

    ```
    int a = 2, b = 4;
    BOOST_TEST(a == b);

    BOOST_TEST(2.101 == 2.100);

    std::string s1 {"sample"};
    std::string s2 {"text"};
    BOOST_TEST(s1 == s2);
    ```

- `BOOST_TEST` 和 `tolerance()` 一起使用，指示浮点数比较的误差：

    ```
    BOOST_TEST(2.101 == 2.100, boost::test_tools::tolerance(0.001));
    ```

- `BOOST_TEST` 和 `per_element()` 一起使用，执行容器的按元素逐个进行比较：

    ```
    std::vector<int> v{ 1, 2, 3 };
    std::vector<int> l{ 1, 2, 3 };
    
    BOOST_TEST(v == l, boost::test_tools::per_elements());
    ```

- `BOOST_TEST` 和三元操作符，带 `||` 或 `&&` 的复合语句一起使用，需要额外的小括号：

    ```
    BOOST_TEST((a > 0 ? true : false));
    BOOST_TEST((a > 2 && b < 5));
    ```

- `BOOST_ERROR` 用于无条件地失败测试并产生一条信息在报告中：

    ```
    BOOST_ERROR("this test will fail");
    ```

- `BOOST_TEST_WARN` 用于在报告中产生一个警告以防测试失败的情况，它不增加遭遇错误的数量并停止测试用例的执行：

    ```
    BOOST_WARN(a == 4, "something is not right");
    ```

- `BOOST_TEST_REQUIRE` 用于确保测试用例的先决条件满足，否则测试用例停止执行：

    ```
    BOOST_TEST_REQUIRE(A == 4, "this is critical");
    ```

- `BOOST_FAIL` 用于无条件停止测试用例的执行，增加遭遇错误的数量，并在报告中产生信息，等同于 `BOOST_TEST_REQUIRE(false, message);`：

    ```
    BOOST_FAIL("must be implemented");
    ```

- `BOOST_IS_DEFINED` 用于检查一个特殊的预处理器符号是否定义于运行时期。和 `BOOST_TEST` 一起使用实行验证和记录：

    ```
    BOOST_TEST(BOOST_IS_DEFINED(UNICODE));
    ```

一些宏用于测试一个特殊的异常是否在表达式计算的过程中抛出，在下面的列表中，`<level>` 是 `CHECK`， `WARN` 或 `REQUIRE`：
- `BOOST_<level>_NO_THROW(expr)` 检查异常是否从 `expr` 里面抛出。任何在 `expr` 计算过程中抛出的异常都被捕获而且无法传播到测试体之外。如果异常出现，则断言失败。
- `BOOST_<level>_THROW（expr, exception_type)` 检查异常 `exception_type` 是否从 `expr` 里面抛出。如果未抛出异常，则断言失败。任何其他的异常不被捕获且会传播到测试体之外。
- `BOOST_<level>_EXCEPTION(expr, exception_type, predicate)` 检查是否有个 `exception_type` 异常从 `expr` 里抛出。如果抛出，则传递表达式到进一步检查的判定之中。如果没有异常抛出或不同于 `exception_type` 的异常抛出，则此断言类似于 `BOOST_<level>_THROW`。

## fixture

测试模块越大而且测试用例越相似，让这些测试用例需要相同的设置，清理，可能相同的数据的可能性越大。包含这些的一个部件叫做测试固定装置（test fixture）或测试上下文（test context）。Boost.Test 提供了一些方法为测试用例，测试组件或模块来定义测试固定装置。

```
struct standart_fixture {
    standart_fixture() { BOOST_TEST_MESSAGE("setup"); }
    ~standart_fixture() { BOOST_TEST_MESSAGE("cleanup"); }
    int n { 42 };
};

struct extended_fixture {
    std::string name;
    int data;

    extended_fixture(std::string const& n = "")
        : name(n), data(0)
    {
        BOOST_TEST_MESSAGE("setup" + name);
    }

    ~extended_fixture()
    {
        BOOST_TEST_MESSAGE("cleanup" + name);
    }
};

void fixture_setup()
{
    BOOST_TEST_MESSAGE("fixture setup");
}

void fixture_cleanup()
{
    BOOST_TEST_MESSAGE("fixture cleanup");
}
```

对于一个或多个测试单元，使用下面的方法定义测试固定装置：
- 为特定测试用例定义一个测试固定装置，使用 `BOOST_FIXTURE_TEST_CASE` 宏：

    ```
    BOOST_FIXTURE_TEST_CASE(test_case, extended_fixture)
    {
        ++data;
        BOOST_TEST(data == 1);
    }
    ```

- 为在一个测试组件中的所有测试用例定义一个测试固定装置，使用 `BOOST_FIXTURE_TEST_SUITE`：

    ```
    BOOST_FIXTURE_TEST_SUITE(suite1, extended_fixture)
    
    BOOST_AUTO_TEST_CASE(case1)
    {
        BOOST_TEST(data == 0);
    }
    
    BOOST_AUTO_TEST_CASE(case2)
    {
        ++data;
        BOOST_TEST(data == 1);
    }
    
    BOOST_AUTO_TEST_SUITE_END()
    ```

- 为在一个测试组件中的所有测试用例定义一个测试固定装置，除了一个或一些测试单元，使用 `BOOST_FIXTURE_TEST_SUITE` 然后覆写它到一个测试单元，对于测试用例使用 `BOOST_FIXTURE_TEST_CASE`，对于嵌套的测试组件，使用 `BOOST_FIXTURE_TEST_SUITE`：

    ```
    BOOST_FIXTURE_TEST_SUITE(suite2, extended_fixture)
    
    BOOST_AUTO_TEST_CASE(case1)
    {
        BOOST_TEST(data == 0);
    }
    
    BOOST_FIXTURE_TEST_CASE(case2, standart_fixture)
    {
        BOOST_TEST(n == 42);
    }
    
    BOOST_AUTO_TEST_SUITE_END()
    ```

- 为测试用例或测试组件定义多个固定装置，用带有 `BOOST_AUTO_TEST_SUITE` 和 `BOOST_AUTO_TEST_CASE` 的`boost::unit_test::fixture`：

    ```
    BOOST_AUTO_TEST_CASE(test_case_multifix,
        * boost::unit_test::fixture<extended_fixture>(std::string("fix1"))
        * boost::unit_test::fixture<extended_fixture>(std::string("fix2"))
        * boost::unit_test::fixture<standart_fixture>())
    {
        BOOST_TEST(true);
    }
    ```

- 为模块一个测试固定装置，使用 `BOOST_GLOBAL_FIXTURE`：

    ```
    BOOST_GLOBAL_FIXTURE(standart_fixture);
    ```

TEST 库支持一些固定装置的模型：
- 一个类模型，构造函数的功能如设置函数，析构函数的功能如清理函数。一个扩展的模块允许构造函数有一个参数。在上面的例子里，`standart_fixture` 实现为第一个模块， `extended_fixture` 实现为第二个模块。
- 一对自由函数模型：一个定义设置，另一个（可选地）定义清除代码。如上面例子中的 `fixture_setup()` 和 `fixture_cleanup()`。

当 fixture 实现为类时，可以有数据成员，这些数据成员可以在测试单元是可访问的。如果一个 fixture 是为了测试组件而定义，那么在该测试组件下的所有测试单元都能隐式地访问到它的成员。如果一个测试组件中包含一个重新定义了该 fixture 的测试组件，那么该 fixture 的定义能被离它最近的测试组件里的测试单元看到。

多个 fixture 装饰器可以用 `operator*` 组合在一起，这个方法的一个缺点是如果用一个带有数据成员的类作为 fixture 装饰器，这些成员将无法被测试单元所访问到。

当每个测试用例执行时，一个新的 fixture 对象被构建，测试用例结束时，fixture 对象就被销毁。

一个全局的 generic 使用通用的测试类模型（带有默认构造函数的模型），可以定义任意数量的全局 generic 在测试文件域中。

## 控制输出

Boost.Test 提供了定制测试日志和测试报告和结果格式的能力，当前支持两种：用户可读的格式和 XML。

用户可以创建和添加自己的格式，在输出中的显示配置既可以通过命令行开关在运行时期完成，也可以在编译时期用各种 API 完成。

```
#define BOOST_TEST_MODULE Controlling output
#include <boost/test/included/unit_test.hpp>

BOOST_AUTO_TEST_CASE(first_test_fcuntion)
{
    BOOST_TEST(true);
}

BOOST_AUTO_TEST_SUITE(test_suite)

BOOST_AUTO_TEST_CASE(test_case)
{
    int a = 32;
    BOOST_TEST(a == 3);
}

BOOST_AUTO_TEST_SUITE_END()
```

控制测试日志输出：
- 使用 `--log_format=<format>` 或 `-f <format>` 命令行选项指定日志格式。`format` 可以为 `HRF`（默认值），`XML` 或 `JUNIT`。
- 使用 `--log_level=<format>` 或 `-l <level>` 命令行选项指定日志等级。`level` 可以为 `error`（对于 HRF 和 XML 是默认的），`warning`，`all` 或 `success`（对于 JUNIT 是默认的）。
- 使用 `--log_sink<stream or file name>` 或 `-k <stream or file name>` 命令行选项指定写入测试日志的位置。可以是 `stdout`（对于 HRM 和 XML 是默认的），`stderr` 或任意文件名（对于 JUNIT 是默认的）。

控制报告输出：
- 使用 `--report_format=<format>` 或 `-m <format>` 命令行选项指定报告格式。`format` 可以为 `HRF`（默认值），`XML`。
- 使用 `--report_level=<level>` 或 `-r level` 命令行选项指定报告等级。`level` 可以为 `confirm`（对于 HRF 和 XML 是默认的），`no`，`short` 或 `detailed`（对于 JUNIT 是默认的）。
- 使用 `--report_sink<stream or file name>` 或 `-e <stream or file name>` 命令行选项指定写入报告的位置。可以是 `stdout`（对于 HRM 和 XML 是默认的），`stderr` 或任意文件名（对于 JUNIT 是默认的）。

可以指定一个单独的文件来报告内存泄露，使用 `--report_memory_leaks_to=<file name>` 命令行选项，如果该选项未指定而内存泄露发送，则报告至标准错误流。