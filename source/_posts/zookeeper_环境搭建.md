---
title: Zookeeper 安装与集群搭建
categories:
  - Zookeeper
top: false
date: 2020-04-08 10:14:08
tags:
- Zookeeper
keywords:
description: 
---

## 安装单机版Zookeeper

### 下载Zookeeper安装包
  > 在 [Zookeeper](http://zookeeper.apache.org) 官网(http://zookeeper.apache.org )下载

### 上传安装包
  > 将下载的`Zookeeper`安装包上传到主机的 **/usr/tools** 目录

### 安装配置Zookeeper
- 解压安装包
  ```sh
  cd /usr/tools
  tar -xzvf /usr/tools/zookeeper-3.4.9.tar.gz -C /usr/apps
  ```
- 创建软连接
  ```sh
  ln -s /usr/apps/zookeeper-3.4.9/ /usr/apps/zk
  ```

- 复制配置文件
  > 复制Zookeeper **安装目录**下的 *conf*目录中的`zoo_sample.cfg`文件，并命名为`zoo.cfg`
  ```sh
  cp zoo_sample.cfg zoo.cfg
  ```
- 修改配置文件
  > 修改数据存放目录
  ```sh
  vim /usr/apps/zk/conf/zoo.cfg
  // 修改dataDir的值
  dataDir=/usr/data/zk
  ```

- 新建数据存放目录
  ```sh
  mkdir -p /usr/data/zk
  ```

- 注册bin目录
  ```sh
  vim /etc/profile
  // 在末尾添加如下zk配置
  export ZK_HOME=/usr/apps/zk
  export PATH=$ZK_HOME/bin:$PATH
  ```

- 重新加载profile文件
  ```sh
  source /etc/profile
  ```

### 操作Zookeeper
  - 开启zk
    ```sh
    zkServer.sh start
    ```

  - 查看zk状态
    ```sh
    zkServer.sh status
    ```

  - 重启zk
    ```sh
    zkServer.sh restart
    ```

  - 停止zk
    ```sh
    zkServer.sh stop
    ```

## 搭建Zookeeper集群
> 下面要搭建一个由**4**台zk构成的zk集群，其中**1**台Leader，**2**台Follower，**1**台Observer。

### 克隆并配置第一台主机
- 克隆前面配置的Zookeeper主机后，要修改如下配置文件
    1. 修改主机名：/etc/hostname
    2. 修改网络配置：/etc/sysconfig/network-scripts/ifcfg-ens33

- 创建`myid`文件
  > 在/usr/data/zk目录中创建表示当前主机编号的myid文件，该编号为当前主机在集群中的唯一标识。
  ```sh
  echo 1 > /usr/data/zk/myid
  cat /usr/data/zk/myid
  ```

- 修改zoo.cfg
  > 在`zoo.cfg`文件中添加zk集群节点列表。
  ```sh
  vim /usr/apps/zk/conf/zoo.cfg
  //添加集群列表，如下
  server.1=192.168.10.100:2888:3888
  server.2=192.168.10.101:2888:3888
  server.3=192.168.10.102:2888:3888
  server.4=192.168.10.103:2888:3888:observer
  ```

### 克隆并配置另外两台
> 克隆上面配置的第一台**集群**Server

  - 修改主机名：/etc/hostname
  - 修改网络配置：/etc/sysconfig/network-scripts/ifcfg-ens33
  - 修改 **myid** 值与zoo.cfg中指定的主机编号相同

    ```sh
    vim /usr/data/zk/myid
    ```

### 克隆并配置第四台主机
> 克隆方式和前面两台一样，但第四台是作为<span color="green">Observer</span>的主机,需要修改zoo.cfg文件

  ```sh
  //在server.1=192.168.10.100:2888:3888 上面添加如下配置
  peerType=observer
  ```

### 启动zk集群
> 使用`zkServer.sh start` 命令，逐个启动每一个Zookeeper节点主机


## 伪集群的搭建
> 这里搭建的伪集群和上面的集群一样，是由四个zk服务组成，其中第四个为Observer。伪集群与珍视集群的搭建差不多。

### 复制配置文件
> 这里需要4个配置文件，都放在zk安装目录的conf目录下，命名分别为zoo1.cfg、zoo2.cfg、zoo3.cfg、zoo4.cfg。

### 修改配置文件
> 每个配置文件配置的dataDir分别为/usr/data/zk1、/usr/data/zk2、/usr/data/zk3、/usr/data/zk4
> clientPort=<四个配置文件不能相同> (2181、2182、2183、2184)
> **peerType=observer**   //  **Observer**主机(server.4)需要配置这个，其余三台 **不需要**
> server.1=192.168.10.111:2666:3666
> server.2=192.168.10.111:2777:3777
> server.3=192.168.10.111:2888:3888
> server.4=192.168.10.111:2999:3999:observer

### 创建数据目录
  ```sh
  mkdir -p /usr/data/zk1
  mkdir -p /usr/data/zk2
  mkdir -p /usr/data/zk3
  mkdir -p /usr/data/zk4
  ```

### 创建myid文件
> 分别在/usr/data/zk1、/usr/data/zk2、/usr/data/zk3、/usr/data/zk4文件夹里创建myid文件，内容分别为1、2、3、4 。
  ```sh
  echo 1 > /usr/data/zk1/myid
  echo 2 > /usr/data/zk2/myid
  echo 3 > /usr/data/zk3/myid
  echo 4 > /usr/data/zk4/myid
  ```

### 伪集群启动
> 伪集群的启动需要指定每台Server启动所使用的配置文件。进入zk安装目录
  ```sh
  cd /usr/apps/zk
  bin/zkServer.sh start conf/zk1.cfg
  bin/zkServer.sh start conf/zk2.cfg
  bin/zkServer.sh start conf/zk3.cfg
  bin/zkServer.sh start conf/zk4.cfg
  ```

> 查看各个server的状态
  ```sh
  cd /usr/apps/zk
  bin/zkServer.sh status conf/zk1.cfg
  bin/zkServer.sh status conf/zk2.cfg
  bin/zkServer.sh status conf/zk3.cfg
  bin/zkServer.sh status conf/zk4.cfg
  ```