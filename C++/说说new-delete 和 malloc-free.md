# 说说 new/delete 和 malloc/free

new/delete 是 C++ 的运算符，malloc/free 是 C/C++ 语言的标准库函数，都用于申请和释放动态内存。

## 区别

### new 从自由存储区(Free Store)获得内存，而 malloc 从堆(Heap)里获得内存

C++ 有几个不同的内存区域，用来存储对象及其他类型的值。自由存储区是 C++ 两个动态内存区之一，使用 new/delete 来分配/释放。

另外一个动态内存区是堆，使用 malloc/free 来分配/释放。

但是堆和自由存储区是不同的，在一个区域内分配的内存不能在另一个区域被安全回收，所以 new/delete 和 malloc/free 不能混着用，必须配对。

[C++不同的内存区域](http://www.gotw.ca/gotw/009.htm)

### new 可以初始化内存，而 malloc 不能

new 分配内存后，调用类型的构造函数可以初始化这块内存

```
int* pi = new int(10);
string* ps = new string(10, 's');
```

而 malloc 只分配内存，并不能对其内存进行初始化。

### new/delete 返回一个元素类型的指针，malloc/free 返回一个void*

new 返回数据类型的指针，而 malloc 返回 void*，还需要通过显式地转换才能得到目标类型的指针。

```
int* p = (int*) malloc (sizeof(int)*10);
```

### new 从不返回NULL（那样会抛出一个错误），malloc 失败时返回NULL

当内存耗尽或者发送错误时，如果malloc调用失败，会返回一个NULL指针。而new不会返回NULL，而是抛出一个异常来表示错误的出现。每次使用malloc还得检查malloc返回的指针，所以使用new相对来说更加方便。

### new 被Type-ID调用（编译器计算其大小），malloc 必须指定字节大小

new可以自动计算其所需大小，而malloc则需指定大小

new:
```
int* p = new int;
```
malloc:
```
int* p = (int*) malloc (sizeof(int));
```

### new有个显式操作数组的版本，malloc分配数组需要人工计算其空间大小

new指定对象的数量进行分配内存，因为编译器会根据对象的数量计算分配内存的大小。而malloc则是需要指定全部的字节来分配大小。

new/delete操作一个大小为10的int数组：
```
int* pia = new int[10];
delete [] pia; 
```
malloc分配一个大小为10的int数组：
```
int* pia = (int*) malloc (sizeof(int) * 10);
free(pia);
```

### new 重新分配内存很慢，malloc 内存重新分配大块内存很容易

通过 new 分配的内存，重新分配内存时会从原来的内存空间拷贝到新的内存空间，拷贝完成后再释放原来的内存空间，这是一个很慢的过程。

而 malloc 则是调用 realloc，扩展内存空间时会更快些，这也是 malloc 优于 new 的一个地方。

### new/delete 能被重载，malloc/free 不能被重载

new/delete是运算符，可以被重载。

```
void* operator new (std::size_t) throw(std::bad_alloc);
void* operator delete(void*) throw();
```

malloc/free是标准库里的函数，不能被重载。

### new/delete可以调用malloc/free，而malloc/free不可调用new/delete

```
void* A::operator new(std::size_t sz)
{
	void *p = malloc(sz)
	return p;
}

void A::operator delete(void* p)
{
	free(p);
}
```

### new/delete 被用来初始/销毁对象，而malloc/free不能

new/delete使用构造函数和析构函数来初始/销毁对象，而malloc/free不能。

## 为什么有了 malloc/free 还要 new/delete

对于非内部数据类型的对象而言，光用 malloc/free 无法满足动态对象的要求：对象在创建时需要自动调用构造函数，对象在销毁时要自动调用析构函数。由于 malloc/free 是库函数而不是运算符，不在编译器控制权限之内，不能把调用构造函数和析构函数的任务强加给它们。因此，C++ 语言需要一个能够完成动态内存分配的初始化工作的运算符 new，以及一个能够完成清理与释放内存工作的运算符 delete。

## 为什么有了 new/delete 却不淘汰 malloc/free

1. C++ 程序要经常调用 C 函数，而且有些 C++ 实现可能使用 malloc/free 来实现 new/delete，而 C 程序只能使用 malloc/free 来管理动态内存。
2. 在某些情况下，malloc/free 可以提供比 new/delete 更高的效率。

# 参考：
> http://stackoverflow.com/questions/240212/what-is-the-difference-between-new-delete-and-malloc-free
> http://zamanbakshifirst.blogspot.com/search/label/heap
