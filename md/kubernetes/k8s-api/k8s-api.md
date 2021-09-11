# Kubernetes API

>REST API 是 Kubernetes 的基本结构。 Kubernetes 所有的**操作**与**系统组件间的通信**以及**外部用户命令**本质都是调用 API Server 处理的 REST API。Kubernetes 中万物皆 API 对象。



## 通用概念

### API 版本控制

不同的 API 版本代表着不同的稳定性和支持级别。下面为各级别的摘要。

- **Alpha**
  1. 版本名称包含 `alpha`，如 `v1alpha1`。
  2. 软件可能会有 Bug。某些特性可能默认禁用。
  3. 对某个特性的支持可能会随时被删除。
  4. API 可能在以后的软件版本中以不兼容的方式更改。
  5. 缺陷长期支持，建议仅用于短期测试集群。
  
- **Beta**

  1. 版本名称包含 `beta`，如， `v2beta3`。
  2. 软件被很好的测试过。启用某个特性被认为是安全的，特性默认开启。
  3. 长期支持。

- **Stable**

  1. 版本名称如 `vX`，其中 `X` 为整数，如 `v1`。
  2. 特性的稳定版本会出现在后续很多版本的发布软件中。



### API 组

API 组能够简化对 Kubernetes API 的扩展。 API 组信息出现在 REST 路径中，也出现在序列化对象的 `apiVersion` 字段中。API 组可大致分为两类。

- **核心组**（也称 *legacy*）：其 REST 路径为 `/api/vi`。同时，核心组并不作为 `apiVersion` 字段的一部分，比如在资源清单中见的非常多的 `apiVersion: v1`。*Pod*，*Node*，*Volume*，*Service* 等，都是常见的核心组资源。
- **普通组**：不同于核心组的最大特点无分组，普通组即具有分组信息的组。其 REST 路径为 `/apis/$GROUP_NAME/$VERSION`，同时在序列化对象中使用 `apiVersion: $GROUP_NAME/$VERSION` ，如 `apiVersion: batch/v1`。



来看看 Kubernetes 项目下的 pkg 目录（存放 Kubernetes API 对象的目录）。

![image-20210610113145622](./img/image-20210610113145622.png)

API 组和 API 版本的作用一目了然，v1 或 v1beta1 文件夹下就是相应版本的 API 对象的代码了。这种设计使得 Kubernetes API 的扩展变得非常容易。



### 标准 API 术语

大多数 Kubernetes API 资源类型都是对象：它们代表的是集群中某一概念的具体实例，例如一个 *Pod* 或 *Namespace*。

Kubernetes 一般会利用标准的 RESTful 术语来描述 API 概念：

- **资源类型（Resource Type）**是在 URL 中使用的名称（`pods`，`namespaces`，`services` 等）。
- 所有资源类型都有具有一个 JSON 形式（其对象的模式定义）的具体表示，称作**类别（Kind）**。用户可以通过编写 Yaml 格式的文件来定义和描述想要操作的资源，也就是常说的**资源清单**。
- 某资源类型的实例的列表称作**集合（Collection）**。
- 资源类型的单个实例被称作**资源（Resource）**。



所有资源类型要么是集群作用域的（`/apis/GROUP/VERSION/*`），要么是名字空间作用域的（`/apis/GROUP/VERSION/namespaces/NAMESPACE/*`）。名字空间作用域的资源类型会在其名字空间被删除时也被删除。

一个简单例子，可用下列路径来检视集合和资源：

- **集群作用域的资源**：
  1. `GET /apis/GROUP/VERSION/RESOURCETYPE` - 返回指定资源类型的资源的集合。
  2. `GET /apis/GROUP/VERSION/RESOURCETYPE/NAME` - 返回指定资源类型下名称为 NAME 的资源。
- **名字空间作用域的资源**：
  1. `GET /apis/GROUP/VERSION/RESOURCETYPE` - 返回所有名字空间中指定资源类型的全部实例的集合。
  2. `GET /apis/GROUP/VERSION/namespaces/NAMESPACE/RESOURCETYPE` - 返回名字空间 NAMESPACE 内给定资源类型的全部实例的集合。
  3. `GET /apis/GROUP/VERSION/namespaces/NAMESPACE/RESOURCETYPE/NAME` - 返回名字空间 NAMESPACE 中给定资源类型的名称为 NAME 的实例。

几乎所有对象资源类型都支持标准的 HTTP 动词（`GET`，`PUT`，`POST`，`PATCH` 和 `DELETE`）来进行检视，创建，更新和删除。同时，Kubernetes 中使用术语 LIST 来描述返回资源集合的操作，以便与返回单个资源的，通常称作 GET 的操作相区分。



Kubernetes API 实际上就是基于资源的，通过 HTTP 提供的编程接口。Kubernetes 平台的任何能力，其实都是通过一个个的 API 对象所表现出来的，开发者能做的就是通过 Kubernetes API 去操作，调整这些 API 对象，然后以控制循环的方式来使整个集群不断的趋向于期望状态。



[Kubernetes API 文档（v1.21）](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.21/#-strong-api-overview-strong-)就展示出了非常鲜明的声明式 API 特点。相较于传统用的命令式 API 文档，它更像是一份各类 API 对象的字典，其中详细的记录了每个 API 对象的各个字段及其效果。

![image-20210610142715097](./img/image-20210610142715097.png)




### 客户端库

在使用 Kubernetes REST API 编写应用程序时，可以根据开发者的编程语言习惯选择使用合适的客户端库。

| 语言   | 客户端库                                                     | 样例程序                                                     |
| ------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| ⭐ Go   | [github.com/kubernetes/client-go/](https://github.com/kubernetes/client-go/) | [🔗](https://github.com/kubernetes/client-go/tree/master/examples) |
| Python | [github.com/kubernetes-client/python/](https://github.com/kubernetes-client/python/) | [🔗](https://github.com/kubernetes-client/python/tree/master/examples) |
| Java   | [github.com/kubernetes-client/java](https://github.com/kubernetes-client/java/) | [🔗](https://github.com/kubernetes-client/java#installation)  |



## HTTP REST API 访问

Kubernetes 集群搭建之后，除了使用官方的 kubectl 工具与 API Server 进行交互，还可以使用 Postman 工具或者 curl 命令。

而与 API Server 交互通常需要一个有正确权限的 *ServiceAccount*，该 *ServiceAccount* 通过 *ClusterRole*/*Role*，*ClusterRoleBinding*/*RoleBinding* 给用户赋予相关资源的操作权限，*ServiceAccount* 对应的 Token 则被用于 API Server 进行基本的认证。



### curl

首先，创建一个名为 `apiviewer` 的 *ServiceAccount*。

```bash
kubectl create sa apiviewer
```

![image-20210610145552385](./img/image-20210610145552385.png)



查看这个 *ServiceAccount* 对应的 *Secret* 名称。

```bash
kubectl get sa/apiviewer -o yaml
```

![image-20210610145747058](./img/image-20210610145747058.png)



查看这个 *Secret* 的数据。

```bash
kubectl get secret/apiviewer-token-7hcs7 -o yaml
```

![image-20210610150337360](./img/image-20210610150337360.png)



创建一个 *ClusterRoleBinding*，将系统预留角色 cluster-admin 分配给 apiviewer。

```bash
kubectl create clusterrolebinding apiadmin --clusterrole cluster-admin --serviceaccount default:apiviewer
```

![image-20210610150747247](./img/image-20210610150747247.png)



获取 *Secret* 中的 Bearer Token。

```bash
TOKEN=$(kubectl get secret/apiviewer-token-7hcs7 -o jsonpath='{.data.token}' | base64 -d)
```

然后将证书也提取出来。

```bash
kubectl get secret/apiviewer-token-7hcs7 -o jsonpath="{.data['ca\.crt']}" | base64 -d > ca.crt
```

![image-20210610151626577](./img/image-20210610151626577.png)



获取 API Server URL。

```bash
APISERVER=https://$(kubectl get endpoints/kubernetes --no-headers | awk '{ print $2 }' | cut -d "," -f 1)
```

![image-20210610151913559](./img/image-20210610151913559.png)



现在就可以通过 `curl` 命令查询 default *namespace* 下的所有 *Pod* 了。

```bash
curl -s $APISERVER/api/v1/namespaces/default/pods/ \
--header "Authorization: Bearer $TOKEN" \
--cacert ca.crt
```

![image-20210610152323701](./img/image-20210610152323701.png)





### Postman

如果要使用 Postman 工具访问 API Server，过程其实与 curl 大同小异。首先，将之前 *Secret* 中提取出的证书拷贝至 windows。双击 .crt 文件，信任该证书。

![image-20210610154021113](./img/image-20210610154021113.png)



在 Postman 工具中新建一个标签，输入 API Server 的请求路径。点击下方的 Authorization，选择 Bearer Token 的 Type，然后在右侧的 Token 中输入之前获取到的 *Secret* 中的 Token。

![image-20210610154555280](./img/image-20210610154555280.png)



请求 API Server，可以看到请求头中已经添加上了刚设置的 Authorization 参数，且请求成功响应。

![image-20210610154818744](./img/image-20210610154818744.png)


