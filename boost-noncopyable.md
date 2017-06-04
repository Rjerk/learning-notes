# boost::noncopyable

boost::noncopyable 删除的 copy constructor 和 copy assignment，使类不能被拷贝或者赋值。

```
namespace boost
{
    class noncopyable;
}
```

Usage:

```
#include <boost/core/noncopyable>

class X : private boost::noncopyable {

};
```

在 C++11 中可自定义 class noncopyable，即将 noncopyale 的 copy constructor 和 copy assignment operator 设为 delete：

```
class noncopyable {
private:
    noncopyable() = default;
    ~noncopyable() = default;
protectedd:
    noncopyable(const noncopyable&) = delete;
    void operator=(const noncopyable&) = delete;
};
```