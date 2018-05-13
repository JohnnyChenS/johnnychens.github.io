---
layout: post
title:  "使用InjectMocks注解将模拟依赖注入Spring测试中"
description: "Inject mocks into spring test using InjectMocks annotation - 使用InjectMocks注解将模拟依赖注入到Spring测试用例中"
date:   2018-05-02 10:33:21 +0800
categories: SpringTest
---

[上一篇](/springtest/2018/04/09/writing-tests-of-controller-layer.html) 介绍完了如何编写一个简单的控制器层测试用例，这一篇将介绍如何<b>"对一些外部依赖进行模拟"</b>, 也就是当你的程序依赖于一个第三方的接口返回值时，在测试用例中需要mock掉这个接口，假设它永远返回了正确的数据，以避免他的返回结果对我们的测试结果产生影响。

还是用在之前Demo代码的基础上来做修改：
- 加入了一个简单的用户身份验证的拦截器```AuthorizationInterceptor```,用来介绍如何在测试用例的路由过程中注入该拦截器；

```java

@Component
public class AuthorizationInterceptor implements HandlerInterceptor {
    private String[] needLoginUriArr= new String[]{"/weather/today"};//需要拦截器验证的URL路径

    @Resource
    private UserDao userDao;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws IOException {
        String   uidAttr         = request.getHeader("uid");

        response.setCharacterEncoding("UTF-8");
        response.setContentType("application/json; charset=utf-8");

        for (String needLoginUri : needLoginUriArr) {
            if (request.getRequestURI().startsWith(needLoginUri) &&
                    (uidAttr == null || userDao.get(Integer.parseInt(uidAttr)) == null)) { //身份验证失败就抛出错误提示
                response.getWriter().write("{\"error\":\"you are not authorized to visit this page.\"}");
                response.getWriter().flush();
                response.getWriter().close();
                return false;
            }
        }

        return true;
    }
}

```

- 一个Configure类将拦截器注册到项目中:

```java
@Configuration
public class Configure extends WebMvcConfigurerAdapter{
    @Resource
    private AuthorizationInterceptor authorizationInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(authorizationInterceptor).addPathPatterns("/**");//对所有路径都使用该拦截器规则
    }
}
```

- 创建一个WeatherService类，这个类调用了百度的天气预报API，可以查询各地的天气情况，这里我只简单封装了该API示例中的北京地区天气的url（否则需要注册百度API的开发者账号）;

```java

@Service
public class WeatherService {
    private String getFromBaiduApi(String city) throws IOException {
        String urlStr = "http://api.map.baidu.com/telematics/v3/weather?";

        Map<String, String> params = new HashMap<>();
        params.put("location", city);
        params.put("output", "json");
        params.put("ak", "E4805d16520de693a3fe707cdc962045");

        return HttpUtil.get(urlStr, params);
    }

    public JSONObject getToday(String city) throws IOException {
        String weatherData = getFromBaiduApi(city);

        return JSONObject.parseObject(weatherData);
    }
}

```

- 一个WeatherController的控制器，访问后向用户问好并显示今日的天气情况:

```python
#HTTP Request URL: 
http://localhost:8080/weather/today

#Response:
hello Johnny: 
 Here is today's weather: 雷阵雨 28 ~ 16℃
```

接下来说说如何在测试用例中加入拦截器，其实很简单，在spring-test.xml文件中注册一个mvc拦截器Bean就可以了:
```xml
<mvc:interceptors>
    <bean id="authorizationInterceptor" class="ny.john.demo.Interceptor.AuthorizationInterceptor"/>
</mvc:interceptors>
```
这样一来，Spring Test在初始化之后mockMvc实例被注入了拦截器，下面这个测试用例会因为没有带上用户的身份标识而被拦截器拒绝：
```java
@Test
public void getTodayNoAuthorization() throws Exception {
    MvcResult result = mockMvc.perform(MockMvcRequestBuilders.get("/weather/today")).andExpect(MockMvcResultMatchers.status().is2xxSuccessful()).andReturn();
    String response = result.getResponse().getContentAsString();

    JSONObject responseJson = JSON.parseObject(response);
    Assert.assertEquals("you are not authorized to visit this page.", responseJson.getString("error"));
}
```

>最后，终于可以介绍本文的主题了，如何在测试用例中“打桩”模拟依赖代码的响应。

所谓 ***打桩*** 就是将A方法的内部实现用A1方法来替代，以解除对A方法中某些代码的依赖。Spring Test中，通过```@Mock ```注解来创建一个完全被模拟的Bean实例，并用```@InjectMocks ```注解来将这个Bean注入到它的依赖对象中:

```java
@RunWith(SpringJUnit4ClassRunner.class)
@WebAppConfiguration
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
@ContextConfiguration(locations = {"classpath:spring-test.xml"})
public class InjectMocksControllerTest {
    @Mock
    private WeatherService weatherService;

    @InjectMocks
    @Resource
    private WeatherController weatherController;
}
```
上面的示例代码省略了部分无关的内容，为了着重介绍这两个Bean的依赖关系。

先看WeatherController的源码，它的类定义中对WeatherService有一个依赖：
```java
@RestController
@RequestMapping(value = "/weather")
public class WeatherController {

    @Resource
    private WeatherService weatherService;

    @Resource
    private UserDao userDao;

    @RequestMapping(value = "/today")
    public String hello(HttpServletRequest request) throws IOException {
        
    }
}
```

所以在测试用例中，我们创建了一个Mock的weatherService，注意：```@Mock ```注解会创建一个对应类的空对象实例，它什么方法体也没有，所以我们必须在后面的代码中模拟他的方法体行为。

然后，我们用```@InjectMocks ```和```@Resource ```标注一个weatherController, 用@Resource标注是告诉Spring Test这是一个注册过的Bean，测试框架会从Context中找到同名的Bean，同时它还被标注了@InjectMocks注解，所以这个Bean的私有成员weatherService会被我们的空对象***替换掉***。

接下来，要使用一种新的方式来初始化我们的测试用例。在之前的介绍中我们是直接使用```spring-test.xml ```文件配置的context来初始化我们的MockMvc：
```java
@Before
public void setUp() throws Exception {
    MockitoAnnotations.initMocks(this);
    mockMvc = MockMvcBuilders.webAppContextSetup(context).build();
}
```
这次因为我们声明了一个注入过“桩实例”的Controller Bean，所以不能用实例化过的context，必须直接使用这个controller:
```java
@Before
public void setUp() throws Exception {
    MockitoAnnotations.initMocks(this);
    mockMvc = MockMvcBuilders.standaloneSetup(weatherController).addInterceptors(authorizationInterceptor).build();
    /**
     * 这里做了两件事：
     * 1. 用那个注入过的weatherController实例化了MVC框架；
     * 2. 向框架注册了权限拦截器（因为我们没有用spring-test.xml文件初始化过的上下文，所以拦截器没有被注入，必须显式的这样注册一下）
     */
}
```
剩下的内容就很简单了，在测试用例里面指定一下刚才Mock出来的weatherService空对象的指定方法的返回结果，然后判定一下控制器调用的结果是否符合预期就行了：

```java
@Test
public void getToday() throws Exception {
    mockUser();

    JSONObject weatherJson = JSONObject.parseObject("{\"temperature\":\"28 ~ 16℃\",\"weather\":\"雷阵雨\"}");
    //这一行声明weatherService的getToday方法接受一个字符串类型的参数，然后返回一个我们给定的JSONObject对象
    Mockito.when(weatherService.getToday(Mockito.anyString())).thenReturn(weatherJson);

    MvcResult result = mockMvc.perform(MockMvcRequestBuilders.get("/weather/today").header("uid",1)).andExpect(MockMvcResultMatchers.status().is2xxSuccessful()).andReturn();
    String response = result.getResponse().getContentAsString();

    Assert.assertEquals("hello Johnny: \n Here is today's weather: 雷阵雨 28 ~ 16℃", response);
}
```

>可以通过 [这里](https://github.com/JohnnyChenS/spring-test-demo.git) 下载到我的Demo代码