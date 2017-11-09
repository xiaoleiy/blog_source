---
title: 基于Docker、Ambari部署大数据集群环境（续）- 大数据集群服务安装部署
updated: 2017/11/04 21:38:25
categories: 
- Techs
tags: [Docker, Ambari, Ubuntu]
---

接上一篇，本文继续讲述如何基于Docker容器采用Ambari完成大数据环境的搭建。首先，除了上一篇用来安装Ambari软件包的容器（作为Master节点）之外，还要创建其他几台Docker容器，作为Agent节点；本文不覆盖Docker技术本身，但是对创建的Docker容器有如下要求：每个Docker容器赋予一个固定IP，避免容器重启之后IP被动态分配，导致集群节点之间无法通信；

<!-- more -->
### 配置Master、Agent节点之间的SSH证书认证
1. Agent容器上，开启root用户登录权限
	* 登录Agent容器，执行命令`passwd root` 设置root用户的密码；
	* 用vi编辑 `/etc/ssh/sshd_config` 文件，修改属性 PermitRootLogin 属性为yes，改为：`PermitRootLogin  yes`
	* 在所有Agent容器上执行相同的上述操作
2. Master容器上，生成SSH证书
	* 登录Master容器，在`/etc/hosts`文件中，以FQDN（Fully Qualified Domain Name）的形式追加Master及所有Agent机器的主机名及IP地址映射，参考如下（第二列为Agent的域名，自定义即可；第三列为Agent的hostname，创建Docker容器时生成）：
			172.17.0.2	ambari_master   7f4584bdc01d
			172.17.0.4	ambari_agent01	0c3438ea3089
			172.17.0.5	ambari_agent02	8b8312c4fe15
	* 执行命令 `ssh-keygen`，输入参数均默认即可，执行完成之后会在 `~/.ssh/` 目录下生成一对SSH认证证书，包含公钥和私钥2个文件：id\_rsa、id\_rsa.pub
	* 在`~/.ssh/config`文件（如果没有则手动创建即可）中添加如下配置，以指定连接Agent的私钥：
			Host 172.17.0.4
			   IdentitiesOnly yes
			   IdentityFile ~/.ssh/id_rsa
			 Host 172.17.0.5
			   IdentitiesOnly yes
			   IdentityFile ~/.ssh/id_rsa
			 ControlMaster auto
			 ControlPath /tmp/%r@%h:%p
	* 执行命令`scp ./id\_rsa.pub root@172.17.0.4:~/` 将公钥文件传输到所有Agent机器上
3. Agent容器上，配置SSH公钥
	* 登录Agent容器，如果没有`~/.ssh`目录则创建，在root根目录下执行命令 `cat ./id\_rsa.pub  > ~/.ssh/authorized_keys `，完成公钥的配置
	* 在`/etc/hosts`文件中，以FQDN（Fully Qualified Domain Name）的形式追加Master及所有Agent机器的主机名及IP地址映射，参考如下：
			172.17.0.2	ambari_master   7f4584bdc01d
			172.17.0.4	ambari_agent01	0c3438ea3089
			172.17.0.5	ambari_agent02	8b8312c4fe15
	* 在所有Agent容器上执行上述操作
	* 在Ambari安装过程中，Master容器会携带自己的私钥，通过scp/ssh/sftp的方式向Agent容器传输文件，并且执行命令

### Agent容器完成Java / ntp 的安装
参考[上一篇文章](https://xiaoleiy.github.io/2017/09/27/introduction-ambari-setup-on-docker/#more)安装Java和ntp服务。

### 安装Hadoop集群环境
登录 Amarbi 首页，点击“Launch Install Wizard”进入Hadoop集群环境安装指导；
{% asset_img Launch Install Wizard.png 启动集群环境的安装 %}

按照如下指导步骤，完成环境安装：
1. _Getting Started_ 输入该集群环境的名字，点击“Next”进入下一步；
2. _Select Version_ 选择HDP版本，HDP是 Hortonworks 开发的Hadoop大数据基础套件，由于它集成了Hadoop/Spark/Flume等诸多组件，因此给开发者带来了很大方便；笔者选择的是HDP最新版本 HDP 2.6.2.0，软件仓库选择“Use Public Repository”，这样在安装过程中， Ambari 会从远程软件仓库下载各组件安装包。点击“Next”进入下一步；
3. _Install Options_ Target Hosts 一栏，输入Agent容器的hostname，每行一个；Host Registration Information 一栏，上传Master容器内创建的SSH密钥文件，便于Master远程通过密钥向各个Agent机器安装HDP套件；点击“Next”进入下一步；
4. _Confirm Hosts_ 等待所有Agent机器都完成安装；如果安装失败，点击状态列可以查看日志，以找到具体原因；Ambari会进入各Agent容器去检查服务、目录等安装配置情况，可以点击“Click to see the warnings”以确认具体配置问题，并做修复。
	{% asset_img Confirm-Hosts.jpg 集群节点确认页面 %}
	确认安装完成且无配置问题之后，点击“Next”进入下一步；
5. _Choose Services_  选择需要安装的组件，组件之间存在依赖关系，Ambari会自己识别，如果被依赖组件没有选择，在点击“Next”的时候他会提示是否选择。点击“Next”进入下一步；
6. _Assign Masters_ 为各个组件选择Master节点，笔者当前按照Ambari默认选择即可，点击“Next”进入下一步；
7. _Assign Slaves and Clients_ 为各个组件选择Slave节点已经客户端，笔者当前按照Ambari默认选择即可，点击“Next”进入下一步；
8. _Customize Services_ 完成各组件的参数配置修改，Ambari会将需要配置的组件标记出来，点击对应页签之后，手动修改属性，基本上都是数据库或服务密码。需要关注的是，Hive的数据库需要选择已存在于Master节点上的MySQL数据库，并输入对应密码。
	{% asset_img Customize-Services.jpg 服务配置页面 %}	
	完成属性修改之后，点击“Next”进入下一步；
9. _Review_ 确认上一步的各组件参数配置，点击“Next”进入下一步；
10. _Install, Start and Test_ 开始安装，Ambari会在各个Agent节点上同时安装各组件
	{% asset_img Install-Start-and-Test.jpg 启动服务安装 %}	
	点击链接可查看当前安装进展，如下图：
	{% asset_img Install-Details.png 服务安装详情查看 %}	
	点击正在安装的组件条目，还可以查看日志路径，以及当前输出的日志信息；安装完成之后，Ambari会启动所有服务，并测试是否成功，这个过程中，如果出现异常或告警，也会在最后显示出来，以便我们自己进行核对修正。
	{% asset_img Install-Review.png 服务安装结果核对 %}
	点击“Next”进入下一步；
11. _Summary_ 对于之前的服务安装、启动、测试情况进行汇总显示
	{% asset_img Install-Start-Test-Summary.png 服务安装、启动、测试完成 %}
	点击“Complete”完成安装；

完成安装之后，Ambari会引导进入Dashboard管理页面，可以进行各服务、集群节点等的管理；

	