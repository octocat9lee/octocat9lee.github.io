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

openSIPS 数据库配置文件位于`/opt/opensips/etc/opensips`路径下的`opensipsctlrc`。关于配置文件的详细说明参见官方文档[Database Deployment v2.3](http://www.opensips.org/Documentation/Install-DBDeployment-2-3)。

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
#USERCOL="username"
.......
# 其他不变
```
配置完成后的[配置文件](https://gitee.com/zhoulee/tools/blob/master/opensips/opensipsctlrc)。
<!--more-->
## 创建数据库
当对数据库凭证配置完成后，使用`/opt/opensips/sbin`目录下的`opensipsdbctl`工具生成数据库。具体步骤如下：
``` bash
[root@localhost sbin]# ./opensipsdbctl create
MySQL password for root: octocat(MySQL root 用户密码)

生成过程中的交互选项：字符集选择->latin1  其他选项选择 n
```
>NOTE：如果出现ERROR 1101 (42000) at line 2: BLOB, TEXT, GEOMETRY or JSON column 'extra_hdrs' can't have a default value这类错误，则在`/etc/my.cnf`文件内的`[mysqld]`标签下添加一行 ：
`sql-mode = "NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION"`

当数据库创建完成后，在MySQL数据库中可以看到名称为`opensips`的数据库，包含27张表，对于每张表的描述和字段的含义参考官方文档[OpenSIPS database tables](http://www.opensips.org/Documentation/Install-DBSchema-2-3)。

因为没有自己域名系统，因此使用官方的域名。通过openSIPS提供的管理接口，使用官方的`opensips.org`作为整个openSIPS系统的Domain，具体操作如下：
``` bash
opensipsctl domain add opensips.org
```
当添加完成后，在`opensips`数据库的`domain`能够查看到对应的添加记录信息。当直接使用IP地址作为SIP账号域名时不需要使用该步骤添加域名配置。

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

## 配置RTPProxy模块
RTPProxy模块指定openSIPS与RTP转发通讯相关信息。下面使用UNIX套接字与RTP进行通讯，该配置需要与RTPProxy运行时的配置保持一致。
``` bash
loadmodule "rtpproxy.so"
modparam("rtpproxy", "rtpproxy_sock", "unix:/tmp/rtpproxy.unix") # CUSTOMIZE ME
```
最后的[配置文件参考](https://gitee.com/zhoulee/tools/blob/master/opensips/opensips.cfg)。

# RTPProxy编译和运行

## RTPProxy项目
项目主页以及GitHub地址：
``` bash
主页：http://www.rtpproxy.org/
GitHub项目：https://github.com/sippy/rtpproxy
```

## RTPProxy编译

``` bash
# mkdir /opt/rtpproxy
# git clone -b master https://github.com/sippy/rtpproxy.git
# git -C rtpproxy submodule update --init --recursive
# cd rtpproxy
# ./configure --prefix=/opt/rtpproxy
# make -j
# make install
```
当执行完上述命令时，`RTPProxy`将被安装到`/opt/rtpproxy`目录下。
对于低版本的`git`，由于不支持`-C`选项，故执行如下命令：
``` bash
# cd rtpproxy
# git submodule update --init --recursive
```
在最新版本的 RTPProxy 中，因为使用了 stdatomic 文件，需要将 GCC 版本升级到 4.9 及以上方可编译。通过查看提交记录，在[rtpp_memdeb.c](https://github.com/sippy/rtpproxy/commits/master/src/rtpp_memdeb.c)的 `3a345c587590600ed9bcd82282f77044e7811ed3` 提交中使用了 stdatomic 。如果不需要使用最新的特性，使用如下命令切换到如下版本即可解决：
``` bash
# git checkout 15786f2f229e241b901636d8c98c4b849317aeca
```

因为RTPProxy对每一路会话的流量进行统计，并限制每一路会话流量为100，所以当使用RTPProxy对视频流进行转发时会经常导致花屏现象，具体解决方案如下：[H264 frame corrupted when using rtpproxy](https://github.com/sippy/rtpproxy/issues/38)
``` bash
What you can do in a short-term, however, is to increase max PPS rate from
100 currently to say 1000, that would give you sufficient resolution to
handle this kind of streams, at expense of some extra CPU burnout. This can
be done by changing:

#define      MAX_RTP_RATE    100

to

#define      MAX_RTP_RATE    1000

and then re-compiling the code.
```

## 运行RTPProxy
``` bash
# cd /opt/rtpproxy
#./rtpproxy -A 10.0.204.60 -l 10.0.204.60 -s unix:/tmp/rtpproxy.unix -m 2000 -M 2100 -F

选项参数含义：
-A  本机外网IP
-l  本地内网IP
-s  与opensips通讯的socket，与opensips.cfg配置文件保持一致
-F  不检查是否为超级用户模式
-m  RTP最小端口
-M  RTP最大端口
-d  调试消息输出级别
```
如果需要对 RTPProxy 进行调试，使用如下方式运行 RTPProxy :
``` bash
# rtpproxy -d DBUG:LOG_MAIL -A 10.0.204.60 -l 10.0.204.60 -s unix:/tmp/rtpproxy.unix -m 2000 -M 2100 -F
```
此时，能够在 `/var/log/maillog` 文件中查看日志输出。

# 基本功能测试
至此，`openSIPS`环境基本配置完毕，可以使用SIP客户端进行基本功能测试。在运行RTPProxy之后，再运行opensips程序。在调式模式下，输出信息中没有错误信息，即可认为`opensips`运行成功。

##  添加测试账号
``` bash
# ./opensipsctl add zhoulee@10.0.204.60:5060 123456
# ./opensipsctl add octocat@10.0.204.60:5060 123456
```
当测试账号添加后，可以在`opensips`数据库的`subscriber`表中查看到对应的账号信息。

## linphone
使用[linphone](https://www.linphone.org/)作为SIP客户端，对部署的opensips环境进行基本功能测试。在linphone界面选择`USE A SIP ACCOUNT`，在如下界面中输入添加测试账号时的账号和SIP服务器信息，具体配置如下所示：
<center>
![avatar](https://gitee.com/zhoulee/blog-images/raw/master/linphone_client.jpg)
</center>
添加完成后，在SIP服务器端使用MI命令查看在线用户信息：
``` bash
# ./opensipsctl ul show
Domain:: location table=512 records=1
    AOR:: zhoulee
        Contact:: sip:zhoulee@10.0.20.121;transport=udp Q=
            ContactID:: 3070469783448198127
            Expires:: 557
            Callid:: D8Wj6UbWPj
            Cseq:: 21
            User-agent:: Linphone Desktop/4.1.1 (belle-sip/1.6.3)
            Received:: sip:10.0.20.121:5060
            State:: CS_NEW
            Flags:: 0
            Cflags:: NAT
            Socket:: udp:10.0.204.60:5060
            Methods:: 4294967295
            SIP_instance:: <urn:uuid:01c3eeec-76ce-4c2a-9aa9-63afcffd7027>
```
使用两个测试账号即可进行音视频对讲测试。

## tcpdump调试
在测试过程中，可以使用`tcpdump`工具获取SIP协议交互报文。
``` bash
# tcpdump -i ens192 -nn -s0 -SvX port 5060 -w opensips_register.pcap
```
查看是否接收到RTP数据包，`10.0.22.40`为手机客户端IP地址
``` bash
# tcpdump -i ens192 -nn -s0 udp and host 10.0.22.40

当输出如下信息时，表示客户端已经和rtpprox在进行数据传输：
14:37:05.155346 IP 10.0.22.40.7076 > 10.0.204.60.2062: UDP, length 78
14:37:05.174776 IP 10.0.204.60.2062 > 10.0.22.40.7076: UDP, length 83
14:37:05.176192 IP 10.0.22.40.7076 > 10.0.204.60.2062: UDP, length 80
并且，可以看出，rtpproxy通讯使用的端口号为2062
```
同时，使用如下命令查看rtpproxy打开的端口情形：
``` bash
# netstat -anp | grep rtpproxy
udp        0      0 10.0.204.60:2046        0.0.0.0:*                           126226/./rtpproxy   
udp        0      0 10.0.204.60:2047        0.0.0.0:*                           126226/./rtpproxy   
udp     2176      0 10.0.204.60:2062        0.0.0.0:*                           126226/./rtpproxy   
udp        0      0 10.0.204.60:2063        0.0.0.0:*                           126226/./rtpproxy   
unix  2      [ ACC ]     STREAM     LISTENING     1314540  126226/./rtpproxy    /tmp/rtpproxy.unix
```
可以看出，`2062`的端口号已经被udp协议占用。

# 参考资料
[Tutorials-GettingStarted Video](https://www.opensips.org/Documentation/Tutorials-GettingStarted)



