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

	上述例子，如果没有成功的创建目录，就会抛出异常:

	```
	exception: filesystem error: cannot create directory: [/tmp/test]
								 path1: "/tmp/test"
	```

**切换文件系统类型**  
下面的代码演示了如何对给出的文件名字，根据不同的文件类型表现不同的行为：

filesystem/switchinit.cpp
	
	```
	#include <string>
	#include <iostream>
	#include <filesystem>
	namespace std 
	{
		using namespace std::experimental;
	}

	void checkFilename(const std::string& name)
	{
		using namespace std::filesystem;

		switch(path p(name);status(p).type())
		{
			case file_type::not_found:
				std::cout << p << "not found\n";
				break;
			case file_type::directory:
				std::cout << p << "\n";
				for(auto& entry : std::filesystem::derectory_iterator(p))
				{
					std::cout << "- " << entry.path() << '\n';
				}
				break;
			default:
				std::cout << p << "exsits\n";
		}
	}

	int main()
	{
		for(auto name : {".","ifinit.cpp","nofile"})
		{
			checkFilename(name);
			std::cout << '\n';
		}
	}
	```

这个例子使用了带有初始化声明的`swtich`去构造文件类型：

	```
	switch(path p(name);status(p).type())
	{
		...
	}
	```

`status(p).type()` 创建了一个文件状态，`type()`方法创建了文件类型。如果不关注其中的状态或者状态，可以直接忽略掉他们，不进行调用。

### 19.2 原则和术语
在介绍`filesystem`库之前，必须要说明一些设计原则和术语。这些原则和术语规定了不同操作系统如何使用共同API。

#### 19.2.1 可移植性免责声明
c++ 标准不仅标准化了所有系统的文件系统的相同点，在很多时候，它遵循`POSIX`标准，并且c++标准要求尽可能的接近`POSIX`。只要合理的事情就应该存在，虽然有一些局限性。如果不合理的事情出现了，那么就会产生错误，例如：
+ 文件名的字符不支持
+ 创建了不受系统支持的文件元素

具体的文件系统会产生差异：
+ **大小写敏感**
	"hello.txt","Hello.txt","hello.TXT"可能代表3种不同的文件
+ **绝对路径与相对路径**  
	在一些`POSIX-based`的系统中，"/bin"是绝对路径，在windows中不是

#### 19.2.2 命名空间
在标准库中，文件系统有自己的命名空间，通常都会引入别名：

	```
	namespace fs = std::filesystem
	```

可以通过别名调用命名空间中的方法，下面的例子都使用别名。

### 19.3 Paths
`path`是文件系统里面最重要的类,它表现了文件系统中的文件的位置。他包括根目录的名字，根目录，以及根目录下的所有文件。路径可以是绝对的，也可以是相对的。
+ 通用的可移植格式
+ 特定系统下的原生模式

例如：在windows下，通用格式的`/tmp/test.txt`就会映射到`\tmp\test.txt`, OpenVMS会映射到`[tmp]test.txt`。
还有些特殊的目录：
+ "." 表示当前目录
+ "." 表示上一级目录

通用格式如下：
	
	```
	[rootname][rootdir][releativepath]
	```

where:
+ 在windows中，rootname 可以是C: ，在POSIX下，可以是//host
+ rootdir 是个目录分隔符
+ 相对路径是由路径分割符分割的一个序列
根据定义，路径可以有一个'/'或多个，也可以是由特定的起始符'//'
通用的路径例子有：

	```
	//host1/bin/hello.txt
	.
	tmp/
	/a/b//../c
	```

最后一个例子，在windows下是相对路径，在POSIX下是绝对路径(因为对于windows路径来说，缺少分区)

另一种情况，在windows中C:/bin是个绝对路径，而在POSIX下，是相对路径(bin是C:的子目录)

在windows中，反斜杠'\'也可以表示目录分隔符

	```
	\\host1\bin\hello.txt
	.
	tmp\
	\a\b\..\c
	```

路径可能是空的。这说明没有路径，和"."表示的意思不同。

#### 19.2.4 标准化
在一个常规路径中：
+ 路径名中只有一个前置分隔符
+ "."表示单独的文件名(代表当前路径)
+ 如果不是相对路径，不能出现".."
+ 文件名末尾只能是文件分隔符，而不是"."或者".."

注意：规范化路径区分了以分隔符结尾的文件夹名字和不以分隔符结尾的文件名字。因为在不同的操作系统中，不同的名字表现行为不同(例如：带有分隔符的链接可能就会被解析)。
`Effect of Path Normalization`列表展示了一些windows和POSIX系统的例子。 在POSIX系统中，C:bar和C:都只是文件名字。不像在windows系统中有特殊的意义。

			Path 			   | POSIX normalized | Windows normalized
			:-:  			   | :-:              | :-:
			foo/.///bar/../	   | foo/			  | foo\
			//host/../foo.txt  | //host/foo.txt   | \\host\foo.txt
			./f/../.f/		   | .f/              | .f\
			C:bar/../		   | C:/           	  | C:
			C:/bar/..		   | C:/			  | C:\
			/./../data.txt 	   | /data.txt 		  | \data.txt
			././			   | .				  | .

			Table 19.1  Effect of Path Normalization
### 19.3 许多细节正在完善
### 19.4 后记
在Beman Dawes 的领导下，文件系统库作为boost库已经开发了很多年。2014年被c++标准库收录，作为测试版本，`File System Technical Specification`(见[https://wg21.link/n4100][1]).
在[https://wg21.link/p0218r0][2]中，Beman Dawes提议`File System Technical Specification`。Beman Dawes，Nicolai Josuttis 和Jamie Allsop 在[https://wg21.link/p0219r1][3]中提示支持相对路径。一些细小的错误修复由Beman Dawes在[https://wg21.link/p0317r1][4]上、Nicolai Josuttis在[https://wg21.link/p0392r0][5]上、Jason Liu和Hubert Tong在[https://wg21.link/p0430r2][6]上提出，尤其是文件系统小组的成员（Beman Dawes，S. Davis Herring，Nicolai Josuttis，Jason Liu，Billy O’Neal，P.J. Plauger和Jonathan Wakely）在[https://wg21.link/p0492r2][7]上提出的修复。

[1]:[https://wg21.link/n4100]
[2]:[https://wg21.link/p0218r0]
[3]:[https://wg21.link/p0219r1]
[4]:[https://wg21.link/p0317r1]
[5]:[https://wg21.link/p0392r0]
[6]:[https://wg21.link/p0430r2]
[7]:[https://wg21.link/p0492r2]


---
**文件系统库相关细节介绍**  
