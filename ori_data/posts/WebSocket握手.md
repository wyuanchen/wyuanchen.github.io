---
title: WebSocket握手
date: 2016-07-18 21:53:38
categories: 网络协议
tags: WebSocket
---

### WebSocket是基于HTTP协议的，具体建立连接的过程如下：   

1. TCP三次握手建立一个连接    
2. 发生HTTP协议的握手请求，请求把协议升级为WebSocket       
3. 服务端接收到握手请求后，返回HTTP响应允许协议升级为WebSocket       
4. 客户端和服务端在该连接中以后都用WebSocket协议发生信息和接受信息      
5. 关闭TCP连接    

### WebSocket握手请求          

```
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
Origin: http://example.com
```


<!-- more -->

#### 相关字段解释     

```
Upgrade: websocket
Connection: Upgrade
```

- 告诉服务器我请求用WebSocket协议       

```
Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
```

- `Sec-WebSocket-Key` 是一个Base64 encode的值，这个是浏览器随机生成的      
服务端并不需要将这个值进行反编码，只需要将客户端传来的这个值首先去除首尾的空白，然后和一段固定的 GUID RFC4122 字符串进行连接，固定的 GUID 字符串为 258EAFA5-E914-47DA-95CA-C5AB0DC85B11。连接后的结果使用 SHA-1（160数位）FIPS.180-3 进行一个哈希操作，对哈希操作的结果，采用 base64 进行编码，然后作为服务端响应握手的一部分返回给浏览器。    

一个具体的例子：

1. 客户端握手请求中的 `Sec-WebSocket-Key` 头字段的值为 `dGhlIHNhbXBsZSBub25jZQ==`   
2. 服务端在解析了握手请求的头字段之后，得到 `Sec-WebSocket-Key` 字段的内容为 `dGhlIHNhbXBsZSBub25jZQ==`，注意前后没有空白     
3. 将 `dGhlIHNhbXBsZSBub25jZQ==` 和一段固定的 GUID 字符串进行连接，新的字符串为 `dGhlIHNhbXBsZSBub25jZQ==258EAFA5-E914-47DA-95CA-C5AB0DC85B11`。   
4. 使用 SHA-1 哈希算法对上一步中新的字符串进行哈希。得到哈希后的内容为（使用 16 进制的数表示每一个字节中内容）：`0xb3 0x7a 0x4f 0x2c 0xc0 0x62 0x4f 0x16 0x90 0xf6 0x46 0x06 0xcf 0x38 0x59 0x45 0xb2 0xbe 0xc4 0xea`     
5. 对上一步得到的哈希后的字节，使用 `base64` 编码，得到最后的字符串`s3pPLMBiTxaQ9kYGzzhZRbK+xOo=`       
6. 最后得到的字符串，需要放到服务端响应客户端握手的头字段 `Sec-WebSocket-Accept` 中。        


- `Sec_WebSocket-Protocol` 是一个用户定义的字符串，用来区分同URL下，不同的服务所需要的子协议。   
- `Sec-WebSocket-Version` 是告诉服务器所使用的Websocket Draft（协议版本）   

### WebSocket握手的响应   

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: HSmrc0sMlYUkAGmm5OPpG2HaGWk=
Sec-WebSocket-Protocol: chat
```

由服务端告诉客户端即将升级的是WebSocket协议。    
然后，`Sec-WebSocket-Accept` 这个则是经过服务器确认，并且加密过后的 `Sec-WebSocket-Key`。   
`Sec-WebSocket-Protocol` 则是表示最终使用的协议     

 
**只有握手还是走HTTP结束，在握手结束后的通信全都是走WebSocket协议，不是HTTP了。**           
