# C++：RTTI

运行时类型识别（run-time type identification, RTTI）是指在程序运行时保存期对象动态类型的行为。

其功能由两个运算符实现：
- dynamic_cast：用于将基类的指针或引用安全地转换成派生类的指针或引用
- typeid：用于返回表达式的类型

## dynamic_cast

dynamic_cast 运算符使用形式如下：

```
dynamic_cast<type*>(e)   (1)
dynamic_cast<type&>(e)   (2)
dynamic_cast<type&&>(e)  (3)
```

type 必须是一个类类型，通常情况下该类类型应该有虚函数。(1)中，e 必须是个有效的指针；(2)中，e 必须是个左值；(3)中，e 不能是左值。

e 的类型必须符合三个条件中的一个：目标 type 的共有派生类、目标 type 的共有基类或者目标 type 的类型。

如果类型符合，则转换成功；否则转换失败。如果转换目标是指针类型且转换失败，则结果为 0；如果转换目标是引用类型且转换失败，则抛出一个 `std::bad_cast` 异常。

示例：

```
struct Base {
	virtual void f() {}
};

struct Derived : public Base { };

struct S {
	S() {} 
};

int main()
{
	Base b;
	Derived d;
	
	Base* bp = &d;
	if (Derived* dp1 = dynamic_cast<Derived*>(&b)) { // NULL, 'b' is not a 'Derived'
		cout << "pointer1 cast ok.\n";
	} else {
		cerr << "pointer1 cast failed.\n";
	}
	if (Derived* dp2 = dynamic_cast<Derived*>(bp)) {  // ok
		cout << "pointer2 cast ok.\n";
	} else {
		cerr << "pointer2 cast failed.\n";
	}

	if (S* sp = dynamic_cast<S*>(bp)) {
		cout << "pointer3 cast ok.\n";
	} else {
		cerr << "pointer3 cast failed.\n";
	}
	
	try {
		Base& br = dynamic_cast<Base&>(*bp); // ok
	} catch (std::bad_cast) {
		cerr << "reference1 cast failed.\n";
	}
	
	try {
		Derived& dr = dynamic_cast<Derived&>(*bp); // ok
	} catch (std::bad_cast) {
		cerr << "reference2 cast failed.\n";
	}
	
	try {
		S& sr = dynamic_cast<S&>(*bp); // std::bad_cast
	} catch (std::bad_cast) {
		cerr << "reference3 cast failed.\n";
	}
}
```

输出：

```
pointer1 cast failed.
pointer2 cast ok.
pointer3 cast failed.
reference3 cast failed.
```

## typeid

typeid 表达式的形式是 typeid(e)，其中 e 可以是任意表达式或者类型的名字。typeid(e) 操作的结果是一个常量对象的引用，其类别是标准库类型 typeinfo 或 typeinfo 的共有派生类型。

如果表达式 e 是一个一用，则 typeid 返回该引用所引用对象的类型。如果 typeid 作用于数组或函数时，并不会执行向指针的标准类型转换，所得的结果是数组类型而非指针类型。

当运算对象不属于类类型或者是一个不包含任何虚函数的类时，typeid 运算符指示的是运算对象的静态类型。当运算对象是定义了至少一个虚函数的类的左值时，typeid 的结果直到运行时才会求得。

typeid 是否需要运行时检查决定了表达式是否被求值。只有当类型含有虚函数时，编译器才会对表达式求值。反之，typeid 返回表达式的静态类型。编译器无需对表达式求值也能知道表达式的静态类型。

如果表达式的动态类型可能与静态类型不同，则必须在运行时对表达式求值以确定返回的类型。这条规则适用于 `typeid(*p)` 的情况。如果指针 p 所指的类型不含虚函数，则 p 不一定非得是一个有效的指针；否则 `*p` 在运行时求值，此时 p 必须是一个有效的指针。如果 p 是一个空指针，则 `typeid(*p)` 将抛出一个 bad_typeid 的异常。

```
struct Base {
	virtual void f() {}
};

struct Derived : public Base { };

int main()
{
	Derived* dp = new Derived;
	Base* bp = dp;
	if (typeid(*bp) == typeid(*dp))
		cout << "*bp and *dp have same type.\n";
	
	if (typeid(*bp) == typeid(Derived))
		cout << "*bp is Derived type.\n";
    // 当 typeid 作用于指针时，返回该指针静态编译时的类型
    if (typeid(bp) == typeid(Derived))
        cout << "bp is Derived type.\n";
    else if (typeid(bp) == typeid(Base*))
    	cout << "bp is Base* type.\n";

    Base* bp2 = nullptr;
    try {
    	if (typeid(*bp2) == typeid(Derived)) 
    		cout << "*bp2 have Derived type.\n";
	} catch (std::bad_typeid) { // catch std::bad_typeid
		cout << "bad_typeid.\n";
	}
    
	int* p = nullptr;
	if (typeid(*p) == typeid(int))
		cout << "*p have int type.\n";
}
```

输出：

```
*bp and *dp have same type.
*bp is Derived type.
bp is Base* type.
bad_typeid.
*p have int type.
```

## 使用 RTTI

某些情况下使用 RTTI 非常有用，比如为具有继承关系的类实现相等运算符时。


## typeinfo 类

typeinfo 类定义在 <typeinfo> 中，包含一个类型的特定实现信息，包括类型名和比较两个类型的相等性和顺序关系操作。它没有默认构造函数，也拷贝和移动构造函数以及赋值运算符都被定义为删除的。创建 typeinfo 的唯一途径是使用 typeid 运算符。

typeinfo 类的 name 成员函数返回一个 C 风格的字符串，表示类型的名字。其返回值因编译器而异不一定与在程序中使用的名字一致，唯一要求是类型不同则返回的字符串必须有所区别。

```
int arr[10];
Derived d;
Base* bp = &d; 
	
cout << typeid(42).name() << "\n"
     << typeid(arr).name() << "\n"
	 << typeid(std::string).name() << "\n"
	 << typeid(Derived).name() << "\n"
	 << typeid(bp).name() << "\n";

if (typeid(Base).before(typeid(Derived))) {
    cout << "Base goes before Derived in this implementation.\n";
} else {
	cout << "Derived goes before Base in this implementation.\n";
}
```

输出：

```
i
A10_i
Ss
7Derived
P4Base
Base goes before Derived in this implementation.
```