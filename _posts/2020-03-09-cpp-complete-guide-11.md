---
layout: post
title: "cpp complete guide 11"
subtitle: "cpp complete guide直译"
excerpt: "&ensp;&ensp;&ensp;&ensp;把string用作模版参数"
date: 2020-03-14 +08:00
categories: [cpp complete guide]
author: "kcwl"
comments: true
---

# Chapter 11
## 把string用作模版参数

随着时间的推移，不同版本的c++放宽了作为模板参数使用的规则，c++ 17又一次出现了这种情况。现在可以使用模板，而不需要在当前范围之外定义它们。



### 11.1 在模版中使用字符串
非类型模版参数只能是数字常量，成员对象指针，左值引用或者`std::nullptr_t`(空指针的类型)。传递指针的时候，需要链接，这意味着不可以传递字符串。在c++17中却可以这样做。例如：

	```
	template<const char* str>
	class Message 
	{
		...
	};

	extern const char hello[] = "Hello World!"; // external linkage
	const char hello11[] = "Hello World!"; // internal linkage

	void foo()
	{
		Message<hello> msg; // OK (all C++ versions)
		Message<hello11> msg11; // OK since C++11
		static const char hello17[] = "Hello World!"; // no linkage
		Message<hello17> msg17; // OK since C++17
	}
	```

从c++17开始，需要两行代码使模版支持字符串，可以将第一行放在与类实例化相同的范围内。这个特性也解决了c++11以来的一个缺陷，当向类模版传递一个指针的时候：

	```
	template<int* p> struct A 
	{
	};
	int num;
	A<&num> a; // OK since C++11
	```

以前不能使用编译期函数返回地址，现在可以了：

	```
	int num;
	...
	constexpr int* pNum() {
	return &num;
	}
	A<pNum()> b; // ERROR before C++17, now OK
	```

### 11.2 后记
允许对所有非类型模板参数进行常量计算是Richard Smith在https://wg21.link/n4198首先提出的。最终由Richard Smith在https://wg21.link/n4268拟定。



[1]:[https://wg21.link/n4198]
[2]:[https://wg21.link/n4268]