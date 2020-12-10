---
title: jenkins 使用 总结
date:  2018-12-18 11:26:05 
tags: [jenkins]
categories: [linux,jenkins]
---
@[toc]
# 写在前面
> 这不是一篇从头到尾的教程，是我根据网上的教程进行搭建，中间遇到的问题或者说觉得重要的点的一些记录，适合参考着看。
# 安装
```
wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat/jenkins.repo
rpm --import https://pkg.jenkins.io/redhat/jenkins.io.key
yum install jenkins
```
# 配置
```
// 配置端口 
vi /etc/sysconfig/jenkins
JENKINS_PORT="8080" #这里是加入的
JAVA_CMD="$JENKINS_JAVA_CMD $JENKINS_JAVA_OPTIONS -DJENKINS_HOME=$JENKINS_HOME -jar $JENKINS_WAR"
```
# 启动
```
// 启动/停止/重启/状态
service jenkins start
service jenkins stop
service jenkins restart
service jenkins status
```
# 操作
```
访问 http://ip:8080
// 查看密码
cat /var/lib/jenkins/secrets/initialAdminPassword
// 选择左边的默认安装
// 创建管理员用户
```
# 邮件配置
```
// 邮件通知
系统设置：
	Jenkins Location > 系统管理员邮件地址: 这里需要配置
	邮件通知 > 
		SMTP服务器:smtp.163.com 或 smtp.qq.com 
		勾选 使用SMTP认证
		用户名：邮箱名（这里需要与上面 系统管理员邮件地址 一致）
		密码：这里填的是设置邮箱 开通 POP3/STMP 功能时设置的认证密码，不是邮箱密码！！！

	测试：
		勾选 通过发送测试邮件测试配置
		Test e-mail recipient：接受的邮箱地址，测试的时候可以直接填 系统管理员邮件地址 地址，就是自己发给自己，发送给其他人时，因为内容中有 test 关键字，所以会报 554  错误，意思是内容非法
错误代码说明：
https://blog.csdn.net/u013938484/article/details/51939587
```
# 新建任务
> 在这之前需要安装插件，gitLab/Maven Integration
> 参考链接：https://blog.csdn.net/alinyua/article/details/81103570
```
1.构建一个自由风格的软件项目（适合多个项目代码融合在一起的项目，灵活度高）：
	可以在《构建》选项中 选择 执行 shell脚本，脚本里可以 写 maven 构建，然后docker发布，很灵活
2.构建一个maven项目（适合单个maven项目）：
	在 《Build》选项中 配置好就可以了。这里配置完后，执行构建，就会把打包成 jar/war，在 《Post Steps》中可以选择执行shell，来启动项目

构建项目时没有 ·构建maven项目·：
参考链接：http://www.cnblogs.com/zhizhao/p/9442411.html
原因：没有安装插件，插件名：Maven Integration Plugin
```
# 错误记录
```
错误一（启动时报错）：
	错误信息：Starting Jenkins bash: /usr/bin/java: 没有那个文件或目录
	解决办法：
		查看Java_home：
			[root@kk ~]# echo $JAVA_HOME
			/opt/jdk-8u171-linux-x64/jdk1.8.0_191
		修改配置文件：
			vi /etc/rc.d/init.d/jenkins
			candidates="
			/etc/alternatives/java
			/usr/lib/jvm/java-1.8.0/bin/java
			/usr/lib/jvm/jre-1.8.0/bin/java
			/usr/lib/jvm/java-1.7.0/bin/java
			/usr/lib/jvm/jre-1.7.0/bin/java
			/usr/bin/java
			/opt/jdk-8u171-linux-x64/jdk1.8.0_191/bin/java # 此处是加上的内容
			"
错误二（源码配置时）：
	错误信息：stderr: error: The requested URL returned error: 401 Unauthorized while accessing http://192.168.1.33:8000/csbr/blockchain.git/info/refs
	解决办法：url上加上用户名
	例如：http://username@192.168.1.33:8000/csbr/blockchain.git

错误三（执行 mvn install 进行项目构建时）
	错误信息：[ERROR] Failed to execute goal org.apache.maven.plugins:maven-compiler-plugin:3.8.0:compile (default-compile) on project csbr-boot-starter: Error while storing the mojo status: /var/lib/jenkins/workspace/test/csbr-boot-starter/target/maven-status/maven-compiler-plugin/compile/default-compile/inputFiles.lst (权限不够)

	解决办法：
		vim /etc/sysconfig/jenkins
		JENKINS_USER="root" # 修改用户为root
		
错误四（设置密码后时候首次登录失败）
https://blog.csdn.net/galen2016/article/details/84648620
linux 中 config.xml 在 /var/lib/jenkins 目录中，重新设置密码后记得把配置文件换回来，然后重启jenkins

错误五（构建shell中执行 nohup 启动失败）
问题的根本在于是Jenkins使用processTreeKiller杀掉了所有子进程，而且这是Jenkins的默认行为。为了解决该问题，我们需要在启动前加上这句
BUILD_ID=DONTKILLME
防止Jenkins 杀死我们的进程。
如下：
BUILD_ID=DONTKILLME
nohup java -jar test.jar &
```
# 结合实际项目说明

我们git上的项目是结合在一起的，不是每个服务对应git上的一个项目
git 目录
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181218113027697.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3phaWdlNjY=,size_16,color_FFFFFF,t_70)
各个服务是在一个项目中，所以无法在构建项目时选择 构建一个 maven 项目，我选择的是 **构建一个自由风格的软件项目**
那么问题来了：如何对每个项目进行打包，并发布呢
我是通过 **在《构建》选项中 选择 执行 shell脚本**
具体shell内容：
```shell
#!/bin/bash

echo "======================";
echo $PATH;
echo "======================";

# 生成 start jar包
echo "===========start install start========="
/opt/maven/apache-maven-3.5.3/bin/mvn -f /var/lib/jenkins/workspace/test/csbr-boot-starter install

echo "===========core package start========="
# 打包 core
/opt/maven/apache-maven-3.5.3/bin/mvn -f /var/lib/jenkins/workspace/test/blockchain-core package

# 把 core 部署到docker
cp  /var/lib/jenkins/workspace/test/blockchain-core/target/blockchain-core-0.1.jar /opt/docker/projects/blockchain-core
echo "===========docker start========="
bash /opt/docker/run start blockchain-core
```
具体操作就是 自己调用 maven 命令进行打包，然后 调用自己编写的脚本，把打包好的 jar/war 部署到docker中

**执行 mvn 命令时会报错，提示 该 命令找不到，所以我用的是绝对路径**
**还会遇到错误三，解决办法看上面**

希望本文对您有所帮助！
