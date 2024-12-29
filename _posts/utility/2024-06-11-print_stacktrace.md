---
layout: post
title: 'C++ 程序输出当前堆栈'
date: 2024-06-11
author: Mr Chen
# cover: '/assets/img/shan.jpg'
#cover: 'https://images.unsplash.com/photo-1653629154302-8687b83825e2'
#cover_author: 'rogov'
#cover_author_link: 'https://unsplash.com/@rogovca'
tags: 
- C++
---

将当前程序运行时的堆栈信息输出到控制台，或者 Log 中，用于问题调查。
互联网中找到的，使用均有内存问题，以下是修复后的版本，可正常运行在 Linux 下。

## 头文件

~~~cpp
#include <cxxabi.h>
#include <execinfo.h>
#include <stdio.h>
#include <stdlib.h>
~~~

## 源码

~~~cpp
std::string Utils::PrintStackTrace() {
  static constexpr size_t kMaxFrames = 128;
  static constexpr size_t kMaxPrettyFunctionNameLength = 512;

  std::ostringstream oss;
  void *addrlist[kMaxFrames + 1];

  // 获取当前堆栈中的地址
  int addrlen = backtrace(addrlist, sizeof(addrlist) / sizeof(void *));

  if (addrlen == 0) {
    return "";
  }
  oss << "StackTraces: " << std::endl;

  // 解析地址为函数名
  char **symbollist = backtrace_symbols(addrlist, addrlen);

  // 解析函数名的更多信息
  size_t funcNameSize = kMaxPrettyFunctionNameLength;
  char *funcName = static_cast<char *>(malloc(funcNameSize));

  // 跳过第一个元素（即本身）
  for (int i = 1; i < addrlen; i++) {
    char *beginName = nullptr;
    char *beginOffset = nullptr;
    char *endOffset = nullptr;

    // 解析符号名称及偏移量
    for (char *p = symbollist[i]; *p; ++p) {
      if (*p == '(') {
        beginName = p;
      } else if (*p == '+') {
        beginOffset = p;
      } else if (*p == ')' && beginOffset) {
        endOffset = p;
        break;
      }
    }

    if (beginName && beginOffset && endOffset && beginName < beginOffset) {
      *beginName++ = '\0';
      *beginOffset++ = '\0';
      *endOffset = '\0';

      // 解码并打印函数名称
      int status;
      char *ret =
          abi::__cxa_demangle(beginName, funcName, &funcNameSize, &status);
      if (status == 0 && i > 1) {
        oss << (i - 1) << " " << ret << std::endl;
      }
    } else {
      // 如果无法解析，直接输出原始符号信息
      oss << symbollist[i] << std::endl;
    }
  }
  free(funcName);
  free(symbollist);
  return oss.str();
}
~~~


## 编译

`ldflags` 参数一定要指定 `-ldynamiclib` 告诉链接器将所有符号添加到动态符号表中。这使得运行时调试工具（如 GDB）和运行时函数（如 backtrace）可以访问这些符号。
主要用于调试和诊断目的，以便在运行时能够获取更详细的堆栈信息和符号解析。

对性能几乎没有影响，但是由于增加了符号表信息，会增加包体大小。






