# Item 26：避免在 forwarding reference 上的重载

假如有个函数以名字作为参数，并记录当前时间，然后将名字添加进全局数据结构：

```
std::multiset<std::string> names;

void logAndAdd(const std::string& name)
{
    auto now = std::chrono::system_clock::now();
    log(now, "logAndAdd");
    names.emplace(name);
}
```

这段代码不怎么有效率，比如这些调用：

```
std::string petName("Darla");

logAndAdd(petName);

logAndAdd(std::string("Persephone"));

logAndAdd("Patty Dog");
```

第一个调用中 name 和变量 petName 绑定，name 然后被传递给 names.emplace，因为 name 是一个 lvalue，所以被拷贝进了 names，这是无可避免的拷贝。

第二个调用中，name 和一个 rvalue 绑定，但它本身是一个 lvalue，所以还是被拷贝进 names，但是它其实是可以被移动进去的。本来只需要一次移动操作，却花费了一次拷贝操作。

第三个调用中，name 还是和一个 rvalue 绑定，这个 rvalue 是一个从 "Patty Dog" 隐式创建而来的临时 std::string 变量。但这次和第二次不同，传递给 logAndAdd 的本就是一个字符串字面量，它可以被直接传入 emplace，而无需创建一个临时的 std::string。emplace 是本可以用这个字符串字面量来直接在 names 内部创建一个 std::string 对象的，连一次移动都不需要，却花费了一次拷贝操作。

于是可以使用 forwarding reference 重写 logAndAdd：

```
template <typename T>
void logAndAdd(T&& name)
{
    auto now = std::chrono::system_clock::now();
    log(now, "logAndAdd");
    name.emplace(std::forward<T>(name));
}


std::string petName("Darla");

logAndAdd(petName); // copy lvalue into multiset.

logAndAdd(std::string("Persephone")); // move rvalue instead of copying it.

logAndAdd("Patty Dog"); // create std::string in multiset 
                        // intead of copying a temporary std::string.
```

这样就得到最优效率。

有时我们使用索引而不是名字来使用 logAndAdd，我们需要为其添加一个重载版本：

```
std::string nameFromIdx(int idx); // return name corresponding to idx.

void logAndAdd(int idx)
{
    auto now = std::chrono::system_clock::now();
    log(now, "logAndAdd");
    names.emplace(nameFromIdx(idx));
}
```

以下调用是正确的：

```
std::string petName("Darla");

logAndAdd(petName);                   // as before, these
logAndAdd(std::string("Persephone")); // calls all invoke 
logAndAdd("Patty Dog");               // the T&& overload

logAndAdd(22); // calls int overload.
```

然而下面的调用会出错：

```
short nameIdx = 10;
logAndAdd(nameIdx); // error!
```

为什么出错？

有两个版本的 logAndAdd，第一个使用 forwarding reference 的可以推导 T 为 short，产生一个完整匹配。而以 int 为参数的重载版本的 logAndAdd 也能匹配 short 类型，但需要一次转换，所以根据重载决议规则，forwarding reference 的重载版本被调用。

所以 name 被绑定到一个 short 对象上，其后被转发到 emplace 函数的 name 上。然而 std::string 却没有一个以 short 为参数的构造函数，所以调用 logAndAdd 失败。

因为使用 forwarding reference 的模板函数几乎可以为所有参数的类型创建一个完全匹配，所以 将重载和 forwarding reference 进行结合是一个坏主意。

如果写一个 perfect-forwarding constructor 来解决这个问题就会掉坑里。假如有个 Person 类做相同的事情，取一个 string 或 index 来查找 string：

```
class Person {
public:
    template <typename T>
    explicit Persion(T&& n)
        : name(std::forwarding<T>(n)) {}

    explicit Persion(int idx)
        : name(nameFromIdx(idx)) {}
    ...

private:
    std::string name;
};
```

对于 logAndAdd 来说，传递一个不是 int 的整数类型（比如 size_t，short）将会调用 forwarding referece 构造函数而不是 int 构造函数，这会导致编译错误，正如上面已经解释过的。而且情况会更糟，因为适当条件下，C++ 会生成 copy constructor 和 move constructor：

```
class Person {
public:
    template <typename T>
    explicit Person(T&& n)
        : name(std::forward<T>(n)) {}

    explicit Persion(int idx);

    Person(const Person& rhs);

    Person(Person&& rhs);

    ...
};
```

如果有如下定义：

```
Person p("Nancy");

auto cloneOfP(p); // create new Person from p; this won't compile!
```

当我们试图从 p 中创建一个 Person，看起来会调用 copy constructor，但实际上调用的是 perfect-forwarding constructor，这个函数以一个 Person 对象来初始化 Person 的 std::string 数据成员，编译器会报错。

为什么调用的不是 copy constructor？是这样的，当 cloneOfP 以一个 non-const 左值 p 来进行初始化时，模板构造函数被实例化为以类型为 Person 的 non-const 左值为参数：

```
class Person {
public:
    explicit Person(Person& n)
        : name(std::forward<Person&>(n)) {}
    explicit Person(int idx);
    Person(const Person& rhs);
};
```

所以 p 其实可以被传递给 copy constructor 或实例化的模板函数，但是 copy constructor 需要在 p 上添加 const 来匹配器参数类型，而模板函数则不需要，所以编译器选择后者为更好的匹配函数。

如果改成这样就会不同：

```
const Person cp("Nancy");

auto cloneOfP(cp); // calls copy constructor.
```

而实例化后的模板函数：

```
class Person {
public:
    explicit Person(const Person& n);

    Person(const Person& rhs);
};
```

C++ 的一个重载决议规则是，当一个实例化的模板函数和非模板函数匹配的一样好，就选择匹配非模板函数。因此 copy constructor 被调用。

当加入继承后，perfect-forwarding constructor 和编译器生成的 copy constructor 会发生更多角逐，派生类的 copy operations 和 move operations 的常规实现会让人惊讶：

```
class SpecialPerson: public Person {
public:
    SpecialPerson(const SpecialPerson& rhs)
        : Person(rhs) { ... } // call base class forwarding ctor.
    SpecialPerson(SpecailPerson&& rhs)
        : Person(std::move(rhs)) { ... } // call base class forwarding ctor.
};
```

派生类的 copy constructor 和 move constructor 不会调用其基类的 copy constructor 和 move constructor，而是调用基类的 perfect-forwarding constructor。经过模板实例化和重载决议，最终会编译失败，因为 std::string 没有一个 SpecialPerson 的构造函数。

因此，最好尽可能避免在 forwarding reference 上使用重载。但是如果需要使用函数转发更多的参数类型该怎么办， 请看 Item 27。


