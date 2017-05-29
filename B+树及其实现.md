# B+树及其实现

## B+树介绍

B+树是一种树数据结构，是为磁盘等存储设备设计的一种平衡搜索树，一般用于数据库系统中的文件索引和存储。

B+树采用平衡树结构，其树根到树叶的每条路径的长度相同。一般分为三部分：根结点，内部结点，叶结点。

典型的结点结构如图：

![](https://github.com/Rjerk/learning-notes/blob/master/img/bpt1.png?raw=true)

每个结点存放n-1个搜索码(K1...Kn-1)和n个指针(P1...Pn)，并且搜索码按序存放(K1<K2...<Kn-1)。

### 根结点

在根结点中，除非整棵树只有一个结点去索引一个记录，否则至少有两个指针被使用，所有指针指向下一层的结点。

### 内部结点

内部结点的所有指针指向树中结点。对于指针P1，它指向其内搜索码全部小于K1的结点；对于i = 2, 3...n-1，指针Pi指向其内搜索码小于Ki且大于Ki-1的结点；指针Pn指向其内搜索码大于Kn-1的结点。

内部结点至少包含⌈n/2⌉个指针，至多容纳n个指针。

### 叶结点

在叶节点中，指针Pi指向具有搜索码Ki的一条文件记录，而最后一个指针Pn指向下一个叶结点的指针。每个叶结点的搜索码至少包含⌈(n-1)/2⌉个，至多包含n-1个。

如果Li和Lj是两个叶结点，i < j，则Li所有搜索码的值都小于Lj中的搜索码。

如下图，第一个叶结点中搜索码1、2 小于下一个叶结点中搜索码3、4。

![](https://github.com/Rjerk/learning-notes/blob/master/img/bpt2.png?raw=true)

## B+树操作

假设树中不包含重复搜索码。

### 查询

如果树中存在指定值V，从根结点开始查找。

1. 检查当前结点最小的Ki使得V <= Ki。

   如果V < Ki，就在Pi指向的结点进行查找；

   如果V == Ki，就在Pi+1指向的结点进行查找。

   重复以上过程，直到查找到叶结点。

2. 在叶结点中，如果找到等于V的搜索码Ki，则找到Pi指向的记录，返回该叶结点及索引i。

   否则树中不存在V。

### 插入

1. 查找到新搜索码可插入的结点。

2. 如果该结点搜索码个数小于n-1个，则插入key/pointer到结点中，更新记录，结束。

3. 如果该结点已满，则分裂为两个结点，将搜索码分到新结点中，使得每个结点有一半或刚好超过一半的搜索码。

   如果该结点是叶结点，更新指向下一个结点的指针指向新结点，将新结点的最小搜索码插入到父结点中，父结点重复步骤2。

   如果该结点是内部结点，在分裂过程中除开最中间一个搜索码，其他搜索码如上进行派分。

如图，插入12：

![](https://github.com/Rjerk/learning-notes/blob/master/img/bpt3.png?raw=true)

插入12，包含[9、10、11]的叶结点要分裂成两个结点，[9、10]以及[11、12]，此时将新结点的最小值11插入到原结点的父节点中，其内包含[9、13、16]。

插入11需要进行分裂为两个结点，除开中间搜索码13，其他搜索码分裂为两个结点，[9、11]以及[16]，将13插入原结点父结点中。

由于其原结点是根节点，没有父结点，则创建一个新的结点作为根节点，其指针指向刚刚分裂的两个内部结点。

结果如下图所示：

![](https://github.com/Rjerk/learning-notes/blob/master/img/bpt4.png?raw=true)

### 删除

1. 查找到要删除的搜索码对应的叶结点。

2. 在叶结点中删除搜索码并更新记录。

   如果在内部结点包含此结点中，以叶结点中新的最小搜索码取代。

   **2.1** 如果叶结点包含的搜索码删除后少于⌈(n-1)/2⌉个，而其兄弟结点的搜索码数大于⌈(n-1)/2⌉，则借取兄弟结点：

   1) case1: 如果是当前结点的右兄弟包含大于⌈(n-1)/2⌉个结点，借此结点的最左边key/pointer。

   case2: 如果是当前结点的左兄弟包含大于⌈(n-1)/2⌉个结点，借此结点的最右边key/pointer。

   2) 在父结点中更新当前结点和兄弟结的分离搜索码。

   内部结点的借取：

   1) 将当前结点与其兄弟结点的在父结点中的分离搜索码移到当前结点中。

   2) 使得兄弟结点的最左边（最右边）孩子的指针成为当前结点的最右边（最左边）孩子的指针。

   3) 使得兄弟结点的最左边（最右边）搜索码成为当前结点和兄弟结点在父结点中的分离搜索码。

   **2.2** 如果兄弟结点数为⌈(n-1)/2⌉，无法借取，就合并兄弟叶结点：

   1) 将当前叶结点的所有key/pointer移动到兄弟结点中。

   2) 从父结点中删除指向当前叶结点的指针。

   3) 在父结点中删除此叶结点与其兄弟结点的分离搜索码。

   内部结点的合并：

   1) 将当前结点与其兄弟结点在父结点中的分离搜索码移到兄弟结点中。

   2) 将当前结点的key/pointer移到兄弟结点中。

   3) 将父结点中指向当前结点的key/pointer删除。

   如果在合并过程中根节点为空，则使根节点唯一孩子成为新的根节点。


如图，删除13、15、1。

![](https://github.com/Rjerk/learning-notes/blob/master/img/bpt4.png?raw=true)

删除13：

![](https://github.com/Rjerk/learning-notes/blob/master/img/bpt5.png?raw=true)

删除13后，其内结点只有[15], 前面的兄弟结点无法分发，需在其后面的兄弟结点移动16进来，此时两个结点为[15, 16], [20, 25]，此时将搜索码20更新入父结点。

删除15：

![](https://github.com/Rjerk/learning-notes/blob/master/img/bpt6.png?raw=true)

删除15后，其内结点只有[16], 而其后兄弟结点刚好满足叶结点容纳元素最小个数的条件无法分发，只能合并。合并到其后的兄弟结点后为[16, 20, 25]，此时它的父结点搜索码为空，只有一个指向叶结点的指针，不满足条件，此父结点需要借取它的兄弟的结点。

此父结点将父结点中的分离搜索码13移动到其内，从前面的兄弟结点[9, 11]拿走一个指向[11, 12]的孩子，此时父结点为[13], 父结点的两个孩子为[11, 12], [16, 20, 25]。它的左兄弟的最右边搜索码11移入它的父结点内。

删除1：

![](https://github.com/Rjerk/learning-notes/blob/master/img/bpt7.png?raw=true)

删除1后，当前叶结点合并入其兄弟结点内成为[4, 9, 10]，其父结点为空，该父结点与其兄结点进行合并，其兄弟结点此时为[11, 13]，其孩子为[4, 9, 10], [11, 12], [16, 20, 25]。此时根结点为没有搜索码，于是以其唯一孩子取而代之称为新的根节点。

## B+树实现

使用一个基类Node负责定义叶结点LeafNode和内部结点InternalNode共有的成员，每个派生类定义各自特有的成员，这里只列出部分成员：

基类Node:
```
class Node {
public:
	Node(int order_);
	Node(int order_, Node* parent_);
	virtual ~Node();

	int getOrder() const;
	Node* getParent() const;
	void setParent(Node* p);
	bool isRoot() const;
	
	virtual bool isLeaf() const = 0; 
	virtual size_t num() const = 0;
	virtual size_t maxNum() const = 0;
	virtual size_t minNum() const = 0;
private:
	const int order;
	Node* parent;
};
```

叶结点LeafNode的数据成员next指向下一个叶结点，mappings存储Key/Record*.
```
class LeafNode: public Node {
	using MappingType = std::pair<KeyType, Record*>;
public:
	LeafNode(int order_);
	LeafNode(int order_, Node* parent_);
	size_t insertToLeaf(KeyType key, ValueType value);
	size_t removeAndDeleteRecord(KeyType key);
	bool searchInLeaf(KeyType key);

    ...

	bool isLeaf() const override;
	size_t num() const override;
	size_t maxNum() const override;
	size_t minNum() const override;

    ...

private:
	LeafNode* next;
	std::vector<MappingType> mappings;
};
```

内部结点InternalNode使用mappings存储关键字和子节点指针对。

```
class InternalNode: public Node {
	using MappingType = std::pair<KeyType, Node*>;
public:
	InternalNode(int order_);
	~InternalNode();
	
	Node* lookUp(KeyType key) const;
	
    ...
	
	bool isLeaf() const override;
	size_t num() const override;
	size_t maxNum() const override;
	size_t minNum() const override;

    ...

private:
	std::vector<MappingType> mappings;
};
```

B+ tree有四种功能，查找，插入，删除，打印。

```
class BPlusTree {
	using LeafType = std::pair<LeafNode*, size_t>;
	using Entry = std::tuple<KeyType, ValueType, LeafNode*>;
public:
	BPlusTree(int order_ = default_order);
	~BPlusTree();
	LeafNode* search(KeyType key) const;
	void insert(KeyType key, ValueType value);
	void remove(KeyType key);
	void print();
private:

    ...	

	template <typename T>
	void coalesceOrRedistribute(T* node);
	template <typename T>
	void coalesce(T* node, T* sibling, InternalNode* parent, int node_index);
	template <typename T>
	void redistribute(T* node, T* sibling, InternalNode* parent, int node_index);
	void ajustRoot();
	bool empty() const;
private:
	Node* root;
	const int order;
};
```

### 查找的实现

找到key应在的叶结点
```
LeafNode* BPlusTree::findLeaf(KeyType key) const
{
	if (empty()) {
		return nullptr;
	}
	auto node = root;
	while (!node->isLeaf()) {
		auto internal_node = static_cast<InternalNode*>(node);
		node = internal_node->lookUp(key);
	}
	return static_cast<LeafNode*>(node);
}
```

然后在叶结点中找key.
```
bool BPlusTree::search(KeyType key) const
{
	LeafNode* leaf = findLeaf(key);
	return leaf->lookUp(key) != nullptr;
}
```

### 插入的实现

```
void BPlusTree::insert(KeyType key, ValueType value)
{
	if (empty()) {
		insertAsRoot(key, value);
	} else {
		insertIntoLeaf(key, value);
	}
}
```

树空时创建一个叶结点并将key/value插入
```
void BPlusTree::insertAsRoot(KeyType key, ValueType value)
{
	LeafNode* newnode = new LeafNode(order);
	newnode->insertToLeaf(key, value);
	root = newnode;
}
```

非空时找到应插入的结点，再插入key/value，插入后叶结点满了就分为两个结点。
```
void BPlusTree::insertIntoLeaf(KeyType key, ValueType value)
{
	LeafNode* node = findLeaf(key);
	if (node == nullptr) {
		// not found.
		return ;
	}

	if (node->insertToLeaf(key, value) > node->maxNum()) {
		LeafNode* sibling = split(node);
		sibling->setNext(node->getNext());
		node->setNext(sibling);
		KeyType k = sibling->firstKey();
		insertToParent(node, sibling, k);
	}
}
```

node分一半到兄弟结点里。
```
template <typename T>
T* BPlusTree::split(T* node)
{
	T* sibling = new T(order, node->getParent());
	node->moveHalfTo(sibling);
	return sibling;
}
```

### 删除的实现

```
void BPlusTree::remove(KeyType key)
{
	if (empty()) {
		return ;
	} else {
		return removeFromLeaf(key);
	}
}
```

树非空时找到key，然后删除相应record，如果删除后结点不够满，就进行合并或分发操作。
```
void BPlusTree::removeFromLeaf(KeyType key)
{
	LeafNode* leafnode = findLeaf(key);
	if (!leafnode) {
		return ;
	}
	if (!leafnode->lookUp(key)) {
		return ;
	}
	
	size_t new_size = leafnode->removeAndDeleteRecord(key);
	if (new_size < leafnode->minNum()) {
		coalesceOrRedistribute(leafnode);
	}
}
```

如果其兄弟结点内搜索码大于⌈(n-1)/2⌉个，即两个结点搜索码之和大于最大值，则进行分发(redistribute)，否则进行合并操作(coalesce).
```
template <typename T>
void BPlusTree::coalesceOrRedistribute(T* node)
{
	if (node->isRoot()) {
		ajustRoot();
		return ;
	}
	auto parent = static_cast<InternalNode*>(node->getParent());
	size_t node_index = parent->nodeIndex(node);
	// node_index == 0, sibling is right sibling, else it is left sibling.
	size_t sibling_index = (node_index == 0 ? 1 : node_index-1);
	// get sibling node.
	T* sibling = static_cast<T*>(parent->sibling(sibling_index));
	
	if (node->num() + sibling->num() >= sibling->maxNum()) {
		redistribute(sibling, node, parent, node_index);
	} else {
		coalesce(sibling, node, parent, node_index);
	}
}
```

### 打印的实现

使用队列queue进行分层打印。

```
void BPlusTree::print() const
{
	if (root == nullptr) {
		cout << "empty tree.\n";
	} else {
		std::queue<Node*> q0;
        std::queue<Node*> q1;
        auto currentLevel = &q0;
        auto nextLevel = &q1;
        currentLevel->push(root);
        while (!currentLevel->empty()) {
            printCurrentLevel(currentLevel, nextLevel);
            std::swap(currentLevel, nextLevel);
        }
	}
}
```

```
void BPlusTree::printCurrentLevel(std::queue<Node*>* currentLevel, std::queue<Node*>* nextLevel) const
{
    cout << "|";
    while (!currentLevel->empty()) {
        Node* currentNode = currentLevel->front();
        cout << currentNode->displayAllKey();
        cout << " |";
        if (!currentNode->isLeaf()) {
            auto internal_node = static_cast<InternalNode*>(currentNode);
            internal_node->queuePushChild(nextLevel);
        }
        currentLevel->pop();
    }
    cout << endl;
}
```

以上完整实现见 [B+ Tree implementation](https://github.com/Rjerk/snippets/tree/master/data-structure-exercise/b%2Btree-implementation)


### 算法测试

命令：

- 插入 i [key]
> 插入(key, key) 并打印b+树
- 删除 d [key]
> 删除(key, key) 并打印b+树
- 查找 s [key]
> 返回查找结果
- 打印 p
> 按层打印整个b+树的结点与key

```
b+tree> i 1
|1 |
b+tree> i 2
|1 2 |
b+tree> i 5
|1 2 5 |
b+tree> i 3
| 3 |
|1 2 |3 5 |
b+tree> i 3
| 3 |
|1 2 |3 5 |
b+tree> i 4
| 3 |
|1 2 |3 4 5 |
b+tree> i 9
| 5 |
| 3 |5 |
|1 2 |3 4 |5 9 |
b+tree> s 1
1 exists in tree.
b+tree> s 2
2 exists in tree.
b+tree> s 10
Not found.
b+tree> d 1
| 5 |
| 4 |5 |
|2 3 |4 |5 9 |
```