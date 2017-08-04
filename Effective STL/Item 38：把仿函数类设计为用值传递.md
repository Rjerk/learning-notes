# Item 38: 遵循按值传递的原则来设计函数子类

C/C++ 里函数指针是用值传递的。

在 STL 中，函数对象在函数之间来回传递的时候也是按值传递的。比如 for_each:

```
template< class InputIt, class UnaryFunction >
UnaryFunction for_each( InputIt first, InputIt last, UnaryFunction f );
```

它的参数传递一个函数对象，返回值也是一个函数对象。

因此，如果自己编写函数对象，要确保该对象按值传递后能正常工作，意味着两件事：
1. 函数对象必须尽可能小，否则复制的开销会非常昂贵。
2. 函数对象必须是单态的，不能是多态的，否则在复制时会产生 slicing 问题。

然而有时候无法保证这两条，所以要既允许函数对象很大/保持多态性，又可以与 STL 所采用的按值传递函数对象的习惯保持一致，可以用这样的方法：将所需的数据和虚函数从函数子类中分离开，放到一个新类中，然后在函数子类中包含一个指针，指向这个新类的对象，即 Pimpl Idiom.

如果有这样一个包含大量数据和使用多台的类：

```
template <typename T>
class BPFC: public unary_function<T, void> {
public:
    virtual void operator()(const T& val) const;
private:
    Widget x; // Widget has large data.
    int x;
};
```

修改为：

```
template <typename T>
class BPFCImpl: public unary_function<T, void> {
private:
    Widget w;
    int x;
    ...
    virtual ~BFPCImpl(); // 多态类需要虚析构函数
    virtual void operator()(const T& val) const;
    friend class BPFC<T>; // 允许访问 BPFC 内部数据
};

template <typename T>
class BPFC: public unary_function<T, void> {
public:
    void operator()(const T& val) const 
    {
        pImpl->operator()(val);
    }
    ...
private:
    BPFCImpl<T>* pImpl;
};
```

这样 BPFC 函数子类就变得小巧而且是单态的了。

需要注意的是，如果函数子类用到 Pimpl Idiom，它必须以合理的方式来支持拷贝操作，确保拷贝构造函数正确处理了所执行的 BPFCImpl 对象，函数对象总是按值传递的，我们可以使用 `std::shared_ptr`来保证这一点：

```
template <typename T>
class BPFC: public unary_function<T, void> {
public:
    void operator()(const T& val) const 
    {
        pImpl->operator()(val);
    }
    ...
private:
    std::shared_ptr<BPFCImpl<T>> pImpl;
};
```