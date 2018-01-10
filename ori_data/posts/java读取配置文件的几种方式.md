---
title: java读取配置文件的几种方式
date: 2017-01-28 10:44:16
categories: Java
tags: Java
---
## 1.使用绝对路径(不推荐,移植性差)

```java
public class Configuration {

    public static final String picTempDirUrl;
    public static final String picStoreDir;


    static{
        Properties properties=new Properties();
        InputStream in=new FileInputStream("/home/ubuntu/img/config.properties");//配置文件在磁盘的绝对路径
        try {
            properties.load(in);
            in.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
        picTempDirUrl= (String) properties.get("picTempDirUrl");
        picStoreDir= (String) properties.get("picStoreDir");
    }
}
```

<!-- more -->

## 2.使用getResourceAsStream
### Class.getResourceAsStream(String path) ：

> 路径名不以’/'开头时默认是从此类所在的包下取资源，以'/'开头则是从ClassPath根下获取。其只是通过path构造一个绝对路径，最终还是由ClassLoader获取资源。

### Class.getClassLoader.getResourceAsStream(String path) 

> 默认则是从ClassPath根下获取，经过测试无论path是不是以'/'开头，都是从ClassPath根下开始操作，最终是由ClassLoader获取资源。


**以下是用Class.getClassLoader.getResourceAsStream(String path)获取配置文件的一个例子**
```java
public class Configuration {

    public static final String picTempDirUrl;
    public static final String picStoreDir;


    static{
        Properties properties=new Properties();
        InputStream in=Configuration.class.getClassLoader().getResourceAsStream("config.properties");
        try {
            properties.load(in);
            in.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
        picTempDirUrl= (String) properties.get("picTempDirUrl");
        picStoreDir= (String) properties.get("picStoreDir");
    }
}
```
**配置文件可以放在src目录里或者resource目录里**

## 3.ServletContext.getResourceAsStream() 

```java
    Properties properties = new Properties();
    properties.load(getServletContext().getResourceAsStream("/WEB-INF/filename.properties"));
```