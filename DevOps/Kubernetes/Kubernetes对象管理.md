官方文档链接：https://kubernetes.io/zh/docs/tasks/manage-kubernetes-objects/

[TOC]

使用构建在 `kubectl` 命令行工具中的指令式命令可以直接快速创建、更新和删除 Kubernetes 对象。本文档解释这些命令的组织方式以及如何使用它们来管理现时对象。

# 1. 指令式命令

## 1.1 创建对象

`kubectl` 工具支持动词驱动的命令，用来创建一些最常见的对象类别。 命令的名称设计使得不熟悉 Kubernetes 对象类型的用户也能做出判断。

- `run`：创建一个新的 Pod 来运行一个容器。
- `expose`：创建一个新的 Service 对象为若干 Pod 提供流量负载均衡。
- `autoscale`：创建一个新的 Autoscaler 对象来自动对某控制器（如 Deployment） 执行水平扩缩。

`kubectl` 命令也支持一些对象类型驱动的创建命令。 这些命令可以支持更多的对象类别，并且在其动机上体现得更为明显，不过要求 用户了解它们所要创建的对象的类别。

- `create <对象类别> [<子类别>] <实例名称>`

某些对象类别拥有自己的子类别，可以在 `create` 命令中设置。 例如，Service 对象有 ClusterIP、LoadBalancer 和 NodePort 三种子类别。 下面是一个创建 NodePort 子类别的 Service 的示例：

```
kubectl create service nodeport <服务名称>
```

在前述示例中，`create service nodeport` 命令也称作 `create service` 命令的子命令。 可以使用 `-h` 标志找到一个子命令所支持的参数和标志。

```
kubectl create service nodeport -h
```

## 1.2 更新对象

`kubectl` 命令也支持一些动词驱动的命令，用来执行一些常见的更新操作。 这些命令的设计是为了让一些不了解 Kubernetes 对象的用户也能执行更新操作， 但不需要了解哪些字段必须设置：

- `scale`：对某控制器进行水平扩缩以便通过更新控制器的副本个数来添加或删除 Pod。
- `annotate`：为对象添加或删除注解。
- `label`：为对象添加或删除标签。

`kubectl` 命令也支持由对象的某一方面来驱动的更新命令。 设置对象的这一方面可能对不同类别的对象意味着不同的字段：

- `set <字段>`：设置对象的某一方面。

`kubectl` 工具支持以下额外的方式用来直接更新现时对象，不过这些操作要求 用户对 Kubernetes 对象的模式定义有很好的了解：

- `edit`：通过在编辑器中打开现时对象的配置，直接编辑其原始配置。
- `patch`：通过使用补丁字符串（Patch String）直接更改某现时对象的的特定字段。 关于补丁字符串的更详细信息，参见 [API 约定](https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#patch-operations) 的 patch 节。

## 1.3 删除对象

你可以使用 `delete` 命令从集群中删除一个对象：

- `delete <类别>/<名称>`

你可以使用 `kubectl delete` 来执行指令式命令或者指令式对象配置。不同之处在于 传递给命令的参数。要将 `kubectl delete` 作为指令式命令使用，将要删除的对象作为 参数传递给它。下面是一个删除名为 `nginx` 的 Deployment 对象的命令：

```shell
kubectl delete deployment/nginx
```

## 1.4 查看对象

用来打印对象信息的命令有好几个：

- `get`：打印匹配到的对象的基本信息。使用 `get -h` 可以查看选项列表。
- `describe`：打印匹配到的对象的详细信息的汇集版本。
- `logs`：打印 Pod 中运行的容器的 stdout 和 stderr 输出。

## 1.5 使用 set 命令再创建对象之前修改对象

有些对象字段在 `create` 命令中没有对应的标志。在这些场景中， 你可以使用 `set` 和 `create` 命令的组合来在对象创建之前设置字段值。 这是通过将 `create` 命令的输出用管道方式传递给 `set` 命令来实现的， 最后执行 `create` 命令来创建对象。下面是一个例子：

```sh
kubectl create service clusterip my-svc --clusterip="None" -o yaml --dry-run=client | kubectl set selector --local -f - 'environment=qa' -o yaml | kubectl create -f -
```

1. 命令 `kubectl create service -o yaml --dry-run=client` 创建 Service 的配置，但 将其以 YAML 格式在标准输出上打印而不是发送给 API 服务器。
2. 命令 `kubectl set selector --local -f - -o yaml` 从标准输入读入配置，并将更新后的 配置以 YAML 格式输出到标准输出。
3. 命令 `kubectl create -f -` 使用标准输入上获得的配置创建对象。

## 1.6 在创建之前使用 --edit 更改对象

你可以用 `kubectl create --edit` 来在对象被创建之前执行任意的变更。 下面是一个例子：

```sh
kubectl create service clusterip my-svc --clusterip="None" -o yaml --dry-run=client > /tmp/srv.yaml
kubectl create --edit -f /tmp/srv.yaml
```

1. 命令 `kubectl create service` 创建 Service 的配置并将其保存到 `/tmp/srv.yaml` 文件。
2. 命令 `kubectl create --edit` 在创建 Service 对象打开其配置文件进行编辑。

# 2. 使用配置文件对 Kubernetes 对象进行命令式管理

可以使用 `kubectl` 命令行工具以及用 YAML 或 JSON 编写的对象配置文件来创建、更新和删除 Kubernetes 对象。 本文档说明了如何使用配置文件定义和管理对象。

## 2.1 创建对象

你可以使用 `kubectl create -f` 从配置文件创建一个对象。 请参考 [kubernetes API 参考](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.21/) 有关详细信息。

- `kubectl create -f <filename|url>`

## 2.2 更新对象

> **警告：** 使用 `replace` 命令更新对象会删除所有未在配置文件中指定的规范的某些部分。 不应将其规范由集群部分管理的对象使用，比如类型为 `LoadBalancer` 的服务， 其中 `externalIPs` 字段独立于配置文件进行管理。 必须将独立管理的字段复制到配置文件中，以防止 `replace` 删除它们。

你可以使用 `kubectl replace -f` 根据配置文件更新活动对象。

- `kubectl replace -f <filename|url>`

## 2.3 删除对象

你可以使用 `kubectl delete -f` 删除配置文件中描述的对象。

- `kubectl delete -f <filename|url>`

> **说明：**
>
> 如果配置文件在 `metadata` 节中设置了 `generateName` 字段而非 `name` 字段， 你无法使用 `kubectl delete -f <filename|url>` 来删除该对象。 你必须使用其他标志才能删除对象。例如：
>
> ```shell
> kubectl delete <type> <name>
> kubectl delete <type> -l <label>
> ```

## 2.4 查看对象

你可以使用 `kubectl get -f` 查看有关配置文件中描述的对象的信息。

- `kubectl get -f <filename|url> -o yaml`

`-o yaml` 标志指定打印完整的对象配置。 使用 `kubectl get -h` 查看选项列表。

## 2.5 局限性

当完全定义每个对象的配置并将其记录在其配置文件中时，`create`、 `replace` 和`delete` 命令会很好的工作。 但是，当更新一个活动对象，并且更新没有合并到其配置文件中时，下一次执行 `replace` 时，更新将丢失。 如果控制器,例如 HorizontalPodAutoscaler ,直接对活动对象进行更新，则会发生这种情况。 这有一个例子：

1. 从配置文件创建一个对象。
2. 另一个源通过更改某些字段来更新对象。
3. 从配置文件中替换对象。在步骤2中所做的其他源的更改将丢失。

如果需要支持同一对象的多个编写器，则可以使用 `kubectl apply` 来管理该对象。

## 2.6 从 URL 创建和编辑对象而不保存配置

假设你具有对象配置文件的 URL。 你可以在创建对象之前使用 `kubectl create --edit` 对配置进行更改。 这对于指向可以由读者修改的配置文件的教程和任务特别有用。

```shell
kubectl create -f <url> --edit
```

## 2.7 从命令式命令迁移到命令式对象配置

从命令式命令迁移到命令式对象配置涉及几个手动步骤。

1. 将活动对象导出到本地对象配置文件：

   ```
   kubectl get <kind>/<name> -o yaml > <kind>_<name>.yaml
   ```

2. 从对象配置文件中手动删除状态字段。

3. 对于后续的对象管理，只能使用`replace`。

   ```
   kubectl replace -f <kind>_<name>.yaml
   ```

## 2.8 定义控制器选择器和 PodTemplate 标签

> **警告：** 不建议在控制器上更新选择器。

推荐的方法是定义单个不变的 PodTemplate 标签，该标签仅由控制器选择器使用，而没有其他语义。

标签示例：

```yaml
selector:
  matchLabels:
      controller-selector: "apps/v1/deployment/nginx"
template:
  metadata:
    labels:
      controller-selector: "apps/v1/deployment/nginx"
```

# 3. 使用配置文件对 Kubernetes 对象进行声明式管理

你可以通过在一个目录中存储多个对象配置文件、并使用 `kubectl apply` 来递归地创建和更新对象来创建、更新和删除 Kubernetes 对象。 这种方法会保留对现有对象已作出的修改，而不会将这些更改写回到对象配置文件中。 `kubectl diff` 也会给你呈现 `apply` 将作出的变更的预览。

## 3.1 概览

声明式对象管理需要用户对 Kubernetes 对象定义和配置有比较深刻的理解。

以下是本节内容中使用的术语的定义：

- 对象配置/配置文件：一个定义 Kubernetes 对象的配置的文件。配置文件通常存储于类似 Git 这种源码控制系统中。
- 现时对象配置/现时配置：由 Kubernetes 集群所观测到的对象的现时配置值。这些配置保存在 Kubernetes 集群存储（通常是 etcd）中。
- 声明式配置写者/声明式写者：负责更新现时对象的人或者软件组件。

## 3.2 创建对象

使用 `kubectl apply` 来创建指定目录中配置文件所定义的所有对象，除非对应对象已经存在：

```shell
kubectl apply -f <目录>/
```

此操作会在每个对象上设置 `kubectl.kubernetes.io/last-applied-configuration: '{...}'` 注解。注解值中包含了用来创建对象的配置文件的内容。

> **说明：** 添加 `-R` 标志可以递归地处理目录。

下面是一个对象配置文件示例：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  minReadySeconds: 5
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

执行 `kubectl diff` 可以打印出将被创建的对象：

```
kubectl diff -f https://k8s.io/examples/application/simple_deployment.yaml
```

> **说明：**
>
> `diff` 使用[服务器端试运行（Server-side Dry-run）](https://kubernetes.io/zh/docs/reference/using-api/api-concepts/#dry-run) 功能特性；而该功能特性需要在 `kube-apiserver` 上启用。
>
> 由于 `diff` 操作会使用试运行模式执行服务器端 apply 请求，因此需要为 用户配置 `PATCH`、`CREATE` 和 `UPDATE` 操作权限。 参阅[试运行授权](https://kubernetes.io/zh/docs/reference/using-api/api-concepts#dry-run-authorization) 了解详情。

使用 `kubectl apply` 来创建对象：

```
kubectl apply -f https://k8s.io/examples/application/simple_deployment.yaml
```

使用 `kubectl get` 打印其现时配置：

```shell
kubectl get -f https://k8s.io/examples/application/simple_deployment.yaml -o yaml
```

输出显示注解 `kubectl.kubernetes.io/last-applied-configuration` 被写入到 现时配置中，并且其内容与配置文件相同：

```
kind: Deployment
metadata:
  annotations:
    # ...
    # This is the json representation of simple_deployment.yaml
    # It was written by kubectl apply when the object was created
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment",
      "metadata":{"annotations":{},"name":"nginx-deployment","namespace":"default"},
      "spec":{"minReadySeconds":5,"selector":{"matchLabels":{"app":nginx}},"template":{"metadata":{"labels":{"app":"nginx"}},
      "spec":{"containers":[{"image":"nginx:1.14.2","name":"nginx",
      "ports":[{"containerPort":80}]}]}}}}      
  # ...
spec:
  # ...
  minReadySeconds: 5
  selector:
    matchLabels:
      # ...
      app: nginx
  template:
    metadata:
      # ...
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.14.2
        # ...
        name: nginx
        ports:
        - containerPort: 80
        # ...
      # ...
    # ...
  # ...
```

## 3.3 更新对象

你也可以使用 `kubectl apply` 来更新某个目录中定义的所有对象，即使那些对象已经存在。 这一操作会隐含以下行为：

1. 在现时配置中设置配置文件中出现的字段；
2. 在现实配置中清楚配置文件中已删除的字段。

```
kubectl diff -f <目录>/
kubectl apply -f <目录>/
```

> **说明：** 使用 `-R` 标志递归处理目录。

使用 `kubectl apply` 来创建对象：

```shell
kubectl apply -f https://k8s.io/examples/application/simple_deployment.yaml
```

> **说明：** 出于演示的目的，上面的命令引用的是单个文件而不是整个目录。

使用 `kubectl get` 打印现时配置：

```shell
kubectl get -f https://k8s.io/examples/application/simple_deployment.yaml -o yaml
```

输出显示，注解 `kubectl.kubernetes.io/last-applied-configuration` 被写入到 现时配置中，并且其取值与配置文件内容相同。

通过 `kubectl scale` 命令直接更新现时配置中的 `replicas` 字段。 这一命令没有使用 `kubectl apply`：

```
kubectl scale deployment/nginx-deployment --replicas=2
```

使用 `kubectl get` 来打印现时配置：

```shell
kubectl get deployment nginx-deployment -o yaml
```

输出显示，`replicas` 字段已经被设置为 2，而 `last-applied-configuration` 注解中 并不包含 `replicas` 字段。

现在更新 `simple_deployment.yaml` 配置文件，将镜像文件从 `nginx:1.14.2` 更改为 `nginx:1.16.1`，同时删除`minReadySeconds` 字段：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.16.1 # update the image
        ports:
        - containerPort: 80
```

应用对配置文件所作更改：

```shell
kubectl diff -f https://k8s.io/examples/application/update_deployment.yaml
kubectl apply -f https://k8s.io/examples/application/update_deployment.yaml
```

使用 `kubectl get` 打印现时配置：

```shell
kubectl get -f https://k8s.io/examples/application/update_deployment.yaml -o yaml
```

输出显示现时配置中发生了以下更改：

- 字段 `replicas` 保留了 `kubectl scale` 命令所设置的值：2； 之所以该字段被保留是因为配置文件中并没有设置 `replicas`。
- 字段 `image` 的内容已经从 `nginx:1.14.2` 更改为 `nginx:1.16.1`。
- 注解 `last-applied-configuration` 内容被更改为新的镜像名称。
- 字段 `minReadySeconds` 被移除。
- 注解 `last-applied-configuration` 中不再包含 `minReadySeconds` 字段。

> **警告：** 将 `kubectl apply` 与指令式对象配置命令 `kubectl create` 或 `kubectl replace` 混合使用是不受支持的。这是因为 `create` 和 `replace` 命令都不会保留 `kubectl apply` 用来计算更新内容所使用的 `kubectl.kubernetes.io/last-applied-configuration` 注解值。

## 3.4 删除对象

有两种方法来删除 `kubectl apply` 管理的对象。

### 建议操作：`kubectl delete -f <文件名>`

使用指令式命令来手动删除对象是建议的方法，因为这种方法更为明确地给出了 要删除的内容是什么，且不容易造成用户不小心删除了其他对象的情况。

```shell
kubectl delete -f <文件名>
```

### 替代方式：`kubectl apply -f <目录名称/> --prune -l your=label`

只有在充分理解此命令背后含义的情况下才建议这样操作。

> **警告：** `kubectl apply --prune` 命令本身仍处于 Alpha 状态，在后续发布版本中可能会 引入一些向后不兼容的变化。

> **警告：** 在使用此命令时必须小心，这样才不会无意中删除不想删除的对象。

作为 `kubectl delete` 操作的替代方式，你可以在目录中对象配置文件被删除之后， 使用 `kubectl apply` 来辩识要删除的对象。 带 `--prune` 标志的 `apply` 命令会首先查询 API 服务器，获得与某组标签相匹配 的对象列表，之后将返回的现时对象配置与目录中的对象配置文件相比较。 如果某对象在查询中被匹配到，但在目录中没有文件与其相对应，并且其中还包含 `last-applied-configuration` 注解，则该对象会被删除。

```shell
kubectl apply -f <directory/> --prune -l <labels>
```

> **警告：** 带剪裁（prune）行为的 `apply` 操作应在包含对象配置文件的目录的根目录运行。 如果在其子目录中运行，可能导致对象被不小心删除。 因为某些对象可能与 `-l <标签>` 的标签选择算符匹配，但其配置文件不在当前 子目录下。
>
> 疑问：如果是在根目录，并且存在子目录，不用设置 -R 标志吗？

## 3.5 查看对象

你可以使用 `kubectl get` 并指定 `-o yaml` 选项来查看现时对象的配置：

```shell
kubectl get -f <文件名 | URL> -o yaml
```

## 3.6 apply 操作是如何计算配置差异并合并变更的？

> **注意：** *patch* 是一种更新操作，其作用域为对象的一些特定字段而不是整个对象。 这使得你可以更新对象的特定字段集合而不必先要读回对象。

`kubectl apply` 更新对象的现时配置，它是通过向 API 服务器发送一个 patch 请求 来执行更新动作的。 所提交的补丁中定义了对现时对象配置中特定字段的更新。 `kubectl apply` 命令会使用当前的配置文件、现时配置以及现时配置中保存的 `last-applied-configuration` 注解内容来计算补丁更新内容。

### 3.6.1 合并补丁计算

`kubectl apply` 命令将配置文件的内容写入到 `kubectl.kubernetes.io/last-applied-configuration` 注解中。 这些内容用来识别配置文件中已经移除的、因而也需要从现时配置中删除的字段。 用来计算要删除或设置哪些字段的步骤如下：

1. 计算要删除的字段，即在 `last-applied-configuration` 中存在但在 配置文件中不再存在的字段。
2. 计算要添加或设置的字段，即在配置文件中存在但其取值与现时配置不同的字段。

下面是一个例子。假定此文件是某 Deployment 对象的配置文件：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.16.1 # update the image
        ports:
        - containerPort: 80
```

同时假定同一 Deployment 对象的现时配置如下：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    # ...
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment",
      "metadata":{"annotations":{},"name":"nginx-deployment","namespace":"default"},
      "spec":{"minReadySeconds":5,"selector":{"matchLabels":{"app":nginx}},"template":{"metadata":{"labels":{"app":"nginx"}},
      "spec":{"containers":[{"image":"nginx:1.14.2","name":"nginx",
      "ports":[{"containerPort":80}]}]}}}}      
  # ...
spec:
  replicas: 2
  # ...
  minReadySeconds: 5
  selector:
    matchLabels:
      # ...
      app: nginx
  template:
    metadata:
      # ...
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.14.2
        # ...
        name: nginx
        ports:
        - containerPort: 80
      # ...
```

下面是 `kubectl apply` 将执行的合并计算：

1. 通过读取 `last-applied-configuration` 并将其与配置文件中的值相比较， 计算要删除的字段。 对于本地对象配置文件中显式设置为空的字段，清除其在现时配置中的设置， 无论这些字段是否出现在 `last-applied-configuration` 中。 在此例中，`minReadySeconds` 出现在 `last-applied-configuration` 注解中，但 并不存在于配置文件中。 **动作：** 从现时配置中删除 `minReadySeconds` 字段。
2. 通过读取配置文件中的值并将其与现时配置相比较，计算要设置的字段。 在这个例子中，配置文件中的 `image` 值与现时配置中的 `image` 不匹配。 **动作**：设置现时配置中的 `image` 值。
3. 设置 `last-applied-configuration` 注解的内容，使之与配置文件匹配。
4. 将第 1、2、3 步骤得出的结果合并，构成向 API 服务器发送的补丁请求内容。

### 3.6.2 不同类型字段的合并方式

下面是 `kubectl apply` 将执行的合并计算：

1. 通过读取 `last-applied-configuration` 并将其与配置文件中的值相比较， 计算要删除的字段。 对于本地对象配置文件中显式设置为空的字段，清除其在现时配置中的设置， 无论这些字段是否出现在 `last-applied-configuration` 中。 在此例中，`minReadySeconds` 出现在 `last-applied-configuration` 注解中，但 并不存在于配置文件中。 **动作：** 从现时配置中删除 `minReadySeconds` 字段。
2. 通过读取配置文件中的值并将其与现时配置相比较，计算要设置的字段。 在这个例子中，配置文件中的 `image` 值与现时配置中的 `image` 不匹配。 **动作**：设置现时配置中的 `image` 值。
3. 设置 `last-applied-configuration` 注解的内容，使之与配置文件匹配。
4. 将第 1、2、3 步骤得出的结果合并，构成向 API 服务器发送的补丁请求内容。

官方链接：[各类型字段的具体更新方式](https://kubernetes.io/zh/docs/tasks/manage-kubernetes-objects/)

## 3.7 默认字段值

API 服务器会在对象创建时其中某些字段未设置的情况下在现时配置中为其设置默认值。

API 服务器会在对象创建时其中某些字段未设置的情况下在现时配置中为其设置默认值。

下面是一个 Deployment 的配置文件。文件未设置 `strategy`：

[`application/simple_deployment.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/zh/examples/application/simple_deployment.yaml) ![Copy application/simple_deployment.yaml to clipboard](https://d33wubrfki0l68.cloudfront.net/0901162ab78eb4ff2e9e5dc8b17c3824befc91a6/44ccd/images/copycode.svg)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  minReadySeconds: 5
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

使用 `kubectl apply` 创建对象：

```shell
kubectl apply -f https://k8s.io/examples/application/simple_deployment.yaml
```

使用 `kubectl get` 打印现时配置：

```shell
kubectl get -f https://k8s.io/examples/application/simple_deployment.yaml -o yaml
```

输出显示 API 在现时配置中为某些字段设置了默认值。 这些字段在配置文件中并未设置。

```yaml
apiVersion: apps/v1
kind: Deployment
# ...
spec:
  selector:
    matchLabels:
      app: nginx
  minReadySeconds: 5
  replicas: 1           # API 服务器所设默认值
  strategy:
    rollingUpdate:      # API 服务器基于 strategy.type 所设默认值
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate # API 服务器所设默认值
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.14.2
        imagePullPolicy: IfNotPresent    # API 服务器所设默认值
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP       # API 服务器所设默认值
        resources: {}         # API 服务器所设默认值
        terminationMessagePath: /dev/termination-log    # API 服务器所设默认值
      dnsPolicy: ClusterFirst       # API 服务器所设默认值
      restartPolicy: Always         # API 服务器所设默认值
      securityContext: {}           # API 服务器所设默认值
      terminationGracePeriodSeconds: 30        # API 服务器所设默认值
# ...
```

在补丁请求中，已经设置了默认值的字段不会被重新设回其默认值，除非 在补丁请求中显式地要求清除。对于默认值取决于其他字段的某些字段而言， 这可能会引发一些意想不到的行为。当所依赖的其他字段后来发生改变时， 基于它们所设置的默认值只能在显式执行清除操作时才会被更新。

为此，建议在配置文件中为服务器设置默认值的字段显式提供定义，即使所 给的定义与服务器端默认值设定相同。这样可以使得辩识无法被服务器重新 基于默认值来设置的冲突字段变得容易。

**示例：**

```yaml
# last-applied-configuration
spec:
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

# 配置文件
spec:
  strategy:
    type: Recreate   # 更新的值
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

# 现时配置
spec:
  strategy:
    type: RollingUpdate    # 默认设置的值
    rollingUpdate:         # 基于 type 设置的默认值
      maxSurge : 1
      maxUnavailable: 1
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

# 合并后的结果 - 出错！
spec:
  strategy:
    type: Recreate     # 更新的值：与 rollingUpdate 不兼容
    rollingUpdate:     # 默认设置的值：与 "type: Recreate" 冲突
      maxSurge : 1
      maxUnavailable: 1
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

**解释：**

1. 用户创建 Deployment，未设置 `strategy.type`。
2. 服务器为 `strategy.type` 设置默认值 `RollingUpdate`，并为 `strategy.rollingUpdate` 设置默认值。
3. 用户改变 `strategy.type` 为 `Recreate`。字段 `strategy.rollingUpdate` 仍会取其 默认设置值，尽管服务器期望该字段被清除。 如果 `strategy.rollingUpdate` 值最初于配置文件中定义，则它们需要被清除 这一点就更明确一些。
4. `apply` 操作失败，因为 `strategy.rollingUpdate` 未被清除。 `strategy.rollingupdate` 在 `strategy.type` 为 `Recreate` 不可被设定。

建议：以下字段应该在对象配置文件中显式定义：

- 如 Deployment、StatefulSet、Job、DaemonSet、ReplicaSet 和 ReplicationController 这类负载的选择算符和 `PodTemplate` 标签
- Deployment 的上线策略

### 3.7.1 如何清除服务器端按默认值设置的字段或者被其他写者设置的字段

没有出现在配置文件中的字段可以通过将其值设置为 `null` 并应用配置文件来清除。 对于由服务器按默认值设置的字段，清除操作会触发重新为字段设置新的默认值。

## 3.8 如何将字段的属主在配置文件和直接指令式写者之间切换

更改某个对象字段时，应该采用下面的方法：

- 使用 `kubectl apply`.
- 直接写入到现时配置，但不更改配置文件本身，例如使用 `kubectl scale`。

### 3.8.1 将属主从直接指令式写者更改为配置文件

将字段添加到配置文件。针对该字段，不再直接执行对现时配置的修改。 修改均通过 `kubectl apply` 来执行。

### 3.8.2 将属主从配置文件改为直接指令式写者

在 Kubernetes 1.5 中，将字段的属主从配置文件切换到某指令式写者需要手动 执行以下步骤：

- 从配置文件中删除该字段；
- 将字段从现时对象的 `kubectl.kubernetes.io/last-applied-configuration` 注解 中删除 （`kubectl annotate`）。

## 3.9. 更改管理方法

Kubernetes 对象在同一时刻应该只用一种方法来管理。 从一种方法切换到另一种方法是可能的，但这一切换是一个手动过程。

> **说明：** 在声明式管理方法中使用指令式命令来删除对象是可以的。

### 3.9.1 从指令式命令管理切换到声明式对象配置

从指令式命令管理切换到声明式对象配置管理的切换包含以下几个手动步骤：

1. 将现时对象导出到本地配置文件：

   ```
   kubectl get <kind>/<name> -o yaml > <kind>_<name>.yaml
   ```

2. 手动移除配置文件中的 `status` 字段。

   > **说明：** 这一步骤是可选的，因为 `kubectl apply` 并不会更新 status 字段，即便 配置文件中包含 status 字段。

3. 设置对象上的 `kubectl.kubernetes.io/last-applied-configuration` 注解：

   ```
   kubectl replace --save-config -f <kind>_<name>.yaml
   ```

4. 更改过程，使用 `kubectl apply` 专门管理对象。

### 3.9.2 从指令式对象配置切换到声明式对象配置

1. 在对象上设置 `kubectl.kubernetes.io/last-applied-configuration` 注解：

   ```shell
   kubectl replace -save-config -f <kind>_<name>.yaml
   ```

2. 自此排他性地使用 `kubectl apply` 来管理对象。

## 3.10 定义控制器选择算符和 PodTemplate 标签

> **警告：** 强烈不建议更改控制器上的选择算符。

建议的方法是定义一个不可变更的 PodTemplate 标签，仅用于控制器选择算符且 不包含其他语义性的含义。

**示例：**

```yaml
selector:
  matchLabels:
      controller-selector: "apps/v1/deployment/nginx"
template:
  metadata:
    labels:
      controller-selector: "apps/v1/deployment/nginx"
```

# 4. 使用 kubectl patch 更新 API 对象

这个任务展示如何使用 `kubectl patch` 就地更新 API 对象。 这个任务中的练习演示了一个策略性合并 patch 和一个 JSON 合并 patch。

## 4.1 使用策略合并 patch 更新 Deployment

下面是具有两个副本的 Deployment 的配置文件。每个副本是一个 Pod，有一个容器：

[`application/deployment-patch.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/zh/examples/application/deployment-patch.yaml) ![Copy application/deployment-patch.yaml to clipboard](https://d33wubrfki0l68.cloudfront.net/0901162ab78eb4ff2e9e5dc8b17c3824befc91a6/44ccd/images/copycode.svg)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: patch-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: patch-demo-ctr
        image: nginx
      tolerations:
      - effect: NoSchedule
        key: dedicated
        value: test-team
```

创建 Deployment：

```
kubectl create -f https://k8s.io/examples/application/deployment-patch.yaml
```

查看与 Deployment 相关的 Pod：

```
kubectl get pods
```

输出显示 Deployment 有两个 Pod。`1/1` 表示每个 Pod 有一个容器:

```
NAME                        READY     STATUS    RESTARTS   AGE
patch-demo-28633765-670qr   1/1       Running   0          23s
patch-demo-28633765-j5qs3   1/1       Running   0          23s
```

把运行的 Pod 的名字记下来。稍后，你将看到这些 Pod 被终止并被新的 Pod 替换。

此时，每个 Pod 都有一个运行 nginx 镜像的容器。现在假设你希望每个 Pod 有两个容器：一个运行 nginx，另一个运行 redis。

创建一个名为 `patch-file-containers.yaml` 的文件。内容如下:

```
spec:
	template:
		spec:
			containers:
			- name: patch-demo-ctr-2
				image: redis
```

修补你的 Deployment：

```shell
kubectl patch deployment patch-demo --patch "$(cat patch-file-containers.yaml)"
```

查看修补后的 Deployment：

```shell
kubectl get deployment patch-demo --output yaml
```

输出显示 Deployment 中的 PodSpec 有两个容器:

```yaml
containers:
- image: redis
  imagePullPolicy: Always
  name: patch-demo-ctr-2
  ...
- image: nginx
  imagePullPolicy: Always
  name: patch-demo-ctr
  ...
```

查看与 patch Deployment 相关的 Pod:

```shell
kubectl get pods
```

输出显示正在运行的 Pod 与以前运行的 Pod 有不同的名称。Deployment 终止了旧的 Pod，并创建了两个 符合更新的部署规范的新 Pod。`2/2` 表示每个 Pod 有两个容器:

```
NAME                          READY     STATUS    RESTARTS   AGE
patch-demo-1081991389-2wrn5   2/2       Running   0          1m
patch-demo-1081991389-jmg7b   2/2       Running   0          1m
```

仔细查看其中一个 patch-demo Pod:

```shell
kubectl get pod <your-pod-name> --output yaml
```

输出显示 Pod 有两个容器:一个运行 nginx，一个运行 redis:

```
containers:
- image: redis
  ...
- image: nginx
  ...
```

### 4.1.1 策略性合并类的 patch 的说明

你在前面的练习中所做的 patch 称为`策略性合并 patch（Strategic Merge Patch)`。 请注意，patch 没有替换`containers` 列表。相反，它向列表中添加了一个新 Container。换句话说， patch 中的列表与现有列表合并。当你在列表中使用策略性合并 patch 时，并不总是这样。 在某些情况下，列表是替换的，而不是合并的。

对于策略性合并 patch，列表可以根据其 patch 策略进行替换或合并。 patch 策略由 Kubernetes 源代码中字段标记中的 `patchStrategy` 键的值指定。 例如，`PodSpec` 结构体的 `Containers` 字段的 `patchStrategy` 为 `merge`：

```
type PodSpec struct {
  ...
  Containers []Container `json:"containers" patchStrategy:"merge" patchMergeKey:"name" ...`
```

你可以在 [Kubernetes API 文档](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.21/#podspec-v1-core) 中看到 patch 策略。

创建一个名为 `patch-file-tolerations.yaml` 的文件。内容如下:

```yaml
spec:
  template:
    spec:
      tolerations:
      - effect: NoSchedule
        key: disktype
        value: ssd
```

对 Deployment 执行 patch 操作：

```
kubectl patch deployment patch-demo --patch "$(cat patch-file-containers.yaml)"
```

查看修补后的 Deployment：

```shell
kubectl get deployment patch-demo --output yaml
```

输出结果显示 Deployment 中的 PodSpec 只有一个容忍度设置：

```shell
tolerations:
      - effect: NoSchedule
        key: disktype
        value: ssd
```

请注意，PodSpec 中的 `tolerations` 列表被替换，而不是合并。这是因为 PodSpec 的 `tolerations` 的字段标签中没有 `patchStrategy` 键。所以策略合并 patch 操作使用默认的 patch 策略，也就是 `replace`。

```go
type PodSpec struct {
  ...
  Tolerations []Toleration `json:"tolerations,omitempty" protobuf:"bytes,22,opt,name=tolerations"`
```

## 4.2 使用 JSON 合并 patch 更新 Deployment

策略性合并 patch 不同于 [JSON 合并 patch](https://tools.ietf.org/html/rfc7386)。 使用 JSON 合并 patch，如果你想更新列表，你必须指定整个新列表。新的列表完全取代现有的列表。

`kubectl patch` 命令有一个 `type` 参数，你可以将其设置为以下值之一:

| 参数值    | 合并类型                                                     |
| --------- | ------------------------------------------------------------ |
| json      | [JSON Patch, RFC 6902](https://tools.ietf.org/html/rfc6902)  |
| merge     | [JSON Merge Patch, RFC 7386](https://tools.ietf.org/html/rfc7386) |
| strategic | 策略合并 patch                                               |

有关 JSON patch 和 JSON 合并 patch 的比较，查看 [JSON patch 和 JSON 合并 patch](https://erosb.github.io/post/json-patch-vs-merge-patch/)。

`type` 参数的默认值是 `strategic`。在前面的练习中，我们做了一个策略性的合并 patch。

下一步，在相同的 Deployment 上执行 JSON 合并 patch。创建一个名为 `patch-file-2` 的文件。内容如下:

```
spec:
  template:
    spec:
      containers:
      - name: patch-demo-ctr-3
        image: gcr.io/google-samples/node-hello:1.0
```

在 patch 命令中，将 `type` 设置为 `merge`：

```shell
kubectl patch deployment patch-demo --type merge --patch "$(cat patch-file-2.yaml)"
```

查看修补后的 Deployment：

```shell
kubectl get deployment patch-demo --output yaml
```

patch 中指定的`containers`列表只有一个 Container。 输出显示你所给出的 Contaier 列表替换了现有的 `containers` 列表。

```yaml
spec:
  containers:
  - image: gcr.io/google-samples/node-hello:1.0
    ...
    name: patch-demo-ctr-3
```

列表中运行的 Pod：

```shell
kubectl get pods
```

在输出中，你可以看到已经终止了现有的 Pod，并创建了新的 Pod。`1/1` 表示每个新 Pod只运行一个容器。

```shell
NAME                          READY     STATUS    RESTARTS   AGE
patch-demo-1307768864-69308   1/1       Running   0          1m
patch-demo-1307768864-c86dc   1/1       Running   0          1m
```

## 4.3 使用带 retainKeys 策略的策略合并 patch 更新 Deployment

[`application/deployment-retainkeys.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/zh/examples/application/deployment-retainkeys.yaml) ![Copy application/deployment-retainkeys.yaml to clipboard](https://d33wubrfki0l68.cloudfront.net/0901162ab78eb4ff2e9e5dc8b17c3824befc91a6/44ccd/images/copycode.svg)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: retainkeys-demo
spec:
  selector:
    matchLabels:
      app: nginx
  strategy:
    rollingUpdate:
      maxSurge: 30%
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: retainkeys-demo-ctr
        image: nginx
```

创建 Deployment：

```shell
kubectl apply -f https://k8s.io/examples/application/deployment-retainkeys.yaml
```

这时，Deployment 被创建，并使用 `RollingUpdate` 策略。

创建一个名为 `patch-file-no-retainkeys.yaml` 的文件，内容如下：

```yaml
spec:
  strategy:
    type: Recreate
```

修补你的 Deployment:

```bash
kubectl patch deployment retainkeys-demo --type merge --patch "$(cat patch-file-no-retainkeys.yaml)"
```

在输出中，你可以看到，当 `spec.strategy.rollingUpdate` 已经拥有取值定义时， 将其 `type` 设置为 `Recreate` 是不可能的。

```shell
The Deployment "retainkeys-demo" is invalid: spec.strategy.rollingUpdate: Forbidden: may not be specified when strategy `type` is 'Recreate'
```

更新 `type` 取值的同时移除 `spec.strategy.rollingUpdate` 现有值的方法是 为策略性合并操作设置 `retainKeys` 策略：

创建另一个名为 `patch-file-retainkeys.yaml` 的文件，内容如下：

```yaml
spec:
  strategy:
    $retainKeys:
    - type
    type: Recreate
```

使用此 patch，我们表达了希望只保留 `strategy` 对象的 `type` 键。 这样，在 patch 操作期间 `rollingUpdate` 会被删除。

使用新的 patch 重新修补 Deployment：

```bash
kubectl patch deployment retainkeys-demo --type strategic --patch "$(cat patch-file-retainkeys.yaml)"
```

检查 Deployment 的内容：

```shell
kubectl get deployment retainkeys-demo --output yaml
```

输出显示 Deployment 中的 `strategy` 对象不再包含 `rollingUpdate` 键：

```shell
spec:
  strategy:
    type: Recreate
  template:
```

### 4.3.1 关于使用 retainKeys 策略的策略合并 patch 操作的说明

在前文练习中所执行的称作 *带 `retainKeys` 策略的策略合并 patch（Strategic Merge Patch with retainKeys Strategy）*。 这种方法引入了一种新的 `$retainKey` 指令，具有如下策略：

- 其中包含一个字符串列表；
- 所有需要被保留的字段必须在 `$retainKeys` 列表中给出；
- 对于已有的字段，会和对象上对应的内容合并；
- 在修补操作期间，未找到的字段都会被清除；
- 列表 `$retainKeys` 中的所有字段必须 patch 操作所给字段的超集，或者与之完全一致。

策略 `retainKeys` 并不能对所有对象都起作用。它仅对那些 Kubernetes 源码中 `patchStrategy` 字段标志值包含 `retainKeys` 的字段有用。 例如 `DeploymentSpec` 结构的 `Strategy` 字段就包含了 `patchStrategy` 为 `retainKeys` 的标志。

```go
type DeploymentSpec struct {
  ...
  // +patchStrategy=retainKeys
  Strategy DeploymentStrategy `json:"strategy,omitempty" patchStrategy:"retainKeys" ...`
```

## 4.4 kubectl patch 命令的其他形式

`kubectl patch` 命令使用 YAML 或 JSON。它可以接受以文件形式提供的补丁，也可以 接受直接在命令行中给出的补丁。

创建一个文件名称是 `patch-file.json` 内容如下：

```json
{
   "spec": {
      "template": {
         "spec": {
            "containers": [
               {
                  "name": "patch-demo-ctr-2",
                  "image": "redis"
               }
            ]
         }
      }
   }
}
```

以下命令是等价的：

```shell
kubectl patch deployment patch-demo --patch "$(cat patch-file.yaml)"
kubectl patch deployment patch-demo --patch 'spec:\n template:\n  spec:\n   containers:\n   - name: patch-demo-ctr-2\n     image: redis'

kubectl patch deployment patch-demo --patch "$(cat patch-file.json)"
kubectl patch deployment patch-demo --patch '{"spec": {"template": {"spec": {"containers": [{"name": "patch-demo-ctr-2"
```

