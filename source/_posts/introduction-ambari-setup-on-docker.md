---
title: 基于Docker、Ambari部署大数据集群环境 - Ambari套件安装
updated: 2017/09/27 20:46:25
categories: 
- Techs
tags: [Docker, Ambari, Ubuntu]
---

Ambari是Apache的开源项目，它帮助用户在GUI页面上简单的部署、管理、监控Hadoop集群环境。Ambari支持的Hadoop组件包括HDFS、Hive、HBase、Spark、Yarn等，HortonWorks官方也是采用Ambari来完成自家HDP套件的安装、管理及监控的。除了预置的组件之外，Ambari还支持自定义组件的安装，同时，支持RESTful的API，继而可以通过命令行等方式调用Ambari来完成一些自动化的任务。

本文共分为两部分，第一部分介绍如何在Docker虚拟化环境中部署Ambari；第二部分介绍如何基于Ambari来部署和管理Hadoop集群。

<!-- more -->
### 环境信息
* Docker发行版：Docker for Mac
* Docker版本：17.06.2-ce
* Docker容器OS：Ubuntu 14.04
* Ambari版本：2.5.2.0

### Docker环境准备
1. 拉取Docker镜像：在宿主机上执行命令 `docker pull ubuntu:14.04` 从远端仓库中获取Ubuntu的镜像，也可以获取其他OS的镜像，本文以Ubuntu为例
2. 启动Docker容器：执行如下命令，以ubuntu:14.04镜像为基础启动容器：
		docker run -itd --name ambari_new -p 8080:8080 -p 3306:3306 -v /Users/yuxiaolei/Workspace/dockerShared:/dockerShared ubuntu:14.04 /bin/bash

由于Ambari启动Web程序的时候占用8080端口，因此要从Docker宿主机上访问Ambari页面，需要通过参数 -p 来制定端口映射；
作为新手，笔者在容器内部署好Ambari之后，才发现Web页面的8080端口和MySQL的3306端口（可选）没有暴露给Docker宿主机，也就没法从宿主机上通过浏览器来登陆Ambari，因此必须想办法在已有容器上开放端口。

有两个方法：
1）如果宿主机为Linux系统，则修改iptables防火墙来指定端口映射规则；
2）如果是非Linux系统，可以将已装Ambari的容器commit为新的镜像，再基于该镜像创建新的容器。此时，就可以在 `docker run` 命令中添加参数 -p 来指定端口映射了。

还有一个问题，Ambari将其数据存储在数据库中，支持MySQL、PostgreSQL等数据库；容器内安装MySQL之后，基于上一步创建的新容器里，会发现MySQL启动不起来，执行命令 `/etc/init.d/mysql restart` 启动失败，在 `/var/log/mysql/error.log` 日志文件中打印有 `170802 14:02:59 [ERROR] Fatal error: Can't open and lock privilege tables: Got error 140 from storage engine` 的错误，经过网上查资料，需要在创建容器的时候添加参数 `-v /var/lib/mysql` 将MySQL数据存储路径声明为数据卷，即可解决问题。
启动容器之后，执行命令 `docker exec -it ambari /bin/bash` 进入容器内部。

### Ambari安装
1. 配置Ubuntu的软件仓库源：
	国内建议采用阿里云的软件源，在root账号下用vim打开`/etc/apt/sources.list`文件，删除文件所有内容，粘贴如下内容：
	```shell
	deb http://mirrors.aliyun.com/ubuntu/ trusty main restricted universe multiverse
	deb http://mirrors.aliyun.com/ubuntu/ trusty-security main restricted universe multiverse
	deb http://mirrors.aliyun.com/ubuntu/ trusty-updates main restricted universe multiverse
	deb http://mirrors.aliyun.com/ubuntu/ trusty-proposed main restricted universe multiverse
	deb http://mirrors.aliyun.com/ubuntu/ trusty-backports main restricted universe multiverse
	deb-src http://mirrors.aliyun.com/ubuntu/ trusty main restricted universe multiverse
	deb-src http://mirrors.aliyun.com/ubuntu/ trusty-security main restricted universe multiverse
	deb-src http://mirrors.aliyun.com/ubuntu/ trusty-updates main restricted universe multiverse
	deb-src http://mirrors.aliyun.com/ubuntu/ trusty-proposed main restricted universe multiverse
	deb-src http://mirrors.aliyun.com/ubuntu/ trusty-backports main restricted universe multiverse
	```
	执行命令 `apt-get update` 完成软件列表更新

2. 安装Ambari所依赖的软件
	* **Oracle JDK：** 逐条执行如下命令，以添加WebUpd8团队（[https://launchpad.net/~webupd8team/+archive/ubuntu/java](https://launchpad.net/~webupd8team/+archive/ubuntu/java)）提供的Oracle JDK仓库源，并从该仓库安装JDK：
	```shell
	apt-get install software-properties-common
	sudo add-apt-repository ppa:webupd8team/java
	sudo apt-get update
	sudo apt-get install oracle-java8-installer
	sudo apt-get install oracle-java8-set-default
	```
		完成安装之后，在 ~/.bashrc 文件末尾添加命令 `export JAVA_HOME=/usr/lib/jvm/java-8-oracle ` 以配置JAVA\_HOME 环境变量。

	* **MySQL：** 执行命令 `apt-get install mysql-server` 安装MySQL服务器，安装完成后执行命令 `mysql -uroot -proot` 进入MySQL客户端，执行如下SQL代码：
	```sql
	create database ambari;
	use ambari;
	GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'root_password' WITH GRANT OPTION;
	FLUSH PRIVILEGES;
	exit;
	```
		由于Ambari的数据存储在MySQL数据库中，这里为Ambari单独创建了database，并为其赋了完全控制权限；说明：假设MySQL数据库root用户的密码为：root\_password
		修改 `/etc/mysql/my.conf`，将`skip-external-locking`注释掉，并确保 `bind-address = 0.0.0.0` 配置，以使MySQL可被远程主机访问。
		执行命令`/etc/init.d/mysql restart`重启MySQL 服务。

	* **时间同步服务器ntp：** 执行命令 `apt-get install ntp`  安装ntp时间同步服务器，以便于集群环境中各节点的时钟一致；执行命令 `sudo service ntp restart`  重启ntp服务。

3. 下载Ambari仓库文件
	* 进入 `cd /etc/apt/sources.list.d` 目录，执行命令 `wget http://public-repo-1.hortonworks.com/ambari/ubuntu14/2.x/updates/2.5.2.0/ambari.list` 从HortonWorks仓库中下载Ambari源文件，下载后切勿修改list文件名；
	* 执行命令 `apt-key adv --recv-keys --keyserver keyserver.ubuntu.com B9733A7A07513CAD` 以信任远端仓库的GPG签名
	* 执行命令 `apt-get update` 更新Ambari软件源
	* 执行命令 `apt-get install ambari` 安装Ambari套件，由于软件包较大（700多MB），这里情耐心等待，不过apt-get支持断点下载，网络终端后重新执行命令时不会从零开始下载
4. 配置Ambari：
	* 执行命令 `mysql -uroot -proot` 进入MySQL客户端，执行命令 `source ambari` 进入ambari的数据库，并执行命令 `source /var/lib/ambari-server/resources/Ambari-DDL-MySQL-CREATE.sql` 来完成Ambari的数据库表初始化操作；
	* 执行命令 `ambari-server setup` 启动Ambari的引导式配置操作，根据指导做配置即可。需要注意的是，JDK不要选择由Ambari从网络下载，应该选择自定义路径，然后输入 `/usr/lib/jvm/java-8-oracle`  即可；
5. 启动Ambari：执行命令`ambari-server start`，启动日志存储路径为 `/var/log/ambari-server/ambari-server.log`
6. 启动之后，由于我们之前做了Docker容器的端口映射，因此可以在宿主机上打开浏览器输入 [http://localhost:8080](http://localhost:8080/ "http://localhost:8080") 即可访问Ambari登陆页面
7. 登陆用户名和密码均为admin，登陆之后就可以看到Ambari的首页了，如下图：
	{% asset_img DraggedImage.png Ambari首页 %}
	

下一篇文章会介绍Ambari分布式环境配置和管理。