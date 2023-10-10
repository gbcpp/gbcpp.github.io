---
layout: post
title: BOOST 在 Windows 下通过 VS 进行编译
date: 2018-03-12
author: Mr Chen
cover: '/assets/img/boost_logo.webp'
#cover_author: 'rogov'
#cover_author_link: 'https://unsplash.com/@rogovca'
tags: 
- BOOST
---


# 下载 Boost 源码

<!--more-->

从官方下载 Boost 源码，之前用的一直是 boost 1.58.0，今天去官方网站看到最新的已经是 boost 1.66.0，最新版本的下载地址是：[下载地址](https://akamai.bintray.com/59/596389389c005814ecb2a6b64c31dccd2c3e6fbc5a802b4dfada999ae5844628?__gda__=exp=1520835412~hmac=6d6c97450b9b0b9421f696c922c483f87493fe9a10890fc7235a1c30c120f8e1&response-content-disposition=attachment%3Bfilename%3D%22boost_1_66_0.7z%22&response-content-type=application%2Fx-7z-compressed&requestInfo=U2FsdGVkX19TK62uO_jw5nC8JAtKkRu4cX28_lg3Z3y4xQkcbs9DqFSaTA0PDQKfCQ9Q94pRYGu60LzIyLyeN5mNvuokt1GI_MdVPpMBQLpNhTWhePsfBOrGtyhg6MOCF0JFr3mSugB0Sihk9fO5Wr1BR4MLQS9n78rtxF7JwNmR7sZ6on6X6jtu9UTYTMBy&response-X-Checksum-Sha1=075d0b43980614054b1f1bafd189f863bba6600e&response-X-Checksum-Sha2=596389389c005814ecb2a6b64c31dccd2c3e6fbc5a802b4dfada999ae5844628)

建议使用迅雷进行下载，通过浏览器下的比较慢。
下载后解压到自己的开源项目目录下，如：`E:\OpenSource\boost_1_66_0`

# Windows 下通过 Visual Studio 2015 编译

启动 VS2015 的 X86 本机工具命令提示符，即 `VS2015 x86 Native Tools Command Prompt` ，进入 boost 源码目录：`E:\OpenSource\boost_1_66_0`，然后执行以下命令：

~~~
    bootstrap.bat   //首先执行此命令生成 b2 编译工具
    bjam stage --toolset=msvc-14.0 --without-graph --without-graph_parallel --stagedir="D:\boost\boost_1_63_0\bin\vc14" link=static runtime-link=static runtime-link=static threading=multi debug release  
~~~

以上命令生成 32bit 静态库；如果希望生成 64bit，则使用以下命令：
~~~
bjam stage --toolset=msvc-14.0 architecture=x86 address-model=64 --without-graph --without-graph_parallel --stagedir="D:\boost\boost_1_63_0\bin\vc14-x64" link=static runtime-link=static runtime-link=static threading=multi debug release
~~~

# 相关编译选项

下面详细解释一下每个参数的含义：
- stage/install：
stage表示只生成库（dll和lib），install还会生成包含头文件的include目录。本人推荐使用stage，因为install生成的这个include目录实际就是boost安装包解压缩后的boost目录（`E:\OpenSource\boost_1_66_0\boost`，只比include目录多几个非hpp文件，都很小），所以可以直接使用，而且不同的IDE都可以使用同一套头文件，这样既节省编译时间，也节省硬盘空间。
 
- toolset：
指定编译器，可选的如borland、gcc、msvc（VC6）、msvc-14.0（VS2015）等。
 
- without/with：
选择不编译/编译哪些库。因为python、mpi等库我都用不着，所以排除之。还有wave、graph、math、regex、test、program_options、serialization、signals这几个库编出的静态lib都非常大，所以不需要的也可以without掉。这可以根据各人需要进行选择，默认是全部编译。但是需要注意，如果选择编译python的话，是需要python语言支持的，应该到 python [官方主页](http://www.python.org/)下载安装。查看boost包含库的命令是bjam --show-libraries。
 
- stagedir/prefix：
stage时使用stagedir，install时使用prefix，表示编译生成文件的路径。推荐给不同的IDE指定不同的目录。
 
- build-dir：编译生成的中间文件的路径。这个本人这里没用到，默认就在根目录（`E:\OpenSource\boost_1_66_0`）下，目录名为bin.v2，等编译完成后可将这个目录全部删除（没用了），所以不需要去设置。
 
- link：
生成动态链接库/静态链接库。生成动态链接库需使用shared方式，生成静态链接库需使用static方式。一般boost库可能都是以static方式编译，因为最终发布程序带着boost的dll感觉会比较累赘。
 
- runtime-link：
动态/静态链接C/C++运行时库。同样有shared和static两种方式，这样runtime-link和link一共可以产生4种组合方式，各人可以根据自己的需要选择编译。一般link只选static的话，只需要编译2种组合即可，即link=static runtime-link=shared和link=static runtime-link=static，本人一般就编这两种组合。
 
- threading：
单/多线程编译。一般都写多线程程序，当然要指定multi方式了；如果需要编写单线程程序，那么还需要编译单线程库，可以使用single方式。
 
- debug/release：
编译debug/release版本。一般都是程序的debug版本对应库的debug版本，所以两个都编译。

- address-model=32/64 	
寻址模式(生成32位还是64位库)

