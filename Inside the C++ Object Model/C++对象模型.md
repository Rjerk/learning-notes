# C++ 对象模型

C++ 中有两种类数据成员：static 和 non-static, 以及三种类成员函数：static、nonstatic 以及 virtual.

- static 数据成员被独立地存储在类对象之外；
- non-static 数据成员被分配在每个类对象之内；
- static 和 non-static 成员函数也被独立地存放在类对象之外；
- virtual 函数则以两步支持：
 1. 每个类产生一个虚表 vtbl（virtual table)用来存放一些指向虚函数的指针。
 2. 每个类对象有一个指针指向相关的虚表，这个指针通常称为 vptr。vptr 的设定和重置都由每个类的构造函数、析构函数和拷贝赋值运算符自动完成。用来支持 runtime type identification（RTTI）的每个类关联的 type_info object 也是通过虚表指出来，通常放在虚表的第一个 slot 里。

用一个 Point 类来说明：

```
class Point {
public:
    Point(float xval);
    virtual ~Point();

    float x() const;
    static int PointCount();
protected:
    virtual ostream& print(ostream& os) const;

    float _x;
    static int _point_count;
};
```

对应的 C++ 对象模型： 

![](https://github.com/Rjerk/learning-notes/blob/master/img/c++_object_model1.png?raw=true)


### 加上继承情况下的对象模型

C++ 最初采用的继承模型是：基类的部分数据成员被直接存放在派生类对象中，这提供了对基类成员最紧凑和最有效的存取，但缺点是基类成员的任何改变都会让使用基类或派生类对象的代码重新编译。

自引入虚基类（virtual base class），需要一些间接的基类表现方法。虚基类的原始模型是在基类对象中为每一个有关联的虚基类加上一个指针，其他演化出来的模型一般是导入一个虚基类表或扩展已存在的虚表，以维护每个虚基类的位置。

