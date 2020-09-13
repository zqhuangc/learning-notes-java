https://draveness.me/docker

* 镜像(image)    
Docker 镜像（Image）就是一个只读的模板

* 仓库(repository)    
仓库（Repository）是集中存放镜像文件的场所。  
仓库分为公开仓库（Public）和私有仓库（Private）两种形式。  
**Docker 仓库的概念跟 Git 类似**

* 容器(container)    
Docker 利用容器（Container）来运行应用。容器是从镜像创建的运行实例。它可以被启动、开始、停止、删除。每个容器都是相互隔离的、保证安全的平台。可以把容器看做是一个简易版的 Linux 环境（包括root用户权限、进程空间、用户空间和网络空间等）和运行在其中的应用程序。



#### NaLinux 命名空间、控制组和 UnionFS 三大技术支撑了目前 Docker 的实现
mespaces
Linux 的命名空间解决了进程和网络隔离的问题
进程
网络
Docker 为我们提供了四种不同的网络模式，Host、Container、None 和 Bridge 模式

libnetwork
它提供了一个连接不同容器的实现，同时也能够为应用给出一个能够提供一致的编程接口和网络层抽象的容器网络模型。容器网络模型由以下的几个主要组件组成，分别是 Sandbox、Endpoint 和 Network：

挂载点
文件系统。
如果一个容器需要启动，那么它一定需要提供一个根文件系统（rootfs），容器需要使用这个文件系统来创建一个新的进程，所有二进制的执行都必须在这个根文件系统中。  
为了保证当前的容器进程没有办法访问宿主机器上其他目录，我们在这里还需要通过 libcontainer 提供的 pivot_root 或者 chroot 函数改变进程能够访问个文件目录的根节点。

chroot（change root）

容器和镜像的区别就在于，所有的镜像都是只读的，而每一个容器其实等于镜像加上一个可读写的层，也就是同一个镜像可以对应多个容器。

Control Groups（简称 CGroups）就是能够隔离宿主机器上的物理资源，例如 CPU、内存、磁盘 I/O 和网络带宽。

存储驱动

AUFS

UnionFS 其实是一种为 Linux 操作系统设计的用于把多个文件系统『联合』到同一个挂载点的文件系统服务。而 AUFS 即 Advanced UnionFS 其实就是 UnionFS 的升级版，它能够提供更优秀的性能和效率。


# Docker安装部署
docker安装非常简单，支持各种平台，请到官网自行安装下载docker下载

###Docker常用命令

* 获取镜像

* docker pull
  从仓库获取所需要的镜像。
  使用示例：docker pull centos:centos6

实际上相当于 docker pull registry.hub.docker.com/centos:centos6
命令，即从注册服务器 registry.hub.docker.com 中的 centos 仓库来下载标记为 centos6 的镜像。
有时候官方仓库注册服务器下载较慢，可以从其他仓库下载。 从其它仓库下载时需要指定完整的仓库注册服务器地址。

### 查看镜像列表

* docker images
  列出了所有顶层（top-level）镜像。实际上，在这里我们没有办法区分一个镜像和一个只读层，所以我们提出了top-level镜像。只有创建容器时使用的镜像或者是直接pull下来的镜像能被称为顶层（top-level）镜像，并且每一个顶层镜像下面都隐藏了多个镜像层。

使用示例：

$ docker images

> REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
> centos                   centos6             6a77ab6655b9        8 weeks ago         194.6 MB
> ubuntu                   latest              2fa927b5cdd3        9 weeks ago         122 MB
> 在列出信息中，可以看到几个字段信息

```
REPOSITORY          来自于哪个仓库，比如 ubuntu
TAG                 镜像的标记，比如 14.04
IMAGE ID            它的 ID 号（唯一）
CREATED             创建时间
SIZE                镜像大小
```



### 利用 Dockerfile 来创建镜像

* docker build
  使用 docker commit 来扩展一个镜像比较简单，但是不方便在一个团队中分享。我们可以使用 docker build 来创建一个新的镜像。为此，首先需要创建一个 Dockerfile，包含一些如何创建镜像的指令。新建一个目录和一个 Dockerfile。

mkdir hainiu
cd hainiu
touch Dockerfile
Dockerfile 中每一条指令都创建镜像的一层，例如：

> FROM centos:centos6
>
> MAINTAINER sandywei <sandy@hainiu.tech>
>
> \# move all configuration files into container
>
> RUN yum install -y httpd
>
> EXPOSE 80
>
> CMD ["sh","-c","service httpd start;bash"]

#Dockerfile 基本的语法是

使用 # 来注释
FROM 指令告诉 Docker 使用哪个镜像作为基础，默认从本地
接着是维护者的信息

RUN开头的指令会在创建中运行，比如安装一个软件包，在这里使用yum来安装了一些软件，更详细的语法说明请参考 Dockerfile

### 使用 docker build 来生成镜像

编写完成 Dockerfile 后可以使用 docker build 来生成镜像。

$ docker build -t hainiu/httpd:1.0 .

> Sending build context to Docker daemon 2.048 kB
> Step 1 : FROM centos:centos6
>  ---> 6a77ab6655b9
> Step 2 : MAINTAINER sandywei <sandy@hainiu.tech>
>  ---> Running in 1b26493518a7
>  ---> 8877ee5f7432
> Removing intermediate container 1b26493518a7
> Step 3 : RUN yum install -y httpd
>  ---> Running in fe5b6f1ef888
> .....
> Step 5 : CMD sh -c service httpd start
>  ---> Running in b2b94c1601c2
>  ---> 5f9aa91b0c9e
> Removing intermediate container b2b94c1601c2
> Successfully built 5f9aa91b0c9e
> 其中 -t 标记来添加 tag，指定新的镜像的用户信息。 “.” 是 Dockerfile 所在的路径（当前目录），
> 也可以替换为一个具体的 Dockerfile 的路径。注意一个镜像不能超过 127 层。

###用docker images 查看镜像列表

$ docker images

> REPOSITORY                 TAG               IMAGE ID            CREATED             SIZE
> hainiu/httpd               1.0               5f9aa91b0c9e        3 minutes ago       292.4 MB
> centos                   centos6             6a77ab6655b9        8 weeks ago         194.6 MB
> ubuntu                   latest              2fa927b5cdd3        9 weeks ago         122 MB
> 细心的朋友可以看到最后一层的ID（5f9aa91b0c9e）和 image id 是一样的

#### 上传镜像

* docker push

用户可以通过 docker push 命令，把自己创建的镜像上传到仓库中来共享。例如，用户在 Docker Hub 上完成注册后，可以推送自己的镜像到仓库中。

运行实例：

$ docker push hainiu/httpd:1.0

#### 创建容器

* docker create <image-id>

docker create 命令为指定的镜像（image）添加了一个可读写层，构成了一个新的容器。注意，这个容器并没有运行。

docker create 命令提供了许多参数选项可以指定名字，硬件资源，网络配置等等。

运行示例：

创建一个centos的容器，可以使用仓库＋标签的名字确定image，也可以使用image－id指定image。返回容器id

#### 查看本地images列表
$ docker images

##### 用仓库＋标签
$ docker create -it --name centos6_container centos:centos6

##### 使用image－id

$ docker create -it --name centos6_container 6a77ab6655b9 bash
b3cd0b47fe3db0115037c5e9cf776914bd46944d1ac63c0b753a9df6944c7a67

#### 查看容器列表

可以使用 docker ps查看一件存在的容器列表,不加参数默认只显示当前运行的容器

$ docker ps -a
可以使用 -v 参数将本地目录挂载到容器中。

$ docker create -it --name centos6_container -v /src/webapp:/opt/webapp centos:centos6

--name 指定容器名字

这个功能在进行测试的时候十分方便，比如用户可以放置一些程序到本地目录中，来查看容器是否正常工作。本地目录的路径必须是绝对路径，如果目录不存在 Docker 会自动为你创建它。

### 启动容器

* docker start <container-id>
  Docker start命令为容器文件系统创建了一个进程隔离空间。注意，每一个容器只能够有一个进程隔离空间。



#### 通过名字启动

$ docker start -i centos6_container

#### 通过容器ID启动
$ docker start -i b3cd0b47fe3d
进入容器

* docker exec <container-id>
  在当前容器中执行新命令，如果增加 -it参数运行bash 就和登录到容器效果一样的。

* docker exec -it centos6_container bash
  停止容器#

* docker stop <container-id>
  删除容器#

* docker rm <container-id>
  运行容器

* docker run <image-id>
  docker run就是docker create和docker start两个命令的组合,支持参数也是一致的，如果指定容器名字时，容器已经存在会报错,可以增加 --rm 参数实现容器退出时自动删除。

运行示例:

docker create -it --rm --name centos6_container centos:centos6

#### 查看容器列表

* docker ps

docker ps 命令会列出所有运行中的容器。这隐藏了非运行态容器的存在，如果想要找出这些容器，增加 -a 参数。

#### 删除镜像

docker rmi <image-id>
删除构成镜像的一个只读层。你只能够使用docker rmi来移除最顶层（top level layer）
（也可以说是镜像），你也可以使用-f参数来强制删除中间的只读层。

#### commit容器

docker commit <container-id>
将容器的可读写层转换为一个只读层，这样就把一个容器转换成了不可变的镜像。

镜像保存#

docker save <image-id>
创建一个镜像的压缩文件，这个文件能够在另外一个主机的Docker上使用。和export命令不同，这个命令
为每一个层都保存了它们的元数据。这个命令只能对镜像生效。

使用示例:

* 保存centos镜像为centos_images.tar 文件

$ docker save  -o centos_images.tar centos:centos6

* 或者直接重定向

$ docker save  -o centos_images.tar centos:centos6 > centos_images.tar
容器导出

docker export <container-id>
创建一个tar文件，并且移除了元数据和不必要的层，将多个层整合成了一个层，只保存了当前统一视角看到
的内容。expoxt后的容器再import到Docker中，只有一个容器当前状态的镜像；而save后的镜像则不同，
它能够看到这个镜像的历史镜像。

#### inspect

docker inspect <container-id> or <image-id>
docker inspect命令会提取出容器或者镜像最顶层的元数据











dockerfile：

From 默认从本地



build