---
layout: post
title: 'float 和 int 在 Linux 下的计算性能对比'
date: 2023-10-30
author: Mr Chen
categories: Tech
cover: '/assets/img/shan.jpg'
tags: 
- Linux
- float
---

# Linux下 Float 与 Int 类型计算性能对比

由于 float 在内存中的存储结构与 int 不同，需要存储指数和尾数，并且 cpu 是不支持直接对 float 进行运算的，需要将 float 各部分进行转换后计算，对结果再次进行反向操作进行存储，所以大部分情况下均不支持直接使用 float 进行运行，建议用 int 进行转换运行结果。


## 代码

~~~cpp
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

#include <iostream>

static constexpr int64_t kCalculateNum = 100 * 1000 * 1000L;

static int64_t testFloat()  // return void* in POSIX
{
  struct timespec start_ts, end_ts;
  clock_gettime(CLOCK_MONOTONIC, &start_ts);
  volatile float d = 999999.f;
  for (int n = 0; n < kCalculateNum; ++n) {
    d = d * 0.7f;
  }
  clock_gettime(CLOCK_MONOTONIC, &end_ts);
  auto diff_ns = (end_ts.tv_sec - start_ts.tv_sec) * 1e9 +
                 (end_ts.tv_nsec - start_ts.tv_nsec);
  return diff_ns / 1000;
}

static int64_t testInt()  // return void* in POSIX
{
  struct timespec start_ts, end_ts;
  clock_gettime(CLOCK_MONOTONIC, &start_ts);
  volatile int32_t d = 999999;
  for (int n = 0; n < kCalculateNum; ++n) {
    d = d * 7;
  }
  clock_gettime(CLOCK_MONOTONIC, &end_ts);
  auto diff_ns = (end_ts.tv_sec - start_ts.tv_sec) * 1e9 +
                 (end_ts.tv_nsec - start_ts.tv_nsec);
  return diff_ns / 1000;
}

int main(void) {
  auto float_duration_us = testFloat();
  auto int_duration_us = testInt();
  std::cout << "Float calculate times: " << kCalculateNum
            << " took time: " << float_duration_us << "us.\n";
  std::cout << "Int calculate times: " << kCalculateNum
            << " took time: " << int_duration_us << "us.\n";
  std::cout << "Float / Int radio: "
            << (float_duration_us / int_duration_us * 1.0f) << std::endl;
  return 0;
}
~~~


## 执行结果

- 编译

~~~bash
g++ test_float.cpp -g3 -o test_float.exe
~~~

- 结果

多次执行，结果相当，计算 float 类型耗时大约是 int 整形的 62倍，即性能是 int 的 1/62 左右。

~~~bash
gobert@DESKTOP-EJV14VH:~/develop/debug$ g++ test_float.cpp -g3 -o test_float.exe
gobert@DESKTOP-EJV14VH:~/develop/debug$ ./test_float.exe
Float calculate times: 100000000 took time: 2537060us.
Int calculate times: 100000000 took time: 40442us.
Float / Int radio: 62
~~~

由于不同的 CPU 和体系架构对 FPU 支持情况的不同，以及不同的运算方法对结果也有很大的影响， 比如 加、减、乘、除，以及系数的不同：大于 1 或者小于 1 等均有不同的结果，上述结论为选择了较为复杂的计算场景，仅供参考！！
