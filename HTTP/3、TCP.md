TCP报文头部结构

关于三次握手和四次挥手看[这里](https://zhuanlan.zhihu.com/p/86426969)

## 目录
- [前置知识](#before)
- [三次握手](#catch)
- [四次挥手](#shake)
- [为什么](#why)
- [TCP 长连接和短连接的区别](#long-short)
- [TCP和UDP的区别](#tcp-udp)

---
## <span id="before">**前置知识**</span>

TCP 报文常用：

seq 序号：占 32 位，用来标识从 TCP 源端向目的端发送的字节流，发起方发送数据时对此进行标记

ack 序号（确认序号）：占 32 位，只有 ACK 标志位是 1 时，确认序号才有效，ack = seq + 1

标识位：
- SYN: 发起一个新连接
- ACK: 确认序号有效
- FIN: 释放一个连接

---
## <span id="catch">**三次握手**</span>

![三次握手](./tcp/三次握手.png)

1. 客户端将标识位 SYN 设置为 1，并且随机产生一个字节流值 `seq=x`，之后向服务端发送连接请求报文，请求后，客户端进入**SYN-SENT**状态，等待服务器确认

2. 服务端收到数据包后，通过 SYN=1 知道客户端要求建立连接，服务器设置 `SYN=1，ACK=1，ack=x+1`，并随机生成一个 seq=y，将该数据包发送给客户端来确认请求连接。发送完成后服务端进入**SYN-RECEIVED**

3. 当客户端收到后，检查 ack 是否为 x+1，ACK=1，如果正确，就将标识位 `ACK=1，ack=y+1` 发送给服务端，客户端发送完这个报文后进入**ESTABLISHED**状态。服务端收到并确认数据包正确后也进入**ESTABLISHED**状态，连接建立成功

---
## <span id="shake">**四次挥手**</span>

![四次挥手](./tcp/四次挥手.png)

1. 客户端认为数据已经发送完成，向服务端B发送连接释放请求 FIN，其中 `FIN=1，seq=u`，并进入 **FIN_WAIT_1** 状态

2. 服务端收到连接释放请求 FIN 后，告诉应用层释放TCP链接，然后发送ACK包，其中 `ACK=1，ack=u+1，seq=v`，并进入**CLOSE-WAIT**状态，此时表明客户端到服务端的连接已经释放，不再接收客户端的数据，但可以继续向客户端发送数据

3. 服务端发送完数据后，向客户端发送连接释放请求 FIN，其中 `FIN=1，ACK=1，ack=u+1，seq=w`，然后进入**LAST-ACK**状态。通过延迟确认的技术，（通常由时间限制，否则对方会认为需要重传）可以将2、3合并，延迟ACK包的发送

4. 客户端收到服务端的连接释放请求 FIN 后，向服务端发送确认应答 ACK，其中 `ACK=1，seq=u+1，ack=w+1`，随后进入**TIME-WAIT**状态。该状态会持续2MSL（最大段生存期，指报文在网络中生存的时间，超时会被抛弃），若该时间内没有收到服务端的重发请求的话，进入**CLOSED**状态。服务端收到确认应答后进入**CLOSED**状态。

---
## <span id="why">**为什么**</span>

### **问题一：为什么要三次握手，而不是两次握手？（为什么客户端要再次发送确认报文）**

&emsp;&emsp;这是为了确认在即将建立连接时，两端都处于准备建立连接的状态，防止失效的连接请求报文被服务器接收而产生错误。**通过三次握手，能够让双方都知道彼此的接收能力、发送能力没问题**。

&emsp;&emsp;假设客户端发送请求连接报文A，因为网络原因超时，此时TCP启动超时重传机制，再次发送连接请求报文B，顺利到达服务端并建立连接。结束连接后，服务端又收到报文A，此时若不同客户端确认，直接进入**ESTABLISHED**状态并发送同意连接报文至客户端，而客户端已经处于**CLOSED**状态，因此连接不会建立且服务端一直处于等待状态，造成资源浪费。

&emsp;&emsp;举个例子，两个人发微信聊天，如果A对B说“听得到吗”，B回复A“听得到”，正常情况下这两次交流就可以确认并开始互发消息。但假如第一次A问“听得到吗”的时候在隧道内，导致消息并未及时发出去，而A就把微信关闭了，当B接收到A后回复一句“听得到”并做好准备聊天，导致空等。而如果A收到B的回复后再回一句“好的好的，咱开始吧”，让B知道A现在也准备好了，加一道保险，更加安全。

### **问题二：为什么连接的时候是三次握手，关闭时候却是四次挥手？**

&emsp;&emsp;关闭连接时，服务器收到对方的**FIN**报文时，仅仅表示对方不再发送数据了，但是还能接收数据，而自己也未必全部数据都发送给对方了。于是己方可以立即关闭，也可以发送一些数据给对方后，再发送**FIN**报文给对方来表示同意现在关闭连接。因此己方的**ACK**和**FIN**会分开发送，导致多了一次。




### **问题三：为什么A要进入TIME-WAIT状态，等待2MSL后再进入CLOSED状态？**

&emsp;&emsp;为了保证B能收到A的确认应答。

&emsp;&emsp;假如A发完确认应答后直接进入**CLOSED**状态，如果因为网络问题造成B未收到应答，会导致B不能正常关闭。

### **问题四：什么是SYN攻击？**

&emsp;&emsp;服务器端的资源分配是在二次握手时分配的，而客户端的资源是在完成三次握手时分配的，所以服务器容易受到 `SYN` 洪泛攻击。**`SYN` 攻击就是 `Client`在短时间内伪造大量不存在的 `IP` 地址，并向 `Server` 不断地发送 `SYN` 包，`Server` 则回复确认包，并等待 `Client` 确认，由于源地址不存在，因此 `Server` 需要不断重发直至超时，这些伪造的 `SYN` 包将长时间占用未连接队列，导致正常的 `SYN` 请求因为队列满而被丢弃，从而引起网络拥塞甚至系统瘫痪**。`SYN` 攻击是一种典型的 `DoS/DDoS` 攻击。

### **问题五：三次握手过程中可以携带数据吗？**

&emsp;&emsp;**第一、二次是不可以携带数据的，第三次可以。**

&emsp;&emsp;为什么这样呢?大家可以想一个问题，假如第一次握手可以携带数据的话，如果有人要恶意攻击服务器，那他每次都在第一次握手中的 SYN 报文中放入大量的数据。因为攻击者根本就不理服务器的接收、发送能力是否正常，然后疯狂着重复发 SYN 报文的话，这会让服务器花费很多时间、内存空间来接收这些报文。

&emsp;&emsp;也就是说，第一次握手不可以放数据，其中一个简单的原因就是会让服务器更加容易受到攻击了。而对于第三次的话，此时客户端已经处于 ESTABLISHED 状态。对于客户端来说，他已经建立起连接了，并且也已经知道服务器的接收、发送能力是正常的了，所以能携带数据也没啥毛病。

---
## <span id="long-short">**TCP 的长连接和短连接**</span>

**短连接：**

客户端与服务器建立连接开始通信，经过一次或指定次数的通信结束后，断开本次 TCP 连接。当再次通信时，重新建立 TCP 的连接。

- 优点：不长期占用服务器的内存
- 缺点：频繁操作还是会浪费资源去重复构建请求

**长连接：**

TCP 与服务器建立连接之后一直处于连接状态，可以连续发送多个数据包，在 TCP 连接保持期间，如果没有数据包发送，需要双方发送检测包以维持连接。一般需要自己做在线维持。

- 优点：
    - 传输数据快
    - 服务器嫩巩固主动第一时间传输数据到客户端
- 缺点：一直保持连接，当客户端数量较多时会占用很多系统资源，尤其是高并发分布式集群环境中

---
## <span id="tcp-udp">**TCP和UDP的区别**</span>

1. TCP 是面向链接的，虽然说网络的不安全不稳定特性决定了多少次握手都不能保证连接的可靠性，但 TCP 的三次握手在最低限度上（实际上也是很大程度上）保证了连接的可靠性；而 UDP 不是面向连接的，UDP 传送数据前并不与对方建立连接，对接收到的数据也不发送确认信号，发送端不知道数据是否会正确接收，当然也不用重发，所以说 UDP 是无连接的、不可靠的一种数据传输协议；
2. 由于 1 中所说的特点，使得 UDP 的开销更小数据、数据传输速率更高，因为不必进行收发数据的确认，所以 UDP 的实时性更好。

&emsp;&emsp;知道了两者的区别，就不难理解为什么采用 TCP 传输协议的 MSN 比采用 UDP 的 QQ 传输文件慢了，但并不意味着 QQ 的通信是不安全的，因为开发者可以手动对 UDP 的数据收发进行验证，比如发送方对每个数据包进行编号然后由接收方进行验证之类的行为。UDP 因为在底层协议的封装上没有采用类似 TCP 的**三次握手**而实现了 TCP 所无法达到的传输效率。

