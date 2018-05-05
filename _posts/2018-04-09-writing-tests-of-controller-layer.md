---
layout: post
title:  "Spring项目编写Controller层的测试用例"
description: "Cover Spring Project with JUnit test on Controller layer. - Spring项目的控制器层测试用例编写"
date:   2018-04-09 20:13:01 +0800
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

首先导入Spring Test的依赖包
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

建立一个入下图结构的Demo项目：

<img src="{{"/assets/img/structure.png" | absolute_url}}" title="结构" width="300px" />

接下来介绍一下配置文件: test/resources/spring-test.xml
```xml
<!-- 首先引入数据库配置文件: -->
<context:property-placeholder location="classpath:datasource.properties"/>
<!-- component-scan 初始化测试用例需要扫描的包文件,这样在测试用例启动的时候spring-test框架就会把dao, service, controller 等Bean资源加载到内存中 -->
<context:component-scan base-package="ny.john.demo.controller, ny.john.demo.service, ny.john.demo.dao"/>
<!-- 接下来是mysql 和 redis 初始化过程, 声明对应名称的Bean文件，包括redis和mysql的连接池管理 -->
<!-- 最后声明一下框架输出的数据类型，包括了html和Restful结构的json字符串 -->
<mvc:annotation-driven>
    <mvc:message-converters>
        <bean class="org.springframework.http.converter.StringHttpMessageConverter">
            <property name="supportedMediaTypes">
                <list>
                    <value>text/html;charset=UTF-8</value>
                    <value>text/plain;charset=UTF-8</value>
                    <value>application/json;charset=UTF-8</value>
                </list>
            </property>
        </bean>
    </mvc:message-converters>
</mvc:annotation-driven>
```
准备好了配置文件以后，就可以开始我们的控制器测试用例编写了：
>几个重要的注解：

```java
@RunWith(SpringJUnit4ClassRunner.class) //声明当前类文件将使用SpringJUnit来启动
@WebAppConfiguration //声明当前类文件将加载Spring的Web应用上下文配置，也就是说我们的Spring mvc结构的上下文会被我们的测试用例理解
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD) //声明当前类的每一个测试用例执行过后上下文都是不可信的，将会被重新加载；也就是说每一个Test过后都会重新初始化上下文
@ContextConfiguration(locations = {"classpath:spring-test.xml"}) //指明上下文的配置文件地址
```
>测试用例初始化方法：

```java
@Before //用这个注解标注的方法将会在每一个测试用例执行之前执行，你可以用来放一些初始化的过程，例如：清空数据库
public void setUp() throws Exception {
    MockitoAnnotations.initMocks(this);
    mockMvc = MockMvcBuilders.webAppContextSetup(context).build(); //使用我们配置的上下文内容初始化当前项目为一个web应用
    truncateData(); //这个方法里会清空数据库和redis的内容，避免两个测试用例之间的数据互相污染
}

/**
 * truncat the mysql and redis test db
 * between every test method
 */
private void truncateData() {
    try {
        Connection       conn = dataSource.getConnection();
        DatabaseMetaData meta = conn.getMetaData();
        ResultSet        rs   = meta.getTables(null, null, "%", null);

        while (rs.next())
            if (rs.getString(4).toLowerCase().equals("table"))
                jdbcTemplate.update("TRUNCATE TABLE " + rs.getString(3));
    } catch (SQLException e) {
        e.printStackTrace();
    }

    stringRedisTemplate.getConnectionFactory().getConnection().flushDb();
}
```

了解了初始化过程以后，就可以开始写测试用例啦，简单的测试用例编写如下：
```java
@Test
public void getUser() throws Exception {
    mockUser();//向数据库里插入一条用户信息
    MvcResult result = mockMvc.perform(MockMvcRequestBuilders.get("/user/get/1")).andExpect(MockMvcResultMatchers.status().is2xxSuccessful()).andReturn(); //模拟一个http请求访问我们的UserController： /user/get/1
    String response = result.getResponse().getContentAsString();

    JSONObject responseJson = JSON.parseObject(response);
    Assert.assertEquals(1, responseJson.getIntValue("id")); //断言返回结构是否符合我们的预期
    Assert.assertEquals(32, responseJson.getIntValue("age"));
}
```

可以通过 [这里](https://github.com/JohnnyChenS/spring-test-demo.git) 下载到我的demo代码