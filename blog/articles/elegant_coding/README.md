## 前言
作为一名优秀的程序员， 不仅要有严谨的思维， 敏捷的开发，还要不断去追求优雅的代码。  
很多时候， 我们去看别人的代码， 会觉得头疼， 尽管在web开发中已经有诸如MVC模式开发， 使得代码的结构有一定的结构， 代码易于维护， 但在项目中，会发现service层变得异常臃肿，没有采用一定的设计模式及代码规范， 使得代码“惨不忍睹”。

下面就让我谈一谈如何才能写出质量好的代码：
并且我会结合一些框架的源码， 来看看别人优秀的代码是怎样练成的。

## 优雅代码之良好的命名规范  

    除了boolean 用 is开头，变量名用引文，驼峰命名这些规范以外，还可以有：  
    boolean：   
        authorizationCachingEnabled 以enabled结尾表示能够，一般用作开关  
        restartNeeded   以needed结尾  
        shouldTrack     以should 开头  
        有时候直接以一个单词作为变量(不一定非要用前缀或后缀)： boolean running 
        
    方法名：
    对于方法，我们不要吝啬我们的双手， 总感觉节俭是正确的，
    而在方法名， 我们可以通过驼峰来表达方法的含义：比如
    Cache<Object, AuthorizationInfo> getAvailableAuthorizationCache()
    我们一看就知道是获取有用的授权缓存。


## 优雅代码之guava api (java8 用户可忽略)  
    在开发中， 要善于别人写好的library, 一是提高开发效率， 同时别人良好的封装使得自己的代码更易读懂， 屏蔽很多细节。
    
    在这里我要介绍guava api 之函数表达式。
    函数式编程有其优美的表达方式， 如python，javascript, scala的lambda表达式，在java8 中也支持了stream， lamdba表达式，然而对于java7的程序员来讲， 无疑是一大遗憾。 不过， 在众多的api中， google 的guava api 给我们带来了一丝欣喜。 
```
在java8 中
    比如对于集合的操作, 进行偶数过滤
    list = Arrays.asList(1,2,3).stream().filter((i)->i%2==0).collect(Collectors.toList());


在guava中
    FluentIterable.from(list).filter(new Predict<Integer>(){
        @Override
            public boolean apply(Integer i) {
                return i%2==0;
            }
    }).toList();

```
##  优雅代码之极简开发Lombok   

    代码不是越写越多才好， 重复， 冗余的代码我们应该尽量避免。
    对于实体类（java bean） 我们总是要写get, set方法， 无参构造等。 
    虽然如idea， eclipse 为我们能够自动生成这些代码， 但我们打开某个实体类时， 会觉得代码好长， 而我们都知道这些方法就是get和set方法。
    接下来就要讲一个利器了-- lombok
    
    lombok:
        利用注解的形式， 在编译时来为我们生成对应的代码
        
    我们用的最多的就属@Data， 在类上申明即可，会自动生成所有字段的get和set方法,
    包含了@ToString，@EqualsAndHashCode，@Getter / @Setter和@RequiredArgsConstructor的功能  
    
 ```java   
@Data
public class User {
         private Integer id;
         private String name;
}
 ```
    还有一个注解在这里提一下:@Slf4j  
    相当于 Logger logger = LoggerFactory.getLogger(ClassName.class);

    对于其他注解 在这里就不强调了
    @Getter / @Setter
    @ToString

    
    
>  想更深的了解lombok， 可以去 [官网](https://projectlombok.org/)上了解  
>  对于使用idea的用户，需要安装一个插件及打开annotation processing， 详见[配置](http://blog.csdn.net/zhglance/article/details/54931430)




##  优雅代码之巧用设计模式

在日常开发中，仅仅靠”过程式“编程--一步步按照逻辑写下来，我们能完成日常的开发任务。而作为优秀的coder， 不仅完成开发任务， 还要让代码像搭建房子一样， 有良好的结构。 在这里我想简单谈一下设计模式，选择合适的设计模式， 能让我们的代码更加有结构化。在众多的设计模式中， 不要滥用设计模式， 选择合适的设计模式才是最重要的。 

      
有一些设计模式是为了提高性能的， 比如 flyweight, singleton  
有一些设计模式则是为了降低耦合性： 比如publish/subscribe，dynamic proxy;  
有一些设计模式则是为了让代码更加内聚：builder  
有些设计模式是为了更加有结构：Adapter, Facade 

我们可以将相似的功能以chain的形式组合在一起。
我们可以将冗长的if else 在子类中实现。


