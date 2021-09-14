# 容器探针

>在 Kubernetes 中，用户可以为 *Pod* 里的容器定义一个用于健康检查的**“探针”（Probe）**。进而，kubelet 会根据这个 Probe 的返回值来决定这个容器的状态。Kubernetes 即通过这种检测机制来确保容器的健康。



## 基本概念

Probe 是由 kubelet 调用由容器实现的 Handler 处理程序对容器执行的定期诊断。 目前有三类 Handler：

- **ExecAction**： 在容器内执行指定命令。如果命令退出时返回码为 0 则认为诊断成功。
- **TCPSocketAction**： 对容器的 IP 地址上的指定端口执行 TCP 检查。如果端口打开，则诊断被认为是成功的。
- **HTTPGetAction**： 对容器的 IP 地址上指定端口和路径执行 HTTP `GET` 请求。如果响应的状态码大于等于 `200` 且小于 `400`，则诊断被认为是成功的。

每次探测都将获得以下三种结果之一：

- `Success`（成功）：容器通过了诊断。
- `Failure`（失败）：容器未通过诊断。
- `Unknown`（未知）：诊断失败，因此不会采取任何行动。



针对运行中的容器，kubelet 可以选择是否执行以下三种类型的探针，以及如何针对探测结果作出反应：

- `livenessProbe`：指示容器是否正在运行。若存活探测失败，kubelet 会立即杀死容器， 接下来容器则根据所属 *Pod* 的重启策略决定其下一步的动作。当容器不提供存活态探针时， 默认状态为 `Success`。
- `readinessProbe`：指示容器是否准备好为请求提供服务。若就绪探测失败， **端点控制器（Endpoints  Controller）**将从与 *Pod* 匹配的所有 *Service* 的 *Endpoints* 列表中删除该 *Pod* 的 IP 地址。 注意，初始延迟之前的就绪态状态值为 `Failure`，当如，如果容器不提供就绪态探针，则默认状态为 `Success`。
- `startupProbe`: 指示容器中的应用是否已经启动。**如果提供了启动探针，则所有其他探针都会被禁用，直到此探针成功**。同样的，若启动探测失败，kubelet 将杀死容器，容器依其 *Pod* 重启策略进行重启。 若容器没有提供启动探测，默认状态为 `Success`。



## 探测器 / 探针

### livenessProbe

livenessProbe 通常被称为**存活态探针**，它通常被用来探测一个 *Pod* 中的某个容器是否在正常的运行。如果用户允许容器中的进程在遇到问题或不健康的情况下自行崩溃，其实就不一定需要存活态探针，此时 kubelet 将根据 *Pod* 的 `restartPolicy` 字段所指定的重启策略自动执行修复动作。

*Pod* 可指定的恢复策略如下：

- `Always`：在任何情况下，只要容器不在运行状态，就自动重启容器。
- `OnFailure`：只在容器异常时才自动重启容器。
- `Never`：从来不重启容器。

当然，如果用户希望容器在探测失败时被杀死并重新启动，那么请务必指定一个存活态探针。 同时，设置 *Pod* 的 `restartPolicy` 字段为 `Always` 或 `OnFailure`。



来看一个 Kubernetes 文档中的例子。

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: test-liveness-exec
spec:
  containers:
  - name: liveness
    image: busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

其中，`periodSeconds` 字段指定了 kubelet 应该每 5 秒执行一次存活探测。 `initialDelaySeconds` 字段告诉 kubelet 在执行第一次探测前应该等待 5 秒，等待容器先成功运行起来。`command` 字段则要求 kubelet 在容器内执行 `cat /tmp/healthy` 命令来进行探测。 如果命令执行成功并且返回值为 0，kubelet 就会认为这个容器是健康存活的；反之，kubelet 会杀死这个容器并重新启动它。

注意，当容器启动时，它会立马执行如下命令。也就是说，30s 之后，它删除了作为自己正常运行的”标志“，healthy 文件。

```bash
/bin/sh -c "touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600"
```



来具体实践一下这个过程。首先，创建这个 *Pod*。

```bash
kubectl create -f test-liveness-exec.yaml
```



查看这个 *Pod* 的状态。可以看到，由于已经通过了健康检查，这个 *Pod* 进入了 Running 状态。

```bash
kubectl get pod
```

```yaml
NAME                READY     STATUS    RESTARTS   AGE
test-liveness-exec  1/1       Running   0          10s
```



30 s 之后，再次查看 *Pod* 的事件。显然，这个健康检查探查到 /tmp/healthy 文件已经不存在了，所以它报告容器是不健康的。

```bash
kubectl describe pod test-liveness-exec
```

```bash
FirstSeen LastSeen    Count   From            SubobjectPath           Type        Reason      Message
--------- --------    -----   ----            -------------           --------    ------      -------
2s        2s      1   {kubelet worker0}   spec.containers{liveness}   Warning     Unhealthy   Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
```



接下来，再次查看这个 *Pod* 的状态时，会发现此时 ***Pod* 并没有进入 Failed 状态，而是保持了 Running 状态**。

```bash
kubectl get pod test-liveness-exec
```

```bash
NAME           READY     STATUS    RESTARTS   AGE
liveness-exec  1/1       Running   1          1m
```

其实，如果注意到 `RESTARTS` 字段从 0 到 1 的变化，就能立马明白原因了，**这个异常的容器其实已经被 Kubernetes 重启了**。

需要注意的是，Kubernetes 并没有 Docker 中的 Stop 语义。所以虽然说是 Restart（重启）了容器，但实际上是把容器杀了重建，这就是在上文中提到的 *Pod* 恢复机制。其中，*Pod* 的 `restartPolicy` 字段默认会被设置为 `Always`，所以这个名为 liveness 的容器才会在刚刚”发生异常“的时候被重新创建。



需要强调的是，这个 *Pod* 的恢复过程，**是会永远发生在当前节点上的，而不会跑到别的节点上去**。事实上，一旦一个 *Pod* 与某个 *Node* 绑定后，除非这个绑定关系发生了变化，即 `pod.spec.node` 字段被修改，否则它永远都不会离开这个节点。这也就意味着，如果这个宿主机宕机了，这个 *Pod* 也不会主动迁移到其他节点上去。

而如果用户想让 *Pod* 在上述情况中“主动迁移”到其他的可用节点上，就必须使用像 *Deployment* 这样的控制器来管理 *Pod*，哪怕只需要一个 *Pod* 副本。这也正是为什么在生产环境中，开发者常常更倾向于去部署一个单 *Pod* 的 *Deployment* ，而并非单独部署一个不被控制器管理的 *Pod*。



当然，除了在容器中执行命令的探测方式外，livenessProbe 也可以定义为发起 HTTP 或者 TCP 请求。

```yaml
   ...
   livenessProbe:
     httpGet:
       path: /healthz
       port: 8080
       httpHeaders:
       - name: X-Custom-Header
         value: Awesome
     initialDelaySeconds: 3
     periodSeconds: 3
```

```yaml
    ...
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
```

所以，*Pod* 其实可以暴露一个健康检查 URL，如 `/healthz`，或者直接让健康检查去检测应用的监听端口。这两种配置方式，在 Web 服务类应用中非常常见。



### readinessProbe

readinessProbe 被称为就绪态探针，通常 kubelet 使用 readinessProbe 来确认**容器什么时候准备好了并可以开始接受请求流量**， 当一个 *Pod* 内的所有容器都准备好了，这个 *Pod* 就被认为就绪。 这种信号的主要用途就是控制哪些 *Pod* 能作为 *Service* 的后端 *Endpoints*。 当 *Pod* 还未准备好时，Kubernetes 会把它从 *Service* 的 *Endpoints* 中剔除。



readinessProbe 的配置和 livenessProbe 的配置类似。

```yaml
    ...
    readinessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

同理，HTTP 和 TCP 方式的 readinessProbe 配置也和 livenessProbe 的配置类似。

需要注意的是，readinessProbe 与 livenessProbe 可以在同一个容器上并行使用。两者均可确保流量不会发给还未准备好的容器，同时，容器会在它们失败的时候被重新创建。



### startupProbe

在某些场景下，**应用程序在启动时需要较多的初始化时间**，这时再使用 livenessProbe 就会不太合适，因为需要预估一个比较大的 `initialDelaySeconds` 参数来确保容器正常启动，显然，这个行为是非常不稳定的：**没人能够每次都猜对这个需要较长时间初始化的容器到底得花多久才能正常启动**。

在上述场景下，一旦定义的 `initialDelaySeconds` 参数过小，则有概率发生探测动作的死锁：即容器还未正常运行起来，就被  livenessProbe 判定为不健康，被干掉后又根据 *Pod*  恢复策略被重新拉起，以此循环，”反复鞭尸“。所以，Kubernetes v1.16 针对这个问题引入了一个新的探测器：即**启动态探针（startupProbe）**。



可以这样来定义 startupProbe。

```yaml
   ...
   startupProbe:
     httpGet:
       path: /healthz
       port: 8080
     failureThreshold: 30
     periodSeconds: 10
```

现在这个容器就有 `failureThreshold * periodSeconds` （30 * 10 = 300）秒的时间来启动，期间每隔 10s 发起一次探测。**一旦启动探测成功一次，livenessProbe 就会接管对容器的探测**；如果启动探测一直没有成功，容器则会在 5 分钟后被杀死，并且根据 `restartPolicy` 来决定接下来的 *Pod* 状态。

