# 讨厌的连接建立

在任何通讯时， 首先都需要建立连接。

不知道大家有没有这种感觉， 很不希望自己去手动建立一个连接， 如http， 数据库连接， 消息服务器连接等等。 

我们只需要关心拿一个有用的连接, 执行动作， 由于网络原因等问题失败抛出异常。

我们发现我们对连接并不熟悉。不知道大家会不会担心下列几个问题？

1. 连接什么时候建立
2. 连接如何复用
3. 连接断开如何处理
4. 连接资源如何回收



下面我举一些我们以往常用的一些连接：

### 数据库连接 jdbc 
在学习jdbc编程时， 我们会做以下几步：
```
Class.forName(driverClass);
DriverManager.getConnection(String url,String user,String pass) // 获取连接
createStatement();
statement.execute();
connect.close()
```

我们应该清除， 这样每次创建连接， 销毁连接其实是很消耗资源的，**在小并发情况下是没什么问题， 但对于高并发来讲却是一个灾难。**  
所以就有了连接池的概念， 连接池来管理这些连接，对于连接池的用法及详情 可以参考我的[druid连接池](http://note.youdao.com/noteshare?id=c2d690322b64e6d4788329939bebf841&sub=BEA46DEEBB3D4C5C8EB11767C9342618)//todo， 那么我们只需要向连接池‘借’和‘还’数据库的连接就好了。 

#### 连接池-datasource
如果有了连接池， 我们将会有以下简单调用， 连接的生命周期有连接池获取。
```
connection = dataSource.getConnection(); 
createStatement();
execute();
connection.close(); // 归还连接
```


### Http 
再来看看Http连接， 先来看看最传统的java.net包下的URLConnection, 在发送http请求时我们通常需要：
```
URLConnection connection = realUrl.openConnection();
connection.connect();
writeSomething(connection, "something")
in = new BufferedReader(new InputStreamReader(connection.getInputStream(),"UTF-8"));
res = readFromStream(in); // 同步执行
// **在读完之后只能关闭， 不能在执行写操作， 否则会报错**
connection.close()
```
这里我们看到， 每次读和写一次要创建一个连接， 建立http连接就需要3次握手。 




#### OkHttp 或 HttpClient
通过OkHttp[官网](http://square.github.io/okhttp/) 我们可以看到其介绍



```
HTTP/2 support allows all requests to the same host to share a socket.    // 对于同一个主机共享一个socket
Connection pooling reduces request latency (if HTTP/2 isn’t available). // 连接池化
Transparent GZIP shrinks download sizes. // 压缩
Response caching avoids the network completely for repeat requests. // 返回缓存

```

示例 
```
OkHttpClient client = new OkHttpClient(); // 全局可以一个 

String run(String url) throws IOException {
  Request request = new Request.Builder()
      .url(url)
      .build();         //请求参数

  Response response = client.newCall(request).execute(); // 执行
  return response.body().string();  // 返回
}

```

> 至于它时怎么做到上述的优点的， 大家可以自行取研究。













### 消息服务器kafka
作为Kafka的生产者和消费者，

// todo









