---
layout: post
title: "cpp complete guide 19"
subtitle: "cpp complete guide直译"
excerpt: "&ensp;&ensp;&ensp;&ensp;由于c++17的推进，boost.filesystem库最终被c++标准库收录。经过调整，使得该库和标准库一致，并且增加了一些扩展方法(例如：在filesystem paths中相对路径的操作)。"
date: 2020-03-01  +08:00
categories: [cpp complete guide]
author: "kcwl"
comments: true
---


# Chapter 19
## The Filesystem Library


由于c++17的推进，boost.filesystem库最终被c++标准库收录。经过调整，使得该库和标准库一致，并且增加了一些扩展方法(例如：在filesystem paths中相对路径的操作)。

### 19.1 基础例子

先来一些基础例子：

**通过filesystem path 输出属性** 
	下面的示例允许通过一个字符串来打印相关路径文件的类型：

	```
	#include <istream>
	#include <filesystem>

	int main(int argc, char* argv[])
	{
		if(argc<2)
		{
			std::cout << "Usage: " << argv[0] << " <path> \n";
			return 0;
		}

		std::filesystem::path p(argv[1]);			//p 是一个文件系统路径(也许不存在)
		if(!exists(p))
		{
			std::cout << "path " << p << " does not exist\n";
		}
		else
		{
			if(is_regular_file(p))				//如果是普通文件
			{
				std::cout << p << " exists with " << file_size(p) << "bytes\n";
			}
			else if(is_directory(p)) 			//如果是目录
			{
				std::cout << p << " is a derectory containing:\n";
				for(auto& e : std::filesystem::directory_iterator(p))
				{
					std::cout << "   "  << e.path() << '\n';
				}
			}
			else
			{
				std::cout << p << " exists,but is no regular file or directory\n";
			}
		}
	}
	```

首先，检查一下，传递过来的字符串是不是一个已经存在的文件路径：

	```
	std::filesystem::path p(argv[1]); // p represents a filesystem path (might not exist)
	if (!exists(p)) // does path p actually exist?
	{ 
		...
	}
	```

如果是，继续运行：
1. 如果是一个普通的文件，输出文件大小

	```
	if (is_regular_file(p)) 	// is path p a regular file?
	{ 
		std::cout << p << " exists with " << file_size(p) << " bytes\n";
	}
	```

执行程序:
	checkpath checkpath.cpp

会有下面类似的输出：
	"./checkpath.cpp" exists with 897 bytes

2. 如果是一个目录，遍历目录下的文件，然后输出：

	```
	if (is_directory(p)) // is path p a directory?
	{ 
		std::cout << p << " is a directory containing:\n";
		for (auto& e : std::filesystem::directory_iterator(p)) 
		{
			std::cout << " " << e.path() << ’\n’;
		}
	}
	```

这里使用directory_iterator，迭代器提供了begin()和end()方法，可以通过使用一个范围循环遍历元素。这种情况下，我们使用path()方法，输出文件的系统路径。
	
执行程序:
	checkpath .

会有下面类似的输出:

	```
	"." is a directory containing:
	"./checkpath.cpp"
	"./checkpath.exe"
	```

**创建不同种类的文件**  
下面的例子演示了在/tmp下创建不同种类的文件：

	```
	#include <iostream>
	#include <fstream>
	#include <filesystem>

	int main()
	{
		namespace fs = std::filesystem;
		try
		{
			//create /tmp/test:
			fs::path tmpDir("/tmp");
			fs::path testPath = tmpDir / "test";
			if(!create_derectory(testPath))
			{
				std::cout << "test directory already exists" << '\n';
			}

			//create link to /tmp/test
			create_symlink(testPath,fs::path("testdir.link"));

			//crate data file /tmp/test/data.txt
			testPath /= "data.txt";
			std::ofstream dataFile(testPath.string());
			if(dataFile)
			{
				dataFile << "this is my data\n";
			}

			//recursively list all files(also following symlinks)
			auto allopt = fs::directory_options::follow_directory_symlink;
			for(auto& e : fs::recursive_directory_iterator(".",allopt))
			{
				std::cout << e.path() << '\n';
			}
		}
		catch(fs::filesystem_error& e)
		{
			std::cerr << "exception: " << e.what() << '\n';
			std::cerr << "     path1:" << e.path1() << '\n';
		}
	}
	```

首先，我们把std::filesystem声明成一个简短的namespace：
	```
	namespace fs = std::filesystem;
	```

然后，初始化2个路径，使用 /操作符定义/tmp路径下的子文件夹 /tmp/test：
	
	```
	fs::path tmpDir("/tmp");
	fs::path testPath = tmpDir / "test";
	```

接着，创建了一个/tmp/test的快捷方式

	```
	create_symlink(testPath, fs::path("testdir.lnk"));
	```

然后在/tmp/test下面创建一个带有内容的文件/tmp/test/data.txt:

	```
	testPath /= "data.txt";
	std::ofstream dataFile(testPath.string());
	if (dataFile) 
	{
		dataFile << "this is my data\n";
	}
	```

然后使用 '/=' 扩展路径，并且使用string()返回一个对应string参数的字符串，最后，我们遍历当前的文件夹

	```
	auto allopt = fs::directory_options::follow_directory_symlink;
	for (auto& e : fs::recursive_directory_iterator(".", allopt)) 
	{
		std::cout << " " << e.path() << ’\n’;
	}
	```

因为使用了递归迭代器，使用选项，遍历快捷方式指向的文件夹，得到了以下输出：

	```
	"./createfiles.cpp"
	"./createfiles.exe"
	"./testdir.lnk"
	"./testdir.lnk/data.txt"
	```

根据系统和权限的不同，可能会遇到一些问题。对于那些没有返回值的，捕捉相应的异常，并打印抛出异常的第一个文件的路径。

	```
	try 
	{
		...
	}
	catch (fs::filesystem_error& e) 
	{
		std::cerr << "exception: " << e.what() << ’\n’;
		std::cerr << " path1: " << e.path1() << ’\n’;
	}
	```