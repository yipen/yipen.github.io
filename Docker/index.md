# Docker on Linux VS Windows

## Docker的安装
Docker 可以被安装在win10或者windows server2016及更高版本，具体安装方法可参考：
https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-docker/configure-docker-daemon
### 可能问题：
- Get-NetNat | Remove-NetNat
目前windows的NAT虚拟网络只能支持一个，因此在开始创建NAT网络之前需要先检查是否已存在NAT网络。</br>
- Docker Client版本不匹配：</br>
Cmd: set `DOCKER_API_VERSION=1.39` </br>
Powershell: $env:DOCKER_API_VERSION=1.39 </br>
这种一般发生在host已经安装过另一个版本的docker，并设置了docker client的version

## Docker 在windows和Linux的network driver比较
通过Docker network ls可以列出安装Docker时创建的网络，通过—network可以指定容器的网络连接类型。
### Linux：
- Bridge：默认的网络驱动。如果你的应用们都跑在独立的container上并且需要相互通信，可以使用这个类型。</br>
- Host：与host公用一个网络设置，这种类型，可以提高container网络带宽。</br>
- Container: 新创建的容器不会创建自己的网卡，配置自己的IP，而是和一个指定的容器共享IP、端口范围等。同样，两个容器除了网络方面，其他的如文件系统、进程列表等还是隔离的。两个容器的进程可以通过lo网卡设备通信。</br>
- None：这种设置的container通常用于和自定义的网络驱动结合。</br>
- Overlay: overlay的网络可以连接多个Docker daemons并且让swarm service可以互相通信。也可以通过overlay网络让一个swarm service和一个container通信，或者两个在不同的Docker daemon的container通信。</br>
- Macvlan: 会给container分配一个MAC地址，让container在你的网络中作为一个实体设备。Docker dameon会把流量通过MAC地址route到container。</br>
</br>

### Windows
Hyper-v是什么？hyper-v不就是一个虚拟机的技术吗？对，其实它有点像虚拟机，但是hyper－v的技术略有不同，速度会明显比虚拟机快很多，只是在申请资源或者获取资源时，比Windows server Container的速度稍稍慢一点点，Windows server container可能3秒，它可能4、5秒。但是资源的隔离度比较好一些，类似于虚拟机。</br>
- NAT: 起一个本地或者是数据本地的IP地址，如果你想外网访问的话，把Docker映射出来，这种方式比较适合做一个JOB类型的应用在上面，不需要外边可以访问它，但是容器里面可以去下载东西。</br>
- Transparent: 这种模型是现在主要用于生产的模型，首先它是通过mac 地址伪装实现数据透传，对网络的性能本身折损也比较少，它也支持把多个工作网卡绑到一个交换机上，然后把这个交换机给容器用。网络模型在容器宿主机以外的机器上看到Windows容器和一台物理机没有什么区别。</br>
- Overlay: Docker engine 在swarm模式下运行，container可以和其他绑定在同一个overlay网络的containers通信。Each overlay network that is created on a Swarm cluster is created with its own IP subnet, defined by a private IP prefix. The overlay network driver uses VXLAN encapsulation.</br>
- L2bridge: 采用openstack网络的Flat模式，所用的网络跟宿主机的网络是一样的，和宿主机在同一个网段，这样有很大的局限性，网络和宿主机混在一起没有办法做到多租户隔离，然后网段用光，适用于比较小的集群。</br>
- L2tunnel: Similar to l2bridge, however this driver should only be used in a Microsoft Cloud Stack. Packets coming from a container are sent to the virtualization host where SDN policy is applied.</br>
 
## Docker/Container 远程连接
### Linux:
SSH连接container，host的某个端口和container的端口mapping（SSH一般是22），通过公钥，SSH到端口即可。</br>
</br>
### Windows:
- SSH连接，需要有个SSH windows的image，其他的类似。</br>
- TLS 连接到Docker：</br>
	https://docs.microsoft.com/en-us/virtualization/windowscontainers/management/manage_remotehost</br>
	通过dockertls生成证书到，copy一份到client端，就可以从一台机器的docker remote到另一个Docker上。</br>
 
## 参考
网络详细可以参考：http://www.ywnds.com/?p=4714 </br>
Windows container：http://blog.shurenyun.com/shurenyun-docker-224/ </br>

