---
title: mysql数据备份与恢复
date: 2020-04-14 10:45:39
tags: [mysql]
categories: [linux,mysql]
---
# 增量备份
> 编辑数据库配置问件 my.cnf，默认位置 /etc/my， 或者使用  whereis my.cnf 进行查找
```
[mysqld]
# binlog 设置
binlog_format = MIXED  # binlog 记录方式，详情百度
log_bin = /home/mysql/bin_log/mysql-bin.log  # 日志存放位置
expire_logs_days = 7 #binlog过期清理时间
server-id=123456
max_binlog_size=1000m # 每个binlog文件大小
```
# 全量备份
- 编写全量备份脚本
```
采用mysqldump命令进行全量备份
参数说明；
-u : 账号
-p：密码
-F:  刷新binlog日志，即新开一个binlog日志文件，方便后面进行数据恢复的时候拿到binlog日志文件
--master-data: 记录mysqldump执行时的 binlog日志位置
```
```
#/bin/sh
stamp="`date +%Y%m%d%H%M%s`"
mysqldump -uroot -p abc  -F  --master-data  wbwf > /home/mysql/mysql_dump/wbwf_${stamp}.sql
oldFileDate=$(date -d"7 day ago" +%Y%m%d)
fileName=/home/mysql/mysql_dump/wbwf_${oldFileDate}*
rm -rf ${fileName}
```
- 添加脚本到linux定时任务
>  这里设置的是12个小时执行一次，具体可百度 linux crontab
>  1.编辑定时任务：crontab -e  
>  2.保存
>  3.查看定时任务：crontab -l 
```
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root

# For details see man 4 crontabs

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name  command to be exec
0 */12 * * * sh /home/mysql/mysqldump.sh
```
# 数据恢复
- 选择需要恢复的全量文件，即我们定时dump下来的文件
```
例如：采用 wbwf_2020041409561586829377.sql 该文件
1.查看该文件记录的binlog日志位置
	执行： cat wbwf_2020041409561586829377.sql | grep 'CHANGE MASTER'
	执行结果：CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000028', MASTER_LOG_POS=154;
2.使用 mysqlbinlog 命令解析binlog文件成sql语句
	执行：mysqlbinlog --database=wbwf --start-position=154 mysql-bin.000028 > 28.sql
	参数说明：
		--database：指定需要导出的数据库
		--start-position：指定开始位置，即第一步中获取到的 MASTER_LOG_POS
	该binlog文件后面的文件也解析出来
	mysqlbinlog --database=wbwf  mysql-bin.0000xx > xx.sql
	整理导出的binlog sql文件，移除掉误操作的 sql语句
```
- 对上面整理好的文件进行导入
```
对于dump下来的全量sql文件使用 source 命令进行导入，(大文件sql下使用source命令效率比navicat高出不少）
在服务器登录mysql:  mysql -u root -p
选择数据库：use wbwf;
进行导入：source wbwf_2020041409561586829377.sql;

对于整理出来的binlog sql文件采用source命令或者 navicat 进行导入既可
```
# 完
