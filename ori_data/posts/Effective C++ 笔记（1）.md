---
title: Effective C++ 笔记（1）
date: 2017-04-16 20:48:44
categories: C++
tags: C++
---


## 尽量以 const, enum, inline 替换 #define

当我们以常量替换`#define`，有两种特殊情况值得说说，

第一是定义常量指针，  
由于常量定义式通常放在头文件内（以便被不同源码含入），因此有必要将指针（而不是指针所指之物）声明为`const`。

例如要在头文件内定义一个常量的（不变的）`char*-based`字符串，你必须写`const`两次：

```cpp
const char* const NAME = "Wen Yuan";
```

其实用`string`对象通常比`char*-based`合宜，所以上面的代码可以改为：

```cpp
const std::string NAME("Wen Yuan");
```

第二个值得注意的是`class`专属常量，   
为了将常量的作用域（scope）限制于`class`内，你必须让它成为`class`的一个成员（member）；而为确保此常量至多只有一份实体，你必须让它成为一个`static`成员。

<!-- more -->

例如：

```cpp
class A{
private:
    static const int NUM = 5;  //常量声明式
    int scores[NUM];  //使用该常量
}
```

然而你看到的是`NUM`的声明式而非定义式。通常 C++ 要求你对你所使用的任何东西提供一个定义式，但如果它是个`class`专属常量又是`static`且为整数类型（例如`int`，`char`，`bool`），则需特殊处理：

- 只要不取它们的地址，你可以声明并使用它们，而无须提供定义式。    

- 但如果你取某个`class`专属常量的地址，或纵使你不取其地址而你的编译器却坚持要看到一个定义式，你就必须另外提供定义式如下：

    ```cpp
    const int A::NUM; //NUM的定义，下面告诉你为什么没有给予赋值
    ```
    
    请把这个式子放进一个实现文件而非头文件。   
    由于`class`常量已经在声明时获得初值（例如之前声明`NUM`时为它设置初始值`5`），因此定义时不可以再设置初值。

**注意：我们无法使用`#define`创建一个`class`专属常量**

旧式编译器也许不支持上述语法，它们不允许`static`成员在其声明式上获得初值。   
此外所谓的`in-class 初值设定`也只允许对整数常量进行，因此你可以把初值放在定义式：

```cpp
class B{
private:
    static const double FACTOR; //static class 常量声明，位于头文件内
}
```

```cpp
const double B::FACTOR = 6.66; //static class 常量定义，位于实现文件内
```

这几乎是你在任何时候唯一需要做的事情，唯一的例外是当你在`class`编译期间需要一个`class`常量值，例如：

```cpp
class A{
private:
    static const int NUM; //这样不行
    int scores[NUM];
}
```

在上述的`scores`数组声明式中，编译器坚持必须在编译期间知道数组的大小，而且万一这时候你的编译器不允许`static 整数型 class 常量完成 in class 初值设定`，可以改用所谓的`the enum hack`补偿做法。

其理论基础是：一个属于枚举类型（enumerated type）的数值可权充`ints`被使用，   
于是可以这么写

```cpp
class A{
private:
    enum{NUM = 5}; //"the enum hack"-令NUM成为5的一个记号名称
    
    int scores[NUM];
}
```

基于数个理由`enum hack`值得我们认识：

- `enum hack`的行为某方面比较像`#define`而不像`const`，   
    有时候这正是你想要的，例如取一个`const`的地址是合法的，但是取一个`enum`的地址就不合法，而取一个`#define`的地址也通常不合法。   
    如果你不想让别人获得一个`pointer`或者`reference`指向你的某个整数常量，`enum`可以帮助你实现这个约束。


另外一个常见的`#define`误用情况是以它实现宏（macros）。   

宏看起来像函数，但不会招致函数调用（function call）带来的额外开销。   
例如

```cpp
#define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))
```

你可以获得宏带来的效率以及一般函数的所有可预料行为和类型安全性（type safety），只要你写出`template inline`函数：

```cpp
template<typename T>
inline void callWithMax(const T& a, const T& b)
{
    f(a > b ? a : b);   
}
```

有了`const`、`enum`和`inline`，我们可以对预处理器（特别是`#define`）的需求降低了，但并非完全消除。   
`#include`仍然是个必需品，而`#ifdef`和`#ifndef`也继续扮演控制编译的重要角色。

### 小结

- 对于单纯常量，最好以`const`或者`enums`替换`#define`

- 对于形似函数的宏（macros），最好改用`inline`函数替换`#define`


## 尽可能使用const

### 两个成员函数如果只是常量性不同，也算是重载。

```cpp
class TestBook{
public:
    ...
    const char& operator[](std::size_t position) const  //operator[] for const 对象
    {
        return text[position];
    }

    char& operator[](std::size_t position) //operator[] for non-const 对象
    {
        return text[position];
    }
};
```

`TextBook`的`operator[]s`可以被这么使用：

```cpp
TextBook tb;
std::cout << tb[0]; //调用non-const TextBook::operator[]

const TextBook  ctb;
std::cout << ctb[0]; //调用const TextBook::operator[]
```

### bitwise constness 和 logical constness

考虑以下例子：

```cpp
class CTextBlock{
public:
    ...
    std::size_t length() const;
private:
    char* pText;
    std::size_t textLength; //最近一次计算的文本区域长度
    bool lengthIsValid; //目前的长度是否有效
};
std::size_t CTextBlock::length() const
{
    if (!lengthIsValid)
    {
        //错误！在const函数内不能修改成员数据textLength和lengthIsValid
        textLength = std::strlen(pText); 
        lengthIsValid = true;
    }
    return textLength;
}
```

`length()`的实现不是`bitwise const`，因为`textLength`和`lengthIsValid`都可能被修改。   
但是这两个数据被修改对`const CTextBlock`对象而言虽然可接受，但编译器不通过，它们坚持`bitwise constness`，怎么办？

解决方法：

利用 C++ 的一个与`const`相关的摆动场：`mutable（可变的）`。

`mutable`会释放掉`non-static`成员变量的`bitwise constness`约束：

```cpp
class CTextBlock{
public:
    ...
    std::size_t length() const;
private:
    char* pText;

    //mutable表示这些变量可能总是会被修改，基本是在const成员函数内
    mutable std::size_t textLength; 
    mutable bool lengthIsValid;
};
std::size_t CTextBlock::length() const
{
    if (!lengthIsValid)
    {
        //现在可以正常运行了
        textLength = std::strlen(pText); 
        lengthIsValid = true;
    }
    return textLength;
}
```

### 在 const 和 non-const 成员函数中避免重复

假设`TextBlock`的`operator[]`不单只是返回一个`reference`指向字符，也执行边界检验，标记访问信息甚至可能进行数据完善性检验。如：

```cpp
class TestBlock{
public:
    ...
    const char& operator[](std::size_t position) const  //operator[] for const 对象
    {
        ... //边界检验
        ... //标记数据访问
        ... //检验数据完整性
        return text[position];
    }

    char& operator[](std::size_t position) //operator[] for non-const 对象
    {
        ... //边界检验
        ... //标记数据访问
        ... //检验数据完整性
        return text[position];
    }
};
```

这里发生大量代码重复以及伴随着编译时间、维护、代码膨胀等问题。

当然，可以把边界检验...等所有代码转移到另外一个成员函数（往往是`private`）并令两个版本的`operator[]`调用它，但是这样还是重复了一些代码，例如：函数调用、两次`return`语句等等。

所以你真正应该做的是实现`operator[]`的机能一次，并且使用它两次。    
也就是说，你必须令其中一个调用另一个。这促使我们将常量性转除（casting away constness）。

解决如下：

```cpp
class TextBlock{
public:
    const char& operator[](std::size_t position) const //一如既往
    {
        ... //边界检验
        ... //标记数据访问
        ... //检验数据完整性
        return text[position];
    }
    
    char& operator[](std::size_t position) //现在只能调用 const op[]
    {
        return 
            const_cast<char&>( //将op[]返回值的 const 转除
                static_cast<const TextBlock&>(*this) //为 *this 加上 const
                    [position] //调用 const op[]
            ); 
    }
};
```

更值得了解的是，反向做法：令`const`版本调用`non-const`版本以避免重复，这个是错误的行为。

记住，`const`成员函数承诺绝不改变其对象的逻辑状态（logical state），`non-const`成员函数却没有这样的承诺。   
如果在`const`函数内调用`non-const`函数，就是冒了这样的风险：你曾经承诺不改动的那个对象被改动了。

因此，当`const`和`non-const`成员函数有着实质等价的实现时，令`non-const`版本调用`const`版本可避免代码重复。


## 确定对象被使用前已先被初始化

最佳处理办法是：永远在使用对象之前先将它初始化。

对于无任何成员的内置类型，你必须手工完成此事。   
例如：

```cpp
int x = 0; //手工初始化
const char* text = "hello world"; //手工初始化

double d;
std::cin >> d; //以读取 input stream 的方式完成初始化
```

至于内置类型以外的任何其他东西，初始化责任落在构造函数身上。   
规则很简单：确保每一个构造函数都把对象的每一个成员初始化。

这个规则很容易奉行，**重要的是别混淆了赋值（assignment）和初始化（initialization）**。  
考虑一个例子：

```cpp
class PhoneNumber{...};

class ABEntry{
public:
    ABEntry(const string& name, const list<PhoneNumber>& phones);
private:
    string theName;
    list<PhoneNumber> thePhones;
    int num;
};
ABEntry::ABEntry(const string& name, const list& phones)
{
    //以下这些都是赋值，不是初始化
    theName = name;
    thePhones = phones;
    num = 0;
}
```

C++ 规定：对象的成员变量的初始化动作发生在进入构造函数本体之前。

- 非内置类型的成员（例如`theName`、`thePhones`）初始化的时间发生于这些成员的`default`构造函数被自动调用之时。

- 内置类型的成员（例如`num`）不保证一定在你所看到的那个赋值动作`num = 0;`的时间点之前获得初值。

**因此，构造函数的一个较佳的写法是：使用成员初值列（member initialization list）替换赋值动作**。   
例如：

```cpp
ABEntry::ABEntry(const string& name, const list<PhoneNumber>& phones)
    :theName(name),              //现在，这些都是初始化
     thePhones(phones),
     num(0)
{} //现在，构造函数本体不必有任何动作
```

这个构造函数和上一个的最终结果相同，但效率通常较高。

基于赋值的那个版本首先调用`default`构造函数为`theName`、`thePhones`设初值，然后再对她们赋予新值。因此`default`构造函数的一切作为浪费了。

而成员初值列（member initialization list）的做法则避免了这一个问题，因为初值列中针对各个成员变量而设的实参会被拿去作为各成员变量之构造函数的实参。   
本例中的`theName`以`name`为初值进行`copy`构造，`thePhones`以`phones`为初值进行`copy`构造。

### 内置类型和非内置类型的初始化与赋值效率比较

- 对于大多数类型而言，比起先调用`default`构造函数然后再调用`copy assignment`操作符，单只调用一次`copy`构造函数是比较高效的。

- 对于内置类型，其初始化和赋值的成本相同，但为了一致性，最好也是通过成员初始值列来初始化。

同样道理，当你想要`default`构造一个成员变量，你都可以使用成员初值列，例如：

```cpp
ABEntry::ABEntry()
    :theName(), //调用 theName 的 default 构造函数
     thePhones(), //调用 thePhones 的 default 构造函数
     num(0) //记得把 num 显式初始化为0
{}
```

- 如果成员变量是 const 或 reference，这种成员变量一定需要初值，不能被赋值。

为了避免需要记住成员变量核实必须在成员初值列中初始化，何时不需要，最简单的做法就是：总是使用成员初值列。

### 在初值列中遗漏那些“赋值表现像初始化一样好”的成员变量

许多 classes 拥有多个构造函数，每个构造函数都有自己的成员初值列，如果这种 classes 存在许多成员变量和/或 base classes，多份成员初值列的存在就会导致不受欢迎的重复（在初值列内）和无聊的工作。    
这种情况下可以合理的在初值列中遗漏那些“赋值表现像初始化一样好”的成员变量，改用它们的赋值操作，并将那些赋值操作移往某个函数（通常是`private`），供所有构造函数调用。   
这种做法在“成员变量的初值是由文件或数据库读入”时特别有用。

### 成员初始化次序

- `base classes`更早于其`derived classes`被初始化     

- 而`class`的**成员变量总是以其声明次序被初始化**


 还是之前的例子：

```cpp
class PhoneNumber{...};

class ABEntry{
public:
    ABEntry(const string& name, const list<PhoneNumber>& phones);
private:
    string theName;
    list<PhoneNumber> thePhones;
    int num;
};

ABEntry::ABEntry(const string& name, const list<PhoneNumber>& phones)
    :theName(name),              //现在，这些都是初始化
     thePhones(phones),
     num(0)
{} //现在，构造函数本体不必有任何动作
```

成员初始化的顺序：

1. `theName`
2. `thePhones`
3. `num`

即使它们在成员初值列中以不同的次序出现（很不幸那是合法的），也不会有任何影响。    
这样做有可能会造成一些晦涩错误：两个成员变量的初始化带有次序性。例如初始化`array`时需要指定大小，因此代表大小的那个成员变量必须先有初值。

所以为了避免迷惑，并且避免上述所说的一些晦涩错误，当你在成员初值列中条列各个成员时候，最好总是以其声明次序为次序。


### 定义于不同编译单元内的non-local static 对象的初始化次序

#### static 对象和编译单元（translation unit）

所谓`static`对象，其寿命从被构造出来知道程序结束为止，因此`stack`和`heap-based`对象都被排除。   
这种对象包括`global`对象、定义于`namespace`作用域内的对象、在`classes`内、在函数内、以及在`file`作用域内被声明为`static`的对象。    

- 其中函数内的`static`对象称为`local static`对象（因为它们对函数而言是`local`）

- 其他`static`对象称为`non-local static`对象

程序结束时`static`对象会被自动销毁，也就是它们的析构函数会在`main()`结束时被自动调用。

编译单元（translation unit）指产出单一目标文件（single object file）的那些源码。    
基本上它是单一源码文件加上其所含入的头文件（`#include` files）。

#### 初始化次序问题

现在，我们关心的问题涉及至少两个源码文件，每一个内含至少一个`non-local static`对象（也就是说该对象是`global`或位于`namespace`作用域内，抑或在`class`内或`file`作用域内被声明为`static`）。   

真正的问题是：如果某编译单元内的某个`non-local static`对象的初始化动作使用了另一个编译单元内的某个`non-local static`对象，它所用到的这个对象可能尚未被初始化，因为 C++ 对**“定义于不同编译单元内的`non-local static`对象”的初始化次序并无明确定义**。


例子：假设你有一个`FileSystem class`，你可能会产出一个特殊对象，位于`global`或`namespace`作用域内，象征着单一文件系统。

```cpp
class FileSystem{
public:
    ...
    std::size_t numDisks() const; //众多成员函数之一
};
extern FileSystem tfs; //预备给客户使用的对象
```

现在假设客户建立了一个`class`用以处理文件系统内的目录（directories），很自然他们的`class`会用上`theFileSystem`对象：

```cpp
class Directory{
public:
    Directory();
};
Directory::Directory()
{
    ...
    std::size_t disks = tfs.numDisks(); //使用 tfs 对象
}
```

进一步假设，这些客户决定创建一个`Directory`对象，用来放置临时文件：

```cpp
Directory tempDir(); //为临时文件而做出的目录,注意前提这个是一个 non-local static 变量
```

现在，初始化次序的重要性显现出来了：    
除非`tfs`在`tempDir`之前先被初始化，否则`tempDir`的构造函数会用到尚未初始化的`tfs`。    
但`tfs`和`tempDir`是不同的人在不同的时间于不同的源码文件建立起来的，它们是定义于不同编译单元内的`non-local static`对象，如何能够确定`tfs`会在`tempDir`之前先被初始化？    

#### 解决方案

无法确定，C++对这个情况的初始化顺序也无明确定义。但是我们可以通过一些设计来解决这个问题：

1. 将每个`non-local static`对象搬到自己的专属函数内（该对象在此函数内被声明为`static`）。    

2. 这些函数返回一个`reference`指向它所含的对象。   

3. 然后对象调用这些函数，而不是直接指涉这些对象。

换句话说，`non-local static`对象被`local static`对象替换了。    
同时这也是单例模式（Singleton）的一个常见实现手法。

这个手法的基础在于：   
C++保证，函数内的`local static`对象会在“该函数被调用期间”“首次遇上该对象的定义式”时被初始化。   

所以如果你采取这种方式，就可以保证你所获得的那个`reference`将指向一个历经初始化的对象。更棒的是，如果你从未调用过`non-local static`对象的“仿真函数”，就绝对不会引发构造和析构成本，真正的`non-local static`对象可没这等便宜！

例如：

```cpp
class FileSystem{...}; //同前

FileSystem& tfs() //这个函数用来替换 tfs 对象
{
    static FileSystem fs;
    return fs;
}

class Directory{...}; //同前

Directory::Directory()
{
    std::size_t disks = tfs().numDisks();
}

Directory& tempDir() //这个函数用来替换 tempDir 对象
{
    static Directory td;
    return td;
}
```

这种结构下的`reference-returning`函数往往十分单纯：   

1. 第一行定义并且初始化一个`local static`对象

2. 第二行返回它

这样的单纯性使它们成为绝佳的`inlining`候选人，尤其如果它们被频繁调用的话。    

但是从另外一个角度看，这些函数“内含`static`对象”的事实使它们在多线程系统中带有不确定性。   
任何一种`non-const static`对象，无论它是`local`或是`non-local`，在多线程环境下“等待某事发生”都会有麻烦，处理这个麻烦的一种做法是：    

在程序的单线程启动阶段（single-threaded startup portion）手工调用所有`reference-returning`函数，这可消除与初始化有关的”竞速形势（race conditions）“。


#### 小结

- 为内置型对象进行手工初始化，因为 C++ 不保证初始化它们

- 构造函数最好使用成员初值列（member initialization list），而不要在构造函数本体内使用赋值操作。初值列列出的成员变量，其排列次序应该和它们在`class`中的声明次序相同。

- 为免除”跨编译单元之初始化次序“问题，请以`local static`对象替换`non-local static`对象。













