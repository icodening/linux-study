# 构建编译环境
````
基础环境
1. macOS 10.15.7
2. CPU x86 Intel i7
3. Docker version 20.10.14, build a224086
````

## 构建流程

1. Docker 拉取 ``ubuntu`` 镜像.  [官方镜像仓库](https://hub.docker.com/_/ubuntu?tab=tags)
````bash
docker pull ubuntu:20.04
````

2. 运行 ``ubuntu`` 镜像. (暴露22端口是方便后续支持SSH连接方便操作)
````bash
docker run -itd --name ubuntu2004 -p 50022:22 imageId
````

3. 进入刚才创建的``ubuntu``容器
````bash
docker exec -it ubuntu2004 /bin/bash
````

4. 更新``apt``源
````bash
apt-get update
````

5. 安装``vim``
````bash
apt install vim
````

6. 修改``apt``镜像源.   [镜像源地址](https://mirror.tuna.tsinghua.edu.cn/help/ubuntu/)
````bash
vim /etc/apt/sources.list
````
```text
# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-updates main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-backports main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-security main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-security main restricted universe multiverse

# 预发布软件源，不建议启用
# deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-proposed main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-proposed main restricted universe multiverse
```
7. 安装必备软件
````bash
apt install libncurses-dev bison flex libssl-dev libelf-dev build-essential bc make gcc
````

8. 下载``Linux Kernel``源码. [Linux Kernel下载地址](https://kernel.org)  
 我这里选择的是 ``5.18`` 版本
````bash
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.18.tar.xz
````
9. 解压源码
````bash
tar -xvf linux-5.18.tar.xz
````

10. 配置内核参数
````bash
cd linux-5.18
make menuconfig
````

11. 编译内核  
````bash
make -j4 bzImage
````
