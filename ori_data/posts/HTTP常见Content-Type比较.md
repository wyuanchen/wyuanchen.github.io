---
title: HTTP常见Content-Type比较
date: 2017-01-28 11:02:37
categories: 网络协议
tags: Http
---
## 1.   application/x-www-form-urlencoded  
最常见的 `POST` 提交数据的方式了。浏览器的原生 form 表单，如果不设置 `enctype` 属性，那么最终就会以 `application/x-www-form-urlencoded `方式提交数据。
传递的key/val会经过URL转码，所以如果传递的参数存在中文或者特殊字符需要注意。
```
//例子
//b=曹,a=1

POST  HTTP/1.1(CRLF)
Host: www.example.com(CRLF)
Content-Type: application/x-www-form-urlencoded(CRLF)
Cache-Control: no-cache(CRLF)
(CRLF)
b=%E6%9B%B9&a=1(CRLF)
//这里b参数的值"曹"因为URL转码变成其他的字符串了
```

<!-- more -->

## 2. text/xml
```
//例子

POST http://www.example.com HTTP/1.1(CRLF) 
Content-Type: text/xml(CRLF)
(CRLF)
<?xml version="1.0"?>
<resource>
    <id>123</id>
    <params>
        <name>
            <value>homeway</value>
        </name>
        <age>
            <value>22</value>
        </age>
    </params>
</resource>
```

## 3.application/json   

```

//例子
//传递json

POST  HTTP/1.1(CRLF)
Host: www.example.com(CRLF)
Content-Type: application/json(CRLF)
Cache-Control: no-cache(CRLF)
Content-Length: 24(CRLF)
(CRLF)
{
    "a":1,
    "b":"hello"
}
```

## 4. multipart/form-data   

使用表单上传文件时，必须让 `form` 的 `enctyped` 等于这个值。
并且Http协议会使用boundary来分割上传的参数  

```
//例子
//a="曹",file1是一个文件

POST  HTTP/1.1(CRLF)
Host: www.example.com(CRLF)
//注意data;和boundary之间有一个空格,并且----WebKitFormBoundary7MA4YWxkTrZu0gW是可以自定义的
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW(CRLF)
Cache-Control: no-cache(CRLF)
Content-Length: 728
(CRLF)
//如果有Content-Length的话，则Content-Length指下面所有的字节总数，包括boundary
//这里用自定义的boundary来进行分割,注意会在头部加多"--"
------WebKitFormBoundary7MA4YWxkTrZu0gW(CRLF)
Content-Disposition: form-data; name="a"(CRLF)
(CRLF)
曹(CRLF)
------WebKitFormBoundary7MA4YWxkTrZu0gW(CRLF)
Content-Disposition: form-data; name="file1"; filename="1.jpg"
Content-Type: application/octet-stream(CRLF)
(CRLF)
//此处是参数file1 对应的文件的二进制数据
[654dfasalk;af&6…](CRLF)
//最后一个boundary会分别在头部和尾部加多"--"
------WebKitFormBoundary7MA4YWxkTrZu0gW--(CRLF)

```

```

//多个文件同时上传

POST  HTTP/1.1(CRLF)
Host: www.example.com(CRLF)
//注意data;和boundary之间有一个空格,并且----WebKitFormBoundary7MA4YWxkTrZu0gW是可以自定义的
Content-Type: multipart/form-data; boundary=---------------------------418888951815204591197893077
Cache-Control: no-cache(CRLF)
Content-Length: 12138(CRLF)
(CRLF)
-----------------------------418888951815204591197893077(CRLF)
// 文件1的头部boundary
Content-Disposition: form-data; name="userfile[]"; filename="文件1.md"(CRLF)
Content-Type: text/markdown(CRLF)
(CRLF)
// 文件1内容开始
// ...
// 文件1内容结束
-----------------------------418888951815204591197893077(CRLF)
// 文件2的头部boundary
Content-Disposition: form-data; name="userfile[]"; filename="文件2"(CRLF)
Content-Type: application/octet-stream(CRLF)
(CRLF)
// 文件2内容开始
// ...
// 文件2内容结束
-----------------------------418888951815204591197893077(CRLF)
// 文件3的头部boundary
Content-Disposition: form-data; name="userfile[]"; filename="文件3"(CRLF)
Content-Type: application/octet-stream(CRLF)
(CRLF)
// 文件3内容开始
// ...
// 文件3内容结束
-----------------------------418888951815204591197893077(CRLF)
// 参数username的头部boundary
Content-Disposition: form-data; name="username"(CRLF)
(CRLF)
zhangsan
-----------------------------418888951815204591197893077(CRLF)
// 参数password的头部boundary
Content-Disposition: form-data; name="password"(CRLF)
(CRLF)
zhangxx
-----------------------------418888951815204591197893077-- 
// 尾部boundary，表示结束

```

**注意**
**`(CRLF)`指`\r\n`**

附上其他一些博客
[http协议](https://callmeli.github.io/2016/08/22/http%E5%8D%8F%E8%AE%AE/)
[HTTP协议详解](http://www.jianshu.com/p/abaf583f1183)
