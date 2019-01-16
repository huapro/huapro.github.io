---
layout:     post
title:      "Github"
subtitle:   "Github 命令总结 "
date:       2018-04-06  08:16:00
author:     "Hua"
header-img: "img/网络.JPG"
catalog: true
multilingual: false
tags:
- github
---

Github命令
xbk
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

### Git--将服务器代码更新到本地

1. git status（查看本地分支文件信息，确保更新时不产生冲突）

2. git checkout -- [file name] （若文件有修改，可以还原到最初状态; 若文件需要更新到服务器上，应该先merge到服务器，再更新到本地）

3. git branch（查看当前分支情况）

4. git checkout [remote branch](若分支为本地分支，则需切换到服务器的远程分支)

5. git pull

若命令执行成功，则更新代码成功！

## UTF-8 BOM
在我们通常使用的windows系统中，我发现了一个有趣的现象。我新建一个空的文本文档，点击文件-另存为-编码选择UTF-8，然后保存。此时这个文件明明是空的，却占了3字节大小。原因在于：此时保存的编码方式自动会变为UTF-8 BOM

一、一个汉字在不同的编码方式中占多少字节？

1.在UTF-8中，一个汉字占3个字节（一个字符占一个字节）

2.在ASCII码中，一个汉字占2个字节（一个字符占一个字节）

3.在Unicode编码中，一个汉字占2个字节（一个字符同样占两个字节，所以JAVA中char a = '中';是可以的）

二、UTF-8与UTF-8 BOM

BOM即byte order mark，具体含义可百度百科或维基百科，UTF-8文件中放置BOM主要是微软的习惯，但是放在别的系统上会出现问题。

不含BOM的UTF-8才是标准形式，UTF-8不需要BOM

带BOM的UTF-8文件的开头会有U+FEFF，所以我新建的空文件会有3字节的大小。

三、创建UTF-8（而非UTF-8 BOM）文件的方法

在发现文件另存为UTF-8缺得到UTF-8 BOM文件后，我们怎样才能得到UTF-8呢？

法1.先另存为UTF-8保存，再使用notepad++打开，把里面的编码设置为无BOM的UTF-8然后保存。（此方法治标不治本，因为当你再次在里面写汉字时，