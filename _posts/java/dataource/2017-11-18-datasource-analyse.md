---
layout: post
title: "多数据源(问题分析篇)"
date: 2017-11-18 09:00:00 +0800
categories: java
tag: [datasource, transaction]
---
* content
{:toc}

在[上一篇文章]({{ '/2017/11/15/datasource' | prepend: site.baseurl}})中主要介绍了多数据源及其配置， 还介绍了多数据源事务的问题， 在本章中， 重点讲一下在分布式系统中多数据源会遇到的问题， 以及如何解决办法

<!-- more -->
**问题1：  分布式下，服务的消费者远程调用服务接口(Rpc), 对于服务的提供者是多数据源操作的情况下， 如何切换数据源？**

问题分析
对于rpc， 消费者调用服务接口应该就像本地调用服务一样， rpc 框架屏蔽了底层通讯的细节， 也就是说我们就会这样调用.
```
    service.invoke(param1, ... paramn)
```

如果如上篇多数据源将的那样操作， 对于controller层， 我们只需用配置拦截器在本地线程的设置key, 但是现在消费者和提供者是处于不同的进程， 无法通过设置本地线程的而达到跳库的目的的。 

这时候我们想， 要想进行数据源切换，就必须要让服务提供者知道我们要去哪个数据库进行操作？    

最简单的办法就是修改接口， **将要操作的数据库的key以参数传递**。 
大致上服务端暴露的接口如下
```

service.invoke(String dbKey, param...)

```
针对这种情况：服务提供者就需要在内部进行手动切换数据源。

再进一步想每一个service接口内部首先都需要进行切换数据源， 有没有方法能够避免这样的操作， 但是似乎增加一个参数是不可避免的了。此时， 我又想到了用动态代理， 对于dbKey这个参数前加入一个注解， 只要有这个注解的， 代理会为我进行切换数据源， 这样我岂不不用在每个service内部进行跳库了。 此时的service方法大致成：  

```
service.invoke(@DBKey String dbKey, params ...)

```

如何实现代理层呢？

此时， 代理需要做哪些事情呢？
1. 查看被代理者是否有带有@DBkey的注解
2. 如果有，则将@DBKey 的位置的参数值拿出进行跳库操作， 
3. 调用被代理者方法

下面使用spring aop进行切面配置： 大致思路如下
```
  public void toSkip(final JoinPoint point) throws Throwable {
        Method method = ((MethodSignature)point.getSignature()).getMethod(); //  获取被代理的方法
        Annotation[][] annotations = method.getParameterAnnotations();             // 获取参数列表的注解， 因为一个参数可能有多个注解所以返回值是二维数组
        Object[] args = point.getArgs();                            // 获取参数
        int position = -1;                                          // 设置
        if (annotations.length != 0) {                                        // 如果没有注解， 就不需要查找了
        
            position = getAnnotationPosition(annotations, DBKey.class);     // 查找注解在参数列表的位置
        }
        if (position!=-1){
            //
            Object db = args[position];                              // 提取相应的key
            if (db instanceof String){
                 DBContextHolder.setDBType((String) tenant);   // 此注解会覆盖DBContextHolder中的
            }else {
//                throw new IllegalArgumentExcetption();
            }
        }else {

        }
    }
    
写一个针对annotation的工具：
public class AnnotationUtils {

    /**
     * 查找annotation 在 annotation列表中的为位置。 
     */
    public static int getAnnotationPosition(Annotation[][] annotations, Class<? extends Annotation> annotation){
        for (int i=0; i< annotations.length; i++){
            if(isAnnotationContains(annotations[i], annotation)){
                return i;
            }
        }
        return -1;
    }

    private static boolean isAnnotationContains(Annotation[] annotations, Class<? extends Annotation> clazz) {
       for (Annotation anno:annotations){
            if (anno.annotationType() ==clazz){
                return true;
            }
       }
       return false;
    }
}

```

####  除此方法之外还有没有其他方法吗？  
了解到在rpc调用过程中， rpc 会将请求内容， 请求方法名，参数列表等进行封装， 往往在请求时会写入一个请求id， 在返回时结果会携带这个id。 其实还可以携带一些其他内容， 但是可能需要修改rpc框架， 在进行序列化时，从DBContextHolder中取出， 写入request中， 在response中取出， 设置服务端的dbContextHolder. 这样就感觉DBContextHolder 似乎是一个了。这样的好处是无需修改服务端的接口就可达到目的。 


#### 我还有另一个想法：   
既然request中的id和reponse中的id是一样的， 将其作为key存入分布式缓存中， 服务端从缓存中通过id取出切换的数据源的key，从而实现跳库。但是这种想法会存在很大问题就是， 每次请求的id都是唯一的， 那么在高并发的情况下，将会在缓存中存入大量一次性的key-value。


故在这种情况下， 还是推荐第一种方法， 通过动态代理减少跳库动作和业务代码的耦合性, 虽然在参数中增加了一定的耦合性， 不过还是可以接受的， 针对修改rpc源码， 尤其是已经写好的框架， 可能会比较麻烦， 不过应该在同步调用还是可以做到的。

