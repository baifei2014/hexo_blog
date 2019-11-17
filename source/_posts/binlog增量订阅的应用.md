---
title: Mysql binlog增量订阅的应用
tags: [mysql, binlog, kafka]
date: 2019-11-18 00:16:17
---

## 引言
在业务开发中，我们可能会需要将Mysql中的某些表数据同步进ES中提高查询效率。但是当数据表变化时，还要将新数据同步进ES。最直接的办法就是在修改表数据时，直接去调整ES。这样的问题是会造成和业务耦合严重，会出现很多无关业务的代码，那么有什么办法可以解决这个问题，自动获取Mysql表数据变化，然后帮我们同步进ES？下面我们来介绍下Mysql中的binlog日志文件。

## binlog介绍

binlog是一种包含了数据表所有变化事件的二进制文件，它有两个重要的用途：

- 主从复制，主数据库发送包含数据表变化事件的binlog文件给从数据库，从数据库执行这些事件，就能得到和主数据库一样的数据。
- 数据恢复，还原备份后，会重新执行备份后binlog文件中的事件，binlog会保持数据库从备份起就处于最新状态。

通过上面的了解，我们可以知道，binlog中包含了数据表变化的所有事件，通过执行这些事件，我们就可以实现主从复制与数据恢复，说白了，binlog就保存了类似的增删改操作，那我们如果通过获取binlog文件，是不是就能实现当Mysql数据表变化时，实时更新ES了。下面就让我们来了解一个binlog增量订阅的工具，阿里巴巴开源项目canal。

## canal介绍

canal是阿里巴巴内部为了满足跨机房同步的业务需求，开发出来的数据库增量订阅工具。主要用途是基于MySQL数据库增量日志分析，提供增量数据订阅消费。那canal是如何实现增量订阅的，这还要了解下MySQL主从复制的原理：

MySQL主从复制原理：

- MySQL主数据库将数据变更写入binlog文件。
- MySQL从数据库将主数据库binlog文件拷贝到它的中继日志。
- MySQL从数据库执行binlog文件中的事件，获得和主数据一样的数据。

canal模拟MySQL主从复制原理，将自己伪装成MySQL slave，向master发送dump请求。master收到请求后，开始向slave推送binlog文件。最后canal解析这些binlog文件。这些解析到的数据就是增量内容。

## canal使用教程
下面就来介绍下canal结合kafka的快速使用教程。

### 安装启动kafka

#### 第一步下载解压代码

```shell
cd /usr/local/kafka
```

#### 启动kafka

kafka依赖zookeeper，在启动kafka之前，先要启动zookeeper，kafka解压包里已经包含了zookeeper，所以直接启动就行了：

```shell
bin/zookeeper-server-start.sh config/zookeeper.properties
```

启动完zookeeper，可以接着启动kafka了：

```shell
bin/kafka-server-start.sh config/server.properties &
```

到这里，我们kafka已经算是准备好了，下面就剩启动canal了。

### 安装启动canal

在安装启动canal之前，我们还需要首先确认MySQL已经开启了binlog，并且binlog格式是行模式，配置如下：

```shell
[mysqld]
log-bin=mysql-bin
binlog-format=ROW
server_id=1
```

前面讲了，canal是伪装自己是MySQL的slave，所以我们首先要创建一个具有slave权限的用户，命令如下：

```shell
CREATE USER canal IDENTIFIED BY 'canal'; 
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
FLUSH PRIVILEGES;
```

首先下载解压最新的canal包：

```shell
cd /usr/local/canal
```

然后修改配置参数，打开conf/example/instance.properties文件，编辑数据库用户名和密码，并且设置消息队列的topic。

修改canal配置文件，打开conf/canal.properties文件，编辑serverMode为kafka，并且设置kafka端口。


最后启动canal命令如下：

```shell
sh bin/startup.sh
```

### 验证是否成功

canal和kafka都已经安装成功了，那怎么看看是否生效呢？我们验证下，刚才我们增量订阅的是test库里的sales_scripts表，那我们就删除新增数据，看消息是否会入到kafka里，首先我们删除一条记录，这里我们查看kafka消息，命令如下：

```shell
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092  --from-beginning --topic test.sales_scripts
```

我们会发现有一条消息，内容如下：

```json
{"data":[{"id":"6044","keyword":"中国朝代","content":"收到~如果这边您没有其他补充的话，那我去给您根据经验设计安排了~请稍等，此外 您之前在网上有对咱们无二之旅的服务模式和特色路书有初步了解吗？需要为您简单做个介绍吗？","category":"无二之旅服务模式介绍 收费 路书","created_at":"2018-04-22 11:09:49","updated_at":"2018-08-01 14:55:02","role_id":null,"staff_id":"189","suggestion":"秦朝","regenerator_id":"189","sub_type":"text","stage":null,"deleted_at":null,"type":"script","scope":"public","country_name":null}],"database":"test","es":1573981175000,"id":1,"isDdl":false,"mysqlType":{"id":"int(11)","keyword":"varchar(191)","content":"text","category":"varchar(191)","created_at":"datetime","updated_at":"datetime","role_id":"int(11)","staff_id":"int(11)","suggestion":"varchar(191)","regenerator_id":"int(11)","sub_type":"varchar(191)","stage":"varchar(191)","deleted_at":"datetime","type":"varchar(255)","scope":"varchar(255)","country_name":"varchar(255)"},"old":null,"pkNames":["id"],"sql":"","sqlType":{"id":4,"keyword":12,"content":2005,"category":12,"created_at":93,"updated_at":93,"role_id":4,"staff_id":4,"suggestion":12,"regenerator_id":4,"sub_type":12,"stage":12,"deleted_at":93,"type":12,"scope":12,"country_name":12},"table":"sales_scripts","ts":1573981463457,"type":"DELETE"}
```

然后新增一条记录，又会产生一条新的消息：

```json
{"data":[{"id":"6044","keyword":"中国朝代","content":"收到~如果这边您没有其他补充的话，那我去给您根据经验设计安排了~请稍等，此外 您之前在网上有对咱们无二之旅的服务模式和特色路书有初步了解吗？需要为您简单做个介绍吗？","category":"无二之旅服务模式介绍 收费 路书","created_at":"2018-04-22 11:09:49","updated_at":"2018-08-01 14:55:02","role_id":null,"staff_id":"189","suggestion":"秦朝","regenerator_id":"189","sub_type":"text","stage":null,"deleted_at":null,"type":"script","scope":"public","country_name":null}],"database":"test","es":1573981727000,"id":2,"isDdl":false,"mysqlType":{"id":"int(11)","keyword":"varchar(191)","content":"text","category":"varchar(191)","created_at":"datetime","updated_at":"datetime","role_id":"int(11)","staff_id":"int(11)","suggestion":"varchar(191)","regenerator_id":"int(11)","sub_type":"varchar(191)","stage":"varchar(191)","deleted_at":"datetime","type":"varchar(255)","scope":"varchar(255)","country_name":"varchar(255)"},"old":null,"pkNames":["id"],"sql":"","sqlType":{"id":4,"keyword":12,"content":2005,"category":12,"created_at":93,"updated_at":93,"role_id":4,"staff_id":4,"suggestion":12,"regenerator_id":4,"sub_type":12,"stage":12,"deleted_at":93,"type":12,"scope":12,"country_name":12},"table":"sales_scripts","ts":1573981727241,"type":"INSERT"}
```

到这里我们就能确定，canal增量订阅已经成功了，消息被如实的写到kafka里了。如果需要将变化写入ES，那就增加一个消息者消费消息就行了。这里就不做介绍了。

## 小结

通过利用MySQL的binlog日志文件，不仅可以实现主从复制还有数据恢复，我们还可以根据业务需求做很多实用性的操作。并且通过结合canal和kafka，可以避免并发频繁的写入ES，造成压力。更安全的执行数据同步。






