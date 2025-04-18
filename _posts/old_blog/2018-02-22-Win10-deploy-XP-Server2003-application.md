---
layout: post
title: Windows 10 开发部署支持 Windows XP 和 Windows Server 2003 的应用程序
author: Mr Chen
date: 2018-02-22 11:47:18
categories: Tech
tags:
- Windows
---

# 场景

在 Windows 10 下通过 VS2015 开发的桌面程序，部署到 Windows Server 2003 系统中，运行不起来，运行会报 “不是有效的 Win32 应用程序” 或者是 “不能定位 XXX.XXX 到 api-ms-win-...-...-l1-1-0.dll”，这里原因有两个，一个是在编译时默认指定的平台工具集系统版本较高，另一个就是编译时依赖了只有 Windows Vista 以上版本的系统才支持的 UCRT 动态库，对此，只要按照以下方法重新编译下，即可解决低版本系统下运行的问题。

<!--more-->

# 解决低版本系统报错： “不是有效的 Win32 应用程序”

将工程的平台工具集即 `Platform Toolset` 由 `Visual Studio 2015 (v140)` 改为 `Visual Studio 2015 - Windows XP (v140_xp)`；

重新编译后，通过 ExeScope 依次查看 `头部`--`可选头部`--`操作系统主版本` 发现其值为 0005，即说明该目标文件可以运行在 Windows version 为 5.0 以上的系统中了。

# 解决 “无法定位...到 ucrt 库”

VC++ 需要去除通用 CRT（UCRT）库的依赖。

## 什么是 UCRT 库

首先介绍下什么是 CRT 库：

`C 运行时库 (CRT) 是集成了 ISO C99 标准库的 C++ 标准库。 实现 CRT 的 Visual C++ 库支持用于 .NET 开发的本机代码开发、本机和托管混合代码以及纯托管代码。 所有版本的 CRT 都支持多线程开发。 大多数的库都支持通过静态链接将库直接链接到代码中，或通过动态链接让代码使用常用 DLL 文件。`

UCRT（通用 CRT） 包含通过标准 C99 CRT 库导出的函数和全局函数。 UCRT 现为 Windows 组件，并作为 Windows 10 的一部分提供。 静态库、DLL 导入库和 UCRT 的头文件现在 Windows 10 SDK 中提供。 安装 Visual C++ 时，Visual Studio 安装程序将安装使用 UCRT 所需 Windows 10 SDK 的子集。 可以在 Visual Studio 2015 支持的任何 Windows 版本上使用 UCRT。 可以使用 vcredist 重新分发它，以便支持 Windows 10 以外的 Windows 版本。

在 Visual Studio 2015 中，CRT 已重构为新的二进制文件。

Windows 下所有的 ucrt 库均存放在 `C:\Program Files (x86)\Windows Kits\10\Redist\ucrt` 目录下，而这些库是只有 Windows Vista 以上版本的系统才支持的，所以不能依赖此动态库，解决方法便是在编译程序时，将所有的工程的 `属性`--`C/C++`--`Code Generation`--`Runtime Library` 改为 `Multi-threaded (/MT)` 即可，重新编译后，通过 Stud_PE 查看其依赖项已经没有了 api 开发的 curt 动态库，如：
~~~
api-ms-win-crt-runtime-l1-1-0.dll
api-ms-win-crt-stdio-l1-1-0.dll
api-ms-win-crt-math-l1-1-0.dll
api-ms-win-crt-locale-l1-1-0.dll
api-ms-win-crt-heap-l1-1-0.dll
~~~
**注意：这里是可以将工程的运行时库编译选项改为 MT 的情况下，如果你的工程不能改为 MT 那就只能将 UCRT 库全部带上了，但是如果是 UCRT 的动态库的话，是不支持 Windows XP 和 Server 2003 一下版本的系统的！**

关于 CRT 库的更多的介绍，请参考微软官方的文档：https://msdn.microsoft.com/zh-cn/library/abx4dbyh.aspx