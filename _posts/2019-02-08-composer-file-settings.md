---
layout: post
title:  "Composer Packagist配置项介绍"
description: "配置Packagist项目的composer.json文件"
date:   2019-02-08 11:13:44 +0800
categories: PHP
---

自[上篇](/php/2018/05/17/create-a-packagist.html)介绍过如何创建一个Packagist项目后，今天记录一下composer文件中的其他配置项内容,以我的 [Amazon GCOD](https://github.com/JohnnyChenS/amazon.gcod) 项目为例:

```bash
{
    "name": "johnnychens/amazon.gcod",
    "description": "This class is for making a request to Amazon GiftCode on Demand service API.",
    "license": "MIT",
    "authors": [
        {
            "name": "johnnychen85",
            "email": "chz0321@gmail.com"
        }
    ],

    "require-dev": {
        "phpunit/phpunit": "^4.8"
    },

    "autoload": {
        "psr-4": {
            "Amazon\\": "src/Amazon"
        }
    }
}
```
详细介绍以下几个选项：
- autoload的四种模式（psr-0, psr-4, classmap, files）
- 版本号约束规则（~和^符号解释）
- minimal stability解释

>首先解释一下autoload的四种模式

***PSR-0 和 PSR-4:***
- 首先介绍一下它们的定义，它们是PHP的两个autoload标准规则，简单地说它们规定将每一级命名空间转义为一个文件夹路径，区别在于PSR-4不再将下划线"_"转义为一个文件路径分割符，而PSR-0已经被PHP官方标记为过时不再推荐使用。举个例子：

```php
namespace \ny\john\demo;

class Test_Api{

}
```

对于上面这个类按照PSR-0规则, 文件路径是： ``/ny/john/demo/Test/Api.php``;
而按PSR-4规则, 文件路径是: ``/ny/john/demo/Test_Api.php``.

- 接下来介绍一下它们在composer中的用法，

```php
"autoload": {
        "psr-4": {
            "Amazon\\": "src/Amazon" #是一个命名空间和一个路径目录的对应关系
        }
    }
```
意思是按照psr-4规则，遇到```"Amazon\"```这个命名空间，就在```"src/Amazon"```目录下寻找;

***classmap和files***
```php
"autoload": {
        "classmap": ["src/"], #告诉composer扫描src目录下的所有文件并声称对应的类名和文件路径的映射关系
        "files": [
            "app/Helpers/functions.php" #扫描所有文件，运行时实时加载，主要用来载入工具函数
        ]
    }
```
*classmap和PSR-0, PSR-4都是采用懒加载的方式，当声明使用到了某个类才将他加载进来；而files则是运行时实时加载的。*

如果你的项目修改了autoload配置，记得要运行``composer dump-autoload``命令来更新当前项目的autoload缓存文件：

<img src="{{"/assets/img/autoload-structure.png" | absolute_url}}" title="autoload文件结构" width="300px" />

> 接下来介绍依赖库的版本号约束运算符（下一个重要版本）：

首先介绍一下**语义化版本号**的定义：

版本格式：主版本号.次版本号.修订号，先行版本号及版本编译元数据可以加到“主版本号.次版本号.修订号”的后面，作为延伸。例如：
```php
1.2.11-alpha 
#主版本为1，次版本为2，修订号为11,先行版本号alpha
```

版本号递增规则如下：
- 主版本号：当你做了不兼容的 API 修改，
- 次版本号：当你做了向下兼容的功能性新增，
- 修订号：当你做了向下兼容的问题修正。


**～符号和^符号：**
- ～符号：~1.2 等同于 >=1.2,<2.0.0 ; ~1.2.3 等同于 >=1.2.3,<1.3.0
- ^符号：^1.2.3 等同于 >=1.2.3,<2.0.0
