---
layout:     post
title:      "Are You Ready? Git"
subtitle:   "Git提交免密码 "
date:       2018-01-12 12:00:00
author:     "Hua"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - github
---

> “Yeah It's on. ”




相信大家一定遇到过git和开发工具结合不是很好的情况，pull不下来，push不上去。But we are coder rather than  coolie.  (Just be a Joke!)  这时，git命令行就因其很好的容错提交方便了我们的开发。以下是命令行经常遇到的一问题。

    git避免重复输入用户名密码。

1、在git bash命令行中输入   echo $HOME  查看git home路径。

2、进入home对应的路径中。

touch .git-credentials    创建.git-credentials

vim .git-credentials 编辑

在./git-credentials中加入以下文本（此处文本URL可以固定写成这样，如果你的URL和这个不一样，执行完以下操作之后只需要在命令行输入一次

用户名密码会自动把你所使用的URL追加进去），username 和password分别代表用户名密码

https://username:password@github.com

git config --global credential.helper store

这是查看home路径中的.gitconfig，会在之前

[user]

name =**********

email=***********

的基础上多出

[credential]

helper = store

3、在命令行正常执行pull ，push，如果是在以上操作完之后第一次执行向任何URL的pull push，需要输入一次用户名密码，以后不再需要输入。

第一次向新的URL输入用户名密码之后会发现 .git-credentials中追加了类似 https://username:password@hello.com的内容。

