---
layout: post
title: "cpp complete guide 7"
subtitle: "cpp complete guide直译"
excerpt: "&ensp;&ensp;&ensp;&ensp;其他语言特性"
date: 2020-03-14 +08:00
categories: [cpp complete guide]
author: "kcwl"
comments: true
---

# Chapter 7
## 其他语言特性

c++还有一些小的改动，将在本文中介绍


### 7.1 嵌套命名空间
2003年，c++标准委员会接受了嵌套命名空间：

	```
	namespace A::B::C
	{
		...
	}
	```

等价于：

	```
	namespace A 
	{
		namespace B 
		{
			namespace C 
			{
				...
			}
		}
	}
	```
c++17将不再支持嵌套内联空间。这只是因为内联应用于最后一个名称空间还是应用于所有名称空间并不明显(两者都同样有用)。

### 7.2 定义表达式求值顺序
很多书上的代码看起来都是有效的，但是严格来说，其中有可能有未定义行为。下面一个查找并替换子串的例子：

	```
	std::string s = "I heard it even works if you don’t believe";
	s.replace(0,8,"").replace(s.find("even"),4,"sometimes").replace(s.find("you don’t"),9,"I");
	```

通常来说，这个代码可以替换前8个字符不会出问题，`even`用`sometimes`替换，`you don't`用`I`替换，最后得到了：

	```
	it sometimes works if I believe
	```

但是，在c++ 17之前，这个结果是没有保证的，因为find()调用了返回的位置要从替换开始，可以在处理整个语句期间的任何时候以及需要它们的结果之前执行。实际上，所有find()调用(计算替换的起始索引)都可能在任何替换发生之前进行处理，从而使结果字符串变为:

	```
	it sometimes works if I believe
	```

但是也有可能会变成其他结果：

	```
	it sometimes workIdon’t believe
	it even worsometiIdon’t believe
	it even worsometimesf youIlieve
	```

另外一个例子：使用输出操作符连续输出表达式结果

	```
	std::cout << f() << g() << h();
	```

通常假设的调用顺序是`f()`,`g()`,`h()`，然而这种假设是错误的。`f()`,`g()`,`h()`可能会按照任意的顺序被调用。当这些函数相互调用的时候，可能会产生糟糕的影响。

c++17之前，下面的代码具体未定义行为：

	```
	i = 0;
	std::cout << ++i << ’ ’ << --i << ’\n’;
	```

c++17之前，可能会输出1和0，也可能输出0和-1，也可能输出0，0。`i`的类型是什么并不影响(对于基本类型，有些编译器会提出警告)。
为了修正所有这些意料之外的行为，对一些操作符的求值保证进行了改进，他们现在指定了一个执行顺序:
+ For
	
	```
	e1 [ e2 ]
	e1 . e2
	e1 . * e2
	e1 -> * e2
	e1 << e2
	e1 >> e2
	```

现在,以上操作符，`e1`保证在`e2`之前执行。

但是，请注意，相同函数调用的不同参数的求值顺序仍然是未定义的。像：

	```
	e1.f(a1,a2,a3)
	```

现在`e1`可以保证在`a1`,`a2`,`a3`之前执行，但是`a1`,`a2`,`a3`不能保证执行顺序

+ 所有的赋值操作符

	```
	e2 = e1
	e2 += e1
	e2 * = e1
	...
	```

现在，`e1`保证在`e2`之前执行。

+ 最后，在`new`表达式中：

	```
	new Type(e)
	```

现在可以保证分配在求值e和初始化之前执行,新值保证在使用已分配和已初始化的值之前发生。

这些保证适用于所有类型。


因此，c++17之后

	```
	std::string s = "I heard it even works if you don’t believe";
	s.replace(0,8,"").replace(s.find("even"),4,"always")
	.replace(s.find("don’t believe"),13,"use C++17");
	```

保证了求值顺序，结果只有一种：

	```
	it always works if you use C++17
	```

现在，每一个`find()`表达式都是在之前`find()`执行之后执行的。

另外一个例子:

	```
	i = 0;
	std::cout << ++i << ’ ’ << --i << ’\n’;
	```

输出结果保证为1 0，支持任意类型。

然而，对于大多数类型，这种未定义行为还是存在的。例如：

	```
	i = i++ + i; // still undefined behavior
	```

这里，右边的i可能是i在增加之前或之后的值。另一个例子是`new`表达式在传递参数之前插入空格的函数的求值顺序(见101页10.2.1节)。

### 7.3 使用int类型更加灵活的初始化enum
对于具有固定底层类型的枚举，c++ 17可以使用它的整数值用于直接列表初始化的类型。这适用于具有指定类型和所有类型的未限定作用域的枚举作用域枚举，因为它们始终具有底层默认类型。

	```
	// unscoped enum with underlying type:
	enum MyInt : char { };
	MyInt i1{42}; // OK since C++17 (ERROR before C++17)
	MyInt i2 = 42; // still ERROR
	MyInt i3(42); // still ERROR
	MyInt i4 = {42}; // still ERROR
	// scoped enum with default underlying type:
	enum class Salutation { mr, mrs };
	Salutation s1{0}; // OK since C++17 (ERROR before C++17)
	Salutation s2 = 0; // still ERROR
	Salutation s3(0); // still ERROR
	Salutation s4 = {0}; // still ERROR
	```

如果`Salutation`也有明确的类型，那么同样适用:

	```
	// scoped enum with specified underlying type:
	enum class Salutation : char { mr, mrs };
	Salutation s1{0}; // OK since C++17 (ERROR before C++17)
	Salutation s2 = 0; // still ERROR
	Salutation s3(0); // still ERROR
	Salutation s4 = {0}; // still ERROR
	```

对于没有作用域的enum(enum without class)，他们没有明确的默认类型，无法使用此种初始化方式：

	```
	enum Flag { bit1=1, bit2=2, bit3=4 };
	Flag f1{0}; // still ERROR
	```

还要注意，列表初始化仍然不允许缩小，所以您不能传递浮点数值:

	```
	enum MyInt : char { };
	MyInt i5{42.2}; // still ERROR
	```

该特性的目的是支持通过定义新的整型类型的技巧，支持枚举映射到int类型。如果没有这个特性，就无法在不进行强制转换的情况下初始化新对象。

### 7.4 auto声明灵活的初始化列表
在c++ 11中引入了带括号的统一初始化之后，我们可以通过`auto`来声明一些，我们不确定类型的列表：

	```
	int x{42}; // initializes an int
	int y{1,2,3}; // ERROR
	auto a{42}; // initializes a std::initializer_list<int>
	auto b{1,2,3}; // OK: initializes a std::initializer_list<int>
	```

这些不一致被修正为直接的列表初始化(没有=的大括号初始化),我们现在有以下行为:

	```
	int x{42}; // initializes an int
	int y{1,2,3}; // ERROR
	auto a{42}; // initializes an int now
	auto b{1,2,3}; // ERROR now
	```

请注意，这是一个破坏性的更改，甚至可能会在不提示的情况下导致不同的程序行为(例如，当打印a时)。因此，采用这种更改的编译器通常在c++ 11模式中也是如此采用这种更改。对于主要的编译器，Visual Studio 2015, g++ 5, clang 3.8都采用了修复。

还请注意，复制列表初始化(使用=进行大括号初始化)仍然具有初始化行为，当使用`auto`时，总是初始化为`std::initializer_list<>`:

	```
	auto c = {42}; // still initializes a std::initializer_list<int>
	auto d = {1,2,3}; // still OK: initializes a std::initializer_list<int>
	```

现在我们对于直接初始化(没有`=`)和拷贝初始化(有`=`)有两种表现：

	```
	auto a{42}; // initializes an int now
	auto c = {42}; // still initializes a std::initializer_list<int>
	```

初始化变量和对象的推荐方法应该总是使用直接的列表初始化(不带=的大括号初始化)。


### 7.5 十六进制浮点字面量
c++17标准提供了使用十六进制浮点字面量的能力(有些编译器在c++17之前就支持了)。当需要精确的浮点表示法时，这种表示法特别有用(或十进制浮点值。一般不能保证存在精确的值)。例如：

	```
	#include <iostream>
	#include <iomanip>
	int main()
	{
		// init list of floating-point values:
		std::initializer_list<double> values {
		0x1p4, // 16
		0xA, // 10
		0xAp2, // 40
		5e0, // 5
		0x1.4p+2, // 5
		1e5, // 100000
		0x1.86Ap+16, // 100000
		0xC.68p+2, // 49.625
	};
	
	// print all values both as decimal and hexadecimal value:
	for (double d : values) {
	std::cout << "dec: " << std::setw(6) << std::defaultfloat << d
	<< " hex: " << std::hexfloat << d << ’\n’;
	}
	```

该程序通过使用不同的现有表示法和新的表示法来定义不同的浮点值十六进制浮点表示法。新表示法是一种基于`+2`的科学表示法:
+ 有效和/尾数是用十六进制格式写的。
+ 指数用十进制形式表示，并以2为基底进行解释。

例如，0xAp2是指定十进制值40(10乘以2的2次方)的一种方法,值也可以表示为0x1.4p+5，即1.25乘以32(0.4是十六进制四分之一，2的5次方是32)。

下面的程序输出：

	```
	dec: 16 hex: 0x1p+4
	dec: 10 hex: 0x1.4p+3
	dec: 40 hex: 0x1.4p+5
	dec: 5 hex: 0x1.4p+2
	dec: 5 hex: 0x1.4p+2
	dec: 100000 hex: 0x1.86ap+16
	dec: 100000 hex: 0x1.86ap+16
	dec: 49.625 hex: 0x1.8dp+5
	```

可以看到，在c++11中就已经提供了`std::hexfloat`来作为十六进制输出符号。

### 7.6 UTF-8字符
c++11之后，c++支持以`u8`作为前缀声明utf-8字符串，但是此特性只允许字符串使用，c++17弥补了这个缺陷：

	```
	char c = u8’6’; // character 6 with UTF-8 encoding value
	```
这样你就可以保证你的字符值是UTF-8中字符‘6’的值。你可以使用所有7位US-ASCII字符，其UTF-8代码具有相同的值。是,这指定具有7位US-ASCII、ISO Latin-1、ISO-8859-15和the的正确字符值基本的Windows字符集。2通常，你的源代码用US-ASCII/UTF-8来解释字符。不管怎样，前缀不是必须的。c的值几乎总是54(十六进制36)。

这里有些历史背景。在源代码中，字符串和字符的前缀是必要的，c++对可以使用的字符进行标准化，但并不对他们的值进行标准化。他们的值取决于编译器的执行字符集。源字符集几乎都是7位的US-ASCII。所以在任务c++程序中，带和不带前缀的字符具有相同的值。

但在非常罕见的情况下，情况可能并非如此。例如，在旧的IBM主机上，使用EBCDIC字符集，字符“6”的值将改为246(十六进制F6)。因此，在使用EBCDIC字符集的程序中，上述字符c的值应该是
因此，在UTF-8编码平台上运行程序可能会打印字符o，它是ASCII值为246的字符(如果可用)。在有些情况下,这个前缀可能是必要的。

注意，u8只能用于单个字符和具有单个字节(代码)的字符初始化，如:

	```
	char c =u8'ö';
	```

以上操作是不允许的，因为UTF-8中的德语umlaut o的值是两个字节的序列195182(16进制C3 B6)。

+ u8只能作为当字节的US-ASCII和UTF-8编码的前缀
+ u 作为双字节UTF-16的前缀
+ U 作为四字节UTF-32的前缀
+ l 作为宽字节没有明确编码的前缀

### 7.7 异常规范作为类型的一部分
从c++17 开始，异常规范成为了方法的一部分，下面的两个方法的类型并不相同：

	```
	void f1();
	void f2() noexcept; // different type
	```

c++17之前，这2个方法是同样的类型。
所以，现在编译器就会检测你是否使用带有抛出异常的函数：

	```
	void (*fp)() noexcept; // pointer to function that doesn’t throw
	fp = f2; // OK
	fp = f1; // ERROR since C++17
	```
使用没有抛出异常的方法，也可以使用抛出异常的方法进行赋值：

	```
	void (*fp2)(); // pointer to function that might throw
	fp2 = f2; // OK
	fp2 = f1; // OK
	```

所以，新特性并没有破坏以前的程序，现在可以确保不再违反函数指针中的noexcept需求(这可能会破坏现有的程序)。不允许用不同的异常为相同的签名重载函数名规范(因为只允许重载不同返回类型的函数):

	```
	void f3();
	void f3() noexcept; // ERROR
	```

注意，所有其他规则都不受影响。例如，仍然不允许忽略基类的noexcept规范:

	```
	class Base 
	{
	public:
		virtual void foo() noexcept;
		...
	};
	class Derived : public Base 
	{
	public:
		void foo() override; // ERROR: does not override
		...
	};
	```

子类的`foo()`方法和基类的`foo()方法拥有不同的类型，所以不能重写。代码不能通过编译。即使没有`override`，代码也不能通过编译,因为我们不能使用宽松规范来重载。


**使用条件异常规范**  
在使用条件noexcept规范时，函数的类型取决于条件是`true`还是`false`。

	```
	void f1();
	void f2() noexcept;
	void f3() noexcept(sizeof(int)<4); // same type as either f1() or f2()
	void f4() noexcept(sizeof(int)>=4); // different type than f3()
	```

`f3()`的类型取决于条件的结果
+ 如果`sizeof(int)`大于等于4，结果为
	
	```
	void f3() noexcept(false);    //same type as f1()
	```

+ 如果`sizeof(int)`小于4，结果为：

	```
	void f3() noexcept(true); // same type as f2()
	```

因为f4()的异常条件使用了`f3()`的负表达式，所以`f4()`总是有一个不同类型(即。，它保证在`f3()`不抛出时抛出，反之亦然。
“老式的”空抛出规范仍然可以使用，但自从c++ 17以来就被弃用了:

	```
	void f5() throw(); // same as void f5() noexcept but deprecated
	```

动态抛出异常从c++11开始就被弃用了：

	```
	void f6() throw(std::bad_alloc); // ERROR: invalid since C++17
	```

**对通用库的影响**  
将noexcept声明作为类型的一部分可能会对泛型库产生一些影响。例如，下面的程序在c++ 14之前有效，但不再用c++ 17编译:

	```
	#include <iostream>
	template<typename T>
	void call(T op1, T op2)
	{
		op1();
		op2();
	}
	void f1() 
	{
		std::cout << "f1()\n";
	}
	void f2() noexcept 
	{
		std::cout << "f2()\n";
	}
	int main()
	{
		call(f1, f2); // ERROR since C++17
	}
	```

这个问题产生的原因是因为从c++17以后，`f1()`和`f2()`编译器不认为他们是同一个类型，导致编译失败。想要编译成功，那么需要提供两种类型：

	```
	template<typename T1, typename T2>
	void call(T1 op1, T2 op2)
	{
		op1();
		op2();
	}
	```

如果您想要或必须重载所有可能的函数类型，那么您现在还必须加倍重载。例如，这适用于标准类型trait std::is_function<>的定义。

主模板是这样定义的，一般来说，类型T没有函数:

	```
	// primary template (in general type T is no function):
	template<typename T> struct is_function : std::false_type { };
	```

该模板派生自`std::false_type(参考192页第20.3节),一般情况下，`is_function::value`对于任何类型T都产生`false`。
对于所有属于函数的类型，都存在偏特化，它们派生自std::true_type(参见192页20.3节)，以便成员值为true:

	```
	// partial specializations for all function types:
	template<typename Ret, typename... Params>
	struct is_function<Ret (Params...)> : std::true_type { };
	template<typename Ret, typename... Params>
	struct is_function<Ret (Params...) const> : std::true_type { };
	template<typename Ret, typename... Params>
	struct is_function<Ret (Params...) &> : std::true_type { };
	template<typename Ret, typename... Params>
	struct is_function<Ret (Params...) const &> : std::true_type { };
	...
	```

在c++ 17之前，已经有24个偏特化，因为函数类型可以有const和volatile限定符，以及lvalue(&)和rvalue(&&)引用限定符，您需要这些带有可变参数列表的函数的重载。现在，在c++ 17中，通过添加noexcept限定符，部分专门化的数量增加了一倍，现在我们有48个偏特化

	```
	...
	// partial specializations for all function types with noexcept:
	template<typename Ret, typename... Params>
	struct is_function<Ret (Params...) noexcept> : std::true_type { };
	template<typename Ret, typename... Params>
	struct is_function<Ret (Params...) const noexcept> : std::true_type { };
	template<typename Ret, typename... Params>
	struct is_function<Ret (Params...) & noexcept> : std::true_type { };
	template<typename Ret, typename... Params>
	struct is_function<Ret (Params...) const& noexcept> : std::true_type { };
	...
	```

没有实现noexcept重载的库可能不再编译代码，这些代码使用它们将函数或函数指针传递到需要noexcept的地方。


### 7.8 单参数的static_assert
从c++17开始，`static_assert()`需要的消息参数变成可选的。这意味着得到的诊断消息完全是特定于平台的：

	```
	#include <type_traits>
	template<typename T>
	class C 
	{
		// OK since C++11:
		static_assert(std::is_default_constructible<T>::value,"class C: elements must be default-constructible");
		// OK since C++17:
		static_assert(std::is_default_constructible_v<T>);
		...
	};
	```


新的断言表达式也可以使用\_v后缀的type traits。


### 7.9 宏条件  __has_include
c++ 17扩展了预处理器，使其能够检查是否包含特定的头文件。例如：

	```
	#if __has_include(<filesystem>)
	# include <filesystem>
	# define HAS_FILESYSTEM 1
	#elif __has_include(<experimental/filesystem>)
	# include <experimental/filesystem>
	# define HAS_FILESYSTEM 1
	# define FILESYSTEM_IS_EXPERIMENTAL 1
	#elif __has_include("filesystem.hpp")
	# include "filesystem.hpp"
	# define HAS_FILESYSTEM 1
	# define FILESYSTEM_IS_EXPERIMENTAL 1
	#else
	# define HAS_FILESYSTEM 0
	#endif
	```

如果相应的`#include`命令有效，那么`_has_include(…)`中的条件的值为1 (true)。其他都不重要(例如，答案不取决于文件是否已包含)。

### 7.10 后记
嵌套名称空间定义(请参阅第53页第7.1节)最早由Jon Jagger在2003年在[https://wg21.link/n1524][1]上提出。2014年，Robert Kawulak在[https://wg21.linkn4026][2]上提出了一项新提议。最终被由 Robert Kawulak和Andrew Tomazos在[https://wg21.link/n4230]上采纳。

改进的表达式求值顺序(见第54页7.2节)最初是由Gabriel Dos Reis提出的, 最终是由Gabriel Dos Reis, Herb Sutter和Jonathan Caves在[https://wg21.link/p0145r3][4]中接受。

放宽enum初始化(参见第56页7.3节)是由Gabriel Dos Reis首先提出的在[https://wg21.link/p0138r0][5]。最终被接受的措辞是由Gabriel Dos Reis在[https://wg21.link/p0138r2][6]拟定的。

使用auto修复列表初始化(参见第57页的7.4节)最初是由Ville Voutilainen在[https://wg21.link/n3681][7]和[https://wg21.link/3912][8]。列表的最终修复使用auto初始化是由James Dennett在[https://wg21.link/n3681][9]中提出的。

十六进制浮点文字(见第57页7.5节)是由Thomas Koppe在[https://wg21.link/p0245r0][10]首先提出的。最终是由Köppe Koppe在[https://wg21.link/p0245r1][11]拟定的。

UTF-8字符字面量的前缀(见第59页第7.6节)最初是由Richard Smith在[https://wg21.link/n4197][12]提出的。最终由Richard Smith在[https://wg21.link/n4267][13]拟定的。

首先，将异常规范作为函数类型的一部分(参见第60页的7.7节)由Jens Maurer在[https://wg21.link/n4320][14]中提出。最后由Jens Maurer在[https://wg21.link/p0012r1][15]上接受。

Walter E. Brown在[https://wg21.link/n3928][16]中提出接受单参数static_assert(参见第63页第7.8节)。

预处理器子句_has_include()(参见64页第7.9节)首先由以下人提出Clark Nelson和Richard Smith作为[https://wg21.link/p0061r0][17]的一部分。终于接受了措辞由Clark Nelson和Richard Smith在[https://wg21.link/p0061r1][18]中提出。



[1]:[https://wg21.link/n1524]
[2]:[https://wg21.linkn4026]
[3]:[https://wg21.link/n4230]
[4]:[https://wg21.link/p0145r3]
[5]:[https://wg21.link/p0138r0]
[6]:[https://wg21.link/p0138r0]
[7]:[https://wg21.link/n3681]
[8]:[https://wg21.link/3912]
[9]:[https://wg21.link/n3681]
[10]:[https://wg21.link/p0245r0]
[11]:[https://wg21.link/p0245r1]
[12]:[https://wg21.link/n4197]
[13]:[https://wg21.link/n4267]
[14]:[https://wg21.link/n4320]
[15]:[https://wg21.link/p0012r1]
[16]:[https://wg21.link/n3928]
[17]:[https://wg21.link/p0061r0]
[18]:[https://wg21.link/p0061r1]