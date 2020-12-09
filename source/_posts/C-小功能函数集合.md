---
title: C++小功能函数集合
tags:
  - C++
toc: true
date: 2020-09-30 15:16:54
---
本篇主要收集在工作中经常使用的小功能函数，与具体项目业务无关，主要涉及诸如字符串处理、字符编码转换等简单功能的小函数。
<!--more-->
# Windows
## 字符集
### UTF8 和 GB2312 字符集转换
``` c++
std::string Utf82Gb2312(const char* utf8)
{
    int len = MultiByteToWideChar(CP_UTF8, 0, utf8, -1, NULL, 0);
    std::unique_ptr<wchar_t []> wstr(new wchar_t[len + 1]);
    if(wstr == NULL)
    {
        return std::string();
    }
    memset(wstr.get(), 0, len + 1);
    MultiByteToWideChar(CP_UTF8, 0, utf8, -1, wstr.get(), len);
    len = WideCharToMultiByte(CP_ACP, 0, wstr.get(), -1, NULL, 0, NULL, NULL);
    std::unique_ptr<char []> str(new char[len + 1]);
    if(str == NULL)
    {
        return std::string();
    }
    memset(str.get(), 0, len + 1);
    WideCharToMultiByte(CP_ACP, 0, wstr.get(), -1, str.get(), len, NULL, NULL);

    std::string ret(str.get());
    return ret;
}

std::string Gb23122Utf8(const char* gb2312)
{
    int len = MultiByteToWideChar(CP_ACP, 0, gb2312, -1, NULL, 0);
    std::unique_ptr<wchar_t []> wstr(new wchar_t[len + 1]);
    if(wstr == NULL)
    {
        return std::string();
    }
    memset(wstr.get(), 0, len + 1);
    MultiByteToWideChar(CP_ACP, 0, gb2312, -1, wstr.get(), len);
    len = WideCharToMultiByte(CP_UTF8, 0, wstr.get(), -1, NULL, 0, NULL, NULL);
    std::unique_ptr<char []> str(new char[len + 1]);
    if(str == NULL)
    {
        return std::string();
    }
    memset(str.get(), 0, len + 1);
    WideCharToMultiByte(CP_UTF8, 0, wstr.get(), -1, str.get(), len, NULL, NULL);
    std::string ret(str.get());
    return ret;
}
```

## 字符串处理
### 获取字符串对应的十六进制字符串
``` c++
std::string String2Hex(const std::string& input)
{
    static const char hex_digits[] = "0123456789ABCDEF";

    std::string output;
    output.reserve(input.length() * 2);
    for(unsigned char c : input)
    {
        output.push_back(hex_digits[c >> 4]);
        output.push_back(hex_digits[c & 15]);
    }
    return output;
}
```
## 其他
### 使用 libcurl 上传文件
``` c
int FuncWriteResp(void *buffer, size_t size, size_t nmemb, void *userp)
{
    size_t dataLen = size * nmemb;
    std::string *str = dynamic_cast<std::string*>((std::string *)userp);
    if(0 == dataLen || nullptr == str || nullptr == buffer)
    {
        return 0;
    }

    str->append((char*)buffer, dataLen);
    return dataLen;
}

bool CurlPostUploadFile()
{
    std::string filename("log.txt");
    std::string strUrl("http://124.225.155.39:5030/upload/broadcast_resource");

    bool result = true;
    std::string response;
    struct curl_httppost *formpost = NULL;
    struct curl_httppost *lastptr = NULL;
    struct curl_slist *headers = NULL;
    curl_formadd(&formpost, &lastptr, CURLFORM_PTRNAME, "file", CURLFORM_FILE, filename.c_str(), CURLFORM_END);

    CURL *curl = curl_easy_init();
    headers = curl_slist_append(headers, "Content-Type:multipart/form-data");

    curl_easy_setopt(curl, CURLOPT_URL, strUrl.c_str());
    curl_easy_setopt(curl, CURLOPT_HTTPHEADER, headers);
    curl_easy_setopt(curl, CURLOPT_HTTPPOST, formpost);
    curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, FuncWriteResp);
    curl_easy_setopt(curl, CURLOPT_WRITEDATA, (void*)&response);

    CURLcode res = curl_easy_perform(curl);
    if(CURLE_OK != res)
    {
        printf("upload curl error, url: %s, error info:", strUrl.c_str(), curl_easy_strerror(res));
        result = false;
    }

    if(!result)
    {
        printf("upload failed, response: %s", response.c_str());
    }

    if(headers)
    {
        curl_slist_free_all(headers);
    }

    if(formpost)
    {
        curl_formfree(formpost);
    }

    curl_easy_cleanup(curl);

    return true;
}
```

### 自定义 TCP 协议解析
```
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <string.h>
#include <string>
#include <WinSock.h>

#pragma  comment(lib, "ws2_32")

#pragma pack(push,1)
struct TcpHeader
{
    uint32_t type;
    uint32_t length;
};
#pragma pack(pop)

void ReadFile(const std::string &path, const unsigned char *dst, int &length)
{
    FILE *file = fopen(path.c_str(), "rb");
    if(file == nullptr)
    {
        length = 0;
    }
    fseek(file, 0, SEEK_END);

    length = ftell(file);
    rewind(file);

    fread((void*)dst, 1, length, file);

    fclose(file);
}

void SaveFile(const unsigned char *h264, const int &length)
{
    FILE *file = fopen("D:\\parseTCP\\accesss.h264", "wb");
    if(file != nullptr)
    {
        fwrite(h264, length, 1, file);
        fclose(file);
    }
}

void SaveFileAppend(const unsigned char *h264, const int &length)
{
    FILE *file = fopen("D:\\parseTCP\\ott.h264", "ab+");
    if(file != nullptr)
    {
        fwrite(h264, length, 1, file);
        fclose(file);
    }
}

void ParsePrivateTcp()
{
    printf("sizeof of tcp header %d\n", sizeof(TcpHeader));
    int size = 50 * 1024 * 1024;
    unsigned char *buff = new unsigned char[size];
    unsigned char *h264 = new unsigned char[size];
    memset(buff, 0, size);
    memset(h264, 0, size);

    int fileLen = 0;
    std::string path("D:\\parseTCP\\access.tcp");
    ReadFile(path, buff, fileLen);

    int h264Offset = 0;
    int offset = 0;
    while(offset < fileLen)
    {
        TcpHeader *header = (TcpHeader*)(buff + offset);
        uint32_t type = ntohl(header->type);
        uint32_t len = ntohl(header->length);
        printf("heder type %u, length %u\n", type, len);
        if(type == 101)
        {
            memcpy(h264 + h264Offset, buff + offset + sizeof(TcpHeader) + 8, len - 8);
            h264Offset = h264Offset + len - 8;
        }
        offset = offset + sizeof(TcpHeader) + len;
    }

    printf("h264 length %u\n", h264Offset);

    SaveFile(h264, h264Offset);

    delete[] buff;
    delete[] h264;
}

void TcpServer()
{
    SOCKET serversoc;
    SOCKET clientsoc;
    SOCKADDR_IN serveraddr;
    SOCKADDR_IN clientaddr;
    unsigned char buf[2048];
    int len;

    WSADATA wsa;
    WSAStartup(MAKEWORD(2, 0), &wsa);

    if((serversoc = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP)) <= 0)
    {
        printf("create server socket failed\n");
        return;
    }

    serveraddr.sin_family = AF_INET;
    serveraddr.sin_port = htons(5678);
    serveraddr.sin_addr.S_un.S_addr = htonl(INADDR_ANY);

    if(bind(serversoc, (SOCKADDR *)&serveraddr, sizeof(serveraddr)) != 0)
    {
        printf("bind failed\n");
        return;
    }

    printf("listen on 5678 port...\n");

    if(listen(serversoc, 1) != 0)
    {
        printf("listen failed!\n");
        return;
    }

    len = sizeof(SOCKADDR_IN);

    if((clientsoc = accept(serversoc, (SOCKADDR *)&clientaddr, &len)) <= 0)
    {
        printf("accept failed!\n");
        return;
    }

    printf("connect success...\n");

    while(true)
    {
        memset(buf, 0, 2048);
        int recvlen = recv(clientsoc, (char *)buf, 2048, 0);
        if(recvlen <= 0)
        {
            printf("close socket\n");
            break;
        }
        printf("recv len: %d\n", recvlen);
        SaveFileAppend(buf, recvlen);
    }

    closesocket(clientsoc);
    closesocket(serversoc);
    WSACleanup();
}
```
# Linux
