---
layout: post
title: 'OBS building on windows'
date: 2024-09-16
author: Mr.Chen
cover: '/assets/img/hu.png'
tags: OBS
---

> 在 Windows 上编译、定制开发 OBS 其实已经精力了多次了，但是每次都没有记录下来一些有用的信息，这次在此进行详细的记录。

## 环境准备

Windows 10 1909+ (or Windows 11)
Visual Studio 2022 (at least Community Edition)
Windows 10 SDK (minimum 10.0.20348.0)
C++ ATL for latest v143 build tools (x86 & x64)
MSVC v143 - VS 2022 C++ x64/x86 build tools (Latest)
Git for Windows
CMake 3.24 or newer