---
layout: post
title: "发布与订阅"
date: 2017-11-12 15:00:00 +0800
categories: 优雅代码
tag: [guava, publish/subscribe]
---
* content
{:toc}


我称其为单块架构的利器
## 前言
在设计模式中， 有一种叫做发布/订阅模式， 即某事件被发布， 订阅该事件的角色将自动更新。
那么订阅者和发布者直接耦合， 也就是说在发布者内要通知订阅者说我这边有东西发布了， 你收一下。 

正如在rxJava中， 主要就是发布订阅设计模式
<!-- more -->

```java
Observable.just(1).subscribe(new Subsriber(){

    @Override
    public void onCompleted() {
    System.out.println("onCompleted ");
    }

    @Override
    public void onError(Throwable arg0) {
    }

    @Override
    public void onNext(Long arg0) {
        System.out.println("onNext " + arg0);
    }
})

我们可以看到， 发布者发布一个数字1， 订阅者订阅了这个

```


而加入eventBus， 发布者与生产者之间的耦合性就降低了。因为这时候我们去管理eventbus就可以， 发布者只要向eventbus发送信息就可以， 而不需要关心有多少订阅者订阅了此消息。模型如下：
![image](https://gss2.bdstatic.com/-fo3dSag_xI4khGkpoWK1HF6hhy/baike/c0%3Dbaike116%2C5%2C5%2C116%2C38/sign=17cf297a30dbb6fd3156ed74684dc07d/aa64034f78f0f736e9d6d1800355b319ebc41302.jpg)


## 为什么说eventBus 是单块架构的利器呢？ 
首先单块架构就是在一个进程内， 在一个进程内， 我们还是希望模块与模块之间（功能与功能之间）是松耦合的，而在一个模块中是高度内聚的， 如何降低一定的耦合， 使得代码更加有结构， guava eventbus就是支持进程内通讯的桥梁。 

### 想象一下以下业务
我们希望在数据到来之后， 进行入库， 同时能够对数据进行报警预测， 当发生报警了， 能够有以下几个动作， 向手机端发送推送， 向web端发送推送， 向手机端发送短信。

在一般情况下我们可以这样实现： （伪代码如下）

```
processData(data){
    insertintoDB(data); //执行入库操作
    predictWarning(data);   // 执行报警预测
}
在predictWarning(data)中{
    if(data reaches warning line){
        sendNotification2App(data); //向手机端发送推送
        sendNotification2Web(data); // 向web端发送推送
        sendSMS2APP(data);      //手机端发送短信
    }
}
在这里我不去讲具体是如何向web端发送推送， 如何发送短信。主要用到第三方平台

```


### 分析
入库和报警预测是没有直接联系，或者是不分先后顺序的， 同样在报警模块中， 向3个客户端发送信息也应该是没有联系的， 所以以上虽然可以实现功能， 但不符合代码的合理性。 


应该是怎么样的逻辑呢？ 如下图
![image](http://chuantu.biz/t6/141/1510794613x2061543562.png)

当数据事件触发， 发布到data EventBus 上， 入库和预警分别订阅这个eventBus, 就会触发这两个事件， 而在预警事件中， 将事件发送到warning EventBus 中， 由下列3个订阅的客户端进行发送消息。


### 如何实现
先来讲同步， 即订阅者收到事件后依次执行, 下面都是伪代码， 具体的入库细节等我在这里不提供。
```
@Component
public class DataHandler{
    
    @Subscribe
    public void handleDataPersisist(Data data){
        daoImpl.insertData2Mysql(data);
    }
    
    @Subscribe
    public void predictWarning(Data data){
        if(data is warning){ // pseudo code  如果预警
            Warning warning = createWarningEvent(data);  // 根据data创建一个Warning事件
            postWarningEvent(warning)
        }
    }
    
    protected postWarningEvent(Warning warning){
        EventBusManager.warningEventBus.post(warning);// 发布到warning event 上
    }
    
    @PostConstruct   // 由spring 在初始化bean后执行
    public void init(){
        register2DataEventBus();
    }
    
    // 将自己注册到eventBus中
    protected void register2DataEventBus(){
        EventBusManager.dataEventBus.register(this);
    }
    
}

@Component
public class WarningHandler{
    @Subscribe
    public void sendNotification2AppClient(Warning warning){
        JpushUtils.sendNotification(warning);
    }
    @Subscribe
    public void sendSMS(Warning warning){
        SMSUtils.sendSMS(warning);
    }
    @Subscribe
    public void send2WebUsingWebSocket(Warning warning){
        WebsocketUtils.sendWarning(warning);
    }
    
    @PostConstruct   // 由spring 在初始化bean后执行
    public void init(){
        register2WarningEventBus();
    }
    
    // 将自己注册到eventBus中
    protected void register2DataEventBus(){
        EventBusManager.warningEventBus.register(this);
    }
}


/**
 * 管理所有的eventBus
 **/
public class EventBusManager{
    public final static EventBus dataEventBus = new EventBus();
    public final static EventBus warningEventBus = new EventBus();
    
}



简化
// 我们发现每一个Handler都要进行注册，
public abstract class BaseEventBusHandler{
    
    @PostConstruct
    public void init(){
        register2EventBus();
    }
    private void register2EventBus(){
        getEventBus().register(this);
    }
    protected abstract EventBus getEventBus();
}
这样在写自己的eventBus只需要

@Component
public class MyEventBus extends BaseEventBusHandler{
    @Override
    protected abstract EventBus getEventBus(){
        retrun EventBusManager.myEventBus;
    }
}

在目前的应用场景下， 同步是我们不希望的， 异步场景也很容易实现。
只需要将EventBus 改成
 AsyncEventBus warningEvent = new AsyncEventBus(Executors.newFixedThreadPool(1))
 AsyncEventBus dataEventBus = new AsyncEventBus(Executors.newFixedThreadPool(3))
```

>当然还有很多很多场景我们利用发布和订阅模式， 在一定时候巧妙结合guavaEventBus, 这样我们单块架构的结构更加合理。 


至于`分布式的架构`， 我们可以使用MQ 如rabbitMq, kafka。 这里的篇章我会在后续给出。


如果大家感兴趣的话： 可以多去看看响应式编程， 有很多特性会吸引着你。  
如果大家想对发布和订阅设计模式是如何实现的， 其实也简单， 后续我会简单实现这种设计模式。 

在本篇最后， 还推荐大家去学习一下vert.x ， 后续我也会去补充。
在vert.x 中， 用到了eventbus（非guava eventBus）。




--- 
图片引用：  
图一： 来自百度百科