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

#####  2.1 config目录下编辑logstash.yml文件

```shell
http.host: "0.0.0.0"

#配置文件都在目录 conf.d/*.conf中，意味着，所有.conf结尾的都是配置文件，都会执行

path.config: /usr/share/logstash/conf.d/*.conf

#安全配置

xpack.monitoring.elasticsearch.username: elastic
xpack.monitoring.elasticsearch.password: luoji_elk
```

##### 2.2 conf.d目录下新建文件mynginx.conf文件

```shell
#输入模块定义
input{
    beats {
	   port => 5044
    }
}
#过滤模块定义
filter{
   
  if [fields][document_type] == "nginx-access" {   
     mutate {
       remove_field => ["host","agent","ecs","tags","@version","@timestamp","input","log"]
       add_field => {"type" => "nginx-access"}
     }
     json {
       source => "message"
       #移除多余字段
       remove_field => "message"
     }
     #grok插件
     grok {
        #从时间中获取day
         match => { "@timestamp" => "(?<day>.{10})" }
     }
 
  }
  if [fields][document_type] == "nginx-error" {
      
     mutate {
       remove_field => ["host","agent","ecs","tags","@version","@timestamp","input","log"]
       add_field => {"type" => "nginx-error"}
     }
     grok {
       match => [ "message" , "(?<timestamp>%{YEAR}[./-]%{MONTHNUM}[./-]%{MONTHDAY}[- ]%{TIME}) \[%{LOGLEVEL:severity}\] %{POSINT:pid}#%{NUMBER}: %{GREEDYDATA:errormessage}(?:, client: (?<remote_addr>%{IP}|%{HOSTNAME}))(?:, server: %{IPORHOST:server}?)(?:, request: %{QS:request})?(?:, upstream: (?<upstream>\"%{URI}\"|%{QS}))?(?:, host: %{QS:request_host})?(?:, referrer:\"%{URI:referrer}\")?"]
       remove_field => ["message"]       
     }
     date {
        match => ["timestamp","yyyy/MM/dd HH:mm:ss","ISO8601"]
        target => "@timestamp"
     }
     #grok插件
     grok {
        #从时间中获取day
         match => { "@timestamp" => "(?<day>.{10})" }
     }

     mutate {
       remove_field => ["timestamp"]
     }


  }

}

# 输出块定义
output{
      if [fields][document_type] == "nginx-access" {
           elasticsearch {
               hosts => ["elasticsearch:9200"]
               user => elastic
               password => luoji_elk
               index => "nginx-access-%{day}"
               manage_template => true
               template_overwrite => true
               template_name => "nginx-access"
               template => "/usr/share/logstash/myfile/nginx-access.json"
            }
      }
 
      if [fields][document_type] == "nginx-error" {
          elasticsearch {
               hosts => ["elasticsearch:9200"]
               user => elastic
               password => luoji_elk
               index => "nginx-error-%{day}"
               manage_template => true
               template_overwrite => true
               template_name => "nginx-error"
               template => "/usr/share/logstash/myfile/nginx-error.json"
            }
      }

}
```

##### 2.3编辑索引模板文件 nginx-access.json和nginx-error.json

###### 2.3.1编辑nginx-access.json

```shell
{  
    "index_patterns": ["nginx-access-*"],
    "settings": {
        "index.number_of_shards": 5,
        "number_of_replicas": 0
     },
    "mappings" : {
        "properties" : {
          "@timestamp": {
            "type" : "date",
            "format" : "dateOptionalTime"
          },
         "@version": {
            "type": "integer"
          },
         "client": {
            "type": "keyword",
            "ignore_above": 256
          },
          "host": {
            "type": "keyword",
            "ignore_above": 256
          },
          "domain": {
            "type": "keyword",
            "ignore_above": 256
          },
         "port": {
            "type": "keyword"
          },
          "url": {
            "type": "keyword",
            "ignore_above": 256
          },
          "args": {
            "type": "text",
            "index": false
          },
          "request_method": {
            "type": "keyword"
          },
          "scheme": {
            "type": "keyword"
          },
          "status": {
            "type": "keyword"
          },
          "referer": {
            "type": "keyword",
            "ignore_above": 256
          },
          "size": {
            "type": "keyword"
          },
          "responsetime": {
            "type": "scaled_float",
            "scaling_factor": 1000
          },
          "ua": {
            "type": "text",
            "index": false
          },
          "type":{
            "type":"keyword"
          },
          "day":{
             "type":"date",
             "format":"yyyy-MM-dd"
          }
      }
   }  
}
```

###### 2.3.1编辑nginx-error.json

```shell
{  
    "index_patterns": ["nginx-error-*"],
    "settings": {
        "index.number_of_shards": 5,
        "number_of_replicas": 0
     },
    "mappings" : {
        "properties" : {
          "@timestamp": {
            "type" : "date",
            "format" : "dateOptionalTime"
          },
         "pid": {
            "type": "integer"
          },
          "severity": {
            "type": "keyword",
            "ignore_above": 256
          },
         "errormessage": {
            "type": "text",
            "index": false
          },
          "server": {
             "type": "keyword",
             "ignore_above": 256
          },
          "request": {
            "type": "keyword",
            "ignore_above": 256
          },
         "port": {
            "type": "integer"
          },
          "upstream": {
            "type": "keyword",
            "ignore_above": 256
          },
          "remote_addr": {
            "type": "keyword",
            "ignore_above": 256
          },
          "request_host": {
            "type": "keyword",
            "ignore_above": 256
          },
          "type":{
            "type":"keyword"
          },
          "day":{
             "type":"date",
             "format":"yyyy-MM-dd"
          }
      }
   }  
}

```

#### 	3.kibana配置

​	

#### 	4.filebeat配置

​	//todo zsj

#### 	5.elastalert配置  

​    elastalert是使用python写的一个报警框架，通过查询ElasticSearch中的记录然后进行比对，通过配置好的报警规则对日志发送报警邮件。

##### 5.1 config.json

​    针对elastalertserver的配置文件，如果kibana中没有安装elastalert插件，可以不适用该服务。

```

{
  "appName": "elastalert-server",
  "port": 3030,
  "wsport": 3333,
  "elastalertPath": "/opt/elastalert",
  "start": "2020-01-01T00:00:00",
  "end": "2020-06-01T00:00:00",
  "verbose": false,
  "es_debug": false,
  "debug": false,
  "rulesPath": {
    "relative": true,
    "path": "/rules"
  },
  "templatesPath": {
    "relative": true,
    "path": "/rule_templates"
  },
  "es_host": "elasticsearch",
  "es_port": 9200,
  "writeback_index": "elastalert_status"
}
```

##### 5.2 elastalert.yaml

​	该配置为elastalert主要配置。

```
# The elasticsearch hostname for metadata writeback
# Note that every rule can have its own elasticsearch host
es_host: elasticsearch

# The elasticsearch port
es_port: 9200

# This is the folder that contains the rule yaml files
# Any .yaml file will be loaded as a rule
rules_folder: rules

# How often ElastAlert will query elasticsearch
# The unit can be anything from weeks to seconds
run_every:
  seconds: 5000

# ElastAlert will buffer results from the most recent
# period of time, in case some log sources are not in real time
buffer_time:
  minutes: 1

# Optional URL prefix for elasticsearch
#es_url_prefix: elasticsearch

# Connect with TLS to elasticsearch
#use_ssl: True

# Verify TLS certificates
#verify_certs: True

# GET request with body is the default option for Elasticsearch.
# If it fails for some reason, you can pass 'GET', 'POST' or 'source'.
# See http://elasticsearch-py.readthedocs.io/en/master/connection.html?highlight=send_get_body_as#transport
# for details
#es_send_get_body_as: GET

# Option basic-auth username and password for elasticsearch
es_username: elastic
es_password: luoji_elk

# The index on es_host which is used for metadata storage
# This can be a unmapped index, but it is recommended that you run
# elastalert-create-index to set a mapping
writeback_index: elastalert_status

# If an alert fails for some reason, ElastAlert will retry
# sending the alert until this time period has elapsed
alert_time_limit:
  days: 2

```

##### 5.3 报警规则配置

​	报警规则配置都在rules文件夹下，此处以frequency的报警为例。

```
# (OptionaL) Connect with SSL to Elasticsearch
#use_ssl: True

# (Optional) basic-auth username and password for Elasticsearch
es_username: elastic
es_password: luoji_elk

# (Required)
# Rule name, must be unique
name: Example frequency rule

# (Required)
# Type of alert.
# the frequency rule type alerts when num_events events occur with timeframe time
type: frequency

# (Required)
# Index to search, wildcard supported
index: nginx*

# (Required, frequency specific)
# Alert when this many documents matching the query occur within a timeframe
num_events: 50

# (Required, frequency specific)
# num_events must occur within this amount of time to trigger an alert
timeframe:
  seconds: 20

# (Required)
# A list of Elasticsearch filters used for find events
# These filters are joined with AND and nested in a filtered query
# For more info: http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl.html
filter:
- term:
    status: "200"

#邮箱设置
smtp_host: smtp.qq.com
smtp_port: 465
smtp_ssl: true
from_addr: "45@qq.com"
smtp_auth_file: /opt/elastalert/smtp_auth_file.yaml
# (Required)
# The alert is use when a match is found
alert:
- "email"

# (required, email specific)
# a list of email addresses to send alerts to
email:
- "n@dingtalk.com
```

以上配置中达到报警规则时会以邮件的形式通知收件人，同时需要配置一个发送放的用户名和授权码。

```
user: 45@qq.com
password: ********#此处为授权码，不是密码
```



## 3.使用说明

​	此处主要介绍kibana界面的操作说明，包含主要的功能模块：日志发现，可视化，仪表盘，索引管理，用户管理，报警规则，其他高级功能逐步完善。

### 3.1 Discover

The first thing to do in **Discover** is to select an [index pattern](https://www.elastic.co/guide/en/kibana/7.x/index-patterns.html), which defines the data you want to explore and visualize. If you haven’t yet created an index pattern, you can add a [sample data set](https://www.elastic.co/guide/en/kibana/7.x/add-sample-data.html), which has a pre-built index pattern.

先建一个搜索模式

![](D:\elk\doc_elk\images\indexpattern01.png)

![](D:\elk\doc_elk\images\indexpattern02.jpg)

![](D:\elk\doc_elk\images\indexpattern03.png)

创建好搜索模式之后打开Discover标签页 如下图

1 选择搜索模式

2 选择关注的字段

3 设置时间范围，可以设置页面多长时间刷新一次

4 日志数据

5 filters 可以设置过滤条件比如 status : 200 可以输入也可以点击+Add filter 按钮添加索引条件

![image-20200510225538493](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200510225538493.png)



### 3.2 Visualize

*Visualize* enables you to create visualizations of the data from your Elasticsearch indices, which you can then add to dashboards for analysis.

Kibana visualizations are based on Elasticsearch queries. By using a series of Elasticsearch [aggregations](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/search-aggregations.html) to extract and process your data, you can create charts that show you the trends, spikes, and dips you need to know about.

To begin, open **Visualize**, then click **Create new visualization**.

Visualize使您能够从Elasticsearch索引创建数据的可视化，然后可以将其添加到仪表板进行分析。

Kibana可视化基于Elasticsearch查询。通过使用一系列Elasticsearch聚合来提取和处理数据，您可以创建图表，向您显示需要了解的趋势、峰值和下探。

首先，打开“可视化”，然后单击“创建新可视化”。

**[Most frequently used visualizations](https://www.elastic.co/guide/en/kibana/7.x/most-frequent.html)**

- **Line, area, and bar charts** — Compares different series in X/Y charts.
- **Pie chart** — Displays each source contribution to a total.
- **Data table** — Flattens aggregations into table format.
- **Metric** — Displays a single number.
- **Goal and gauge** — Displays a number with progress indicators.
- **Tag cloud** — Displays words in a cloud, where the size of the word corresponds to its importance.

-**折线图、面积图和条形图**-比较X/Y图表中的不同系列。

-**饼图**-显示每个源对总计的贡献。

-**数据表**-将聚合扁平化为表格式。

-**公制**-显示单个数字。

-**目标和量表**-显示带有进度指标的数字。

-**Tag cloud**-在云中显示单词，其中单词的大小与其重要性相对应。

##### 3.2.1Line

选择新建可视化选择类型line,再选择资源

![](D:\elk\doc_elk\images\line01.png)

![](D:\elk\doc_elk\images\v02.png)

配置

1 配置纵坐标总量

2  3 配置横坐标

4 播放 展示效果

5 保存并且命名

![](D:\elk\doc_elk\images\line03.png)

##### 3.2.2area

类似Line操作 选择新建可视化选择类型area,再选择资源

设置纵坐标 横坐标  点击播放查看效果

![](D:\elk\doc_elk\images\area01.png)

可以添加sub-buckets对数据近一步细分 点击播放查看效果

![](D:\elk\doc_elk\images\area02.png)

![](D:\elk\doc_elk\images\area03.png)

可以做一下图行配置如更改图像类型，点击播放查看效果

![](D:\elk\doc_elk\images\area04.png)

##### 3.2.3 bar chart

可以建立垂直方向条形图和水平方向条形图

配置展示各个客户端的访问量垂直条形图以及以15分钟为递增的显示产生的数据量，点击播放查看效果

![](D:\elk\doc_elk\images\bar01.png)

![](D:\elk\doc_elk\images\bar02.png)

##### 3.2.4 pie chart

类似Line操作 选择新建可视化选择类型pie,再选择资源

配置展示根据响应时间区间显示数据的组成部分（聚合类型为范围，字段选择响应时间，配置响应时间区间 ），点击查看播放效果

![](D:\elk\doc_elk\images\pie01.png)

![](D:\elk\doc_elk\images\pie02.png)

##### 3.2.5 metric

类似Line操作 选择新建可视化选择类型metric,再选择资源

配置展示数据量，点击查看播放效果

![](D:\elk\doc_elk\images\metric01.png)

![](D:\elk\doc_elk\images\metric02.png)

##### 3.2.6 Goal and gauge

类似Line操作 选择新建可视化选择类型goal,再选择资源

配置展示nginx错误日志是error级别的日志量 ，点击查看播放效果

![](D:\elk\doc_elk\images\goal01.png)

##### 3.2.7 Tag cloud

类似Line操作 选择新建可视化选择类型 Tag cloud,再选择资源

配置展示根据nginx访问日志的响应状态显示不同大小颜色的tag，点击查看播放效果

![](D:\elk\doc_elk\images\tag01.png)

##### 3.2.8 Timelion

Timelion is a time series data visualizer that enables you to combine totally independent data sources within a single visualization. It’s driven by a simple expression language you use to retrieve time series data, perform calculations to tease out the answers to complex questions, and visualize the results.

Timelion是一个时间序列数据可视化工具，使您能够在单个可视化中组合完全独立的数据源。它是由一种简单的表达式语言驱动的，您可以使用它来检索时间序列数据，执行计算来梳理复杂问题的答案，并可视化结果。

note:表达式输入时会有提示

![](D:\elk\doc_elk\images\timelion01.png)

##### 3.2.9 其他

添加保存好可视化之后就可以在Visualize下看到可视化列表了

![](D:\elk\doc_elk\images\vs01.png)

### 3.3 Dashboard

A *dashboard* is a collection of visualizations, searches, and maps, typically in real-time. Dashboards provide at-a-glance insights into your data and enable you to drill down into details.

仪表板是可视化、搜索和地图的集合，通常是实时的。仪表板提供对数据的一目了然的洞察力，并使您能够深入了解详细信息。

1. 新建仪表盘
2. 添加内容
3. 保存仪表盘
4. 输入名称
5. 可以在仪表盘列表查看新添加的仪表盘

![](D:\elk\doc_elk\images\dh01.png)

![](D:\elk\doc_elk\images\dh02.png)

![](D:\elk\doc_elk\images\dh03.png)

![](D:\elk\doc_elk\images\dh04.png)



![](D:\elk\doc_elk\images\dh05.png)

![](D:\elk\doc_elk\images\dh06.png)





### 3.4 Elastalert
##### 3.4.1 插件安装

- 下载[elastalert](https://github.com/bitsensor/elastalert-kibana-plugin)插件，要求和kibana版本要一致，也可以选择相近的版本。

- 拷贝插件到容器内

  `docker cp plugin.zip image:/usr/share/kibana/bin`

- 进入kibana容器内部

  ```
  #修改package.json 使kibana的版本和插件版本一致
  cd /usr/share/kibana/bin
  ./kibana-plugin install file:///usr/share/kibana/bin/plugin.zip
  #安装完成后，修改plugins目录下package.json下插件的版本号和kibana版本一致
  ```

- 配置config/kibana.yml

  插件需要配合elastalert服务端使用，默认连接localhost:3030，如果需要修改host和port，在config/kibana.yml增加以下配置。

  ```
  elastalert-kibana-plugin.serverHost: 123.0.0.1
  elastalert-kibana-plugin.serverPort: 9000
  ```

- 重启kibana。

- 重新提交一份新的kibana镜像。

  `docker commit -a "author" -m “add plugin" containerID new_image_id`

##### 3.4.2 插件使用

* 安装完成后在kibana中可以可能到elastalert的图标，如下图所示。页面中会自动加载在elastalert中配置好的报警规则。

![image-20200511105456987](C:\Users\luoji\AppData\Roaming\Typora\typora-user-images\image-20200511105456987.png)

* 新建规则

  ![image-20200511105751981](C:\Users\luoji\AppData\Roaming\Typora\typora-user-images\image-20200511105751981.png)

### 3.5 Stack Monitoring

elk服务监控，可以对es，kibana，logstash等服务的磁盘占用，内存使用，索引详情等进行实时监控

![image-20200511110004009](C:\Users\luoji\AppData\Roaming\Typora\typora-user-images\image-20200511110004009.png)

### 3.6 Management

//todo zsj



## 4.出错处理设计

​	整理在测试阶段出现的问题和遇到的困难。

### 4.1 kibana请求报错

### 4.2 logstash过滤问题

### 4.3 elastalert配置问题

elastalert使用最新版的docker时会一直报错，显示无法连接到elasticsearch，原因是最新版的docker不支持es 7.0以上的版本，需要重新下载3.0.0-beta.0版本的镜像，同时会开启elastalert server服务。



### 4.4 待完善



## 5. 设计总结

##### 5.1 技术文档参考

​    在设计过程中很多软件的安装和配置需要参考网上的资源，如何高效的查找有用的文档，同时兼容其他模块的配置变得有些难度，网络的资源很多都已经过时，并且内容较多不容易快速找到目标答案。这样就会浪费很多的时间。经过此次项目的实践，对资料的查找方法总结如下：

* 查阅官网文档，快速找到目标解决方案。
* 浏览github上的issues，能更快的找到出错的原因。
* 使用google
* 做好笔记

##### 5.2 错误解决流程

​    elk设计到的框架比较多，配置文件随着也会增多，调试问题占用了大量的时间，如何快速的解决开发中出现的问题，对工作效率会有很大的影响。

* 看日志，仔细分析日志
* 查找网上有效的资源，参考5.1
* 尽量避免反复的尝试
* ![]()寻找同事的帮助