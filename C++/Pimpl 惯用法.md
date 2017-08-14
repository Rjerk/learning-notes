# C++: Pimpl 惯用法

Pimpl 是 "pointer to implementation" 的缩写，该技巧可以避免在头文件里暴露私有细节。

它利用了C++的一个特点，即可以将类的数据成员定义为指向某个已经声明过的类型的指针，这里的类型仅仅作为名字引入，并没有完整地定义，因此我们就可以将该类型的定义隐藏在.cpp 文件中。这通常称为不透明指针，因为用户无法看到它所指向的对象细节。

本质上，Pimpl 是一种同时在逻辑上和物理上隐藏私有数据成员和函数的办法。

## 使用 Pimpl

这个 API 暴露了定时器在不同平台上存储的底层细节，任何人都可以从头文件中看到这些平台定义。

```
#include <string>
#include <iostream>

#ifdef _WIN32
#include <windows.h>
#else
#include <sys/time.h>
#endif 

class AutoTimer {
public:
	explicit AutoTimer(const std::string& name);
	~AutoTimer();
private:
	double getElapsed() const;
	std::string m_name;
#ifdef _WIN32
	DWORD m_start_time;
#else
	struct timeval m_start_time;
#endif
};
```

设计者的真正目的是将所有私有成员隐藏在 .cpp 文件中，这样就无需包含任何繁琐的平台相关项了。

Pimpl 惯用法将所有私有成员放在一个类中，这个类在头文件前置声明，在 .cpp 文件中定义。

重构后的 autotimer.h:

```
#include <string>
#include <memory>

class AutoTimer {
public:
	explicit AutoTimer(const std::string& name);
	~AutoTimer();
private:
	class Impl;
	Impl* m_impl; // 使用智能指针 
};
```

AutoTimer::Impl 类的定义，包含了暴露在原有头文件中的所有私有方法和变量。

autotimer.cpp

```
#include "autotimer.h"
#include <iostream>

#ifdef _WIN32
#include <windows.h>
#else
#include <sys/time.h>
#endif

class AutoTimer::Impl {
public:
	double getElapsed() const
	{
#ifdef _WIN32
		return (GetTickCount() - m_start_time) / 1e3;
#else
		struct timeval end_time;
		gettimeofday(&end_time, NULL);
		double t1 = m_start_time.tv_usec / 1e6 + m_start_time.tv_sec;
		double t2 = end_time.tv_usec / 1e6 + end_time.tv_sec;
		return t2 - t1; 
#endif
	}
	
	std::string m_name;
#ifdef _WIN32
	DWORD m_start_time;
#else
	struct timeval m_start_time;
#endif
};

AutoTimer::AutoTimer(const std::string& name):
	m_impl(new AutoTimer::Impl())
{
	m_impl->m_name = name;
#ifdef _WIN32
	m_impl->m_start_time = GetTickCount();
#else
	gettimeofday(&m_impl->m_start_time, NULL);
#endif
}

AutoTimer::~AutoTimer()
{
	std::cout << m_impl->m_name << ": took " << m_impl->getElapsed() << " secs\n";
	delete m_impl;
	m_impl = nullptr;
}
```

完整可运行代码在：[这里](https://github.com/Rjerk/snippets/tree/master/cpp-pimpl/new_delete_pimpl)

Impl 类声明为 AutoTimer 类的私有内嵌类，将其声明为内嵌类是为了避免与该实现相关的符号污染全局命名空间，而将其声明为私有的表示它不会污染类的公有 API。通常只有在 .cpp 文件中其他类或自由函数必须访问 Impl 成员时，才采用共有内嵌类（或共有非内嵌类）。

将所有私有成员和私有方法防止在 Impl 类中，可以保持数据和操作这些数据的方法的封装性，从而避免在公有头文件中声明私有方法。两个注意事项：
- 不能在实现类中隐藏私有虚方法。它们必须出现在公有类中，以保证任何派生类都能够覆盖它们。
- 虽然可以将共有类传递给需要使用它的实现类的方法，但必要时可以再实现类中增加指回公有类的指针，以便 Impl 调用公有方法。

## 拷贝语义

如果没有为类定义拷贝构造函数和拷贝赋值操作符，C++ 编译器会默认创建，但这种默认的构造函数只能执行对象的浅拷贝。如果客户拷贝了使用 Pimpl 类的对象，则两个对象会同时指向同一个 Impl 实现对象，在析构时可能会尝试删除同一个 Impl 对象，导致崩溃。

两个解决方案：

1. 禁止拷贝类。如果不打算创建对象的副本，就可以将其声明为不可拷贝的。可以通过显式声明拷贝构造函数和拷贝赋值操作符达到这一目的，不需要提供它们的实现，仅声明就能防止编译器为其生成默认版本。C++11 提供了 = delete 可以将其声明为删除的函数，使它不能通过任何方式被使用。

```
#include <string>
#include <memory>

class AutoTimer {
public:
	explicit AutoTimer(const std::string& name);
	~AutoTimer();
    AutoTimer(const AutoTimer&) = delete;
    const AutoTimer& operator=(const AutoTimer&) = delete;
private:
	class Impl;
	Impl* m_impl; // 使用智能指针 
};
```

2. 显示定义拷贝语义。如果希望用户能够拷贝 Pimpl 的对象，就应该声明并定义自己的拷贝构造函数和拷贝赋值运算符。它们可以执行对象的深拷贝，即创建对象的副本，而非仅拷贝指针。

### Pimpl 与智能指针

使用 Pimpl 可能忘记在析构函数中删除对象，或者在分配前，销毁后对其访问，从而引发错误。

C++11 增加了 std::unique_ptr，可以在构造函数中动态分配一个 AutoTimer::Impl 对象，在析构时也析构 AutoTimer::Impl 对象。由于 unique_ptr 是不可复制的，如果使用这种智能指针声明不希望用户拷贝的对象，就无需处理拷贝构造函数和拷贝赋值运算符。

autotimer.h

```
#include <string>
#include <memory>

class AutoTimer {
public:
	explicit AutoTimer(const std::string& name);
	~AutoTimer();
private:
	class Impl;
	std::unique_ptr<Impl> m_impl;
};
```

autotimer.cpp

```
#include "autotimer.h"
#include <iostream>

#ifdef _WIN32
#include <windows.h>
#else
#include <sys/time.h>
#endif

class AutoTimer::Impl {
public:
	double getElapsed() const
	{
#ifdef _WIN32
		return (GetTickCount() - m_start_time) / 1e3;
#else
		struct timeval end_time;
		gettimeofday(&end_time, NULL);
		double t1 = m_start_time.tv_usec / 1e6 + m_start_time.tv_sec;
		double t2 = end_time.tv_usec / 1e6 + end_time.tv_sec;
		return t2 - t1; 
#endif
	}
	
	std::string m_name;
#ifdef _WIN32
	DWORD m_start_time;
#else
	struct timeval m_start_time;
#endif
};

// make_unique 是 C++14 的特性
AutoTimer::AutoTimer(const std::string& name):
	m_impl(std::make_unique<Impl>())
{
	m_impl->m_name = name;
#ifdef _WIN32
	m_impl->m_start_time = GetTickCount();
#else
	gettimeofday(&m_impl->m_start_time, NULL);
#endif
}

AutoTimer::~AutoTimer()
{
	std::cout << m_impl->m_name << ": took " << m_impl->getElapsed() << " secs\n";
}
```

完整可运行代码在：[这里](https://github.com/Rjerk/snippets/tree/master/cpp-pimpl/unique_ptr_pimpl)

## Pimpl 的优点

- 信息隐藏

私有成员完全隐藏在共有接口之外，使得实现细节得以隐藏。信息隐藏也意味着共有头文件能够更加感觉、更加清晰地表达真正的共有接口，因此更易于用户阅读和理解。信息隐藏带来的另一个好处是，用户不能轻易使用“脏”手段访问私有成员，比如：

```
#define private public // 使私有成员变为共有的
#include "your_api.h" // 现在可以访问私有成员
#undef private // 回到默认的私有语义
```

- 降低耦合

不用 Pimpl，共有头文件就必须包含所有私有成员变量所需的头文件，这就增加了 API 与系统其他部分在编译时的耦合度。使用 Pimpl 可以将那些依赖项转移到 .cpp 文件中去，并移除耦合的元素。

- 加速编译

将与实现相关的头文件移入 .cpp 文件中带来的另一个隐含的结果就是 API 的引用层次得以降低，这将直接影响编译时间。

- 更好的二进制兼容性

采用 Pimpl 的对象的大小从不改变，因为对象总是单个指针的大小。对私有成员变量做任何修改都只影响到隐藏在 .cpp 文件内部的实现类的大小。如果对实现做出重大改变，对象的二进制表示可以不变。

- 惰性分配

可以在需要时再构造。如果类需要分配有限或高成本的资源（如网络连接），惰性分配很有用。

## Pimpl 的缺点

- 引入性能冲击

Pimpl 主要缺点是必须为创建的每个对象分配并释放实现对象，这使得对象增加了一个指针，同时因为必须通过指针间接访问所有成员变量，这种额外的调用层次与 new 和 delete 类似，可能引入性能冲击。

- 代码难以阅读和调试

访问所有私有成员都需要增加形如 m_impl-> 的前缀，增加的抽象层次使得实现代码难以阅读和调试。当 Impl 类拥有指回公有类的指针时，情况会更加复杂。而且必须记得为类定义拷贝构造函数或禁止拷贝类。这使得开发人员承担更多负担。

- 编译器失去捕获 const 方法中对成员变量的修改

Pimpl 惯用法里，成员变量存在于独立的对象中，编译器检查 const 方法中的 m_impl 指针值是否发生变化，而不检查 m_impl 指针指向的成员值的变化。事实上，除了构造函数和析构函数，采用 Pimpl 的类中的每个成员函数都可以定义为 const。

下面的 const 方法改变了 Impl 对象中的变量：

```
void PimpledObject::constMethod() const
{
    m_imple->m_name = "string changed by a const method";
}
```