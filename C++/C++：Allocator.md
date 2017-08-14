# C++：Allocator

Allocator 为访问/寻址、内存分配/释放以及对象的构造/析构提供了一种封装策略。

Allocator 用来解决不同平台的内存模型差异问题，把底层的内存分配与释放以策略模式进行封装，主要供 STL 内部实现使用，对 STL 用户几乎是透明的。

有如下三种 Allocator:
- allocator 默认分配器
- scoped_allocator_adaptor 实现多级容器的多级分配器
- polymorphic_allocator 基于 `std::memory_resource` 构造，支持运行时多态机制的分配器

## std::allocator

```
#include <memory>

template< class T >
struct allocator;
```

`std::allocator` 模板类是所有标准库容器的默认 Allocator。可以将内存分配和对象构造分离开，提供了一种类型感知的内存分配方法，它分配的内存是原始的、未构造的。

## std::scoped_allocator_adaptor

```
#include <scoped_allocator>

template< class OuterAlloc, class... InnerAlloc >
class scoped_allocator_adaptor : public OuterAlloc;
```

`std::scoped_allocator_adaptor` 模板类用于多级容器。

如果想要一个内含 String 的容器并想要为容器和其中元素使用相同的分配器，如果这样做：

```
template <typename T>
using Allocator = SomeFancyAllocator<T>;

using String = std::basic_string<char, std::char_traits<char>, Allocator<char>>;
using Vector = std::vector<String, Allocator<String>>;

Allocator<String> as( some_memory_resource);
Allocator<char> ac(as);
Vector v(as);
v.push_back(String("hello", ac));
v.push_back(String("world", ac));
```

然而这样容易插入一个使用一个不同 allocator 的 string 导致错误：

```
v.push_back(String("opps, not using same memory resource"));
```

`std::scoped_allocator_adaptor` 的目的就在于如果支持用一个 allocator 来构造就可以自动传递 allocator 给它构造的对象。

```
template <typename T>
using Allocator = SomeFancyAllocator<T>;

using String = std::basic_string<char, std::char_traits<char>, Allocator<char>>;

using Vector = std::vector<String, std::scoped_allocator_adaptor<Allocator<String>>>;

Allocator<String> as( some_memory_resource);
Allocator<char> ac(as);
Vector v(as);
v.push_back(String("hello"));
v.push_back(String("world"));
```

现在 vector 的构造器被自动使用来构建其元素，甚至被插入的元素 String("hello") 和 String("world") 使用的也不是同一个构造器。因为 `basic_string` 被隐式地声明为 `const char*`，所以最后两行可以简化为：

```
v.push_back("hello");
v.push_back("world");
```

使得代码更简洁易读，也不容易出错。

## std:: pmr::polymorphic_allocator

```
#include <memory_resource>

template< class T >
class polymorphic_allocator;
```

`std::pmr::polymorphic_allocator` 的分配行为取决于它所构建的内存资源。运行时多态机制使得对象使用 `std::pmr::polymorphic_allocator` 像是在使用不同的 allocator。 

如果使用一个带有特定 allocator 的 vector，可以使用 Allocator 模板参数：

```
auto vec = std::vector<int, my_allocator>();
```

然而当另一个 vector 使用不同的 allocator 时，它们就不是同一个类型的 vector，即使它们的元素类型是一样的:

```
auto vec1 = std::vector<int, my_allocator>();
auto vec2 = std::vector<int, other_allocator>();
auto vec = vec1; // ok
vec = vec2; // error
```

一个多态 allocator 可以通过动态分派定义 allocator 的行为（而不是使用模板机制）。

```
class my_memory_resource : public std::pmr_memory_resource { ... };

my_memory_resource mem_res1;
auto vec1 = std::pmr::vector<int>(0, mem_res1);

class other_memory_resource : public std::pmr_memory_resource { ... };

other_memory_resource mem_res2;
auto vec2 = std::pmr::vector<int>(0, mem_res2);

auto vec = vec1; // type is std::pmr::vector<int>
vec = vec2; // ok, have the same type.
```
