## 环境准备
1. win10 host
2. Hyper-v和Containers这两个windows feature可用
3. docker desktop
4. minkicube

## Docker desktop安装
首先[下载](https://docs.docker.com/desktop/windows/install/)Docker Desktop安装包，下载完成后直接双击installer.exe进行安装。</br>
安装完成之后你可以启动Docker desktop, 它是支持windows container和Linux container的切换的，只需要启动之后，右击图标就可以进行选择。</br>
<img src="../_image/switch.PNG" alt="菜单栏" width="300" height="350" /></br>
上图中显示的，switch to Linux container,选择之后就会切换到Linux container的模式，反之就是Windows container模式。</br>


## minikube install
https://minikube.sigs.k8s.io/docs/start/
按照官网手册，选择windows方式下载安装

启动minikube后，你可以查找docker container,找到一个gcr.io/k8s-minikube/kicbase的image启动的container。你可以通过docker命令进入到container里面进行环境查询。