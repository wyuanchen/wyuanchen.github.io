---
title: const关键字复习
date: 2017-03-17 23:56:06
categories: C++
tags: C++
---

## const对象默认为文件的局部变量

在全局作用域里定义非`const`变量时，它在整个程序中都可以访问，我们可以把一个非`const`变量定义在一个文件中，假设已经做了合适的声明，就可以在另外的文件中使用这个变量：

```cpp
//file_1.cpp
int a; //definition

//file_2.cpp
extern int a; //使用的count来自file_1.cpp
a++;
```

与其他变量不同，除非特别说明，在全局作用域声明的`const`变量是定义该对象的文件的局部变量。此变量只存在于那个文件中，不能被其他文件访问。   
通过指定`const`变量为`extern`，就可以在整个程序中访问`const`对象。

<!-- more -->

```cpp
//file_1.cpp
extern const int V = 8;

//file_2.cpp
extern const int V; //使用的V来自file_1.cpp
V++;
```

## const对象的动态数组

### 内置类型

如果我们在自由存储区中创建的数组存储了**内置类型的`const`对象**，则必须为这个数组提供初始化： 因为数组元素都是`const`对象，无法赋值。实现这个要求的唯一方法是对数组做值初始化。

```cpp
const int *p1 = new const int[100]; //错误

const int *p2 = new const int[100](); //正确
```

### 非内置类型

C++允许定义类类型的const数组，但该类类型必须提供默认构造函数：

```cpp
const string *p3 = new string[100]; //这里会调用string类的默认构造函数初始化数组元素
```


## 用const修饰指针

- 指针本身是常量不可变：`char* const pContent; `

- 指针所指向的内容是常量不可变：`const char *pContent; `

- 两者都不可变：`const char* const pContent; `


## 用const修饰函数的参数

### 指针传递

如果输入参数采用**“指针传递”**，那么加`const` 修饰可以防止意外地改动该指针，起到保护作用

### 值传递

如果输入参数采用**“值传递”**，由于函数将自动产生临时变量用于复制该参数，该输入参数本来就无需保护，所以不要加`const` 修饰

### 引用传递

为了提高效率，可以将函数声明改为`void Func(A &a)`，因为**“引用传递”**仅借用一下参数的别名而已，不需要产生临时对象。    

但是函数`void Func(A &a)` 存在一个缺点：“引用传递”有可能改变参数a，这是我们不期望的。解决这个问题很容易，加`const`修饰即可，因此函数最终成为`void Func(const A &a)`。

#### 内置数据类型

对于**内置数据类型**的输入参数，不要将**“值传递”**的方式改为**“`const` 引用传递”**。否则既达不到提高效率的目的，又降低了函数的可理解性。    

**因为内部数据类型的参数不存在构造、析构的过程，而复制也非常快，“值传递”和“引用传递”的效率几乎相当**
          
例如`void Func(int x)` 不应该改为`void Func(const int &x)`。

#### 非内置数据类型

对于非内置数据类型的输入参数，应该将**“值传递”**的方式改为**“`const` 引用传递”**，目的是提高效率。

例如将`void Func(A a)` 改为`void Func(const A &a)`。


## 用const 修饰函数的返回值

如果给以“指针传递”方式的函数返回值加`const`修饰，那么函数返回值（即指针）的内容不能被修改，该返回值只能被赋给加`const`修饰的同类型指针。

例子：

```cpp
const char * GetString(void);

//如下语句将出现编译错误：
char *str = GetString();

//正确的用法是
const char *str = GetString();
```

### 值传递方式

如果函数返回值采用**“值传递方式”**，由于函数会把返回值复制到外部临时的存储单元中，加`const` 修饰没有任何价值。

- 例如不要把函数`int GetInt(void)` 写成`const int GetInt(void)`。

- 同理不要把函数`A GetA(void) `写成`const A GetA(void)`，其中`A` 为用户自定义的数据类型。

如果**返回值不是内部数据类型**，将函数`A GetA(void)` 改写为`const A & GetA(void)`的确能提高效率。    
但此时千万千万要小心，**一定要搞清楚函数究竟是想返回一个对象的“拷贝”还是仅返回“别名”就可以了，否则程序会出错**。


### 引用传递方式

函数返回值采用“引用传递”的场合并不多，这种方式一般只出现在类的赋值函数中，目的是为了实现链式表达。

```cpp
class A
{
A & operate = (const A &other); // 赋值函数
};

A a, b, c; // a, b, c 为A 的对象

a = b = c; // 正常的链式赋值
(a = b) = c; // 不正常的链式赋值，但合法
```

如果将赋值函数的返回值加`const`修饰，那么该返回值的内容不允许被改动。   
上例中，语句 `a = b = c` 仍然正确，但是语句 `(a = b) = c` 则是非法的。


## const修饰成员变量

- `const`修饰类的成员函数，表示成员常量，不能被修改   

- **同时它只能在初始化列表中赋值**
    ```cpp
    class A
    { 
        ...
        const int nValue;         //成员常量不能被修改
        ...
        A(int x): nValue(x) { } ; //只能在初始化列表中赋值
    }
    ```

## const修饰成员函数

- `const` 关键字只能放在函数声明的尾部，大概是因为其它地方都已经被占用了

- 一个函数通过在其后面加关键字`const`，它将被声明为常量函数

- 在C++，**只有将成员函数声明为常量函数才有意义**

- `const`成员函数不可以修改对象的数据,不管对象是否具有`const`性质

- 在C++中，一个对象的所有方法都接收一个指向对象本身的隐含的`this`指针；常量方法则获取了一个**隐含的常量`this`指针**，所以：

    - 不可以修改对象的数据成员，不管该成员是否具有`const`

    - 不可以调用其他**非`const`成员函数**，因为任何非`const`成员函数会有修改成员变量的企图

- **常量函数可以被任何对象调用，而非常量函数则只能被非常量对象调用，不能被常量对象调用**，因为任何非`const`成员函数会有修改成员变量的企图
    例如：

    ```cpp
    class Fred {
    public:
      void inspect() const;   // This member promises NOT to change *this
      void mutate();          // This member function might change *this
    };

    void userCode(Fred& changeable, const Fred& unchangeable)
    {
      changeable.inspect();   // OK: doesn't change a changeable object
      changeable.mutate();    // OK: changes a changeable object

      unchangeable.inspect(); // OK: doesn't change an unchangeable object
      unchangeable.mutate();  // ERROR: attempt to change unchangeable object
    } 
    ```

- 在类中允许存在同名的常量函数和非常量函数（**算是函数重载**），编译器根据调用该函数的对象选择合适的函数

    - 当非常量对象调用该函数时，先调用非常量函数

    - 当常量对象调用该函数时，只能调用常量函数
    
    如果在类中只有常量函数而没有与其同名的非常量函数，则非常量与常量对象都可调用该常量函数。如： 

    ```cpp
    class A
    {
        void f() const { cout<<"const function f is called"<<endl; }
        void f() { cout<<"non-const function f is called"<<endl; }
        void g() { cout<<"non-const function g is called"<<endl; } 
    };

    void main()
    {
        A a;
        const A& ref_a = a;
        a.f(); //calls void f()
        ref_a.f();//calls void f () const
        a.g(); //ok, normal call
        ref_a.g(); //error, const object can not call non-const function
    }
    ```  

- `const`关键字不能用在构造函数与析构函数中。因为构造函数和析构函数都必须更改对象


## 将const类型转化为非const类型的方法

### 用const_cast来去除const限定

`const_cast`转换符是用来移除变量的`const`或`volatile`限定符。

例如：

```cpp
const int constant = 21;
const int* const_p = &constant;
int* modifier = const_cast<int*>(const_p); //注意这行
*modifier = 7;
```

### 传统转换方式实现const_cast运算符

准转换运算符是可以用传统转换方式实现的。   

`const_cast`实现原因就在于**C++对于指针的转换是任意的，它不会检查类型，任何指针之间都可以进行互相转换**，因此`const_cast`就可以直接使用显式转换`(int*)`来代替：

```cpp
const int constant = 21;
const int* const_p = &constant;
int* modifier = (int*)(const_p);
```

### 注意

C++里是`const`，就是`const`，外界千变万变，我就不变。不然真的会乱套了，`const`也没有存在的意义了。

`const_cast`用来丢弃变量的`const`声明，但不能改变变量所指向的对象的const属性。     即：`const_cast`用于原本非`const`的对象；如果用于原本`const`的对象，结果不可预知（C++语言未对此种情况进行规定）

一般情况下`const_cast`是用于这种情形：    
`const`指针（变量）指向非const对象，程序员确认这一点（所指向的对象非`const`）时，使用`const_cast`操作符丢弃变量的`const`修饰获得一个非`const`指针


