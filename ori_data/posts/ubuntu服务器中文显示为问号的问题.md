---
title: Ubuntu系统中文显示为问号的问题
date: 2017-05-08 21:57:56
categories: Linux
tags: Linux
---

## 问题

因为新租的Linux服务器的字符集设置影响到后台接口返回的JSON数据中文全部变为 `？？？` 等问号乱码，并且系统中文件名中文也都是`？？？`

## 原因

简单的说是因为服务器没有安装`zh_CN.UTF-8`字符集，导致不支持中文。

`locale`     
可以执行这个命令，查看和语言编码有关的环境变量

## 解决

### 第一步

安装基本的软件包（第2步安装 zh_CN 中文字符集时要用到）

```sh
sudo apt-get update     //ubuntu系统更新软件包列表
sudo apt-get install  -y language-pack-zh-hans
sudo apt-get install -y language-pack-zh-hant
```

<!-- more -->

### 第二步

```sh
cd /usr/share/locales
sudo ./install-language-pack zh_CN   //开始安装zh_CN中文字符集
```

![](http://s3.51cto.com/wyfs02/M00/71/AD/wKioL1XWwMii2h6fAAC5JNHceVw071.jpg)

### 第三步

```sh
sudo vim /etc/environment    //编辑环境变量配置文件
```

添加下面`zh_CN.UTF-8`有关的环境变量，添加完就变成默认的了哦：
```sh
LANG=zh_CN.UTF-8
LANGUAGE=en_US:en
LC_CTYPE="zh_CN.UTF-8"
LC_NUMERIC="zh_CN.UTF-8"
LC_TIME="zh_CN.UTF-8"
LC_COLLATE="zh_CN.UTF-8"
LC_MONETARY="zh_CN.UTF-8"
LC_MESSAGES="zh_CN.UTF-8"
LC_PAPER="zh_CN.UTF-8"
LC_NAME="zh_CN.UTF-8"
LC_ADDRESS="zh_CN.UTF-8"
LC_TELEPHONE="zh_CN.UTF-8"
LC_MEASUREMENT="zh_CN.UTF-8"
LC_IDENTIFICATION="zh_CN.UTF-8"
LC_ALL=zh_CN.UTF-8
```

### 第四步

```sh
source /etc/environment   //使刚才添加的环境变量生效。最好重新登录shell再执行这指令
```

我当时到这里还不行，最后重启了一下服务器就可以了~

```sh
sudo reboot
```

## 结果

`locale` 命令看下输出结果吧！

也可以进入  `/var/lib/locales/supported.d` 


```sh
cat local
显示：
en_US.UTF-8 UTF-8
zh_CN.UTF-8 UTF-8
而没有安装中文之前只显示：
en_US.UTF-8 UTF-8
注：locale -a 可以查看操作系统支持的字符集。
```



