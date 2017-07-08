# Singleton 模式

## 意图

保证一个类只有一个实例，并提供一个访问它的全局访问点。

一个全局变量使得一个对象可以被访问，但是不能防止用户实例化多个对象。更好的办法是，让类自身负责保存它的唯一实例，它可以保证没有其他实例可以被创建，并提供一个访问该实例的方法。这就是 Singleton 模式。

## 适用性

可以使用 Singleton 模式的情况：

- 当类只能有一个实例且客户可以从一个众所周知的访问点访问它时。
- 当这个唯一实例应该是通过子类化可扩展的，而且客户应该无需更改代码就可以使用一个扩展的实例时。

## 结构

![](https://github.com/Rjerk/learning-notes/blob/master/img/Singleton.png?raw=true)

- Singleton
  - 定义一个 Instance 操作，允许客户访问它的唯一实例。Instance 是一个类操作。
  - 可能负责创建它的唯一实例。
  - 客户只能通过 Singleton 的 Instance 操作访问一个 Singleton 的实例。

## 效果

Singleton 模式的优点：

- 对唯一实例的受控访问

Singleton 类封装它的唯一实例，所以可以严格控制客户怎样以及何时访问它。

- 缩小名称空间

对全局变量进行改进，避免了那些存储唯一实例的全局变量污染名称空间。

- 允许对操作和表示的精化

Singleton 类可以有子类，用这个扩展类的实例来配置一个应用很容易，可以用所需要的类的实例在运行时刻配置应用。

- 允许可变数目的实例

允许 Singleton 类的多个实例，并控制应用所使用的实例数目，只有允许访问 Singleton 实例的操作需要改变。

- 比类操作更灵活

另一种封装单件功能的方式是使用类操作，即 C++ 中的静态成员函数，但 C++ 中静态成员函数不是虚函数，所以子类不能多态地重定义它们。

## 实现

所需要考虑的实现问题：

- 保证唯一的实例

常用方法是将创建这个实例的操作隐藏在一个类操作后，由它保证只有一个实例被创建，并且这个操作可以访问保存的唯一实例的变量，并保证单件在它首次使用前被创建和使用。

```
class Singleton {
public:
	static Singleton* Instance()
	{
		if (_instance == nullptr) {
			_instance = new Singleton;
		}
		return _instance;
	}
protected:
	Singleton();
private:
	static Singleton* _instance;
};
```

- 创建 Singleton 类的子类

即建立它的唯一实例。指向单件实例的变量必须用子类的实例进行初始化。

最简单的技术是在 Singleton 的 Instance 操作中决定你想使用的是哪一个单件。

另一个选择 Singleton 子类的方法是将 Instance 的实现从父类中分离出来并把它放在子类中，这允许 C++ 程序员在链接时期就决定单件的类（即通过链入一个包含不同实现的对象文件），但对单件客户隐藏这一点。但是这使得难以在运行时期选择单件类，虽然可以使用条件语句来决定子类增加灵活度，但也硬性限定了可能的 Singleton 的集合。

一个更灵活的方法是使用一个单件注册表，可能的 Singleton 类的集合不是由 Instance 定义，而是根据名字在一个众所周知的注册表中注册它们的单件实例。注册表在字符串和单件之间建立映射，当 Instance 需要一个单件，它参考注册表，根据名字请求单件，它所需要的只是所有 Singleton 类的一个公共接口，该接口包含了对注册表的操作。

```
class Singleton {
public:
	static Singleton* Instance()
	{
		if (_instance == nullptr) {
			const char* singleton_name = getenv(/*SINGLETON*/);
			_instance = lookUp(singleton_name);
		}
		return _instance;
	}
	static void Register(const char* name, Singleton*);
protected:
	Singleton();
	static Singleton* lookUp(const char* name);
private:
	static Singleton* _instance;
	static List<NameSingletonPair>* _registry;
};
```

Singleton 类可以在它们的构造器中注册它们自己，如 MySingleton 子类中：

```
MySingleton::MySingleton() {
    // ...
    Singleton::register("MySingleton", this);
}
```

除非实例化类否则这个构造器不会被调用，可以在 MySingleton 实现文件中定义一个 MySingleton 的一个静态实例来避免这个问题：

```
static MySingleton the_singleton;
```

一个简单 Singleton 类的实现：[Singleton.cpp](https://github.com/Rjerk/snippets/blob/master/design-patterns/Singleton.cpp)

## 相关模式

很多模式可以使用 Singleton 模式实现：

- Abstract Factory
- Builder
- Prototype