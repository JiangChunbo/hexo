---
title: SMB 协议对接
date: 2022-09-14 15:53:39
tags:
---


```java
<dependency>
    <groupId>com.hierynomus</groupId>
    <artifactId>smbj</artifactId>
    <version>0.11.5</version>
</dependency>
```

SMB 关于 Windows 的共享文件夹

如果开启了密码保护需要提供**用户账户**和**密码**

计算机处于睡眠状态时无法访问共享目录


非递归访问:

```java
final String IP = "127.0.0.1";
final String USERNAME = "";
final String PASSWORD = "";
final String SHARE_NAME = "";

SMBClient client = new SMBClient();
Connection connection = null;
AuthenticationContext ac = null;
Session session = null;
try {
    connection = client.connect(IP);
    ac = new AuthenticationContext(USERNAME, PASSWORD.toCharArray(), null);
    session = connection.authenticate(ac);
} catch (Exception e) {
    throw new JiashiboException("SMB 建立连接失败");
}

if (session == null) {
    throw new JiashiboException("无法访问共享文件夹 " + SHARE_NAME);
}
Share share = session.connectShare(SHARE_NAME);
if (!(share instanceof DiskShare)) {
    throw new JiashiboException("无法访问共享文件夹 " + SHARE_NAME);
}
DiskShare diskShare = (DiskShare) share;
List<String> fileNames = diskShare.list("").stream().map(FileIdBothDirectoryInformation::getFileName).collect(Collectors.toList());

System.out.println(fileNames);
```



打开一个文件:

```java
File smbFileRead = dirShare.openFile(
    fileName, // 路径
    EnumSet.of(AccessMask.GENERIC_READ), // Set<AccessMask>
    null, // Set<FileAttributes>
    SMB2ShareAccess.ALL, // Set<SMB2ShareAccess>
    SMB2CreateDisposition.FILE_OPEN, // SMB2CreateDisposition
    null); // Set<SMB2CreateOptions>
```