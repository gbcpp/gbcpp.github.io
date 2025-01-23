---
layout: post
title: 'golang 问题记录'
subtitle: go 常见问题记事本
date: 2025-01-21
author: Mr Chen
cover: '/assets/img/shan.jpg'
categories: Notes
tags: 
- Life
---

&emsp;&emsp;
>记录 Golang 学习、开发过程中的一些疑难杂症。

# 环境问题

## missing go.sum entry for module

曾经编译过的代码出现类似如下的错误 `missing go.sum entry for module`，各种依赖缺失的 mod，如下：

```
base/file.go:10:2: missing go.sum entry for module providing package github.com/astaxie/beego/logs; to add:
	go mod download github.com/astaxie/beego
base/name_generate.go:6:2: missing go.sum entry for module providing package github.com/google/uuid (imported by ben/base); to add:
	go get ben/base
helpers/data_analysis/base/bar_base.go:6:2: missing go.sum entry for module providing package github.com/go-echarts/go-echarts/v2/charts (imported by ben/helpers/data_analysis/base); to add:
	go get ben/helpers/data_analysis/base
helpers/data_analysis/base/metrics.go:9:2: missing go.sum entry for module providing package github.com/go-echarts/go-echarts/v2/components (imported by ben/helpers/data_analysis/base); to add:
	go get ben/helpers/data_analysis/base
helpers/data_analysis/base/bar_base.go:7:2: missing go.sum entry for module providing package github.com/go-echarts/go-echarts/v2/opts (imported by ben/helpers/data_analysis/base); to add:
	go get ben/helpers/data_analysis/base
```

**解决办法**

使用 `go mod tidy` 来整理依赖，该命令有如下作用：

- 删除不需要的依赖包
- 下载新的依赖包
- 更新 `go.sum`

重新执行 `go run` `go build` 问题解决。





