---
layout: post
title: "cpp complete guide 20"
subtitle: "cpp complete guide直译"
excerpt: "&ensp;&ensp;&ensp;&ensp;这部分介绍了c++17标准之后，标准库中存在的扩展和修改"
date: 2020-03-08 +08:00
categories: [cpp complete guide]
author: "kcwl"
comments: true
---

# Part IV
## 标准库扩展和修改


这部分介绍了c++17标准之后，标准库中存在的扩展和修改


# Chapter 20
## Type Traits 扩展

c++17扩展了type traits 特征库(标准type方法)使用的能力，和增加了一些新的type traits。

### 20.1 Type Traits \_v 后缀
从c++17开始，可以使用 \_v 后缀获取所有的trype traits value值(通过 \_t获取type)。例如：对于任意类型T
	
	```
	std::is_const<T>::value` 
	```

现在可以写成
	
	```
	std::is_const_v<T>			//since C++17
	```

这种变化适用于所有的type traits。每一个type traits 都有一个对应的相关定义。例如：

	```
	namespace std
	{
		template<typename T>
		constexpr bool is_const_v = is_const<T>::value;
	}
	```

通常我们会在运行时代替判断条件使用：

	```
	if(std::is_signed_v<char>)
	{
		...
	}
	```

在编译期，同样可以使用相关的type traits：

	```
	if constexpr (std::is_signed_v<char>)
	{
		...
	}
	```

或者是在实例化的时候：

	```
	// primary template for class C¡T¿
	template<typename T, bool = std::is_pointer_v<T>>
	class C 
	{
		...
	};

	// partial specialization for pointer types:
	template<typename T>
	class C<T, true> 
	{
		...
	};
	```

class C 提供了指针的偏特化。

\_v后缀也可以用来求 non-boolean数据，例如  std::extent<>可以来获取原始数组的维度

	```
	int a[5][7];
	std::cout << std::extent_v<decltype(a)> << '\n'			//prints 5
	std::cout << std::extent_v<decltype(a),1> << '\n'		//prints 7

### 20.2 新增Type Traits
c++17新增了一些type traits，另外，`is_literal_type<>`和`result_of<>` 被取消。

**`is_aggregate<>`**  
`std::is_aggregate<T>`判断 T是不是聚合类型(aggregate type):

	```
	template<typename T>
	struct D: std::string,std::complex<T>
	{
		std::string data;
	}
	D<float> s、=\{\{"hello"\},\{4.5,6.7\},"world"\};			//OK since c++17
	std::cout << std::is_aggregate<decltype(s)>::value;	//outputs: 1(true)
	```

### 20.3 `std::bool_constant<>`
如果需要通过traits 获取boolean值，可以使用模版别名`bool_constant<>`:

	```
	namespace std
	{
		template<bool B>
		using bool_constant = integral_constant<bool,B>;		//since c++17
		using true_type = bool_constant<true>;
		using false_type = bool_constant<false>;
	}
在c++17中，`std::true_type`和`std::false_type`是直接定义`intergral_constant<bool,true>`和`intergral_constant<boo,false>`。
通常：没有特殊属性的boolean traits 会继承自`std::false_type`，有特殊属性的，继承自`std::true_type`。例如：

	```
	// primary template: in general T is not a void type
	template<typename T>
	struct IsVoid : std::false_type {};

	// specialization for type void:
	template<>
	struct IsVoid<void> : std::true_type {};
	```

现在可以通过继承`bool_constant<>`定义自己规定在编译期的boolean类型。例如：

	```
	template<typename T>
	struct IsLargerThanInt : std::bool_constant<sizeof(T)>sizeof(int)>{};
	```

然后就可以在编译期判断是否这个类型比int大：

	```
	template<typename T>
	void fool(T x)
	{
		if constexpr (IsLargerThanInt<T>::value)
		{
			...
		}
	}
	```

增加对应的 \_v后缀：
	
	```
	template<typename T>
	static cosntexpr auto IsLargerThanInt_v = IsLargerThanInt<T>::value;
	```

这样就可以使用短小的调用了：

	```
	template<typename T>
	void fool(T x)
	{
		if constexpr (IsLargerThanInt_v<T>)
		{
			...
		}
	}
	```

另外一个例子，可以定义一个traits检查移动构造函数是否抛出异常：
	
	```
	template<typename T>
	struct IsNothrowMoveConstructibleT : std::bool_constant<noexcept(T(std::devlval<T>()))>{};
	```

### 20.4 std::void_t<>
鉴于日益增长的需求，c++17标准化了`std::void_t<>`。定义如下：

	```
	namespace std 
	{
		template<typename ...> using void_t = void;
	}
	```

也就是说，可以为模版参数的变量列表指定void。在我们希望在参数列表中单独处理类型的时候，非常有用。

主要应用于，在我们定义新的type traits时，拥有了检查条件的能力。下面的例子演示了主要应用：

	```
	#include <utility>				//for declval<>
	#include <type_traits>			//for true_type, false_type, and void_t

	//primary template:
	template<typename, typename = std::void_t<>>
	struct HasVarious : std::false_type{};

	//partial specialization(may be SFINAEd away):
	template<typename T>
	struct HasVarious<T,std::void_t<std::declval<T>().begin(),typename T::difference_type,typename T::iterator>> : std::true_type{};
	```

上述例子，定义了一个新的type traits `HasVarious<>`,主要检查了3件事：
+ 是否有"begin()"方法？
+ 是否有"difference_type"成员？
+ 是否有"iterator"成员？

当对应的表示提供上述3个条件的时候，偏特化实例才会生效：

	```
	if constexpr (HasVarious<T>::value)
	{
		...
	}
	```
如果其中任何一个条件都没有满足，即没有对应的成员方法和成员变量，那么偏特化就会SFINAE,意味着此表达式不会遵循判断条件，以至于SFINAE，但这不是错误，所以只会匹配到基础函数，最后的`value`就是`false`。

同样的，可以通过std::void_t 方便的定义 检查一个或多个条件。

### 20.5 后记
变量模版是Stephan T. Lavavej 2014年第一次在[https://wg21.link/n3854][1]上提议。最后由Alisdair Meredith 在[https://wg21.link/p0006r0][2]上收录到标准库中。
`std::is_aggregate<>`是由美国一个传统机构介绍给c++17标准(见[https://wg21.link/lwg2911][3])
`std::bool_constant<>`是Zhihao Yuan第一次在[https://wg21.link/n4334][4]中提议，最后收录入标准库([https://wg21.link/n4389][5])
`std::void_t<>`是由Walter E. Brown在[https://wg21.link/n3911][6]中提议。


[1]:[https://wg21.link/n3854]
[2]:[https://wg21.link/p0006r0]
[3]:[https://wg21.link/lwg2911]
[4]:[https://wg21.link/n4334]
[5]:[https://wg21.link/n4389]
[6]:[https://wg21.link/n3911]

---
**c++17新增模版**  

Trait | Effect
:-:   | :-:
is_aggregate<T> | 是否是聚合类型
has_unique_object_representations<T> 	| 任何两个相同的对象都有相同的内存
is_invocable<T,Args...> 	| 是否是可调用的参数
is_nothrow_invocable<T,Args...> 	| 是否是可调用的参数(nothrow)
is_invocable_r<RT,T,Args...> 	| 是否是可调用的参数(returning RT)
is_nothrow_invocable_r<RT,T,Args...> 	| 是否是可调用的参数(returning RT,nothrow)
invoke_result<T,Args...> 	| 被用作可调用的参数列表的结果
is_swappable<T> 	| 类型T是否可以使用swap()方法
is_nothrow_swappable< T> 	| 类型T是否可以使用swap()方法不抛出异常
is_swappable_with<T,T2> 	| 类型T是否可以使用swap()方法，交换特定类型值
is_nothrow_swappable_with<T,T2> 	| 类型T是否可以使用swap()方法，交换特定类型值，不抛出异常
conjunction<B...> 	| 与预算
disjunction<B... > 	| 或运算
negation<B > 	| 非运算