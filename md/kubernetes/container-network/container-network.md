# Docker 容器网络

>了解容器网络的原理可以帮助开发者更好的理解容器间是如何通信的。事实上，如今 Kubernetes 社区层出不穷的容器网络方案，其实本质都是为了解决**容器跨主通信**这一问题。



## 基础原理

### 网络栈

Linux 容器可见的网络栈，其实是被它所在的 Network Namespace 所隔离的。一个 Linux 网络栈通常包括了：

- 网卡（Network Interface）
- 回环设备（Loopback Device）
- 路由表（Routing Table）
- iptables 规则

对于一个进程而言，这些要素其实就构成了它**发起和响应网络请求的基本环境**。

值得一提的是，容器是可以通过 `--network=host` 参数直接使用宿主机网络栈的。

```bash
docker run –d --network=host --name nginx-host nginx
```

该情况下，容器启动后直接监听的就是宿主机上的 80 端口。这样直接使用宿主机网络栈的方式，虽然可以为容器提供良好的网络性能，但也会不可避免地引入共享网络资源所带来的相关问题，最常见的，比如端口冲突。所以，在大多数情况下，用户都希望容器进程能使用自身 Network Namespace  中的网络栈，这些网络设备是不被“其他人”可见的，进而拥有一套独立的 IP 与端口。



一个显而易见的问题：在这个被隔离的容器进程中，应该如何与其他 Network Namespace 中的容器进行通信呢？

为了理解这个问题，可以把每一个容器类比为一台主机，它们都拥有一套独立的网络栈。如果想要实现两台主机之间的通信，最直接的办法，就是把它们用一根网线连接起来；而若要实现多台主机之间的通信，那就需要用网线把它们连接在一个交换网络上，比如交换机。

在 Linux 中，能够起到虚拟交换机作用的网络设备就是**网桥（Bridge）**。它是一个工作在**数据链路层（Data Link）**的设备，主要功能是**根据 MAC 地址学习来将数据包转发到网桥的不同端口**上，网桥的每个端口与一个网段相连。而为了实现上述目的，Docker 项目就在宿主机上默认创建了一个名为 docker0 的网桥，而凡是连接在 docker0 网桥上的容器，就可以通过它来进行通信。



### Veth Pair

将容器“连接”到 docker0 网桥上时，依赖一种名为 Veth Pair 的虚拟设备。

Veth Pair 设备的特点是：它被创建出来时，总是以两张**虚拟网卡（Veth Peer）**的形式成对出现。并且，由其中一个网卡发出的数据包，总是直接出现在与之对应的另一张网卡上，即使这两张网卡并不处于同一个 Network Namespace 中。这就使得 Veth Pair 设备常常被用作连接不同 Network Namespace 的“网线”。



一个例子，首先启动了一个名为 nginx-1 的容器。

```bash
docker run –d --name nginx-1 nginx
```

进入到这个容器中查看它的网络设备。

```bash
# 在宿主机上
$ docker exec -it nginx-1 /bin/bash
# 在容器内
$ ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.2  netmask 255.255.0.0  broadcast 0.0.0.0
        inet6 fe80::42:acff:fe11:2  prefixlen 64  scopeid 0x20<link>
        ether 02:42:ac:11:00:02  txqueuelen 0  (Ethernet)
        RX packets 364  bytes 8137175 (7.7 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 281  bytes 21161 (20.6 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

$ route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         172.17.0.1      0.0.0.0         UG    0      0        0 eth0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 eth0
```

这张名为 eth0 的网卡，正是一个 Veth Pair 设备在容器内的一端。通过 `route` 命令查看 nginx-1 容器的路由表，可以看到，eth0 网卡是这个容器里的默认路由设备；同时，所有对 172.17.0.0/16 网段的请求，也会被交给 eth0 来处理（第二条 172.17.0.0 路由规则）。



而该 Veth Pair 设备的另一端，就在宿主机上。

```bash
# 在宿主机上
$ ifconfig
...
docker0: Link encap:Ethernet  HWaddr 02:42:d8:e4:df:c1
          inet addr:172.17.0.1  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:d8ff:fee4:dfc1/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:309 errors:0 dropped:0 overruns:0 frame:0
          TX packets:372 errors:0 dropped:0 overruns:0 carrier:0 collisions:0 txqueuelen:0
          RX bytes:18944 (18.9 KB)
          TX bytes:8137789 (8.1 MB)
          veth9c02e56 Link encap:Ethernet  HWaddr 52:81:0b:24:3d:da
          inet6 addr: fe80::5081:bff:fe24:3dda/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:288 errors:0 dropped:0 overruns:0 frame:0
          TX packets:371 errors:0 dropped:0 overruns:0 carrier:0 collisions:0 txqueuelen:0
          RX bytes:21608 (21.6 KB)
          TX bytes:8137719 (8.1 MB)
```

nginx-1 容器对应的 Veth Pair 设备，在宿主机上是一张虚拟网卡，它的名字叫作 veth9c02e56。通过 `brctl show` 命令，可以看到这张网卡被“插”在了 docker0 上。

```bash
$ brctl show
bridge name    bridge id            STP enabled    interfaces
docker0        8000.0242d8e4dfc1    no             veth9c02e56
```



之后，再于这台宿主机上启动另一个 Docker 容器 nginx-2，就会发现一个新的、名为 vethb4963f3 的虚拟网卡，也被“插”在了 docker0 网桥上。

```bash
$ docker run –d --name nginx-2 nginx
$ brctl show
bridge name    bridge id            STP enabled    interfaces
docker0        8000.0242d8e4dfc1    no             veth9c02e56
												   vethb4963f3
```

此时，如果在 nginx-1 容器里 `ping` nginx-2 容器的 IP 地址 172.17.0.3，就会发现同一宿主机上的两个容器是默认相通的。



### 通信流程

上述现象的原理其实非常简单。当用户在 nginx-1 容器里访问 nginx-2 容器的 IP 地址，比如执行 `ping 172.17.0.3` 指令时，该目的 IP 地址会匹配到 nginx-1 容器里的第二条路由规则。而这条路由规则的**网关（Gateway）**是 0.0.0.0，这就意味着这是一条直连规则，即凡是匹配到这条规则的 IP 包，应该经过本机（这里指容器）的 eth0 网卡，通过二层网络直接发往目的主机。

而要通过二层网络到达 nginx-2 容器，就需要有 `172.17.0.3` 目的 IP 地址对应的 MAC 地址。所以 nginx-1 容器的网络协议栈，会通过 eth0 网卡发送一个 ARP 广播（ARP - Address Resolution Protocol：通过三层的 IP 地址找到其对应的二层 MAC 地址的协议），来通过 IP 地址查找对应的 MAC 地址。



前面提到过，nginx-1 容器内的 eth0 网卡是一个 Veth Pair 设备的一端，它的另一端位于宿主机上（Host Namespace），并且被“插”在了宿主机的 docker0 网桥上。

而一旦一张虚拟网卡被“插”在网桥上，它就会变成该网桥的“从设备”。从设备会被“剥夺”调用网络协议栈处理数据包的资格，**从而“降级”成为网桥上的一个端口**。而这个端口唯一的作用，就是接收流入的数据包，然后把这些数据包的“生杀大权”（转发或丢弃），全部交给对应的网桥来处理。

所以，在收到这些 ARP 请求之后，docker0 网桥就会扮演二层交换机的角色，**把 ARP 广播转发到其他被“插”在 docker0 上的虚拟网卡上**。这样，同样连接在 docker0 上的 nginx-2 容器的网络协议栈就会收到这个 ARP 请求并响应，从而将 `172.17.0.3` 所对应的 MAC 地址回复给了 nginx-1 容器。



有了这个目的 MAC 地址，nginx-1 容器的 eth0 网卡就可以将数据包发出去。根据 Veth Pair 设备的原理，这个数据包会立刻出现在宿主机上的 veth9c02e56 虚拟网卡上。不过，此时 veth9c02e56 网卡调用网络协议栈的资格已经被“剥夺”，所以这个数据包就直接流入到了 docker0 网桥里。

docker0 则继续处理转发的过程，扮演二层交换机的角色。此时，docker0 网桥根据数据包的目的 MAC 地址，也就是 nginx-2 容器的 MAC 地址，在它的 **CAM 表（交换机通过 MAC 地址学习维护的端口和 MAC 地址的对应表）**里查到对应的端口为 vethb4963f3，然后将数据包发往这个端口。

而这个端口正是 nginx-2 容器“插”在 docker0 网桥上的另一块虚拟网卡，当然，它也是一个 Veth Pair 设备。这样，数据包就进入到了 nginx-2 容器的 Network Namespace 里。所以，nginx-2 容器看到的情况是，它自己的 eth0 网卡上出现了流入的数据包。这样，nginx-2 的网络协议栈就会对请求进行处理，最后将响应返回到 nginx-1。



同宿主机上的不同容器间通过 docker0 网桥进行通信的流程可总结如下。

![image-20210616171527082](./img/image-20210616171527082.png)

在熟悉了 docker0 网桥的工作方式后，不难理解，在默认情况下，被限制在 Network Namespace 里的容器进程，实际上是通过 Veth Pair 设备 + 宿主机网桥的方式，实现了跟同其他容器的数据交换。



类似的，当在一台宿主机上，访问该宿主机上的容器的 IP 地址时，这个请求的数据包，也是先根据路由规则到达 docker0 网桥，然后被转发到对应的 Veth Pair 设备，最后出现在容器里。

![image-20210617091604611](./img/image-20210617091604611.png)



同样地，当一个容器试图连接到另外一个宿主机时，如 `ping 10.168.0.3`，它发出的请求数据包，首先经过 docker0 网桥出现在宿主机上。然后根据宿主机的路由表里的直连路由规则（10.168.0.0/24 via eth0），对 `10.168.0.3` 的访问请求就会交给宿主机的 eth0 处理。

接下来，这个数据包就会经宿主机的 eth0 网卡转发到宿主机网络上，最终到达 10.168.0.3 对应的宿主机上。当然，这个过程的前提条件是互相通信的两台宿主机其网络本身是相通的。

![image-20210617092014803](./img/image-20210617092014803.png)



## 跨主通信

接着上述最后一个场景，可以联想到这样一个问题：如果在另外一台宿主机，如 `10.168.0.3`上也有一个 Docker 容器。那么，nginx-1 容器该如何访问它？这个问题，其实就是容器的跨主通信问题。



在 Docker 的默认配置下，一台宿主机上的 docker0 网桥，和其他宿主机上的 docker0 网桥，是没有任何关联的，它们互相之间也没办法连通。所以，连接在这些网桥上的容器，自然也没办法进行通信。

有这样一个思路，可以通过软件的方式，创建一个整个集群“公用”的网桥，然后将集群内所有的容器都连接到这个网桥上，从而实现互相通信。这时，集群内的容器网络就会类似于下图。

![image-20210617092850458](./img/image-20210617092850458.png)

可见，构建这种容器网络的核心在于：需要在已有的宿主机网络上，再通过软件构建一个覆盖在已有宿主机网络之上的，可以把所有容器连通在一起的虚拟网络。这种技术被称为：**Overlay Network（覆盖网络）**。自然的，实现容器的跨主通信也还有很多其他的方法。

