---
layout: post
title:  "代码重构 -- 设计模式的尝试"
description: "代码重构中的一次实践，装饰模式的运用 - code refactory, a practise of design pattern - decorator pattern."
date:   2018-04-21 20:53:12 +0800
categories: DesignPattern
---
>最近尝试重构了一下之前我在github上的一个[项目](https://github.com/JohnnyChenS/amazon.gcod.git)：调用亚马逊的官方AWS API来生成亚马逊礼物卡。重构的过程中尝试使用了一下设计模式，在多次修改以后，考虑到使用[装饰模式](http://design-patterns.readthedocs.io/zh_CN/latest/structural_patterns/decorator.html)是一个比较合适的结构，于是把代码的重构过程和思路记录下来。

先介绍一下这个项目的业务逻辑：
- 首先，你要有一个亚马逊的开发者账号，并且由于你需要向用户发放亚马逊礼物卡，所以需要在账号上绑定你的信用卡账号，每生成一张礼品卡回扣去相应的金额；
- 其次，每次生成礼物卡都是一次http请求调用亚马逊API的过程；

介绍完业务逻辑，简单说下我的旧代码，逻辑很简单，封装了一个class AmazonGiftCard，一个config文件，class里面有createGiftCard和cancelGiftCard两个方法，用来创建和撤销礼物卡; 有一个私有方法：sendRequest，用来向亚马逊API发起请求；还有很多的私有方法是用来生成一堆必要的加密串和校验串以及http请求的参数的，下文再做详述:
```php
class AmazonGiftCard(){
    private $__partnerId;
    private $__accessKey;
    private $__privateKey;

    public __construct($accessKey, $privateKey, $partnerId){

    }

    public createGiftCard($value){
        $url = "/createGfitCard";
        $params = [
            "value" -> $value
        ];
        return __sendRequest($url, $params);
    }

    public cancelGiftCard($cardId){
        $url = "/cancelGfitCard";
        $params = [
            "cardId" -> $cardId
        ];
        return __sendRequest($url, $params);
    }

    private __sendRequest($url, $params){

    }
}
```

使用起来也非常简单，直接用你的配置文件实例化就可以了：

```php
$giftCardService = new AmazonGiftCard("your_access_key", "your_private_key", "your_partner_id");
$giftCardService->createGfitCard(5); //创建一张5美金的礼物卡
```
重构过程的思路是这样的：
- 首先，把请求亚马逊API的方法剥离出来，封装成一个基础的AWS服务类，里面封装了HTTP POST方法，生成POST请求参数的几个私有方法，校验串生成方法等通用的调用AWS API必须的方法步骤；
- 接下来，一开始打算直接把礼品卡生成类作为AWS服务类的派生类，扩展出createGiftCard和cancelGiftCard方法就可以了；
- 后来，觉得这样做还不够“高级”，既然礼品卡类是AWS服务类的扩展，那么不如使用包装模式，让礼品卡类直接持有一个基类的实例，***这样的好处是，你可以写各种各样的包装类，实现不同的调用AWS基础服务的功能***； 例如：你还有一个类调用了AWS的存储服务S3,你就可以再写一个包装类叫做S3服务类，同样持有AWS基类的实例，封装一个upload方法和download方法用来上传和下载文件到你的AWS云上；

于是重构后的代码是这样的：
```php
/**
 * AWS基础服务类
 */
class AwsService{
    /**
     * 生成签名
     * @return String
     */
    function generateSignature() {

    }

    /**
     * 向AWS API发起请求
     * @return {[type]} [description]
     */
    function sendRequest($url, $op, $params) {
        
    }
}

/**
 * 礼品卡服务类
 */
class GCServiceWrapper{
    const __SERVICE_NAME__ = 'AGCODService';
    const __SERVICE_TARGET__ = 'com.amazonaws.agcod';

    private $__awsService;

    public function __construct(){
        $this->__awsService = new AwsService();
    }

    public function createGfitCard($value){
        $params = [
            'value' => $value,
            'sign' => $this->__awsService->generateSignature()
        ];

        return $this->__awsService->sendRequest($url, 'createGfitCard', $params);
    }
}
```

可以通过 [这里](https://github.com/JohnnyChenS/amazon.gcod.git) 下载到我的代码