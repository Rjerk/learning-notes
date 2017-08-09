# 利用 SFINAE 解决函数模板的重载决议问题

今天实现 `std::vector` 时碰上一个重载决议的问题。

有两个 `insert` 函数：

```
template <typename T, typename Allocator>
typename Vector<T, Allocator>::iterator
Vector<T, Allocator>::insert(const_iterator pos, size_type count, const T& value)
{
    checkIterator(pos);
    auto p = const_cast<iterator>(pos);
    if (count == 0) {
        return p;
    }
    for (size_type i = 0; i < count; ++i) {
        p = insert(p, value);
    }
    return p;
}

template <typename T, typename Allocator>
template <typename InputIt>
typename Vector<T, Allocator>::iterator
Vector<T, Allocator>::insert(const_iterator pos, InputIt first, InputIt last)
{
    checkIterator(pos);
    auto p = const_cast<iterator>(pos);
    if (first == last) {
        return p;
    }
    for (auto iter = first; iter != last; ++iter) {
        p = insert(p, *iter);
        ++p;
    }
    return p - (last-first);
}
```

当我调用第一个 `insert` 时，编译器却给我匹配了第二个：

```
Vector<int> vi = {1, 2, 3};
vi.insert(vi.cend(), 4, 5); // get compile error, using insert(const_iterator pos, InputIt first, InputIt last).
```

了解 `SFINAE` 规则后对解决这个问题有帮助。

## 模板实参推断

当一个函数模板进行特化时，所有模板参数都会对应一个类型。这些类型可以被显示指定或从默认模板实参推导出来。

当你调用一个函数模板时，编译器会对每个模板参数进行推导对应类型：

```
template <T>
T doNothing(T t)
{
    return t;
}

int i = doNothing(1); // T 被推导为 int
double f = doNothing(1.0f); // T 被推导为 float
double f = doNothing<long>(1u); // T 被推导为 long
```

## 模板形参替换

函数模板形参被实参替换两次：
1. 显示指定的模板形参于模板实参推导前替换
2. 推导的实参和从默认项获得的实参于模板实参推导后替换

```
template <typename T>
T doNothing(T t)
{
    return t;
}

int i = doNothing(1); // int doNothing(int);
```

## 类型推导失败

如果一个替换得到一个无效的类型或表达式，类型推导失败。

```
template <typename T>
typename T::iterator begin(T& t)
{
    return t.begin();
}

int i = 0;
int* iter = begin(i); // 替换失败
```

用 `int` 替换 `T` 得到一个无效的类型 `int::iterator`，所以类型推导失败。

## Substitution Failure Is Not An Error

替换 `Substitution` 发生在重载决议（overload resolution）之前，所以编译器知道在重载集合中哪个函数模板特化被选中。

当类型推导失败，说明替换产生了一个无效类型，这意味着当前的一个函数模板特化不在重载集合（overload set）中。

替换失败不是错误，是因为可能在重载集合中还有其他的函数模板特化可以使用。

```
template <typename T>
typename T::iterator begin(T& t)
{
    return t.begin();
}

template <typename T, size_t N>
T* begin(T (&array)[N])
{
    return array;
}

int a[2];
int* b = begin(a);
```

在上述调用中，实例化两次，第一个 `begin(T& t)` 替换失败，第二个 `begin(T (&array)[N])` 替换成功，编译器丢弃第一个特化而不是报错，然后选择第二个进行匹配，编译正确。

通过 `SFINAE` 规则，我们使得模板匹配更为精确。

## 使用 `std::enable_if` 控制重载集合

假如有三种编码类型: `int`，`enum` 和 `float`，并用 `encode` 进行统一调用：

```
template <typename T>
void encode(std::ostream& stream, T const& int_value)
{
    stream << "int: " << int_value << "\n";
}

template <typename T>
void encode(std::ostream& stream, T const& enum_value)
{
    stream << "enum: " << static_cast<typename std::underlying_type<T>::type>(enum_value) << '\n';
}

template <typename T>
void encode(std::ostream& stream, T const& float_value)
{
    stream << "float: " << float_value << '\n';
}

int main()
{
    encode(std::cout, 1);
	
    enum class color { red, green, blue };
    encode(std::cout, color::blue);
	
    encode(std::cout, 3.0);
}
```

我们会得到编译错误，因为这三个函数签名其实是一样的：`void encode(std::ostream&, T const&)`。

这时候可以用 `std::enable_if` 来控制重载集合。

```
template <typename T, typename std::enable_if<std::is_integral<T>::value, T>::type* = nullptr>
void encode(std::ostream& stream, T const& int_value)
{
    stream << "int: " << int_value << '\n';
}

template <typename T, typename std::enable_if<std::is_enum<T>::value>::type* = nullptr>
void encode(std::ostream& stream, T const& enum_value)
{
    stream << "enum: " << static_cast<typename std::underlying_type<T>::type>(enum_value) << '\n';
}

template <typename T, typename std::enable_if<std::is_floating_point<T>::value>::type* = nullptr>
void encode(std::ostream& stream, T const& float_value)
{
    stream << "float: " << float_value << '\n';
}
```

`std::is_integral<T>::value` 返回一个 `bool` 类型的编译器常数，如果 `T` 是一个 `integral`，则 `value` 为 `true`，否则为 `false`。

`std::enable_if` 的声明为：

```
template< bool B, class T = void >
struct enable_if;
```

当 `B` 为 `true`，`std::enable_if` 就有一个成员类型 `type`，它等于 `T`；如果 `B` 为 `false`，就没有成员类型 `type`，那么 `Substitution` 就会失败，对应的特化就不会被添加到重载集合中参与重载决议。

## 解决问题

利用输入迭代器的 `input_iterator_tag` 来限制重载集合：

```
template <typename InputIterator>
using RequireInputIterator =
    typename std::enable_if<std::is_convertible<typename std::iterator_traits<InputIterator>::iterator_category,
	                                        std::input_iterator_tag>::value
                           >::type;

template <typename T, typename Allocator>
template <typename InputIt, typename = RequireInputIterator<InputIt>>
typename Vector<T, Allocator>::iterator
Vector<T, Allocator>::insert(const_iterator pos, InputIt first, InputIt last);
```

这样，调用 `vi.insert(vi.cend(), 4, 5)` 时会检查到第二个和第三个参数没有 `input_iterator_tag`，根据 `SFINAE` 规则丢弃 `insert(const_iterator pos, InputIt first, InputIt last)` 的特化版本。