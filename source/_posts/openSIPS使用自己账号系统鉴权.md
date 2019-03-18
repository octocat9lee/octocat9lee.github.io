---
title: openSIPS使用自己账号系统鉴权
tags:
  - openSIPS
toc: true
date: 2019-03-18 10:58:02
---
通常情况下，我们都不会用`opensips`数据库的`subscriber`表中的账号和密码进行鉴权，而是使用项目业务系统的账号系统进行鉴权。因此需要对鉴权模块进行相关配置。

# 鉴权模块配置

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
<!--more-->

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

