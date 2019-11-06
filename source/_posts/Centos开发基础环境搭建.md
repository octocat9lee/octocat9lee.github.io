---
title: Centos开发基础环境搭建
tags:
  - 技术点滴
toc: true
date: 2019-03-14 09:38:30
---
# 系统信息
``` bash
# cat /etc/redhat-release  || uname -a || cat /etc/issue || cat /proc/version || uname -r
CentOS Linux release 7.4.1708 (Core)
Linux version 3.10.0-693.el7.x86_64 (builder@kbuilder.dev.centos.org) (gcc version 4.8.5 20150623 (Red Hat 4.8.5-16) (GCC) ) #1 SMP Tue Aug 22 21:09:27 UTC 2017

# getconf LONG_BIT
```
# 安装GCC
``` bash
# yum -y update
# yum -y install gcc
# yum -y install gcc-c++
```
<!--more-->
# 安装CMake
``` bash
# yum list cmake | grep cmake
# yum -y install cmake
```
# 安装MySQL
数据库版本采用`MySQL 5.7`

## 配置YUM源

``` bash
检测MySQL源是否已经安装，若安装则跳过该步骤
# yum repolist enabled | grep "mysql.*-community.*"

配置yum源
a 下载MySQL源安装包
# wget http://dev.mysql.com/get/mysql57-community-release-el7-8.noarch.rpm

b 安装MySQL源
# yum localinstall mysql57-community-release-el7-8.noarch.rpm

c 检查MySQL源是否安装成功
# yum repolist enabled | grep "mysql.*-community.*"

显示如下提示则表示安装成功：
mysql-connectors-community/x86_64       MySQL Connectors Community            95
mysql-tools-community/x86_64            MySQL Tools Community                 84
mysql57-community/x86_64                MySQL 5.7 Community Server           327
```

## 安装MySQL数据服务

``` bash
# yum install mysql-community-server
# mysql --version
```
## 启动MySQL服务
``` bash
启动MySQ服务器
# systemctl start mysqld

查看MySQL状态
# systemctl status mysqld

加入开机启动项
# systemctl enable mysqld
# systemctl daemon-reload
```

## 修改Root密码
MySQL安装完成之后，在/var/log/mysqld.log文件中给root生成了一个默认密码。通过下面的方式找到root默认密码，然后登录mysql进行修改：
``` bash
# grep 'temporary password' /var/log/mysqld.log
2019-03-14T02:37:16.550265Z 1 [Note] A temporary password is generated for root@localhost: CaKbf4O<_3yw

使用默认root密码登录mysql
# mysql -u root -p
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'octocat';
```
NOTE：MySQL 5.7 默认安装了密码安全检查插件（validate_password），默认密码检查策略要求密码必须包含：大小写字母、数字和特殊符号，并且长度不能少于8位。否则会提示`ERROR 1819 (HY000): Your password does not satisfy the current policy requirements`错误。修改密码策略：
``` bash
在/etc/my.cnf文件添加validate_password_policy配置，指定密码策略
选择0（LOW），1（MEDIUM），2（STRONG）其中一种，选择2需要提供密码字典文件
validate_password_policy = 0

如果不需要密码策略，在my.cnf文件中添加如下配置禁用即可：
validate_password = off

重新启动MySQL服务使配置生效
# systemctl restart mysqld
```
## MySQL配置文件
MySQL配置文件为/etc/my.cnf，配置如下：
``` bash
[client]
default-character-set = utf8
port            = 3306
socket          = /var/lib/mysql/mysql.sock

## Here is entries for some specific programs
## The following values assume you have at least 32M ram
[mysqld_safe]
socket          = /var/lib/mysql/mysql.sock
nice            = 0

[mysqld]
character-set-server = utf8
user            = mysql
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/lib/mysql/mysql.sock
port            = 3306
basedir         = /usr
datadir         = /var/lib/mysql
tmpdir          = /tmp
lc-messages-dir = /usr/share/mysql

## SQL语句大小写不敏感
lower_case_table_names = 1

skip-external-locking

##
## Instead of skip-networking the default is now to listen only on
## localhost which is more compatible and is not less secure.
## 绑定地址需修改, 否则数据库使用单独的服务器会导致连接不上
bind-address  = 0.0.0.0

## 关闭密码校验
validate_password = off

sql-mode="NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION"

```
## 允许远程登录
针对root和需要远程登录的账户做以下操作，以便于用户可以远程登录。
``` bash
mysql> GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'octocat' WITH GRANT OPTION;
mysql> FLUSH PRIVILEGES;
```

# openSIPS
本节所述的内容都是基于opensips-2.3.0，版本[下载地址](http://opensips.org/pub/opensips/)。
## 安装依赖项
``` bash
# yum install mysql-devel
# yum install flex
# yum install bison
# yum install ncurses-libs
# yum install ncurses-devel
# yum install lynx
# yum install libxml2-devel
```

## 编译安装
解压并切换到代码根目录，如果希望使用默认编译选项，直接执行`make all`；如果希望修改编译参数，执行`make menuconfig`进入配置界面。

1)	选择—>`Configure Compile Options`—>`Configure Excluded Modules`，按空格选中 `db_mysql`

2)	按q键返回上一级，选择—>`Configure Install Prefix`，更改安装目录，如果不更改，默认安装在`/usr/local`目录下

3)	选择—>`Save Changes`保存修改

4)	按q返回，选择—>`Compile And Install OpenSIPS`，回车安装

如果出现依赖错误，先通过yum安装依赖。安装完成后连续按q直至退出。

## 安装目录介绍
``` bash
/opt/opensips         //opensips安装路径
├── etc//配置目录
│   └── opensips
│       ├── opensips.cfg //主要配置脚本文件
│       ├── opensipsctlrc
│       ├── osipsconsolerc
├── lib64
│   └── opensips
│       ├── modules //包含的模块动态库的目录
│       │   ├── acc.so
│       │   ├── alias_db.so
                   ... 
│       │   └── usrloc.so
│       └── opensipsctl    //命令行MI操作用到的文件
│           ├── dbtextdb
│           │   └── dbtextdb.py
│           ├── opensipsctl.base
│           ├── opensipsctl.ctlbase
│           ├── opensipsctl.dbtext
│           ├── opensipsctl.fifo
│           ├── opensipsctl.mysql
│           ├── opensipsctl.sqlbase
│           ├── opensipsctl.unixsock
│           ├── opensipsdbctl.base
│           ├── opensipsdbctl.dbtext
│           └── opensipsdbctl.mysql
├── sbin
│   ├── opensips //opensips程序可执行文件（启动opensips）
│   ├── opensipsctl //用于opensips命令行MI操作的脚本
│   ├── opensipsdbctl //opensips数据库操作脚本
│   ├── opensipsunix
│   ├── osipsconfig
│   └── osipsconsole
```

## 生成openSIPS配置
运行osipsconfig命令，依次选择—>`Generate OpenSIPS Script`—>`Residential Script`—>`Configure Residential Script`，选择如下：

<center>
![avatar](https://gitee.com/zhoulee/blog-images/raw/master/opensips_cfg.jpg)
</center>

按q键返回，选择—>`Generate Residential Script`回车，生成新的配置文件。按q键三次退出配置命令；新生成的配置文件位于`/opt/opensips/etc/opensips_residential_*.cfg`。

# 参考资料

[How To Install GCC on CentOS 7](https://linuxhostsupport.com/blog/how-to-install-gcc-on-centos-7/)