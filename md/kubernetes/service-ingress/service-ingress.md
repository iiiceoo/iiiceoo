# Service 与 Ingress

>Nginx 经常被用来为应用提供**统一的访问入口**与**负载均衡能力**，Kubernetes 中通过 *Service* + *Ingress* 机制实现了类似的效果，完成**服务发现**与**反向代理**。



## 需求来源

 *Service* 是 Kubernetes 中非常重要的服务对象。为什么需要服务发现，亦或者说 Kubernetes 为何需要 *Service*？

1. ***Pod* IP 不固定**：

   传统的应用部署一般会指定服务器，用户可以通过明确的 **IP 地址**去访问其他服务器；与传统的应用部署不同，Kubernetes  集群中的应用均由 *Pod* 部署， *Pod* 的生命周期是短暂的。

   在 *Pod* 的生命周期过程中，**它的创建或销毁都伴随着其 IP 地址的变化**，仍以传统方式中指定 IP 的方式去访问应用的相关实例，肯定是不妥的。

   很自然的会联想到其实上述问题非常适合于 DNS 的应用场景。Kubernetes 也正是这么做的，其内部的 DNS 服务（CoreDNS）会随着相关资源（如 *Pod*，*Service*）的创建或销毁来动态的修改地址记录，以确保在 Kubernetes 集群内部每个 *Pod* 均能通过域名被正确访问。

2. **Pod 间负载均衡的需求**：

   *Deployment* 的应用部署模式需要创建一组 *Pod*，*Service* 为这些 *Pod* 提供一个统一的访问入口与负载均衡能力。具体的，Kubernetes 通过 kube-proxy 组件来为每个 *Node* 动态的生成相关 iptables 或 IPVS 规则来实现负载。



## Service 基础

### ClusterIP

先来看一个简单的 *Service* 定义。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hostnames
spec:
  selector:
    app: hostnames
  ports:
  - name: default
    protocol: TCP
    port: 80
    targetPort: 9376
```

其中，`selector` 字段用来声明该 *Service* 仅代理携带了 `app=hostnames` 标签的 *Pod*，规则是 *Service* 的 80 端口，代理 *Pod* 的 9376 端口。这里被代理的应用 *Deployment* 定义如下。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hostnames
spec:
  selector:
    matchLabels:
      app: hostnames
  replicas: 3
  template:
    metadata:
      labels:
        app: hostnames
    spec:
      containers:
      - name: hostnames
        image: k8s.gcr.io/serve_hostname
        ports:
        - containerPort: 9376
          protocol: TCP
```

其中 serve_hostname 是一个特殊的镜像，它会在 9376 端口每次被访问时，返回自己的 hostname。



而被 `selector` 选中的 *Pod*，就称为 *Service* 的 *Endpoints*。

```bash
$ kubectl get endpoints hostnames
NAME        ENDPOINTS
hostnames   10.244.0.5:9376,10.244.0.6:9376,10.244.0.7:9376
```

需要注意的是，只有处于 Running 状态，且 readinessProbe 检查通过的 *Pod*，才会出现在 *Service* 的 *Endpoints* 列表中。当某个 *Pod* 出现问题时，Kubernetes 会自动把它从 *Endpoints* 中摘除，避免流量被打到一个无效的后端负载。



查看该 *Service* 的信息，可见分配给该 *Service* 的 CLUSTER-IP（VIP）为 10.0.1.175。

```bash
$ kubectl get svc hostnames
NAME        TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
hostnames   ClusterIP   10.0.1.175   <none>        80/TCP    5s
```



通过 *Service* 访问其代理的 *Pod*。

```bash
$ curl 10.0.1.175:80
hostnames-0uton

$ curl 10.0.1.175:80
hostnames-yp2kp

$ curl 10.0.1.175:80
hostnames-bvc05
```

像这样通过三次连续不断地访问 *Service* 的 VIP 地址和代理端口 80，依次返回了三个 *Pod* 的 hostname，恰好印证了 *Service* 以**轮询（Round Robin）**的方式负载。当然，这种形式的 *Service* 就被称为 ClusterIP。



### iptables

事实上，*Service* 是由 kube-proxy 组件，通过 iptables 来实现的。

对于上文中创建的名为 hostnames 的 *Service* 来说，一旦它被提交给 Kubernetes，那么 kube-proxy 就可以通过 *Service* 的 Informer 感知到这样一个 *Service* 对象的添加。而作为对这个事件的响应，它就会在宿主机上创建这样一条 iptables 规则。

```iptables
-A KUBE-SERVICES -d 10.0.1.175/32 -p tcp -m comment --comment "default/hostnames: cluster IP" -m tcp --dport 80 -j KUBE-SVC-NWV5X2332I4OT4T3
```

这条 iptables 规则的含义是：凡是目的地址是 10.0.1.175，目的端口是 80 的 IP 包，都应该跳转到另外一条名为 KUBE-SVC-NWV5X2332I4OT4T3 的 iptables 链进行处理。

而 10.0.1.175 正是这个 *Service* 的 VIP。所以这条 iptables 规则，就为这个 *Service* 设置了一个固定的入口地址。并且，由于 10.0.1.175 仅仅是一条 iptables 规则上的配置，并没有真正的网络设备，**所以 `ping` 这个地址，是不会有任何响应的**。



而即将跳转到的 KUBE-SVC-NWV5X2332I4OT4T3 链，实际上是一组随机模式（–mode random）的 iptables 链。

```iptables
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-WNBA2IHDGP2BOBGZ
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-X3P2623AGDH6CDF3
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -j KUBE-SEP-57KPRZ3JQVENLNBR
```

随机转发的目的地分别是 KUBE-SEP-WNBA2IHDGP2BOBGZ，KUBE-SEP-X3P2623AGDH6CDF3 和 KUBE-SEP-57KPRZ3JQVENLNBR，而这三条链指向的最终目的地，其实就是这个 *Service* 所代理的三个 *Pod*。所以这一组规则就是 *Service* 实现负载均衡的逻辑。

需要注意的是，**iptables 规则的匹配是从上到下逐条进行的**，所以为了保证上述三条规则每条被选中的概率都相同，应该将它们的 `probability` 字段的值分别设置为 **1/3**，**1/2** 和 **1**。



最后，来看看“最终目的地”的三条链的规则明细。

```iptables
-A KUBE-SEP-57KPRZ3JQVENLNBR -s 10.244.3.6/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-57KPRZ3JQVENLNBR -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.3.6:9376

-A KUBE-SEP-WNBA2IHDGP2BOBGZ -s 10.244.1.7/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-WNBA2IHDGP2BOBGZ -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.1.7:9376

-A KUBE-SEP-X3P2623AGDH6CDF3 -s 10.244.2.3/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-X3P2623AGDH6CDF3 -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.2.3:9376
```

可以看到，这三条链其实是三条 DNAT 规则。但在 DNAT 规则之前，iptables 对流入的 IP 包还设置了一个“标志”（–set-xmark）。

而 DNAT 规则的作用，就是在 PREROUTING 检查点之前，也就是在路由之前，将流入 IP 包的目的地址和端口改成 `–to-destination` 所指定的新的目的地址和端口。这个目的地址和端口，正是被代理 *Pod* 的 IP 地址和端口。

这样，访问 *Service* VIP 的 IP 包经过上述 iptables 规则处理之后，就已经变成了访问具体某一个后端 *Pod* 的 IP 包了。不难理解，这些 *Endpoints* 对应的 iptables 规则，正是 kube-proxy 通过监听 *Pod* 的变化事件，**在各个宿主机上生成并维护的**。



### IPVS

在了解了 *Service* 有关 iptables 的基本工作原理后，其实 Kubernetes 的 kube-proxy 还支持 IPVS 模式。

kube-proxy 通过 iptables 处理 *Service* 的过程，其实是需要在宿主机上设置相当多的 iptables 规则的。而且，kube-proxy 还需要在控制循环中不断的刷新这些规则来确保它们始终是正确的。

不难想到，当宿主机上存在大量 *Pod* 的时候，成百上千条 iptables 规则不断地被刷新，会大量占用宿主机的 CPU 资源。一直以来，基于 iptables 的 *Service* 实现，都是制约 Kubernetes 项目承载更多量级 *Pod* 的主要障碍。而 IPVS 模式的 *Service*，就是解决这个问题的一个行之有效的方法。



IPVS 模式的工作原理，其实跟 iptables 模式类似。当 *Service* 被创建之后，kube-proxy 首先会在宿主机上创建一张虚拟网卡（kube-ipvs0），并为它分配 *Service* 的 VIP 作为其 IP 地址。

```bash
$ ip addr
...
73：kube-ipvs0：<BROADCAST,NOARP>  mtu 1500 qdisc noop state DOWN qlen 1000
    link/ether  1a:ce:f5:5f:c1:4d brd ff:ff:ff:ff:ff:ff
    inet 10.0.1.175/32  scope global kube-ipvs0
        valid_lft forever  preferred_lft forever
```



接下来，kube-proxy 就会通过 Linux 的 LVS 模块，为这个 VIP 设置一个 IPVS 的 virtual server，然后，为该 virtual server 创建对应的 real server，并且三个 real server 之间使用轮询的方式来作为负载均衡的策略。可通过 `ipvsadm` 命令查看相关配置。

```bash
$ ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
	->  RemoteAddress:Port           Forward  Weight ActiveConn InActConn
TCP  10.102.128.4:80 rr
	->  10.244.3.6:9376				 Masq     1       0          0
	->  10.244.1.7:9376    			 Masq     1       0          0
	->  10.244.2.3:9376    			 Masq     1       0          0
```

三个 real server 的 IP 地址和端口，对应的正是三个被代理的 *Pod*。

而相比于 iptables，IPVS 在内核中的实现其实也是基于 Netfilter 的 NAT 模式，所以在转发这一层上，理论上 IPVS 并没有显著的性能提升。但是，IPVS 并不需要在宿主机上为每个 *Pod* 设置 iptables 规则，而是把对这些“规则”的处理放到了内核态，这极大地降低了维护这些规则所需的代价。



### Headless Service

Kubernetes 中，*Service* 和 *Pod* 都会被分配对应的 DNS A 记录。对于 ClusterIP 模式的 *Service* 来说，比如上文中的例子，它的 A 记录的格式是：`<svc-name>.<namespace>.svc.cluster.local`。当访问这条 A 记录的时候，其实解析到的就是该 *Service* 的 VIP。

除了 ClusterIP 模式的 *Service*，还有一种 *Service* 称为 *Headless Service*。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hostnames
spec:
  selector:
    app: hostnames
  clusterIP: None
  ports:
  - name: default
    protocol: TCP
    port: 80
    targetPort: 9376
```

可以看到，所谓的 *Headless Service*，其实与 ClusterIP 模式的 *Service* 几乎没有差异，区别它的 `clusterIP` 字段为 `None`，即：该 *Service* 没有一个 VIP 作为“头”，这就是 Headless 的含义。所以，这个 *Service* 被创建后并不会被分配一个 VIP，而是会直接以 DNS A 记录的方式暴露出它所代理的 *Pod*。

当按照这样的方式创建了一个 *Headless Service* 后，它所代理的所有 *Pod*，都会被绑定一个这种格式的 DNS A 记录：`<pod-name>.<svc-name>.<namespace>.svc.cluster.local`，这个 DNS A 记录，正是 Kubernetes 为 *Pod* 分配的唯一“身份证”。

*Headless Service* 也可以通过 *Service* 的 DNS A 记录：`<svc-name>.<namespace>.svc.cluster.local` 来访问，这时所返回的就是所有被代理的 *Pod* 的 IP 地址的集合。当然，如果客户端没办法解析这个集合的话，它可能会只会拿到第一个 *Pod* 的 IP 地址。



## 暴露 Service

大致了解了 Service 机制的基本原理后，很容易发现这样一个事实：*Service* 的访问信息在 Kubernetes 集群之外，其实是无效的。

很好理解，所谓 *Service* 的访问入口，其实就是每台宿主机上由 kube-proxy 生成与维护的 iptables 规则，以及 CoreDNS 生成的 DNS A 记录。一旦脱离了这个集群，这些信息对用户而言，自然就没有作用了，其他人可没有这些 iptables 规则，也不认识这些的域名。



### NodePort

一个必须要解决的问题就是：如何从 Kubernetes 集群之外，访问到 Kubernetes 内创建的 *Service*。

一个最常见的方式：NodePort。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  type: NodePort
  ports:
  - nodePort: 8080
    targetPort: 80
    protocol: TCP
    name: http
  - nodePort: 443
    protocol: TCP
    name: https
  selector:
    run: my-nginx
```

在这个 *Service* 的定义中，它的类型被声明为 `type=NodePort`。然后在 `ports` 字段里定义了 *Service* 的 8080 端口代理 *Pod* 的 80 端口，*Service* 的 443 端口代理 *Pod* 的 443 端口。

当然，如果不显式地声明 `nodePort` 字段，Kubernetes 会随机分配可用端口来设置代理。这个端口的范围默认是 30000 - 32767，可以通过 API Server 的 `--service-node-port-range` 参数来修改它。



可以这样访问这个 *Service*，就能访问到某一个被代理 *Pod* 的 80 端口了。

```url
<任何一台宿主机的 IP 地址>:8080
```

NodePort 的工作模式也非常简单。kube-proxy 会在每台宿主机上生成这样一条 iptables 规则。

```iptables
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/my-nginx: nodePort" -m tcp --dport 8080 -j KUBE-SVC-67RL4FN6JRUPOJYM
```

这条 iptables 规则的含义是：凡是目的端口是 8080 的 IP 包，都应该跳转到另外一条名为 KUBE-SVC-67RL4FN6JRUPOJYM 的 iptables 链进行处理。而 KUBE-SVC-67RL4FN6JRUPOJYM 其实就是一组随机模式的 iptables 规则。接下来的流程，就与 ClusterIP 模式无二了。

值得注意的是，上述 NodePort 其实会使用掉宿主机上的 8080 端口，如果大量的 *Service* 都采用 NodePort 的方式来暴露，那么大量的端口会被消耗掉。



### LoadBalancer

从外部访问 *Service* 的第二种方式，多于公有云上的 Kubernetes 服务。一个 LoadBalancer 类型的 *Service* 定义如下。

```yaml
kind: Service
apiVersion: v1
metadata:
  name: example-service
spec:
  ports:
  - port: 8765
    targetPort: 9376
  selector:
    app: example
  type: LoadBalancer
```



各大公有云服务提供商都会去实现自己的**云控制管理器（Cloud Controller Manager，CCM）**组件。它指的是嵌入特定云中控制平面的相关控制逻辑，可以理解为公有云服务提供商为自己的商业产品（如 LB）所实现的一个控制器，它可以处理与平台强相关的控制循环以对接到公有云的各种商业服务。这种机制使得不同的云厂商都能很好的与 Kubernetes 集成。

来看看 LoadBalancer 的架构图。

![image-20210616104127972](./img/image-20210616104127972.png)

可以看到 CCM 监听了 *Service* 的相关事件，在上述 LoadBalancer 类型的 *Service* 被提交后，CCM 则会调用公有云相关接口，为用户创建一个负载均衡服务，并且把被代理的 *Pod* 的 IP 地址配置给负载均衡服务做后端。

通俗来说，就是云厂商帮用户起了个 Nginx，并按照用户提交的 *Service* 动态生成了相关 `location` 和 `upstream` 配置，并且这个 Nginx 是可以在 Kubenetes 集群外部被访问的。值得注意的是，公有云提供的负载均衡服务肯定是要大米的。



LoadBalancer 的另一种实现 [MetalLB](https://github.com/metallb/metallb)。



### ExternalName

最后一种方式，是 Kubernetes 在 1.7 之后支持的一个特性，ExternalName。

```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  type: ExternalName
  externalName: my.database.example.com
```

上述 *Service* 中，指定了一个 `externalName=my.database.example.com` 的字段。特别的，这个 *Service* 中是无 `selector` 字段的。

这时，当通过 *Service* 的 DNS A 记录 `my-service.default.svc.cluster.local` 访问它时，Kubernetes 会返回  `my.database.example.com`。所以说，ExternalName 类型的 *Service*，其实是在 CoreDNS 中添加了一条 CNAME 记录。这时，访问 `my-service.default.svc.cluster.local` 就和访问 `my.database.example.com` 域名是同一个效果。

ExternalName 模式其实是摈弃内部机制，依赖外部设施，比如某个用户想要自己实现负载均衡的服务。这时，*Service* 就会与一个域名一一对应起来，而整个负载均衡的工作都全全交由外部实现了。



## Ingress

简单的了解 *Service* 暴露的三种方法后，不难感受到，其实仅有 LoadBalancer 模式比较适合于生产。但在实际的场景下，这个做法既浪费成本又高。

用户其实更希望看到 Kubernetes 为用户内置一个全局的负载均衡器（7 层）。然后，通过访问的 URL，把请求转发给不同的后端 *Service*。为了描述这种全局的，为了代理不同后端 *Service* 而设置的负载服务，就出现了 *Ingress*。

*Ingress* 的功能其实非常容易理解：**所谓 *Ingress*，就是 *Service* 的 “*Service*”**。



来看这样一个例子，假如现在有这样一个站点：`https://cafe.example.com`。

- `https://cafe.example.com/coffee`：咖啡点餐系统。
- `https://cafe.example.com/tea`：茶水点餐系统。

这两个系统，分别由名为 coffee 和 tea 这样两个 *Deployment* 来提供服务。



那么现在，如何使用 *Ingress* 来创建一个统一的负载均衡器，从而实现当用户访问不同的域名时，就可以访问到不同的 *Deployment* ？可以这样定义这个 *Ingress*。

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cafe-ingress
spec:
  tls:
  - hosts:
    - cafe.example.com
    secretName: cafe-secret
  rules:
  - host: cafe.example.com
    http:
      paths:
      - path: /tea
        backend:
          serviceName: tea-svc
          servicePort: 80
      - path: /coffee
        backend:
          serviceName: coffee-svc
          servicePort: 80
```

上述 *Ingress* 中最值得注意的就是 `rules` 字段。Kubernetes 中，这个字段称为：IngressRule。

IngressRule 的键称为 `host`。它必须是一个符合**标准的域名格式（Fully Qualified Domain Name）**的字符串，不能是 IP 地址。`host` 字段定义的值，就是这个 *Ingress* 的入口。这也就意味着，当用户访问 `cafe.example.com` 的时候，实际上访问到的就是这个 *Ingress* 对象。这样，Kubernetes 就能使用 IngressRule 来对用户的请求进行下一步转发。

而接下来 IngressRule 规则的定义，则依赖于 `path` 字段。可以简单地理解为，这里的每一个 `path` 都对应一个后端 *Service*。所以在这个例子中定义了两个 `path`，它们分别对应 coffee 和 tea 这两个 *Deployment* 的 *Service*。



通过上述的讲解，不难看到，**所谓 Ingress 对象，其实就是 Kubernetes 项目对“反向代理”的一种抽象**。

一个 *Ingress* 对象的主要内容，实际上就是一个“**反向代理**”服务**配置文件的描述**，好比 Nginx。而这个代理服务对应的转发规则，就是 IngressRule。

*Ingress* 中通过 `host` 字段作为入口， `path` 字段来声明具体的转发策略，非常类似 Nginx 的 `listen` 与 `location` 关键字。而被 *Ingress* 反向代理的 *Service* 中的各个 *Pod* 负载，就可以类比 Nginx `upstream` 配置中的 `server` 关键字。写法都是相通的。



有了 *Ingress* 这样一个统一的抽象，Kubernetes 的用户就无需关心 *Ingress* 的具体细节了。在实际的使用中，用户只需要从社区里选择一个具体的 Ingress Controller，把它部署在 Kubernetes 集群里即可。

然后，这个 Ingress Controller 会根据用户所定义的 *Ingress* 对象，提供对应的代理能力。目前，业界常用的各种反向代理项目，如 Nginx，HAProxy，Envoy，Traefik 等，都已经为 Kubernetes 专门维护了对应的 Ingress Controller。



来看看 Nginx 官方所维护的 Ingress Controller 资源清单的定义。

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        ...
    spec:
      serviceAccountName: nginx-ingress-serviceaccount
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.20.0
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -> 33
            runAsUser: 33
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
            - name: http
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
```

它定义了一个使用 nginx-ingress-controller 镜像的 *Pod*，这个 *Pod* 本身，就是一个监听 *Ingress* 以及它所代理的后端 *Service* 变化的控制器。

当一个新的 *Ingress* 由用户创建后，nginx-ingress-controller 就会根据 *Ingress* 内定义的内容，生成一份对应的 **Nginx 配置文件（/etc/nginx/nginx.conf）**，并使用这个配置文件启动一个 Nginx 服务。同时，一旦 *Ingress* 被更新，nginx-ingress-controller 就会动态的 `reload` 这份配置文件。

此外，nginx-ingress-controller 还允许通过 *ConfigMap* 来对上述 Nginx 配置文件进行定制。该 *ConfigMap* 的名字，需要以参数的方式传递给 nginx-ingress-controller。而在这个 *ConfigMap* 里添加的字段，将会被合并到最后生成的 Nginx 配置文件当中。



当然，为了让用户能够用到这个 Nginx，还需要创建一个 *Service* 来把 Nginx Ingress Controller 管理的 Nginx 服务暴露出去。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
    - name: https
      port: 443
      targetPort: 443
      protocol: TCP
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
```

可以看到，这个 *Service* 的唯一工作，就是将所有携带 `ingress-nginx` 标签的 *Pod* 的 80 和 433 端口暴露出去了。这里没有显示的来指定 `NodePort` 字段，所以 kubernetes 会自动分配端口。

```bash
$ kubectl get svc -n ingress-nginx
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   NodePort   10.105.72.96   <none>        80:30044/TCP,443:31453/TCP   3h
```

可见 30044 和 31453  即为 NodePort。接下来就可以通过 `<任何一台宿主机的 IP 地址>:<NodePort>` 来访问这个 Nginx 了。


