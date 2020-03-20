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
#user  nobody;
worker_processes  4;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        charset utf-8;
        listen       9090;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        client_max_body_size 1024m;

        location /static {
            alias /opt/upload;
        }

        location /download {
            alias  /opt/nginx/upload;
            autoindex on;
            autoindex_localtime on;
            autoindex_exact_size off;
        }

        location /upload {
            upload_pass /nginx_upload_rename;
            upload_store /opt/nginx/upload;

            upload_store_access user:rw group:r all:r;
            upload_resumable on;
            upload_set_form_field "${upload_field_name}_name" $upload_file_name;
            upload_set_form_field "${upload_field_name}_content_type" $upload_content_type;
            upload_set_form_field "${upload_field_name}_path" $upload_tmp_path;

            upload_aggregate_form_field "${upload_field_name}_md5" $upload_file_md5;
            upload_aggregate_form_field "${upload_field_name}_size" $upload_file_size;

            upload_limit_rate 10000000;
            upload_cleanup 400 404 499 500-505;
        }

        location /nginx_upload_rename {
            proxy_pass http://localhost:9092;
        }

        location / {
            root   html;
            index  index.html index.htm;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
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

C++ 方式重命名文件：

``` bash
struct NginxUploadRequest
{
    std::string filename;     // 原始文件名
    std::string filetype;     // 文件类型
    std::string uploadPath;   // nginx 上传后随机文件名
    std::string filemd5;      // 文件 MD5
    std::string filesize;     // 文件大小
};

static const std::string NGINX_UPLOAD_PATH("/opt/nginx/upload/");

static bool FileExistInDir(const std::string &filename)
{
    // 文件存在返回 true; 不存在返回 false
    bool exist(true);
    if((::access(filename.c_str(), F_OK)) == -1)
    {
        exist = false;
    }
    return exist;
}

static std::string GetNewFileAbsolutePath(const std::string &originPath, const std::string &newName)
{
    char buf[512] = {0};
    strncpy(buf, originPath.c_str(), originPath.length());
    const char *dir = ::dirname(buf);
    return std::string(dir) + "/" + newName;
}

static bool RenameUploadFile(const std::string &originPath, const std::string &newPath)
{
    bool result(false);
    // 防止错误路径执行重命名命令
    if(originPath.substr(0, NGINX_UPLOAD_PATH.length()) == NGINX_UPLOAD_PATH)
    {
        struct stat originStat;
        stat(originPath.c_str(), &originStat);
        if(S_ISREG(originStat.st_mode))
        {
            // 常规文件才重命名
            int ret = ::rename(originPath.c_str(), newPath.c_str());
            if(ret == 0)
            {
                result = true;
            }
        }
        else
        {
            LOG4_ERROR("not regular file, path: " << originPath);
        }
    }
    else
    {
        LOG4_ERROR("upload path error, path: " << originPath);
    }
    return result;
}

static std::string GetNginxUploadKeyValue(const std::string &key, const std::string &content)
{
    const std::string PREFIX("Content-Disposition: form-data; name=\"");
    const std::string POSTFIX("\n------WebKitFormBoundary");
    std::string searchKey = PREFIX + key + std::string("\"\n\n");
    std::size_t begin = content.find(searchKey);
    std::string value;
    if(begin != std::string::npos)
    {
        begin += searchKey.length();
        std::size_t end = content.find(POSTFIX, begin);
        std::size_t len = end - begin;
        value = content.substr(begin, len);
    }
    return value;
}

static NginxUploadRequest ParseNginxUploadRequest(const std::string &request)
{
    NginxUploadRequest loadFields;
    std::string body(request);
    // 统一换行符为 \n
    body.erase(std::remove(body.begin(), body.end(), '\r'), body.end());
    std::size_t hasBody = body.find("Content-Disposition: form-data; name=\"file_name\"");
    if(hasBody != std::string::npos)
    {
        loadFields.filename = GetNginxUploadKeyValue("file_name", body);
        loadFields.filetype = GetNginxUploadKeyValue("file_content_type", body);
        loadFields.uploadPath = GetNginxUploadKeyValue("file_path", body);
        loadFields.filemd5 = GetNginxUploadKeyValue("file_md5", body);
        loadFields.filesize = GetNginxUploadKeyValue("file_size", body);
        LOG4_TRACE("nginx upload infos, filename: " << loadFields.filename << ", type: " << loadFields.filetype
            << ", path: " << loadFields.uploadPath << ", md5: " << loadFields.filemd5 << ", size: " << loadFields.filesize);
    }
    else
    {
        LOG4_ERROR("upload request error: " << request);
    }
    return loadFields;
}

// cinatra http router function
m_https.route("/nginx_upload_rename", [this](cinatra::request const &req, cinatra::response &res, cinatra::context_container &ctx) {
    std::string request = req.body().to_string();
    NginxUploadRequest uploads = ParseNginxUploadRequest(request);

    JsonStringify json;
    int code = 1;
    std::string msg("Filename is Empty");
    if(!uploads.filename.empty())
    {
        std::string newFile = GetNewFileAbsolutePath(uploads.uploadPath, uploads.filename);
        if(!FileExistInDir(newFile))
        {
            bool renameRet = RenameUploadFile(uploads.uploadPath, newFile);
            if(renameRet)
            {
                json.AddKeyValue(std::make_pair("FileName", uploads.filename));
                json.AddKeyValue(std::make_pair("FileMD5", uploads.filemd5));
                json.AddKeyValue(std::make_pair("FileSize", uploads.filesize));
                code = 0;
                msg = std::string("Success");
            }
            else
            {
                code = 3;
                msg = std::string("Rename Failed");
            }
        }
        else
        {
            code = 2;
            msg = std::string("File Exist");
        }
    }

    if(code != 0)
    {
        // 设置 400, nginx upload module 自动删除上传的文件
        res.set_status(cinatra::response::bad_request);
    }

    JsonStringify::JsonObjectValue objValue;
    objValue.push_back(std::make_pair("Code", JsonValue(std::to_string(code), kJSON_INT)));
    objValue.push_back(std::make_pair("Msg", msg));
    json.AddObject("Res", objValue);
    std::string response = json.Stringify();
    LOG4_TRACE("nginx rename response: " << response);
    res.response_text(response);
});
```


# 参考资料

[Nginx upload module (v 2.2.0)](http://www.grid.net.ru/nginx/upload.en.html)
[centos7 使用nginx上传文件](https://www.jianshu.com/p/ef9f75094a65)
[nginx-upload-module上传文件重命名](https://www.jianshu.com/p/7d2b0567521f)
