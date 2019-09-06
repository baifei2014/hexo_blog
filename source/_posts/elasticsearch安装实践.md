---
layout: blog
title: Elasticsearch安装实践
tags: [Elasticsearch, java]
date: 2019-09-05 12:43:47
---

## 前言

Elasticsearch非常厉害，需要搜索技术的互联网公司大半都在使用它。尤其是对于一些中小型公司来说，没有足够的技术实力并且也没有必要开发自己的搜索引擎，选择Elasticsearch开源搜索引擎是一种非常不错的选择。对于后端工程师来说，掌握Elasticsearch已经成了技术栈不可或缺的一部分了。下面我们就来讨论下它的安装使用。

## 安装准备

ES是一款基于java的应用，要想安装启动必须要有java环境，所以我们首先要安装java的jdk组件，首先可以从官网下载，这里我下载解压到`/usr/local/jdk`目录，然后配置环境变量：

```bash
export JAVA_HOME=/usr/local/jdk
export PATH=${JAVA_HOME}/bin:$PATH
```

这样我们就可以成功运行java命令了，到这里，java开发环境已经准备好了。

## 安装ES

官网介绍了多平台多种不同的安装方式，这里介绍下通过rpm安装ES。

### 安装公钥
首先，先下载安装一个公钥，用来保证后面安装包的准确性：

```bash
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```

### 设置ES安装源

在`/etc/yum.repos.d/`创建文件名为`elasticsearch.repo`的文件，然后复制以下内容到文件内：

```bash
[elasticsearch-7.x]
name=Elasticsearch repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
```

这里我们已经准备好了所有安装要准备的事，下面就正式执行安装了，执行如下命令就可以进行安装：

```bash
sudo yum install elasticsearch
```

## 启动ES服务

ES安装完，就可以执行启动命令将ES服务运行起来，默认端口是9200，所以要保留有这个端口便于ES使用，在启动时，可能会因为机器内存问题导致ES服务挂掉，ES默认的初始堆内存和最大堆内存都是1g，所以如果服务器内存小于1g，ES就会因内存不足而被kill，如果在启动时发现ES因内存挂掉，那我们可以通过修改配置文件解决，因为我使用的机器内存小于1g，内存不足，所以我修改配置，将ES最大堆内存设置为256m，设置如下：

```bash
-Xms256m
-Xmx256m
```

当应用启动完后，可以访问9200端口查看是否已正常运行，直接在命令行执行`curl http://localhost:9200`，返回信息如下：

```json
{
  "name" : "jiang-2018",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "5R-pIrfASG-hiBgpcvvUkg",
  "version" : {
    "number" : "7.3.0",
    "build_flavor" : "default",
    "build_type" : "rpm",
    "build_hash" : "de777fa",
    "build_date" : "2019-07-24T18:30:11.767338Z",
    "build_snapshot" : false,
    "lucene_version" : "8.1.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

看到这里，ES就安装完了。

## 参考资料
- [ES官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/rpm.html#rpm-repo)



