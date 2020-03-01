---
layout: post
title: "cpp complete guide 2"
subtitle: "cpp complete guide直译"
excerpt: "&ensp;&ensp;&ensp;&ensp;在c++17中，if 和switch 允许我们在条件(if)或者选择条件(switch)声明中使用变量初始化。"
date: 2020-03-01  +08:00
categories: [cpp complete guide]
author: "kcwl"
comments: true
---

# Chapter 2
## 初始化if和switch


在c++17中，if 和switch 允许我们在条件(if)或者选择条件(switch)中使用变量初始化。
	现在，可以这样写：

	```
	if(status s = check(); s != status::success)
	{
		return s;
	}
	```

s的生命周期等同于if 的生命周期。




### 2.1 初始化if
任何在if中使用初始化的值，在else或者then结束前都有效。

	```
	if (std::ofstream strm = getLogStrm(); coll.empty()) 
	{
		strm << "<no data>\n";
	}
	else 
	{
		for (const auto& elem : coll) 
		{
			strm << elem << ’\n’;
		}
	}
	// strm no longer declared
	```

在else或者then执行之后，strm的析构函数就会被调用。另外一个例子是根据某些条件来说用锁：

	```
	if(std::lock_guard<std::mutex> lg{collMutex};!coll.empty())
	{
		std::cout << coll.front() << '\n';
	}
	```

因为模版推导，也可以写成这样：

	```
	if (std::lock_guard lg{collMutex}; !coll.empty()) 
	{
		std::cout << coll.front() << ’\n’;
	}
	```

不管怎么写，都等同于：

	```
	{
		std::lock_guard<std::mutex> lg{collMutex};
		if (!coll.empty()) 
		{
			std::cout << coll.front() << ’\n’;
		}
	}
	```

略有不同的是，因为lg是定义在if声明中的，所以它的生命周期只在if内，就像是for循环的初始化一样。

任何对象初始化都要有自己的名字。否则，当初始化成功之后，就会进行析构。例如：初始化一个没有名字的lock guard，当条件达成时，锁不会生效：
	
	```
	if (std::lock_guard<std::mutex>{collMutex};  		// run-time ERROR:
		!coll.empty())
	{ 													// - no longer locked
		std::cout << coll.front() << ’\n’; 				// - no longer locked
	}
	```

原则上,'\_'作为变量也是可以的，但是并不是所有人都喜欢这样的命名：

	```
	if (std::lock_guard<std::mutex> _{collMutex}; 		// OK, but...
		!coll.empty()) 
	{
		std::cout << coll.front() << ’\n’;
	}
	```

第三个例子：向一个unordered map中插入数据,使用if 判断是否插入成功

	```
	std::map<std::string, int> coll;
	...
	if (auto [pos,ok] = coll.insert({"new",42}); !ok) 
	{
		// if insert failed, handle error using iterator pos:
		const auto& [key,val] = *pos;
		std::cout << "already there: " << key << ’\n’;
	}
	```

c++17之前，对应的写法：

	```
	auto ret = coll.insert({"new",42});
	if (!ret.second)
	{
		// if insert failed, handle error using iterator ret.first
		const auto& elem = *(ret.first);
		std::cout << "already there: " << elem.first << ’\n’;
	}
	```

### 2.2 初始化switch
在switch声明中初始化新变量，可以让我们决定在哪里开始控制switch的选择。比如：根据路径的类型输出相应的路径

	```
	using namespace std::filesystem;
	...
	switch(path p(name);status(p).type())
	{
		case file_type::not_found:
			std::cout << p << "not found\n";
			break;
		case file_type::directory:
			std::cout << p << ":\n";
			for(auto& e : std::filesystem::directory_iterator(p))
			{
				std::cout << "-" << e.path() << '\n';
			}
			break;
		default:
			std::cout << p << "exusts\n";
			break;
	}
	```

p变量在整个swtich的声明都可以使用。

### 2.3 后记
带有初始化的if 和 switch是由Thomas Koppe 在[https://wg21.link/p0305r0][1]上提出的， 最初只是扩展if，最后Thomas Koppe 在[https://wg21.link/p0305r1][2]中完善了这个提议。


[1]:https://wg21.link/p0305r0
[2]:https://wg21.link/p0305r1