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
# Linux
