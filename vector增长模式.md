# vector是如何增长的

vector将元素保存在连续的内存空间中，因为元素是连续存储的，和内置数组所使用的机制一样，所以以下标计算其地址非常快速，元素也能快速访问。但是，添加或删除元素却会非常耗时。插入或删除元素时，需要将插入或删除位置之后的元素移动以保持连续存储。并且有时因为空间不够还需要分配额外的存储空间，于是每个元素必须移动到新的存储空间，然后释放旧的存储空间。如果频繁地进行内存分配和释放，性能则会极差。

标准库实现了一种可以简单高效的策略安排vector的增长，即每次分配新的存储空间时，会分配比需求更大的内存空间以作备用，来保存更多的新元素，而且无需频繁地分配新的空间。

## capacity 和 size

每个vector有两个大小，一个是容器内的元素所占的空间大小，即size；另一个是为在不分配新的内存空间的前提下最多能保存多少元素，即capacity. capacity至少与size一样大，具体分配多少取决于标准库的具体实现。

如果有v是一个vector，v.size()和v.capacity()可以得到v的size和capacity.

你可以将一个vector想象成这样：
![](http://i1.piimg.com/567571/cfa1466af1a28be5.png)

每次v.push_back(i)时，元素i被存储在预留空间，这样就无需重新分配新的内存空间。

## 重新分配内存空间

只有在执行insert操作时size与capacity相等，或者调用resize或reserve时给定的大小超过当前的capacity时，vector才可能重新分配内存空间。

### 重新分配内存空间的代价

重新分配内存涉及到四步：
1. 分配足够的内存空间给新的capacity
2. 将元素从旧的内存空间拷贝到新的内存空间
3. 销毁旧的内存空间中的元素
4. 释放旧的内存空间

如果vector有n个元素，除非1、4的代价超过2、3，则重新分配内存空间要O(n)的时间。这里涉及到tradeoff，即分配大量额外的内存空间使得我们以后一段时间内不会重新再次分配，这个策略的代价可能会造成大量空间的浪费。如果只分配少量的额外内存，可能会花费大量的时间在重新分配额外的内存空间上。

### 重新分配的策略

假设我们每次填满vector都使得它的capacity加1，即尽可能少的分配额外的空间，每次push_back元素都会引起重新分配，每次花费O(n)的时间，则增长到k个元素时用掉的时间为O(1+2+...k)即O(k^2)，这样无疑是很糟糕的策略。

又假设我们每次填满vector使得capacity加上常量C，会不会变得好点？假若共有k*C个元素，每次新填充C个元素发生重新分配，用时O(C+2C+3C+...kC)即O(k^2)，看来还是没变得好点。

假若capacity每次翻倍呢，会好点吗？假如最后有n个元素已经填入，差一点就要重新分配了。

![](http://i1.piimg.com/567571/35bcb5f455adfe75.png)

如图，总共的重新分配次数为n/2 + n/4 + ... + 1，约等于n，即用时O(n)。即使元素再多点，再重新分配一次，时间还是O(n)，可以看出这个策略已经好太多了。

### 在本机标准库的capacity的内存空间的重新分配

```C++
#include <iostream>
#include <vector>

int main()
{
	std::vector<int> vi;
	for (int i = 0; i < 40; ++i) {
		vi.push_back(i);
		std::cout << "vi.size() = " << vi.size() 
			  << ", vi.capacity()= " << vi.capacity() << std::endl;
	}
	return 0;
}
```
输出：
> vi.size() = 1, vi.capacity()= 1
> vi.size() = 2, vi.capacity()= 2
> vi.size() = 3, vi.capacity()= 4
> vi.size() = 4, vi.capacity()= 4
> vi.size() = 5, vi.capacity()= 8
> vi.size() = 6, vi.capacity()= 8
> vi.size() = 7, vi.capacity()= 8
> vi.size() = 8, vi.capacity()= 8
> vi.size() = 9, vi.capacity()= 16
> vi.size() = 10, vi.capacity()= 16
> vi.size() = 11, vi.capacity()= 16
> vi.size() = 12, vi.capacity()= 16
> vi.size() = 13, vi.capacity()= 16
> vi.size() = 14, vi.capacity()= 16
> vi.size() = 15, vi.capacity()= 16
> vi.size() = 16, vi.capacity()= 16
> vi.size() = 17, vi.capacity()= 32
> vi.size() = 18, vi.capacity()= 32
> vi.size() = 19, vi.capacity()= 32
> vi.size() = 20, vi.capacity()= 32
> vi.size() = 21, vi.capacity()= 32
> vi.size() = 22, vi.capacity()= 32
> vi.size() = 23, vi.capacity()= 32
> vi.size() = 24, vi.capacity()= 32
> vi.size() = 25, vi.capacity()= 32
> vi.size() = 26, vi.capacity()= 32
> vi.size() = 27, vi.capacity()= 32
> vi.size() = 28, vi.capacity()= 32
> vi.size() = 29, vi.capacity()= 32
> vi.size() = 30, vi.capacity()= 32
> vi.size() = 31, vi.capacity()= 32
> vi.size() = 32, vi.capacity()= 32
> vi.size() = 33, vi.capacity()= 64
> vi.size() = 34, vi.capacity()= 64
> vi.size() = 35, vi.capacity()= 64
> vi.size() = 36, vi.capacity()= 64
> vi.size() = 37, vi.capacity()= 64
> vi.size() = 38, vi.capacity()= 64
> vi.size() = 39, vi.capacity()= 64
> vi.size() = 40, vi.capacity()= 64

可以看出，在本机标准库中，采用的策略是在每次需要分配新的内存空间时将capacity设置为size的两倍。

当然，所有实现都应遵循的原则是：确保用push_back向vector添加元素的操作有高效率，通过n次调用push_back添加n个元素的vector，所花费的时间不能超过n的常数倍。
