---
title: Centos 使用 rsync 同步文件
tags:
  - 技术点滴
toc: true
date: 2020-07-07 10:50:23
---
使用 rsync 在两台 Centos 服务器间同步多个文件夹，并配置定时任务，实现自动同步。
<!--more-->
#  环境
主服务器：`10.0.204.75`
备份服务器：`10.0.204.64`
主服务器和备份服务器安装 `rsync` 软件：
``` bash
yum -y install rsync
```

# 服务端配置
在主服务器 `10.0.204.75` 上进行服务器端的配置，`rsync` 服务端安装完成之后是没有生成 `rsync.conf` 文件的，需要手动在 `/etc` 目录下创建 `rsyncd.conf` 配置文件。
``` bash
vim /etc/rsyncd.conf
```

服务端 `rsyncd.conf` 配置文件示例：

``` bash
# global setting
uid = root
gid = root
list = false
use chroot = no
read only = yes
hosts allow = 10.0.204.64 10.0.204.75
max connections = 4
log file = /var/log/rsyncd.log
pid file = /var/run/rsyncd.pid
lock file = /var/run/rsync.lock
secrets file = /etc/rsyncd.secrets
　　
# rsync path
[rms]
path = /opt/upload
ignore errors
auth users = root
comment = rms

[nginx]
path = /home/nginx/upload
ignore errors
auth users = root
comment = nginx
```
 
配置同步的用户名和密码：

``` bash
# touch /etc/rsyncd.secrets
# echo "root:123456" > /etc/rsyncd.secrets
# chown root:root /etc/rsyncd.secrets
# chmod 600 /etc/rsyncd.secrets
```

启动 `rsync`，并设置为开机启动：
``` bash
# /usr/bin/rsync --daemon
# vi /etc/rc.d/rc.local

rm -f /var/run/rsyncd.pid
/usr/bin/rsync --daemon
```

# 客户端配置
在备份服务器 `10.0.204.64` 上生成与服务端对应密码配置文件，注意这里只需要服务器 `rsyncd.secrets` 中的密码。
``` bash
# touch /etc/rsyncd.secrets
# echo "123456" > /etc/rsyncd.secrets
# chown root:root /etc/rsyncd.secrets
# chmod 600 /etc/rsyncd.secrets
```

在备份服务器端执行同步命令：
``` bash
#!/bin/bash
rsync -au --password-file=/etc/rsyncd.secrets root@10.0.204.75::rms /home/backup/static
rsync -au --password-file=/etc/rsyncd.secrets root@10.0.204.75::nginx /home/backup/static
```

建立定时任务：
``` bash
# crontab -e
00 02 * * * /home/backup/rsync_backup.sh
```
