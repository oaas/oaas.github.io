## 转发容器的端口

> 之前我们接触到了如何转发容器的端口到集群内部和外部，接下来让我们深入探讨一下；

##### Kubernetes有四种类型的网络模式
* 容器与容器之间的通信问题
* Pod与Pod之间的通信问题
* Pod与Service的通信问题
* 外部到内部的通信问题


####  容器与容器之间的通信

```
# 1.  创建一个nginx pod
[root@srv-k8s-master1 ~]# cat nginxpod.yaml
apiVersion: "v1"
kind: Pod
metadata:
  name: nginxpod
spec:
  containers:
    - name: nginx80
      image: nginx
      ports:
        - containerPort: 80
          hostPort: 80
    - name: nginx8800
      image: msfuko/nginx_8800
      ports:
        - containerPort: 8800
          hostPort: 8800

# 2. 创建          
[root@srv-k8s-master1 ~]# kubectl create -f nginxpod.yaml
pod "nginxpod" created 

# 3. 查看pod的状态
[root@srv-k8s-master1 ~]# kubectl get pods nginxpod
NAME       READY     STATUS    RESTARTS   AGE
nginxpod   2/2       Running   0          2m

# 4. 获取pod内容器的id， ip
[root@srv-k8s-master1 ~]# kubectl describe pod nginxpod
Name:		nginxpod
Namespace:	new-namespace
Node:		srv-k8s-node2/10.1.96.233
Start Time:	Tue, 07 Nov 2017 08:14:31 -0500
Labels:		<none>
Status:		Running
IP:		192.168.96.54
Controllers:	<none>
Containers:
  nginx80:
    Container ID:		docker://fc7acfe09fdbc802695afd25f0c04627b3234814c74a024fdc6567e6861a882e
    Image:			nginx
    Image ID:			docker-pullable://docker.io/nginx@sha256:9fca103a62af6db7f188ac3376c60927db41f88b8d2354bf02d2290a672dc425
    Port:			80/TCP
    State:			Running
      Started:			Tue, 07 Nov 2017 08:14:41 -0500
    Ready:			True
    Restart Count:		0
    Volume Mounts:		<none>
    Environment Variables:	<none>
  nginx8800:
    Container ID:		docker://624d9f56a0a9865274c90cda692ec84a83a1c1a1f573e27052f4fce4e6361368
    Image:			msfuko/nginx_8800
    Image ID:			docker-pullable://docker.io/msfuko/nginx_8800@sha256:615e8478cb47bba5eed681298edac2c44e3a050403f86d68aa68ae669fd03da2
    Port:			8800/TCP
    State:			Running
      Started:			Tue, 07 Nov 2017 08:15:24 -0500
    Ready:			True
    Restart Count:		0
    Volume Mounts:		<none>
    Environment Variables:	<none>
......

# 5. 使用jq（JSON解析器）去查看容器的网络信息
[root@srv-k8s-node2 ~]# docker inspect fc7acfe09fdb | jq '.[]| {NetworkMode: .HostConfig.NetworkMode, NetworkSettings: .NetworkSettings}'
{
  "NetworkMode": "container:9e7373aa91c29a66f534c57c9d18af69889f417a9e7e5c77e0d4690d71c85459",
  "NetworkSettings": {
    "Bridge": "",
    "SandboxID": "",
    "HairpinMode": false,
    "LinkLocalIPv6Address": "",
    "LinkLocalIPv6PrefixLen": 0,
    "Ports": null,
    "SandboxKey": "",
    "SecondaryIPAddresses": null,
    "SecondaryIPv6Addresses": null,
    "EndpointID": "",
    "Gateway": "",
    "GlobalIPv6Address": "",
    "GlobalIPv6PrefixLen": 0,
    "IPAddress": "",
    "IPPrefixLen": 0,
    "IPv6Gateway": "",
    "MacAddress": "",
    "Networks": null
  }
}
[root@srv-k8s-node2 ~]# docker inspect 624d9f56a0a9 | jq '.[]| {NetworkMode: .HostConfig.NetworkMode, NetworkSettings: .NetworkSettings}'
{
  "NetworkMode": "container:9e7373aa91c29a66f534c57c9d18af69889f417a9e7e5c77e0d4690d71c85459",
  "NetworkSettings": {
    "Bridge": "",
    "SandboxID": "",
    "HairpinMode": false,
    "LinkLocalIPv6Address": "",
    "LinkLocalIPv6PrefixLen": 0,
    "Ports": null,
    "SandboxKey": "",
    "SecondaryIPAddresses": null,
    "SecondaryIPv6Addresses": null,
    "EndpointID": "",
    "Gateway": "",
    "GlobalIPv6Address": "",
    "GlobalIPv6PrefixLen": 0,
    "IPAddress": "",
    "IPPrefixLen": 0,
    "IPv6Gateway": "",
    "MacAddress": "",
    "Networks": null
  }
}

# 6. 从第五步的信息可以看出，NetworkMode为映射的Container模式，被映射到的网络容器为9e7373aa91c29a66f534c57c9d18af69889f417a9e7e5c77e0d4690d71c85459， 让我们看一夏pause(kubernetes自动创建的容器)
[root@srv-k8s-node2 ~]# docker inspect 9e7373aa91c2 | jq '.[]| {NetworkMode: .HostConfig.NetworkMode, NetworkSettings: .NetworkSettings}'
{
  "NetworkMode": "default",
  "NetworkSettings": {
    "Bridge": "",
    "SandboxID": "b4272c190a1adb852095649f9ffa9619f29c1424577b7a39f230cd8c3b0c9b11",
    "HairpinMode": false,
    "LinkLocalIPv6Address": "",
    "LinkLocalIPv6PrefixLen": 0,
    "Ports": {
      "80/tcp": [
        {
          "HostIp": "0.0.0.0",
          "HostPort": "80"
        }
      ],
      "8800/tcp": [
        {
          "HostIp": "0.0.0.0",
          "HostPort": "8800"
        }
      ]
    },
    "SandboxKey": "/var/run/docker/netns/b4272c190a1a",
    "SecondaryIPAddresses": null,
    "SecondaryIPv6Addresses": null,
    "EndpointID": "9e77451e9524a08609984084ae32024e4f5cc8621be603322e6019fdd56ab2a5",
    "Gateway": "192.168.96.1",
    "GlobalIPv6Address": "",
    "GlobalIPv6PrefixLen": 0,
    "IPAddress": "192.168.96.54",
    "IPPrefixLen": 24,
    "IPv6Gateway": "",
    "MacAddress": "02:42:c0:a8:60:36",
    "Networks": {
      "bridge": {
        "IPAMConfig": null,
        "Links": null,
        "Aliases": null,
        "NetworkID": "1a423b0c96e6f8049e914361f4f531905bebd19a8c538b1d2193e59a1af39c60",
        "EndpointID": "9e77451e9524a08609984084ae32024e4f5cc8621be603322e6019fdd56ab2a5",
        "Gateway": "192.168.96.1",
        "IPAddress": "192.168.96.54",
        "IPPrefixLen": 24,
        "IPv6Gateway": "",
        "GlobalIPv6Address": "",
        "GlobalIPv6PrefixLen": 0,
        "MacAddress": "02:42:c0:a8:60:36"
      }
    }
  }
}
[root@srv-k8s-node2 ~]# ip add | grep inet
    inet 127.0.0.1/8 scope host lo
    inet 10.1.96.233/24 brd 10.1.96.255 scope global ens33
    inet 192.168.96.1/24 scope global docker0
    inet 192.168.96.0/16 scope global flannel0

# 7. 如第六步所示，Pause容器的网络模式为default, 被分配到这个pod的ip地址为192.168.96.54，容器对应的网关是是192.168.96.1,即docker0，这个pause容器在创建pod的时候自动创建，被用来处理或路由pod内的网络，因此，pod内容器会共享网络命名空间，所以他们可以通过localhost+不同的端口进行通信；
```

![](https://raw.githubusercontent.com/oaas/kubernetes/master/chapter3/ctoc.png)


#### Pod与Pod之间的通信

![](https://raw.githubusercontent.com/oaas/kubernetes/master/chapter3/poptopod.png)

###### 如上图所示，由于每个容器有自己的ip地址，这使得容器之间的通信变的更加容易。通过flannel作为overlay网络，每个node都会分配一个子网网段，每个数据包都会被封装成UDP包。这样的话，通过使用pod的ip，数据包从pod出发，达到veth0，连接docker0之后到达flannel0会被封装成UDP的包，然后通过主机的网卡转发到其他的node节点；


#### Pod到Service之间的通信
###### Pod随时都有可能挂掉，因此Pod的IP地址就会经常发生变化，当我们想暴露pod或RC的端口时，我们可以创建service作为代理或负载均衡器，kubernetes会分配给service一个虚拟ip地址，因此service可以接受用户的请求，然后将流量分发到后端的pod；

```
# 1. 准配rc的配置
[root@srv-k8s-master1 ~]# cat nginx-rc.yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: podtoservcie
spec:
  replicas: 2
  selector:
    app: nginx
  template:
    metadata:
      name: podtoservice
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx-podtoservice
        image: nginx
        ports:
        - containerPort: 80

# 2. 创建        
[root@srv-k8s-master1 ~]# kubectl create -f nginx-rc.yaml
replicationcontroller "podtoservcie" created

# 3. 获取pod的信息
[root@srv-k8s-master1 ~]# kubectl get pod | grep podtoser
podtoservcie-3jf11           1/1       Running   0          1m
podtoservcie-kvz28           1/1       Running   0          1m

# 4. 创建一个service，暴露rc的端口
[root@srv-k8s-master1 ~]# kubectl expose rc podtoservcie --port=80
service "podtoservcie" exposed

# 5. 获取service的信息
[root@srv-k8s-master1 ~]# kubectl describe service podtoservcie
Name:			podtoservcie
Namespace:		new-namespace
Labels:			app=nginx
Selector:		app=nginx
Type:			ClusterIP
IP:			192.168.16.229
Port:			<unset>	80/TCP
Endpoints:		192.168.96.30:80,192.168.96.44:80,192.168.96.45:80 + 10 more...
Session Affinity:	None
No events.

# 6. 如第五步信息显示，service的虚拟IP地址为192.168.16.229, 对外暴露的端口为80, 然后分发客户端的流量到192.168.96.30:80,192.168.96.44:80,192.168.96.45:80等

# 7. 更多信息请查看node节点的iptables规则
```

![](https://raw.githubusercontent.com/oaas/kubernetes/master/chapter3/servicetorc.png)


#### 由外到内的通信

###### 被用在如果有第三方的负载均衡器，比如aws的ELB，或者说是直接访问node的ip的场合

##### 下面我们创建一个service，然后通过node的ip地址直接访问；

```
# 1. 借助上面创建的名字为podtoservcie的replicaton controller，我们直接创建一个基于LoadBalancer的service
[root@srv-k8s-master1 ~]# kubectl expose rc podtoservcie --port=80 --type=LoadBalancer --name="my-second-service"
service "my-second-service" exposed

# 2. 查看service的详细信息
[root@srv-k8s-master1 ~]# kubectl describe service my-second-service
Name:			my-second-service
Namespace:		new-namespace
Labels:			app=nginx
Selector:		app=nginx
Type:			LoadBalancer
IP:			192.168.99.133
Port:			<unset>	80/TCP
NodePort:		<unset>	30812/TCP
Endpoints:		192.168.96.30:80,192.168.96.44:80,192.168.96.45:80 + 10 more...
Session Affinity:	None
No events.

# 3. 请在对应的node节点上查看详细的iptables规则
[root@srv-k8s-node2 ~]#  iptables -nvL -t nat | grep my-second-service
......

```

#####  如下图所示，从$node_ip+30812或ClusterIp访问的流量通过负载均衡的机制被重定向到node节点对应的pod上。

![](https://raw.githubusercontent.com/oaas/kubernetes/master/chapter3/eti.png)


## 更加灵活的管理容器

###### 在Kubernetes中，Pod是最小的计算单元，我们已经知道Pod在基本用法：Pod通常是被replication controller管理，然后通过service暴露。 接下来我们讨论两个新的功能： Job和DaemonSet，这两个功能可以使我们更加高效的使用Pod.

#### 什么是Job Pod和DamonSet Pod？

> Job Pod：Pod完成任务之后将会被终止;

> DamondSet Pod: Pod会在每一个node节点上创建；

####  两个新的功能都属于Kubernetes API的扩展。此外， daemonSet类型的Pod默认是被禁用的，因此需要启用之后才能使用。通过一下方式启用daemonSet:


```
# 1. 添加--runtime-config在apiserver的配置文件中
[root@srv-k8s-master1 ~]# grep "KUBE_API_ARGS" /etc/kubernetes/apiserver
KUBE_API_ARGS = "--runtime-config=extensions/v1beta1/daemonsets=true"

# 2. 重启使其生效
[root@srv-k8s-master1 ~]# systemctl restart kube-apiserver

# 3. 验证是否成功开启
[root@srv-k8s-master1 ~]# curl http://localhost:8080/apis/extensions/v1beta1
{
  "kind": "APIResourceList",
  "groupVersion": "extensions/v1beta1",
  "resources": [
    {
      "name": "daemonsets",
      "namespaced": true,
      "kind": "DaemonSet"
    },
    
......    
```


#### 创建一个Job Pod

```
# 1. 准备yaml文件, 如果是一个Job Pod，restartPolicy应该被设置为Never或OnFailure. 因为指定的工作完成后，Pod就会被终止；
[root@srv-k8s-master1 ~]# cat job-dpkg.yaml
apiVersion: extensions/v1beta1
kind: Job
metadata:
  name: package-check
spec:
  selector:
    matchLabels:
      image: ubuntu
      test: dpkg
  template:
    metadata:
      labels:
        image: ubuntu
        test: dpkg
        owner: lz
    spec:
      containers:
      - name: package-check
        image: ubuntu
        command: ["dpkg-query", "-l"]
      restartPolicy: Never

# 2. 创建      
[root@srv-k8s-master1 ~]# kubectl create -f job-dpkg.yaml
job "package-check" created

 # 3. 验证结果
 [root@srv-k8s-master1 ~]# kubectl get job
NAME            DESIRED   SUCCESSFUL   AGE
package-check   1         1            2m

[root@srv-k8s-master1 ~]# kubectl describe job package-check
Name:		package-check
Namespace:	new-namespace
Image(s):	ubuntu
Selector:	image=ubuntu,test=dpkg
Parallelism:	1
Completions:	1
Start Time:	Tue, 07 Nov 2017 19:29:59 -0500
Labels:		image=ubuntu
		owner=lz
		test=dpkg
Pods Statuses:	0 Running / 1 Succeeded / 0 Failed
No volumes.
Events:
  FirstSeen	LastSeen	Count	From			SubObjectPath	Type		Reason			Message
  ---------	--------	-----	----			-------------	--------	------			-------
  1m		1m		1	{job-controller }			Normal		SuccessfulCreate	Created pod: package-check-2pbkx
  
# 4. 名为package-check-2pbkx的pod在执行完成dpkg命令后就退出了
[root@srv-k8s-master1 ~]# kubectl logs package-check-2pbkx
Desired=Unknown/Install/Remove/Purge/Hold
| Status=Not/Inst/Conf-files/Unpacked/halF-conf/Half-inst/trig-aWait/Trig-pend
|/ Err?=(none)/Reinst-required (Status,Err: uppercase=bad)
||/ Name                     Version                       Architecture Description
+++-========================-=============================-============-======================================================================
ii  adduser                  3.113+nmu3ubuntu4             all          add and remove users and groups
ii  apt                      1.2.24                        amd64        commandline package manager
ii  base-files               9.4ubuntu4.5                  amd64        Debian base system miscellaneous files
 ......
 ......
```

#### 创建一个DaemonSet的Pod

如果Kubernets DaemonSet被创建，被定义的Pod将会部署在每一台node节点上. 确保运行在node节点的容器占用同样的资源，此时，这个容器是作为daemon进程存在的；
 
 ```
 
 ```