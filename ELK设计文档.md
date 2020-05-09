# ELK设计文档

[TOC]

## 1.引言

​	  **ELK** 并不是一款软件，而是一整套解决方案，是三个软件产品的首字母缩写，Elasticsearch-->存储数据，Logstash-->搜集数据，  Kibana-->展示数据。

### 1.1 编写目的

  	该设计文档为运维和开发人员提供基础的参考方案，整理在设计过程中的思路和遇到的困难，方便后续维护和优化。

### 1.2 项目风险

​		本次搭建的elk框架还有很多潜在的问题和很大的优化空间，涉及到的技术比较多，如docker，jvm，python，es集群，logstash过滤等。在使用中会逐步完善。

### 1.3 参考资料

​		elastic官网https://www.elastic.co/

​		kibana用户手册https://www.elastic.co/guide/cn/kibana/current/index.html

​		es权威指南https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html

​		logstash参考文档https://www.elastic.co/guide/en/logstash/current/index.html

​		elastalert docker网址https://github.com/bitsensor/elastalert

​		elasalert使用手册https://elastalert.readthedocs.io/en/latest/index.html



## 2.设计概述

​		本项目需要满足对例如ngninx等日志的统计和分析工作，同时能够在此基础上实现一些功能的扩展，比如分析用户行为，监控应用状态，异常报警等。

### 2.1 基础环境搭建

![timg](C:\Users\luoji\Desktop\timg.jpg)

​		如上图所示，此处增加了filebeat，由于logstash占用资源比较严重，为了不影响生产环境下系统的正常运行，采用轻量级的filebeat对日志进行采集。

​        网络通信采用docker-compose管理。

### 2.2 软件安装

* 本项目统一采用docker进行安装和管理。
* elkb中所有软件需要版本号完全一致，考虑到需要增加用户和角色管理，统一使用**7.2.1**版本。

* 使用docker-compose对docker镜像统一管理。
* 使用docker安装elastalert，实现报警功能。
* 具体安装过程参考confluence[文档](http://192.168.1.101:8090/pages/viewrecentblogposts.action?key=ELK)

### 2.3 详细配置

#### 	1.elasticsearch配置

​	//todo zsj

#### 	2.logstash配置

​	//todo yy

#### 	3.kibana配置

​	

#### 	4.filebeat配置

​	//todo zsj

#### 	5.elastalert配置  



## 3.使用说明

​	此处主要介绍kibana界面的操作说明，包含主要的功能模块：日志发现，可视化，仪表盘，索引管理，用户管理，报警规则，其他高级功能逐步完善。

### 3.1 Discover

//todo  编辑

### 3.2 Visualize

//todo yy

### 3.3 Dashboard

//todo yy

### 3.4 Elastalert
//todo lihu

### 3.5 Stack Monitoring

### 3.6 Management

//todo zsj



## 4.出错处理设计

​	整理在测试阶段出现的问题和遇到的困难。

### 4.1 kibana请求报错

### 4.2 logstash过滤问题

### 4.3 elastalert配置问题

### 4.4 待完善



## 5. 设计总结

