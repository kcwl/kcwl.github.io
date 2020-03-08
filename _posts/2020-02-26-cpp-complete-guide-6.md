---
layout: post
title: "cpp complete guide 6"
subtitle: "cpp complete guide直译"
excerpt: "&ensp;&ensp;&ensp;&ensp;c++11和c++14引入的lambda表达式非常成功。这允许我们将方法指定为参数，这在我们需要指定具体行为的时候非常有用。"
date: 2020-03-07  +08:00
categories: [cpp complete guide]
author: "kcwl"
comments: true
---

# Chapter 6
## Lambda表达式

c++11和c++14引入的lambda表达式非常成功。这允许我们将方法指定为参数，这在我们需要指定具体行为的时候非常有用。
c++17提升了lambda的能力，使得更多的地方可以使用它：
+ 在常量表达式中使用(例如:在编译期)
+ 在需要拷贝一个当前对象的时(例如：线程中的lambdas)



### 6.1 constexpr Lambdas
从c++17开始，lambda在constexpr中隐式调用。意味着任何一个lambda表达式(符合编译期运算的)都可以用在编译期。(例如：字面类型，非静态变量，非虚函数，没有try/catch,没有new/delete)。
例如：使用lambda表达式计算值平方，传递给`std::array<>`作为编译期常量，声明大小：

	```
	auto squared = [](auto vale)		//implictly constexpr since c++17
	{
		return val*val;
	};

	std::array<int,squared(5)> a;		//OK since c++17 => std::array<int,25>
	```

在constexpr表达式中使用非const特性，会使lambda丧失编译期的特性，但依然可以在运行期使用：

	```
	auto squared2 = [](auto val)		//implicitly constexpr since c++17
	{
		static int calls = 0;			//OK，but disables lambda for constepxr contexts
		...
		return val*val;
	};
	std::array<int,squared2(5)> a;		//ERROR:static variable in compile-time context
	std::cout << squared2(5) << '\n';   //OK
 	```

要想查明lambda表达式是否在编译期有效，可以加上constexpr限定符：

	```
	auto squared3 = [](auto val) constexpr //OK since c++17
	{
		return val*val;
	};
	```

定义返回值：

	```
	auto squared3i = [](int val) constexpr -> int		//OK since c++17
	{
		return val*val;
	}
	```

通常，对于lambda表达的规则来说：如果lambda表达式使用在运行期，它就会表现成运行期相对应的方式。如果使用在编译期，但是存在非编译期的特性，就会产生一个编译错误：

	```
	auto squaried4 = [](auto val) constexpr 
	{
		static int calls = 0;				//	ERROR:static variable in compile-time context
		...
		return val*val;
	}
	```

constexpr lambda 不管是显式还是隐式调用，都是constexpr表达式。以下定义：

	```
	auto squared = [](auto val)  // implicitly constexpr since C++17
	{ 
		return val*val;
	};
	```

转换成closure type:

	```
	class CompilerSpecificName {
	public:
		...
		template<typename T>
		constexpr auto operator() (T val) const 
		{
			return val*val;
		}
	};
	```

值得注意的是：closure type 自动声明成constexpr，c++17以后，不管是显式还是隐式调用，都会被声明为constexpr。

### 6.2 传递`this`的副本给lambda
在成员方法中，你没有隐式调用对象成员方法的权利。在lambda里面，如果没有捕获`this`指针,不可以使用成员或者对象(独立于是否通过->调用成员)

	```
	class C {
	private:
		std::string name;
	public:
		...
		void foo() 
		{
			auto l1 = [] { std::cout << name << ’\n’; }; // ERROR
			auto l2 = [] { std::cout << this->name << ’\n’; }; // ERROR
			...
		}
	};
	```

在c++11和c++14中，必须要传递`this`或者'='值捕获或者'&'引用捕获：

	```
	class C {
	private:
		std::string name;
	public:
		...
		void foo()
		{
			auto l1 = [this] { std::cout << name << ’\n’; }; // OK
			auto l2 = [=] { std::cout << name << ’\n’; }; // OK
			auto l3 = [&] { std::cout << name << ’\n’; }; // OK
			...
		}
	};
	```

这里面就存在一个问题，即使复制`this`也会捕获底层对象(仅仅是指针被复制)。如果lambda的生命周期超过调用成员函数的周期，就会产生问题。一个重要的例子，当一个lambda定义了一个新线程的任务，该对象应该使用自己对象的副本来避免任何的并发和生存周期问题。可能会传递当前对象副本和状态。

从c++14开始就有了一种变通的方法，但是既不好看，也不好用：

	```
	class C {
	private:
		std::string name;
	public:
		...
		void foo() {
			auto l1 = [thisCopy=*this] 
			{ 
				std::cout << thisCopy.name << ’\n’; 
			};
			...
		}
	};
	```

例如：程序不小心使用了`this`，可以使用`=`或者`&`捕获其他对象：
	
	```
	auto l1 = [&, thisCopy=*this] 
	{
		thisCopy.name = "new name";
		std::cout << name << ’\n’; // OOPS: still the old name
	};
	```
从c++17开始，你可以明确要求捕获`*this`:

	```
	class C {
	private:
		std::string name;
	public:
		...
		void foo() 
		{
			auto l1 = [*this] { std::cout << name << ’\n’; };
			...
		}
	};
	```

这里，`*this`就表示1以值传递的方式捕获`this`指针的副本。在不存在矛盾的情况下，也可以把`*this`和其他捕获变量结合使用:

	```
	auto l2 = [&, *this] { ... }; // OK
	auto l3 = [this, *this] { ... }; // ERROR
	```

下面是个完整的例子：

	```
	#include <iostream>
	#include <string>
	#include <thread>
	class Data {
	private:
		std::string name;
	public:
		Data(const std::string& s) : name(s) {}

		auto startThreadWithCopyOfThis() const 
		{
			// start and return new thread using this after 3 seconds:
			using namespace std::literals;

			std::thread t([*this] 
			{
				std::this_thread::sleep_for(3s);
				std::cout << name << ’\n’;
			});
			return t;
		}
	};
	int main()
	{
		std::thread t;
		{
			Data d{"c1"};
			t = d.startThreadWithCopyOfThis();
		} // d is no longer valid
		t.join();
	}
	```
lambda接受`*this`的副本，这意味着传递了d的副本。调用d的析构函数后，线程使用传递的对象就没有问题了。

如果我们还是通过[this],[=],[&]，线程就会产生未定义行为，因为我们打印的name调用了一个已经销毁的对象。

### 6.3 后记
constexpr lambdas 由Faisal Vali,Ville Voutilainen 和Gabriel Dos Reis 在[https://wg21.link/n4487][1]中体术，最终在[https://wg21.link/p0170r1][2]上被Faisal Vali,Jens Maurer和Richard Smith 采纳。

`*this`捕获是由 H. Carter Edwards, Christian Trott, Hal Finkel Jim Reus, Robin Maffeo, 和 Ben Sander在[https://wg21.link/p0018r0][3]提出，最终被H. Carter Edwards, Daveed Vandevoorde, Christian Trott, Hal Finkel,Jim Reus, Robin Maffeo, 和 Ben Sander 在[https://wg21.link/p0180r3][4]中采纳。


[1]:[https://wg21.link/n4487]
[2]:[https://wg21.link/p0170r1]
[3]:[https://wg21.link/p0018r0]
[4]:[https://wg21.link/p0180r3]