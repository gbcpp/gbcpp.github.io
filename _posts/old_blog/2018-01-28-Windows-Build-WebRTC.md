---
layout: post
title: Windows 下编译 WebRTC
date: 2018-01-28 23:58:14
author: Mr Chen
categories: RTC
tags:
- RTC
---

# 一、系统环境准备

## 1、参考官方文档

~~~
https://webrtc.org/native-code/development/   //官方编译指导首页
https://webrtc.org/native-code/development/prerequisite-sw/  //依赖工具安装指导页
http://dev.chromium.org/developers/how-tos/install-depot-tools //用于下载代码的工具包安装指导页面
~~~
<!--more-->

## 2、Windows 10 & MingW & VS2015

如果已经安装了 cygwin，则建议先将其重命名为 cygwin_bak，使其目录内的工具无效，因为 WebRTC 官方文档已经明确不推荐使用 CygWin 了，否则编译过程中可能会发生各种奇奇怪怪的错误，原文如下：

~~~
Chromium is mostly designed to be run using the native Windows tools and the Msys (Git for Windows) toolchain. Cygwin is not recommended, and likely things will fail in cryptic ways.
~~~

自行下载并安装 MingW。

自行下载安装 VS2015，因为最新的 webrtc 使用到了 C++ 11的一些新特性，如果使用 VS2013 等较旧的一些 IDE，可能会由于 对 C++ 11 支持的不够完整，而产生编译错误。

## 3、安装 python
下载并安装 python，并将其目录配置在环境变量中，一定要安装 Python 2.7 版本的，太高或者太低均会报错！

安装完成后，将 Python 所在目录添加到 PATH 环境变量中，并通过
~~~
set PTATH=C:
exit
~~~
使环境变量生效，确认 Python 是否已经正确配置，执行命令 
~~~
where python
~~~
如果可以有效输出 python.exe 所在目录，则配置正确。

## 4、安装 depot_tools

安装 WebRTC 代码下载工具 depot_tools（Google），参考页面：
~~~
http://dev.chromium.org/developers/how-tos/install-depot-tools
~~~
，Windows 下设置环境变量，需将 depot_toolss 所在目录添加到环境变量 ‘PATH’ 的最前面（目录中不要有空格），记得在标准的 cmd 中执行下 

~~~
gclient
~~~

命令，首次启动，它会自动下载安装所依赖的一些工具，**但切记不要在非标准的 cmd 中执行此命令，如 cmder**；

## 5、下载安装 Windows SDK 10

官方下载地址 [Windows10 sdk](https://developer.microsoft.com/zh-cn/windows/downloads/windows-10-sdk)

并配置环境变量 WINDOWSSDKDIR 指向 SDK 安装目录，如

~~~
WINDOWSSDKDIR=D:\Windows Kits\10
~~~

## 6、修改系统语言

首先更改 Windows 系统区域语言为  英语，具体方法如下：

~~~
Control Panel → System and Security → System → Advanced system settings
中文：
控制面板->时钟、语言和区域->区域->管理（选项卡）->更改系统区域设置； 并重启
~~~

## 7、 设置默认编译工具 IDE 版本

系统环境变量中添加变量 DEPOT_TOOLS_WIN_TOOLCHAIN ，值设为 0；这个环境变量作用是告诉 deppt_tools 使用本地已安装的默认的 Visual Studio 版本去编译；否则 depot_tools 会使用 Google 内部默认的版本；

##  8、设置环境变量，用于生成 VS 工程文件

~~~
set DEPOT_TOOLS_WIN_TOOLCHAIN=0
set GYP_GENERATORS=ninja,msvs-ninja
set GYP_MSVS_VERSION=2015
~~~

## 9、安装 DirectX SDk 

安装DirectX SDK June 2010，安装完成后可能会报错，错误代码“s1023”,这是因为与系统已有的visual c++ redistributable packages版本冲突，不用管它，直接退出安装程序即可。这里我们需要的只是安装目录下的头文件和库。

#二、 下载 WebRTC 源码

注：CMD 命令行窗口使用管理员权限打开；

## 1、下载源码

选定好 WebRTC 源码存放目录后，如 E:/OpenSource 目下，通过 CMD 进入此目录，依次执行以下命令：

~~~
mkdir webrtc-checkout
cd webrtc-checkout
fetch --nohooks webrtc
gclient sync
~~~
前三条命令一次获取 WebRTC 源码，需要下载较长的时间。

如果在执行 gclient sync 命令时，发现以下错误信息：

~~~
Please follow the instructions at https://www.chromium.org/developers/how-tos/build-instructions-windows                                                                          
***
returned non-zero exit status 1 in E:\OpenSource\webrtc-checkout                          
~~~

仔细查看报错信息，可以看到，命令执行失败，并建议我们先参考

~~~
https://www.chromium.org/developers/how-tos/build-instructions-windows 
~~~
指导页完成 Windows 下的一些编译准备工作，可能是漏掉了一些前期准备工作，主要是

**更改 Windows 系统区域语言为  英语** 

具体方法如下：

~~~
Control Panel → System and Security → System → Advanced system settings
中文：
控制面板->时钟、语言和区域->区域->管理（选项卡）->更改系统区域设置； 并重启
~~~

**系统环境变量中添加变量 DEPOT_TOOLS_WIN_TOOLCHAIN ，值设为 0** 

这个环境变量作用是告诉 deppt_tools 使用本地已安装的默认的 Visual Studio 版本去编译；否则 depot_tools 会使用 Google 内部默认的版本；

至此环境已经彻底准备好，重新执行
~~~
gclient sync
~~~
命名执行成功！

## 2、 可选项：指定如何跟踪处理新的分支

  ~~~
  git config branch.autosetupmerge always
  git config branch.autosetuprebase always
  ~~~

  **git config branch.autosetupmerge always**

  表示 自动从远程分支合并提交到本地分支，如果不做此设置，你需要手动 merge；

  **git config branch.autosetuprebase always**

  设置在执行 git pull  命令时做 rebase 而不是是 merge；可选的值还有 never：不自动 rebase， local 跟踪本地分支进行 rebase， remote 跟踪远程分支进行 rebase， always 所有跟踪的分支都自动进行 rebase；  ​

## 3、 可选项：创建一个新的本地分支

  ~~~
  cd src
  git checkout master
  git new-branch your-branch-name
  ~~~

  建议创建下新的本地开发分支，如： git new-branch dev  ​

* 编译前更新下最新代码

  ~~~
  git pull
  ~~~

  **注：**如果在上一步中，没有创建新的分支，则使用

   ~~~
  git fetch
   ~~~

  命令代替，以获取最新代码。

**注：**由于WebRTC 目前还在频繁的更新中，建议定期的去下载更新下编译工具链及其依赖，通过执行
gclient sync 既可!

# 三、编译

WebRTC  目前使用 GN 来生成构建脚本，Ninja 进行构建，所以系统平台均是。

所以网上说的通过 GYP 生成 VS 解决方案工程文件的博文都已失效，用的均为旧版本的 webrtc。

## 1、通过 Ninja 编译

### a、 生成 Ninja 工程文件

Ninja 工程文件由 GN 生成，为其选择一个放置的目录中，如 out/Debug 或者 out/Release，这里官方建议选择 out/Default 这样可以放置 debug 和 release，在 src 目录下还行一下命令：

~~~
gn gen out/Default
~~~

如果 提示 gn 命令 not found，需检查 depot_tools 环境变量设置是否生效。

执行以下命令生成 Ninja 工程文件

~~~
gn gen out/Default
~~~

如果需要生成 release 工程文件，需在后面加上关闭 debug 的参数  **--args=is_debug= false**

如果需要清理 Ninja 工程文件，但保持 GN 环境配置不变的话，可以执行以下命令：

~~~
gn clean out/Default
~~~

### b、通过 Ninja 命令编译

~~~
ninja -C out/Default
~~~

## 2、通过 VS IDE 编译

除了 ninja 外，其他的构建系统不受支持(可能会失败)，比如Windows上的Visual Studio或者OSX上的Xcode。GN支持一种混合使用的方法，即由 Visual studio/xcode 用于编辑和驱动编译。

官方原文如下：

Other build systems are **not supported** (and may fail), such as Visual Studio on Windows or Xcode on OSX. GN supports a hybrid approach of using [Ninja](https://ninja-build.org/) for building, but Visual Studio/Xcode for editing and driving compilation.

### a、生成 VS 解决方案工程文件
src 目录下执行以下命令（默认生成 Debug ）：
~~~
gn gen --ide=vs out/Debug
~~~

会在 out/VS 目录下生成 all.sln 解决方案文件；如果需要生成 release 工程文件，需在后面加上关闭 debug 的参数  **--args='is_debug= false'**
即：
~~~
gn gen --ide=vs out/Release --args=is_debug= false
~~~

### b、打开 all.sln 文件编译

