---
layout: post
title: "cpp complete guide 13"
subtitle: "cpp complete guide直译"
excerpt: "&ensp;&ensp;&ensp;&ensp;扩展使用声明(using)"
date: 2020-03-14 +08:00
categories: [cpp complete guide]
author: "kcwl"
comments: true
---

# Chapter 13
## 扩展使用声明

使用声明进行了扩展，允许使用逗号分隔声明列表，允许在包扩展中使用它们。例如：

	```
	class Base 
	{
	public:
		void a();
		void b();
		void c();
	};
	class Derived : private Base 
	{
	public:
		using Base::a, Base::b, Base::c;
	};
	```

在c++17之前，需要定义三份声明。


### 13.1 可变参数使用声明
使用声明分隔的逗号使我们能够从基类的可变参数列表中派生出相同类型的所有操作。这种技术的一个很酷的应用是创建和设置lambda重载。通过定义以下几点:

	```
	template<typename... Ts>
	struct overload : Ts...
	{
		using Ts::operator()...;
	};

	// base types are deduced from passed arguments:
	template<typename... Ts>
	overload(Ts...) -> overload<Ts...>;
	```

你可以像下面一样重载2个lambda表达式：

	```
	auto twice = overload {
		[](std::string& s) { s += s; },
		[](auto& v) { v *= 2; }
	};
	```

我们创建了一个overload对象，使用了模版推导，推导了lambdas的类型，并且用聚合初始化初始化了每个lambda的子对象。然后，using声明使两个函数调用操作符都可用于类型重载。如果没有using声明，基类将有相同成员函数操作符()的两个不同的重载，这是不明确的。
现在你可以向第一个重载函数传递一个支持`+=`操作的值，向第二个重载传递一个支持`*=`的值：

	```
	int i = 42;
	twice(i);
	std::cout << "i: " << i << ’\n’; // prints: 84
	std::string s = "hi";
	twice(s);
	std::cout << "s: " << s << ’\n’; // prints: hihi
	```

`std::variant visitors`就是基于这个技术实现的。


### 13.2 变参使用基类构造函数
再加上一些关于继承构造函数的说明，下面的内容现在也是可能的:您可以声明一个可变参数类模板Multi，它派生自一个基类的每个传递的类型:

	```
	template<typename T>
	class Base
	{
		T value{};
	public:
		Base() 
		{
			...
		}
		Base(T v) : value{v} 
		{
			...
		}
	...
	};
	template<typename... Types>
	class Multi : private Base<Types>...
	{
	public:
		// derive all constructors:
		using Base<Types>::Base...;
		...
	};
	```

使用所有基类构造函数的using声明，可以为每种类型派生相应的构造函数。
现在我们声明3种类型构造：

	```
	using MultiISB = Multi<int,std::string,bool>;
	```

你可以使用对应的构造函数初始化对象：

	```
	MultiISB m1 = 42;
	MultiISB m2 = std::string("hello");
	MultiISB m3 = true;
	```

根据新的语言规则，每个初始化调用对应的构造函数进行匹配基类和所有其他基类的默认构造函数。因此:

	```
	MultiISB m2 = std::string("hello");
	```
他们都调用了对应的(`Base<int>`,`Base<std::string>`,`Base<bool>`)基类构造函数。
理论上，你也可以明确使用基类的赋值操作符`=`:

	```
	template<typename... Types>
	class Multi : private Base<Types>...
	{
		...
		using Base<Types>::operator=...;
	};
	```

### 13.3 后记
Robert Haberlach在[https://wg21.link/p0195r0][1]中提出使用声明以逗号分隔。最终由Robert Haberlach和Richard Smith在[https://wg21.link/p0195r2][2]中接受。

各种问题要求明确继承构造函数。最终是由Richard Smith在[https://wg21.link/n4429][3]中接受。

Vicente J. Botet Escriba建议在重载中添加一个通用的重载函数也是普通函数和成员函数。然而，这篇论文并没有被写进c++ 17。见[https://wg21.link/p0051r1][4]细节。


[1]:[https://wg21.link/p0195r0]
[2]:[https://wg21.link/p0195r2]
[3]:[https://wg21.link/n4429]
[4]:[https://wg21.link/p0051r1]

