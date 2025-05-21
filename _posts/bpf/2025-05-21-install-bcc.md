---
layout: post
title: 'Install bcc on ubuntu22.04'
subtitle: 
date: 2025-05-21
author: Mr Chen
cover: '/assets/img/shan.jpg'
categories: Notes
tags: 
- bcc
- ebpf
---


> BCC (BPF Compiler Collection) 是一个强大的工具集，用于开发和运行基于 eBPF 的系统性能分析和网络监控工具。最近需要在为新的运行于 Ubuntu22.04 之上的服务排查内存泄漏的问题，而在 Ubuntu 系统中，虽然可以通过 apt 安装 BCC，但存在很多诡异的兼容性问题，通过源码编译安装的方式能获取最新功能并解决版本兼容问题。这里记录 bcc 在 Ubuntu22.04 下进行编译安装，并进行内存泄漏检测的全部过程。

## 环境准备


```bash
sudo su
apt update && apt upgrade -y

# 安装必要依赖
apt install -y bison build-essential cmake flex git libedit-dev \
  libllvm14 llvm-14-dev libclang-14-dev python3.11 zlib1g-dev libelf-dev \
  libfl-dev python3-distutils
```

- **安装内核头文件**

bcc 需要依赖 Linux 内核的头文件，在 orb 虚拟机下，通过 `uname -r` 获取到的内核版本有特殊的后缀，无法直接安装。
如我的 orb 虚拟机：

```bash
root@ubuntu22:~/tools# uname -r
6.13.7-orbstack-00283-g9d1400e7e9c6

# 通过 apt install lin ux-headers-$(uname -r) 是查找不到对应的安装包的
root@ubuntu22:~/tools# apt install -y linux-header-$(uname -r)
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
E: Unable to locate package linux-header-6.13.7-orbstack-00283-g9d1400e7e9c6
E: Couldn't find any package by glob 'linux-header-6.13.7-orbstack-00283-g9d1400e7e9c6'
```

如何通过 `apt install linux-headers-$(uname -r)` 安装报错，则可以通过安装通用内核头文件版本进行替代：

```bash
apt install -y linux-headers-generic
```

## 源码编译步骤

### 获取源码

```bash
git clone https://github.com/iovisor/bcc.git
cd bcc
git checkout v0.34.0
```

### 编译

```bash
mkdir build; cd build
cmake ..
make
make install
cmake -DPYTHON_CMD=python3 ..
pushd src/python/
make
make install
popd
```

### 验证安装

```bash
# 依然在 bcc/build 目录下，执行  ./examples/cpp/HelloWorld 输出如下内容即表示编译安装 c++版本可以正常运行
root@:~/workspace/tools/bcc/build# ./examples/cpp/HelloWorld
Starting HelloWorld with BCC 0.34.0+1f63ae6e
           <...>-357844  [002] ....1 1312511.496364: bpf_trace_printk: Hello, World! Here I did a sys_clone call!


# 验证 python 工具是否可用，输出如下内容标识正常可用
(base) root@iZ2ze8gd919c5qtgeqbovnZ:~/workspace/tools/bcc/build# python3 ../examples/hello_world.py
b'           <...>-356665  [000] ....1 1312599.534997: bpf_trace_printk: Hello, World!'
b''
b'           <...>-3542460 [008] ....1 1312599.535595: bpf_trace_printk: Hello, World!'
b''
b'         flannel-3542460 [008] ....1 1312599.535643: bpf_trace_printk: Hello, World!'
b''
b'         flannel-3542460 [008] ....1 1312599.535675: bpf_trace_printk: Hello, World!'
b''
b'           <...>-3542462 [011] ....1 1312599.535707: bpf_trace_printk: Hello, World!'
b''
b'     cri-dockerd-356665  [000] ....1 1312599.536306: bpf_trace_printk: Hello, World!'
b''
b'           <...>-3542465 [008] ....1 1312599.536762: bpf_trace_printk: Hello, World!'
b''
b'         portmap-3542465 [008] ....1 1312599.536807: bpf_trace_printk: Hello, World!'
b''
b'         portmap-3542465 [008] ....1 1312599.536850: bpf_trace_printk: Hello, World!'
```

## 常见问题

### 找不到lbbcc.so.o 中某个符号

执行 hello_world.py 脚本输出报错类似如下：

```bash
AttributeError: /lib/x86_64-linux-gnu/libbcc.so.0: undefined symbol: bpf_module_create_b
```

原因为 python3 中加载以来的 libbcc.so 版本与编译安装的不匹配导致，将 bcc/build 中生成的对 python3 进行覆盖即可：

```bash
# 进入 bcc/build 目录
cp -rf ./src/python/bcc-python3/bcc/* /usr/lib/python3/dist-packages/bcc/
```

### 找不到 asm/types.h 头文件

```bash 
root@ubuntu22:~/tools# /usr/share/bcc/examples/hello_world.py)
In file included from <built-in>:2:
In file included from /virtual/include/bcc/bpf.h:12:
In file included from include/linux/types.h:6:
include/uapi/linux/types.h:5:10: fatal error: 'asm/types.h' file not found
#include <asm/types.h>
         ^~~~~~~~~~~~~
1 error generated.
Traceback (most recent call last):
  File "/usr/share/bcc/examples/hello_world.py", line 12, in <module>
    BPF(text='int kprobe__sys_clone(void *ctx) { bpf_trace_printk("Hello, World!\\n"); return 0; }').trace_print()
    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/lib/python3/dist-packages/bcc/__init__.py", line 505, in __init__
    raise Exception("Failed to compile BPF module %s" % (src_file or "<text>"))
Exception: Failed to compile BPF module <text>
```

>在 orb 下的 ubuntu 虚拟机中有遇到，且暂时一直待解决，网上其他人说的解决方案均不生效。


## 内存泄漏检测

### 示例代码

命名为 `test.cpp`。

```cpp
#include <unistd.h>
#include <iostream>
#include <vector>

static std::vector<char*> g_vec;

void memory_test() {
    char* ptr = new char[1024]; // 故意泄漏内存
    char* ptr2 = (char *)malloc(111);
    delete[] ptr;
    g_vec.push_back(ptr2);
    std::cout << __PRETTY_FUNCTION__ << std::endl;
}

void my_func (int num) {
  (void)num;
  memory_test();
  sleep(1);
}

int main() {
    // 模拟内存泄漏
    for(int i = 0; i < 1000; ++i) {
      my_func(5);
    }

    sleep(30);

    for (auto itor : g_vec) {
      free(itor);
    }

    return 0;
}
```

### 静态编译链接方式检测

使用静态方式进行编译：

```bash
g++ -g -static test.cpp -o test.exe
```

生成的 `test.exe` 二进制文件无任何依赖，所以系统调用的一些接口符号均在 `test.exe` 文件中。

通过 `memleak` 进行检测(需要 root 权限)：

```bash
# -O 为指定符号文件； -p 为指定进程的 ID
memleak -O test.exe -p ${pid}
```

*输出内容：*

```bash
(base) root@iZ2ze8gd919c5qtgeqbovnZ:~/workspace/tools/bcc/build# ps -aef | grep test
root     3553018 3552090  0 21:56 pts/0    00:00:00 ./test.exe
root     3553054 3357603  0 21:56 pts/10   00:00:00 grep --color=auto test
(base) root@iZ2ze8gd919c5qtgeqbovnZ:~/workspace/tools/bcc/build# memleak-bpfcc -O /root/workspace/debug/test.exe -p 3553018
Attaching to pid 3553018, Ctrl+C to quit.
[21:56:42] Top 10 stacks with outstanding allocations:
	555 bytes in 5 allocations from stack
		memory_test()+0x33 [test.exe]
		my_func(int)+0x14 [test.exe]
		main+0x2e [test.exe]
		__libc_start_call_main+0x6a [test.exe]
[21:56:47] Top 10 stacks with outstanding allocations:
	256 bytes in 1 allocations from stack
		operator new(unsigned long)+0x1c [test.exe]
		std::allocator_traits<std::allocator<char*> >::allocate(std::allocator<char*>&, unsigned long)+0x2c [test.exe]
		std::_Vector_base<char*, std::allocator<char*> >::_M_allocate(unsigned long)+0x2e [test.exe]
		void std::vector<char*, std::allocator<char*> >::_M_realloc_insert<char* const&>(__gnu_cxx::__normal_iterator<char**, std::vector<char*, std::allocator<char*> > >, char* const&)+0x95 [test.exe]
		std::vector<char*, std::allocator<char*> >::push_back(char* const&)+0x7c [test.exe]
		memory_test()+0x60 [test.exe]
		my_func(int)+0x14 [test.exe]
		main+0x2e [test.exe]
		__libc_start_call_main+0x6a [test.exe]
	1110 bytes in 10 allocations from stack
		memory_test()+0x33 [test.exe]
		my_func(int)+0x14 [test.exe]
		main+0x2e [test.exe]
		__libc_start_call_main+0x6a [test.exe]
[21:56:52] Top 10 stacks with outstanding allocations:
	256 bytes in 1 allocations from stack
		operator new(unsigned long)+0x1c [test.exe]
		std::allocator_traits<std::allocator<char*> >::allocate(std::allocator<char*>&, unsigned long)+0x2c [test.exe]
		std::_Vector_base<char*, std::allocator<char*> >::_M_allocate(unsigned long)+0x2e [test.exe]
		void std::vector<char*, std::allocator<char*> >::_M_realloc_insert<char* const&>(__gnu_cxx::__normal_iterator<char**, std::vector<char*, std::allocator<char*> > >, char* const&)+0x95 [test.exe]
		std::vector<char*, std::allocator<char*> >::push_back(char* const&)+0x7c [test.exe]
		memory_test()+0x60 [test.exe]
		my_func(int)+0x14 [test.exe]
		main+0x2e [test.exe]
		__libc_start_call_main+0x6a [test.exe]
	1665 bytes in 15 allocations from stack
		memory_test()+0x33 [test.exe]
		my_func(int)+0x14 [test.exe]
		main+0x2e [test.exe]
		__libc_start_call_main+0x6a [test.exe]
```

每 10 秒钟输出一次，`Top 10 stacks with outstanding allocations:` 下面为内存申请且未释放的前 10 个堆栈信息。


### 非静态编译链接方式检测

使用非静态方式进行编译：

```bash
g++ -g test.cpp -o test.exe
```

生成的 `test.exe` 二进制文件有系统标准库依赖，如下：

```bash
(base) root@:~/workspace/debug# ldd test.exe
	linux-vdso.so.1 (0x00007fffab6d7000)
	libstdc++.so.6 => /lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007f3ca7fb6000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f3ca7d8d000)
	libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f3ca7ca6000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f3ca81f3000)
	libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f3ca7c86000)
```

我们要动态追踪的系统调用 malloc、free 等在 libc.so 中，所以通过 -O 指定的符号文件在 /lib/x86_64-linux-gnu/libc.so.6 中。


通过 `memleak` 进行检测(需要 root 权限)：

```bash
# -O 为指定符号文件； -p 为指定进程的 ID
memleak -O /lib/x86_64-linux-gnu/libc.so.6 -p ${pid}
```

*输出内容：*

同静态编译链接方式内容。

## 参考资料

BCC 官方文档：https://github.com/iovisor/bcc/blob/master/INSTALL.md
