---
title: TCP 协议笔记
date: 2022-07-20 10:15:26
tags:
- TCP
---


# TCP
---

## 报文种类

- SYN
- Data
- FIN
- Reset
- ACK


SYN、Data、FIN 这三种报文发送方一定要收到 ACK 报文，如果在超时时间内不确认，发送方会一直重传，直到对方确认，或者达到重传上线次数而 Reset 连接。


## 三次握手


<img src="https://cannedbread-1302516612.cos.ap-shanghai.myqcloud.com/crypto/TCP%20%E6%8F%A1%E6%89%8B.drawio.svg">


三次握手本质上是四次报文交互：

1. A 发送 SYN 报文给 B
2. B 发送 ACK 报文给 A，确认步骤 1
3. B 发送 SYN 报文给 A
4. A 发送 ACK 报文给 B，确认步骤 3

为了缩短延迟，将步骤 2、步骤 3 合并在一个报文发送，即 SYN + ACK 报文，形成三次握手。


第 1 次握手：

Client 将标志位 SYN 置为 1，随机产生一个 Initial sequence number（简称为 seq），假设 seq = x

将该数据包发送给 Server，Client 进入 SYN_SENT 状态


第 2 次握手：

Server 接收到数据包后，由标志位 SYN = 1 知道该报文是 Client 请求建立连接

Server 也会记录 Client 的 seq=x

> Server 不会将 SYN 与 SYN+ACK 混淆，因为只有 SYN=1

Server 准备一个报文，将标志位 SYN 和 ACK 都置为 1，ack = x + 1，随机产生一个 seq = y，并将该数据包发送给 Client，进入 SYN_RCVD 状态



第 3 次握手：

Client 收到确认后，检查 ack 是否为 x + 1，如果正确，则将标志位 ACK 置为 1，ack = y + 1，并将该数据包发送给 Server，Client 进入 ESTABLISHED 状态

Server 检查 ack 是否为 y + 1，ACK 是否为 1，如果正确，则建立连接成功，Server 进入 ESTABLISHED 状态



## 四次挥手

![在这里插入图片描述](https://img-blog.csdnimg.cn/2021071716371797.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5MjkxOTE5,size_16,color_FFFFFF,t_70)
假设 Client 主动关闭连接，

第 1 次挥手：Client 发送一个 FIN，以及一个序号 seq，告知 Server：我打算关闭连接了，Client 进入 FIN-WAIT-1 状态

第 2 次挥手：Server 收到 FIN 后，发送 ACK 给 Client，Server 进入 CLOSE-WAIT 状态，这时候 Server 可能还有一些数据没有传输完毕，继续传输。

第 3 次挥手：当 Server 没有数据发送给 Client 时，Server 发送一个 FIN 给 Client，告知 Client：我也没有数据给你了，我也可以关闭了，Server 进入 LAST-ACK 状态

第 4 次挥手，Client 收到 FIN，Client 进入 TIME_WAIT 状态，发送 ACK 给 Server，Server 接收到 ACK 之后，进入 CLOSED 状态。


### TIME_WAIT 存在的问题？
一个连接进入 TIME_WAIT 状态后需要等待 2 * MSL 的事件才能断开连接释放连接占用的资源

服务器：短时间内关闭了大量的 Client 连接，会造成服务器上出现大量的 TIME_WAIT 连接，占据大量的 tuple，严重消耗着服务器的资源。

客户端：短时间内大量的短链接，会大量消耗 Client 机器的接口，端口只有 65535 个，端口被耗尽，无法发起新的连接。

关于 Windows 的端口使用问题：
参数 [MaxUserPort](https://docs.microsoft.com/zh-cn/previous-versions/office/exchange-server-analyzer/bb397382(v=exchg.80))
[TcpTimedWaitDelay](https://docs.microsoft.com/zh-cn/previous-versions/office/exchange-server-analyzer/bb397379(v=exchg.80)?redirectedfrom=MSDN)


# TCP 协议端口连接状态
## LISTENING
提供某种服务，侦听远方 TCP 端口的连接请求，当提供的服务没有连接时，处于 LISTENING 状态，端口时开放的，等待连接。

## SYN_SENT(客户端状态)
客户端调用 connect，发送一个 SYN 请求建立一个连接，在发送连接请求后等待匹配的连接请求，此时状态为 SYN_SENT

## SYN_RECEIVED（服务端状态）
在收到和发送一个连接请求后，等待对方对连接请求的确认，当服务器收到客户端发送的同步信号时，将标志位 ACK 和 SYN 置为 1 发送给客户端，此时服务端处于 SYN_RCVD 状态，如果连接成功就变为 ESTABLISED，正常情况下 SYN_RECEIVED 非常短暂

## ESTABLISHED
ESTABLISHED 状态是表示两台机器正在传输数据

## FIN-WAIT-1
等待远程 TCP 连接中断请求，或先前的连接中断请求的确认，主动关闭端应用程序调用 close，TCP 发出 FIN 请求主动关闭连接，之后进入 FIN_WAIT1状态

## FIN-WAIT-2
从远程 TCP 等待连接中断请求，主动关闭端接到 ACK 后，就进入了 FIN-WAIT-2。这是在关闭连接时，客户端和服务端两次握手之后的状态，是著名的半关闭状态，在这个状态下，应用程序还有接受数据的能力，但是已经无法发送数据，但是也有一种可能是，客户端一直处于 FIN-WAIT-2 状态，服务端则一直处于 WAIT_CLOSE 状态，而直到应用层来决定关闭这个状态。

## CLOSE-WAIT
等待从本地用户发来的连接中断请求，被动关闭端 TCP 接到 FIN 后，就发出 ACK 以回应 FIN 请求（它的接收也作为文件结束符传递给上层应用程序），并进入 CLOSE_WAIT

## CLOSING
等待远程 TCP 对连接中断的确认，处于此种状态比较少见

## LAST-ACK
等待原来的发向远程 TCP 的连接中断请求的确认，被动关闭段一段时间后接收到文件结束符的应用程序将调用 CLOSE 关闭连接，TCP 也发送一个 FIN，等待对方的 ACK，进入 LAST-ACK

## TIME-WAIT
在主动关闭段接收到 FIN 后，TCP 就发送 ACK 包，并进入 TIME-WAIT 状态，等待足够的事件以确保 TCP 接收到连接中断请求的确认，很大程度上保证了双方都可以正常结束，但是也存在问题，必须等待 2MSL 时间的过去才能进行下一次连接。

## CLOSED
被动关闭端在接收到 ACK 包后，就进入了closed的状态，连接结束，没有任何连接状态





