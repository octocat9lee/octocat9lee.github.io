---
title: mediasoup部署
tags:
  - WebRTC
toc: true
date: 2021-01-06 13:28:59
---
本文记录个人在 `Ubuntu 16.04` 和 `Centos 7` 环境下 `mediasoup` 部署过程，主要涉及编译环境搭建，依赖模块安装，公网 `IP` 和端口配置以及 `HTTPS` 设置等关键步骤。

<!--more-->

# 基础信息
## 操作系统和 nodejs 版本信息
操作系统配置信息：
`Centos 7` 版本信息：
> CentOS Linux release 7.4.1708 (Core)

`Ubuntu 16.04` 版本信息：
> Ubuntu 16.04.7 LTS
> gcc 5.4.0 20160609
> g++ 5.4.0 20160609

`Nodejs 和 npm` 版本信息：
> Node: v10.15.2
> NPM: 6.4.1

## Centos GCC 升级

因 mediasoup 编译要求 `GCC` 版本大于 `4.8`，但 `Centos 7.4` 通过 `yum` 安装的 `GCC` 版本为 `4.8.5`，不能够直接编译 `mediasoup` 使用的 `mediasoup-worker` 程序。所以，需要对 `GCC` 版本进行升级处理。具体的操作步骤如下：

- 安装 `SCL`
``` bash
yum install -y centos-release-scl
```

- 安装 `GCC`
``` bash
yum install -y devtoolset-9-gcc devtoolset-9-gcc-c++ devtoolset-9-binutils
```

- 启动 `GCC`（临时）
``` bash
scl enable devtoolset-9 bash
```

> 完成 `mediasoup-worker` 程序编译后，请关闭 `shell` 终端，不再使用 `9.3.1` 版本 `GCC`，因为 `SCL` 仅升级了 `GCC` 版本，并未升级 `libstdc++` 和 `libc` 依赖库。如果继续使用 `9.3.1` 版本 `GCC` 将导致 `mediasoup-worker` 程序在启动时找不到高版本的库函数，例如提示 `libstdc++.so.6: version `GLIBCXX_版本号' not found 和 libc.so.6: version `GLIBC_版本号' not found`。个人因为永久启动高版本 `GCC`，差点对 `libstdc++.so.6` 升级处理。

## python 升级
`mediasoup` 要求 `python` 版本高于 `3.7`，因此需要对 `python` 进行升级：
``` bash
add-apt-repository ppa:deadsnakes/ppa
apt-get update
apt-get upgrade
apt-get install python3.7
update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.7 2
update-alternatives --config python3
python3 --version
apt-get install python3-pip
```

## 生成自签名 `HTTPS` 证书
使用如下名称生成 `HTTPS` 协议使用的公钥和私钥对，其中key表示私钥，crt表示公钥。
``` bash
openssl req -new -newkey rsa:1024 -x509 -sha256 -days 3650 -nodes -out fullchain.pem -keyout privkey.pem
```
生成证书后，将证书和私钥文件复制到 `mediasoup-demo/server/certs` 目录下。

# mediasoup 部署
- 克隆项目：
``` bash
git clone https://github.com/versatica/mediasoup-demo.git
cd mediasoup-demo
git checkout v3
```

- 安装`Server`
``` bash
cd server
npm install
```
该步骤会下载 `node` 需要的 `node_modules`，其中 `c++` 部分的 `mediasoup` 代码会下载到 `mediasoup-demo/server/node_modules/mediasoup` 目录下，这个目录其实就是 `mediasoup` ，这个项目 `worker` 目录下是 `c++`，修改后直接 `make` 就可以。

如果使用 `npm install` 安装 `server` 的 `mediasoup` 失败，可考虑使用如下操作方式：
``` bash
# 设置 npm 安装源为淘宝源
npm config set registry https://registry.npm.taobao.org
cd server
npm install
# 使用 package.json 安装失败后，单独安装 mediasoup
npm install mediasoup@3 --save
```

- 安装浏览器 `app`
``` bash
cd app
npm install
# 若安装失败，尝试执行如下命令安装：
npm install --unsafe-perm
```

- 全局安装 `gulp`
``` bash
npm install -g gulp-cli
```

## 配置修改
### 服务端配置
复制 `mediasoup-demo/server/config.example.js` 重命名 `config.js`，修改后的配置文件示例如下：
``` bash
/**
 * IMPORTANT (PLEASE READ THIS):
 *
 * This is not the "configuration file" of mediasoup. This is the configuration
 * file of the mediasoup-demo app. mediasoup itself is a server-side library, it
 * does not read any "configuration file". Instead it exposes an API. This demo
 * application just reads settings from this file (once copied to config.js) and
 * calls the mediasoup API with those settings when appropriate.
 */

const os = require('os');

module.exports =
{
    // Listening hostname (just for `gulp live` task).
    domain : process.env.DOMAIN || 'localhost',
    // Signaling settings (protoo WebSocket server and HTTP API server).
    https  :
    {
        listenIp   : '0.0.0.0',
        // NOTE: Don't change listenPort (client app assumes 4443).
        listenPort : process.env.PROTOO_LISTEN_PORT || 15025,
        // NOTE: Set your own valid certificate files.
        tls        :
        {
            cert : process.env.HTTPS_CERT_FULLCHAIN || `${__dirname}/certs/fullchain.pem`,
            key  : process.env.HTTPS_CERT_PRIVKEY || `${__dirname}/certs/privkey.pem`
        }
    },
    // mediasoup settings.
    mediasoup :
    {
        // Number of mediasoup workers to launch.
        numWorkers     : Object.keys(os.cpus()).length,
        // mediasoup WorkerSettings.
        // See https://mediasoup.org/documentation/v3/mediasoup/api/#WorkerSettings
        workerSettings :
        {
            logLevel : 'warn',
            logTags  :
            [
                'info',
                'ice',
                'dtls',
                'rtp',
                'srtp',
                'rtcp',
                'rtx',
                'bwe',
                'score',
                'simulcast',
                'svc',
                'sctp'
            ],
            rtcMinPort : process.env.MEDIASOUP_MIN_PORT || 15026,
            rtcMaxPort : process.env.MEDIASOUP_MAX_PORT || 15038
        },
        // mediasoup Router options.
        // See https://mediasoup.org/documentation/v3/mediasoup/api/#RouterOptions
        routerOptions :
        {
            mediaCodecs :
            [
                {
                    kind      : 'audio',
                    mimeType  : 'audio/opus',
                    clockRate : 48000,
                    channels  : 2
                },
                {
                    kind       : 'video',
                    mimeType   : 'video/VP8',
                    clockRate  : 90000,
                    parameters :
                    {
                        'x-google-start-bitrate' : 4000
                    }
                },
                {
                    kind       : 'video',
                    mimeType   : 'video/VP9',
                    clockRate  : 90000,
                    parameters :
                    {
                        'profile-id'             : 2,
                        'x-google-start-bitrate' : 4000
                    }
                },
                {
                    kind       : 'video',
                    mimeType   : 'video/h264',
                    clockRate  : 90000,
                    parameters :
                    {
                        'packetization-mode'      : 1,
                        'profile-level-id'        : '4d0032',
                        'level-asymmetry-allowed' : 1,
                        'x-google-start-bitrate'  : 4000
                    }
                },
                {
                    kind       : 'video',
                    mimeType   : 'video/h264',
                    clockRate  : 90000,
                    parameters :
                    {
                        'packetization-mode'      : 1,
                        'profile-level-id'        : '42e01f',
                        'level-asymmetry-allowed' : 1,
                        'x-google-start-bitrate'  : 4000
                    }
                }
            ]
        },
        // mediasoup WebRtcTransport options for WebRTC endpoints (mediasoup-client,
        // libmediasoupclient).
        // See https://mediasoup.org/documentation/v3/mediasoup/api/#WebRtcTransportOptions
        webRtcTransportOptions :
        {
            listenIps :
            [
                {
                    ip          : process.env.MEDIASOUP_LISTEN_IP || '192.168.104.8',
                    announcedIp : process.env.MEDIASOUP_ANNOUNCED_IP || '10.12.13.14'
                }
            ],
            initialAvailableOutgoingBitrate : 40000000,
            minimumAvailableOutgoingBitrate : 6000000,
            maxSctpMessageSize              : 262144,
            // Additional options that are not part of WebRtcTransportOptions.
            maxIncomingBitrate              : 15000000
        },
        // mediasoup PlainTransport options for legacy RTP endpoints (FFmpeg,
        // GStreamer).
        // See https://mediasoup.org/documentation/v3/mediasoup/api/#PlainTransportOptions
        plainTransportOptions :
        {
            listenIp :
            {
                ip          : process.env.MEDIASOUP_LISTEN_IP || '192.168.104.8',
                announcedIp : process.env.MEDIASOUP_ANNOUNCED_IP || '10.12.13.14'
            },
            maxSctpMessageSize : 262144
        }
    }
};
```

主要包括修改 `HTTPS` 监听端口为 15025，`RTP` 传输端口范围为 `15026~15038`，并设置公网IP，其中 `192.168.104.8` 表示部署机器 `IP` 地址,`10.12.13.14` 表示公网 `IP` 地址。

若使用默认端口，因为在环境变量中没有对 `MEDIASOUP_LISTEN_IP` 和 `MEDIASOUP_ANNOUNCED_IP` 进行设置，所以，需要将 `config.js` 中的 `0.0.0.0` 的 `IP` 地址进行设置，否则实时视频通话会失败。在默认端口情形下，配置设置类似如下：
``` bash
listenIp :
{
    ip          : process.env.MEDIASOUP_LISTEN_IP || '192.168.104.8',
    announcedIp : process.env.MEDIASOUP_ANNOUNCED_IP || '192.168.104.8'
}
```

### 客户端配置
因为客户端默认监听在 3000 端口，当需要更该客户端默认监听端口时，需在 `mediasoup-demo/app/gulpfile.js` 文件中的 `browserSync` 添加端口配置，具体方式如下：
``` bash
browserSync(
    {
        open      : 'external',
        host      : config.domain,
        port      : 15024,
        startPath : '/?info=true',
        server    :
        {
            baseDir : OUTPUT_DIR
        },
        https     : config.https.tls,
        ghostMode : false,
        files     : path.join(OUTPUT_DIR, '**', '*')
    });
```
因服务器端将 `HTTPS` 监听端口设置为 15025，所以客户端代码亦需要修改对应端口，修改 `mediasoup-demo/app/lib/urlFactory.js` 文件中的端口：
``` bash
let protooPort = 15025;
```
修改完成后，在 `mediasoup-demo/app` 路径下执行 `gulp` 命令打包客户端代码。客户端代码打包成功后，前端客户端页面会发布到 `mediasoup-demo/server/public` 路径下。

完成配置修改后，客户端将对 `15024` 端口进行监听服务。


## `Nginx` 发布 `app`
为简化 `app` 的发布，因此使用 `nginx` 部署 `HTTPS` 服务器，发布客户端。
其中 `/home/mediasoup/mediasoup-demo/server/public` 目录为客户端发布目录。
`nginx` 配置如下所示：

``` bash
worker_processes auto;
#error_log  logs/error.log;

events {
  worker_connections 1024;
}

http {
  sendfile off;
  tcp_nopush on;
  directio 512;
  include /opt/nginx/conf/mime.types;
  # aio on;

  gzip on;

  gzip_vary on;
  gzip_proxied any;
  gzip_comp_level 6;
  gzip_buffers 16 8k;
  gzip_http_version 1.1;
  gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

  # HTTP server required to serve the player and HLS fragments
  server {
    listen 15024 ssl;
    ssl_certificate /home/mediasoup/mediasoup-demo/server/certs/fullchain.pem;
    ssl_certificate_key /home/mediasoup/mediasoup-demo/server/certs/privkey.pem;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    location / {
      root /home/mediasoup/mediasoup-demo/server/public;
      try_files $uri /index.html;
      index index.html index.htm index.php;
    }

    location ~* \.(css|gif|ico|jpg|js|png|ttf|woff)$ {
      root /home/mediasoup/mediasoup-demo/server/public;
    }

    location ~* \.(eot|otf|ttf|woff|svg)$ {
      add_header Access-Control-Allow-Origin *;
      root /home/mediasoup/mediasoup-demo/server/public;
    }
  }
}
```

# 服务运行

## 启动 `mediasoup`
使用 `nohup` 方式启动 `mediasoup` 后台运行，因某些云服务器在关闭 `shell` 终端时，同样也会结束 `nohup` 运行的进程，需要执行 `exit` 命令退出。
``` bash
cd mediasoup-demo/server/
nohup node server.js &
exit
```

## 启动 `nginx`
启动 `nginx` 服务器，发布客户端。

``` bash
/opt/nginx/sbin/nginx
```

## 测试
打开浏览器，输入 `https:\\10.12.13.14:15024` 进入房间。

# 参考资料
[github mediasoup demo](https://github.com/versatica/mediasoup-demo)
[Ubuntu部署mediasoup](https://blog.csdn.net/qq_22948593/article/details/116797801)


