---
layout: post
title: "Git commit message书写总结"
subtitle: '无规矩不成方圆'
date: "2020-02-19 22:13:10 +0800"
author: dairugang
cover: '/images/2020-02-19-git-commit-message-rule/cover.jpg'
tags: git
---

## 为什么要有规范

作为团队协作工具，Git的提交message非常必要，如果写的比较规范，可以方便查看代码库是如何演进的。如果写的五花八门，那么对将来有可能发生的版本回溯等会造成很大的干扰。

另外，如果按照固有的规范的话，也有工具能帮助直接生成 changelog。

<!-- more -->

## 规范有哪些

现在在网上搜索，比较流行的就是 AngularJS 的规范，具体可[点击查看（需要翻墙）](https://docs.google.com/document/d/1QrDFcIiPjSLDn3EL15IJygNPiHORgU1_OOAqWjiDU5Y/edit#heading=h.7mqxm4jekyct)。

也可以 [点击这里](https://github.com/angular/angular/commits/master) 直接查看Github上的AngularJS提交记录。


### AngularJS规范内容

每次提交，Commit message 都包括三个部分：Header，Body 和 Footer。其中Header是必须的，Body和Footer是可以忽略的。

格式如下：

```
<type>(<scope>): <subject>
// 空一行
<body>
// 空一行
<footer>
```

关于各个部分的说明，参考 [阮一峰写的blog](http://www.ruanyifeng.com/blog/2016/01/commit_message_change_log.html)。

我们希望采用的type包括

- feat：新的功能或特征
- fix：修复bug，可以在footer部分加上 close #bugNo
- update: 更新某个功能或者特征
- docs：针对文档的修改，比如修改Readme也算
- style：针对代码样式的修改，比如添加注释，调整格式等
- test：添加测试用例，或测试相关的内容
- refactor：重构，包括添加共通的类或者方法，重构框架等
- build：跟构建相关内容的修改
- revert：撤销某次的提交

> 注：一般的生成changelog工具，会只生成feat和fix的相关内容。

## 通过工具进行规范

如果使用AngularJS规范，可以通过以下几种方式或工具进行协助。

（1）[commitizen](https://github.com/commitizen/cz-cli)

这个是网上最常搜索到的工具。优点是很规范，强制要求每个步骤，不过也有如下缺点：

- 需要node和npm环境，对于非node项目不友好
- 在windows下使用体验差，我在windows中的 `git bash` 中无法上下选择type

（2）使用git commit template

1. 建立模板文件

在项目中建立 `.git_template` 模板文件，可以自由定义，我推荐的格式如下：

```
<type>(<scope>): <subject>

# type: feat,fix,test,docs,style
# scope: view,model,controller

```

可以充分在模板文件中的注释部分写上对message的要求，比如type的说明，比如有哪些模块等。

2. 设置模板

通过如下命令设置模板：

```
// 当前项目有效
git config commit.template .git_template 
```

3. 提交代码

使用 `git commit` 的时候，会自动用模板填充，然后在模板的基础上进行修改。

> 此方法的优点是比较自由，好控制，学习成本低。
缺点也是过于自由，可能有人还是会写出不规范的内容，另外要求每个人都要在项目环境中设置。

（3）IDEA 插件

如果是使用IDEA进行开发的，IDEA中有如下插件可以完成类似的工作。

- [Git Commit Template](https://plugins.jetbrains.com/plugin/9861-git-commit-template)

## CHANGELOG生成

通过 [Conventional Changelog](https://github.com/conventional-changelog/conventional-changelog) 可以根据commit message来自动生成CHANGELOG内容。

```
# 安装
npm install -g conventional-changelog
cd my-project
# 生成
conventional-changelog -p angular -i CHANGELOG.md -w
```

上面命令不会覆盖以前的 Change log，只会在CHANGELOG.md的头部加上自从上次发布以来的变动。

> 此工具也需要node环境。

## 参考链接

- [阮一峰：Commit message 和 Change log 编写指南](http://www.ruanyifeng.com/blog/2016/01/commit_message_change_log.html)
- [git commit 代码提交规范](https://segmentfault.com/a/1190000017205604)
- [Git commit message规范](https://mp.weixin.qq.com/s/mAUqTPCqcYjoNDqAxktyfA)
