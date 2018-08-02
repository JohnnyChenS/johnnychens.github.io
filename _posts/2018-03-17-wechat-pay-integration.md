---
layout: post
title:  "微信支付对接"
description: "Integration of Wechat Pay. -- 微信支付的对接流程。"
date:   2018-03-17 16:13:12 +0800
---

> 本文记录一下微信支付接入过程中踩过的坑。总体来说，微信支付对于想要接入他们的开发人员的所提供的文档有许多不足，很多流程上的介绍让开发人员一头雾水。经过本次对接，记录下过程中的疑问和解决办法，以备后人使用。

* 首先明确的是，我们本次接入的是微信支付的 **[扫码支付](https://pay.weixin.qq.com/wiki/doc/api/native.php?chapter=6_1)** 功能 

#### 1. 下载微信官方的[支付SDK](https://pay.weixin.qq.com/wiki/doc/api/native.php?chapter=11_1)
由于我们是使用JAVA版本的SDK，因此可以直接使用微信官方提供的maven依赖:
```
<dependency>
    <groupId>com.github.wxpay</groupId>
    <artifactId>wxpay-sdk</artifactId>
    <version>0.0.3</version>
</dependency>
```

#### 2. 着手开发前需要了解的一些信息：

- 微信支付要求对接的商家在开发阶段首先必须完成一系列的`功能验收用例`才可以`申请验收完成`并上线功能。
- 基本流程和概念可以在官方的文档中了解: [验收指引](https://pay.weixin.qq.com/wiki/doc/api/jsapi.php?chapter=23_1)；
- 开发过程中所有的操作调用都必须使用沙箱环境的接口，也就是`url中加上/sandboxnew/`的路径接口；
- 沙箱环境下的Sign Key和正式环境下的SignKey是不同的，所以，在接下来的开发过程中，首先你要生成一个用于沙箱环境的sign key.
- 开发过程中的接口调用必须严格按照[验收用例](https://pay.weixin.qq.com/wiki/doc/api/native.php?chapter=23_12)中的要求进行调用，例如：下单接口的支付金额必须是指定的金额，比如3.01元，`否则是不会支付成功的`。

#### 3. 开始开发代码：
了解了上面的信息以后，就可以按照官方SDK中的demo进行接口调用和业务逻辑开发啦。

#### 4. 开发完成，开始验收：
实现了官方的验收用例中除了可选case之外的所有case后，关注[验收用例](https://pay.weixin.qq.com/wiki/doc/api/native.php?chapter=23_12)中的二维码 -> “微信支付商户验收助手”，选择`我的验收`，然后`绑定你的商户`，开始在本地一个一个把用例文档中的case执行一遍，微信的这个验收助手可以`实时的显示你的验收结果`，是否成功，如果出错的话错在哪里，一时找不到原因的话，可以先跑别的用例，`没有要求一定要按照顺序`完成用例，只要把所有的必选用例都通过就可以了。

#### 5. 正式上线支付功能：
完成了所有必选用例的验收以后，等待3个工作日，就可以把你的代码中的useSandbox设置为false，使用`正式的key`和`正式的接口`，尝试支付`任意金额`看看是否可以成功啦！如果成功，证明你已经通过了微信官方的验收，已经正式可以上线你的支付功能了！

#### 附：沙箱环境的Sign Key生成

1. 使用官方提供的[签名校验工具](https://pay.weixin.qq.com/wiki/doc/api/native.php?chapter=20_1)生成一个sign值：

```python
#xml源串
<xml>
  <mch_id>你的微信后台配置中的商户ID</mch_id>
  <nonce_str>任意随机串</nonce_str>
  <sign>任意随机串</sign>
</xml>

#校验结果：
原sign值:你填写的任意随机值
新sign值:这就是你的沙箱环境的sign key了！把它记下来，在接下来的开发过程中你都要用到它
```