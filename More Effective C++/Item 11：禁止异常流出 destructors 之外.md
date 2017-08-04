# Item 11：禁止异常流出 destructors 之外

两个理由

## 避免 terminate 函数在异常传播过程的栈展开机制被调用

如果控制权基于异常的因素离开 destructor，而此时正有另一个异常处于作用状态，C++ 会立即调用 terminate 函数终止程序，甚至不等局部对象销毁。

```
class Session {
public:
    Session();
    ~Session();
    ...
private:
    static void logCreation(Session* objArr);
    static void logDestruction(Session* objArr); 
};

Session::Session()
{
    logDestruction(this);
}
```

如果 logDestruction(this) 抛出一个异常，这个异常并不会被 Session destructor 捕捉，所以它会传播到 destructor 的调用端。但如果这个 destructor 本身是因为某个异常而被调用，terminate 函数就会被调用，于是程序就会被终结。

为阻止异常从 destructor 抛出，可以使用 try/catch

```
Session::~Session()
{
    try {
        logDestruction(this);
    } catch (...) {

    }
```

`catch(...)` 捕获所有异常，并且看起来什么都没做，但是它阻止了 losDestruction 所抛出的异常传出 destructor 之外，terminate 不会被调用。

## 协助确保 destructors 完成其应该完成的任务

```
Session::Session()
{
    logCreation(this);
    startTransaction();
}

Session::~Session()
{
    logDestruction(this);
    endTransaction();
}
```

如果 logDestruction() 抛出一个异常，于 startTransaction() 内开始的那个事务绝不会结束。

即使我们可以重新安排 destructor 内函数调用顺序，但如果 endTransaction() 抛出异常，logDestruction(this) 的任务也不会完成。那么除了投入 try/catch 的怀抱别无他法。

