# Abstract Factory 模式

## 意图

提供一个创建一系列相关或相互依赖对象的接口，而无需指定它们的具体的类。

## 适用性

可以使用 Abstract Factory 模式的情况：

- 一个系统要独立于它的产品的创建、组合和表示时。
- 一个系统要由多个产品系列中的一个来配置时。
- 当你要强调一系列相关的产品对象的设计以便进行联合使用时。
- 当你提供一个产品类库，而只想显示它们的接口而不是实现时。

## 结构

![](https://github.com/Rjerk/learning-notes/blob/master/img/Abstract_Factory.png?raw=true)

- AbstractFactory
 - 声明一个创建抽象产品对象的操作接口。
- ConcreteFactory
 - 实现创建具体产品对象的操作。
- AbstractProduct
 - 为一类产品对象声明一个接口。 
- ConcreteProduct
 - 定义一个将被相应的具体工厂创建的产品对象。
 - 实现 AbstractProduct 接口。
- Client
 - 仅使用由 AbstractFactory 和 AbstractProduct 类声明的接口。 

## 协作

- 通常在运行时刻创建一个 ConcreteFactory 类的实例。这一具体的工厂创建具有特定实现的产品对象，客户使用不同的具体工厂创建不同的产品对象。
- AbstractFactory 将产品对象的创建延迟到它的 ConcreteFactory 子类。

## 效果

Abstract Factory 的优缺点：

- 分离了具体的类

一个工厂封装创建产品对象的责任和过程，它将客户与类的实现分离。客户通过它们的抽象接口操纵实例。产品的类名也在具体工厂的实现中被分离，它们不出现在客户代码中。

- 使得基于交换产品系列

一个具体工厂类在一个应用中仅出现一次，即初始化时。因为一个抽象工厂创建了一个完整的产品系列，所以只需改变具体的工厂就能使用不同的产品配置。

- 有利于产品的一致性

当一个系列中的产品对象被设计成一起工作时，一个应用一次只能使用同一系列中的对象。AbstractFactory 很容易实现这一点。

- 难以支持新种类的产品

由于 AbstractFactory 接口确定了可以被创建的产品集合，新种类产品的添加需要扩展该工厂接口，这涉及到 AbstractFactory 类及其所有子类的改变。

## 实现

实现 AbstractFactory 模式的一些有用的技术：

- 将工厂作为单件

一个应用中的每个产品系列只需一个 ConcreteFactory 的实例，因此工厂通常最好实现为一个 Singleton。

- 创建产品

通常为每个产品定义一个工厂方法（Factory Method）。一个具体的工厂将为每个产品重定义该工厂方法以指定产品，但要求每个产品系列都要有一个新的具体工厂子类。

如果有多个可能的产品系列，具体工厂也可以用 Prototype 模式实现。具体工厂使用产品系列中每一个产品的原型实例来初始化，且通过复制它的原型来创建新的产品。在基于原型的方法中，使得不是每个新的产品系列都需要一个新的具体工厂类。

- 定义可扩展的工厂

AbstractFactory 通常为每一种它可以生产的产品定义一个操作，产品的种类被编码在操作型构中，增加新的产品要求改变 AbstractFactory 的接口以及所有与它相关的类。

一种灵活但不太安全的设计是给创建对象的操作增加一个参数，该参数指定了将被创建的对象的种类。在 C++ 中，仅当所有对象都有相同的抽象基类，或者当产品对象可以被请求它们的客户安全的强制类型转换为正确类型时，才可以使用它。

AbstractFactory 模式的简单实现：[AbstractFactory.h](https://github.com/Rjerk/snippets/blob/master/design-patterns/AbstractFactory.h)

## 相关模式

- Factory Method
- Prototype
- Singleton
