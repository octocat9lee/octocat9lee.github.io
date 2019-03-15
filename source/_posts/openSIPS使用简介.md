---
title: openSIPS使用简介
tags:
  - openSIPS
toc: true
date: 2019-03-15 11:26:49
---
按照[Centos基础开发环境中的openSIPS小节](https://octocat9lee.github.io/2019/03/14/Centos%E5%BC%80%E5%8F%91%E5%9F%BA%E7%A1%80%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA/#openSIPS)的步骤完成openSIPS的编译和生成基础配置文件，下面开始对openSIPS数据库和配置文件进行详细介绍。

# openSIPS数据库

## 配置数据库凭证

openSIPS数据库配置文件位于`/opt/opensips/etc/opensips`路径下的`opensipsctlrc`。关于配置文件的详细说明参见官方文档[Database Deployment v2.3](http://www.opensips.org/Documentation/Install-DBDeployment-2-3)。

进入`opensipsctlrc`文件目录，打开配置文件，去掉关于DB相关注释，基本内容如下所示：

```  bash
# this parameter.
DBENGINE=MYSQL

## database port (PostgreSQL=5432 default; MYSQL=3306 default)
DBPORT=3306

## database host
DBHOST=localhost

## database name (for ORACLE this is TNS name)
DBNAME=opensips

# database path used by dbtext, db_berkeley, or sqlite
DB_PATH="/usr/local/etc/opensips/dbtext"

## database read/write user
DBRWUSER=opensips

## password for database read/write user
DBRWPW="opensipsrw"

## engine type for the MySQL/MariaDB tabels (default InnoDB)
MYSQL_ENGINE="MyISAM"

## database super user (for ORACLE this is 'scheme-creator' user)
DBROOTUSER="root"

# user name column
USERCOL="username"
.......
# 其他不变
```
<!--more-->
## 创建数据库
当对数据库凭证配置完成后，使用`/opt/opensips/sbin`目录下的`opensipsdbctl`工具生成数据库。具体步骤如下：
``` bash
[root@localhost sbin]# ./opensipsdbctl create
MySQL password for root: octocat(MySQL root 用户密码)

生成过程中的交互选项：字符集选择->latin1  其他选项选择 y
```
>NOTE：如果出现ERROR 1101 (42000) at line 2: BLOB, TEXT, GEOMETRY or JSON column 'extra_hdrs' can't have a default value这类错误，则在`/etc/my.cnf`文件内的`[mysqld]`标签下添加一行 ：
`sql-mode = "NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION"`

当数据库创建完成后，在MySQL数据库中可以看到名称为`opensips`的数据库，包含27张表，对于每张表的描述和字段的含义参考官方文档[OpenSIPS database tables](http://www.opensips.org/Documentation/Install-DBSchema-2-3)。

因为没有自己域名系统，因此使用官方的域名。通过openSIPS提供的管理接口，使用官方的`opensips.org`作为整个openSIPS系统的Domain，具体操作如下：
``` bash
opensipsctl domain add opensips.org
```
当添加完成后，在`opensips`数据库的`domain`能够查看到对应的添加记录信息。

## 管理接口
openSIPS提供了很多管理接口（`Manager Interface, MI`），通过这些接口，可以对openSIPS的运行状态进行查询，或者实时更新openSIPS的运行数据。下面列举常见的管理接口：
``` bash
运行、重启、停止opensips：opensipsctl start|restart|stop

查看命令帮助：opensipsctl help

新建用户：opensipsctl add 101 101 #添加用户名101，密码101

查看fifo提供的操作：opensipsctl fifo which

重新加载负载均衡信息：opensipsctl fifo lb_reload

查看负载均衡配置的服务器状态：opensipsctl fifo lb_list

查看注册在opensips上的用户：opensipsctl ul show

查看opensips的统计信息：opensipsctl fifo get_statistics all

查看opensips当前通话数等信息：opensipsctl fifo get_statistics dialog:
```
更多的管理接口参见官方[Management Interface v2.3](http://www.opensips.org/Documentation/Interface-MI-2-3)。

# openSIPS配置
将按照小节[生成openSIPS配置](https://octocat9lee.github.io/2019/03/14/Centos%E5%BC%80%E5%8F%91%E5%9F%BA%E7%A1%80%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA/#%E7%94%9F%E6%88%90openSIPS%E9%85%8D%E7%BD%AE)生成的配置文件重命名为`opensips.cfg`，作为openSIPS运行配置。关于配置文件的详细格式参见文档[Script Format v2.3](http://www.opensips.org/Documentation/Script-Format-2-3)，对于各个字段含义参见官方文档[Core Parameters v2.3](http://www.opensips.org/Documentation/Script-CoreParameters-2-3)。

对于openSIPS模块功能描述参见openSIPS[模块说明文档](http://www.opensips.org/Documentation/Modules-2-3)，对于每个模块支持的配置参数和配置函数参见[模块说明文档](http://www.opensips.org/Documentation/Modules-2-3)的`Parameters section`。

## 设置监听端口
定制配置文件中`listen`字段值，设置SIP服务器监听地址和端口，[设置方式](http://www.opensips.org/Documentation/Script-CoreParameters-2-3#toc63)：
``` bash
listen=tcp:0.0.0.0:5060   # CUSTOMIZE ME
listen=udp:0.0.0.0:5060   # CUSTOMIZE ME 
```

## 配置URI模块
使用如下配置更改URI模块对应字段值，[URI模块各个字段含义文档](http://www.opensips.org/html/docs/modules/2.3.x/uri.html)
``` bash
#### URI module
loadmodule "uri.so"
modparam("uri", "use_uri_table", 0)
modparam("uri", "db_table", "m_user") # my_user_db数据库中的m_user表
modparam("uri", "user_column", "Name") # m_user表中的账号字段名 
modparam("uri", "domain_column", "Domain") # m_user表中的Domain字段名
```

## 配置鉴权Auth模块
通常情况下，我们都不会用`opensips`数据库的`subscriber`表中的账号和密码进行鉴权，而是使用项目业务系统的账号系统进行鉴权。也就是使用业务系统的数据库中的账号密码进行鉴权。如下图配置：
``` bash
#### AUTHentication modules
loadmodule "auth.so"
loadmodule "auth_db.so"
modparam("auth_db", "calculate_ha1", yes)
modparam("auth_db", "user_column", "Name")  # 鉴权表中的账号字段名
modparam("auth_db", "domain_column", "Domain")
modparam("auth_db", "password_column", "Password") # 鉴权表中的密码字段名
modparam("auth_db", "password_column_2", "HA1B")
modparam("auth_db|uri", "db_url", "mysql://opensips:opensipsrw@10.0.15.2:3306/my_user_db") # CUSTOMIZE ME 数据库链接设置
modparam("auth_db", "load_credentials", "")
```
关于`m_user`表中用户名列`Name`，密码列`Password`以及`ha1`列和`ha1b`列计算方法：
1. 用户名列为注册的用户名
2. 密码列为密码明文md5加密后的值，即md5(plaintext)作为`Password`列的值
3. ha1=md5(username:Domain:md5(plaintext) ) 在上面将Domain配置为`opensips.org`
4. ha1b=md5(username@Domain:Domain:md5(plaintext))

例如：假设用户名为`zhoulee`，用户密码明文`123456`，
则`Password`的值为`md5(123456)=e10adc3949ba59abbe56e057f20f883e`，
`ha1= md5(zhoulee:opensips.org:e10adc3949ba59abbe56e057f20f883e)`，
`ha1b=md5(zhoulee@opensips.org:opensips.org:e10adc3949ba59abbe56e057f20f883e)`。
[在线MD5加密网站](http://tool.chinaz.com/tools/md5.aspx)

## 对用户表授权
openSIPS使用的数据库默认为`opensips`，但在URI和Auth模块配置中，把用户表指向了`my_user_db`数据库的`m_user`表，所以要给MySQL的`opensips`用户开放权限：
``` bash
grant select on my_user_db.m_user to opensips@'localhost';
grant select on my_user_db.version to opensips@'localhost';
grant select on my_user_db.m_user to opensips@'%';
grant select on my_user_db.version to opensips@'%';
```
并同时在`my_user_db`数据库的`version`表中插入如下2条记录：
``` bash
insert into version(table_name, table_version) values('m_user', 7);
insert into version(table_name, table_version) values('subscriber', 7);
```

## 配置RTPProxy模块
RTPProxy模块指定openSIPS与RTP转发通讯相关信息。下面使用UNIX套接字与RTP进行通讯，该配置需要与RTPProxy运行时的配置保持一致。
``` bash
loadmodule "rtpproxy.so"
modparam("rtpproxy", "rtpproxy_sock", "unix:/tmp/rtpproxy.unix") # CUSTOMIZE ME
```

## 配置代理鉴权和注册鉴权
修改`proxy_authorize(realm, table)`和`www_authorize(realm, table)`中的`table`字段`subscriber`为自定义鉴权数据表名`m_user`。
<center>
![avatar](https://github.com/octocat9lee/blog-images/raw/master/opensip_subscriber.jpg)
</center>

## 修改Route
将路由设置中的`if(has_totag())`和`route[relay]`函数中的`is_method("INVITE")`判断中添加`UPDATE`字段变成`is_method("INVITE|UPDATE")`。
<center>
![avatar](https://github.com/octocat9lee/blog-images/raw/master/opensips_update.jpg)
</center>


