---
title: filebeat +kafka + logstash收集日志信息
date: 2018-12-19 19:29:42
tags: [filebeat,kafka,logstash]
categories: [linux,elk]
---
@[toc]
# 选择原因
```
logstash 笨重，对环境需要 jdk 1.8+，不适合部署在多个服务上
filebeat 是一个轻量级的日志监控工具，部署简单 无特殊环境要求
最终效果应该是：
	在需要收集日志的服务器上部署 filebeat，然后发送到 kafka （可以进行集群）消息中间件，
然后在logstash 中监听 kafka 的消息即可
```
# kafka 部署
kafka 需要 zookeeper 环境，所以先安装 zookeeper

`============================`zookeeper 安装 start`==============================`
1. zookeeper 下载地址：wget http://mirrors.hust.edu.cn/apache/zookeeper/zookeeper-3.4.10/zookeeper-3.4.10.tar.gz
2. tar -zvxf  解压
3. 复制配置文件(配置文件中配置的端口，默认2181)：
	cd conf
	cp zoo_sample.cfg zoo.cfg
4. 启动zookeeper： ./bin/zkServer.sh start

`============================`zookeeper 安装 end`==============================`
 `============================`kafka 安装start`==============================`
5. 下载地址：http://mirrors.cnnic.cn/apache/kafka/
6. 解压：tar -zvxf 
7.  如果zookeeper 每个改端口，且安装在 同一台机器，那么直接启动 kafka
：bin/kafka-server-start.sh config/server.properties ，如果不是，修改 server.properties 文件
8. 创建主题：bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
9. 启动生产者：bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
10. 启动消费者：bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
可以测试在生产者中输入数据，在消费者端就会有显示
**注意事项：**
		1.启动kafka之前需要启动 zookeeper
		2.如果启动kafka时报错：java.net.UnknownHostException: XX: XX: 未知的名称或服务
		解决办法：
			vi /etc/hosts
			加上
			127.0.0.1	XX
 `============================` kafka 安装 end`==============================`

# filebeat部署
1. 下载https://www.elastic.co/downloads/beats/filebeat#ga-release
2. 解压：tar -zvxf filebeat-6.5.3-linux-x86_64.tar.gz
3. 修改配置文件：
	vi filebeat.yml
	1. 配置监听的文件:
		paths:
			    - /opt/test/test.log
	2. 配置输出项：
		注释掉默认的elasticsearch 的输出项
		添加kafka的配置
```yml
		output.kafka:
		  enabled: true
		  hosts: ["127.0.0.1:9092"] # 生产者的地址
		  topic: test  # 上面创建的kafka topic 名称
```
4. 启动filebeat：./filebeat -e -c filebeat.yml    说明：-e 表示 Log to stderr and disable syslog/file output
5. 测试：
	1.把上面kafka第10步的消费者起起来
	2.echo "test info" >> /opt/test/test.log
	3.观察 kafka 消费端 与 filebeat 端是否有对应的内容输出
	**注意：** filebeat 收集到日志后 会把消息封装成一个json，传给 kafka的也是这个json
	
# logstash 部署
> 该组件的安装详见 ELK 那篇文章 [ELK搭建流程 从0到1 包含过程中遇到的问题](https://blog.csdn.net/zaige66/article/details/84104975)

logstah的配置文件修改
```conf
input {
    kafka{
        bootstrap_servers => ["127.0.0.1:9092"]  # 这里填生产者的地址
        client_id => "test"
        group_id => "test"
        auto_offset_reset => "latest" 
        consumer_threads => 5
        decorate_events => true 
        topics => ["test"]  # 这里填 topic 名
        type => "bhy" 
      }
}
```
**问题**：上面提到过，发送到kafka的数据是用json包装过信息的，包装后的样子
{"@timestamp":"2018-12-19T19:08:15.483Z","@metadata":{"beat":"filebeat","type":"doc","version"ector":{"type":"log"},"input":{"type":"log"},"beat":{"hostname":"kk","version":"6.5.3","name":tform":"centos","version":"6.10 (Final)"},"containerized":true}}
{"@timestamp":"2018-12-19T19:21:45.720Z","@metadata":{"beat":"filebeat","type":"doc","version":"6.5.3","topic":"test"},"source":"/opt/test/test.log","offset":323,"message":"ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff","input":{"type":"log"},"prospector":{"type":"log"},"beat":{"version":"6.5.3","name":"kk","hostname":"kk"},"host":{"name":"kk","architecture":"x86_64","os":{"family":"redhat","codename":"Final","platform":"centos","version":"6.10 (Final)"},"containerized":true}}
里面的 ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff 才是我需要的信息，所以在logstash的 filter 中需要进行配置，对数据进行处理
`====================================问题解决==================================`
logstash最终配置文件：
```conf
input {
    kafka{
        bootstrap_servers => ["127.0.0.1:9092"]
        client_id => "test"
        group_id => "test"
        auto_offset_reset => "latest" 
        consumer_threads => 5
        decorate_events => true 
        topics => ["test"] 
        type => "bhy" 
        codec => json # 这里配置进行json解析
      }

    file{
       		type => "execute"
       		path => ["/opt/test/execute.log"]
       }
}
filter {
	
if [type] == "bhy"{
	# 这里对message字段（就是收集到的日志信息）进行匹配解析
	 grok {
	    patterns_dir => ["/opt/elk/logstash-6.4.3/patterns"]
	    match => {"message" => "%{DATA:time}\|%{DATA:ip}\|%{DATA:request}\|%{JSON_PARAM:param}"}
	  }

	# 这里对解析后的结果进行字段过滤，移除多余的字段
	mutate {
		remove_field =>["prospector"]
		remove_field =>["host"]
		remove_field =>["message"]
	}
  	
}

}
output {    
     stdout{
              codec=>rubydebug
      }
}
```
**贴上最终结果**：
输入日志信息：
	echo "time|ip|requet|param" >> /opt/test/test.log
logstash解析后的数据：
```json
{
         "input" => {
        "type" => "log"
    },
    "@timestamp" => 2018-12-20T16:18:18.590Z,
      "@version" => "1",
         "param" => "param",
          "type" => "bhy",
       "request" => "requet",
          "time" => "time",
            "ip" => "ip",
          "beat" => {
        "hostname" => "kk",
         "version" => "6.5.3",
            "name" => "kk"
    },
        "offset" => 655,
        "source" => "/opt/test/test.log"
}
```
这样就能发送到elasticsearch了！

附上logstash配置参考链接：
> 移除字段:https://blog.csdn.net/zhaoyangjian724/article/details/54343178
> 抽取json :https://blog.csdn.net/zmx729618/article/details/80885179

`==========================================================`
结束！希望对大家有所帮助！
