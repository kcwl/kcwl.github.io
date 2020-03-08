---
layout: post
title: "cpp complete guide 4"
subtitle: "cpp complete guide直译"
excerpt: "&ensp;&ensp;&ensp;&ensp;聚合表达式"
date: 2020-03-08 +08:00
categories: [cpp complete guide]
author: "kcwl"
comments: true
---

# Chapter 4
## 聚合表达式


在c++中，对象有种聚合初始化的方式，允许使用花括号初始化多个变量：

	```
	struct Data
	{
		std::string name;
		double value;
	};

	Data x{"test1",6.778};
	```

从c++17开始，使用聚合初始化方式的类可以有基类，继承于基类的子类初始化方式如下：
	
	```
	struct MoreData : Data
	{
		bool done;
	};

	MoreData y\{{"test1",6.778},false};
	```

聚合初始化现在支持嵌套大括号来将值传递给派生的基类的成员。

### 4.1 扩展聚合初始化的原因

没有这个特性之前，带有基类的聚合初始化定义一个构造函数：

	```
	struct Cpp14Data : Data 
	{
		bool done;
		Cpp14Data(const std::string& s,double d, bool b)
			: Data{s,d},done{b}{}
	};

	Cpp14Data y\{"test1",6.778,false\};
	```

现在，我们可以使用更简单的方法，看到值如何传递到基类:

	```
	MoreData y\{\{"test1",6.778\},false\};
	```

### 4.2 使用聚合初始化扩展
有一个典型的例子，扩展之后的聚合初始化提升了初始化子类带有传统C风格的数据成员或者是操作符。例如：

	```
	struct Data 
	{
		const char* name;
		double value;
	};

	struct PData : Data 
	{
		bool critical;
		void print() const 
		{
			std::cout << ’[’ << name << ’,’ << value << "]\n";
		}
	};

	PData y\{\{"test1", 6.778\}, false\};
	y.print();
	```
括号内的参数传递给了Data。现在，你可以跳过初始化，这样做的话，成员就会调用默认的构造函数,或者是初始化为0,false或者nullptr。例如：

	```
	PData a\{\}; // zero-initialize all elements
	PData b\{\{"msg"\}\}; // same as \{\{"msg",0.0\},false\}
	PData c\{\{\}, true\}; // same as \{\{nullptr,0.0\},true\}
	PData d; // values of fundamental types are unspecified
	```

注意：使用花括号和不使用花括号的区别：
+ 使用花括号初始化，`string name`会调用默认构造函数，`double value`会被初始化为0.0，`bool flag`初始化为false。
+ 不适用花括号初始化，就只有`string name`会调用默认构造函数，其他成员的值会不确定。

可以从非聚合类继承：

	```
	struct MyString : std::string 
	{
		void print() const 
		{
			if (empty()) 
			{
				std::cout << "<undefined>\n";
			}
			else 
			{
				std::cout << c_str() << ’\n’;
			}
		}
	};
	```

也可以继承多种类型：

	```
	template<typename T>
	struct D : std::string, std::complex<T>
	{
		std::string data;
	};
	```

然后初始化他们：

	```
	D<float> s\{\{"hello"\}, \{4.5,6.7\}, "world"\}; // OK since C++17
	std::cout << s.data; // outputs: ”world”
	std::cout << static_cast<std::string>(s); // outputs: ”hello”
	std::cout << static_cast<std::complex<float>>(s); // outputs: (4.5,6.7)
	```
内部初始化列表按照基类的声明传递给基类。新特性可以帮助我们用更少的代码声明一个lambda表达式(见13.1 on 117)

### 4.3 聚合定义
从c++17开始，`aggregate`被定义为：
+ 无手动声明或者隐式构造函数
+ 没有通过声明继承的构造
+ 没有private或者protected成员变量
+ 没有虚函数
+ 没有virtual,private 或者protected基类

在初始化过程中要求，使用聚合初始化的类不能有private,protected基类。
c++17 增加了一个type traits来判断是否为聚合类型：

	```
	template<typename T>
	struct D : std::string, std::complex<T> {
	std::string data;
	};
	D<float> s\{\{"hello"\}, \{4.5,6.7\}, "world"\}; // OK since C++17
	std::cout << std::is_aggregate<decltype(s)>::value; // outputs: 1 (true)
	```


### 4.4 向后不兼容
下面的例子，c++17之后就不能通过编译了：

	```
	struct Derived;
	struct Base {
		friend struct Derived;
	private:
		Base() {
		}
	};

	struct Derived : Base {
	};
	int main()
	{
	Derived d1\{\}; // ERROR since C++17
	Derived d2; // still OK (but might not initialize)
	}
	```

c++17之前，没有聚合的概念，所以：
	
	```
	Derived d1\{\};
	```

调用了Derived的隐式定义的默认构造函数，默认情况下该构造函数调用默认值基类的构造函数。虽然基类的默认构造函数是私有的，通过派生类的默认构造函数调用它依然是有效的，因为派生类
被定义为一个友元类。

从c++17之后，例子中的子类就是聚合类型了，不再拥有隐式的构造函数(构造函数不能通过声明来继承)。所以，初始化是聚合初始化，不允许调用基类的构造函数。不管是否为友元类。

### 4.5 后记
扩展聚合初始化最初是由Oleg Smolsky在[https://wg21.link/n4404][1]中提出的。最终也由Oleg Smolsky在[https://wg21.link/p0017r1][2]中接受。


type trait std::is_aggregate<>被引美国国家机构引入c++ 17的标准化(见[https://wg21.link/lwg2911][3])。


[1]:[https://wg21.link/n4404]
[2]:[https://wg21.link/p0017r1]
[3]:[https://wg21.link/lwg2911]