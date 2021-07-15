---
title: centos 安装基本环境支持 
date: 2021-06-05 14:16:33 
categories: Linux #分类
tags: [jdk,RabbitMQ,Erlang,frp,centos] 
description: linux 安装基本环境支持 
---

[TOC]

# 一、安装 frp 内网穿透

[frp](https://github.com/fatedier/frp/blob/master/README_zh.md)就是一个[反向代理软件](https://www.zhihu.com/question/24723688)，它体积轻量但功能很强大，可以**使处于内网或防火墙后的设备对外界提供服务**，它支持HTTP、TCP、UDP等众多协议。

## 1. 服务端配置步骤

1. 下载安装包

```shell
wget https://github.com/fatedier/frp/releases/download/v0.13.0/frp_0.13.0_linux_amd64.tar.gz
```
2. 解压安装包
```shell
tar -zxvf frp_0.13.0_linux_amd64.tar.gz
```

3. 进入安装包内
```shell
cd frp_0.13.0_linux_amd64
```

4.修改服务器配置文件(配置样例如下)
```shell
vim frps.ini
```
>[common]
>bind_port = 7000 \#这个端口指的是客户端与服务端通信使用的端口
>
>
>
>#配置 dashboard（可选）
>
>dashboard_port = 7500
>
>#dashboard 用户名密码，默认都为 admin
>
>dashboard_user = admin
>dashboard_pwd = admin
>
>#web远程访问端口
>
>#vhost_http_port = 80
>#vhost_http_port = 7001

5. 后台启动（在frp_0.13.0_linux_amd64 目录下）
```shell
nohup ./frps &
```


## 2. 客户端配置步骤

>1. [frp下载](https://github.com/fatedier/frp/releases)
>
>2. 修改配置文件 **frpc.ini** 如下
>
>   [common]
>   server_addr = xx.xxx.xxx.xxx  # 公网服务器ip
>   server_port = 7000 
>
>   
>
>   [ysf]	--可添加多个，每一个名字一样才行
>
>   #web服务网络类型,可选http https
>
>   type = tcp
>   local_ip = 192.168.1.29
>
>   #内网机器的web服务端口
>
>   local_port = 8881
>   remote_port = 8881
>
>3. 在**cmd**指令中启动
>
```shell
frpc.exe
```


## 3. 注意

1. 服务端**frp**配置的所有公网端口均需放开防火墙策略
2. 如是阿里云的小伙伴，还需要在安全组中 开发端口策略

# 二、安装及卸载 jdk

## 1. 安装jdk

1. 下载安装包
```shell
wget --no-cookies --no-check-certificate --header "Cookie: gpw_e24=http%3A%2F%2Fwww.oracle.com%2F; oraclelicense=accept-securebackup-cookie" "http://download.oracle.com/otn-pub/java/jdk/8u141-b15/336fa29ff2bb4ef291e347e091f7f4a7/jdk-8u141-linux-x64.tar.gz"
```
2. 解压
```shell
tar -zxvf jdk-8u141-linux-x64.tar.gz
```

3. 编辑环境文件

```shell
vim /etc/profile
```
4. 在文末添加jdk位置

>JAVA_HOME=/root/java/jdk1.8.0_25  # jdk解压位置，可通过pwd指令查看
>
>PATH=$JAVA_HOME/bin:$PATH
>
>CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
5. 是配置文件生效

```shell
source /etc/profile
```

6. 查看是否成功
```shell
java -version
```


## 2. 卸载jdk

1.**查询系统是否以安装jdk**
```shell
rpm -qa|grep java
```

2.**卸载已安装的jdk** (.noarch文件可以不卸载)

```shell
rpm -e --nodeps xxx
```

3.验证一下是还有jdk

```shell
rpm -qa|grep java
java -version
```

# 三、安装RabbitMQ

RabbitMQ是一个开源的免费的消息队列系统，一端往消息队列中不断写入消息，而另一端则可以读取或者订阅队列中的消息。它是用Erlang编写的，并实现了高级消息队列协议（AMQP）(Advanved Message Queue Protocol)。

## 1. 安装Erlang

1.安装依赖

```shell
yum -y install gcc glibc-devel make ncurses-devel openssl-devel xmlto perl wget gtk2-devel binutils-devel
```

2.安装erlang

```shell
wget http://erlang.org/download/otp_src_22.0.tar.gz
```

3.解压

```shell
tar -zxvf otp_src_22.0.tar.gz
```

4.进入目录下并且配置安装规则

```shell
cd otp_src_22.0/ && ./configure --prefix /root/erlang/otp_src_22.0 && make && make install
```

5.添加环境变量

```shell
echo 'export PATH=$PATH:/root/erlang/otp_src_22.0/bin' >> /etc/profile
```

6.使添加环境变量生效

```shell
source /etc/profile
```

7.查看erlang是否安装成功

```shell
erl -version
```

## 2. 卸载erlang

>```shell
>yum list | grep erlang
>yum -y remove erlang-*
>yum remove erlang.x86_64
>rm -rf /usr/lib64/erlang
>```

## 3. 安装RabbitMQ

1.下载安装包

```shell
wget https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.7.15/rabbitmq-server-generic-unix-3.7.15.tar.xz
```

2.解压

```shell
tar -xvf rabbitmq-server-generic-unix-3.7.15.tar.xz
```

3.查看当前路径

```shell
pwd
```

4.配置环境变量(pwd路径 + /rabbitmq_server-3.7.15/sbin)

```shell
echo 'export PATH=$PATH:/root/rabbitmq/rabbitmq_server-3.7.15/sbin' >> /etc/profile
```

5.使环境变量配置文件生效

```shell
source /etc/profile
```

6.rabbitmq服务操作

```shell
#开启rabbitmq服务
rabbitmq-server -detached
#查看服务状态
rabbitmqctl status
#停止服务
rabbitmqctl stop
```

7.开启rabbitmq应用

```shell
#开启
rabbitmqctl start_app
#关闭
rabbitmqctl stop_app
```

8.开启web可视化插件(访问: http://***IP***:15672/)（默认账号密码均为guest【仅允许本机登录】）

```shell
#开启管理插件
rabbitmq-plugins enable rabbitmq_management
#查看插件集合
rabbitmq-plugins list
```

![image-20210607155617357](centos 安装基本环境支持/image-20210607155617357.png)

## 4. RabbitMQ用户管理

>查看所有用户
>
>```shell
>rabbitmqctl list_users
>```
>
>添加一个用户
>
>```shell
>rabbitmqctl add_user root 123456
>```
>
>配置权限
>
>```shell
>rabbitmqctl set_permissions -p "/" root ".*" ".*" ".*"
>```
>
>查看用权限
>
>```shell
>rabbitmqctl list_user_permissions root
>```
>
>设置tag
>
>```shell
>rabbitmqctl set_user_tags root administrator
>```
>
>删除用户（安全起见，删除默认用户）
>
>```shell
>rabbitmqctl delete_user guest
>```

使用配置的账号密码登录即可：

![image-20210607160801810](centos 安装基本环境支持/image-20210607160801810.png)

# 参考链接

[1. yum 安装卸载方式](https://www.cnblogs.com/qinghuaL/p/11597695.html)
[2. CentOS7安装RabbitMQ](https://www.cnblogs.com/fengyumeng/p/11133924.html)
[3.rabbitmq之后台管理和用户设置(三)](https://www.cnblogs.com/cwp-bg/p/10070467.html)

