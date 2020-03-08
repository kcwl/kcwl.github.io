---
layout: post
title: "cpp complete guide 3"
subtitle: "cpp complete guide直译"
excerpt: "&ensp;&ensp;&ensp;&ensp;c++的一个优势是c++有能力支持header-only开发。"
date: 2020-03-07  +08:00
categories: [cpp complete guide]
author: "kcwl"
comments: true
---

# Chapter 3
## 内联变量


c++的一个优势是c++有能力支持header-only开发。在c++17以前，只有库不需要全局变量或者对象的时候，才可以实现。c++17之后，可以使用`inline` 声明一个变量，它可以在多种环境下使用，他们都指向东一个对象：

	```
	class MyClass
	{
		static inline std::string name = ""; 		//OK since c++17
		...
	};

	inline MyClass myClobalObj;						//OK even if include/defined by multiple CPP files
	```



###3.1 内联变量的动机
c++中，不允许在类结构中初始化静态变量：

	```
	class MyClass
	{
		static std::string name = "";		//Complie-Time ERROR
	};
	```

定义在类外，如果在多个cpp文件里面包含同一个头文件，也经常会出现错误：

	```
	class MyClass 
	{
		static std::string name;		//OK 
	};

	MyClass::name = ""; 		//Link ERROR if included by multiple CPP files
	```

根据one definition rule(ODR)原则，一个变量或者实体必须在一个翻译单元。宏定义也没有办法：

	```
	#ifndef MYHEADER_HPP
	#define MYHEADER_HPP
	class MyClass 
	{
		static std::string name; // OK
		...
	};
	MyClass.name = ""; // Link ERROR if included by multiple CPP files
	#endif
	```

这不是同一个头文件被包含多次的问题，问题在于多个cpp文件包含头文件导致重复定义MyClass.name。 如果你在类外定义了一个对象，那么同样也会link error。

	```
	class MyClass
	{
		...
	};
	MyClass myGlobalObject;		//Link ERROR if included by multiple CPP files
	```

**解决办法**  
很多情况下都有解决方案：
+ 可以在类中初始化static const变量：

	```
	class MyClass
	{
		static const bool trace = false;
	};
	```

+ 也可以定义一个内联方法返回一个stati变量：

	```
	inline std::string getName()
	{
		static std::string name = "initial value";
		return name;
	}
	```

+ 也可以定义一个静态方法，返回值：
	
	```
	std::string getMyGlobalObject()
	{
		static std::string myGlobalObject = "initial value";
		return myGLobalObject;
	}
	```

+ 也可以使用 变量模版(since c++14):

	```
	template<typename T = std::string>
	T myGlobalObject = "initial value";
	```

+ 也可以继承基类的静态成员：

	```
	template<typname Dummy>
	class MyClassStatics
	{
		static std::string name;
	};
	template<typename Dummy>
	std::string MyClassStatics<Dummy>::name = "initial value";
	class MyClass : public MyClassStatics<void>{};
	```

但是这些解决方案都会导致巨大的开销，可读性变差，各种途径去使用全局变量。另外，全局变量的初始化可能推迟到它第一次使用的时候。使得程序不能在开始的时候进行初始化(例如：使用对象来监视进程)。

### 3.2 使用内联变量
现在，通过inline，可以在类内头文件中定义变量，不用担心被多个cpp文件包含：

	```
	class MyClass
	{
		static inline std::string name = "";		//OK since c++17
		...
	};

	inline MyClass myGlobalObj;			//OK even if included/defined by multiple cpp files
	```

当包含标头或包含的第一个翻译单元时执行初始化，输入这些定义。
这里的inline的使用和inline function 是同样的语义：
+ 可以在多个翻译单元被定义，所有的定义都是相同的。
+ 必须在使用的每个翻译单元中都定义。

两者都是通过包含头文件实现的。行为表现等同于变量。也可以使用inline 原子类型：

	```
	inline std::atomic<bool> ready{false};
	```

通常，原子变量必须在定义的时候初始化。在初始化之前，一定要确保类型是完整的。例如：一个类或者结构体，有静态成员，就只能声明成inline。

	```
	struct MyValue
	{
		int value;
		MyValue(int i) : value{i}{}

		//one static object ot hold the maximum value of this type:
		static const MyValue max;				//can only be declared here
		...
	};

	inline const MyValue MyValue::max = 1000;
	```

### 3.3 constexpr 现在意味着 inline
对于静态成员来说，constexpr就意味着inline，下面的定义从c++17开始，就是声明了：

	```
	struct D
	{
		static constexpr int n=5;	//c++11/c++14: declaration
									//since c++17: definition
	};
	```

等同于
	
	```
	struct D
	{
		inline static constexpr int n = 5;
	}
	```

在c++17之前，就可以在没有相关的定义声明变量：

	```
	struct D
	{
		static constexpr int n=5;
	}
	```

但这只在D::n不需要定义的时候才有效，例如：通过值传递：

	```
	std::cout << D::n;			//OK(ostream::operator<<(int) gets D::n by value)
	```

如果D::n 是通过引用传递到一个no-inline方法，调用没有优化掉，这是无效的。例如：

	```
	int inc(const int& i);

	std::cout << inc(D::n);   //usually an ERROR
	```

这段代码违反了one definition rule(ODR)原则。当使用编译器优化编译时，也许会像期望一样工作，或者缺少链接报错。当没有使用编译器优化时，几乎都会因为缺少定义报错。所以，c++17之前，再每个调用单元，都需要定义：

	```
	cosntexpr int D::n;			//c++11/c++14: definition
								//since c++17: redundant declaration(deprecated)
	```

当使用c++17编译时，类本身就是个定义。所以代码现在是有效的，没有前一个定义，这仍然是有效的，但已被弃用。

### 3.4 Inline Variables and thread_local
使用thread_local，可以为每个线程创建一个内联变量：

	```
	struct ThreadData
	{
		inline static thread_local std::string name;		//unique name per thread
		...
	};

	inline thread_local std::vector<std::string> cache;		//one cache per thread
	```

完整的例子，考虑一下下面的代码：

	```
	#include <string>
	#include <iostream>

	struct MyData
	{
		inline static std::string gName = "global";			  	//unique in program
		inline static thread_local std::string tName = "tls";	//unique per thread
		std::string lName = "local";							//for each object
		...
		void print(const std::string& msg) const 
		{
			std::cout << msg << '\n';
			std::cout << " - gName:" << gName << '\n';
			std::cout << "- tName:" << tName << '\n';
			std::cout << "- lName:" << lName << '\n';
		}
	};

	inline thread_local MyData myThreadData;		//one object per thread
	```

在main中 如下使用：

	```
	#include "inlinethreadlocal.hpp"
	#include <thread>

	void foo();

	int main()
	{
		myThreadData.print("main() begin:");
		myThreadData.gName = "thread1 name";
		myThreadData.tName = "thread1 name";
		myThreadData.lName = "thread1 name";
		myThreadData.print("main() later:");
		std::thread t(foo);
		t.join();
		myThreadData.print("main() end:");
	}
	```

在其他线程使用foo()函数：

	```
	#include "inlinethreadlocal.hpp"
	void foo()
	{
		myThreadData.print("foo() begin:");
		myThreadData.gName = "thread2 name";
		myThreadData.tName = "thread2 name";
		myThreadData.lName = "thread2 name";
		myThreadData.print("foo() end:");
	}
	```

输出：

	```
	main() begin:
	- gName: global
	- tName: tls
	- lName: local
	main() later:
	- gName: thread1 name
	- tName: thread1 name
	- lName: thread1 name
	foo() begin:
	- gName: thread1 name
	- tName: tls
	- lName: local
	foo() end:
	- gName: thread2 name
	- tName: thread2 name
	- lName: thread2 name
	main() end:
	- gName: thread2 name
	- tName: thread1 name
	- lName: thread1 name
	```

### 3.5 使用内联变量跟踪 ::new
下面的例子演示了如何只用header-only 方式跟踪`::new`:

	```
	#ifndef TRACKNEW_HPP
	#define TRACKNEW_HPP

	#include <new>
	#include <cstdlib>			//for malloc
	#include <iostream>

	class TrackNew
	{
	private:
		static inline int numMalloc = 0 ;		//num malloc calls;
		static inline long sumSize = 0;			//bytes allocated so for
		static inline bool doTrace = false;		//tracing enabled
		static inline bool inNew = false;		//don't track output inside new overloads
	public:
		//reset new/memory counters
		static void reset()
		{
			numMalloc = 0;
			sumSize = 0;
		}

		//enable print output for each new:
		static void trace(bool b)
		{
			doTrace = b;
		}

		//print current stateL
		static void status()
		{
			std::cerr << numMalloc << " mallocs for " << sumSize << " Bytes" << '\n';
		}

		//implementation of tracked allocation:
		static void* allocate(std::size_t size,const char* call)
		{
			//trace output might again allocate memory, so handle this the usual way:
			if(inNew)
				return std::malloc(size);

			inNew = true;
			//track and trace the allocation:
			++numMalloc;
			sumSize += size;
			void *p = std::malloc(size);
			if(doTrace)
			{
				std::cerr << "#" << numMalloc << " " << call << " (" << size << " Bytes) => " << p << "  (total: " << sumSize << " Bytes)" << '\n';
			}

			inNew = false;
			return p;
		}
	};

	inline void* operator new (std::size_t size)
	{
		return TrackNew::allocate(size,"::new");
	}

	inline void* operator new[](std::size_t size)
	{
		return TrackNew::allocate(size,"::new[]");
	}

	#endif   //TRACKNEW_HPP
	```

考虑下面的例子：

	```
	#include "tracknew.hpp"
	#include <string>

	class MyClass 
	{
		static inline std::string name = "initial name with 26 chars";
		...
	};
	MyClass myGlobalObj; // OK since C++17 even if included by multiple CPP files
	```

	```
	#include "tracknew.hpp"
	#include "tracknewtest.hpp"
	#include <iostream>
	#include <string>
	int main()
	{
		TrackNew::status();
		TrackNew::trace(true);
		std::string s = "an string value with 29 chars";
		TrackNew::status();
	}
	```

输出取决于什么时候初始化跟踪，和一共分配了多少内存。执行上面的例子，输出类似以下：
	
	```
	...
	#33 ::new (27 Bytes) => 0x89dda0 (total: 2977 Bytes)
	33 mallocs for 2977 Bytes
	#34 ::new (30 Bytes) => 0x89db00 (total: 3007 Bytes)
	34 mallocs for 3007 Bytes
	```

`MyClass::name`分配了27字节，`main()`中的`s`分配了30字节。(注意：通常带有SSO优化的库，字符串分配要超过15个字节，避免堆上内存不足的情况，在字符串上分配15个字节的内存，而不是在堆上分配内存)。

### 3.6后记
内联变量是David Krauss在[https://wg21.link/n4147][1]上的想法，被Hal Finkel 和Richard Smith 在[https://wg21.link/n4424]上提议，最终被Hal Finkel 和Richard Smith 在[ https://wg21.link/p0386r2][3]收录进标准库。

[1]:[https://wg21.link/n4147]
[2]:[https://wg21.link/n4424]
[3]:[https://wg21.link/p0386r2]