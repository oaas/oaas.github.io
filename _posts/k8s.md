#第一式： 打造自己的Kubernetes

## 目标：
* 了解Kubernetes架构
* 构建Kubernetes环境
* 构建数据仓库
* 创建Overlay网络
* 配置Kubernetes master节点
* 配置kubernetes node节点
* 在Kubernetes中运行你的第一个容器

## 介绍：
> Kubernetes是一个开源的系统，由Google开发和开源，轻量级, 用来自动部署，调度，扩展，和管理大规模的容器；

## 构成:
* Kubernetes Master
* Kubernetes Nodes
* Etcd
* Overlay Network(flannel)

#####  整体架构图如下：
![](https://github.com/oaas/kubernetes/blob/master/chapter1/arch.jpg?raw=true)

##### 如上图所示：
* Kubernetes master通过http或https的方式连接到etcd去存取数据，同时也连接到flannel去访问容器应用；
* Kubernetes nodes通过http或https的方式连接到Kubernetes master报告自身的状态和获取指令，调度容器；
* Kubernetes nodes通过overlay network(flannel)将所有的容器连接到一块，构成一个集群；

## Kubernetes master
> Kubernetes集群中主要的组件之一提供以下主要的功能

* 授权和认证
* 提供API功能
* 调度容器到nodes节点
* 扩展和复制Controller
* 读取和存储集群配置信息
* 提供命令行接口

#####Kubernetes master又有几个组件构成, 如下图； 
![](https://github.com/oaas/kubernetes/blob/master/chapter1/k8s_master.png?raw=true)

* **kube-apiserver**: 提供基于http/https的RESTful API的访问，是kubernets组件的核心，比如kubectl, scheduler, replication controller, etcd, kubelet, kube-proxy。
* **kube-scheduler**: 基于配置和调度算法，调度容器到适配的node上,比如根据node的cpu,memeory等。
* **kube-controller-manager**: 执行集群的操作，管理node节点，创建或更新集群信息，更改当前的状态到期望的状态。
* **kubectl**: 命令行工具，用于管理集群。

## Kubernetes node
> Worker节点，容器真正运行的地方，被kubernetes master控制。

#####Kubernetes slave又有几个组件构成, 如下图；
![](https://github.com/oaas/kubernetes/blob/master/chapter1/k8s_slave.png?raw=true)

* **Kubelet**: 是node节点上最主要的进程，用来和master沟通，完成以下的操作；
  * 周期性的访问API Controller获取配置和报告自己的信息
  * 执行容器操作
  * 运行http服务器提供简单的API功能

* **Proxy(kube-proxy)**: 
  * 负责流量转发，负载均衡，通过修改iptables的规则路由对应的流量到对应的容器上；
  * iptables -t nat -S: 查看转发规则；

* **Etcd**: 分布式key-value数据库，可以通过RESTful API去执行增删改查(CRUD)操作；
  * 获取kubernetes配置和状态信息: **curl -L "http://{Etcd_Server_Ip}:{Port}/v2/keys/registry"**
* **Overlay Network**: 容器之间的网络交流是非常复杂的部分，因为启动docker的时候，docker会自动分配一个动态的ip地址给容器，如果docker是在一台主机上面，通过Docker Link这种方式是没有问题的，但是在一个集群上，就会面临Ip地址冲突的问题，kubernetesyou很多种解决方式，但是通过Flannel是相对来说比较简单的解决方式；
  * Flannel也是通过Etcd去配置和存储状态信息;
  * 获取配置信息: **curl -L "http://{Etcd_Server_Ip}:{Port}/v2/keys/coreos.com/network/config"** 
  * 获取子网信息: **curl -L "http://{Etcd_Server_Ip}:{Port}/v2/keys/coreos.com/network/subnets"** 
  

  
## 理论这么多，下面来实操一把：
1.  安装Etcd
2.  创建Overlay network
3.  配置master
4.  配置slave

##### 需求:
* Docker需求系统内核版本至少在3.1以上，因此最好选择CentOS 7或Ubuntu 14/16
* Iptables版本在1.4.11以上
* Docker版本1.6.2+以上

##### 我的实验环境：
```
    name               ip              services
srv-k8s-master1    10.1.96.232       master所有组件, etcd
srv-k8s-node1      10.1.96.233       node所有组件
```

##### 在srv-k8s-master1，即master节点操作：
 1. 安装epel镜像

   ```
    rpm -ivh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
   ```
   
 2. 安装etcd
 
   ```
   [root@srv-k8s-master1 ~]# yum install etcd      # 安装etcd
   [root@srv-k8s-master1 ~]# cat /etc/etcd/etcd.conf  | grep -v ^#    #更改etcd配置文件如下
	ETCD_NAME=default      ＃ etcd 实例名字 
	ETCD_DATA_DIR="/var/lib/etcd/default.etcd"    ＃ etcd数据存储目录
	ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"    ＃ 监听的端口, 改为0.0.0.0，以至于其他node可以对其CRUD
	ETCD_ADVERTISE_CLIENT_URLS="http://localhost:2379"  # etcd集群配置，暂时先不做更改
   [root@srv-k8s-master1 ~]# systemctl start etcd      # 启动etcd进程
   [root@srv-k8s-master1 ~]# systemctl enable etcd      # 
   [root@srv-k8s-master1 ~]# netstat -nlpt      ＃ 检查是否已经成功启动etcd
   ```
   
 3. 测试etcd

   ```
	[root@srv-k8s-master1 ~]# etcdctl  set name "lambert"      # 添加key为name，value的lambert的数据
	lambert
	[root@srv-k8s-master1 ~]# etcdctl get name       ＃ 获取key为name的值
	lambert
	
	[root@srv-k8s-master1 ~]# curl http://localhost:2379/v2/keys/favorite -d value="panda"       # 通过RESTful API设置key为favorite, value为panda
	{"action":"create","node":{"key":"/favorite/00000000000000000006","value":"panda","modifiedIndex":6,"createdIndex":6}}
	[root@srv-k8s-master1 ~]# curl -L http://localhost:2379/v2/keys/favorite      ＃ 获取favorite的值
	{"action":"get","node":{"key":"/favorite","dir":true,"nodes":[{"key":"/favorite/00000000000000000006","value":"panda","modifiedIndex":6,"createdIndex":6}],"modifiedIndex":	6,"createdIndex":6}}
	
	[root@srv-k8s-master1 ~]# curl http://localhost:2379/v2/keys/favorite?recursive=true -XDELETE     ＃删除favorite
{"action":"delete","node":{"key":"/favorite","dir":true,"modifiedIndex":7,"createdIndex":6},"prevNode":{"key":"/favorite","dir":true,"modifiedIndex":6,"createdIndex":6}}
[root@srv-k8s-master1 ~]# curl -L http://localhost:2379/v2/keys/favorite    #再次获取 Key not found
{"errorCode":100,"message":"Key not found","cause":"/favorite","index":7}
   ```  
 
4. 创建flannel网段信息；

   ```
   [root@srv-k8s-master1 ~]# etcdctl set /coreos.com/network/config '{ "Network": "192.168.0.0/16" }'
   { "Network": "192.168.0.0/16" }
   ```   
   
5. 创建Overlay Network
   > Kubernetes借助overlay network可以使不同node上的容器相互通信。pod是kubernetes调度的最小单元，在共享的网络命名空间中(networking namespace)中，kubernetes给每一个要创建的pod分配一个ip地址，因此不同的node上的不同pod之间就可以相互通信，pod内的容器之间可以通过localhost+port的方式之间相互交流，传递信息。有很多中实现共享网络的方法，但是最简单的方式是使用flannel.
   1. Flannel给每一个node都分配一个子网网段。
   2. Flannel通过etcd存储ip映射信息.
   3. 有很多中转发网络数据包的方法，但是最简单的方法是通过TUN Device将IP包分装在UDP包中进行转发，默认的UDP端口是8285.
   4. Flanneld是Flannel的监听进程，用来监听etcd中的信息，分配子网到node节点，路由数据包；
   
   ```
   [root@srv-k8s-master1 ~]# yum -y install flannel
   ```
   
6. 配置Flannel，以至于使用etcd

   ```
   [root@srv-k8s-master1 ~]# vim /etc/sysconfig/flanneld
   FLANNEL_ETCD_ENDPOINTS="http://127.0.0.1:2379"    ＃ ETCD连接地址
   FLANNEL_ETCD_PREFIX="/coreos.com/network"    ＃ 修改成我们第四步创建的key；
   ```

7. 启动flannel

   ```
   [root@srv-k8s-master1 ~]# systemctl start flanneld
   [root@srv-k8s-master1 ~]# systemctl enable flanneld
   [root@srv-k8s-master1 ~]# netstat -nlpu
   ```

8. 安装docker
   
   ```
   [root@srv-k8s-master1 ~]# yum install -y docker
   ```
   
9. 为了确保flannel可以将Docker进程的虚拟网卡的相关的流量进行转发，我们需要更改Docker的配置文件；
   1. flanneld进程起来之后，通过**ifconfig**或**ip add | grep flannel | grep inet**查看flannel0虚拟网桥；
      
      ```
      [root@srv-k8s-master1 ~]# ip add | grep flannel | grep inet    # master1获取到192.168.93.0/16网段的地址
   		  inet 192.168.93.0/16 scope global flannel0
   		```  
   2. 当flanneld进程启动时，flannel将会从etcd中获取子网网段的信息，然后写到系统的环境变量文件中去/run/flannel/subnet.env，我们可以通过--subnet-file参数去改变默认的环境变量文件

      ```
      [root@srv-k8s-master1 ~]# cat /run/flannel/subnet.env
		FLANNEL_NETWORK=192.168.0.0/16
		FLANNEL_SUBNET=192.168.93.1/24
		FLANNEL_MTU=1472
		FLANNEL_IPMASQ=false
      ```

   3. 修改docker的环境变量文件，添加**--bip=${FLANNEL_SUBNET}, --mtu=${FLANNEL_MTU}, --ip-masq=${FLANNEL_IPMASQ}**
      >--bip: 指定docker0网桥的ip地址，此处的值为flannel环境变量文件中的FLANNEL_SUBNET
      
      >--mtu: 指定容器网络的最大数据包大小(docker0和veth都适用)
      
      >--ip-masq: 可选参数，是否启用IP伪装
   
   ```
   [root@srv-k8s-master1 ~]# grep "OPTIONS" /etc/sysconfig/docker
	OPTIONS='--selinux-enabled --log-driver=journald --signature-verification=false --bip=192.168.93.1/24 --	mtu=1472 --ip-masq=false'
   ```

10  . 启动Docker服务
    
    ```
    [root@srv-k8s-master1 ~]# systemctl restart docker
    [root@srv-k8s-master1 ~]# systemctl enable docker
    
    [root@srv-k8s-master1 ~]# ip -4 a | grep inet    # 检查docker0地址是否是预先的配置
    inet 127.0.0.1/8 scope host lo
    inet 10.1.96.232/24 brd 10.1.96.255 scope global ens33
    inet 192.168.93.0/16 scope global flannel0
    inet 192.168.93.1/24 scope global docker0      # Docker服务的虚拟网卡地址为192.168.93.1
    
    [root@srv-k8s-master1 ~]# route -n    ＃ 检查路由
	 Kernel IP routing table
	 Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
	 0.0.0.0         10.1.96.1       0.0.0.0         UG    100    0        0 ens33
	 10.1.96.0       0.0.0.0         255.255.255.0   U     100    0        0 ens33
	 192.168.0.0     0.0.0.0         255.255.0.0     U     0      0        0 flannel0
	 192.168.93.0    0.0.0.0         255.255.255.0   U     0      0        0 docker0
    ```
    
   > 网络流是如何走的呢???
   > ![](https://github.com/oaas/kubernetes/blob/master/chapter1/k8s_flanneld.png?raw=true)
   
   > 以上图为例：
   
   > Pod1想要发送数据包给Pod2，之前说过，每个Pod有自己的IP地址，数据流量经过Veth0(另一端连接到docker0)，然后路由到flannel0，数据包被分装后经过node的物理网卡eth0发送给node2，node2进行进行解包，最终将数据发给Pod4.
   
    






















