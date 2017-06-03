# Member Function 的各种调用方式

## Nonstatic Member Function

为保证 Nonstatic member funtions 和一般的 non-member function 的效率一样，比如：

```
float magnitude3d(const Point3d* _this) { ... }
float Point3d::magnitude() const { ... }
```

一般要保证上面两个函数的效率相等，不能因为非成员函数的类对象还要对其成员进行取值而影响效率，比如：

```
Point3d Point3d::magnitude(Point3d* const _this)
{
    return sqrt(_this->_x * _this->_x +
                _this->_y * _this->_y +
                _this->_z * _this->_z);
}
```

在 member function 会有内化操作：

1、改写函数的原型

将 `float Point3d::magnitude();` 改写为：

```
Point3d Point3d::magnitude(Point3d* const _this);
```

2、对于 nonstatic member 全部用 `this` 进行存取

```
Point3d Point3d::magnitude(Point3d* const this)
{
    return sqrt(this->x * this->x +
                this->y * this->y +
                this->z * this->z);
}
```

3、将函数名进行 mangling 操作，使得该函数在程序中称为独一无二的名称：

```
extern magnitude_7Point3dFV(register Point3d* const this);
```

如此一来，每次对于该函数的访问： `obj.magnitude()` 就会转换成：

```
magnitude_7Point3dFV(&obj);
```

而 `ptr->magnitude()` 则转换成：

```
magnitude_7Point3dFV(ptr);
```

## Virtual Member Function

如果 normalize() 是一个 virtual member function，那么对于 `ptr->normalize()` 的调用会被转换为：

```
(*ptr->vptr[1])(ptr);
```

回想一下在 C++ 中的对象模型：对于每个对象会在其中安插一个指向虚表的指针，而虚表内的指针指向虚函数。

```
          virtual table
          | normalize |
vptr ---> | magnitude |
          | other virual function |
```

vptr 是编译器产生的指向虚表的指针，因为一个类对象也可能有多个 vptr，所以 vptr 名字实际上也会被 mangled.

1 是虚表槽的索引值，关联到 normalize() 函数。

第二个 ptr 表示 this.

如果 magnitude() 在 normalize() 函数内调用，那么由于虚拟机制已决议妥当，那么对于 magnitude() 的调用十分有效率：

```
register float mag = Point3d::magnitude(ptr);
```

如果 magnitude() 被声明为 inline 函数会更有效率。使用 class scoped operator 明确调用一个 virtual function 会和调用 nonstatic member function 一样：

```
register float mag = magnitude_7Point3dFv(this);
```

对于 `obj.normalize()`，编译器会将 "经由一个 class object 调用一个 virtual function" 转换为：

```
normalize_7Point3dFv(&obj);
```

## Static Member Function

如果 Point3d::normalize() 是一个 static member funcion, 那么对于 `obj.normalize()`、`ptr->normalize()` 的调用，编译器会将其转换为一般的 nonmember function：

```
normalize_7Point3dSFv();
```

