## K8s 介绍
全新的基于容器技术的分布式架构领先方案。

Kubernetes（k8s）是自动化容器操作的开源平台，这些操作包括部署，调度和节点集群间扩展

master
Node
Pod


使用Kubernetes可以： 
1. 自动化容器的部署和复制 
2. 随时扩展或收缩容器规模 
3. 将容器组织成组，并且提供容器间的负载均衡 
4. 很容易地升级应用程序容器的新版本 
4. 提供容器弹性，如果容器失效就替换它，等等…

Kubernetes解决的问题： 
1. 调度 - 容器应该在哪个机器上运行 
2. 生命周期和健康状况 - 容器在无错的条件下运行 
3. 服务发现 - 容器在哪，怎样与它通信 
4. 监控 - 容器是否运行正常 
5. 认证 - 谁能访问容器 
6. 容器聚合 - 如何将多个容器合并成一个工程

Kubernetes组件组成： 
1. kubectl  

客户端命令行工具，将接受的命令格式化后发送给kube-apiserver，作为整个系统的操作入口。

2. kube-apiserver  

作为整个系统的控制入口，以REST API服务提供接口。 

3. kube-controller-manager  

用来执行整个系统中的后台任务，包括节点状态状况、Pod个数、Pods和Service的关联等。

4. kube-scheduler  

负责节点资源管理，接受来自kube-apiserver创建Pods任务，并分配到某个节点。

5. etcd  

负责节点间的服务发现和配置共享。

6. kube-proxy  

运行在每个计算节点上，负责Pod网络代理。定时从etcd获取到service信息来做相应的策略。 

7. kubelet   

运行在每个计算节点上，作为agent，接受分配该节点的Pods任务及管理容器，周期性获取容器状态，反馈给kube-apiserver。 

8. DNS   

一个可选的DNS服务，用于为每个Service对象创建DNS记录，这样所有的Pod就可以通过DNS访问服务了。