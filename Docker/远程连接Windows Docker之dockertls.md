## 前言
曾经需要完成一个SSH到Windows docker container里的需求（这个需求听上去就很神奇有木有），就对Windows docker研究了一段时间，当时查阅了很多资料，感觉没有看到很系统总结的文章，所以决定在这里自己总结下。先后尝试了两种远程连接到Windows docker的方法，一种是通过dockertls远程连接到你要连接的docker host（docker daemon）上；另一种是通过SSH远程连接到特定的docker container中。本文我们讲讲dockertsl,下一篇文章我们将SSH的方式。

## 通过dockertls远程连接Windows docker host
这个方式比较简单直接，如果你有remote Windows docker的需求，推荐用这个方式，毕竟这也是Azure的官方方法：
准备两台都装有docker的机器，一台是你需要连接的host，一台我们就算普通machine
1. 通过[dockertls](https://hub.docker.com/r/stefanscherer/dockertls-windows/)生成证书，这里会需要用到你的host的IP地址，最好先整一个静态IP，不然IP一换，你就又要重新生成一个新的证书了。
+ 在host机器上找一个你喜欢的folder，执行以下代码：
``` powershell
mkdir server
mkdir client\.docker
docker run --rm `
  -e SERVER_NAME=$(hostname) `
  -e IP_ADDRESSES=127.0.0.1,<your IP address> `
  -v "$(pwd)\server:c:\programdata\docker" `
  -v "$(pwd)\client\.docker:c:\users\containeradministrator\.docker" stefanscherer/dockertls-windows
dir server\certs.d
dir server\config
dir client\.docker
```  
然后你就会在相应的目录里看到生成的证书啦

+ 把生成的证书copy到你要用来远程登录的machine上，放在你的user目录下，比如说C:\user\yiran\.docker
  
2. 修改docker的daemon.json文件，加上：
``` json
{
    "tlsverify":  false,
}
```
3. 保证2375和2376这两个TLS的端口允许访问
~~~
New-NetFirewallRule -DisplayName 'Docker SSL Inbound' -Profile @('Domain', 'Public', 'Private') -Direction Inbound -Action Allow -Protocol TCP -LocalPort 2376
~~~
4. 重新启动docker service，用管理员权限在powershell输入**Restart-Service Docker**

然后就可以来测试看看了，在你的machine上面执行：
```
docker -H tcp://wsdockerhost.southcentralus.cloudapp.azure.com:2376 --tlsverify=0 version
```
不出意外你就可以在上面敲打docker命令来操作host机器上的docker container了。

## 参考
https://docs.microsoft.com/en-us/virtualization/windowscontainers/management/manage_remotehost
https://hub.docker.com/r/stefanscherer/dockertls-windows/