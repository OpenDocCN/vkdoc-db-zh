# 第二章 安装 CockroachDB

### Docker 安装

要使用 Docker 安装 CockroachDB，首先确保你的机器上已安装 Docker。如果尚未安装，请访问 Docker 安装网站 [`docs.docker.com/get-docker`](https://docs.docker.com/get-docker) 获取安装说明。

你可以通过以下命令在 Docker 中启动一个 CockroachDB 实例：

```
$ docker run \
    --rm -it \
    --name=cockroach \
    -p 26257:26257 \
    -p 8080:8080 \
    cockroachdb/cockroach:v21.1.7 start-single-node \
    --insecure
```

该命令将拉取 CockroachDB Docker 镜像并运行它，同时暴露端口 26257 用于客户端连接和端口 8080 用于 HTTP 连接。此命令会阻塞直到进程终止，届时 CockroachDB Docker 容器将被停止并移除。

要测试 CockroachDB 是否已启动并运行，请使用 `cockroach sql` 命令进入 CockroachDB SQL shell：

```
$ cockroach sql --insecure
#
# 欢迎使用 CockroachDB SQL shell。
# 所有语句必须以分号结尾。
# 输入 \q 退出。
#
# 客户端版本：CockroachDB CCL v21.1.5 (x86_64-w64-mingw32, built 2021/07/02 04:03:50, go1.15.11)
# 服务器版本：CockroachDB CCL v21.1.7 (x86_64-unknown-linux-gnu, built 2021/08/09 17:55:28, go1.15.14)
# 集群 ID：0b3ff09c-7bcc-4631-8121-335cfd83b04c
#
# 输入 \? 获取简要介绍。
#
root@:26257/defaultdb>
```

### Kubernetes 安装

要使用 Kubernetes 安装 CockroachDB，请确保已安装以下先决条件：

- 本地 Kubernetes 安装 – 有多种方式可以在你的机器上安装 Kubernetes。以下是一些可用选项：
  - `kind` ([`kind.sigs.k8s.io/docs/user/quick-start`](https://kind.sigs.k8s.io/docs/user/quick-start))
  - `minikube` ([`minikube.sigs.k8s.io/docs/start`](https://minikube.sigs.k8s.io/docs/start))
  - `k3s` ([`rancher.com/docs/k3s/latest/en/installation`](https://rancher.com/docs/k3s/latest/en/installation))
  
  我将使用 `kind` 创建一个本地 Kubernetes 集群，在撰写本文时，这将安装 Kubernetes 1.21 版本。

- `kubectl` – Kubernetes CLI；访问 [`kubernetes.io/docs/tasks/tools`](https://kubernetes.io/docs/tasks/tools) 获取安装说明。

安装好这些先决条件后，我们就可以开始了。

作为一个数据库，CockroachDB 在 Kubernetes 中被视为有状态资源，因此你需要为它提供一个持久卷声明（PVC）。PVC 存在于节点级别，而非 Pod 级别，因此即使你的任何 CockroachDB Pod 被删除或重启，你的数据也是安全的。

要在 Kubernetes 中安装一个简单的 CockroachDB 集群，请创建以下每个小节中描述的文件。如果你不想手动创建以下文件，可以在此处应用来自 Cockroach Labs 的预制 Kubernetes 清单：[`github.com/cockroachdb/cockroach/blob/master/cloud/kubernetes/cockroachdb-statefulset.yaml`](https://github.com/cockroachdb/cockroach/blob/master/cloud/kubernetes/cockroachdb-statefulset.yaml)

#### 1_pod-disruption-budget.yaml

此文件为我们的 StatefulSet 创建 `PodDisruptionBudget` 资源。Pod 中断预算为 Kubernetes 提供了针对给定应用程序（由选择器 "cockroachdb" 标识）的 Pod 故障的容忍度。它确保在任何时间点，集群中不可用的 Pod 永远不会超过 `maxUnavailable` 个。通过将此值设置为 1，我们可以防止 Kubernetes 在滚动更新等操作期间移除超过 1 个 CockroachDB 节点。

如果你在 Kubernetes 中配置的是 5 节点集群，请考虑将此值设置为 2，因为即使有 2 个节点暂时不可用，你仍然会有一个 3 节点集群。

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: cockroachdb-budget
  labels:
    app: cockroachdb
spec:
  selector:
    matchLabels:
      app: cockroachdb
  maxUnavailable: 1
```

运行以下命令创建 Pod 中断预算：

```
$ kubectl apply -f 1_pod-disruption-budget.yaml
```

#### 2_stateful-set.yaml

现在我们进入 Kubernetes 配置的核心部分。是时候创建我们的 StatefulSet 了。StatefulSet 类似于部署，但它提供了围绕 Pod 调度的额外保证，以确保持久磁盘的存在。

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: cockroachdb
spec:
  serviceName: "cockroachdb"
  replicas: 3
  selector:
    matchLabels:
      app: cockroachdb
  template:
    metadata:
      labels:
        app: cockroachdb
    spec:
      affinity:
        podAntiAffinity:
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
      containers:
      - name: cockroachdb
        image: cockroachdb/cockroach:v21.1.7
        imagePullPolicy: IfNotPresent
        resources:
          requests:
            cpu: "1"
            memory: "1Gi"
          limits:
            cpu: "1"
            memory: "1Gi"
        ports:
        - containerPort: 26257
          name: grpc
        - containerPort: 8080
          name: http
        readinessProbe:
          httpGet:
            path: "/health?ready=1"
            port: http
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 2
        volumeMounts:
        - name: datadir
          mountPath: /cockroach/cockroach-data
        env:
        - name: COCKROACH_CHANNEL
          value: kubernetes-insecure
        - name: GOMAXPROCS
          valueFrom:
            resourceFieldRef:
              resource: limits.cpu
              divisor: "1"
        command:
        - "/bin/bash"
        - "-ecx"
        - >-
          exec /cockroach/cockroach
          start
          --logtostderr
          --insecure
          --advertise-host $(hostname -f)
          --http-addr 0.0.0.0
```



