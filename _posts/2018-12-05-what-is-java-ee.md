---
layout: post
title:  "JAVA EE 是什么?"
description: "what is JAVA EE exactly? - JAVA EE 和JAVA SE的区别"
date:   2018-12-05 20:17:05 +0800
categories: JAVA
---

这两天发现了一个之前一直忽略的问题，JDK的名称是<b>“JAVA SE Developement Kit”</b>; 那么JAVA EE有没有JDK呢？于是查了一下，发现还真的有人问[这个问题](https://stackoverflow.com/questions/10438127/is-there-a-java-ee-jdk), 不过正确答案是 -- <b>没有! </b>不过这个问题的实质是，<b>到底什么是JAVA EE，它和JAVA SE的区别到底是什么？</b>

总结一下在StackOverflow上查到的[一些答案](https://stackoverflow.com/questions/2857376/difference-between-java-se-ee-me)：

> JAVA SE 是JAVA的根基，它包括了JAVA的基本语法和常用的类库，包括Collections，Map，etc. 这些常用的JAVA基础类的定义都封装在SE里，而<b>JAVA EE是在SE的基础上扩充了企业级WEB开发需要的类库</b>，包括了Servlet API, JSP, JDBC, JPA, etc.

也就是说，我的电脑里并没有安装JAVA EE，可是我平时开发时使用的Servlet API是从哪里来的呢，我并没有在引用包里加入Servlet API的包依赖呀？原来``Tomcat之所以被称作Web容器，就是因为它实现了Servlet API，为我们的web程序提供了运行环境``。

可以从[Tomcat的官方定义](https://en.wikipedia.org/wiki/Apache_Tomcat)上看出来：

>Apache Tomcat, often referred to as Tomcat Server, is an open-source Java Servlet Container developed by the Apache Software Foundation (ASF). Tomcat implements several Java EE specifications including Java Servlet, JavaServer Pages (JSP), Java EL, and WebSocket, and provides a "pure Java" HTTP web server environment in which Java code can run.

我写了一个Java EE程序来验证我在本地使用``javac``编译时如果不引入tomcat的servlet.jar是否会报错：

```bash
javac  -sourcepath "./src/"  src/ny/john/demo/Dispatcher.java

src/ny/john/demo/Dispatcher.java:9: 错误: 程序包javax.servlet.http不存在

import javax.servlet.http.HttpServlet;
                         ^
```
果然报错了！于是我在编译时指定classpath，包含了tomcat的servlet包，看看效果：

```sh
javac -classpath "./web/WEB-INF/lib/mysql-connector-java-5.1.45-bin.jar:/usr/local/src/apache-tomcat-8.0.36/lib/servlet-api.jar" \
    -sourcepath "./src/"  \
    -Xlint:unchecked \
    src/ny/john/demo/Dispatcher.java

```
编译成功了！


<font size="2" color="#777">其他的一些参考答案：</font>

[What exactly is Java EE?](https://stackoverflow.com/questions/7295096/what-exactly-is-java-ee)

[Do I need to install JEE 7 to run servlets and JSP?](https://www.quora.com/Do-I-need-to-install-JEE-7-to-run-servlets-and-JSP)

[What is Java EE?](https://stackoverflow.com/questions/106820/what-is-java-ee)








