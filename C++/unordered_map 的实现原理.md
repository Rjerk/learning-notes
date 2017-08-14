# unordered_map 的实现原理

```
template<
    class Key,
    class T,
    class Hash = std::hash<Key>,
    class KeyEqual = std::equal_to<Key>,
    class Allocator = std::allocator< std::pair<const Key, T> >
> class unordered_map;
```

unordered map 是一个包含键值对的关联容器，而且每个键是独一无二的。

## 内部布局

对于 unordered container 来说，它们都以 hash table 作为基础，这些 hash table 使用 chaining 做法，于是一个 hash code 被关联到一个 linked list （此技法被称为 [open hashing](https://stackoverflow.com/questions/9124331/meaning-of-open-hashing-and-closed-hashing) 或 closed addressing）。至于 linked list 是单链或双链，则取决于实现。

unordered_map 的典型内部布局：

```
                bucket
                  __     __
                 |__|-> |__|
                 |__|    __
                 |__|-> |__|
                 |__|
-> hashfunc() -> |__|    __      __
                 |__|-> |__| -> |__|
                 |__|
                 |__|
```

对于每个存放的元素（key/value），hash function 把 key 映射到 hash table 中的每个 bucket 中，每个 bucket 管理一个单向 linked list，内含所有 会使 hash function 产生相同数值的元素。 

内部使用 hash table 可以使得插入、删除、查找元素时获得的摊提（amortized）常量时间（偶尔发生的 rehashing 可能是个大型操作，带着线性复杂度）。

## rehashing

load factor （负载系数）被定义为 load factor = n / k，n 是 hash table 所能容纳的元素个数，k 是 bucket 的数量。

当负载系数变得太大，hash table 的操作会变慢甚至失败。当负载系数变得太小，可能会浪费内存空间。

为解决这个问题，使用 rehashing。当容器个数增大，容器内 n'/k 超过最大负载系数，或容器个数缩减，n'/k 小于最小负载系数，都会强制进行 rehashing。

关于 rehashing，有各种各样的实现策略：
- 传统做法是，在单一的 insert 或 earse 后，有时会出现一次内部数据重新组织。
- 递进式（incremental hashing）的做法是，渐进改变 bucket 或 slot 的数量，这对即时环境特别有用。

unordered 容器运行上述二种策略。

rehashing 只可能发生在这些调用之后：insert()，rehash()，reserve() 或 clear()。erase() 绝不会造成指向元素的 iterator、reference 和 pointer 失效。因此如果删除数百元素，bucket 的大小并不会改变。但如果在那之后插入了一个元素，bucket 的大小就有可能缩小。

unordered_map 提供了一些影响内部布局的操作函数。

可以指定一个最大负载系数，c.max_load_factor(val)，一旦超过 val 就会自动 rehashing。

c.rehash(bucket_num) 可以将容器 rehash，使其 bucket 个数至少为 bucket_num。以此接口，必须考虑最大负载系数，如果最大负载系数是 0.7 而我们打算准备 100 个元素，则必须将 100 除以 0.7，以此 143 传给 rehash() 以避免在最高达到 100 个元素的情况下出现进一步 rehashing。

c.reserve(num) 将容器 rehash，使其空间至少可拥有 num 个数。

c.load_factor() 返回当前的负载系数。

c.max_load_factor() 返回当前的最大负载系数。

## hash function

unordered 容器的几乎所有操作 —— 包括拷贝构造和赋值，元素的插入和查找，以及等价比较 —— 的预期行为，都取决于 hash function 的质量。如果 hash function 对不同的元素竟产生相同的数值，hash table 的任何操作都会导致执行效率的低下。

如果没有提供特殊的 hash function，默认的 hash function 是 hash<>，那是 <functional> 提供的一个 function object，可以对付常见类型：所有整型、浮点数类型、pointer、string 以及若干特殊类型。这些类型之外，我们必须提供自己的 hash function。

hash function 必须是一个函数，或者 function object，它接受一个元素类型下的 value 作为参数，返回一个类型为 std::size_t 的 value。将其返回值映射至合法的 bucket 索引范围内，则由容器内部完成。所以我们的目标是提供一个函数，将不同的元素值均匀映射到 [0, size_t] 内。

当提供我们自己的 hash function 时，它应该能提供良好的 hash value 分布，使得两个相等的 value 总是导致相同的 bucket，不同的 value 理应导致不同的 bucket 索引。

有时 hash function 并不总为每个独特的 key 提供独特的 hash value，要比较两个 key 是否精确匹配，使用 KeyEqual 进行 key 相等性的比较。我们可以在类中覆写 operator()，或者使用 std::equal 的特化版本，或者为 key 类型重载 operator==()。

假如 key 类型为：

```
struct Key {
    string first;
    string second;
    int third;
    bool operator==(const Key& other) const {
        return (first == other.first && second == other.second && third == other.third);
    }
};
```

可以自定义 hash function，在 `std` 名称空间内进行模板特化。

```
namespace std {
    template <>
    struct hash<Key> {
        std::size_t operator()(const Key& k) const {
            using std::hash;
            return ( (hash<string>()(k.first)) 
                   ^ ((hash<string>()(k.second) << 1) >> 1) 
                   ^ (hash<int>()(k.third) << 1) );
        }
    };
}
```

然后使用 `std::hash<Key>` 进行 hash value 的计算，使用 `operator==` 进行 Key 相等性检查。 

```
int main()
{
    std::unordered_map<Key, string> mp = {
        {{"John", "Doe", 12}, "first"},
        {{"Mary", "Sue", 11}, "second"}
    };
    Key John {"John", "Doe", 12};
    auto iter = mp.find(John);
    cout << iter->second << endl;
}
```

完整代码见：[unordered_map_hash_func.cpp](https://github.com/Rjerk/snippets/blob/master/c%2B%2B_example/unordered_map_hash_func.cpp)

也可以在类中定义调用运算符，然后将其作为模板参数传递给 unordered_map：

```
struct KeyHasher {
	std::size_t operator()(const Key& k) const {
		using std::hash;
        return ( (hash<string>()(k.first)) 
               ^ ((hash<string>()(k.second) << 1) >> 1) 
               ^ (hash<int>()(k.third) << 1) );
	}
};

int main()
{
    std::unordered_map<Key, string, KeyHasher> mp = {
        {{"John", "Doe", 12}, "first"},
        {{"Mary", "Sue", 11}, "second"}
    };
}
```

完整代码见：[unordered_map_hash_func_2.cpp](https://github.com/Rjerk/snippets/blob/master/c%2B%2B_example/unordered_map_hash_func_2.cpp)



为提供一个更好的 hash function，我们可以使用 boost 提供的 hash_value 和 hash_combine 函数模板。hash_value 提供类似 `std::hash` 的功能，hash_combine 可结合不同的类型的 hash value 为一个。（[boost/functional/hash.hpp Reference](http://www.boost.org/doc/libs/1_64_0/doc/html/hash/reference.html)）

```
#include <boost/functional/hash.hpp>

struct KeyHasher {
	std::size_t operator()(const Key& k) const {
		using boost::hash_value;
		using boost::hash_combine;
		size_t seed = 0;
		hash_combine(seed, hash_value(k.first));
		hash_combine(seed, hash_value(k.second));
		hash_combine(seed, hash_value(k.third));
		return seed;
	}
};
```

完整代码见：[unordered_map_hash_func_boost.cpp](https://github.com/Rjerk/snippets/blob/master/c%2B%2B_example/unordered_map_hash_func_boost.cpp)