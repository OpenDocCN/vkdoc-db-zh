# 第二章 安装 CockroachDB

```
-- join cockroachdb-0.cockroachdb,cockroachdb-1.cockroachdb,cockroachdb-2.cockroachdb
--cache 25%
--max-sql-memory 25%
terminationGracePeriodSeconds: 60
volumes:
- name: datadir
  persistentVolumeClaim:
    claimName: datadir
podManagementPolicy: Parallel
updateStrategy:
  type: RollingUpdate
volumeClaimTemplates:
- metadata:
    name: datadir
  spec:
    accessModes:
    - "ReadWriteOnce"
    resources:
      requests:
        storage: 1Gi
```

运行以下命令来创建有状态集：

```
$ kubectl apply -f 2_stateful-set.yaml
```

无论你是否是经验丰富的 Kubernetes 用户，这里都有很多内容。我将带你了解一些不太明显的配置块，以进一步揭开配置的神秘面纱。

## **podAntiAffinity**

```
preferredDuringSchedulingIgnoredDuringExecution:
- weight: 1
  podAffinityTerm:
    labelSelector:
      matchExpressions:
      - key: app
        operator: In
        values:
        - cockroachdb
    topologyKey: kubernetes.io/hostname
```

你可以要求 Kubernetes 调度器使 Pod 对其他 Pod 具有亲和性或反亲和性。通过这样做，你分别要求它们被放置在一起或彼此远离。

为了强调此配置的重要性，假设我们的 Kubernetes 集群在云中运行，并且每个节点运行在不同的可用区中。这个`podAntiAffinity`规则要求 Kubernetes 在每个不同的节点上启动一个`cockroachdb`Pod。根据你的需求，你可能更喜欢使用`requiredDuringSchedulingIgnoredDuringExecution`规则，该规则保证每个 Pod 调度到不同的节点。这对于具有多个 Kubernetes 节点的集群来说是有意义的，但对于我们的本地单节点 Kubernetes 集群则不然。

## **env**

```
env:
- name: COCKROACH_CHANNEL
  value: kubernetes-insecure
- name: GOMAXPROCS
  valueFrom:
    resourceFieldRef:
      resource: limits.cpu
      divisor: "1"
```

在这个块中，我们设置了在初始化和运行时使用的环境变量。`COCKROACH_CHANNEL`变量将帮助我们识别我们是如何安装集群的（例如，在 Kubernetes 上安全安装还是通过 helm 安装等）。`GOMAXPROCS`变量被传递给 CockroachDB，并由 Go 运行时用来限制分配给 CockroachDB 进程的 CPU 核心数。

## **command**

```
command:
- "/bin/bash"
- "-ecx"
- |
  exec /cockroach/cockroach start \
    --logtostderr \
    --insecure \
    --advertise-host $(hostname -f) \
    --http-addr 0.0.0.0 \
    --join cockroachdb-0.cockroachdb,cockroachdb-1.cockroachdb,cockroachdb-2.cockroachdb \
    --cache 25% \
    --max-sql-memory 25%
```

在这个块中，我们向 CockroachDB 可执行文件传递了一些参数。这些参数告诉 CockroachDB 如何发现其他节点以及如何被其他节点发现，并为数据库设置内存限制。这些在资源有限的机器上运行时非常有用。

## **volumeClaimTemplates**

```
volumeClaimTemplates:
- metadata:
    name: datadir
  spec:
    accessModes:
    - "ReadWriteOnce"
    resources:
      requests:
        storage: 1Gi
```

在这个块中，我们向 Kubernetes 申请 1 gibibyte 的磁盘存储。有多种访问模式可用，但我使用了`ReadWriteOnce`，因为我们只需要为每个节点分配一次磁盘空间。

1 gibibyte = 2³⁰ = 1,073,741,824 字节

## **3_private-service.yaml**

此文件创建 Kubernetes 服务，该服务将被 CockroachDB 节点用于互相发现，也将被 Kubernetes 集群中的其他资源使用。由于它不暴露集群 IP，因此无法在集群外部访问。

暴露端口 26257 将允许 CockroachDB 节点相互通信，暴露端口 8080 将允许像 Terraform 这样的服务获取指标。

```
apiVersion: v1
kind: Service
metadata:
  name: cockroachdb
  labels:
    app: cockroachdb
spec:
  ports:
  - name: tcp
    port: 26257
    targetPort: 26257
  - name: http
    port: 8080
    targetPort: 8080
  publishNotReadyAddresses: true
  clusterIP: None
  selector:
    app: cockroachdb
```

运行以下命令来创建私有服务：

```
$ kubectl apply -f 3_private-service.yaml
```

## **4_public-service.yaml**

此文件创建一个 Kubernetes `LoadBalancer`服务，用于外部连接到 CockroachDB 节点。稍后我们将使用此服务连接到数据库。

> **警告** 根据你的基础设施，当运行托管的 Kubernetes 实例时，`LoadBalancer`服务可能会创建一个可公开访问的端点。考虑创建一个内部负载均衡器，或使用`ClusterIP`服务并通过强化的入口使服务可用。

暴露端口 26257 将允许我们连接到数据库，暴露端口 8080 将允许我们查看其仪表板。以下代码创建了一个 Kubernetes 服务，该服务暴露了在有状态集中定义的端口。

```
apiVersion: v1
kind: Service
metadata:
  name: cockroachdb-public
  labels:
    app: cockroachdb
spec:
  ports:
  - name: tcp
    port: 26257
    targetPort: 26257
  - name: http
    port: 8080
    targetPort: 8080
  selector:
    app: cockroachdb
  type: LoadBalancer
```

运行以下命令来创建私有服务：

```
$ kubectl apply -f 4_public-service.yaml
```

## **5_init.yaml**

此文件初始化 CockroachDB 集群。运行时，它将连接到集群中的第一个节点执行初始化。在我们的例子中，我们本可以指定任何节点来开始初始化，因为它们都配置为通过`--join`参数连接到其他节点。

```
apiVersion: batch/v1
kind: Job
metadata:
  name: init
  labels:
    app: cockroachdb
spec:
  template:
    spec:
      containers:
      - name: init
        image: cockroachdb/cockroach:v21.1.7
        imagePullPolicy: IfNotPresent
        command:
        - "/cockroach/cockroach"
        - "init"
        - "--insecure"
        - "--host=cockroachdb-0.cockroachdb"
      restartPolicy: Never
```

运行以下命令来创建初始化作业：

```
$ kubectl apply -f 5_init.yaml
```

如果你进行到这一步，你将有一个正常运行的 CockroachDB 集群在 Kubernetes 中运行！这个例子中涉及很多部分，所以你遇到的情况可能有所不同。

如果你的本地 Kubernetes 集群在 Docker 中运行并且遇到问题，请确保你已为 Docker 分配了足够的内存来运行此示例。6GB 应该绰绰有余。

要连接到集群的 HTTP 端点，创建一个到公共服务的端口转发并打开`http://localhost:8080`：

```
$ kubectl port-forward service/cockroachdb-public 8080
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
```

要连接到集群的 SQL 端点，创建一个到公共服务的端口转发：

```
$ kubectl port-forward service/cockroachdb-public 26257
Forwarding from 127.0.0.1:26257 -> 26257
Forwarding from [::1]:26257 -> 26257
```

然后像集群在本地运行一样使用`cockroach`命令：

```
$ cockroach sql --insecure
#
# 欢迎使用 CockroachDB SQL shell。
# 所有语句必须以分号结尾。
# 输入 \q 退出。
#
# 服务器版本: CockroachDB CCL v21.1.7 (x86_64-unknown-linux-gnu, built 2021/08/09 17:55:28, go1.15.14) (与客户端版本相同)
# 集群 ID: c3f571a3-1f67-44ab-9a62-d6f3ce6a6b20
#
# 输入 \? 获取简要介绍。
#
root@:26257/defaultdb>
```

如果你想从 Kubernetes*内部*连接到 CockroachDB 集群，请运行以下命令来创建一个打开 SQL shell 的 Kubernetes Pod：

```
$ kubectl run cockroachdb --rm -it \
    --image=cockroachdb/cockroach:v21.1.7 \
    --restart=Never \
    -- sql \
    --insecure \
    --host=cockroachdb-public
```



