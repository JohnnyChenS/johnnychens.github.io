---
layout: post
title:  "什么是PHP-FPM"
description: "What is PHP-FPM"
date:   2019-03-08 11:13:44 +0800
categories: PHP
---

php-fpm，php-cgi，fastcgi，master-worker

mysql_pconnect长数据库连接, 每一个fpm worker进程都会保持一个pconnect,并且不释放，这样的话，mysql server将要维护非常多的连接进程，而不是使用连接池来共用少量的数据库连接


php常驻后台程序（请求执行完不销毁资源）swoole + laravel

php如何保证事务的原子性
