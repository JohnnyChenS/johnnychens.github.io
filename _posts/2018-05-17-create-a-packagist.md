---
layout: post
title:  "创建一个Composer Packagist包"
description: "how to create a composer packagist -- 创建一个Composer Packagist"
date:   2018-05-17 22:13:01 +0800
categories: PHP
---

>本文记录一下创建一个composer packagist包项目的过程以及过程中的一些注意事项。依然以我的 [Amazon GCOD](https://github.com/JohnnyChenS/amazon.gcod) 项目为例子。

- 首先在代码根目录运行 ```composer init``` 命令初始化你的composer.json文件：

```bash
johnny @ Johnny-MBP-2 in ~/Development/my.git/amazon.gcod 
$ composer init ⏎

Welcome to the Composer config generator

This command will guide you through creating your composer.json config.

Package name (<vendor>/<name>) [johnny/amazon.gcod]: #这里填写你的<packagist用户名>/<包名>
Description []: #包的描述
Author [johnny <chz0321@gmail.com>]: #作者信息
Minimum Stability []: #最低稳定版本, 默认是“stable”；这里有一些注意点，我们后面会说到
Package Type []: #默认"library"
License []: #授权协议,一般我都填MIT
```
根据提示进行配置，这些提示都有默认值，大部分不需要改，直接回车就行了；

***Minimum Stability*** : 最低稳定版本，指的是你的项目在引用其他第三方composer代码时，允许引用的最低版本，默认是***stable*** - 稳定；可以显式的指定以下任一种：

```bash
dev #开发版
alpha 
beta 
RC
stable #稳定版
```
>PS. 我们的github代码master分支其实对应的是composer的 ***dev-master*** , 也就是"dev" 开发版; 而当你为你的git项目打上Tag, 这个Tag就会成为一个 "stable" 稳定版;

- 接下来就是把你的git代码发布到Composer的Packagist.org上, 之后Packagist就会从你的github上同步代码，如果你的github项目只有一个master分支，就会发现你的packagist项目上只有一个版本***“dev-master”***, 如果你给你的github项目打过Tag, 就会看到packagist上会出现这个tag的名称对应的版本: 

```bash
#我的packagist项目有两个版本：
dev-master
v1.0
```

- 接着尝试引用一下你自己的包：

```bash
#在一个新建目录中运行
composer require [你的包名]:[版本号]
#eg.:
composer require "johnnychens/amazon.gcod:>=1.0" ⏎
```
>PS. 如果不填写版版本号，默认则是会引用"dev-master"版本，而composer项目的默认最小稳定版本又是"stable"，这样就会出现矛盾，报以下错误 :

```bash
[InvalidArgumentException]
  Could not find package johnnychens/amazon.gcod at any version 
  for your minimum-stability (stable). Check the package spelling 
  or your minimum-stability
```

[这里](https://packagist.org/packages/johnnychens/amazon.gcod) 是我的Packagist包页面