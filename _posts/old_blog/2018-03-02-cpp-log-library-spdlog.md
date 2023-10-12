---
layout: post
title: c++ 开源日志库 spdlog
date: 2018-03-02 00:04:36
author: Mr Chen
# cover: '/assets/img/boost_logo.webp'
tags:
 - spdlog
---


# C++日志组件 spdlog

`spdlog` 是基于C++ 11的开源的轻量级的日志组件，引入也非常简单，仅仅需要引入头文件就可以了。

<!--more-->

## 线程安全

命名空间 `spdlog` 之下的大多数函数都是线程安全的，除了：
~~~cpp
void spdlog::set_pattern(const std::string&);
void spdlog::set_formatter(formatter_ptr);
void spdlog::set_error_handler(log_err_handler);
~~~

日志器对象的大部分方法也是线程安全的，除了：
~~~cpp
void spdlog::logger::set_pattern(const std::string&);
void spdlog::logger::set_formatter(formatter_ptr);
void spdlog::set_error_handler(log_err_handler);
~~~

所有以 `_mt` 结尾的 Sink 都是用于多线程的，以 `_st` 结尾的则是用于单线程的，不过现在单线程的程序很少了吧，建议直接用以 `_mt` 结尾的多线程安全的日志对象；

## 代码示例
~~~cpp
#include "spdlog/spdlog.h"
#include <iostream>
 
// 多线程的基于控制台（stdout）的日志记录器，支持高亮。类似的stdout_color_st是单线程版本
auto console = spdlog::stdout_color_mt( "console" );
// 基于文件的简单日志
auto logger = spdlog::basic_logger_mt("basic_logger", "logs/basic.txt");
// 基于滚动文件的日志，每个文件5MB，三个文件
auto logger = spdlog::rotating_logger_mt("file_logger", "myfilename", 1024 * 1024 * 5, 3);
 
// 定制输出格式
spdlog::set_pattern("*** [%H:%M:%S %z] [thread %t] %v ***");
 
// 多个日志器共享SINK
auto daily_sink = std::make_shared<spdlog::sinks::daily_file_sink_mt>("logfile", 23, 59);
// 下面几个同步日志器共享的输出到目标文件
auto net_logger = std::make_shared<spdlog::logger>("net", daily_sink);
auto hw_logger = std::make_shared<spdlog::logger>("hw", daily_sink);
auto db_logger = std::make_shared<spdlog::logger>("db", daily_sink); 
 
// 一个日志器使用多个SINK
std::vector<spdlog::sink_ptr> sinks;
sinks.push_back( std::make_shared<spdlog::sinks::stdout_sink_st>());
sinks.push_back( std::make_shared<spdlog::sinks::daily_file_sink_st>( "logfile", 23, 59 ));
auto combined_logger = std::make_shared<spdlog::logger>( "name", begin( sinks ), end( sinks ));
spdlog::register_logger( combined_logger );
 
// 异步
// 每个日志器分配8192长度的队列，队列长度必须2的幂
spdlog::set_async_mode(8192); 
// 程序退出前清理
spdlog::drop_all();
 
// 注册日志器
spdlog::register_logger(net_logger);
// 注册后，其它代码可以根据名称获得日志器
auto logger = spdlog::get(net_logger);
 
// 记录日志
// 设置最低级别
console->set_level(spdlog::level::debug);
console->debug("Hello World") ;
// 使用占位符
console->info("Hello {}" ,"World"); 
// 带格式化的占位符：d整数，x十六进制，o八进制，b二进制                
console->warn("Support for int: {0:d}; hex: {0:x}; oct: {0:o}; bin: {0:b}", 42);
// 带格式化的占位符：f浮点数
console->info("Support for floats {:03.2f}", 1.23456);
// 左对齐，保证30字符宽度
console->error("{:<30}", "left aligned");
// 指定占位符位置序号
console->info("Positional args are {1} {0}..", "too", "supported");
 
// 记录自定义类型，需要重载<<操作符
#include <spdlog/fmt/ostr.h> 
class Duck{}
std::ostream& operator<<(std::ostream& os, const Duck& duck){ 
    return os << duck.getName(); 
}
Duck duck;
console->info("custom class with operator<<: {}..", duck);
~~~
输出格式

spdlog默认的输出格式为：
~~~
[2018-03-01 23:46:59.678] [info] [my_loggername] Some message
~~~
要定制输出格式，可以调用：
~~~cpp
//估计大部分人都喜欢下面这样的格式输出格式：
spdlog::set_pattern("[%Y-%m-%d %H:%M:%S.%e][%t][%l] %v");
//或者实现自己的格式化器：
spdlog::set_formatter(std::make_shared<my_custom_formatter>());
~~~

输出格式为：
~~~
[2018-03-01 23:28:04.285][13464][info] Input host name:gobert, ptr:0x9c6a78
~~~

Pattern 格式说明

输出格式的 Pattern 中可以有若干 `%` 开头的标记，含义如下表：

| 标记     | 说明                                                   |
| -------- | ------------------------------------------------------ |
| %v       | 实际需要被日志记录的文本，如果文本中有{占位符}会被替换 |
| %t       | 线程标识符                                             |
| %P       | 进程标识符                                             |
| %n       | 日志记录器名称                                         |
| %l       | 日志级别                                               |
| %L       | 日志级别简写                                           |
| %a       | 简写的周几，例如Thu                                    |
| %A       | 周几，例如Thursday                                     |
| %b       | 简写的月份，例如Aug                                    |
| %B       | 月份，例如August                                       |
| %c       | 日期时间，例如Thu Aug 23 15:35:46 2014                 |
| %C       | 两位年份，例如14                                       |
| %Y       | 四位年份，例如2014                                     |
| %D 或 %x | MM/DD/YY格式日期，例如"08/23/14                        |
| %m       | 月份，1-12之间                                         |
| %d       | 月份中的第几天，1-31之间                               |
| %H       | 24小时制的小时，0-23之间                               |
| %I       | 12小时制的小时，1-12之间                               |
| %M       | 分钟，0-59                                             |
| %S       | 秒，0-59                                               |
| %e       | 当前秒内的毫秒，0-999                                  |
| %f       | 当前秒内的微秒，0-999999                               |
| %F       | 当前秒内的纳秒， 0-999999999                           |
| %p       | AM或者PM                                               |
| %r       | 12小时时间，例如02:55:02 pm                            |
| %R       | 等价于%H:%M，例如23:55                                 |
| %T 或 %X | HH:MM:SS                                               |
| %z       | 时区UTC偏移，例如+02:00                                |
| %+       | 表示默认格式                                           |

## 个人实战应用源码

根据个人的习惯，喜欢在开发后端程序时，喜欢将 log 同时输出到控制台和文件中，那么根据 spdlog 的接口，需要在创建对象时将 console 和 file sink 均传入即可，代码如下：

~~~cpp
    spdlog::set_async_mode(4096);
    spdlog::set_pattern("[%Y-%m-%d %H:%M:%S.%e][%t][%l] %v");
    //创建控制台对象指针
    auto console_log_ptr = spdlog::stdout_logger_mt("console");
    //创建文件对象指针
    auto file_log_ptr = spdlog::rotating_logger_mt("file", "opc-collector.log", 100 * 1024, 2);
    //将以上两种日志对象 sink 组合在一起
    spdlog::sinks_init_list sink_list = { console_log_ptr->sinks().front(), file_log_ptr->sinks().front() };
    //创建一个新的日志对象，以上面两个日志对象作为初始化参数，即实现了同时输出 console 和 file
    auto log_ptr = spdlog::create("loggername", sink_list);
    
    log_ptr->info("Test spdlog output:string:{}, int:{}", "spdlog info test string", 123456);
    log_ptr->flush();
~~~
**注：将多个日志对象同时输出到一个日志对象时，其 loggername 不能保持一致，否则报错警告称：loggername 重复！**

以上内容大部分参考自：https://blog.gmem.cc/spdlog