# C++ 内存管理

## 程序使用内存分区

一个程序要运行，就要将可执行程序加载到内存中，形成一个运行空间。

一个程序占用的内存区分为：

1. 代码区：存放着程序的执行代码。

2. 数据区：存放着代码的全局数据、静态变量、常量等。

3. 堆：用户控制的动态内存区，供程序申请使用。

4. 栈：栈中存储自动变量或者局部变量以及传递的函数参数等。

除了代码区不是我们能在代码中直接控制的，其他三个区都是我们可以利用的。

在 C++ 中将数据又被分为自由存储区、全局/静态存储区、常量存储区，再加上堆区、栈区，总共分为5个区。

- 自由存储区

自由存储区是那些由 malloc() 等分配的内存块，和堆相似，但是是用 free() 来结束生命期的。

- 全局/静态存储区

存储全局变量以及静态变量。这部分存储区在程序编译阶段就已经分配好，在程序的整个运行期间都会存在。在 C 语言里，全局变量又分为初始化和未初始化的，但在 C++ 里没有对此区分，它们公用同一块存储区。

- 常量数据区

存储程序中的常量字符串等，是不可修改的内存区域。如下面的程序代码会在运行是导致程序退出：

```
char* pstr = "localstring";
pstr[1] = 'a'; // 试图访问不能访问的区域，出现 段错误 （segment fault）
```

- 堆

在堆中，我们使用 new/delete 手动分配/释放动态内存，一般 new 和 delete 相对应，如果分配了却没有释放，就会发生内存泄漏。

- 栈

在执行函数时，函数内局部变量的存储单元都可在栈上创建，函数执行结束时这些存储单元将被自动释放。

计算机在底层对栈提供支持：会分配专门的寄存器存放栈的地址，而入栈出栈都会有专门的指令去执行，使得栈的效率比较高。

## 内存分配方式 

三种内存分配方式：

1. 从静态存储区域分配。内存在程序编译时就已经分配好了，这些内存在程序的整个运行期间都会存在。如全局变量，static 变量。

2. 从栈分配。在函数执行期间，函数内局部变量（包括形参）的存储单元都创建在堆栈上，函数结束时这些存储单元自动释放（堆栈清退）。堆栈内存分配的运算内置于处理器的指令集中，效率很高，一般不存在失败的危险，但是分配的内存容量有限，可能出现堆栈溢出。

3. 从堆（heap）或自由存储空间分配，也叫动态内存分配。程序在运行期间用 malloc() 或 new 申请一定的内存，其生存期由程序员调用 free() 或 delete 决定。

在堆上动态分配内存需要一定的开销：

- 应用程序调用操作系统的内存管理模块的堆管理器，搜索符合要求的空闲的连续字节内存。在多次动态分配后，可能出现很多内存碎片，需要对碎片进行整理和合并才能分配成功，会浪费很多时间。
- 如果动态分配失败，需要检查返回值或捕获异常，这也需要开销。
- 动态创建的对象可能被删除多次，甚至在删除后继续使用，或者没有被删除，造成内存泄漏问题。

### C++ 里与内存相关的基础构件

#### malloc()/free()

```
#include <cstdlib>

void* malloc(size_t size);
void free(void* ptr);
```

malloc() 分配 size 大小的**未初始化**的内存块，如果成功返回指向该内存块第一个字节的指针 ptr，失败返回 NULL，如果 size 为 0 也会返回 NULL。

free() 释放 ptr 指向的内存块，如果 ptr 指向的内存块已经被 free() 释放过，则 free(ptr) 会产生未定义行为，如果 ptr 为 NULL，无操作执行。

#### new-expression/delete-expression

- new-expression

```
::(optional) new (placement_params)(optional) ( type ) initializer(optional)	(1)	
::(optional) new (placement_params)(optional) type initializer(optional)	(2)	
// (optional) 指示前面的表达式是可选的
```

使用 new 动态分配并**初始化**一个对象并返回指向该对象的指针，对于内置类型或组合类型来说对象的值是未定义的，而类类型对象会用默认构造函数来初始化。

```
int* pi = new int; // pi 指向未初始化的 int
int* pii = new int(9); // pii 指向的对象值为 9
string* ps = new string; // 初始化为空string
string* pss = new string(5, 's'); // *pss 为 "sssss"
int* p = new int[10]; // p指向一个可存储10个int的数组
```

有时我们在可能会在全局和类中提供自定义的 new-expression 替代函数。如果 new-expression 以可选的 `::` 开头，比如 `::new T` 或 `::new T[n]`，则类的替代函数会被忽略，该函数会在全局区域里寻找。否则，如果 T 是类类型，则从 T 的类作用域中开始寻找。

- placement new

如果有 placement_params，它们作为额外的参数传递给分配函数，这种分配函数叫做 "placement new"，被用来在已分配内存上构造对象。

```
char* ptr = new char[sizeof(T)];
T* tptr = new (ptr) T;
tptr->~T();
delete[] ptr;
```

如果无 initializer，则对象被默认初始化，如果有 initializer 则根据 type 和 initializer 使用值初始化、直接初始化、聚集初始化之一。

- delete-expression

```
::(optional)    delete    expression	(1)	
::(optional)    delete [] expression	(2)	
```

销毁一个由 new 或 new[] 创建的对象，如果 delete 作用在非 new 分配的对象上，则其行为是未定义的。

#### operator new/operator delete

- operator new/operator new[]

```
#include <new>

// replaceable allocation function
void* operator new  ( std::size_t count );
void* operator new[]( std::size_t count );
...
// class-specific allocation function
void* T::operator new  ( std::size_t count );
void* T::operator new[]( std::size_t count );
...
```

- operator delete/operator delete[]

```
#include <new>

// replaceable usual deallocation functions
void operator delete  ( void* ptr );
void operator delete[]( void* ptr );
...

// class-specific usual deallocation functions
void T::operator delete  ( void* ptr );
void T::operator delete[]( void* ptr );
...
```
分配和释放 count 字节大小内存。这些分配函数用 new-expression 来进行调用分配内存并初始化对象，用 delete-expression 来进行释放。 


replaceable allocation function 是用户提供的非成员函数。

```
#include <iostream>
#include <cstdlib>

void* operator new(size_t sz)
{
    std::cout << "global operator new called, size = " << sz << "\n";
    return std::malloc(sz);
}

void operator delete(void* ptr)
{
    std::cout << "global operator delete called.\n";
    std::free(ptr);
}

int main()
{
    int* p1 = new int;
    delete p1;

    int* p2 = new int[10];
    delete[] p2;
}
```
输出：
```
global operator new called, size = 4
global operator delete called.
global operator new called, size = 40
global operator delete called.
```

class-specific allocation function 是可以在类中定义的分配函数，如果使用 `::` 运算符就在全局空间中去找。static 是可选的关键字，无论有没有，分配函数总是 static 的。

```
#include <iostream>
#include <cstdlib>

void* operator new(size_t sz)
{
    std::cout << "global op new called: " << sz << "\n";
    return std::malloc(sz);
}

void operator delete(void* ptr)
{
	std::cout << "global op delete called\n";
	std::free(ptr);
}

struct Class {
    static void* operator new(size_t sz)
    {
        std::cout << "custom new1 for size: " << sz << "\n";
        return ::operator new(sz);
    }
    static void* operator new[](std::size_t sz)
    {
        std::cout << "custom new2 for size: " << sz << "\n";
        return operator new(sz);
    }
    static void operator delete(void* ptr)
    {
    	std::cout << "cunstom delete1\n";
    	::operator delete(ptr);
    }
    static void operator delete[](void* ptr)
    {
    	std::cout << "cunstom delete2\n";
    	operator delete(ptr);
    }
};

int main()
{
    Class* p1 = new Class;
    delete p1;
    Class* p2 = new Class[10];
    delete[] p2;
}
```

输出：

```
custom new1 for size: 1
global op new called: 1
cunstom delete1
global op delete called
custom new2 for size: 10
custom new1 for size: 10
global op new called: 10
cunstom delete2
cunstom delete1
global op delete called
```

#### std::allocator

使用 new 分配内存的同时也会在该内存上构造对象，但是有时将内存分配和构造对象组合在一起可能导致不必要的浪费，比如我们使用 `new string[n]`分配并初始化了 n 个 string 对象，但是可能用到少数，而且每个使用到的 string 元素都被赋值了两次。

`std::allocator` 类模板是用于标准库容器的默认内存分配器，可以用它来分配一块原始的、未构造的内存。

```
#include <memory>

template <class T>
struct allocator;
```

用 allocate 分配未构造的内存：

```
std::allocator<int> alloc;
auto p = alloc.allocate(2); // 分配两个未初始化的 int
auto q = p;
alloc.construct(q++, 1); // 第一个 int 为 1
alloc.construct(q++, 2); // 第二个 int 为 2
cout << *p << " " << *(p+1) << endl; // 1 2
while (q != p)
	alloc.destroy(--q); // 销毁对象
alloc.deallocate(p, 10); // 释放内存
```

还可以使用 allocator 类的两个伴随算法在内存中创建对象。

```
#include <memory>

template< class InputIt, class ForwardIt >
ForwardIt uninitialized_copy( InputIt first, InputIt last, ForwardIt d_first );
template< class InputIt, class Size, class ForwardIt >
ForwardIt uninitialized_copy_n( InputIt first, Size count, ForwardIt d_first);

template< class ForwardIt, class T >
void uninitialized_fill( ForwardIt first, ForwardIt last, const T& value );
template< class ForwardIt, class Size, class T >
ForwardIt uninitialized_fill_n( ForwardIt first, Size count, const T& value );
```

使用 uninitialized_copy 从容器里拷贝元素在内存里创建对象，使用 uninitialized_fill 在内存某范围内创建对象 value。

```
std::allocator<int> alloc;
auto p = alloc.allocate(5);
auto q = p;
vector<int> vi = {1, 2};
uninitialized_copy(vi.begin(), vi.end(), q);
std::uninitialized_fill(q+2, q+5, 10);
for (auto i = 0; i < 5; ++i)
	cout << *q++ << " "; // 1 2 10 10 10
while (q != p)
	alloc.destroy(--q);
alloc.deallocate(p, 5); 
```