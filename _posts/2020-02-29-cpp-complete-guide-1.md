---
layout: post
title: "cpp complete guide 1"
subtitle: "cpp complete guide直译"
excerpt: "&ensp;&ensp;&ensp;&ensp;这部分介绍了c++17的核心语言特性，而不是针对泛型编程。所以，使用c++17的人应该知道他们。针对泛型编程的核心特性在Part II。"
date: 2020-02-29  +08:00
categories: [cpp complete guide]
author: "kcwl"
comments: true
---

# Part I
## 基础语言特性


### 1.结构化绑定 (Structured Bindings)  
结构化绑定允许通过对象的元素或成员进行初始化多个实体。例如，你声明了一个拥有2个成员的结构体

	```
	struct MyStruct 
	{
		int i=0;
		std::string s;
	}  
	MyStruct ms;
	```

你可以通过以下的声明来直接使用新变量绑定成员：

	```
	auto [u,v] = ms;
	```

在这里，u和v就叫做结构化绑定。某种程度上，他们是通过分解对象去初始化，所以有时候也叫做分解声明(decomposing declarations)。结构化绑定也可以用于函数返回值。举个例子:有个返回结构体的方法：

	```
	MyStruct getStruct()
	{
		return MyStruct{42,"hello"};
	}
	```

可以直接通过2个变量去获得返回值数据成员：

	```
	auto [id,val] = getStruct();       // id,val就是结构体的i,s
	```

id,val就是函数返回值的结构体成员i,s的新载体，他们拥有对应的类型，int和string，他们也可以分开使用：

	```
	if (id > 30)
	{
		std::cout << val;
	}
	```

这样做的好处就是直接访问，通过绑定值使代码更具有可读性，更加明确目的。下面的代码演示了如果通过结构化绑定提升代码。没有结构化绑定的时候，如果你想遍历std::map<>，你需要写出以下的代码：

	```
	for (const auto& elem : mymap)
	{
		std::cout << elem.first << ":" << elem.second << '\n';
	}
	```

元素是带有key,value的std::pair, std::pair的成员是first,second，你必须要通过调用他们才可以获得key,value的值。通过结构化绑定，代码更具有可读性：


	```
	for (const auto&[key,val] : mymap)
	{
		std::cout << key << ":" << val << '\n';
	}
	```

可以直接使用key和value 使得更加清晰的展示其语义的名称。

### 1.1 细化结构化绑定 (Structured Bindings)
为了更好的理解结构化绑定，需要一个新的匿名变量。作为结构化绑定引入的新变量指的是匿名变量的成员/元素。

绑定一个匿名实体,确切初始化行为

	```
	auto [u,v] = ms;
	```

就好像是初始化了一个新变量e ，然后让u,v成为这个新对象的成员的别名，类似以下声明：

	```
	auto e = ms;
	auto& u = e.i;
	auto& v = e.s;
	```

唯一的区别在于，结构化绑定没有e，不需要通过e来访问
	结果：

	```
	std::cout << u << ' ' << v << '\n';
	```

输出值 u,v  等同于 e.i,e.v,不通过引用修改结构化绑定的获取到的值，不会产生任何作用
	
	```
	MyStruct ms{42,"hello"};
	auto [u,v] = ms;
	ms.i = 77;
	std::cout << u ;     //42
	u = 99;
	std::cout << ms.i;   //77
	```

u和ms.i都有自己的地址。当使用结构化绑定于返回值时，同上。
	
	```
	auto [u,v] = getStruct();
	```

就好像定义了一个新变量e作为函数getStruct的返回值，使 u,v作为变量e的成员/元素的别名：

	```
	auto e = getStruct();
	auto& u = e.i;
	auto& v = e.s;
	```

也就是说结构化绑定到一个新的实体，这个实体是通过返回值初始化来直接代替返回值。匿名实体保证地址对齐，使得结构化绑定对应着相应的成员。例如: 	

	```
	auto [u,v] = ms;
	assert(&((MyStruct*)&u)->s == &v);    //OK
	```

**使用限定符**  
	在结构化帮顶中，也可以使用const references 限定符。这些限定符都可以和结构化绑定一起用，效果类似于限定结构化绑定。凡是总有例外：
	声明以下：

	```
	const auto& [u,v] = ms;    //引用，所以 u/v 引自 ms.i/ms.s
	```

匿名变量被定义成了const 引用，意味着不可被改变；
	结果：

	```
	ms.i = 77;          //影响u的值
	std::cout << u; 	//输出77
	```

定义一个非const引用，同样可以改变对象的值：

	```
	MyStruct ms{42,"hello"};
	auto& [u,v] = ms;		//初始化 ms的一个引用
	ms.i = 77;				//改变的u的值
	std::cout << u;			//输出77
	u = 99;					//改变ms.i
	std::cout << ms.i;		//输出99
	```

如果我们用来初始化的结构化绑定的变量是临时对象，会延长临时对象的生命周期到结构化绑定的生命周期：
	
	```
	MyStruct getStruct();
	const auto& [a,b] = getStruct();
	std::cout << "a:" << a << '\n';			//OK
	```

**限定符不一定都作用于结构化绑定**  
像前面说的那样，限定符可以作用在新的匿名实体上，但不一定作用于由结构化绑定引入的变量。不同之处可以通过以下例子演示：

	```
	alignas(16) auto [u,v] = ms;   //对齐对象，不是v
	```

这里，我们对齐了匿名对象，但是不是单独的u或者v，u作为第一个成员，强制对齐到16，但是v没有。同样的情况，尽管用了auto，结构化绑定不能够decay。举例：有个带有原始数组的结构体：

	```
	struct S
	{
		const char x[6];
		const char y[3];
	}
	```

然后：

	```
	S s1{};
	auto [a,b] = s1;	//a和b都获得了确切的成员类型
	```

a的类型仍然是const char[6]。 再一次用auto 推导新对象的时候，新对象的类型decay

	```
	auto a2 = a; 	//a2 gets decayed type of a (a2 的类型为char[6])
	```

**移动语义**  
	移动语义支持如下：

	```
	MyStruct ms = {42,"Jim"};
	auto&& [v,n] = std::move(ms);			//匿名实体是个右值
	```

结构化绑定的v和n是ms的一个右值引用。ms仍然持有值。

	```
	std::cout << "ms.s: " << ms.s << '\n';  	//输出 "Jim"
	```

对ms.s的引用n使用move
	
	```
	std::string s = std::move(n);				//把ms.s移动到s
	std::cout << "ms.s: " << ms.s << '\n';		//输出不确定的值
	std::cout << "n: " << n << '\n';			//输出不确定的值
	std::cout << "s: " << s << '\n';			//输出"Jim"
	```
 
通常，被转移的对象是有效状态，但是值不确定，不要对打印的值做任何的假设。下面是move的例子，这个例子不同于右值引用：

	```
	MyStruct ms = {42,"Jim"};
	auto [v,n] = std::move(ms); 			//新的变量是转移之后的对象，
	```

新初始化的变量是从ms转移出来的值，所以，ms会失去自己的：

	```
	std::cout << "ms.s : " << ms.s << '\n';				//输出不确定的值
	std::cout << "n: " << n << '\n';					//输出 "Jim"
	```

同样也可以对之前结构化绑定的值进行move，但是再也不会影响ms.s:

	```
	std::string s = std::move(n);
	n = "Lara";
	std::cout << "ms.s: " << ms.s << '\n;			//输出不确定的值
	std::cout << "n: " << n << '\n';				//输出 "Lara"
	std::cout << "s: " << s << '\n';				//输出 "Jim"
	```

### 1.2 哪里可以使用结构化绑定
原则上，结构化绑定可以用于public成员，原始C风格的数组，和一些类似元组(tuple-like)的对象。

+ 如果结构体或者类中所有的非静态成员都是public，都可以使用结构化绑定。
+ 对于原始数组，可以为每个成员都绑定一个别名
+ 对于任何类型，都可以使用tuple-like API 提取元素。API大致如下：
		- std::tuple_size<type>::value 返回type对象的元素个数
		- std::tuple_element<idx,type>::type 返回第idx个元素的类型
		标准库类型std::pair<>, std::tuple<>, std::array<>都已经提供了这些API。

如果类或者结构体都提供了类似的API，那么就可以使用它们。

在任何情况下，元素的数量或者是数据成员，都必须要和其绑定的数据的个数相同，不可以跳过这些变量，也不可以使用第二次，可以使用非常短的名字(eg: "-"),这样的名字在同一作用域也仅仅能使用一次：

	```
	auto [_,val1] = getStruct(); 				//OK 
	auto [_,val2] = getStruct();				//ERROR: '_'已经被使用过了
	```

 结构化绑定不支持嵌套或者非平面的分解！！！


#### 1.2.1 结构体和类
下面的例子演示了结构体和类如何和结构化绑定相互结合
注意：
在继承中，结构化绑定是有限的！所有的非静态成员变量必须在一个类中定义(他们必须是同一类型,或者是同一类型的明确基类)

    ```
    struct B
    {
        int a = 1;
        int b = 2;
    }

	struct D1 : B{}

	auto [x,y] = D1{};				//OK

	struct D2 : B
	{
		int c = 3;				
	}

	auto [i,j,k] = D2{};			// 编译错误
	```

#### 1.2.2 原始数组
下面的例子使用结构化绑定初始化一个带有2个元素的原始数组

	```
	int arr[]={47,11};
	auto [x,y] = arr;				//x和y是数组元素 47,11的别名
	auto [z] = arr;					//ERROR:  数量不正确
	```

上面的例子只适用于知道数量的数组。通过函数传递的数组无法使用这种方式，因为此时，数组已经退化成了对应的指针类型。c++允许通过引用的方式返回数组的数量，可以通过这个特性使用结构化绑定：

	```
	auto getArr() ->int(&)[2];		//getArr() 返回原始int数组
	...
	auto [x,y] = getArr();			//x和y通过数组返回值初始化
	```

#### 1.2.3 std::pair, std::tuple, std::array
结构化绑定这种机制是可以扩展的，可以对任何类型添加支持。标准库对std::pair<>,std::tuple<>,std::array<>进行了支持。

**std::array**  
	下面的例子演示了i,j,k,l初始化std::array<>的元素

	```
	std::array<int,4> getArray();
	...
	auto [i,j,k,l] = getArray();		//i,j,k,l 通过返回值对应初始化
	```

同样可以通过引用的方式改变数组的元素值：

	```
	std::array<int,4> stdarr{1,2,3,4};
	...
	auto& [i,j,k,l] = stdarr;
	i += 10;			//改变的stdarr的第一个元素
	```

**std::tuple**  
下面的例子通过返回tuple的3个元素进行结构化绑定。

	```
	std::tuple<char,float,std::string> getTuple();
	...
	auto [a,b,c] = getTuple();			//	a,b,c初始化为对应类型
	```
 
**std::pair**  
下面这个例子演示了调用insert()返回值时，直接绑定别名，要比调用std::pair<> 的first和second方法更具可读性：

	```
	std::map<std::string,int> coll;
	auto [pos,ok] = coll.insert({"new",42});
	if(!ok)
	{
		//如果插入失败，就可以通过pos获取插入失败的位置
	}
	```

在c++17之前对应的方式如下：
	
	```
	std::map<std::string,int> coll;
	auto ret = coll.insert({"new",42});
	if(!ret.second)
	{
		//如果插入失败，通过ret.first获取插入失败的位置
	}
	```

### 1.3 Tuple-Like API
上面提到，可以对任意的类型提供tuple-like API，标准库为std::pair<>,std::tuple<>,std::array<>提供了支持。


**提供只读结构化绑定**  
下面的例子演示了如何为class Customer添加只读结构化绑定
lang/customer1.hpp

	```
	#include <string>
	#incldue <utility>      //for std::mvoe()

	class Cusromer 
	{
	private:
		std::string first;
		std::string last;
		long val;
	public:
		Customer(std::string f,std::string l, long v)
			: first(std::move(f)) , last(std::move(l), val(v)){}

		std::string getFirst() const 
		{
			return first;
		}

		std::string getLast() const
		{
			return last;
		}

		long getValue() const
		{
			return val;
		}
	}
	```

下面提供结构化绑定:

	```
	#include "customer1.hpp"
	#include <utility>				//for tuple-like API

	//提供一个tuple-like API 进行结构化绑定
	template<>
	struct std::tuple_size<Customer>
	{
		static constexpr int value = 3;				//类有3个属性
	}

	template<>
	struct std::tuple_elment<2,Cutomer>
	{
		using type = long;							//最后一个属性是long类型
	}

	template<std::size_t Idx>
	struct std::tuple_element<Idx,Cutomer>
	{
		using type = std::string;
	}

	//定义明确的get方法
	template<std::size_t>
	auto get(const Customer& c);

	template<>
	auto get<0>(const Customer&c) {return c.getFirst();}

	template<>
	auto get<1>(const Customer&c) {return c.getLast();}

	template<>
	auto get<2>(const Customer&c) {return c.getValue();}

	以上，已经定义好了Customer的3个属性，把这些属性映射到类中对应的变量上
		(1) 第一个属性是std::string类型
		(2) 第二个属性是std::string类型
		(3) 第三个属性是long 类型

	并且明确的定义了属性的数量
	template<>
	struct std::tuple_size<Customer> 
	{
		static const int value = 3; // we have 3 attributes
	};

	明确的定义了每个成员的类型
	template<>
	struct std::tuple_element<2, Customer> 
	{
		using type = long; // last attribute is a long
	};

	template<std::size_t Idx>
	struct std::tuple_element<Idx, Customer> 
	{
		using type = std::string; // the other attributes are strings
	};
	```

第三个属性使用了index 2进行完全特化，其他属性使用了偏特化，完全特化的优先级高于偏特化。最后定义了对应的get<>()方法：

	```
	template<std::size_t>
	auto get(const Customer& c);

	template<>
	auto get<0>(const Customer&c) {return c.getFirst();}

	template<>
	auto get<1>(const Customer&c) {return c.getLast();}

	template<>
	auto get<2>(const Customer&c) {return c.getValue();}
	```

首先我们定义了一个基础模版方法，然后根据情况进行全特化，所有的特化方法都要和基础方法的参数、返回值保持一致，因为只是提供了特定的实现，没有新的声明。下面的就不会编译成功：

	```
	template<std::size_t>
	auto get(const Customer& c);

	template<>
	std::string get<0>(const Customer&c) {return c.getFirst();}

	template<>
	std::string get<1>(const Customer&c) {return c.getLast();}

	template<>
	long get<2>(const Customer&c) {return c.getValue();}
	```

如果使用了c++17标准，我们可以合并上面3个方法：

	```
	tempalte<std::size_t I>
	auto get(const Customer& c)
	{
		static_assert(I < 3);
		if constexpr (I == 0)
			return c.getFirst();
		else if constexpr (i == 1)
			return c.getLast();
		else
			return c.getValue();
	}
	```

有了上面的api，类Customer就可以使用结构化绑定了。

	```
	#include "structbind1.hpp"
	#include <iostream>
	int main()
	{
		Customer c("Tim", "Starr", 42);
		auto [f, l, v] = c;
		std::cout << "f/l/v: " << f << ’ ’ << l << ’ ’ << v << ’\n’;

		// modify structured bindings:
		std::string s = std::move(f);
		l = "Waters";
		v += 10;
		std::cout << "f/l/v: " << f << ’ ’ << l << ’ ’ << v << ’\n’;
		std::cout << "c: " << c.getFirst() << ’ ’
		<< c.getLast() << ’ ’ << c.getValue() << ’\n’;
		std::cout << "s: " << s << ’\n’;
	}
	```

以上形式的结构化绑定是通过类的对应get()函数所获取的值，所以修改c并不会影响新变量的值(反之亦然)，所以会有以下输出:

	```
	f/l/v: Waters 52
	c: Tim Starr 42
	s: Tim
	```

**提供读写权限的结构化绑定**  
下面的例子演示了如何为class Customer的结构化绑定添加读写权限：

	```
	#include <string>
	#include <utility> // for std::move()
	class Customer {
	private:
		std::string first;
		std::string last;
		long val;
	public:
		Customer (std::string f, std::string l, long v)
		: first(std::move(f)), last(std::move(l)), val(v) {}

		const std::string& firstname() const 
		{
			return first;
		}
		std::string& firstname() 
		{
			return first;
		}
		const std::string& lastname() const 
		{
			return last;
		}
		std::string& lastname() 
		{
			return last;
		}
		long value() const 
		{
			return val;
		}
		long& value() 
		{
			return val;
		}
	};
	```

提供对应的get()方法:

	```
	#include "customer2.hpp"
	#include <utility> 			// for tuple-like API

	// provide a tuple-like API for class Customer for structured bindings:
	template<>
	struct std::tuple_size<Customer> 
	{
		static constexpr int value = 3; // we have 3 attributes
	};

	template<>
	struct std::tuple_element<2, Customer> 
	{
		using type = long; // last attribute is a long
	};
	template<std::size_t Idx>
	struct std::tuple_element<Idx, Customer> 
	{
		using type = std::string; // the other attributes are strings
	};

	// define specific getters:
	template<std::size_t I> 
	decltype(auto) get(Customer& c) 
	{
		static_assert(I < 3);
		if constexpr (I == 0) 
		{
			return c.firstname();
		}
		else if constexpr (I == 1) 
		{
			return c.lastname();
		}
		else // I == 2
		{ 
			return c.value();
		}
	}

	decltype(auto) get(const Customer& c) 
	{
		static_assert(I < 3);
		if constexpr (I == 0) 
		{
			return c.firstname();
		}
		else if constexpr (I == 1) 
		{
			return c.lastname();
		}
		else // I == 2
		{ 
			return c.value();
		}
	}

	template<std::size_t I> 
	decltype(auto) get(Customer&& c) 
	{
		static_assert(I < 3);
		if constexpr (I == 0) 
		{
			return std::move(c.firstname());
		}
		else if constexpr (I == 1) 
		{
			return std::move(c.lastname());
		}
		else // I == 2
		{ 
			return c.value();
		}
	}
	```
	
为了处理const ,no-const ,move对象的情况，这定义了3个重载，为了能让返回值返回引用那么就一定要用decltype(auto)。如果不使用c++17 constexpr 的话，就需要进行特化和偏特化:

	```
	template<std::size_t> decltype(auto) get(Customer& c);
	template<> decltype(auto) get<0>(Customer& c) { return c.firstname(); }
	template<> decltype(auto) get<1>(Customer& c) { return c.lastname(); }
	template<> decltype(auto) get<2>(Customer& c) { return c.value(); }
	```

特化版本的参数和返回值必须要和基础函数一致，否则无法进行编译。如下：

	```
	template<std::size_t> decltype(auto) get(Customer& c);
	template<> std::string& get<0>(Customer& c) { return c.firstname(); }
	template<> std::string& get<1>(Customer& c) { return c.lastname(); }
	template<> long& get<2>(Customer& c) { return c.value(); }
	```

现在就可以使用结构化绑定进行读写值了

	```
	#include "structbind2.hpp"
	#include <iostream>
	int main()
	{
		Customer c("Tim", "Starr", 42);
		auto [f, l, v] = c;
		std::cout << "f/l/v: " << f << ’ ’ << l << ’ ’ << v << ’\n’;
		// modify structured bindings via references:
		auto&& [f2, l2, v2] = c;
		std::string s = std::move(f2);
		l2 = "Waters";
		v2 += 10;
		std::cout << "f2/l2/v2: " << f2 << ’ ’ << l2 << ’ ’ << v2 << ’\n’;
		std::cout << "c: " << c.firstname() << ’ ’
		<< c.lastname() << ’ ’ << c.value() << ’\n’;
		std::cout << "s: " << s << ’\n’;
	}
	```

输出如下：
	
	```
	f/l/v: Tim Starr 42
	f2/l2/v2: Tim Starr 42
	c: Waters 52
	s: Tim
	```

### 1.4 后记
结构化绑定最初是由Herb Sutter、Bjarne Stroustrup和Gabriel Dos Reis提出在[https://wg21/p0144r0][1]。使用大括号而不是方括号来链接。最后Jens Maurer在[https://wg21.link/p0217r3][2]中为这个特性制定了公认的措辞。


[1]:https://wg21/p0144r0
[2]:https://wg21.link/p0217r3