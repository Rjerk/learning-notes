# Item 12：了解 “抛出一个异常” 与 “传递一个参数” 或 “调用一个虚函数” 之间的差异

```
class Widget { .. }

void f1(Widget w);
void f2(Widget& w);
void f3(const Widget& w);
void f4(Widget* w);
```

## 控制权

如果传递的是参数，控制权会回到调用端（除非函数失败以致无法返回）；如果抛出一个异常，控制权则不会回到调用端。

## 复制行为

一个对象被抛出作为异常时，总会发生复制。而传参则不是。

```
istream operator>>(istream& s, Widget& w);
void passAndThrowWidget()
{
    Widget localWidget;
    cin >> localWidget;
    throw localWidget;
}
```

当 localWidget 被交给 `operator >>` 时，并未发生复制行为，而是 `operator >>` 内的 reference w 被绑定到 localWidget 上，对 w 所做任何事情，都会施加到 localWidget 上。
    
这和将 localWidget 当做一个异常不同，不论异常被当做 by value 还是 by reference 传递，总是会发生复制行为，而交给 catch 子句手中的正是那个副本。
    
因此这一差异也解释了另一个事实：“抛出异常” 常常比 “传递参数” 慢。

当对象被复制当做一个异常，其复制行为是由对象的 copy constructor 完成的，这个 copy constructor 相应于该对象的 “静态类型” 而非 “动态类型”。

```
class Widget { ... };
class SpecialWidget : public Widget { ... };
void passAndThrowWidget()
{
    SpecialWidget local_special_widget;
    ...
    Widget& rw = local_special_widget;
    throw rw;
}
```

这里抛出的是一个 Widget 异常，虽然 rw 代表的是一个 SpecialWidget，但是编译器所关心的是 rw 的静态类型：Widget。复制动作永远是以对象的静态类型为本。

```
catch (Widget& w) {
    ...
    throw;
}

catch (Widget& w) {
    ...
    throw w;
}
```

上面两个 catch 语句块之间的差异是，前者重新抛出当前的异常，不论其类型如何，如果最初抛出的异常的类型是 SpecialWidget，第一语句块会传播一个 SpecialWidget 异常，即使 w 静态类型是 Widget，因为此异常被重新抛出时并没有发生复制行为。后者的 catch 语句块则重新抛出新的异常，其类型总是 Widget，w 的静态类型。因此必须使用 `throw ;` 才能重新抛出当前的异常，其间不会有改变被传播异常类型的机会，而且更有效率。

一个被抛出的对象可以简单地用 by reference 方式捕捉，不需要以 by reference-to-const 的方式捕捉。函数调用将一个临时对象传递给一个 non-const reference 参数是不允许的，但对异常则是合法的。

```
// 捕获类型为 Widget，Widget& 或 Widget* 的异常
catch (Widget w) ... 
catch (Widget& w) ...
catch (const Widget& w) ...
catch (Widget* w) ...
```

如果以 by value 方式传递异常，`catch (Widget w) ...` 会付出被抛出物的两个副本的构造代价。而以 by reference 方式捕捉一个异常，`catch (Widget& w) ...` 和 `catch (const Widget& w) ...` 则付出被抛出物的单一副本的构造代价。

## 类型转换

参数传递允许隐式类型转换。而异常则不。

```
void f(int value)
{
    try {
        throw value;
    } catch (double d) {
        ...
    }
}
```

try 语句块里抛出的 int 异常绝不会被用来捕捉 double 异常的 catch 子句捕捉到，后者只能捕获 double 异常，int 异常只能被 int 或 int& 或加上 const 或 volatile 的子句所捕获到。

异常与 catch 子句相匹配的过程中，仅有两种转换可以发生。

第一种是 “继承架构中的类转换”，针对 base class 异常而编写的 catch 子句，可以处理类型为 derived class 的异常。

第二种允许的转换是从一个 “有型指针” 转为 “无型指针”，所以一个针对 `const void*` 而设计的 catch 子句，可以捕获任何指针类型的异常。

## 匹配策略

catch 子句总是依出现顺序做匹配尝试，即最先吻合，而在虚函数中使用最佳吻合策略。
