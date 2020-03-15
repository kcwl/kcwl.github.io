---
layout: post
title: "cpp complete guide 9"
subtitle: "cpp complete guide直译"
excerpt: "&ensp;&ensp;&ensp;&ensp;compile-time if"
date: 2020-03-15 +08:00
categories: [cpp complete guide]
author: "kcwl"
comments: true
---

# Chapter 9
## 编译期if

编译器通过使用`if constexpr(...)`在编译期决定是调用`then`部分还是`else`部分。其他的代码部分被忽略，所有不会产生其他代码。但是并不意味着其他部分的代码是完全忽略丢弃的。它将和未使用的模版一样，进行编译期检查。例如：

	```
	#include <string>
	template <typename T>
	std::string asString(T x)
	{
		if constexpr(std::is_same_v<T, std::string>) 
		{
			return x; // statement invalid, if no conversion to string
		}
		else if constexpr(std::is_arithmetic_v<T>)
		{
			return std::to_string(x); // statement invalid, if x is not numeric
		}
		else 
		{
			return std::string(x); // statement invalid, if no conversion to string
		}
	}
	```

使用了此特性，我们返回一个`std::string`，当参数为int类型或者float类型时候，调用`std::to_string()`，或者直接使用`std::string`构造函数。因为无效的调用被丢弃，下面的代码将被编译(如果使用常规的运行时，则不会出现这种情况)。

	```
	#include "ifcomptime.hpp"
	#include <iostream>
	int main()
	{
		std::cout << asString(42) << ’\n’;
		std::cout << asString(std::string("hello")) << ’\n’;
		std::cout << asString("hello") << ’\n’;
	}
	```

### 9.1 编译期if的动机
如果我们使用以下例子的运行期if：

	```
	#include <string>
	template <typename T>
	std::string asString(T x)
	{
		if (std::is_same_v<T, std::string>) 
		{
			return x; // ERROR, if no conversion to string
		}
		else if (std::is_numeric_v<T>) 
		{
			return std::to_string(x); // ERROR, if x is not numeric
		}
		else 
		{
			return std::string(x); // ERROR, if no conversion to string
		}
	}
	```

这段代码将会编译失败。这是由于模版不会被整体编译的原因造成的。if条件表达式是运行期特性，在编译期，必须要是某个条件为false时，`then`部分才能够编译。因此，当传递std::string或者string literal时，编译会失败，这是因为`std::to_string()`操作的部分没有提供。传递一个数字类型的时候，也会失败，道理同前一个。现在只能使用编译期if，`then`,`else`部分才能不会被丢弃：

+ 当传递std::string值时，第一个if的其他部分被丢弃。
+ 当传递一个数值时，第一个if和最后一个else部分的then部分将被丢弃。
+ 当传递一个字符串文字(即。，键入const char*)，则为第一个if和第二个if的then部分丢弃。

因此，每个无效的组合在编译时都不能再出现，代码编译成功。

注意，丢弃的语句不会被忽略。其结果是，当依赖于模板参数时，它不会被实例化。语法必须正确，不依赖于模板参数的调用必须有效。实际上，执行第一个转换阶段(定义时间)，它检查语法是否正确，以及不依赖于模板参数的所有名称的用法。所有static_assert必须是有效的，即使在未编译的分支中也是如此。例如：

	```
	template<typename T>
	void foo(T t)
	{
		if constexpr(std::is_integral_v<T>) 
		{
			if (t > 0) 
			{
				foo(t-1); // OK
			}
		}
		else 
		{
			undeclared(t); // error if not declared and not discarded (i.e., T is not integral)
			undeclared(); // error if not declared (even if discarded)
			static_assert(false, "no integral"); // always asserts (even if discarded)
		}
	}
	```

这个例子无法通过编译：
+ 尽管T是一个int类型，但是在`else`部分，`undeclared()`没有定义，也会产生编译错误。因为这个函数不依赖模版。
+ 即使它是被丢弃的else部分的一部分，也总是失败，
	
	```
	static_assert(false, "no integral"); // always asserts (even if discarded)
	```

因为这个调用同样不依赖于模板参数。重复编译时条件的静态断言是可以的:

	```
	static_assert(!std::is_integral_v<T>, "no integral");
	```

注意，有些编译器(例如，Visual c++ 2013和2015)不能正确地实现或执行模板的两阶段转换。它们将第一阶段的大部分时间(定义时间)推迟到第二阶段(实例化时间)，因此无效的函数调用甚至一些语法错误都可能发生编译。

### 9.2 使用编译期if
理论上，你可以像运行期if一样，提供给编译期if 条件，也可以把运行期if和编译期if混合用：

	```
	if constexpr (std::is_integral_v<std::remove_reference_t<T>>) {
		if (val > 10) {
			if constexpr (std::numeric_limits<char>::is_signed) 
			{
			...
			}
			else 
			{
				...
			}
		}
		else 
		{
			...
		}
	}
	else 
	{
		...
	}
	```

请注意，不能在函数体外使用if constexpr。因此，你不能用它来代替条件预处理器指令。


#### 9.2.1 编译期if的警告
即使可以使用编译期if，但是也会产生一些不可能的后果。


**编译期if影响返回值类型**  
编译期if可能会影响类型的返回值。例如下面的例子，总是能编译成功，但是返回值可能并不相同：

	```
	auto foo()
	{
		if constexpr (sizeof(int) > 4) 
		{
			return 42;
		}
		else 
		{
			return 42u;
		}
	}
	```

由于我们用了auto，函数会自行推导函数返回值类型，返回值的类型取决于int的大小。
+ 如果size大于4 ，会返回42，是int类型。
+ 如果size小于4，会返回42u，是unsigned int类型。


这样的清空下，就会产生不同的类型,另外一个例子，如果我们跳过`else`部分，就会返回int或者void：

	```
	auto foo() // return type might be int or void
	{
		if constexpr (sizeof(int) > 4) 
		{
			return 42;
		}
	}
	```

请注意，如果这里使用了运行时if，则此代码永远不会编译，因为这两个返回语句都会被考虑在内，因此返回类型的推断是不明确的。

**即使`then`有返回值，`else`也很重要

对于运行时if语句，有一个模式不适用于编译时if语句:编译期if在then和else部分都有返回语句的代码编译后，总是可以在运行时if语句中跳过else语句。
你可以使用:

	```
	if (...) 
	{
		return a;
	}
	else 
	{
		return b;
	}
	```
替代为：

	```
	if (...) 
	{
		return a;
	}
	return b;
	```

此模式不适用于编译时，因为在第二种形式中，返回类型取决于在两个而不是一个返回语句上，这可以产生差异。例如，修改上面的例子导致代码可能编译，也可能编译不了:

	```
	auto foo()
	{
		if constexpr (sizeof(int) > 4) 
		{
			return 42;
		}

		return 42u;
	}
	```

如果结果为true(int的size大于4)，编译器就会推断出2个返回值类型，没有被丢弃的选项。但是我们的目的是返回一个满足条件的类型，所以这个例子是可以编译过的。

**Short-Circuit Compile-Time Conditions**  
如下例子：

	```
	template<typename T>
	constexpr auto foo(const T& val)
	{
		if constexpr (std::is_integral<T>::value) 
		{
			if constexpr (T{} < 10) 
			{
				return val * 2;
			}
		}
		return val;
	}
	```

这里有两个编译期条件，来决定是否是返回一个值，还是两个值。以下两个表达式都可以编译成功：

	```
	constexpr auto x1 = foo(42); // yields 84
	constexpr auto x2 = foo("hi"); // OK, yields ”hi”
	```

运行期if会停止检查判断条件(当使用&&碰到第一个false时，使用 ||碰到第一个true时)，编译期if的表现：

	```
	template<typename T>
	constexpr auto bar(const T& val)
	{
		if constexpr (std::is_integral<T>::value && T{} < 10) 
		{
			return val * 2;
		}

		return val;
	}
	```

但是，如果总是实例化编译时的条件，并且需要作为一个整体有效，这样传递不支持<10的类型就不再编译:

	```
	constexpr auto x2 = bar("hi"); // compile-time ERROR
	```
如果编译时条件的有效性依赖于前面的编译时条件，则必须像在foo()中那样嵌套它们。例如：

	```
	if constexpr (std::is_same_v<MyType, T>) {
		if constexpr (T::i == 42) 
		{
			...
		}
	}
	```

代替：

	```
	if constexpr (std::is_same_v<MyType, T> && T::i == 42) 
	{
		...
	}
	```


#### 9.2.2 其他编译期if示例
**泛型值的完美返回**  

有个例子，在返回值必须要返回的时候，采用`perfect forward`处理返回值。由于`decltype(auto)` 不能推断出`void`类型(因为`void`是个不完整的类型)，所以必须如以下操作：

	```
	#include <functional> // for std::forward()
	#include <type_traits> // for std::is_same<> and std::invoke_result<>
	template<typename Callable, typename... Args>
	decltype(auto) call(Callable op, Args&&... args)
	{
		if constexpr(std::is_void_v<std::invoke_result_t<Callable, Args...>>) 
		{
			// return type is void:
			op(std::forward<Args>(args)...);
			... // do something before we return
			return;
		}
		else {
			// return type is not void:
			decltype(auto) ret{op(std::forward<Args>(args)...)};
			... // do something (with ret) before we return
			return ret;
		}
	}
	```

**标签调度的编译期if** 
标签调度是典型的编译期if的使用场景。在c++17之前，你需要为每一个类型写一个重载函数。现在，可以通过编译期if写在一个函数里面：

	```
	template<typename Iterator, typename Distance>
	void advance(Iterator& pos, Distance n) {
		using cat = std::iterator_traits<Iterator>::iterator_category;
		advanceImpl(pos, n, cat); // tag dispatch over iterator category
	}
	template<typename Iterator, typename Distance>
	void advanceImpl(Iterator& pos, Distance n,std::random_access_iterator_tag)
	{
		pos += n;
	}
	template<typename Iterator, typename Distance>
	void advanceImpl(Iterator& pos, Distance n,std::bidirectional_iterator_tag) 
	{
		if (n >= 0) 
		{
			while (n--) 
			{
			++pos;
			}
		}
		else {
			while (n++) 
			{
				--pos;
			}
		}
	}

	template<typename Iterator, typename Distance>
	void advanceImpl(Iterator& pos, Distance n, std::input_iterator_tag) {
		while (n--) 
		{
			++pos;
		}
	}
	```

现在，我们可以使用一个函数完成这些事情：

	```
	template<typename Iterator, typename Distance>
	void advance(Iterator& pos, Distance n) 
	{
		using cat = std::iterator_traits<Iterator>::iterator_category;
		if constexpr (std::is_same_v<cat, std::random_access_iterator_tag>) {
			pos += n;
		}
		else if constexpr (std::is_same_v<cat,std::bidirectional_access_iterator_tag>) 
		{
			if (n >= 0) {
				while (n--) 
				{
					++pos;
				}
			}
			else {
				while (n++) 
				{
					--pos;
				}
			}
		}
		else // input_iterator_tag
		{ 
			while (n--) 
			{
				++pos;
			}
		}
	}
	```

所以，在某种程度上，我们现在有一个编译时的转换，不同的情况必须得到由if constexpr子句制定。然而，请注意一个可能重要的区别:
+ 重载方法可以***best match***
+ 编译期if可以***first match***

另外一个例子是通过重载实现编译期`get<>`来实现结构化绑定。第三个例子是处理泛型`lambda`中的不同类型，就像在`std:: variable<>`一样(见144页15.2.3节)。

### 9.3 编译期if初始化
编译期if也可以像运行期if一样，使用初始化(见Chapter 2)。例如：这里有个foo()方法可供使用：

	```
	template<typename T>
	void bar(const T x)
	{
		if constexpr (auto obj = foo(x); std::is_same_v<decltype(obj), T>) 
		{
			std::cout << "foo(x) yields same type\n";
			...
		}
		else 
		{
			std::cout << "foo(x) yields different type\n";
			...
		}
	}
	```

你可以根据`constexpr foo()`方法的返回值是否与`x`一样的类型来作为依据，使得以下的例子有不同种表现：

	```
	constexpr auto c = ...;
	if constexpr (constexpr auto obj = foo(c); obj == 0) 
	{
		std::cout << "foo() == 0\n";
		...
	}
	```

注意，obj必须被声明为constexpr才能在条件中使用它的值。

### 9.4 模版外使用编译期if
`if constexpr`可以使用在任何地方，不仅仅局限于模版。我们只需要一个编译时表达式，生成一些可转换为bool的东西，但是剩下的`then`和`else`部分都需要提供，不能被丢弃。
例如：下面的代码无法通过编译，因为`undeclared()`必须要有明确的定义：

	```
	#include <limits>
	template<typename T>
	void foo(T t);

	int main()
	{
		if constexpr(std::numeric_limits<char>::is_signed) 
		{
			foo(42); // OK
		}
		else 
		{
			undeclared(42); // ALWAYS ERROR if not declared (even if discarded)
		}
	}
	```

下面的例子也无法编译成功，因为其中一个`static assert`总是失败：

	```
	if constexpr(std::numeric_limits<char>::is_signed) 
	{
		static_assert(std::numeric_limits<char>::is_signed);
	}
	else 
	{
		static_assert(!std::numeric_limits<char>::is_signed);
	}
	```

模版外的编译期if唯一的好处就是，虽然代码有效，非处于有效代码部分不会成为程序的一部分，从而减少了可执行文件的大小。
例如：

	```
	#include <limits>
	#include <string>
	#include <array>
	int main()
	{
		if (!std::numeric_limits<char>::is_signed) 
		{
			static std::array<std::string,1000> arr1;
			...
		}
		else 
		{
			static std::array<std::string,1000> arr2;
			...
		}
	}
	```

arr1和arr2 不会同时出现在可执行文件中。

### 9.5 后记
编译时if最初是由Walter Bright、Herb Sutter和Andrei Alexandrescu在[https://wg21link/n3329][1]和Ville Voutilainen在[https://wg21link/n4461][2]的提议所激发的，一个constexpr_if语言特性。在[https://wg21link/p0128r0][3]上， Ville Voutilainen提出了constexpr_if (wherethefeaturegotitsnamefrom)。最终由Jens Maurer在[https://wg21.link/p0292r2][4]中制定。

[1]:[https://wg21link/n3329]
[2]:[https://wg21link/n4461]
[3]:[https://wg21link/p0128r0]
[4]:[https://wg21.link/p0292r2]