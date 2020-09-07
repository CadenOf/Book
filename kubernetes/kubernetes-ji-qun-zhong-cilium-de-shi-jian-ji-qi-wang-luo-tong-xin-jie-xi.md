# kubernetes 集群中 cilium 的实践及其网络通信解析

## Cilium Overview

[Cilium](https://links.jianshu.com/go?to=https%3A%2F%2Fcilium.io%2F) 是一个基于 eBPF 和 XDP 的高性能容器网络方案的开源项目，目标是为微服务环境提供网络、负载均衡、安全功能，主要定位是容器平台。

![](../.gitbook/assets/image%20%2819%29.png)

**Why Cilium ?**

现在应用程序服务的发展已从单体结构转变为微服务架构，微服务间的的通信通常使用轻量级的 http 协议。微服务应用往往是经常性更新变化的，在持续交付体系中为了应对负载的变化通常会横向扩缩容，应用容器实例也会随着应用的更新而被创建或销毁。

这种高频率的更新对微服务间的网络可靠连接带来了挑战：

* 传统的 Linux 网络安全连接方式是通过过滤器（例如 iptables ）在 IP 地址及端口上进行的，但是在微服务高频更新的环境下，应用容器实例的 IP 端口是会经常变动。在微服务应用较多的场景下，由于这种频繁的变动，成千上万的 iptables 规则将会被频繁的更新。
* 出于安全目的，协议端口（例如 HTTP 流量的 TCP 端口 80）不再用于区分应用流量，因为该端口用于跨服务的各种消息。
* 传统系统主要使用 IP 地址作为标识手段，在微服务频繁更新的架构中，某个 IP 地址可能只会存在短短的几秒钟，这将难以提供准确的可视化追踪痕迹。

Cilium 通过利用 BPF 具有能够透明的注入网络安全策略并实施的功能，区别于传统的 IP 地址标识的方式，Cilium 是基于 service / pod /container 标识来实现的，并且可以在应用层实现 L7 Policy 网络过滤。总之，Cilium 通过解藕 IP 地址，不仅可以在高频变化的微服务环境中应用简单的网络安全策略，还能在支持 L3/L4 基础上通过对 http 层进行操作来提供更强大的网络安全隔离。BPF 的使用使 Cilium 甚至可以在大规模环境中以高度可扩展的方式解决这些挑战问题。

**Cilium 的主要功能特性**：

* 支持 L3/L4/L7 安全策略，这些策略按照使用方法又可以分为 基于身份的安全策略（Security identity） 基于 CIDR 的安全策略 基于标签的安全策略
* 支持三层扁平网络

  Cilium 的一个简单的扁平 3 层网络具有跨多个群集的能力，可以连接所有应用程序容器。通过使用主机作用域分配器，可以简化 IP 分配，这意味着每台主机不需要相互协调就可以分配到 IP subnet。

  Cilium 支持以下多节点的网络模型：

  * Overlay 网络，当前支持 Vxlan 和 Geneve ，但是也可以启用 Linux 支持的所有封装格式。
  * Native Routing, 使用 Linux 主机的常规路由表或云服务商的高级网络路由等。此网络要求能够路由应用容器的 IP 地址。

* 提供基于 BPF 的负载均衡 Cilium 能对应用容器间的流量及外部服务支持分布式负载均衡。
* 提供便利的监控手段和排错能力

  可见性和快速的问题定位能力是一个分布式系统最基础的部分。除了传统的 tcpdump 和 ping 命令工具，Cilium 提供了：

  1. 具有元数据的事件监控：当一个 Packet 包被丢弃，这个工具不会只报告这个包的源 IP 和目的 IP，此工具还会提供关于发送方和接送方所有相关的标签信息。
  2. 决策追踪：为何一个 packet 包被丢弃，为何一个请求被拒绝？策略追踪框架允许追踪正在运行的工作负载和基于任意标签定义的策略决策过程。
  3. 通过 Prometheus 暴露 Metrics 指标：关键的 Metrics 指标可以通过 Prometheus 暴露出来到监控看板上进行集成展示。
  4. Hubble：一个专门为 Cilium 开发的可视化平台。它可以通过 flow log 来提供微服务间的依赖关系，监控告警操作及应用服务安全策略可视化。

## **Cilium 的部署**

此处使用外部的 etcd 的部署方式，外部 etcd 安装 cilium 在较大的运行环境中能够提供更好的性能。

**Requirements**

```text
1.  Kubernetes >= 1.9
2.  Linux kernel >= 4.9
3.  ETCD >= 3.1.0
4.  kubernetes 环境中安装了 Helm 3
5.  Kubernetes in CNI mode
6.  在所有 worker node 上挂载 BPF 文件系统
7.  推荐：在 kube-controller-manager 上使能 PodCIDR allocation (--allocate-node-cidrs) 
```

**安装 helm 3**

```text
# 下载解压 helm 安装包
[root@k8s-master-01 ~]# wget https://get.helm.sh/helm-v3.1.2-linux-amd64.tar.gz
[root@k8s-master-01 ~]# tar -zxvf helm-v3.1.2-linux-amd64.tar.gz
[root@k8s-master-01 ~]# mv linux-amd64/helm /usr/local/bin/

# verify
[root@k8s-master-01 ~]# helm help
The Kubernetes package manager

Common actions for Helm:

- helm search:    search for charts
- helm pull:      download a chart to your local directory to view
- helm install:   upload the chart to Kubernetes
- helm list:      list releases of charts

Environment variables:
+------------------+-----------------------------------------------------------------------------+
| Name             | Description                                                                 |
+------------------+-----------------------------------------------------------------------------+
| $XDG_CACHE_HOME  | set an alternative location for storing cached files.                       |
| $XDG_CONFIG_HOME | set an alternative location for storing Helm configuration.                 |
| $XDG_DATA_HOME   | set an alternative location for storing Helm data.                          |
| $HELM_DRIVER     | set the backend storage driver. Values are: configmap, secret, memory       |
| $HELM_NO_PLUGINS | disable plugins. Set HELM_NO_PLUGINS=1 to disable plugins.                  |
| $KUBECONFIG      | set an alternative Kubernetes configuration file (default "~/.kube/config") |
+------------------+-----------------------------------------------------------------------------+

Helm stores configuration based on the XDG base directory specification, so

- cached files are stored in $XDG_CACHE_HOME/helm
- configuration is stored in $XDG_CONFIG_HOME/helm
- data is stored in $XDG_DATA_HOME/helm

Use "helm [command] --help" for more information about a command.
```

**挂载 BPF 文件系统**

在 kubernetes 集群所有 node 上挂载 bpf 文件系统

```text
[root@k8s-master-01 ~]# mount bpffs /sys/fs/bpf -t bpf

# verify
[root@k8s-master-01 ~]# mount |grep bpf
bpffs on /sys/fs/bpf type bpf (rw,relatime)

# persistence configuration, don’t worry that ‘bpffs’ displaying as red, seems bpf was new commer, fastab desen’t update that feature.
[root@k8s-master-01 ~]# echo "bpffs        /sys/fs/bpf      bpf     defaults 0 0" >> /etc/fstab
```

**kubernetes 配置**

```text
# 在所有的 kubernetes node 中的 kubelet 配置使用 CNI 模式, kubelet.config 中添加 
--network-plugin=cni

# 在 kube-controller-manager 中使能 PodCIDR, kube-controller-manager.config 中添加
--allocate-node-cidrs=true
```

**cilium 安装**

当使用外部 etcd 作为 cilium 的 k-v 存储，etcd 的 IP 地址需要在 cilium 的 configmap 中配置。

使用 helm 安装 cilium

```text
# 添加 helm cilium repo
[root@k8s-master-01 ~]#  helm repo add cilium https://helm.cilium.io/

# 创建 etcd ssl 证书
[root@k8s-master-01 ~]#  kubectl create secret generic -n kube-system cilium-etcd-secrets \
     --from-file=etcd-client-ca.crt=/etc/etcd/ssl/ca.crt \
     --from-file=etcd-client.key=/etc/etcd/ssl/etcd.key \
     --from-file=etcd-client.crt=/etc/etcd/ssl/etcd.crt

# 安装 cilium，指定 cilium 版本为 v1.7.1, 开启 SSL 验证，开启 prometheus 监控，添加 etcd cluster 的 menber endpoints
[root@k8s-master-01 ~]#  helm install cilium cilium/cilium\
  --version 1.7.1\
  --set global.etcd.enabled=true\ 
  --set global.etcd.ssl=true\ 
  --set global.prometheus.enabled=true\
  --set global.etcd.endpoints[0]=https://172.19.50.7:2379\
  --set global.etcd.endpoints[1]=https://172.19.60.32:2379\
  --set global.etcd.endpoints[2]=https://172.19.100.16:2379\
  --namespace kube-system
NAME: cilium
LAST DEPLOYED: Mon Mar 16 16:44:33 2020
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None

# 验证 cilium pod 都安装成功
[root@k8s-master-01 ~]#  kubectl --namespace kube-system get ds cilium
NAME     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
cilium   4         4         4       4            4           <none>          13h

[root@k8s-master-01 ~]#  kubectl -n kube-system get deployments cilium-operator
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
cilium-operator   1/1     1            1           13h
```

**安装 cilium 连接测试用例**

此用例将会部署一系列的 deployment，它们会使用多种路径来相互访问，连接路径包括带或者不带服务负载均衡和各种网络策略的组合。

部署的 podName 表示连接方式，readiness/liveness 探针则可指示连接是否成功。

```text
[root@k8s-master-01 ~]# kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/HEAD/examples/kubernetes/connectivity-check/connectivity-check.yaml -n app-service

[root@k8s-master-01 ~]# kubectl get pods -o wide -n app-service
NAME                                                     READY   STATUS    RESTARTS   AGE     IP               NODE            NOMINATED NODE   READINESS GATES
echo-a-58dd59998d-n9g9p                                  1/1     Running   0          9m13s   10.244.1.50      k8s-master-02   <none>           <none>
echo-b-669ccc7765-lzqn7                                  1/1     Running   0          9m13s   10.244.2.50      k8s-master-03   <none>           <none>
host-to-b-multi-node-clusterip-6fb94d9df6-rbjwz          1/1     Running   3          9m13s   192.168.66.226   k8s-master-02   <none>           <none>
host-to-b-multi-node-headless-7c4ff79cd-hm6sr            1/1     Running   3          9m13s   192.168.66.226   k8s-master-02   <none>           <none>
pod-to-a-5c8dcf69f7-gldq9                                1/1     Running   3          9m13s   10.244.2.30      k8s-master-03   <none>           <none>
pod-to-a-allowed-cnp-75684d58cc-tf9nn                    1/1     Running   1          9m13s   10.244.2.239     k8s-master-03   <none>           <none>
pod-to-a-external-1111-669ccfb85f-7r4j8                  1/1     Running   0          9m13s   10.244.2.251     k8s-master-03   <none>           <none>
pod-to-a-l3-denied-cnp-7b8bfcb66c-wd4nj                  1/1     Running   0          9m13s   10.244.2.134     k8s-master-03   <none>           <none>
pod-to-b-intra-node-74997967f8-ml5ps                     1/1     Running   3          9m13s   10.244.2.95      k8s-master-03   <none>           <none>
pod-to-b-multi-node-clusterip-587678cbc4-4qcb2           1/1     Running   3          9m13s   10.244.1.28      k8s-master-02   <none>           <none>
pod-to-b-multi-node-headless-574d9f5894-tmfwn            1/1     Running   3          9m13s   10.244.1.138     k8s-master-02   <none>           <none>
pod-to-external-fqdn-allow-google-cnp-6dd57bc859-l49z2   1/1     Running   0          9m12s   10.244.2.62      k8s-master-03   <none>           <none>
```

**安装 hubble** [https://github.com/cilium/hubble](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fcilium%2Fhubble)

hubble 是一个用于云原生工作负载的完全分布式网络和安全可视化平台。它建立在 Cilium 和 eBPF 的基础上，以完全透明的方式深入了解服务以及网络基础结构的通信和行为。

```text
[root@k8s-master-01 ~]# git clone https://github.com/cilium/hubble.git
[root@k8s-master-01 ~]# cd hubble/install/kubernetes
[root@k8s-master-01 ~]# helm install hubble ./hubble \
    --namespace kube-system \
    --set metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,http}" \
    --set ui.enabled=true
```

hubble 对前面安装的测试用例监控信息

![](../.gitbook/assets/image%20%2818%29.png)



## **Cilium 的网络通信解析**

cilium 在 kubernetes 集群中安装好后，此处我们来探究一下在不同 node 上 pod 间的 vxlan 通信方式。

![](../.gitbook/assets/image%20%2817%29.png)

cilium 安装完后，cilium agent 会在 node 上创建 `cilium_net` 与 `cilium_host` 一对 veth pair 及用于跨宿主机通信的 `cilium_vxlan`，然后在 `cilium_host` 上配置其管理的 CIDR IP 作为网关。

如上图示中，通过抓包分析 Container A 与 Container B 之前的通信路径。

**Container A ping Container B** （以下称 Container A -&gt; CA , Container B -&gt; CB）

进入 Node01 上 CA 内 \(10.244.1.154\)，ping CB ip 地址 10.224.6.11

```text
[root@Node01 ~]# docker exec -it 5eb8b6605c64 bash
root@voyager-client-8769496c4-ttzsj:/# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.244.1.154  netmask 255.255.255.255  broadcast 0.0.0.0
        ether 02:16:7a:1d:0c:7e  txqueuelen 0  (Ethernet)
        RX packets 1518748  bytes 139582792 (133.1 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1377417  bytes 286246722 (272.9 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

root@voyager-client-8769496c4-ttzsj:/# ping 10.244.6.11
PING 10.244.6.11 (10.244.6.11) 56(84) bytes of data.
64 bytes from 10.244.6.11: icmp_seq=1 ttl=63 time=0.664 ms
64 bytes from 10.244.6.11: icmp_seq=2 ttl=63 time=1.58 ms
64 bytes from 10.244.6.11: icmp_seq=3 ttl=63 time=0.574 ms
64 bytes from 10.244.6.11: icmp_seq=4 ttl=63 time=0.493 ms
64 bytes from 10.244.6.11: icmp_seq=5 ttl=63 time=0.587 ms
64 bytes from 10.244.6.11: icmp_seq=6 ttl=63 time=0.460 ms
64 bytes from 10.244.6.11: icmp_seq=7 ttl=63 time=0.551 ms
```

进入 Node01 上 cilium agent 容器内，查看 CA 容器作为 cilium endpoint 的信息

```text
[root@Node01 ~]# docker exec -it 7c0ce8909d21 bash

# 通过 cilium bpf endpoint 可看到在宿主机 Node01 上 CA 容器 attach 的网卡 mac 地址 F2:19:FC:0E:0A:8c
root@cilium-agent:~# cilium bpf endpoint list
IP ADDRESS         LOCAL ENDPOINT INFO
10.96.0.10:0       (localhost)
192.168.66.226:0   (localhost)
10.102.101.239:0   (localhost)
10.101.126.212:0   (localhost)
10.99.24.234:0     (localhost)
10.244.1.4:0       id=1928  flags=0x0000 ifindex=10  mac=26:84:C5:50:4D:F9 nodemac=C6:AB:F1:63:C1:FE
10.99.146.112:0    (localhost)
10.96.0.1:0        (localhost)
10.110.33.8:0      (localhost)
10.244.1.208:0     (localhost)
10.111.30.67:0     (localhost)
10.108.89.187:0    (localhost)
10.107.23.171:0    (localhost)
10.244.1.183:0     id=3023  flags=0x0000 ifindex=48  mac=7E:DB:15:12:77:78 nodemac=CA:DE:51:6C:3B:6E
10.244.1.18:0      id=3047  flags=0x0000 ifindex=322 mac=92:BB:35:0B:64:DD nodemac=96:2D:AE:BD:78:6D
10.244.1.154:0     id=3432  flags=0x0000 ifindex=336 mac=02:16:7A:1D:0C:7E nodemac=F2:19:FC:0E:0A:8C
10.244.1.106:0     id=2222  flags=0x0000 ifindex=310 mac=02:EF:1A:74:99:36 nodemac=CE:12:20:E1:99:98

# 在宿主机上可看到对应的网卡为 lxc8b528e748ff4(lxcxxA)
[root@Node01 ~]# ifconfig
cilium_host: flags=4291<UP,BROADCAST,RUNNING,NOARP,MULTICAST>  mtu 1500
        inet 10.244.1.208  netmask 255.255.255.255  broadcast 0.0.0.0
        inet6 fe80::b814:51ff:fe6e:3ec0  prefixlen 64  scopeid 0x20<link>
        ether ba:14:51:6e:3e:c0  txqueuelen 1000  (Ethernet)
        RX packets 28524536  bytes 4348528855 (4.0 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1583  bytes 107646 (105.1 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

cilium_net: flags=4291<UP,BROADCAST,RUNNING,NOARP,MULTICAST>  mtu 1500
        inet6 fe80::cd4:62ff:fe16:2c7c  prefixlen 64  scopeid 0x20<link>
        ether 0e:d4:62:16:2c:7c  txqueuelen 1000  (Ethernet)
        RX packets 1583  bytes 107646 (105.1 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 28524536  bytes 4348528855 (4.0 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

cilium_vxlan: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet6 fe80::f4ee:12ff:fe6e:c46f  prefixlen 64  scopeid 0x20<link>
        ether f6:ee:12:6e:c4:6f  txqueuelen 1000  (Ethernet)
        RX packets 24633319  bytes 3811772734 (3.5 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 24389802  bytes 6318433970 (5.8 GiB)
        TX errors 0  dropped 46 overruns 0  carrier 0  collisions 0

.......
lxc8b528e748ff4: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet6 fe80::f019:fcff:fe0e:a8c  prefixlen 64  scopeid 0x20<link>
        ether f2:19:fc:0e:0a:8c  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
.......
```

CA 容器 ping 出的 icmp 包将会经过网卡 lxc8b528e748ff4 \(lxcxxA\) 路由到 `cilium_host` 网关，路由方式与传统的通过 Linux bridge 这样的二层设备转发不一样，cilium 在每个容器相关联的虚拟网卡上都附加了 bpf 程序，通过连接到 TC \( traffic control \) 入口钩子的 bpf 程序将所有网络流量路由到主机端虚拟设备上，由此 cilium 便可以监视和执行有关进出节点的所有流量的策略，例如 pod 内的 networkPolicy 、L7 policy、加密等规则。

```text
# CA 容器内的默认路由指向 cilium_host 网关地址 10.244.1.208
root@voyager-client-8769496c4-ttzsj:/#  ip route
default via 10.244.1.208 dev eth0 mtu 1450
10.244.1.208 dev eth0 scope link
```

CA 容器内的流量要跨宿主机节点路由到 CB 容器，则需要 `cilium_vxlan` VTEP 设备对流量包进行封装转发到 Node02 上。bpf 程序会查询 tunnel 规则并将流量发送给 `cilium_vxlan`，在 cilium agent 容器内可查看到 bpf 的 tunnel 规则

```text
[root@Node01 ~]# docker exec -it 7c0ce8909d21 bash
root@cilium-agent:~# cilium bpf tunnel list
TUNNEL         VALUE
10.244.3.0:0   192.168.66.196:0
10.244.2.0:0   192.168.66.194:0
10.244.6.0:0   192.168.66.221:0 # CB 容器 ip 为 10.244.6.11, tunnel 对端地址为 192.168.66.221，即为 Node02 主机节点地址
```

在 Node01 上对 `cilium_vxlan` 抓包，可看到 CA 容器对 icmp 包经过了 `cilium_vxlan`

```text
[root@Node01 ~]# tcpdump -i cilium_vxlan icmp -n -vv
tcpdump: listening on cilium_vxlan, link-type EN10MB (Ethernet), capture size 262144 bytes
15:54:23.550645 IP (tos 0x0, ttl 64, id 18157, offset 0, flags [DF], proto ICMP (1), length 84)
    10.244.1.154 > 10.244.6.11: ICMP echo request, id 497, seq 509, length 64
15:54:23.551123 IP (tos 0x0, ttl 64, id 12892, offset 0, flags [none], proto ICMP (1), length 84)
    10.244.6.11 > 10.244.1.154: ICMP echo reply, id 497, seq 509, length 64
15:54:24.574575 IP (tos 0x0, ttl 64, id 18805, offset 0, flags [DF], proto ICMP (1), length 84)
    10.244.1.154 > 10.244.6.11: ICMP echo request, id 497, seq 510, length 64
15:54:24.574999 IP (tos 0x0, ttl 64, id 13181, offset 0, flags [none], proto ICMP (1), length 84)
    10.244.6.11 > 10.244.1.154: ICMP echo reply, id 497, seq 510, length 64
15:54:25.598625 IP (tos 0x0, ttl 64, id 19388, offset 0, flags [DF], proto ICMP (1), length 84)
    10.244.1.154 > 10.244.6.11: ICMP echo request, id 497, seq 511, length 64
15:54:25.599107 IP (tos 0x0, ttl 64, id 13587, offset 0, flags [none], proto ICMP (1), length 84)
```

再对 Node01 上的 eth0 抓包，可看到 `cilium_vxlan` 已将 CA 的流量进行 vxlan 封包，src ip 改为本机 node ip 192.168.66.226, dst ip 改为 192.168.66.221

```text
[root@Node01 ~]# tcpdump -i eth0 -n dst 192.168.66.221 and udp -vv
17:48:38.398691 IP (tos 0x0, ttl 64, id 58330, offset 0, flags [none], proto UDP (17), length 134)
    192.168.66.226.60971 > 192.168.66.221.otv: [no cksum] OTV, flags [I] (0x08), overlay 0, instance 27634
IP (tos 0x0, ttl 64, id 35369, offset 0, flags [DF], proto ICMP (1), length 84)
    10.244.1.154 > 10.244.6.11: ICMP echo request, id 502, seq 1878, length 64
17:48:39.422639 IP (tos 0x0, ttl 64, id 58509, offset 0, flags [none], proto UDP (17), length 134)
    192.168.66.226.60971 > 192.168.66.221.otv: [no cksum] OTV, flags [I] (0x08), overlay 0, instance 27634
IP (tos 0x0, ttl 64, id 36202, offset 0, flags [DF], proto ICMP (1), length 84)
    10.244.1.154 > 10.244.6.11: ICMP echo request, id 502, seq 1879, length 64
17:48:40.446630 IP (tos 0x0, ttl 64, id 59226, offset 0, flags [none], proto UDP (17), length 134)
    192.168.66.226.60971 > 192.168.66.221.otv: [no cksum] OTV, flags [I] (0x08), overlay 0, instance 27634
IP (tos 0x0, ttl 64, id 37119, offset 0, flags [DF], proto ICMP (1), length 84)
    10.244.1.154 > 10.244.6.11: ICMP echo request, id 502, seq 1880, length 64
17:48:41.470617 IP (tos 0x0, ttl 64, id 60174, offset 0, flags [none], proto UDP (17), length 134)
    192.168.66.226.60971 > 192.168.66.221.otv: [no cksum] OTV, flags [I] (0x08), overlay 0, instance 27634
IP (tos 0x0, ttl 64, id 37830, offset 0, flags [DF], proto ICMP (1), length 84)
    10.244.1.154 > 10.244.6.11: ICMP echo request, id 502, seq 1881, length 64
```

到 Node02 上对 eth0 抓包，可看到 CA 容器的流量包已到 Node02 上

```text
[root@Node02 ~]# tcpdump -i eth0 -n src 192.168.66.226 and udp -vv
tcpdump: listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
05:55:08.542658 IP (tos 0x0, ttl 64, id 52637, offset 0, flags [none], proto UDP (17), length 134)
    192.168.66.226.60971 > 192.168.66.221.otv: [no cksum] OTV, flags [I] (0x08), overlay 0, instance 27634
IP (tos 0x0, ttl 64, id 33378, offset 0, flags [DF], proto ICMP (1), length 84)
    10.244.1.154 > 10.244.6.11: ICMP echo request, id 502, seq 2259, length 64
05:55:09.566806 IP (tos 0x0, ttl 64, id 52888, offset 0, flags [none], proto UDP (17), length 134)
    192.168.66.226.60971 > 192.168.66.221.otv: [no cksum] OTV, flags [I] (0x08), overlay 0, instance 27634
IP (tos 0x0, ttl 64, id 34260, offset 0, flags [DF], proto ICMP (1), length 84)
    10.244.1.154 > 10.244.6.11: ICMP echo request, id 502, seq 2260, length 64
05:55:10.567877 IP (tos 0x0, ttl 64, id 53331, offset 0, flags [none], proto UDP (17), length 134)
    192.168.66.226.60971 > 192.168.66.221.otv: [no cksum] OTV, flags [I] (0x08), overlay 0, instance 27634
IP (tos 0x0, ttl 64, id 34318, offset 0, flags [DF], proto ICMP (1), length 84)
    10.244.1.154 > 10.244.6.11: ICMP echo request, id 502, seq 2261, length 64
05:55:11.614841 IP (tos 0x0, ttl 64, id 54206, offset 0, flags [none], proto UDP (17), length 134)
    192.168.66.226.60971 > 192.168.66.221.otv: [no cksum] OTV, flags [I] (0x08), overlay 0, instance 27634
IP (tos 0x0, ttl 64, id 35300, offset 0, flags [DF], proto ICMP (1), length 84)
    10.244.1.154 > 10.244.6.11: ICMP echo request, id 502, seq 2262, length 64
```

对 `cilium_vxlan` 抓包即可看到 CA 容器过来对流量包已被解封

```text
[root@Node02 ~]# tcpdump -i cilium_vxlan -n dst 10.244.6.11 -vv
tcpdump: listening on cilium_vxlan, link-type EN10MB (Ethernet), capture size 262144 bytes
05:59:22.494598 IP (tos 0x0, ttl 64, id 25390, offset 0, flags [DF], proto ICMP (1), length 84)
    10.244.1.154 > 10.244.6.11: ICMP echo request, id 502, seq 2507, length 64
05:59:23.518736 IP (tos 0x0, ttl 64, id 26363, offset 0, flags [DF], proto ICMP (1), length 84)
    10.244.1.154 > 10.244.6.11: ICMP echo request, id 502, seq 2508, length 64
05:59:24.542563 IP (tos 0x0, ttl 64, id 26951, offset 0, flags [DF], proto ICMP (1), length 84)
    10.244.1.154 > 10.244.6.11: ICMP echo request, id 502, seq 2509, length 64
05:59:25.566736 IP (tos 0x0, ttl 64, id 27503, offset 0, flags [DF], proto ICMP (1), length 84)
    10.244.1.154 > 10.244.6.11: ICMP echo request, id 502, seq 2510, length 64
05:59:26.590731 IP (tos 0x0, ttl 64, id 28392, offset 0, flags [DF], proto ICMP (1), length 84)
    10.244.1.154 > 10.244.6.11: ICMP echo request, id 502, seq 2511, length 64
05:59:27.614648 IP (tos 0x0, ttl 64, id 28548, offset 0, flags [DF], proto ICMP (1), length 84)
    10.244.1.154 > 10.244.6.11: ICMP echo request, id 502, seq 2512, length 64
```

至此，CA 容器对流量已到达 cilium 创建的虚拟网卡。

我们知道 Linux 内核本质上是事件驱动的（the Linux Kernel is fundamentally event-driven）, cilium 创建的虚拟网卡接收到流量包将会触发连接到 TC \( traffic control \) ingress 钩子的 bpf 程序，对流量包进行相关策略对处理。

查看 cilium 官方给出的 ingress/egress datapath，可大致验证上述 cilium 的网络通信路径。

首先是 `egress datapath`，图中橙黄色标签为 cilium component，有 cilium 在宿主机上创建的 bpf 程序\(对应着红色标签的 kernel bpf 钩子\)，若使用 L7 Policy 则还有 cilium 创建的 iptables 规则。流量从某个容器 endpoint 通过容器上的 veth pair 网卡 lxcxxx 出发，即会触发 bpf\_sockops.c / bpf\_redir.c bpf 程序，若使用了 L7 Policy 则进入用户空间进行 L7 层的数据处理，若没有使能 L7 Policy 则将触发 TC egress 钩子，bpf\_lxc 对数据进行处理（若使能 L3 加密，则触发其他 bpf 钩子），数据最终被路由到 `cilium_hos`t 网关处，再根据 overlay 模式（vxlan 等）将数据发送出去。

![](../.gitbook/assets/image%20%2816%29.png)

在 cilium `ingress datapath` 中，数据流量进入主机网络设备上，cilium 可根据相关配置，对数据流量进行预处理（prefilter/L3 加解密/ 负载均衡/ L7 Policy 处理）或直接路由到 `cilium_host` 触发相应到 bpf 程序，再到最终的 endpoint 处。

![](../.gitbook/assets/image%20%2815%29.png)

