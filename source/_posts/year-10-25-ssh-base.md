---
title: 使用SSH进行远程连接
date: 2016-10-25 12:59:20
tags:
 - Network
---
![cover](http://oanr6klwj.bkt.clouddn.com/blog/ssh-base-cover.png)
> [*pixiv-ID: 59600727*](http://www.pixiv.net/member_illust.php?mode=medium&illust_id=59600727)

在开发场景中，开发人员免不了和服务器打交道，无论是连接服务器还是自己虚拟机中搭建的开发环境，都离不开SSH的应用。

SSH是一种网络协议，用于计算机之间的加密登陆。通过公钥加密进行信息保护，过程为：

1. 远程主机收到用户登陆请求，发送公钥到用户。
2. 用户用公钥加密密码发送到远程主机。
3. 远程主机用私钥解密登录密码，密码正确则通过验证。

<!--more-->

如果通信被截获，内容也是加密过的，而不会直接暴露出来，因此一直是主流的网络安全解决方案之一。SSH有很多种实现，我们下文所说的OpenSSH就是其中一种。

对于linux，一般使用的都是OpenSSH，对于尚未安装相应软件的机器，需要通过apt-get进行安装：
`$ sudo apt-get install openssh-server openssh-client`

用SSH远程登陆时，通用的命令是(方括号中的内容是可选的)
`$ ssh [-p port] user@host`
- user代表远程所使用的登陆名(需被远程机器上有此用户名)
- host代表远程机器的地址
- 默认会使用22作为远程主机的端口，如果你想改变端口，可以使用p参数

如果想要退出登陆，只需要输入exit就能退出了。

那么能不能不用每次输入密码登陆呢？当然可以，因为SSH支持公钥登陆。

> 首先简单讲讲公钥验证机制：公钥(public key)和私钥(private key)是两份文件，服务端持有公钥，用来加密；客户端持有私钥，用来解密。客户端向服务端发起连接请求时，服务端会生成一串随机数，经过公钥加密后传给客户端。这时客户端会用私钥解密获取随机数，再返回服务端。最后服务端判断一下，如果客户端能够返回正确的随机数，就认为校验通过了 ，可以进行连接。否则就不能连接。

用户需要提供自己的公钥，如果没有，可以生成一个，使用命令
`$ ssh-keygen`
这个命令会在默认为`$HOME/.ssh/`目录下生成两个文件，一个是`id_rsa.pub`(公钥)，另一个是`id_rsa`(私钥)，生成私钥的时候可能会询问你需要不需要设置口令(passphrase)，如果担心它的安全，可以设置一个。

接着你就可以传送公钥到想要访问的主机上面：
`$ ssh-copy-id user@host`

从此登陆这台主机，就不需要密码了。
需注意，主机的`/etc/ssh/sshd_config`文件中需存在以下命令。
```shell
RSAAuthentication yes
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
```

有时需要在本地和远程机器间传送文件，我们可以使用scp命令来进行。

- 上传文件到远程目录：
`$ scp /path/file user@host:/path`
- 从远程目录下载文件：
`$ scp user@host:/path/file /path`
- 上传文件夹到远程目录：
`$ scp -r /path/dir user@host:/path`
- 从远程目录下载文件夹：
`$ scp -r user@host:/path/dir /path`

最后谈谈SSH的局限性：

SSH存在一个风险，就是SSH的协议公钥是没有证书公证的，也就是说，可能存在**“中间人”**截获了登陆请求，并冒充远程主机将自己伪造的公钥发送给用户，就能得到用户接下来发送的密码。这种攻击叫做**中间人攻击**。

SSH解决的办法是在登陆时发送公钥指纹给用户，用户自己根据这一指纹和远程主机公布的指纹进行核对。用户也要自行承担风险。
