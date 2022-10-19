---
title: SMB 协议对接
date: 2022-09-14 15:53:39
tags:
---


# 参考引用

[https://github.com/hierynomus/smbj](https://github.com/hierynomus/smbj)



fileAttributes，包含文件属性掩码
https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-fscc/ca28ec38-f155-4768-81d6-4bfeb8586fc9


# 命令行使用


- 安装 smbclient 工具

```bash
apt -y --force-yes install smbclient
```

- 列出所有的共享名称

```bash
smbclient -L //<HOSTNAME> -U <USERNAME>%<PASSWORD>
```

- 进入特定共享名



# smbclient 命令

- 查看当前文件夹

```bash
smb: \> ls
```

- 创建一个目录

```bash
smb: \> mkdir directory-name
```

- 进入目录
```bash
smb: \> cd DMS_Input_DistAndShop_CIB
```

```bash
yum install samba-client
```


- 下载

```bash
get xx
mget xx
mget xx
```

- 上传

关闭提示，这样可以在选择多个文件上传、下载的时候不会提示是否确认的信息。

```bash
prompt off
```

```bash
mput abc.txt
mput abc*.txt
```

## 其他


默认共享是系统安装完毕后就自动开启的共享,也叫管理共享,常被管理员用于远程管理计算机。在 Windows 2000/XP 及更高级的版本中,默认开启的共享有“c$”、“d$”等所有的逻辑盘以及“admin$”、“ipc$”,这些共享都有“$”标志,意为隐含的。

假设你的计算机名为 CannedBread，在运行对话框输入: `\\CannedBread\C$` 即可访问

## 命令

连接目标 Windows 服务器共享目录（Windows安装完毕会开启驱动器的隐藏共享，需要管理员权限，访问路径是：盘符$)

```bash
smbclient //192.168.199.196/share -U Administrators/JiangChunbo
```

# Java 使用

maven

```xml
<dependency>
    <groupId>com.hierynomus</groupId>
    <artifactId>smbj</artifactId>
    <version>0.11.5</version>
</dependency>
```

SMB 关于 Windows 的共享文件夹

如果开启了**密码保护**需要提供**账户**和**密码**

计算机处于睡眠状态时无法访问共享目录

如果为 EveryOne 开启了访问权限，即使不存在的**账户**，也可以访问，此时密码应该被忽略

如果**账户**是系统存在的，则会进行**密码**认证，失败则不通过。


## 获得一个 SMBClient

无参构造器:

```java
SMBClient client = new SMBClient();
```

自定义配置:

```java
SmbConfig config = SmbConfig.builder()
        .withTimeout(120, TimeUnit.SECONDS) // Timeout sets Read, Write, and Transact timeouts (default is 60 seconds)
        .withSoTimeout(180, TimeUnit.SECONDS) // Socket Timeout (default is 0 seconds, blocks forever)
        .build();
SMBClient client = new SMBClient(config);
```


## 获得一个 Connection

```java
// 注意关闭
try (Connection connection = client.connect("127.0.0.1")) {

} catch (IOException e) {
    // 可能 client connect 超时
    throw new RuntimeException(e.getMessage(), e);
}
```


## 获得一个 Session


获得一个会话示例代码:

```java
Session session = connection.authenticate(
    new AuthenticationContext("USERNAME", "PASSWORD".toCharArray(), null));
```

## 获得 DiskShare

```java
// Connect to Share
try (DiskShare share = (DiskShare) session.connectShare("shareDirectory")) {
    for (FileIdBothDirectoryInformation f : share.list("data", "*.csv")) {
        System.out.println("File : " + f.getFileName());
    }
}
```

## 整体 Demo

获得共享名称为 shareDir 下的所有文件名信息，并输出: 

```java
/**
    * 1. 使用 try 语句块自动关闭资源
    * 2. client.connect 参数 hostname 可能导致抛出异常
    * 3. connection.authenticate 认证失败可能导致抛出异常
    * 4. session.connectShare shareName 不存在可能导致抛出异常
    * 5. share.list 传入的 path 错误可能导致抛出异常
    */
try (SMBClient client = new SMBClient();
        Connection connection = client.connect("127.0.0.1");
        Session session = connection.authenticate(new AuthenticationContext("USERNAME", "PASSWORD".toCharArray(), null));
        DiskShare share = (DiskShare) session.connectShare("shareDir")) {
    for (FileIdBothDirectoryInformation f : share.list("", "*.csv")) {
        System.out.println("File : " + f.getFileName());
    }
} catch (Exception e) {
    throw new RuntimeException(e);
}
```


## `com.hierynomus.smbj.share.File`

- 打开一个文件（读）

```java
File smbFileRead = dirShare.openFile(
    fileName, // 路径
    EnumSet.of(AccessMask.GENERIC_READ), // Set<AccessMask>
    null, // Set<FileAttributes>
    SMB2ShareAccess.ALL, // Set<SMB2ShareAccess>
    SMB2CreateDisposition.FILE_OPEN, // SMB2CreateDisposition
    null); // Set<SMB2CreateOptions>
```


SMB2CreateDisposition: 如果传递 `FILE_OPEN`，那么文件找不到的情况下会抛出异常；如果设置为 `FILE_OPEN_IF`，那么文件找不到的情况下会静默继续


- 远程拷贝

就是直接在远程机器上将一个文件拷贝到另一个文件


```java
// 确保拷贝到的文件夹存在
this.ensureDirectoryExists(diskShare, dir);
com.hierynomus.smbj.share.File backupShareFile = diskShare.openFile(
        "copy-directory\\" + filename,
        EnumSet.of(AccessMask.GENERIC_ALL), // Set<AccessMask>
        null, // Set<FileAttributes>
        SMB2ShareAccess.ALL, // Set<SMB2ShareAccess>
        SMB2CreateDisposition.FILE_CREATE, // SMB2CreateDisposition
        null);
smbFile.remoteCopyTo(backupShareFile);
```


- 如何判断一个 `File` 是否是文件夹

```java
// FileIdBothDirectoryInformation
boolean isDirectory = (f.getFileAttributes() & 0x10) == 0x10;

// File
boolean directory = shareFile.getFileInformation().getStandardInformation().isDirectory();
```



## 案例



将共享文件夹 `\\127.0.0.1\shareDirectory` 下的所有文件都转移到 `\\127.0.0.1\shareDirectory\backup` 文件夹下

```java
public static void transferAllFiles(String path) {
    try (SMBClient client = new SMBClient();
            Connection connection = client.connect("127.0.0.1");
            Session session = connection.authenticate(new AuthenticationContext("USERNAME", "PASSWORD".toCharArray(), null));
            DiskShare share = (DiskShare) session.connectShare("shareDirectory")) {
        List<FileIdBothDirectoryInformation> fileIdBothDirectoryInformationList = share.list(path, "*");
        fileIdBothDirectoryInformationList = fileIdBothDirectoryInformationList.stream().filter(fileIdBothDirectoryInformation -> {
            // 过滤掉文件夹
            return (fileIdBothDirectoryInformation.getFileAttributes() & 0x10) == 0;
        }).collect(Collectors.toList());

        for (FileIdBothDirectoryInformation f : fileIdBothDirectoryInformationList) {
            com.hierynomus.smbj.share.File shareFile = share.openFile(
                    path + "\\" + f.getFileName(), // 路径
                    EnumSet.of(AccessMask.GENERIC_ALL), // Set<AccessMask>
                    null, // Set<FileAttributes>
                    SMB2ShareAccess.ALL, // Set<SMB2ShareAccess>
                    SMB2CreateDisposition.FILE_OPEN, // SMB2CreateDisposition
                    null); // Set<SMB2CreateOptions>
            com.hierynomus.smbj.share.File backupShareFile = share.openFile(
                    path + "\\backup\\" + f.getFileName(),
                    EnumSet.of(AccessMask.GENERIC_ALL), // Set<AccessMask>
                    null, // Set<FileAttributes>
                    SMB2ShareAccess.ALL, // Set<SMB2ShareAccess>
                    SMB2CreateDisposition.FILE_CREATE, // SMB2CreateDisposition
                    null);
            shareFile.remoteCopyTo(backupShareFile);
            // 最后要删掉
            shareFile.deleteOnClose();
        }
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}
```