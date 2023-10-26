---
layout: post
title: '网络包序号回绕'
date: 2023-04-18
author: Mr Chen
# cover: '/assets/img/shan.jpg'
#cover: 'https://images.unsplash.com/photo-1653629154302-8687b83825e2'
#cover_author: 'rogov'
#cover_author_link: 'https://unsplash.com/@rogovca'
tags: 
- 协议
---

> Transform your plain text into static websites and blogs.

## 需求

在实现一套基于 UDP/DataChannel 自定义可靠传输算法时，自定义的协议头中肯定也是要用到 sequence Id 来标记包序号，用于重传，计算 RTT 等需求。


## 方案

一般情况下我们都是使用 32字节的 UINT 记录 sequence id，通过不断的递增包序号总是会达到最大值 UINT_MAX，然后回绕到 0 的，此时需要判断新的 sequence id （比如 1， 2，3， xxx），是要 “大于” 旧的 sequence id （4294967293， 4294967294， 4294967295）的，如果直接通过关系运算符  >  < 等进行判断的话，就会得到错误的结果，回绕后的新包序号 sequence id 要排到旧的包序号 sequence id 之前了，针对此种情况，需要一种特殊的 “关系运算符” 比较，在此提供如下比较方法。

### 方案一

通过比较两个包序号之间差值与 kPacketNumberMask 与运算后的结果，与 UINT_MAX / 4 之间的关系来判断。

此种判断方法的前提是认定递增的包序号，在队列中不可能出现最新和最旧的包序号差值能够达到 (kPacketNumberMask >> 1)，即 UINT_MAX / 4 如此之大的，否则说明包序号设计非常的不合理。

有了正确的判断包序号大小的方法，再配合 std::map 可自定义的 key_compare 便可实现对产生回绕的包序号自动排序能力，完整的 Example：

~~~cpp
// 定义 UINT_MAX 的一半
// BIN: 01111111111111111111111111111111; HEX:7FFFFFFF; DEC:2147483647
const uint32_t kPacketNumberMask = (1u << 31) - 1; 

// 当用 UINT 的较小的数 减去 较大的数不够减时，得到的结果会产生回绕：
// 如：3 - UINT_MAX = 4， 4 用 二进制表示为：100b
// 用 kPacketNumberMask & 100b = 100b， 100b 是小于 (kPacketNumberMask >> 1) 的，结果为 true；


// >
bool greaterThan(uint32_t left, uint32_t right) {
  if ((kPacketNumberMask & (left - right)) < (kPacketNumberMask >> 1)) {
    return true;
  } else {
    return false;
  }
}

// <
bool lessThan(uint32_t left, uint32_t right) {
  return greaterThan(right, left);
}

// <=
bool lessEqual(uint32_t left, uint32_t right) {
  if (left == right) {
    return true;
  }
  if ((kPacketNumberMask & (left - right)) > (kPacketNumberMask >> 1)) {
    return true;
  } else {
    return false;
  }
}

~~~


- Example:

~~~cpp

#include <map>
#include <climits>
#include <iostream>

using namespace std;


const uint32_t kPacketNumberMask = (1u << 31) - 1;

bool greaterThan(uint32_t left, uint32_t right) {
  if ((kPacketNumberMask & (left - right)) < (kPacketNumberMask >> 1)) {
    return true;
  } else {
    return false;
  }
}

bool lessThan(uint32_t left, uint32_t right) {
        return greaterThan(right, left);
}

bool lessEqual(uint32_t left, uint32_t right) {
  if (left == right) {
    return true;
  }
  if ((kPacketNumberMask & (left - right)) > (kPacketNumberMask >> 1)) {
    return true;
  } else {
    return false;
  }
}

struct CompareUint : public std::binary_function<uint32_t, uint32_t, bool> {
  bool operator()(uint32_t lhs, uint32_t rhs) const {
    // return !greaterThan(lhs, rhs);
    return lessEqual(lhs, rhs);
 }
};

int main()
{
  std::map<uint32_t, std::string, CompareUint> pkt_map;
  pkt_map[UINT_MAX - 3] = "UINT_MAX - 3";
  pkt_map[UINT_MAX - 2] = "UINT_MAX - 2";
  pkt_map[UINT_MAX - 1] = "UINT_MAX - 1";
  pkt_map[UINT_MAX - 0] = "UINT_MAX - 0";
  pkt_map[1] = "1";
  pkt_map[2] = "2";

  std::cout << "-------------------------------------------------\n";
  std::cout << "UINT_MAX: " << UINT_MAX << ", +1 = " << (UINT_MAX + 1) << std::endl;
  std::cout << "3 - UINT_MAX = " << (3 -UINT_MAX) << std::endl;
  std::cout << "0 - UINT_MAX = " << (0 -UINT_MAX) << std::endl;
  for (auto& itor : pkt_map) {
    std::cout << "key: " << itor.first << ", value: " << itor.second << std::endl;
  }

  return 0;
}
~~~


- 运行结果：

  > 编译：g++  main.cpp -std=c++17  -o main.exe

~~~bash
-------------------------------------------------
UINT_MAX: 4294967295, +1 = 0
3 - UINT_MAX = 4
0 - UINT_MAX = 1
key: 4294967292, value: UINT_MAX - 3
key: 4294967293, value: UINT_MAX - 2
key: 4294967294, value: UINT_MAX - 1
key: 4294967295, value: UINT_MAX - 0
key: 1, value: 1
key: 2, value: 2
~~~

### 方案二
内核中使用的方法：

~~~cpp
static inline bool before(uint32_t seq1, uint32_t seq2)
{
        return (int32_t)(seq1-seq2) < 0;
}
#define after(seq2, seq1)   before(seq1, seq2)
~~~