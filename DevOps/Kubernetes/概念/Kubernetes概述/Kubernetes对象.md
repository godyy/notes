本文说明了 Kubernetes 对象在 Kubernetes API 中是如何表示的，以及如何在 `.yaml` 格式的文件中表示。

[TOC]



# 1. 理解 Kubernetes 对象

在 Kubernetes 系统中，*Kubernetes 对象* 是持久化的实体。 Kubernetes 使用这些实体去表示整个集群的状态。特别地，它们描述了如下信息：

- 哪些容器化应用在运行（以及在哪些节点上）
- 可以被应用使用的资源
- 关于应用运行时表现的策略，比如重启策略、升级策略，以及容错策略

Kubernetes 对象是 “目标性记录” —— 一旦创建对象，Kubernetes 系统将持续工作以确保对象存在。 通过创建对象，本质上是在告知 Kubernetes 系统，所需要的集群工作负载看起来是什么样子的， 这就是 Kubernetes 集群的 **期望状态（Desired State）**。

操作 Kubernetes 对象 —— 无论是创建、修改，或者删除 —— 需要使用 [Kubernetes API](https://kubernetes.io/zh/docs/concepts/overview/kubernetes-api)。 比如，当使用 `kubectl` 命令行接口时，CLI 会执行必要的 Kubernetes API 调用， 也可以在程序中使用 [客户端库](https://kubernetes.io/zh/docs/reference/using-api/client-libraries/)直接调用 Kubernetes API。

## 1.1 对象规约（Spec）与状态（Status）

几乎每个 Kubernetes 对象包含两个嵌套的对象字段，它们负责管理对象的配置： 对象 *`spec`（规约）* 和 对象 *`status`（状态）* 。 对于具有 `spec` 的对象，你必须在创建对象时设置其内容，描述你希望对象所具有的特征： *期望状态（Desired State）* 。

`status` 描述了对象的 *当前状态（Current State）*，它是由 Kubernetes 系统和组件 设置并更新的。在任何时刻，Kubernetes [控制平面](https://kubernetes.io/zh/docs/reference/glossary/?all=true#term-control-plane) 都一直积极地管理着对象的实际状态，以使之与期望状态相匹配。

## 1.2 描述 Kubernetes 对象

创建 Kubernetes 对象时，必须提供对象的规约，用来描述该对象的期望状态， 以及关于对象的一些基本信息（例如名称）。 当使用 Kubernetes API 创建对象时（或者直接创建，或者基于`kubectl`）， API 请求必须在请求体中包含 JSON 格式的信息。 **大多数情况下，需要在 .yaml 文件中为 `kubectl` 提供这些信息**。 `kubectl` 在发起 API 请求时，将这些信息转换成 JSON 格式。

这里有一个 `.yaml` 示例文件，展示了 Kubernetes Deployment 的必需字段和对象规约：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```



使用类似于上面的 `.yaml` 文件来创建 Deployment的一种方式是使用 `kubectl` 命令行接口（CLI）中的 [`kubectl apply`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#apply) 命令， 将 `.yaml` 文件作为参数。下面是一个示例：

```
kubectl apply -f https://k8s.io/examples/application/deployment.yaml --record
```

输出类似这样:

```
deployment.apps/nginx-deployment created
```

### 必需字段

在想要创建的 Kubernetes 对象对应的 `.yaml` 文件中，需要配置如下的字段：

- `apiVersion` - 创建该对象所使用的 Kubernetes API 的版本
- `kind` - 想要创建的对象的类别
- `metadata` - 帮助唯一性标识对象的一些数据，包括一个 `name` 字符串、UID 和 可选的`namespace`

你也需要提供对象的 `spec` 字段。 对象 `spec` 的精确格式对每个 Kubernetes 对象来说是不同的，包含了特定于该对象的嵌套字段。 [Kubernetes API 参考](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.21/) 能够帮助我们找到任何我们想创建的对象的 spec 格式。

# 2. Kubernetes 对象管理

`kubectl` 命令行工具支持多种不同的方式来创建和管理 Kubernetes 对象。 本文档概述了不同的方法。 阅读 [Kubectl book](https://kubectl.docs.kubernetes.io/) 来了解 kubectl 管理对象的详细信息。

## 2.1 管理技巧

> **警告：**
>
> 应该只使用一种技术来管理 Kubernetes 对象。混合和匹配技术作用在同一对象上将导致未定义行为。

| 管理技术       | 作用于   | 建议的环境 | 支持的写者 | 学习难度 |
| -------------- | -------- | ---------- | ---------- | -------- |
| 指令式命令     | 活跃对象 | 开发项目   | 1+         | 最低     |
| 指令式对象配置 | 单个文件 | 生产项目   | 1          | 中等     |
| 声明式对象配置 | 文件目录 | 生产项目   | 1+         | 最高     |

## 2.2 指令式命令

使用指令式命令时，用户可以在集群中的活动对象上进行操作。用户将操作传给 `kubectl` 命令作为参数或标志。

这是开始或者在集群中运行一次性任务的推荐方法。因为这个技术直接在活跃对象 上操作，所以它不提供以前配置的历史记录。

### 例子

通过创建 Deployment 对象来运行 nginx 容器的实例：

```sh
kubectl create deployment nginx --image nginx
```

### 权衡

与对象配置相比的优点：

- 命令简单，易学且易于记忆。
- 命令仅需一步即可对集群进行更改。

与对象配置相比的缺点：

- 命令不与变更审查流程集成。
- 命令不提供与更改关联的审核跟踪。
- 除了实时内容外，命令不提供记录源。
- 命令不提供用于创建新对象的模板。

## 2.3 指令式对象配置

在指令式对象配置中，kubectl 命令指定操作（创建，替换等），可选标志和 至少一个文件名。指定的文件必须包含 YAML 或 JSON 格式的对象的完整定义。

有关对象定义的详细信息，请查看 [API 参考](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.21/)。

> **警告：**
>
> `replace` 指令式命令将现有规范替换为新提供的规范，并放弃对配置文件中 缺少的对象的所有更改。此方法不应与对象规约被独立于配置文件进行更新的 资源类型一起使用。比如类型为 `LoadBalancer` 的服务，它的 `externalIPs` 字段就是独立于集群配置进行更新。

### 例子

创建配置文件中定义的对象：

```
kubectl create -f nginx.yaml
```

删除两个配置文件中定义的对象：

```
kubectl delete -f nginx.yaml -f redis.yaml
```

通过覆盖活动配置来更新配置文件中定义的对象：

```
kubectl replace -f nginx.yaml
```

### 权衡

与指令式命令相比较的优点：

- 对象配置可以控制在源控制系统中，比如 Git。
- 对象配置可以与流程继承，例如在推送和审计之前检查更新。
- 对象配置提供了用于创建新对象的模版。

与指令式命令相比的缺点：

- 对象配置需要对对象架构有基本的了解。
- 对象配置需要额外的步骤来编写 YAML 文件。

与声明式对象配置相比的优点：

- 指令式对象配置行为更加简单易懂。
- 从 Kubernetes 1.5 版本开始，指令对象配置更加成熟。

与声明式对象配置相比的趣点：

- 指令式对象配置更适合文件，而非目录。
- 对活动对象的更新必需反应在配置文件中，否则会在下一次替换时丢失。

## 2.4 声明式对象配置

使用声明式对象配置时，用户对本地存储的对象配置文件进行操作，但是用户 未定义要对该文件执行的操作。 `kubectl` 会自动检测每个文件的创建、更新和删除操作。 这使得配置可以在目录上工作，根据目录中配置文件对不同的对象执行不同的操作。

> **说明：**
>
> 声明式对象配置保留其他编写者所做的修改，即使这些更改并未合并到对象配置文件中。 可以通过使用 `patch` API 操作仅写入观察到的差异，而不是使用 `replace` API 操作来替换整个对象配置来实现。

### 例子

处理 `configs` 目录中的所有对象配置文件，创建并更新活跃对象。 可以首先使用 `diff` 子命令查看将要进行的更改，然后在进行应用：

```
# 查看将要进行的更改
kubectl diff -f configs/

kubectl apply -f configs/
```

递归处理目录：

```
kubectl diff -R f configs/
kubectl apply -R f configs/
```

### 权衡

与指令式对象配置相比的优点：

- 对活动对象所做的更改即使未合并到配置文件中，也会被保留下来。
- 声明性对象配置更好地支持对目录进行操作并自动检测每个文件的操作类型（创建，修补，删除）。

与指令式对象配置相比的缺点：

- 声明式对象配置难于调试并且出现异常时结果难以理解。
- 使用 diff 产生的部分更新会创建复杂的合并和补丁操作。

# 3. 对象名称和IDs

集群中的每一个对象都有一个[*名称*](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/names/#names) 来标识在同类资源中的唯一性。

每个 Kubernetes 对象也有一个[*UID*](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/names/#uids) 来标识在整个集群中的唯一性。

比如，在同一个[名字空间](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/namespaces/) 中有一个名为 `myapp-1234` 的 Pod, 但是可以命名一个 Pod 和一个 Deployment 同为 `myapp-1234`.

对于用户提供的非唯一性的属性，Kubernetes 提供了 [标签（Labels）](https://kubernetes.io/zh/docs/concepts/working-with-objects/labels)和 [注解（Annotation）](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/annotations/)机制。

## 3.1 名称

客户端提供的字符串，引用资源 url 中的对象，如`/api/v1/pods/some name`。

某一时刻，只能有一个给定类型的对象具有给定的名称。但是，如果删除该对象，则可以创建同名的新对象。

> **说明：**
>
> 当对象所代表的是一个物理实体（例如代表一台物理主机的 Node）时， 如果在 Node 对象未被删除并重建的条件下，重新创建了同名的物理主机， 则 Kubernetes 会将新的主机看作是老的主机，这可能会带来某种不一致性（没有举例？）。

以下是比较常见的三种资源命名约束。

### DNS 子域名

很多资源类型需要可以用作 DNS 子域名的名称。 DNS 子域名的定义可参见 [RFC 1123](https://tools.ietf.org/html/rfc1123)。 这一要求意味着名称必须满足如下规则：

- 不能超过253个字符
- 只能包含小写字母、数字，以及'-' 和 '.'
- 须以字母数字开头
- 须以字母数字结尾

### DNS 标签名

某些资源类型需要其名称遵循 [RFC 1123](https://tools.ietf.org/html/rfc1123) 所定义的 DNS 标签标准。也就是命名必须满足如下规则：

- 最多63个字符
- 只能包含小写字母、数字，以及'-'
- 须以字母数字开头
- 须以字母数字结尾

### 路径分段名称

某些资源类型要求名称能被安全地用作路径中的片段。 换句话说，其名称不能是 `.`、`..`，也不可以包含 `/` 或 `%` 这些字符。

下面是一个名为`nginx-demo`的 Pod 的配置清单：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-demo
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

> **说明：** 某些资源类型可能具有额外的命名约束。

## 3.2 UIDs

Kubernetes 系统生成的字符串，唯一标识对象。

在 Kubernetes 集群的整个生命周期中创建的每个对象都有一个不同的 uid，它旨在区分类似实体的历史事件。

Kubernetes UIDs 是全局唯一标识符（也叫 UUIDs）。

# 4. 名字空间（虚拟集群）

Kubernetes 支持多个虚拟集群，它们底层依赖于同一个物理集群。 这些虚拟集群被称为名字空间。

## 4.1 何时使用多个名字空间

名字空间适用于存在很多跨多个团队或项目的用户的场景。对于只有几到几十个用户的集群，根本不需要创建或考虑名字空间。当需要名称空间提供的功能时，请开始使用它们。

名字空间为名称提供了一个范围。资源的名称需要在名字空间内是唯一的，但不能跨名字空间。 名字空间不能相互嵌套，每个 Kubernetes 资源只能在一个名字空间中。

名字空间是在多个用户之间划分集群资源的一种方法（通过[资源配额](https://kubernetes.io/zh/docs/concepts/policy/resource-quotas/)）。

不必使用多个名字空间来分隔仅仅轻微不同的资源，例如同一软件的不同版本： 应该使用[标签](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/labels/) 来区分同一名字空间中的不同资源。

## 4.2 使用名字空间

名字空间的创建和删除在[名字空间的管理指南文档](https://kubernetes.io/zh/docs/tasks/administer-cluster/namespaces/)描述。

> **说明：** 避免使用前缀 `kube-` 创建名字空间，因为它是为 Kubernetes 系统名字空间保留的。

### 查看名字空间

你可以使用以下命令列出集群中现存的名字空间：

```shell
kubectl get namespace
NAME          STATUS    AGE
default       Active    1d
kube-node-lease   Active   1d
kube-system   Active    1d
kube-public   Active    1d
```

Kubernetes 会创建四个初始名字空间：

- `default` 没有指明使用其它名字空间的对象所使用的默认名字空间
- `kube-system` Kubernetes 系统创建对象所使用的名字空间
- `kube-public` 这个名字空间是自动创建的，所有用户（包括未经过身份验证的用户）都可以读取它。 这个名字空间主要用于集群使用，以防某些资源在整个集群中应该是可见和可读的。 这个名字空间的公共方面只是一种约定，而不是要求。
- `kube-node-lease` 此名字空间用于与各个节点相关的租期（Lease）对象； 此对象的设计使得集群规模很大时节点心跳检测性能得到提升。

### 为请求设置名字空间

要为当前请求设置名字空间，请使用 `--namespace` 参数。

例如：

```shell
kubectl run nginx --image=nginx --namespace=<名字空间名称>
kubectl get pods --namespace=<名字空间名称>
```

### 设置名字空间偏好

你可以永久保存名字空间，以用于对应上下文中所有后续 kubectl 命令。

```shell
kubectl config set-context --current --namespace=<名字空间名称>
# 验证之
kubectl config view | grep namespace:
```

## 4.3 名字空间和 DNS

当你创建一个[服务](https://kubernetes.io/zh/docs/concepts/services-networking/service/) 时， Kubernetes 会创建一个相应的 [DNS 条目](https://kubernetes.io/zh/docs/concepts/services-networking/dns-pod-service/)。

该条目的形式是 `<服务名称>.<名字空间名称>.svc.cluster.local`，这意味着如果容器只使用 `<服务名称>`，它将被解析到本地名字空间的服务。这对于跨多个名字空间（如开发、分级和生产） 使用相同的配置非常有用。如果你希望跨名字空间访问，则需要使用完全限定域名（FQDN）。

## 4.4 并非所有对象都在名字空间中

大多数 kubernetes 资源（例如 Pod、Service、副本控制器等）都位于某些名字空间中。 但是名字空间资源本身并不在名字空间中。而且底层资源，例如 [节点](https://kubernetes.io/zh/docs/concepts/architecture/nodes/) 和持久化卷不属于任何名字空间。

查看哪些 Kubernetes 资源在名字空间中，哪些不在名字空间中：

```shell
# 位于名字空间中的资源
kubectl api-resources --namespaced=true

# 不在名字空间中的资源
kubectl api-resources --namespaced=false
```

## 4.5 自动打标签

**FEATURE STATE:** `Kubernetes 1.21 [beta]`

Kubernetes 控制面会为所有名字空间设置一个不可变更的 [标签](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/labels/) `kubernetes.io/metadata.name`，只要 `NamespaceDefaultLabelName` 这一 [特性门控](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/feature-gates/) 被启用。标签的值是名字空间的名称。

