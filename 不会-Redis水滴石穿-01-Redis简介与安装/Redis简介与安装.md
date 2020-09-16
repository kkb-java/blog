# Redis学习笔记（一）：简介与安装

[TOC]

## 一、Redis介绍

### 1.1、什么是NoSQL

NoSQL，可以拆开理解，**即 Not-Only SQL （不仅仅是 SQL ）**，泛指**非关系型的数据库**。 
关系型数据库大家肯定都不陌生了，mysql、oracle、sql server等等等等。

关系型数据库最典型的数据结构是表（关系表也叫二维表），由二维表及其之间的联系所组成的一个数据组织，说白了就是一种有行有列的数据库。

针对于关系型数据库的缺点，作为良好的补充，nosql应景而生，解决了高并发、高可用、高可扩展、大数据存储问题而产生的数据库解决方案。

### 1.2、什么是Redis

Redis 是一个开源（BSD许可）的，内存中的数据结构存储系统，它可以用作数据库、缓存和消息中间件。 它支持多种类型的数据结构，如 字符串（strings）， 散列（hashes）， 列表（lists）， 集合（sets）， 有序集合（sorted sets） 与范围查询， bitmaps， hyperloglogs 和 地理空间（geospatial） 索引半径查询。 Redis 内置了 复制（replication），LUA脚本（Lua scripting）， LRU驱动事件（LRU eviction），事务（transactions） 和不同级别的 磁盘持久化（persistence）， 并通过 Redis哨兵（Sentinel）和自动 分区（Cluster）提供高可用性（high availability）。

### 1.3、Redis使用场景

内存数据库（登录信息、购物车信息、用户浏览记录等） 

缓存服务器（商品数据、广告数据等等）（É多使用） 

解决分布式集群架构中的 session 分离问题（ session 共享） 

任务队列（秒杀、抢购、12306等等） 

分布式锁的实现 

支持发布订阅的消息模式 

应用排行榜(有序集合) 

网站访问统计 

数据过期处理（可以精确到毫秒）

### 1.3、Redis官网

官网地址：http://redis.io/ 

中文官网地址：http://www.redis.cn/ 

下载地址：http://download.redis.io/releases/

## 二、Redis安装与调试

这里使用CentOS  作为安装环境

1、需要安装c语言 gcc 、wget环境（如已安装可以略过此步）

```java
yum  install -y gcc-c++   
yum  install -y wget
```

2、下载并解压，可以自己指定下载和解压的目录，这里使用路径：/opt/tools/,解压后如图

```
wget http://download.redis.io/releases/redis-5.0.8.tar.gz
tar -zxf redis-5.0.8.tar.gz  
```

![1589964090544](C:\Users\不会\AppData\Roaming\Typora\typora-user-images\1589964090544.png)

3、编译安装,进入redis-5.0.8目录，用make命令安装。这里指定安装目录为/opt/apps/redis，默认安装路径为：/usr/local/bin目录。安装后进入/opt/apps/redis/bin文件夹后目录如图，

```
cd redis-5.0.8
make  &&  make install PREFIX=/opt/apps/redis
```

![1589964261863](C:\Users\不会\AppData\Roaming\Typora\typora-user-images\1589964261863.png)

其中

**redis-benchmark**是官方自带的Redis性能测试工具；

**redis-check-aof**和**redis-check-rdb**用来修复aof和rdb文件的语法错误；

**redis-cli** 是客户端命令行工具，可以通过redis-cli-h 主机 -p端口号连接到指定Redis服务器，比如

./redis-cli -h 127.0.0.1 -p 6379；

**redis.conf** 是redis的配置文件

**redis-sentinel** 是Redis哨兵启动程序

**redis-server** 是Redis服务端启动程序

4、将压缩包中的配置文件拷贝到安装目录下的bin文件夹中

```java 
cp /opt/tools/redis-5.0.8/redis.conf   /opt/apps/redis/bin/
```

5、redis服务的启动

前端启动，然后通过redis-cli连接Redis服务器

```java
方式一：这种方式启动的时候，当前窗口不能执行其它操作，否则会停止redis服务。
./redis-server 
//需要重新开个会话窗口，来到redis的bin目录下执行下面的命令
./redis-cli  //默认连接127.0.0.1 的6379端口；
​```

方式二：在命令后面加& ，表示设置此进程为后台进程
./redis-server &
./redis-cli  
```

后端启动

修改redis.conf, 使用vim redis.conf编辑配置文件

```java
 # 将`daemonize`由`no`改为`yes`  ，此时启动redis会在后台运行
 daemonize yes    
 
 # bind是绑定本机的IP地址，也就是本机的网卡对应的IP地址，每一个网卡都有一个IP地址，如果指定了bind，则说明只允许来自指定网卡的Redis请求。
 #我们可以试一下bind除了127.0.0.1和0.0.0.0之外的任何非本机IP地址，然后重启redis，服务会启动不了。
 #这里我们选择注释它，   
 # bind 127.0.0.1  
 
 # 是否开启保护模式，由yes该为no，此时外部网络可以直接访问
 protected-mode no  
```

然后启动redis服务，连接客户端

```java
./redis-server redis.conf   //启动redis服务
./redis-cli					//连接客户端
```

6、redis测试，set 、 get key1，

```java
127.0.0.1:6379> set key1 hello
OK
127.0.0.1:6379> get key1
"hello"
127.0.0.1:6379> 
```

7、Redis的服务关闭

```java
./redis-cli shutdown
```

到这里，我们的Redis就已经安装成功了，在下一节，我们来介绍Redis的使用！

# 每日一皮：

我能抵御一切！除了诱惑。。。