---
title: Java I/O 笔记
date: 2022-07-20 17:40:13
tags:
---


# I/O

当应用程序发起 I/O 调用后，会经历两个步骤：

1. 内核等待 I/O 设备准备好数据
2. 内核将数据从内核空间拷贝到用户空间。

# Java 中 3 种常见 IO 模型

- BIO (Blocking I/O)

同步阻塞 I/O 模型，应用程序（线程）发起 read 调用后，会一直阻塞，直到内核把数据拷贝到用户空间。

- NIO (Non-blocking/New I/O)

I/O 多路复用模型。属于同步非阻塞 I/O 模型。应用程序会一直发起 read 调用，检查是否有数据。数据从内核空间拷贝到用户空间的这段时间，线程依然是阻塞。

相比于同步阻塞 I/O，同步非阻塞 I/O 避免了无数据阻塞。但是轮询十分消耗 CPU。


I/O 多路复用模型。线程发起 select 调用，询问内核数据是否准备就绪，等内核数据准备好了，用户线程再发起 read 调用。

Java 中的 NIO ，有一个非常重要的选择器 ( Selector ) 的概念，也可以被称为 多路复用器。通过它，只需要一个线程便可以管理多个客户端连接。当客户端数据到了之后，才会为其服务。

# 一般 IO
- InputStream 所有字节输入流的超类
	- FileInputStream 用于操作文件的读取
    - ByteArrayInputStream
    - FilterInputStream
	    - BufferedInputStream
	    - DataInputStream


&nbsp;

- OutputStream
	- FilterOutputStream
		- DataOutputStream
		- PrintStream
		- BufferedOutputStream

&nbsp;
Java 1.1 对基本 I/O 流类库进行了重大改写。Reader 和 Writer 属于其中的新产物，不过它们的出现并不是取代 InputStream 和 OutputStream，因为 InputStream 和 OutputStream 在面向字节的 I/O 仍然具有价值，Reader 和 Writer 则提供了兼容 Unicode 与面向字符的 I/O。
- Reader
	- InputStreamReader
		- FileReader
- Writer
	- OutputStreamWriter
		- FileWriter
	- PrintWriter


案例 1
```java
public static String getFileContent(String path) {
    StringBuilder result = new StringBuilder();
    try (FileInputStream fileInputStream = new FileInputStream(path);
         InputStreamReader inputStreamReader = new InputStreamReader(fileInputStream, StandardCharsets.UTF_8);
         BufferedReader bufferedReader = new BufferedReader(inputStreamReader);) {
        String readLine = null;
        while ((readLine = bufferedReader.readLine()) != null) {
            result.append(readLine);
        }
    } catch (IOException e) {
        e.printStackTrace();
        return null;
    }
    return result.toString();
}
```

案例 2
```java
public static void putFileContent(String content, String path) {
    try (PrintWriter printWriter = new PrintWriter(new File(path), "GBK");
         BufferedWriter bufferedWriter = new BufferedWriter(printWriter);) {
        bufferedWriter.write(content);
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

# BIO
Blocking IO 的缩写，意思是阻塞型 IO

ServerSocket.accept() 方法会一直阻塞到一个连接建立，返回一个新的 Socket 用于客户端和服务器之间的通信。

如果为每个客户端创建一个新的 Thread，有以下问题: 

1. 任何时候都可能有大量的线程处于休眠状态，只是等待输入或输出数据，造成资源浪费

2. 需要为每个线程的调用栈都分配内存，内存消耗问题

3. 即使忽略内存消耗问题，上下文的切换开销也很大



# NIO
## Channel
FileChannel、DatagramChannel、ServerSocketChannel、SocketChannel

## Buffer
> 关于以下属性的解释可以参考：[https://docs.oracle.com/javase/7/docs/api/java/nio/Buffer.html](https://docs.oracle.com/javase/7/docs/api/java/nio/Buffer.html)


mark: 缓冲区的 mark 是在调用 reset 方法时将其位置重置到的索引。

position: 缓冲区的 position 是下一个要读或写的元素的索引。缓冲区的位置永远不会为负，也永远不会大于其 limit。

limit: 缓冲区的 limit 是不应该读取或写入的第一个元素的索引。缓冲区的 limit 永远不会是负的，也永远不会超过它的 capacity 。

capacity: 缓冲区的 capacity 就是它包含的元素的数量。缓冲区的容量永远不会是负的，也永远不会改变。

具有 0 <= mark <= position <= limit <= capacity 

 

clear(): 使 buffer 为新的读取或者相对 put 操作做好准备，它将设置 limit 为 capacity，并且 position 设置为 0

flip(): 使 buffer 为新的序列写入或者相对 get 操作做好准备，他将设置 limit 为当前的位置，并且设置 position 为 0

rewind(): 使 buffer 为重新读取已经包含的数据做好准备，它将保持 limit 不变，设置 position 为 0


### 只读 Buffer
```java
byteBuffer.asReadOnlyBuffer();
```

### MappedByteBuffer
文件的修改在堆外内存进行

> [https://docs.oracle.com/javase/8/docs/api/java/nio/MappedByteBuffer.html](https://docs.oracle.com/javase/8/docs/api/java/nio/MappedByteBuffer.html)


## Selector
select() 方法是阻塞操作。但可以设置超时时间。

SelectionKey
包含以下 int 类型常量 OP_READ、OP_WRITE、OP_CONNECT、OP_ACCEPT
