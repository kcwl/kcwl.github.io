---
layout: post
title: "cpp complete guide 8"
subtitle: "cpp complete guide直译"
excerpt: "&ensp;&ensp;&ensp;&ensp;这部分介绍了c++17相关泛型语言的特性。虽然我们从类模板参数推导开始，它也只影响模板的使用，但是后面的章节特别为泛型代码的程序员提供了特性(function templates、类模板和泛型库)。"
date: 2020-03-15 +08:00
categories: [cpp complete guide]
author: "kcwl"
comments: true
---

# Part II
## 模版语言特性

这部分介绍了c++17相关泛型语言的特性。虽然我们从类模板参数推导开始，它也只影响模板的使用，但是后面的章节特别为泛型代码的程序员提供了特性(function templates、类模板和泛型库)。



# Chapter 8
## 模版参数推导

在c++17之前，你必须明确模版参数类型。例如：例子里面的`double`就不能省略：

	```
	std::complex<double> c{5.1,3.3};
	```

std::mutex 需要二次声明，不能省略:
	
	```
	std::mutex mx;
	std::lock_guard<std::mutex> lg(mx);
	```

从c++17开始，你可以更加灵活的显式定义类型。如果当前类型的构造函数能够推导类型，也可以跳过类型定义。
例如：
+ 现在可以这样定义

	```
	std::complex c{5.1,3.3};
	```

+ 可以这样实现：

	```
	std::mutex mx;
	std::lock_guard lg(mx);
	```	


### 8.1 模版参数推导使用方法
只要类型的构造函数可以推导参数类型，都可以使用模版参数推导。这种推导方式支持任何初始化构造函数(类型提供自己的构造函数):

	```
	std::complex c1(1.1, 2.2); // deduces std::complex<double>
	std::complex c2{2.2, 3.3}; // deduces std::complex<double>
	std::complex c3 = 3.3; // deduces std::complex<double>
	std::complex c4 = {4.4}; // deduces std::complex<double>
	```

c3和c4的初始化是可以完成的。因为c3通过赋值操作符实现初始化，c4通过构造函数实现初始化，足够能推导出参数的类型T，然后用于实部和虚部：

	```
	namespace std {
		template<typename T>
		class complex 
		{
			constexpr complex(const T& re = T(), const T& im = T());
			...
		}
	};
	```

注意：参数推导必须符合显示推导原则。因此，下面的例子不能编译成功：
	
	```
	std::complex c5{5,3.3}; // ERROR: attempts to int and double as T
	```

通常来说，模版推导的时候，不存在类型转换。

变参类型的模版类参数推导同样支持。例如：`std::tuple<>`声明如下：

	```
	namespace std {
		template<typename... Types>
		class tuple;
	};
	```

定义如下：

	```
	std::tuple t{42, ’x’, nullptr};
	```

推导之后的类型为：` std::tuple<int, char, std::nullptr_t>`。

同样可以推导非类型的模版参数。例如：我们可以推导传统数组的类型和大小：

	```
	template<typename T, int SZ>
	class MyClass {
	public:
		MyClass (T(&)[SZ]) 
		{
			...
		}
	};

	MyClass mc("hello"); // deduces T as const char and SZ as 6
	```

因为参数为一个6个字符的字符串，所以推导出SZ为6。
同样的可以根据基类推导lambda表达式的类型，或者推导`auto`模版参数类型。


#### 8.1.1 默认拷贝
如果模版参数推导出可以使用赋值方式，那么它更喜欢使用这种方式。例如：使用一个变量初始化`std::vector`

	```
	std::vector v1{42}; // vector<int> with one element
	```

用一个`vector`初始化另外一个,会通过拷贝构造的方式进行赋值：

	```
	std::vector v2{v1}; // v2 also is vector<int>
	```

这个规则应用于所有提供的初始化:

	```
	std::vector v3(v1); // v3 also is vector<int>
	std::vector v4 = {v1}; // v3 also is vector<int>
	auto v5 = std::vector{v1}; // v3 also is vector<int>
	```

只有在传递了多个元素，从而不能采用拷贝方式时，初始化器列表的元素才会定义新`vector`的元素类型:

	```
	std::vector vv{v, v}; // vv is vector<vector<int>>
	```

当传递变参列表的时候，会尝试一个问题：

	```
	template<typename... Args>
	auto make_vector(const Args&... elems) 
	{
		return std::vector{elems...};
	}
	std::vector<int> v{1, 2, 3};
	auto x1 = make_vector(v, v); // vector<vector<int>>
	auto x2 = make_vector(v); // vector<int> or vector<vector<int>> ?
	```

这种情况下，编译器就会表现出不同的情况，目前这个问题还在讨论中。


#### 8.1.2 推导lambda类型
使用类模板参数推导，我们可以用lambda类型实例化类模板(确切地说:lambda的闭包类型)。例如，我们可以提供一个通用类，包装和计数调用的任意回调:

	```
	#include <utility> // for std::forward()
	template<typename CB>

	class CountCalls
	{
	private:
		CB callback; // callback to call
		long calls = 0; // counter for calls
	public:
		CountCalls(CB cb) : callback(cb) {
		}
		template<typename... Args>
		auto operator() (Args&&... args) 
		{
			++calls;
			return callback(std::forward<Args>(args)...);
		}

		long count() const 
		{
			return calls;
		}
	};
	```

通过包装回调函数，构造函数可以推导CB的模版参数类型。例如：我们可以传递一个lambda作为参数：

	```
	CountCalls sc([](auto x, auto y) 
		{
			return x > y;
		});
	```

`sc`的类型被推导为`CountCalls<TypeOfTheLambda>`。这样我们就可以计算函数回调使用的次数。

	```
	std::sort(v.begin(), v.end(),
		std::ref(sc));
	std::cout << "sorted with " << sc.count() << " calls\n";
	```

这里，包装的lambda用作排序标准，但是它必须通过引用来传递，否则std::sort()只使用它自己的已传递的计数器副本来计算，因为sort()本身根据值接受排序条件。

我们也可以把`lambda`传进`std::for_each()`,因为这个算法(在非并行版本中)返回它自己的已传递回调的副本，以便能够使用它的结果状态:

	```
	auto fo = std::for_each(v.begin(), v.end(),
	CountCalls([](auto i) 
		{
			std::cout << "elem: " << i << ’\n’;
		}));
	std::cout << "output with " << fo.count() << " calls\n";
	```

#### 8.1.3 不支持偏特化类模版参数推导
与函数模板不同的是，类模板的参数可能不是部分推导出来的(通过显式地指定一些模板参数)。例如:

	```
	template<typename T1, typename T2, typename T3 = T2>
	class C
	{
	public:
	C (T1 x = T1{}, T2 y = T2{}, T3 z = T3{}) {
	...
	}
	...
	};
	// all deduced:
	C c1(22, 44.3, "hi"); // OK: T1 is int, T2 is double, T3 is const char*
	C c2(22, 44.3); // OK: T1 is int, T2 and T3 are double
	C c3("hi", "guy"); // OK: T1, T2, and T3 are const char*
	// only some deduced:
	C<string> c4("hi", "my"); // ERROR: only T1 explicitly defined
	C<> c5(22, 44.3); // ERROR: neither T1 not T2 explicitly defined
	C<> c6(22, 44.3, 42); // ERROR: neither T1 nor T2 explicitly defined
	// all specified:
	C<string,string,int> c7; // OK: T1,T2 are string, T3 is int
	C<int,string> c8(52, "my"); // OK: T1 is int,T2 and T3 are strings
	C<string,string> c9("a", "b", "c"); // OK: T1,T2,T3 are strings
	```

第三个模版参数有个默认的值。因此当第二个参数有明确的类型的时候，第三个类型就不是必须要推导了。如果你想知道为什么不支持偏特化，那么这里有个例子：

	```
	std::tuple<int> t(42, 43); // still ERROR
	```

`std::tuple`是个变参模版，所以需要明确任意数量的参数类型。所以在这种情况下，不清楚只指定一种类型是错误的，还是故意的。至少看起来是有些问题的。偏特化模版推导在后续也会添加到c++标准中，不过需要大量的时间去思考这个问题。
没有支持偏特化模版参数推导意味着这种诉求没有得到响应。我们仍然不能简单地使用lambda来指定排序条件关联容器或无序容器的散列函数。

	```
	std::set<Cust> coll([](const Cust& x, const Cust& y) // still ERROR
		{ 
			return x.name() > y.name();
		});
	```

我们仍然需要明确的参数类型，所以，可以写成下面的样子：

	```
	auto sortcrit = [](const Cust& x, const Cust& y) 
		{
			return x.name() > y.name();
		};
	std::set<Cust, decltype(sortcrit)> coll(sortcrit); // OK
	```

#### 8.1.4 类模版参数推导取代便利的函数
原则上，我们可以通过类模版参数推导，取代一些只能通过参数调用才能获取到类型的方法。`make_pair`是最明显的例子，它需要参数提供必要的类型。例如：
		
	```
	std::vector<int> v;
	```

我们可以用：

	```
	auto p = std::make_pair(v.begin(), v.end());
	```

代替：

	```
	std::pair<typename std::vector<int>::iterator,
	typename std::vector<int>::iterator> p(v.begin(), v.end());
	```

现在我们可以不使用`make_pair`了:

	```
	std::pair p(v.begin(), v.end());
	```

然而，std::make_pair()也是一个很好的例子，它演示了有时便利函数所做的不仅仅是推断模板参数。事实上，`std::make_pair()`也会使类型退化，当参数是字符串的时候，就会退化为`const char*`:

	```
	auto q = std::make_pair("hi", "world"); // pair of pointers
	```

这个时候，`p`的类型就是`std::pair<const char*,const char*>`。

使用了类模版参数推导，事情也可能会变的更复杂。看一下`std::pair`的定义：

	```
	template<typename T1, typename T2>
	struct Pair1 {
		T1 first;
		T2 second;
		Pair1(const T1& x, const T2& y) : first{x}, second{y} {
		}
	};
	```

这是个引用传值的例子。根据语言规则，使用引用传递模版类型时，将原始数组类型转换为相应的原始指针类型的机制保证参数类型不会退化。所以：

	```
	Pair1 p1{"hi", "world"}; // deduces pair of arrays of different size, but...
	```

`T1`被推导为`char[3]`,`T2`被推导为`char[6]`。原则上，这种推导是可以的。然而，我们把`T1`,`T2`声明为`first`和`second`。对应的，他们被声明为：

	```
	char first[3];
	char second[6];
	```

并且，不允许从数组的左值初始化数组。就好像如此尝试编译：

	```
	const char x[3] = "hi";
	const char y[6] = "world";
	char first[3] {x}; // ERROR
	char second[6] {y}; // ERROR
	```

现在我们可以通过编译，也可以使用值传递：

	```
	template<typename T1, typename T2>
	struct Pair2 
	{
		T1 first;
		T2 second;
		Pair2(T1 x, T2 y) : first{x}, second{y} {}
	};
	```

如果调用以下类型:
	
	```
	Pair2 p2{"hi", "world"}; // deduces pair of pointers
	```

`T1`和`T2`都会被推导为`const char*`。因为声明了类`std::pair<>`，所以构造函数通过引用获取参数，你可能期望下面的初始化不编译:

	```
	std::pair p{"hi", "world"}; // seems to deduce pair of arrays of different size, but...
	```

由于我们使用了推导规则，所以，通过了编译。

### 8.2 推导规则
你可以明确的给传统函数或者现在的模版类添加推导规则。例如：你可以定义一个在任何时候都需要推导规则的`Pair3`，类型推断的操作应该与类型是通过值传递的一样：

	```
	template<typename T1, typename T2>
	struct Pair3 {
		T1 first;
		T2 second;
		Pair3(const T1& x, const T2& y) : first{x}, second{y} 
		{
		}
	};

	// deduction guide for the constructor:
	template<typename T1, typename T2>
	Pair3(T1, T2) -> Pair3<T1, T2>;
	```

在`->`的左边，我们声明我们要推导的东西。在本例中，它是从通过值传递的任意类型`T1`和`T2`的两个对象创建一个`Pair3`。在`->`右边，我们定义了一个结果推导。在这个例子中：`Pair3`是用`T1`和`T2`两种类型实例化的。你也可能认为这是构造函数做的。然而，构造函数是通过参数引用工作的，所以，传递的字符串或者数组类型不会退化。根据推导规则，值传递的数组和字符串都会退化到相应的类型：

	```
	Pair3 p3{"hi", "world"}; // deduces pair of pointers
	```

根据推导原则，虽然`T1`和`T2`都有自己原来的类型(const char[3],const char[6])，但实际上就像我们如此声明:

	```
	Pair3<const char*, const char*> p3{"hi", "world"};
	```

构造函数仍然通过引用的方式获取参数。推导规则只适用于模版推导，和构造函数的调用无关。


#### 8.2.1 使用模版推导强制类型退化
正如前面的示例所演示的，通常是这些重载的一个非常有用的作用，确保模板参数T在推导过程中衰减。例如：

	```
	template<typename T>
	struct C {
		C(const T&) {}
		...
	};
	```

如果传递一个字符串"hello"，T就会按照规则推导为`const char[6]`:

	```
	C x{"hello"}; // T deduced as const char[6]
	```

当模版参数按引用传递时，模版推导不会使类型退化。

加上个简单的约束：
	
	```
	template<typename T> C(T) -> C<T>;
	```

我们解决了这个问题：

	```
	C x{"hello"}; // T deduced as const char*
	```

现在，由于模版推导使得参数变成了值传递，所以`T`被推导成了`const char*`。因此，对于任何通过引用获取参数的构造函数模版类来说，都是非常合理的。

### 8.2.2 非模版推导
推导规则不是必须是模版，也不必显示的指定调用构造函数。

	```
	template<typename T>
	struct S 
	{
		T val;
	};
	S(const char*) -> S<std::string>; // map S<> for string literals to S<std::string>
	```

下面的定义都是可行的，传递的字符串都转换成了`st::string`:

	```
	S s1{"hello"}; // OK, same as: S<std::string> s1{"hello"};
	S s2 = {"hello"}; // OK, same as: S<std::string> s2 = {"hello"};
	S s3 = S{"hello"}; // OK, both S deduced to be S<std::string>
	```

注意，聚合需要列表初始化(可以进行推导，但不允许初始化):
	
	```
	S s4 = "hello"; // ERROR (can’t initialize aggregates that way)
	```


#### 8.2.3 推导规则与构造函数
推导规则和构造函数之间的竞争。类模版参数推导的使用是根据重载解析最高优先级的构造函数或者推导规则。如果两者的匹配度相同，优先选择推导规则。
例如：

	```
	template<typename T>
	struct C1 {
		C1(const T&) {}
	};
	C1(int) -> C1<long>;
	```

当传值的类型是int的时候，就会采用推导规则，因为推导的结果更加匹配，而且也不是一个模版。所以，最后`T`推导为`long`:

	```
	C1 x1(42); // T deduced as long
	```

当传值的类型是char的时候，构造函数是最佳匹配(因为没有必要的类型转换),所以`T`最后的推导为`char`:

	```
	C1 x3(’x’); // T deduced as char
	```

因为通过值进行参数匹配和通过引用进行参数匹配一样好，所以对于同样好的匹配，通常使用推导规则进行参数匹配(这也是类型退化的好处(见8.2.1节))

#### 8.2.4 显示推导规则
推导规则可以用`explicit`声明。这种情况下，会使初始化和转换操作失效。例如：

	```
	template<typename T>
	struct S 
	{
		T val;
	};
	explicit S(const char*) -> S<std::string>;
	```

复制初始化(使用`=`)，传递类型会忽略推导规则，这里表示初始化无效：

	```
	S s1 = {"hello"}; // ERROR (deduction guide ignored and otherwise invalid)
	```

直接初始化和显示推导规则都还生效：

	```
	S s2{"hello"}; // OK, same as: S<std::string> s1{"hello"};
	S s3 = S{"hello"}; // OK
	S s4 = {S{"hello"}}; // OK
	```

另外一个例子：

	```
	template<typename T>
	struct Ptr
	{
		Ptr(T) { std::cout << "Ptr(T)\n"; }
		template<typename U>
		Ptr(U) { std::cout << "Ptr(U)\n"; }
	};
	template<typename T>
	explicit Ptr(T) -> Ptr<T*>;
	```

会产生以下的作用：

	```
	Ptr p1{42}; // deduces Ptr<int*> due to deduction guide
	Ptr p2 = 42; // deduces Ptr<int> due to constructor
	int i = 42;
	Ptr p3{&i}; // deduces Ptr<int**> due to deduction guide
	Ptr p4 = &i; // deduces Ptr<int*> due to constructor
	```

#### 8.2.5聚合类型的推导规则
可以在通用聚合中使用推导规则来启用类模板的参数推导。例如：

	```
	template<typename T>
	struct A 
	{
		T val;
	};
	```

任何没有推导规则的类模版都是错误的：

	```
	A i1{42}; // ERROR
	A s1("hi"); // ERROR
	A s2{"hi"}; // ERROR
	A s3 = "hi"; // ERROR
	A s4 = {"hi"}; // ERROR
	```

必须显示声明类型：
	
	```
	A<int> i2{42};
	A<std::string> s5 = {"hi"};
	```

如果添加了推导规则：
	
	```
	A(const char*) -> A<std::string>;
	```

就可以如下初始化：

	```
	A s2{"hi"}; // OK
	A s4 = {"hi"}; // OK
	```

但是，与通常的聚合一样，仍然需要花括号。否则，类型T就成功了推导，但初始化是一个错误:

	```
	A s1("hi"); // ERROR: T is string, but no aggregate initialization
	A s3 = "hi"; // ERROR: T is string, but no aggregate initialization
	```

`std::array`的推导规则是另外一个例子。

#### 8.2.6 标准推导规则
c++标准库确定了一系列的推导规则(c++17起)。

**迭代器推导**  
例如，能够从定义范围的迭代器中推断元素的类型初始化时，容器有一个推导规则，比如下面的std::vector<>：

	```
	// let std::vector<> deduce element type from initializing iterators:
	namespace std 
	{
		template<typename Iterator>
		vector(Iterator, Iterator)
		-> vector<typename iterator_traits<Iterator>::value_type>;
	}
	```

下面的例子是允许的：

	```
	std::set<float> s;
	std::vector(s.begin(), s.end()); // OK, deduces std::vector<float>
	```

**std::array<>推导规则**  
`std::array<>`的推导规则很有趣，不仅能推导出类型，还能推导出成员的数量：

	```
	std::array a{42,45,77}; // OK, deduces std::array<int,3>
	```

下面是模版推导规则：

	```
	// let std::array<> deduce their number of elements (must have same type):
	namespace std 
	{
		template<typename T, typename... U>
		array(T, U...)
			-> array<enable_if_t<(is_same_v<T,U> && ...), T>,
				(1 + sizeof...(U))>;
	}
	```

这个推导用到了折叠表达式

	```
	(is_same_v<T,U> && ...)
	```

确定所有传入的值的类型相同。所以，下面的例子无法通关编译

	```
	std::array a{42,45,77.7}; // ERROR: types differ
	```

**(无序)Map 推导规则**  
可以通过实验来定义具有键值对的容器(`map`, `multimap`,`unordered_map`, `unordered_multimap`)的推导规则演示正确的推导行为。
这些容器的元素类型为`std::pair<const keytype,valuetype>`。`const`是必须的,因为元素的位置取决于键的值，所以修改键的能力可能会在容器内造成不一致。
所以，在c++ 17标准中实现std::map<>:

	```
	namespace std 
	{
		template<typename Key, typename T,
		typename Compare = less<Key>,
		typename Allocator = allocator<pair<const Key, T>>>
		class map 
		{
			...
		};
	}
	```

定义以下构造函数：
	
	```
	map(initializer_list<pair<const Key, T>>,
	const Compare& = Compare(),
	const Allocator& = Allocator());
	```

推导规则：

	```
	namespace std {
		template<typename Key, typename T,
		typename Compare = less<Key>,
		typename Allocator = allocator<pair<const Key, T>>>
		map(initializer_list<pair<const Key, T>>,
		Compare = Compare(),
		Allocator = Allocator())
			-> map<Key, T, Compare, Allocator>;
	}
	```

由于所有的参数都是通过值传递的，所以这个演绎指南允许传递的比较器或分配器的类型如前所述递减(参见第76页的8.2.1节)。但是，我们很天真地使用了相同的参数类型，这意味着初始化器列表采用const键类型。但作为一个结果，如下图所示，并没有像Ville Voutilainen在https://wg21.link/lwg3025中所指出的那样:

	```
	std::pair elem1{1,2};
	std::pair elem2{3,4};
	...
	std::map m1{elem1, elem2}; // ERROR with original C++17 guides
	```

因为这里的元素被推断为std::pair，这与要求将const类型作为第一对类型的推导规则不匹配。所以，你仍然要写以下内容:

	```
	std::map<int,int> m1{elem1, elem2}; // OK
	```

所以，在推导时，应该去掉常量：

	```
	namespace std {
		template<typename Key, typename T,
		typename Compare = less<Key>,
		typename Allocator = allocator<pair<const Key, T>>>
		map(initializer_list<pair<Key, T>>,
		Compare = Compare(),
		Allocator = Allocator())
			-> map<Key, T, Compare, Allocator>;
	}
	```

但是，为了仍然支持比较器和分配器的退化，我们还必须重载具有const 键值对的推导规则。否则，当传递带有const和非const的键值对时，构造函数的表现将会不同。

**禁止智能指针推导规则**  
在c++标准库中，有些库并没有推导规则。例如：你希望`shared_ptr`和`unique_ptr`都有自己的推导规则：

	```
	std::shared_ptr<int> sp{new int(7)};
	```

用以下代码代替：

	```
	std::shared_ptr sp{new int(7)}; // not supported
	```

这并不能自动生效，因为对应的构造函数是个模版，无法应用隐式推导：

	```
	namespace std {
		template<typename T> 
		class shared_ptr 
		{
		public:
			...
			template<typename Y> explicit shared_ptr(Y* p);
			...
		};
	}
	```

与`T`不同，`Y`是一个不同的模板参数，因此从构造函数中推导出`Y`并不意味着我们可以推断类型`T`，这是一个功能，可以调用像这样的东西:

	```
	std::shared_ptr<Base> sp{new Derived(...)};
	```

简单的提供对应的推导规则：

	```
	namespace std{
		template<typename Y> shared_ptr(Y*) -> shared_ptr<Y>;
	}
	```

但是，这也意味着在分配数组时要遵循以下规则:

	```
	std::shared_ptr sp{new int[10]}; // OOPS: would deduces shared_ptr<int>
	```

就像在c++中经常发生的那样，我们遇到了C类的棘手问题:指针的类型指向一个对象和一个对象对象数组具有或退化为同一类型。因为这个问题看起来很危险，所以c++标准委员会决定不支持它

(然而)。你仍然需要调用单个对象:

	```
	std::shared_ptr<int> sp1{new int}; // OK
	auto sp2 = std::make_shared<int>(); // OK
	```

对于数组：

	```
	std::shared_ptr<std::string> p(new std::string[10],
		[](std::string* p) 
		{
			delete[] p;
		});
	```

或者:

	```
	std::shared_ptr<std::string> p(new std::string[10],
			std::default_delete<std::string[]>());
	```


### 8.3 后记
类模板参数演绎是由Michael Spertus在2007年[https://wg21.link/n2332][1]上首次提出的。2013年，Michael Spertus和David Vandevoorde在[https://wg21.link/n3602][2]中提出了这一建议。
最终由Michael Spertus, Faisal Vali, and Richard Smith在[https://wg21.link/p0091r3][3]中做出修改， Jason Merrill在[https://wg21.link/p0620r0][4]上做出修改， Michael Spertus and Jason Merrill (as a defect report against C++17)在[https://wg21.link/p702r1][5]中做出修改。
最终由Michael Spertus, Walter E. Brown, and Stephan T. Lavavej在[https://wg21.link/p0433r2][6]和[https://wg21.link/p0739r0][7]上收录进标准库。


[1]:[https://wg21.link/n2332]
[2]:[https://wg21.link/n3602]
[3]:[https://wg21.link/p0091r3]
[4]:[https://wg21.link/p0620r0]
[5]:[https://wg21.link/p702r1]
[6]:[https://wg21.link/p0433r2]
[7]:[https://wg21.link/p0739r0]