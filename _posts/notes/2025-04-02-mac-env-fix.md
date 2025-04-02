<!--
 * @Author: Eddie gbao1987@163.com
 * @Date: 2025-04-02 15:38:32
 * @LastEditors: Eddie gbao1987@163.com
 * @LastEditTime: 2025-04-02 15:52:59
 * @FilePath: /gbcpp.github.io/_posts/notes/2020-01-01-blog-templete copy.md
 * @Description: 这是默认设置,请设置`customMade`, 打开koroFileHeader查看配置 进行设置: https://github.com/OBKoro1/koro1FileHeader/wiki/%E9%85%8D%E7%BD%AE
-->
---
layout: post
title: 'Mac 系统环境配置异常记录'
subtitle: 
date: 2025-04-02
author: Mr Chen
cover: '/assets/img/shan.jpg'
categories: Notes
tags: 
- Life
---


> 整理记录 Mac 下开发、学习环境中遇到的疑难杂症及其解决过程和方案。


# Mac 下 VSCode 和 Cursor IDE 的 vim 插件长按键不生效

- VSCode
  
```bash
# 标准版 VSCode
defaults write com.microsoft.VSCode ApplePressAndHoldEnabled -bool false

# VSCode Insider 版
defaults write com.microsoft.VSCodeInsiders ApplePressAndHoldEnabled -bool false

# VS Codium
defaults write com.vscodium ApplePressAndHoldEnabled -bool false

# VS Codium Exploration 用户
defaults write com.microsoft.VSCodeExploration ApplePressAndHoldEnabled -bool false

# 全局设置（慎用）
defaults delete -g ApplePressAndHoldEnabled
```

- Cursor

虽然 Cursor 也是基于 VSCode，但是需要额外的配置才行。

```bash
defaults write "$(osascript -e 'id of app "Cursor"')" ApplePressAndHoldEnabled -bool false
```

>上述命令执行完后，均需要重启 IDE。

