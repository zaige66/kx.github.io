---
title: mysql半自动备份
date: 2021-02-03 15:57:28 
layout:  mysql半自动备份
tags: [java,mysql,linux]
categories: [java,mysql]
---
@[toc]
# 需要的信息
```java
		// 旧数据库及服务器信息
        String oldLinuxHost = "192.168.30.10";
        int oldLinuxPort = 22;
        String oldLinuxUserName = "root";
        String oldLinuxPwd  = "aaa";
        // binlog日志保存位置
        String binLogPath = "/home/mysql/bin_log";
        // 全量dump文件保存位置
        String dumpPath = "/tmp/";

        // 需要备份的数据库名
        String backupDbName = "wbwf";
        // 新数据库名
        String newDbName = "bbb";
        // 数据截止时间
        String endTime = "2021-01-13 14:48:00";

        // 新数据库及服务器信息
        String newLinuxHost = "192.168.30.10";
        int newLinuxPort = 22;
        String newLinuxUserName = "root";
        String newLinuxPwd  = "aaa";
        String newMysqlUserName = "root";
        String newMysqlPwd = "ccc";
        // 数据库是否安装在docker中
        boolean newMysqlIsDocker = true;
```
# 处理步骤
- 在旧服务器上寻找合适条件dump文件
- 在旧服务器上虚招合适条件的binlog日志，并转换成可导入的.sql文件
- 将准备好的dump文件与.sql文件传输到新的服务器上
- 在新的服务器上使用 source 指令将dump文件与.sql文件导入到数据库
# 代码示例
```java
BackupService backupService = new BackupService();
        // 旧数据库及服务器信息
        String oldLinuxHost = "192.168.30.10";
        int oldLinuxPort = 22;
        String oldLinuxUserName = "root";
        String oldLinuxPwd  = "aaa";
        // binlog日志保存位置
        String binLogPath = "/home/mysql/bin_log";
        // 全量dump文件保存位置
        String dumpPath = "/tmp/";

        // 需要备份的数据库名
        String backupDbName = "wbwf";
        // 新数据库名
        String newDbName = "bbb";
        // 数据截止时间
        String endTime = "2021-01-13 14:48:00";

        // 新数据库及服务器信息
        String newLinuxHost = "192.168.30.10";
        int newLinuxPort = 22;
        String newLinuxUserName = "root";
        String newLinuxPwd  = "aaa";
        String newMysqlUserName = "root";
        String newMysqlPwd = "ccc";
        // 数据库是否安装在docker中
        boolean newMysqlIsDocker = true;

        SshConfig oldSshConfig = new SshConfig(oldLinuxHost, oldLinuxPort, oldLinuxUserName, oldLinuxPwd);
        SshConfig newSshConfig = new SshConfig(newLinuxHost, newLinuxPort, newLinuxUserName, newLinuxPwd);


        // 处理旧服务器上文件
        // 1.寻找合适条件的dump文件
        String dumpFileName = backupService.handleDumpFile(oldSshConfig, endTime, backupDbName);

        // 2.将dump文件与之对应的合适的binlog日志输出为可执行sql文件
        Map<String, String> stringStringMap = backupService.handleBinlogFile(oldSshConfig, dumpPath, dumpFileName, binLogPath, backupDbName, endTime);

        // 将处理好的文件传输到需要备份的机器上
         stringStringMap = backupService.handleFileTransport(oldSshConfig, newSshConfig, stringStringMap.get("binlog"), stringStringMap.get("dump"));

        // 登录mysql，调用source指令，将dump文件与binlog文件导入数据库
        backupService.handleFileImport(newSshConfig,newMysqlUserName,newMysqlPwd,backupDbName,newDbName,stringStringMap.get("dump"),stringStringMap.get("binlog"),newMysqlIsDocker);
```
# 配置说明
```
    该项目用于对mysql数据库进行备份
    1.支持结束时间的选择
    2.支持外网库同步到内网中
    3.支持同步到同一个服务器不同数据库名
    
    需要准备项
    1.mysql数据库需要启动binlog日志功能
    2.编写定时任务，对目标数据库进行定时全量dump
```
## mysql开启binlog日志功能
> 在数据库配置文件my.cnf [mysqld] 配置项下加入如下内容
```
# binlog 设置
binlog_format = MIXED # binlog记录模式
log_bin = /home/mysql/bin_log/mysql-bin.log # binlog输出位置
expire_logs_days = 7 #binlog过期清理时间
server-id=123454 
max_binlog_size=1000m # binlog文件大小限制

```

## 编写dump脚本，并添加到定时任务中
- 新建脚本文件
> vi /home/mysql/mysql_dump/dump.sh
```shell
#/bin/sh
stamp="`date +%Y%m%d%H%M%s`"
# wbwf即指定导出的数据库名
mysqldump -u用户名 -p密码 -F  --master-data  wbwf > /home/mysql/mysql_dump/wbwf_${stamp}.sql
```
- 添加到定时任务
> crontab -e

0 0 */7 * * sh /home/mysql/mysql_dump/dump.sh

```
这里定时任务的执行周期可以与binlog清理日期相结合
```
# 项目地址
> https://github.com/zaige66/mysql_back
