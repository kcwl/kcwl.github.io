---
layout: post
title: "cpp complete guide 5"
subtitle: "cpp complete guide直译"
excerpt: "&ensp;&ensp;&ensp;&ensp;临时变量强制赋值策略"
date: 2020-03-14 +08:00
categories: [cpp complete guide]
author: "kcwl"
comments: true
---

# Chapter 5
## 未初始化的值传递或者是临时变量的强制复制策略


本节主要有2个观点：
+ c++17引入了 在特定环境下避免拷贝副本的规则，在以前，这条规则是可选的。强制复制策略就是基于这一点。
+ 在特定情况下，我们不需要拷贝。甚至我们想要拷贝的对象还没有被实例化。所以，我们讨论的是没有被赋值的值传递(包括参数和返回值)。

以下将是此特性的历史演变，稍后介绍`materialization`。

### 5.1 临时变量强制赋值策略的动机
c++允许某些复制操作被省略，虽然这有可能导致不调用复制构造函数产生副作用影响程序。尤其是在当通过值向函数传递，或者是函数返回值，在临时变量被一个新对象初始化的时候，会发生这种情况。例如：

	```
	class MyClass
	{
		...
	};

	void foo(MyClass param)	// param is initialized by passed argument
	{

	}

	MyClass bar() 
	{
		return MyClass(); // returns temporary
	}
	int main()
	{
		foo(MyClass()); // pass temporary to initialize param
		MyClass x = bar(); // use returned temporary to initialize x
		foo(bar()); // use returned temporary to initialize param
	}
	```
由于这些优化不是必须的，所有拷贝操作需要提供显示或隐式的拷贝或者移动构造函数。尽管拷贝和移动构造没有被调用，但是也需要存在。下面的没有拷贝或者移动构造的例子，就不能通过编译。例如：

	```
	class MyClass
	{
	public:
		...
		// no copy/move constructor defined:
		MyClass(const MyClass&) = delete;
		MyClass(MyClass&&) = delete;
		...
	};
	```

因为移动构造是隐式提供的，所以拷贝构造(赋值操作符或者析构函数)可以不必声明。从c++17开始，从临时对象中初始化对象变成强制性。事实上，稍后我们就可以看到，我们至少把初始化的值或者返回值用来初始化新的对象。这意味着，即使类MyClass的定义完全不支持赋值，这个例子也一样在编译。其他可选的复制策略同样提供，不过需要提供拷贝或者移动构造函数。例如：

	```
	MyClass foo()
	{
		MyClass obj;
		...
		return obj; // still requires copy/move support
	}
	```

在`foo()`里面，obj对象是个左值。所以会遵循`the named return value optimization(NRVO)`原则，需要提供拷贝或移动构造函数。obj如果是参数也是同样的道理：

	```
	MyClass bar(MyClass obj) // copy elision for passed temporaries
	{
		...
		return obj; // still requires copy/move support
	}
	```

当临时对象传递给函数的时候，不再需要拷贝或者移动构造函数，作为返回值的时候需要提供，因为返回值是个左值。作为这种变化的一部分，在价值术语上做了一些修改和澄清

分类(见第42页第5.3节)。

### 5.2 临时变量强制赋值的好处
当然，这个特性的好处就是在返回值的时候提供更好的效率。尽管移动构造可以大大降低拷贝的花销，这也可以做为一个关键点。这可能会减少外部参数的使用，而不是简单的返回一个值(如果返回值是由`return`创建的)。另一个好处是现在可以定义一个始终工作的工厂函数，即使不允许复制和移动，现在也可以返回一个对象。例如：

	```
	#include <utility>
	template <typename T, typename... Args>
	T create(Args&&... args)
	{
		...
		return T{std::forward<Args>(args)...};
	}
	```

这个方法也可以用作`std::atimic<>`，其中没有定义拷贝构造函数，也没有定义移动构造函数:

	```
	#include "factory.hpp"
	#include <memory>
	#include <atomic>

	int main()
	{
		int i = create<int>(42);
		std::unique_ptr<int> up = create<std::unique_ptr<int>>(new int{42});
		std::atomic<int> ai = create<std::atomic<int>>(42);
	}
	```

另外一个影响，对于显示删除的移动构造函数，可以返回一个临时变量来初始化对象：

	```
	class CopyOnly {
	public:
		CopyOnly() {}

		CopyOnly(int) {}
		CopyOnly(const CopyOnly&) = default;
		CopyOnly(CopyOnly&&) = delete; // explicitly deleted
	};

	CopyOnly ret() 
	{
		return CopyOnly{}; // OK since C++17
	}
	CopyOnly x = 42; // OK since C++17
	```

在c++17之前，禁止这种初始化方式。因为拷贝赋值需要讲`42`转换成临时变量，然后在原则上需要提供移动构造函数，虽然它从来没有被调用(事实上，当用户没有声明移动构造函数的时候，才会调用拷贝构造函数)。

### 5.3 分清值别类型
作为在初始化新对象时要求临时人员进行复制删除的提议的副作用，对值类别进行了一些调整。

#### 5.3.1 值别类型
c++程序中的每个表达式都有一个值类别。这个类别特别描述了什么可以用一个表达式来表示。

**值别的历史**  
沿用于C，c++只有基于赋值的lvalue和rvalue:
	
	```
	x = 42;
	```

这里的`x`是个左值，因为它可以出现在赋值表达式的左边，`42`视为右值，因为它只能出现在表达式的右边。但是有了`ANSI-C`，事情变的更复杂，因为x被声明为了`const int`，不能出现在表达式的左边，但它仍然是lvalue。
在c++11中，我们有了移动对象,移动语义也只能出现在表达式的右边，但是却可以改变它，因为赋值表达式可以窃取它的值。因此，引入了类别xvalue，而以前的类别rvalue得到了一个新名称
prvalue。

**c++11之后的值别**  
c++11之后，值别如图5.1所示：c++有了核心值别***lvalue***,***prvalue("pure rvalue")***,和***xvalue(expiring value)***。复合类是***glvalue(genralized lvalue)***,意味着`lvalue`和`xvalue`的集合,***rvalue***(xvalue和prvalue的集合)。


								expression 
							  \swarrow  \searrow
							glvalue           rvalue
						  \swarrow \searrow \swarrow  \searrow
						lvalue          xvalue  			prvalue

`lvalue`的例子有：
+ 变量，方法，成员的名字
+ 字符常量
+ 内置操作符`*`的结果
+ lvalue引用的返回值(type&)

`prvalues`的例子有：
+ 由非字符串文字(或用户定义的文字，其中相关文字操作符的返回类型定义了类别)
+ 内置运算符`&`的结果(即，取一个表达式的地址)
+ 算数运算符的结果
+ 方法的返回值
+ lambda表达式

`xvalue`的例子有：
+ 右值引用返回值(type&&,特别是std::move())
+ 对象的右值引用转换

粗略的讲：
+ 所有的变量都可以是`lvalue`
+ 所有的字符串都是`lvalue`
+ 其他的常量(4.2 true 或者nullptr)都是prvalue
+ 所有的临时变量(特别是返回值对象)都是prvalue
+ `std::move()`的结果是个`xvalue`

例如：

	```
	class X {};

	X v;
	const X c;
	void f(const X&); // accepts an expression of any value category
	void f(X&&); // accepts prvalues and xvalues only, but is a better match
	f(v); // passes a modifiable lvalue to the first f()
	f(c); // passes a non-modifiable lvalue to the first f()
	f(X()); // passes a prvalue to the second f()
	f(std::move(v)); // passes an xvalue to the second f()
	```

有一点很重要,我们说的`glvalue`,`prvalue`,`xvalue`都是表达式术语，不是值术语。例如：变量本身不是`lvalue`;只有表达式变量的表达式才可以称为lvalue:

	```
	int x = 3; // x here is a variable, not an lvalue
	int y = x; // x here is an lvalue
	```

在第一个语句中，`3`是初始化变量`x`的`prvalue`，第二句`x`是lvalue(它的值指定一个表达式包含`3`的对象)；左值转换成`prvalue`，初始化变量`y`。

### 5.3.2 c++17之后的值别
c++17 没有改变这些值别，但是明确了这些值别的含义。
我们有两种方式解释值别的含义：
+ `glvalues`: 对象或函数位置的表达式
+ `prvalues`: 初始化表达式

`xvalue`通常在一个特殊的位置，它表示一个可以重用其资源的对象(通常是因为它的生命周期即将结束)。
c++17 引入了一个新的单元，临时变量初始化的那一刻把它变成`prvalue`。`temporary materialization conversion`是`prvalue`到`xvalue`的转换。在需要glvalue (lvalue或xvalue)时，prvalue有效地出现，这是临时的对象被创建并使用prvalue初始化(回想一下，prvalue主要是“初始化值”)，而prvalue被指定临时值的xvalue代替。在上面的例子中，严格地说，我们有:

	```
	void f(const X& p); // accepts an expression of any value category,
	// but expects a glvalue
	f(X()); // passes a prvalue materialized as xvalue
	```

因为例子中的`f()`有一个引用参数，所以它希望有一个`glvalue`参数。然而，这个`X()`是一个`prvalue`。"temporary materializetion"规则开始起作用，它将表达式`X()`转换为一个`xvalue`,它指定了一个默认构造函数初始化的临时对象。
实例化并不意味着，我们创建了一个新的，不同的对象。左值引用`p`仍然绑定了`xvalue`和`prvalue`,尽管我们稍后还是会把它转换为`xvalue`。
通过这个修改(prvalues不再是对象，而是可以的表达式(用于初始化对象)，所需的复制省略非常合理，因为prvalues no若要使用赋值语法初始化变量，则需要更长的可移动时间。我们只经过一个周围的初始值迟早会物化来初始化一个对象。


### 5.4 Unmaterialized Return Value Passing
非实例化返回值传递适用于通过值返回临时对象(prvalue)的所有形式:
+ 返回不是字符串的常量值
	```
	int f1() // return int by value
	{ 
		return 42;
	}
	```

+ 通过构造或者auto返回的类类型

	```
	auto f2() // return deduced type by value
	{ 
		...
		return MyType{...};
	}
	```

+ 通过decltype(auto)返回临时对象

	```
	decltype(auto) f3() // return temporary from return statement by value
	{ 
		...
		return MyType{...};
	}
	```

	请记住，如果用于的表达式使用decltype(auto)，则声明将按值进行操作初始化(这里是return语句)是一个创建临时(prvalue)的表达式。
因为我们在所有这些情况下都通过值返回一个prvalue，所以我们根本不需要任何的拷贝/移动支持


### 5.5 后记
临时人员初始化的强制副本省略最初是由Richard Smith提出的，在https://wg21.link/p0135r0。最终被接受的措辞也是由理查德制定的史密斯在https://wg21.link/p0135r1。


[1]:[https://wg21.link/p0135r0]
[2]:[https://wg21.link/p0135r1]