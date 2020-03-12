---
title: Nginx搭建文件服务器
tags:
  - 技术点滴
toc: true
date: 2020-03-11 15:12:03
---
# 编译 nginx-upload-module 模块

## 下载源码
nginx 版本和 nginx-upload-module 模块下载地址：
[nginx](<https://nginx.org/en/download.html>)
[nginx-upload-module confiuration](<http://www.grid.net.ru/nginx/upload.en.html>)
[nginx_upload_module-2.2.0.tar.gz](http://www.grid.net.ru/nginx/download/nginx_upload_module-2.2.0.tar.gz)
<!--more-->

在上述页面下载的 nginx_upload_module-2.2.0 [编译报如下错误](<https://github.com/fdintino/nginx-upload-module/issues/72>)：
``` bash
error: 'ngx_http_request_body_t' has no member named 'to_write'
```
建议直接从 github 下载 master 最新的源码：
`https://github.com/fdintino/nginx-upload-module`

## 编译
``` bash
安装编译环境
#yum -y install gcc pcre pcre-devel zlib zlib-devel openssl openssl-devel

配置选项
#./configure --prefix=/opt/nginx --with-http_stub_status_module --with-http_ssl_module --add-module=/home/zhoul/nginx/nginx-upload-module-master

编译
#make -j

安装
#make install

查看编译选项
#/opt/nginx/sbin/nginx -V
nginx version: nginx/1.16.1
...
```

编译成功后，nginx 将安装到  `/opt/nginx` 路径下。

# 配置 nginx.conf
其中关于下载和上传配置如下
``` bash
location /download {
    alias  /opt/nginx/upload;
    autoindex on;
    autoindex_localtime on;
    autoindex_exact_size off;
}

location /upload {
    upload_pass /res_upload;
    upload_store /opt/nginx/upload;
    upload_resumable on;
    upload_set_form_field "${upload_field_name}_name" $upload_file_name;
    upload_set_form_field "${upload_field_name}_content_type" $upload_content_type;
    upload_set_form_field "${upload_field_name}_path" $upload_tmp_path;

    upload_aggregate_form_field "${upload_field_name}_md5" $upload_file_md5;
    upload_aggregate_form_field "${upload_field_name}_size" $upload_file_size;
}

location /res_upload {
    proxy_pass http://localhost:9092;
}
```

# 重命名文件
使用 Node  服务对上传后的文件进行重命名，Node.js 脚本内容如下：
``` bash
const http = require('http');
const fs = require('fs');

http.createServer((req, res) => {
    let body = '';
    req.on('data', chunk => {
        body += chunk;
    });

    req.on('end', () => {
        console.log(body);
        const params = parseForm(body);
        const resParams = response(params);
        rename(params.file_path, resParams.FileName);
        jsonRes = JSON.stringify(resParams)
        res.end(jsonRes);
    })
}).listen(9092)

function parseForm(data) {
    const reg = /name="([\w_]+)"\s+(.+)\s/g;
    const params = {};
    let matched;
    while ((matched = reg.exec(data))) {
        params[matched[1]] = matched[2];
    }
    console.log(params);
    return params;
}

function rename(source, name) {
    const path = require('path');
    const dir = path.dirname(source);
    fs.renameSync(source, path.join(dir, name));
}

function response(params) {
    const res = {};
    const fileName = params.file_name.substring(0, params.file_name.lastIndexOf('.'));
    const fileExtension = params.file_name.substring(params.file_name.lastIndexOf('.') + 1);
    res["FileName"] = fileName + "-" + params.file_md5.substring(0, 8) + "." + fileExtension;
    res["FileMD5"] = params.file_md5;
    res["FileSize"] = params.file_size;
    return res;
}

```

# 参考资料
[Nginx upload module (v 2.2.0)](http://www.grid.net.ru/nginx/upload.en.html)
[centos7 使用nginx上传文件](https://www.jianshu.com/p/ef9f75094a65)
[nginx-upload-module上传文件重命名](https://www.jianshu.com/p/7d2b0567521f)
