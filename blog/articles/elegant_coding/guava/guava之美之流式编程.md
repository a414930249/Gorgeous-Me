用过guava api 的人一定会觉得它很方便， 我也不例外， 除此之外， 我在想， 使用guava 除了有优雅的编程方式， 对我们还有什么帮助呢? 
为此对于下列问题做了个研究：

```
    假设有这么个集合
    1    List a = Lists.newArrayList(1, 2, 3，4，5，6，7，8，9，0，11，12，13); 
    2    FluentIterable<Integer> res = FluentIterable.from(a).filter(
            new Predicate<Integer>() {
            @Override
            public boolean apply(Integer i) {
                return i > 3;

            }
        }).transform(new Function<Integer, Integer>() {
            @Override
            public Integer apply(Integer i) {
                return i + 3;
            }
        });
    3    List  b = res.limit(1).toList();

```

    我们发现， 第2步并没有立刻执行， 而在第3步的时候才执行toList时候才会执行filter和transform里的方法。
    并且我们调用了limit(1), 在得到1个符合的就结束， 并没有遍历整个list最后才取出4个
    
    同时这种现象这让我想起了spark中的懒加载， transformation 运算都是懒加载， 只有在执行actions才会讲任务发送给worker执行。
    
    
那是如何做到的呢？让我们继续探究。

首先guava 中有 Iterables 和 FluentIterables 
可以看到， 对于filter, transform这些transformation, 
FluentIterable的处理仅仅是包装了Iterble 对象
```
from(Iterables.`doTransform`(iterable, params));
如： 
public final FluentIterable<E> filter(Predicate<? super E> predicate) {
    return from(Iterables.filter(iterable, predicate));
  }
  
而FluentIterable 的 from 返回的是FluentIterable 对象
public static <E> FluentIterable<E> from(final Iterable<E> iterable) {
    return (iterable instanceof FluentIterable) ? (FluentIterable<E>) iterable
        : new FluentIterable<E>(iterable) {
          @Override
          public Iterator<E> iterator() {
            return iterable.iterator();
          }
        };
  }
  
而Iterbles 的transformation 方法返回的是FluentIterble的子类对象。 
比如： 
  public static <T> Iterable<T> limit(
      final Iterable<T> iterable, final int limitSize) { // 这里的iterable 非常关键
    checkNotNull(iterable);
    checkArgument(limitSize >= 0, "limit is negative");
    return new FluentIterable<T>() {
      @Override
      public Iterator<T> iterator() {
        return Iterators.limit(iterable.iterator(), limitSize);
      }
    };
  }
这就难怪这些方法为什并不会马上执行了。
**并且会把前面一个iterable 传进来这非常关键**

```

而对于action算子， 如size() 计算大小
```java
  public final int size() {
    return Iterables.size(iterable);
  }
  而对于guava 的Iterator 将执行：
  
    public static int size(Iterator<?> iterator) {
    int count = 0;
    while (iterator.hasNext()) {
      iterator.next();
      count++;
    }
    return count;
  }
  
```

那么关键就是next()和hasNext()方法了，之前说过，transformation的算子返回的都是Iterator, 而这里的iterator 是经过重写的， 
为什么可以像流水一样调用，而不是第一个filter全部结束传给下一个transform的原因如下：
    
    transformation 算子链式调用， 会将自身传到下一个transformation算子
    在执行iterator.hasNext()是执行最后一个算子的hasNext 因为有前面的所以会执行前面的hasNext,
    同理 next() 方法也一样。 比如limit 方法
```java
  public static <T> Iterator<T> limit(
      final Iterator<T> iterator, final int limitSize) {
    checkNotNull(iterator);
    checkArgument(limitSize >= 0, "limit is negative");
    return new Iterator<T>() {
      private int count;

      @Override
      public boolean hasNext() {
        return count < limitSize && iterator.hasNext(); // 调用前面一个算子的hasNext()
      }

      @Override
      public T next() {
        if (!hasNext()) {
          throw new NoSuchElementException();
        }
        count++;
        return iterator.next();  // 调用前面一个算子的next()
      }

      @Override
      public void remove() {
        iterator.remove();
      }
    };
  }
 
```

这样就形成链式调用。 使得从iterator 中获取到一个next， 能够如流水一般往下传递。


随着源码的跟踪， 我们终于解开了这层神秘面纱。 

guava之美远远不止这些， 还有很多我们要去探究。
    



## OTHER
链式调用

为此， 在业务中， 也希望设计出链式调用来使代码更加经凑可读， 下面举一个利用构造者设计模式的链式调用。

我们知道，对于map作为参数， 往往有一下这么几步
```
Map map = new HashMap();
map.put(K,V);
map.put(K,V)
```

```java
public class MapEntity<K,V> extends HashMap{

    private MapEntity(HashMapBuilder builder){
        putAll(builder.map);
    }

    public static class  HashMapBuilder<K,V>{

        private HashMap<K,V> map;

        public HashMapBuilder() {
            this.map = Maps.newHashMap();
        }

        public HashMapBuilder putKV(K key1, V value1){
            map.put(key1,value1);
            return this;
        }
        public HashMapBuilder putKV(K key1, V value1,K key2, V value2){
            map.put(key1,value1);
            map.put(key2,value2);
            return this;
        }
        public HashMapBuilder putKV(K key1, V value1, K key2, V value2, K key3, V value3){
            map.put(key1,value1);
            map.put(key2,value2);
            map.put(key3,value3);
            return this;
        }
        public HashMapBuilder putAll(Map map){
            this.map.putAll(map);
            return this;
        }
        public HashMap<K,V> build(){
            return new MapEntity(this);
        }
    }
    public static HashMapBuilder newBuilder(){
        return new HashMapBuilder();
    }
}

调用示例：
Map map = MapEntity.newBuilder().putKV(K,V).putKV(K,V)....build();

```