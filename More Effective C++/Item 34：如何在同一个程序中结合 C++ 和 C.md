# Item 34：如何在同一个程序中结合 C++ 和 C？

以C++ 组件搭配 C 组件一起发展软件，C++ 和 C 编译器必须产生兼容的目标文件。在这个大前提下，还有四个事情要考虑：

## Name Mangling

C++ 编译器通过 name mangling 可以为程序中的每一个函数编出独一无二的名称，使得 C++ 支持函数重载。 

比如在 C++ 封闭环境中，有一个名为 drawLine 的函数，可能会被编译器重整为 xyzzy。

如果 drawLine 在 C 函数库中，C++ 使用它的一个头文件，其内声明有：

```
void drawLine(int x1, int x2, int y1, int y2);
```

每次对 drawLine 的调用会被 C++ 编译器翻译为一个重整后的函数名称，比如调用 `drawLine(a, b, c, d);`，目标文件内含的是一个这样的函数调用码：`xyzzy(a, b, c, d);`。

如果 drawLine 是 C 函数，那么 “drawLine 编译代码” 所在的目标文件（或动态链接库）内将会有一个名为 drawLine 的函数，其名称并未重整。当企图将该目标文件链接进来时会得到一个错误信息，因为 C++ 编译器会企图寻找一个名为 xyzzy 的函数，而那并不存在。

因此要告诉 C++ 编译器让它不要重整某个函数名称，就得使用 `extern "C"` 指令，意味着这个函数有 C linkage，并对 C++ 函数名称的 name mangling 程序进行压抑。

```
extern "C"
void drawLine(int x1, int x2, int y1, int y2);
```

也可以对一群 C 函数使用：

```
extern "C" {
    void drawLine(int x1, int x2, int y1, int y2);
    void simulate(int iterations);
    void moveCar(int nums);
}
```

甚至可以将 C++ 函数声明为 extern "C"，将 C++ 语言编写的程序库空给其他语言的客户使用：

```
extern "C"
void randomGenerator(int seed); // C++ 函数，被设计用于 C++ 语言之外。
```

`extern "C"` 的运用可以简化 “必须同时被 C 和 C++ 使用” 的头文件的维护工作。当这个文件用于 C++ 时，你希望含有 `extern "C"`；用于 C 时，不希望如此。由于预处理器符号 __cplusplus 只针对 C++ 才有定义，所以这种 “通用于数种语言” 的头文件可以架构如下：

```
#ifdef __cplusplus
extern "C" {
#endif

void drawLine(int x1, int x2, int y1, int y2);
void simulate(int iterations);
void moveCar(int nums);

#ifdef __cplusplus
}
#endif
```

## Static Initialization

static class 对象、global 对象、namespace 内的对象以及文件范围（file scope）内的对象，其 constructors 总在 main 之前就获得执行，这个过程称为 static initialization。通过 static initialization 产生出来的对象，其 destructors 必须在所谓的 static destruction 过程中被调用，那个程序发生在 main 结束之后。

为解决这个问题，许多编译器在 main 函数开始处安插了一个函数调用，调用一个由编译器提供的提供函数，由它来完成 static initilization。编译器同样也会往往在 main 的最尾端安插一个函数调用完成 static 对象的析构。

编译后的 main 可能像这样：

```
int main()
{
    performStaticInitilization();

    the statement you put in main go here;

    performStaticDestruction();
}
```

如果一个 C++ 编译器采用这种方法来构造和析构 static 对象，那么除非程序中有 main，否则这种对象既不会被构造也不会被析构。由于此种做法十分普遍，所以应该尽可能尝试写 main。

貌似在 C 成分中写 main 较为合理 —— 如果程序主要以 C 完成而 C++ 只是个支持库的话，然而 C++ 程序库里含有 static 对象也是可能的，所以还是尽量在 C++ 中写 main 的好。但不必重写 C 代码，只需将 C main 重命名并在 C++ 的 main 函数调用它（当然最好放些注释说明为什么要这么做）。

```
extern "C"
int realMain(int argc, char* argv[]); // C main 的重命名

int main(int argc, char* argv[])
{
    return realMain(argc, argv);
}
```

如果无法在项目中写 main，则可能遭遇没有可移植的办法确保 static 对象的 constructors 和 destructors 会被调用，这时需要查看编译器文档或与厂商接洽。

## 动态内存分配

分配规则为：程序的 C++ 部分使用 new 和 delete，程序的 C 部分使用 malloc 和 free，new/delete 和 malloc/free 的调用要对应。

而且要避免调用标准程序库以外的函数或是大部分计算平台上未稳定的函数，比如非 C 和 C++ 标准的一分子却被广泛使用的 strdup 函数：

```
char* strdup(const char* ps); // 返回一个 ps 所指字符串的副本
```

由 strdup 分配的内存由 delete 还是 free 是否，视其实现而定，这会造成令人头疼的可移植性问题。

## 数据结构的兼容性

在 C++ 和 C 之间传递数据，其对话层次必须限制于 C 能够接受的范围。在 C++ 和 C 之间对数据结构做双向交流，应该是安全的 —— 前提是那些结构体的定义式在 C++ 和 C 中都可编译。例如在 struct 中加上虚函数，或者令其继承另一个 struct，就会对其对象采用不同于 C 的内存布局，无法和 C 进行交换。

## 总结

如果要在同一个程序中混用 C++ 和 C，有几个简单的守则：

- 确定 C++ 和 C 编译器产出兼容的目标文件
- 将双方使用的函数声明为 `extern "C"`
- 尽可能在 C++ 中撰写 main
- 总是以 delete 删除 new 返回的内存；总是以 free 释放 malloc 返回的内存
- 将两个语言间的 “数据结构传递” 限制于 C 所能了解的形式；C++ structs 如果内含非虚函数，则不受此限制。
