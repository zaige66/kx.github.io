---
title: ELK搭建流程 从0到1 包含过程中遇到的问题
date: 2018-11-15 16:17:29 
layout: ELK搭建流程 从0到1 包含过程中遇到的问题
tags: [ELK,kibana,logstash]
categories: [linux,elk]
---
@[toc]
闲言碎语不要讲，开始！
# ELK环境搭建
>参考链接：https://blog.csdn.net/liuge36/article/details/79768152
## 需要注意的地方

 1. JDK1.8以上
 2. elasticsearch 启动时不能以 root 用户启动
# 问题汇总
## elasticsearch
### 1.启动后外网访问不了
> 参考地址：https://blog.csdn.net/buzaiQQ/article/details/67637731
 1. 防火墙是否关闭
 2. 是否开启了代理ip（我就是为了 谷歌搜索 然后开启了 代理ip，通过 内网ip始终访问不到 elasticsearch，一度怀疑人生）
 3. 在 elasticsearch.yml 文件中添加如下配置
 ```yml
 network.host: 0.0.0.0

bootstrap.system_call_filter: false
http.cors.enabled: true
http.cors.allow-origin: "*"
 ```
 ### 2.max number of threads [1024] for user [xxx] is too low, increase to at least [4096
 > 意思是当前用户线程数过低
 > 参考链接：https://elasticsearch.cn/question/3915（重启后生效）
 ## kibana
 ### 1.汉化
 > 参考链接：https://github.com/anbai-inc/Kibana_Hanization
 ### 2.visualize(可视化)条件筛选时没有自己想要的项
  主要是因为我在 elasticsearch 中建立数据类型(type)时如下
 ```json
 {
  "properties": {
    "time": {
      "type": "date",
      "format": "yyyy-MM-dd HH:mm:ss.SSS"
    },
    "ip": {
      "type": "text",
    },
    "request": {
      "type": "text",
    },
    "param": {
      "type": "text"
    }
  }
}
 ```
 后来修成成
 ```json
 {
  "properties": {
    "time": {
      "type": "date",
      "format": "yyyy-MM-dd HH:mm:ss.SSS"
    },
    "ip": {
      "type": "keyword",
      "ignore_above": 256
    },
    "request": {
      "type": "keyword",
      "ignore_above": 256
    },
    "param": {
      "type": "keyword"
    }
  }
}
 ```
 猜测原因是因为 text 不会没有建立索引规则，keyword会生成索引规则
 ### 3.discover(发现) 中查询不到数据
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20181115152602767.png)
 这里有个时间范围，我刚开始没找到是这个原因，一直查询不到数据，可急死我了。
 ### 4.记录的时间与elasticsearch中相差8小时
 > https://blog.csdn.net/weixin_42207486/article/details/82694495
 ## logstash
 > 这个组件问题最多，花了我最多的时间，一度想替换成其他日志采集的工具，但是好像只有这货 能很好的把日志解析成与 elasticsearch 中定义的字段 相对应
 
 ### 1.启动很慢慢慢！！！
 > 不吐不快！ 我等他启动了快5分钟，然后报错说配置文件有问题，启动失败，当时真是...

 去百度，基本都是说 熵值低了，要安装 haveged 
 地址：https://www.jianshu.com/p/8ffc521fc3ed
 但是在我这里没一点用。
 
 后来 谷歌搜索半天，找到一个配置更新热加载的说明
 地址：https://segmentfault.com/a/1190000015715238
 bin/logstash -f test.conf -r
 这个还比较靠谱，我修改的基本也就是配置文件，通过这个热加载配置文件，基本只需要启动一次就好了
 ### 2.grok 的编写
 > 参考链接：https://www.jianshu.com/p/d46b911fb83e
 > grokdebugger：http://grokdebug.herokuapp.com/
 
 大概样子
 ```properties
 grok {
    patterns_dir => ["/opt/elk/logstash-6.4.3/patterns"]
    match => {"message" => "%{DATA:time}\|%{DATA:ip}\|%{DATA:request}\|%{JSON_PARAM:param}"}
  }
 }
 ```
这里就再说说我遇到的坑吧...
我的日志格式是这样的 **时间|ip|访问路径|访问参数**
最开始的 match 配置为 match => {"message" => "%{DATA:time}\\|%{DATA:ip}\\|%{DATA:request}\\|%{DATA:param}"}
结果就是 最后匹配的 param(访问参数) 一直匹配不上
这里真是卡了好久，去百度都不知道怎搜... 在我坚持不懈的 百度+谷歌 下，终于知道原因是：DATA 模板匹配方式是 （.\*?） 这个正则表达式就是匹配不上，后来我就改了下匹配方式，改成 （.\*）就可以匹配上了.
这里就又涉及到另一个点：**如何自定义模板**
### 3.自定义模板
> 找不到相应的参考链接了，大家去官网看吧

新建模板文件

![在这里插入图片描述](https://img-blog.csdnimg.cn/20181115160130341.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3phaWdlNjY=,size_16,color_FFFFFF,t_70)
在logstash安装目录中新建文件夹 **patterns**(随意起名)，在 patterns 中新建文件**JOSN_PATTERN**(随意名)，文件里面编写自定义的模式 **JSON_PARAM (.*)** 注意：名称与 表达式中间有个空格
调用方式：
```
grok {
   patterns_dir => ["/opt/elk/logstash-6.4.3/patterns"]
   match => {"message" => "%{DATA:time}\|%{DATA:ip}\|%{DATA:request}\|%{JSON_PARAM:param}"}
 }
}
```
### 3.监控多个文件并发送到elasticsearch的多个索引中
> 这个容易搜到结果，关键点就是给file中定义一个type属性，其他位置对type 做判断，我直接贴配置文件
```json
input {
       beats{
       		port => 5044
       }
       file{
       		type => "access"
       		path => ["/opt/test/access.log"]
       }
       file{
       		type => "execute"
       		path => ["/opt/test/execute.log"]
       }

}
filter {
if [type] == "access"{
 grok {
    patterns_dir => ["/opt/elk/logstash-6.4.3/patterns"]
    match => {"message" => "%{DATA:time}\|%{DATA:ip}\|%{DATA:request}\|%{JSON_PARAM:param}"}
  }
 }
  else if [type] == "execute"{
    grok {
      patterns_dir => ["/opt/elk/logstash-6.4.3/patterns"]
      match => {"message" => "%{DATA:time}\|%{DATA:method}\|%{NUMBER:executeTime}\|%{DATA:projectName}\|%{NUMBER:port}"}
    }
 }
	
}
output {
 if [type] == "access"{
 	elasticsearch{
    	 hosts => ["192.168.33.33:9200"]
    	 index => "access"
    	 document_type => "access"
     }
 }
  else if [type] == "execute"{
 	elasticsearch{
    	 hosts => ["192.168.33.33:9200"]
    	 index => "execute"
    	 document_type => "execute"
     }
 }
     stdout{
              codec=>rubydebug
      }
}
```
# elasticsearch api操作
> 这个容易搜索，我做个记录
```txt
添加表字段
PUT my_index/_mapping/my_type
// access
{
  "properties": {
    "time": {
      "type": "date",
      "format": "yyyy-MM-dd HH:mm:ss.SSS"
    },
    "ip": {
      "type": "keyword",
      "ignore_above": 256
    },
    "request": {
      "type": "keyword",
      "ignore_above": 256
    },
    "param": {
      "type": "keyword"
    }
  }
}
// execute
{
  "properties": {
    "time": {
      "type": "date",
      "format": "yyyy-MM-dd HH:mm:ss.SSS"
    },
    "method": {
      "type": "keyword",
      "ignore_above": 256
    },
    "executeTime": {
      "type": "integer"
    },
    "projectName": {
      "type": "keyword",
      "ignore_above": 256
    },
    "port": {
      "type": "integer"
    }
  }
}


添加记录--指定id

$ curl -X PUT 'localhost:9200/index/type/id' -d '
{
  "user": "张三",
  "title": "工程师",
  "desc": "数据库管理"
}' 

添加记录--不指定id

$ curl -X POST 'localhost:9200/index/type' -d '
{
  "user": "李四",
  "title": "工程师",
  "desc": "系统管理"
}'

根据id查询
向/Index/Type/Id发出 GET 请求，就可以查看这条记录

使用 GET 方法，直接请求/Index/Type/_search，就会返回所有记录。

删除记录
/Index/Type/Id发出 delete 请求，就可以删除这条记录

更新记录就是使用 PUT 请求，重新发送一次数据。
```

# 写点什么
通过这个搭建，终于体会到百度与谷歌的区别，百度真是个**，还是谷歌靠谱。
整个搭建过程花了两天多时间，痛苦...
希望这篇文章能让大家更容易的搭建起来。加油！

