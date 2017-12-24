1 责任链
### 前言
写方法的原则要尽量做到一个方法只做一件事情， 这样做到职责单一， 日后对于代码的维护也非常容易。 并且代码的复用性也会很高。

在某些情况下， 对于某个事件或某个消息的处理是一个接一个的，  而又不想以下这种写法
```java
process(Message a){
    doSomethingA(a)
    doSomethingB(a)
    doSomethingC(a)
    doSomethingD(a)
}
```

以上这种写法很容易， 可以明显看到执行的顺序，也可以保证一个方法只做一件事情， 让后将这些事情连起来。 


#### 一些弊端
但是我们会发现， A的执行与b的执行没有什么关联， 如果A执行后根据A的结果判断是否执行以下的， 这时候我们就需要在process层加入额外的代码， 当项目复杂程度越来越高时， 就会发现process的代码会变得异常的臃肿。

#### 换一种思路
这时候我们想， 能否使得上一次的调用对下一次的调用**负责**， 而不是通过process这层来管理， 如下图显示

![image](http://upload.ouliu.net/i/20171215160527nt2jq.png)




#### 我们在哪里见过职责链这种模式
1. java web filter  
在java web开发中， 最早的应该就属filter了
实现Filter接口， 重写doFilter.
```
	public void doFilter(ServletRequest req, ServletResponse resp,
			FilterChain chian) throws IOException, ServletException {
        chian.doFilter(...)
	}
```


如果接触过socket编程， 对mina 或者netty框架熟悉的话， 责任链模式将更不陌生。 
在mina中: 
通过FilterChianBuilder, FilterChain, IoFilter 构成一个链。 创建方式如下
```
DefaultIoFilterChainBuilder builder = new DefaultIoFilterChainBuilder();
builder.addFirst("mdcInjectionFilter", new MdcInjectionFilter(MdcInjectionFilter.MdcKey.remoteAddress));
builder.addAfter("mdcInjectionFilter", "codExecutorFilter", new ExecutorFilter());
builder.addAfter("codExecutorFilter", "loggingFilter", new LoggingFilter());
builder.addAfter("loggingFilter", "codecFilter", codecFilter);
builder.addAfter("codecFilter", "executorFilter", executorFilter);
builder.addAfter("executorFilter", "writeExecutorFilter", new ExecutorFilter(IoEventType.WRITE));
DefaultIoFilterChain chain = new DefaultIoFilterChain();
builder.buildFilterChain(chain);

```


这样就构造了一条职责链， 每一个节点负责下面一个节点， 节点内执行一定的逻辑。 至于如何构造职责链可以看我的mina源码分析， 以及我参考mina的filterchain，搭建出恰好符合公司业务的处理链（processor chain)的部分代码。 


3. netty 中   
在netty中， 核心的业务逻辑有一系列ChannelHandler处理。
netty中分为 inBound 和 outBound, 在channelpipeline 中， inbound 和outbound channelhandler 处理读和写。
可以通过下面代码创建:
```
ChannelPipeline pipeline = ch.pipeline();
pipeline.addLast("handler1", firstHandler);  
pipeline.addFirst("handler2", new SecondHandler());  
pipeline.addLast("handler3", new ThirdHandler());  

````
当然在netty还有其他几个重要成员Channel, ChannelHandlerContext, EventLoop, EventLoopGroup, 这些不在这里阐述， 有兴趣的话可以看《netty in action》。

### 后话

职责链模式能让模块结构更加清晰， 节点与节点之间的修改影响会非常小。 但是并不适合所有的场景， 要结合场景分析。 
