---
layout: post
title:  "Spring项目编写Controller层的测试用例"
description: "Cover Spring Project with JUnit test on Controller layer. - Spring项目的控制器层测试用例编写"
date:   2018-04-9 20:13:01 +0800
categories: SpringTest
---

>笔者在从事PHP开发的时候，习惯于编写直接从Controller层覆盖的测试用例，不同于针对每一个Class文件编写单元测试来覆盖该类的每个方法，这是一种偷懒的办法：**我只需要关注于传入控制器的参数是否可以获得预期的输出就可以保证整个接口功能的健壮性**。

```python
#举个例子:
API URI：/user/details/{$userId}

#预期输出：
{
    "id" : 12,
    "username" : "johnnychen",
    "gender" : 1,
    "age" : 32
}

#我只需要mock一个参数userId=12的请求到/usr/details, 并判定预期输出结果就完成了用例的覆盖:
assertEquals(12,$responseJson['id']);
assertEquals("johnnychen",$responseJson['username']);
assertEquals(1,$responseJson['gender']);
assertEquals(32,$responseJson['age']);
```
其中有一些需要注意的地方：
- 由于测试用例是从控制器层开始执行的，所以避免不了读写数据库的过程，你必须准备好一个测试的数据库用来预先写入模拟数据并且判定程序执行后的更新数据，并且需要在每一个测试用例执行完之后清空它以免污染下一个用例的结果；
- 需要对一些外部依赖进行打桩模拟，例如，你的程序依赖于一个第三方的接口返回值，在测试用例中我们需要mock掉这个第三方接口，假设它永远返回了正确的数据，而不是真的每次都去请求它的真实响应，因为对于第三方接口的测试不是我们的目的。

好了，了解完上面的使用场景，接下来就介绍一下我在开发JAVA项目时，如何实现以上测试用例的覆盖的。
