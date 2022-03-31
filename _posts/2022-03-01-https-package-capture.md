---
layout: post
permalink: /https-package-capture
title:  "Https抓包实践"
date:   2022-03-01 00:00:00 +0800
---

本文通过sharkwire抓包www.baidu.com的连接过程，展示TSL的握手过程。本文演示的TLS协议版本为1.2。

![image-20210904174212397](https://i.loli.net/2021/09/04/7WgpFtuoyAOlEjM.png)

ping到baidu的ip后进行过滤，可以看到，首先是进行了tcp连接。

- 不过这里要注意的是，貌似在两个端口进行了连接，可能这样是为了防止某一个请求因为网络延迟出问题吧，直接请求多次只要有一次成功就行，用户体验较好。

然后就开始进行TLS握手，我们过滤条件加上 &&ssh，过滤掉不相干的数据

### Client Hello

首先是客户端发送一个client hello

![image-20210904174235383](https://i.loli.net/2021/09/04/O9wkjiL6uAgyRJT.png)

client hello中主要包含哪些东西？

1. TLS的版本

2. **随机数**：这个是用来生成最后加密密钥的影响因子之一，包含两部分：时间戳（4-Bytes）和随机数（28-Bytes）

3. session-id：用来表明一次会话，第一次建立没有。如果以前建立过，可以直接带过去。后面的扩展内容会详细讲到。

4. **加密算法套装列表**：客户端支持的加密-签名算法的列表，让服务器去选择。

5. 压缩算法：似乎一般都不用

6. 扩展字段：比如密码交换算法的参数、请求主机的名字等等

### Server Hello

服务器回复一个server hello

![image-20210904174255880](https://i.loli.net/2021/09/04/qYR7cxPtfMan2UD.png)

server hello中主要包含哪些东西？

1. 据客户端支持的SSL/TLS协议版本，和自己的比较，确定使用的SSL/TLS协议版本
2. **确定加密套件**，压缩算法
3. 也产生了一个**随机数**

**至此**：

- 客户端和服务端确定好了加密的算法b

- 客户端和服务端都持有了各自生成的随机数，Ra和Rb，这两个随机数会在后续生成对称秘钥时用到

### 服务器发送三元组

hello之后服务器会发送三个包。

**Certificate**

接下来服务端发送 Certificate，即服务器的CA证书

![image-20210904174327783](https://i.loli.net/2021/09/04/FPEHAfMok4tQCuU.png)

可以看到证书还是挺大的，tcp将其分为了3段发送。证书中包含了签发机构、有效时间等信息，客户端收到后验证证书是否可信。

非对称加密的公钥就在证书之中。

**Server Key Exchange**

仅当服务器证书消息不包含足够的数据以允许客户端交换预主密钥(PreMaster Key)时，服务器才会发送 Server Key Exchange 消息。

![image-20210904174423133](https://i.loli.net/2021/09/04/bVFnMvgGkAJW4X7.png)

**Server Hello Done**

最后服务器发送一个 done 报文，表示沟通完毕。

![image-20210904174359683](https://i.loli.net/2021/09/04/N7RCL2okwgDeXBc.png)

> 值得注意的是，上述三个报文有时会组合到一条tcp报文中发送，但本次我的试验它们是分开的，下面客户端回传的三段是组合在一起的。

### 客户端回传三元组

![image-20210725205407666](https://i.loli.net/2021/09/04/kr7E1QbmMXUwLv6.png)

**Client Key Exchange**

- 这个也是交换秘钥参数。
- 这里客户端会根据前面的随机数Ra和Rb，生成一个**PreMaster Key**传给服务器（通常也被称为第三个随机数）。

**Change Cipher Spec**

- 编码改变通知。这一步是客户端通知服务端后面再发送的消息都会使用前面协商出来的秘钥加密了，是一条事件消息。

**Encrypted Handshake Message**

- 这一步是一个额外的验证，客户端将前面的握手消息生成摘要，再用协商好的秘钥加密，这是客户端发出的第一条加密消息。服务端接收后会用秘钥解密，解出来和握手摘要对的上说明前面协商出来的秘钥是一致的。

### 服务器发送

到这里服务器通过三个随机数Ra、Rb、和PreMaster Key进行运算，生成最终用于对称加密的**Master Key**。

接下来服务器还会发送三个报文。

**New Session Ticket**

包含了一个加密通信所需要的信息，这些数据采用一个只有服务器知道的密钥进行加密。目标是消除服务器需要维护每个客户端的会话状态缓存的要求

![image-20210904174509163](https://i.loli.net/2021/09/04/CT1GXJKu6p4WcVj.png)

**Change Cipher Spec**

编码改变通知。这一步是服务端通知客户端后面再发送的消息都会使用加密，也是一条事件消息。

![image-20210904174605425](https://i.loli.net/2021/09/04/sexwvq3I1ED9L7H.png)

**Encrypted Handshake Message**

服务端也会将握手过程的消息生成摘要再用秘钥加密，这是服务端发出的第一条加密消息。客户端接收后会用秘钥解密，能解出来说明协商的秘钥是一致的。

![image-20210904174549784](https://i.loli.net/2021/09/04/DEYKPv6hT2IwO4t.png)

**至此，TLS握手完成。**

此后，数据都以加密的形式进行传输。

![image-20210904174627238](https://i.loli.net/2021/09/04/bt3XrsIHyK5L9iE.png)

### 补充

#### SSL/TLS发展历程

这是维基百科上TLS词条下的图，可以看到目前TLS1.2是主流，TLS1.1以前的协议都被官方废弃了。

![image-20210904173832513](https://i.loli.net/2021/09/04/f21gOU98ZABGqTm.png)

#### **Master Key具体是如何生成的？**

1. 客户端随机生成随机值Ra，计算Pa = Ra * Q(x, y)，Q(x, y)为全世界公认的某个椭圆曲线算法的基点。将Pa发送至服务器。
2. 服务器随机生成随机值Rb，计算Pb =Rb * Q(x, y)。将Pb发送至客户端。由于算法的不可逆转性，外界无法通过Pa、Pb推算出Ra、Rb。
3. 因为算法保证了Ra * Pb= Rb *Pa，所以客户端计算S = Ra * Pb；服务器计算S = Rb *Pa，提取其中的S的x向量作为密钥（pre master key）
4. master key是使用伪随机算法，结合Ra，Rb，pre master key三个随机因素生成的。

更加具体的加密算法就很难深究了，感兴趣可以看 RFC5246https://www.rfc-editor.org/info/rfc5246