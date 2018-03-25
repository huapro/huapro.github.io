---
layout:     post
title:      "Github"
subtitle:   "Github 命令总结 "
date:       2018-03-24 17:39:00
author:     "Hua"
header-img: "img/网络.JPG"
catalog: true
multilingual: false
tags:
- github
---

Github命令

>“Github应用总结 ”

##     git避免重复输入用户名密码。
相信大家一定遇到过git和开发工具结合不是很好的情况，pull不下来，push不上去。But we are coder rather than  coolie.  (Just be a Joke!)  这时，git命令行就因其很好的容错提交方便了我们的开发。以下是命令行经常遇到的一问题。



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

## git命令
### git怎样删除远程仓库的某次错误提交？

假设你有3个commit如下：

commit 3
commit 2
commit 1
其中最后一次提交commit 3是错误的，那么可以执行：

git reset --hard HEAD~1
你会发现，HEAD is now at commit 2。

然后再使用**git push --force**将本次变更强行推送至服务器。这样在服务器上的最后一次错误提交也彻底消失了。

*值得注意的是，这类操作比较比较危险，例如：在你的commit 3之后别人又提交了新的commit 4，那在你强制推送之后，那位仁兄的commit 4也跟着一起消失了。*

### git怎本地的代码回滚到某一次commit（提交）
使用Git管理我们开发的项目时，有时候我们会想将本地的代码回滚到某一次commit（提交）时，可以使用以下命令：

git reset --hard xxx......xxxx

其中“xxx......xxxx”为你想回滚到的版本。

如某一commit为：12eeeeae0f0a3cb9de7442086135cea51adc1592

则回滚到该版本的操作为：

git reset --hard 12eeeeae0f0a3cb9de7442086135cea51adc1592


