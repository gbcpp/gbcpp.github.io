---
title: Linux 基于 webrtc 进行流媒体开发
tags: webrtc Linux Ubuntu g++
author: Mr Chen
date: 2019-01-23 10:56:31
categories: RTC
tags: 
- WebRTC
- OpenSource
- Linux
---


在 Linux 平台下开发实时通话 SDK 其实是主要应用于未来的 IOT 行业，先基于 Ubuntu 作为开发平台，完成后再基于每个客户提供的其交叉编译工具链进行交叉编译以供用户使用。

<!--more-->

## 基于 Ubuntu 下载 webrtc


### 自行下载编译
    
如果是自己下载编译的话，可以参考官方教程，或者参考本人之前的一篇博客： https://www.jianshu.com/p/09f065f3feb0 ，其实 Ubuntu 下编译 webrtc 是比较简单的，只要可以翻墙，其它都只是时间问题了。


### 无法翻墙下载

那么可以通过下载本人配置的一个 Docker 镜像进行进行编译： https://hub.docker.com/r/cgb0210/ubuntu_build ，由于 Docker 不适合作为开发环境，下载后，建议用户将 develop 目录下的文件拷贝到自己的开发环境中，并参考 https://www.jianshu.com/p/09f065f3feb0 配置相关环境变量，实现本地编译,过程主要包含 3 步：

### 配置 depot_tools 环境变量

将 `export PATH=$PATH:/home/gobert/develop/depot_tools` 添加到 `/etc/profile` 文件尾部；

### 安装 GoLang 并配置环境

### 安装 Linux 依赖包

执行 webrtc 目录下的 `/build/install-build-deps.sh` 脚本命令，如： `sudo sh build/install-build-deps.sh`；


## 编译 webrtc 

默认情况下，Linux 下基于 ninja 编译的话，其编译器使用的是 clang，且其依赖的 stdc 标准库为其 ./buildtools/third_party/libc++ 内置的，所以如果外部想要基于 clang 编译的 libwebrtc.a 进行开发的话，上层依然需要使用 clang 做为编译器，且链接 webrtc 内部的 libc++ 标准库，否则将会产生各种 `error: undefined reference to symbol ...`，无法解决，虽然 clang 比 g++ 要优秀很多，但本人目前还是习惯使用 g++ 进行开发，所以需要指定 g++ 编译器，同时需要注意以下事项：

### RTTI （Run-Time Type Identification 运行时类型识别）

webrtc 内部各模块配置了不同的 rtti 属性，开、关各不相同，默认情况下，加载 libwebrtc.a 会产生 `undefined reference to `typeinfo for [classname]'` 错误，为此我们可以在链接 libwebrtc.a 的根编译脚本文件中配置统一去除 rtti 属性，以解决此类问题，打开根目录下的 BUILD.gn 文件，在 `rtc_static_library("webrtc")` 尾部添加 `cflags_cc = [" -fno-rtti" ]` 即可，如下：

~~~
if (!build_with_chromium) {
# Target to build all the WebRTC production code.
rtc_static_library("webrtc") {
    
    ...
    
    cflags_cc = [" -fno-rtti" ]
    }
}
~~~

这样便可以解决所有 undefined reference to `typeinfo for [classname]'` 类的错误。

### 找不到 builtin_audio_decoder_factory 模块的所有符号

这应该是 webrtc branch：68 版本的 bug，在各平台均会出现此类问题，原因在于其在编译脚本中漏写了 `builtin_audio_decoder_factory` 模块，加上即可，打开 `audio/BUILD.gn` 文件，在 `"../api/audio_codecs:builtin_audio_encoder_factory",` 下面添加一行 `"../api/audio_codecs:builtin_audio_decoder_factory",` 即可。

### 编译

编译命令如下：

~~~
gn gen -C out/linux/Release  --args="is_debug=false target_cpu=\"x64\" rtc_include_tests=false rtc_use_h264=true rtc_initialize_ffmpeg=true ffmpeg_branding=\"Chrome\" is_component_build=false is_clang=false treat_warnings_as_errors=false use_custom_libcxx=false strip_debug_info=true use_rtti=false"
~~~
其中 `rtc_include_tests=false` 为禁止编译单元测试程序，可大大节省编译时间；
`rtc_use_h264=true rtc_initialize_ffmpeg=true ffmpeg_branding=\"Chrome\"` 为激活内部 H.264 编解码器。


## Demo 程序

### 编译测试程序加载并调用 webrtc 接口。

~~~cpp
#include <stdio.h>
#include <string>

#include "api/jsep.h"

using namespace std;
using namespace webrtc;

int main(int argc, char const *argv[])
{
    SdpParseError error;
    auto ptr = CreateSessionDescription("offer", "sdp", &error);
    
    if (!ptr) {
        printf("CreateSessionDescription return null, error str:%s\n", error.description.c_str());
    } else {
        printf("CreateSessionDescription success!\n");
    }

    return 0;
}
~~~

### 编写 CMakeLists.txt 脚本文件

~~~
cmake_minimum_required(VERSION 3.5)
project(demo)

set(CMAKE_POSITION_INDEPENDENT_CODE     TRUE)

set(CMAKE_C_FLAGS               "${CMAKE_C_FLAGS}")
SET(CMAKE_CXX_FLAGS             "${CMAKE_CXX_FLAGS} -fno-exceptions -fPIC")
set(CMAKE_SHARED_LINKER_FLAGS   "${CMAKE_SHARED_LINKER_FLAGS}")
set(CMAKE_EXE_LINKER_FLAGS      "${CMAKE_EXE_LINKER_FLAGS}")

set(ARCH_PATH linux)
set(WEBRTC_PATH "/home/gobert/develop/webrtc/src")
set(WEBRTC_LIBRARY_PATH "/home/gobert/develop/webrtc/src/out/Linux/Release/obj")

add_definitions(
    "-DWEBRTC_POSIX"
    "-DWEBRTC_LINUX"
    "-DUSE_GLIB=1"
)

if (Linux)
set(ARCH_PATH linux)
elseif (Arm)
set(ARCH_PATH arm)
endif ()

include_directories(
    ${WEBRTC_PATH}
    )

set(SOURCE_FILES
    main.cpp
)

link_directories(
    ${link_directories}
    ${WEBRTC_PATH}/out/Linux/Release/obj
    /usr/lib/x86_64-linux-gnu/
)

link_libraries(
    "${WEBRTC_LIBRARY_PATH}/libwebrtc.a"
    )

add_executable(demo ${SOURCE_FILES})

TARGET_LINK_LIBRARIES(demo pthread stdc++)

set_target_properties(demo PROPERTIES OUTPUT_NAME "demo")

~~~

### 编译

~~~
cmake && make
~~~
