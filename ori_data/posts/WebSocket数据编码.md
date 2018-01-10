---
title: WebSocket数据编码
date: 2016-07-25 17:22:39
categories: 网络协议 
tags: WebSocket 
---

WebSocket传输的数据都是以Frame（帧）的形式实现的，就像TCP/UDP协议中的报文段Segment。下面就是一个Frame：（以bit为单位表示）   

```
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-------+-+-------------+-------------------------------+
 |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
 |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
 |N|V|V|V|       |S|             |   (if payload len==126/127)   |
 | |1|2|3|       |K|             |                               |
 +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
 |     Extended payload length continued, if payload len == 127  |
 + - - - - - - - - - - - - - - - +-------------------------------+
 |                               |Masking-key, if MASK set to 1  |
 +-------------------------------+-------------------------------+
 | Masking-key (continued)       |          Payload Data         |
 +-------------------------------- - - - - - - - - - - - - - - - +
 :                     Payload Data continued ...                :
 + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
 |                     Payload Data continued ...                |
 +---------------------------------------------------------------+

```

<!-- more -->

### 各字段解释   

#### FIN：1 bit   

表示这是一个消息的最后的一帧。第一个帧也可能是最后一个。    
- 0 : 还有后续帧     
- 1 : 最后一帧    

#### RSV1、2、3： 1 bit each      

除非一个扩展经过协商赋予了非零值以某种含义，否则必须为0   
如果没有定义非零值，并且收到了非零的RSV，则websocket链接会失败    

#### Opcode： 4 bit     

解释说明 “Payload data” 的用途/功能  
如果收到了未知的opcode，最后会断开链接   
定义了以下几个opcode值:   

- 0 : 代表连续的帧   
- 1 : text帧   
- 2 ： binary帧      
- 3-7 ： 为非控制帧而预留的     
- 8 ： 关闭握手帧    
- 9 ： ping帧    
- A :  pong帧      
- B-F ： 为非控制帧而预留的     

#### Mask： 1 bit  

定义“payload data”是否被添加掩码   
如果置1， “Masking-key”就会被赋值   
**所有从客户端发往服务器的帧都会被置1**        

#### Payload length： 7 bit | 7+16 bit | 7+64 bit    

- payload data 的长度如果在0~125 bytes范围内，它就是payload length，  
- 如果是126， 紧随其后的被表示为16 bits的2 bytes无符号整型就是payload length，     
- 如果是127， 紧随其后的被表示为64 bits的8 bytes无符号整型就是payload length   

#### Masking-key： 0 or 4 bytes    

所有从客户端发送到服务器的帧都包含一个32 bits的掩码（如果“mask bit”被设置成1），否则为0 bit。    
一旦掩码被设置，所有接收到的payload data都必须与该值以一种算法做异或运算来获取真实值。   

#### Payload data: (x+y) bytes   

它是"Extension data"和"Application data"的总和，一般扩展数据为空。

#### Extension data: x bytes   

- 除非扩展被定义，否则就是0   
- 任何扩展必须指定其Extension data的长度   

#### Application data: y bytes   

占据"Extension data"之后的剩余帧的空间   

## Masking   

当mask字段的值为1时，payload-data字段的数据需要经这个掩码进行解密。   

- 服务器推送到客户端的消息中，mask字段是0,也就是说Masking-key为空。这样的话，数据的解析就不涉及到掩码，直接使用就行。    
- 如果消息是从客户端发送到服务器，那么mask一定是1,Masking-key一定是一个32bit的值。    

下面为mask为1的一个例子   

当读取到payload-data时，首先将数据按byte依次与Masking-key中的4个byte按照如下算法做异或。    

```java
private void maskPayload(byte[] payload) {
   for(int i=0;i<payload.length;i++){
        payload[i]^=mask[i%4];
    }
}
```

## 帧的类别      

### 控制帧   

- 控制帧用来说明WebSocket的状态信息，用来控制分片、连接的关闭等等。所有的控制帧必须有一个小于等于125字节的payload，并且control Frames不允许被分片。     
- Opcode为0x0（持续的帧），0x8（关闭连接），0x9（Ping帧）和0xA（Pong帧）代表控制帧     


### 数据帧    

前面我们总是谈到“控制帧”和“非控制帧”，想必大家已經看出来一些门路。其实数据帧就是非控制帧。因为这个帧并不是用来提供协议连接状态信息的。   
数据帧由最高符号位是0的Opcode确定，现在可用的几个数据帧的Opcode是0x1（utf-8文本）、0x2（二进制数据）。      


## 分片（Fragment）   

理论上来说，每个帧（Frame）的大小是没有限制的，因为payload-data在整个帧的最后。但是发送的数据有不能太大，否则 WebSocket 很可能无法高效的利用网络带宽。那如果我们想传点大数据该怎么办呢？WebSocket协议给我们提供了一个方法：分片，将原本一个大的帧拆分成数个小的帧。下面是把一个大的Frame分片的图示：   

>  编号：      0  1  ....  n-2 n-1
  分片：     |——|——|......|——|——|
  FIN：      0  0  ....   0  1
  Opcode：   !0 0  ....   0  0   

由图可知，    

- 第一个分片的FIN为0，Opcode为非0值（0x1或0x2）   
- 最后一个分片的FIN为1，Opcode为0    
- 中间分片的FIN和Opcode二者均为0     

## 关闭连接   

正常的连接关闭流程   

1. 发送关闭连接请求（Close Handshake）
即发送Close Frame（Opcode为0x8）。一旦一端发送/接收了一个Close Frame，就开始了Close Handshake，并且连接状态变为Closing。   
2. Close Frame中如果包含Payload data，则data的前2字节必须为两字节的无符号整形，（同样遵循网络字节序：BE）用于表示状态码，如果2byte之后仍有内容，则应包含utf-8编码的关闭理由。     
3. 如果一端在之前未发送过Close Frame，则当他收到一个Close Frame时，必须回复一个Close Frame。但如果它正在发送数据，则可以推迟到当前数据发送完，再发送Close Frame。比如Close Frame在分片发送时到达，则要等到所有剩余分片发送完之后，才可以作出回复。   
4. 关闭WebSocket连接   
5. 当一端已经收到Close Frame，并已发送了Close Frame时，就可以关闭连接了，close handshake过程结束。这时丢弃所有已经接收到的末尾字节。   
6. 关闭TCP连接   
7. 当底层TCP连接关闭时，连接状态变为Closed。     


### clean closed

- 如果TCP连接在Close handshake完成之后关闭，就表示WebSocket连接已经clean closed（彻底关闭）了。   
- 如果WebSocket连接并未成功建立，状态也为连接已关闭，但并不是clean closed。   

### 正常关闭   

正常关闭过程属于clean close，应当包含close handshake。  

通常来讲，应该由服务器关闭底层TCP连接，而客户端应该等待服务器关闭连接，除非等待超时的话，那么自己关闭底层TCP连接。    


### 异常关闭

由于某种算法或规定，一端直接关闭连接。（特指在open handshake（打开连接）阶段）
底层连接丢失导致的连接中断。 

### 连接失败

由于某种算法或规范要求指定连接失败。这时，客户端和服务器必须关闭WebSocket连接。当一端得知连接失败时，不准再处理数据，包括响应close frame。     

