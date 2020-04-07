---
title: Zookeeper Curator客户端
categories:
  - Zookeeper
top: false
date: 2020-04-07 15:12:07
tags: 
- Zookeeper
- Curator
keywords: Zookeeper,Curator
description: 
---

## 简介
> `Curator` 是 **Netflix** 公司开源的一套`zk`客户端框架，与`ZKclient`一样，其也封装了zk原生API，其目前已经称为Apache的顶级项目。同时Curator还提供了一套**易用性、可读性** *更强* 的`Fluent风格`的客户端API框架。

## API介绍
> 这里主要以Fluent风格客户端API为主进行介绍。

### 创建会话

#### 普通API创建newClient()
> 在CuratorFrameworkFactory类中提供了两个**静态方法**用于完成会话的创建

- `newClient`(String connectString,int sessionTimeoutMs,int connectionTimeoutMs,RetryPolicy retrypolicy)
- `newClient`(Stirng connectString,RetryPolicy retryPolicy)

|参数名称|意义|
|----------------------|-------------------------------------------------------------------------------|
|`connectString`|指定zk服务器 *列表* ，由英文逗号分开的host:port字符串组成|
|`sessionTimeoutMs`|设置**会话超时**时间，单位毫秒，默认`60`秒|
|`connectionTimeoutMs`|设置**连接超时**时间，单位毫秒，默认`15`秒|
|`retryPolicy`| **重试策略**，内置了四种策略：`ExponentialBackoffRetry`、`RetryNTimes`、`RetryOneTime`、`RetryUntilElapsed`|

#### Fluent风格创建
```Java
public class FluentTest{
  public static void main(String[] args) throws Excepetion{
    //创建重试策略对象；第一秒重试一次，最多重试3次
    ExponentialBackoffRetry retryPolicy = new ExponentialBackoffRetry(1000,3);

    //创建客户端
    CuratorFramework client = CuratorFrameworkFactory
                                  .builder()
                                  .connectString("zkOS:2180")
                                  .sessionTimeoutMs(5000)
                                  .connectionTimeoutMs(3000)
                                  .retryPolicy(retryPolicy)
                                  .namespace("drunk")
                                  .build();
    //开启客户端
    client.start();
  }
}
```

### 创建节点
> 下面使用的client为前面所创建的`Curator`客户端实例

- 创建一个<span style="color:blue;">**持久**</span>节点，初始内容为空
  `client.create().forPath(path)`

- 创建一个<span style="color:blue;">**持久**</span>节点，附带初始内容；`Curator`在指定数据内容时，只能使用`byte[]`作为方法参数。
  `client.create().forPath(path,"mydata".getBytes())`

- 创建一个<span style="color:green;">**临时**</span>节点，初始内容为空。 **说明**：`CreateMode`为枚举类型
  `client.create().withMode(CreateMode.EPHEMERAL).forPath(path)`

- 创建一个<span style="color:green;">**临时**</span>节点，并自动*递归*创建 **父节点**; 若指定的节点多级父节点均不存在，则会自动创建。
  `client.create().creatingParentsIfNeeded().withMode(CreateMode.EPHEMERAL).forPath(path)`

### 删除节点
> delete()

- 删除一个节点，只能删除叶子节点，其**父节点** <span style="color:red;">不会</span>被*删除*
  `client.delete().forPath(path)`

- 删除一个节点，并*递归删除*其**所有子节点**
  `client.delete().deletingChildrenIfNeeded().forPath(path)`

### 更新数据
> setData()

- 设置一个节点的数据内容，返回值为`org.apache.zookeeper.data.Stat`状态对象
  `client.setData().forPath(path,newData)`

### 检查节点是否存在
> checkExists()

- 检查节点是否存在，返回值为`org.apache.zookeeper.data.Stat`状态对象，若stat为**null**，说明该节点**不存在**，否则说明该节点是存在的
  `Stat stat = client.checkExists().forPath(path)`

### 获取节点数据内容
> getData()

- 读取**一个节点**的数据内容,返回值为`byte[]`数组
  `byte[] data = client.getData().forPath(path)`

### 获取子节点列表
> getChildren()

- 读取一个节点的所有子节点
  `List<String> childrenNames = client.getChildren().forPath(path)`

### watcher注册
> usingWatcher()

> curator绑定watcher的操作有三个：checkExists()、getData()、getChildren()。这三个方法的共性是，它们都是用于获取的，这三个操作用于watcher注册的方法是相同的，都是usingWatcher()。
> usingWatcher(CuratorWatcher watcher)
> usingWacther(Watcher watcher)

> 这两个方法中的参数CuratorWatcher与Watcher都为接口，这两个接口均包含一个process()方法，它们的区别是CuratorWatcher中的process()方法能够抛出异常，这样就可以在日志中记录该异常。

- 监听节点的存在性变化
  ```Java
  Stat stat = client.checkExists().usingWatcher(
    (CuratorWatcher)event -> {System.out.println("节点存在性发生变化")}
  ).forPath(path)
  ```

- 监听节点的内容变化
  ```Java
  Byte[] data = client.getData().usingWatcher(
    (CuratorWatcher)event -> {System.out.println("节点数据内容发生变化")}
  ).forPath(path)
  ```

- 监听节点子节点列表的变化
  ```Java
  List<String> sons = client.getChildren().usingWatcher(
    (CuratorWatcher)event -> {System.out.println("节点的子节点列表发生变化")}
  ).forPath(path)
  ```

  ### 代码演示

  ```Xml
  <!-- curator依赖 -->
  <dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-framework</artifactId>
    <version>2.12.0</version>
  </dependency>
  ```

  ```Java
  public class FluentTest{
    public static void main(String[] args) throws Excepetion{
      //创建重试策略对象；第一秒重试一次，最多重试3次
      ExponentialBackoffRetry retryPolicy = new ExponentialBackoffRetry(1000,3);

      //创建客户端
      CuratorFramework client = CuratorFrameworkFactory
                                    .builder()
                                    .connectString("zkOS:2180")
                                    .sessionTimeoutMs(5000)
                                    .connectionTimeoutMs(3000)
                                    .retryPolicy(retryPolicy)
                                    .namespace("drunk")
                                    .build();
      //开启客户端
      client.start();

      //指定要创建和操作的节点，注意，其是相对于/logs节点的
      String nodePath = “/host”；

      //创建节点
      String nodeName = client.create().forPath(nodePath,"drunk".getBytes());
      System.out.println("新创建的节点名称为： " + nodeName);

      //获取数据内容并注册watcher
      byte[] data = client.getData().usingWatcher((CuratorWatcher)event -> {
        System.out.println(event.getPath + "数据内容发生变化");
      }).forPath(nodePath);
      System.out.println("节点数据内容为： " + new String(data));

      //更新数据内容
      client.setData().forPath(nodePath,"new---Drunk".getBytes());
      //获取更新后的数据内容
      byte[] newData = client.getData().forPath(nodePath);
      System.out.println("更新过的数据内容为： " + new String(newData))；

      //删除节点
      client.delete().forPath(nodePath);

      //判断节点是否存在
      Stat stat = client.checkExists().forPath(nodePath);
      boolean isExists = stat == null ? false : true;
      System.out.println(nodePath + "节点还存在吗？" + isExists)
    }
  }
  ```

