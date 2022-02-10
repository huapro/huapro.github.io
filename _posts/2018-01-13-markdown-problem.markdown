---
layout:     post
title:      "遇到问题，解决问题，记录问题"
subtitle:   " \"Win7安装MarkdownPad2，报Awesomium.Windows.Controls.WebControl 错误的解决方案\""
date:       2018-01-13 20:30:16
author:     "Hua"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 杂七杂八
---

> “Yeah It's on. ”


## 前言

错误如图

![img](/img/in-post/post-technical-2018/2018-01-13-error-of-markdown.jpg)

解决办法

将注册表 
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Lsa\FipsAlgorithmPolicy\Enabled 
项的值改为0

——Hua 于 2018.01.13
