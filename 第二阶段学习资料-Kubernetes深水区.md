# 第二阶段学习资料：Kubernetes 深水区

生成日期：2026-05-08  
对应路线文档：[工作路线完善版.md](</Users/imwl/Documents/New project/工作路线完善版.md>)

## 目标

第二阶段的目标不是“会写更多 YAML”，而是理解 Kubernetes 为什么这样工作。

阶段结束时，你应该能做到：

- 解释一次 `kubectl apply` 从客户端到实际 Pod 变化的完整链路。
- 理解 apiserver、etcd、controller-manager、scheduler、kubelet、kube-proxy、CoreDNS、CNI 的职责边界。
- 能定位 Pod 为什么 Pending、CrashLoopBackOff、ImagePullBackOff、OOMKilled、NotReady。
- 能解释 Service 为什么不通，问题是在 DNS、Service、EndpointSlice、kube-proxy、CNI、NetworkPolicy 还是应用。
- 能设计一套稳定发布基线，包括 request / limit、readiness、graceful shutdown、PDB、HPA、rollout、回滚。
- 能写一个最小 Operator，并说明它的 reconcile 边界、owner reference、finalizer、status condition。

第一阶段解决的是“Linux 与网络下沉能力”。第二阶段要解决的是“把这些底层能力映射回 Kubernetes 控制面和数据面”。

## 建议环境

不要在生产集群上做实验。建议准备一个可销毁的本地集群：

- 一台 Linux VM 或 macOS / Linux 本机。
- Docker 或 containerd。
- kind，建议至少 1 个 control-plane + 2 个 worker。
- kubectl。
- Helm。
- jq、yq。
- kubectx / kubens 可选。
- stern 或 kail 可选，用于多 Pod 日志。
- crictl 可选，用于节点容器运行时观察。
- Go 1.22+，用于 Operator 实验。
- Kubebuilder，用于 CRD / Controller 实验。

创建一个多节点 kind 集群：

```bash
cat <<'EOF' > kind-k8s-deep.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF

kind create cluster --name k8s-deep --config kind-k8s-deep.yaml
kubectl cluster-info
kubectl get nodes -o wide
```

清理环境：

```bash
kind delete cluster --name k8s-deep
```

## 学习顺序

推荐顺序：

```text
Kubernetes API 与声明式对象
    ↓
apiserver / etcd / watch / resourceVersion
    ↓
controller / reconcile / ownerReference / finalizer
    ↓
scheduler / requests / limits / QoS / eviction
    ↓
kubelet / CRI / Pod 生命周期 / probes
    ↓
Service / EndpointSlice / kube-proxy / CoreDNS / CNI
    ↓
稳定发布基线：readiness / PDB / HPA / graceful shutdown
    ↓
CRD / Operator / Kubebuilder
```

这条顺序的重点是：先理解 Kubernetes 的控制循环，再看 Pod 如何真正跑起来，最后写一个自己的控制器。

## 第 1 周：Kubernetes API 与对象生命周期

### 要理解什么

Kubernetes 的核心不是容器，而是 API。Pod、Deployment、Service、ConfigMap、Ingress、Gateway、CRD 都是 API 对象。

你要理解：

- Kubernetes API 是控制面的入口。
- API 对象通常包含 `metadata`、`spec`、`status`。
- `spec` 表示你期望的状态。
- `status` 表示系统观察到的状态。
- `metadata.resourceVersion` 用于并发控制、list / watch 和变更追踪。
- `kubectl apply` 不是“直接创建容器”，而是向 apiserver 提交对象期望状态。

重点概念：

- Group / Version / Kind。
- Resource。
- Namespace scope 与 cluster scope。
- `metadata.generation`。
- `status.observedGeneration`。
- `resourceVersion`。
- Server-side apply。
- Admission。
- Event。

### 必读资料

- [Kubernetes API Concepts](https://kubernetes.io/docs/reference/using-api/api-concepts/)
- [The Kubernetes API](https://kubernetes.io/docs/concepts/overview/kubernetes-api/)
- [Kubernetes Objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/)
- [Server-Side Apply](https://kubernetes.io/docs/reference/using-api/server-side-apply/)

### 实验一：观察 apply 到对象变化

创建一个 Deployment：

```bash
kubectl create deployment api-demo --image=nginx --replicas=2
kubectl get deployment api-demo -o yaml
kubectl get replicaset
kubectl get pod -l app=api-demo -o wide
```

观察关键字段：

```bash
kubectl get deployment api-demo -o json | jq '.metadata.generation, .metadata.resourceVersion, .status.observedGeneration'
kubectl get deployment api-demo -o json | jq '.spec.replicas, .status.replicas, .status.availableReplicas'
kubectl get events --sort-by=.metadata.creationTimestamp | tail -30
```

修改副本数：

```bash
kubectl scale deployment api-demo --replicas=3
kubectl get deployment api-demo -w
```

你要回答：

- `spec.replicas` 和 `status.availableReplicas` 的区别是什么？
- `metadata.generation` 什么时候变化？
- `resourceVersion` 能不能当业务版本号？
- `kubectl scale` 改的是哪个对象的哪个字段？
- Pod 是谁创建出来的，是 `kubectl` 还是 controller？

清理：

```bash
kubectl delete deployment api-demo
```

## 第 2 周：apiserver、etcd 与 watch

### 要理解什么

控制面核心链路：

```text
kubectl / controller / kubelet
    ↓
apiserver
    ↓
etcd
    ↓
watch
    ↓
controller reconcile
```

apiserver 是 Kubernetes API 的前门。etcd 是 Kubernetes 的一致性键值存储。controller、scheduler、kubelet 等组件通过 apiserver 读写对象，而不是直接操作 etcd。

你要理解：

- 为什么 etcd 慢会拖慢控制面。
- 为什么 etcd 需要 quorum。
- 为什么磁盘 I/O 和网络延迟会影响 etcd。
- watch 如何让 controller 及时收到对象变化。
- list + watch 为什么比不断全量 list 更适合控制器。
- etcd 备份和恢复为什么是集群灾备核心。

### 必读资料

- [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)
- [Kubernetes API Concepts: Resource versions and watches](https://kubernetes.io/docs/reference/using-api/api-concepts/)
- [Operating etcd clusters for Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)
- [etcd Maintenance](https://etcd.io/docs/v3.5/op-guide/maintenance/)
- [etcd Disaster Recovery](https://etcd.io/docs/v3.7/op-guide/recovery/)
- [etcd Failure Modes](https://etcd.io/docs/v3.5/op-guide/failures/)

### 实验二：观察控制面组件

如果你使用 kind：

```bash
kubectl get pods -n kube-system -o wide
kubectl get pods -n kube-system
docker ps --format '{{.Names}}'
docker exec -it k8s-deep-control-plane crictl ps
```

观察静态 Pod：

```bash
docker exec -it k8s-deep-control-plane ls /etc/kubernetes/manifests
docker exec -it k8s-deep-control-plane cat /etc/kubernetes/manifests/kube-apiserver.yaml
docker exec -it k8s-deep-control-plane cat /etc/kubernetes/manifests/etcd.yaml
```

你要回答：

- control-plane 组件为什么在 kind 里也是 Pod？
- 静态 Pod 是谁管理的？
- kubelet 如果停了，静态 Pod 会怎样？
- apiserver、etcd、scheduler、controller-manager 分别监听什么端口？

### 实验三：观察 watch

终端一：

```bash
kubectl get pods -w
```

终端二：

```bash
kubectl create deployment watch-demo --image=nginx --replicas=1
kubectl scale deployment watch-demo --replicas=3
kubectl delete deployment watch-demo
```

再用原始 API 看 watch：

```bash
kubectl proxy
```

另开终端：

```bash
curl -N 'http://127.0.0.1:8001/api/v1/namespaces/default/pods?watch=true'
```

你要回答：

- watch 事件里有哪些类型？
- ADDED、MODIFIED、DELETED 分别代表什么？
- controller 为什么需要 list 后再 watch？

### 实验四：etcd 快照观察

只在本地 kind 集群做。不同发行版的证书路径可能不同。

```bash
docker exec -it k8s-deep-control-plane sh
```

在 control-plane 容器中：

```bash
export ETCDCTL_API=3
etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=table

etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /tmp/kind-etcd.snapshot
```

你要回答：

- etcd health 与 apiserver health 是一回事吗？
- endpoint status 里哪些字段值得关注？
- 为什么生产集群必须定期备份 etcd？
- 为什么不建议业务系统直接访问 Kubernetes etcd？

## 第 3 周：controller、reconcile 与 Workload

### 要理解什么

Kubernetes 的“自动化”来自 controller。controller 持续观察实际状态，与期望状态对比，然后修正差异。

典型链路：

```text
Deployment
    ↓
ReplicaSet
    ↓
Pod
```

你要理解：

- controller 不是执行一次就结束，而是持续 reconcile。
- ownerReference 建立对象从属关系。
- finalizer 用于删除前清理外部资源。
- status condition 用于表达控制器观察到的状态。
- Deployment controller 如何通过 ReplicaSet 做滚动发布。
- Garbage Collector 如何清理从属对象。

### 必读资料

- [Kubernetes Controllers](https://kubernetes.io/docs/concepts/architecture/controller/)
- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
- [Owners and Dependents](https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/)
- [Finalizers](https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/)

### 实验五：拆开 Deployment、ReplicaSet、Pod

```bash
kubectl create deployment deploy-demo --image=nginx:1.25 --replicas=3
kubectl get deployment,rs,pod -l app=deploy-demo
kubectl get rs -l app=deploy-demo -o yaml | less
kubectl get pod -l app=deploy-demo -o json | jq '.items[0].metadata.ownerReferences'
```

升级镜像：

```bash
kubectl set image deployment/deploy-demo nginx=nginx:1.27
kubectl rollout status deployment/deploy-demo
kubectl get rs -l app=deploy-demo
kubectl rollout history deployment/deploy-demo
```

回滚：

```bash
kubectl rollout undo deployment/deploy-demo
kubectl rollout status deployment/deploy-demo
```

你要回答：

- 为什么一次 Deployment 更新会产生新的 ReplicaSet？
- 旧 ReplicaSet 为什么还保留？
- Pod 的 ownerReference 指向谁？
- 如果手动删除一个 Pod，谁会把它补回来？
- `rollout undo` 回滚的是什么？

清理：

```bash
kubectl delete deployment deploy-demo
```

## 第 4 周：调度、资源管理、QoS 与驱逐

### 要理解什么

Pending 类问题大多与调度有关。你要学会看 scheduler 的约束条件：

- request 是否超过节点可分配资源。
- nodeSelector 是否匹配。
- nodeAffinity / podAffinity / podAntiAffinity 是否过严。
- taint / toleration 是否匹配。
- topologySpreadConstraints 是否导致无法放置。
- priority / preemption 是否生效。
- PVC 绑定、StorageClass、zone 约束是否影响调度。

资源管理上，你要真正理解：

- request 用于调度和资源保障。
- limit 用于限制容器资源使用。
- CPU limit 可能导致 throttling。
- memory limit 可能导致 OOMKilled。
- QoS 会影响节点资源压力下的驱逐优先级。

### 必读资料

- [Scheduling, Preemption and Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/)
- [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/)
- [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Pod Quality of Service Classes](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/)
- [Node-pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)

### 实验六：制造 Pending

创建一个请求超大资源的 Pod：

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pending-cpu
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: "100"
        memory: "128Mi"
EOF
```

观察：

```bash
kubectl get pod pending-cpu
kubectl describe pod pending-cpu
kubectl get events --sort-by=.metadata.creationTimestamp | tail -30
kubectl describe nodes | less
```

你要回答：

- Pod 为什么 Pending？
- scheduler 的错误信息在哪里？
- request 太大和 limit 太大对调度的影响是否一样？

清理：

```bash
kubectl delete pod pending-cpu
```

### 实验七：taint 与 toleration

给一个 worker 加 taint：

```bash
kubectl get nodes
kubectl taint nodes k8s-deep-worker dedicated=gpu:NoSchedule
```

创建普通 Pod：

```bash
kubectl run no-toleration --image=nginx --restart=Never
kubectl describe pod no-toleration
```

创建带 toleration 的 Pod：

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: with-toleration
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
  containers:
  - name: app
    image: nginx
EOF
```

观察：

```bash
kubectl get pods -o wide
kubectl describe pod with-toleration
```

清理：

```bash
kubectl delete pod no-toleration with-toleration --ignore-not-found
kubectl taint nodes k8s-deep-worker dedicated=gpu:NoSchedule-
```

你要回答：

- toleration 是否保证 Pod 一定调度到这个节点？
- taint 是“吸引”Pod 还是“排斥”Pod？
- 如果要专属节点，只用 toleration 够不够？

### 实验八：QoS 观察

创建三个 Pod：

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: qos-besteffort
spec:
  containers:
  - name: app
    image: nginx
---
apiVersion: v1
kind: Pod
metadata:
  name: qos-burstable
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "64Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: qos-guaranteed
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "100m"
        memory: "64Mi"
EOF
```

观察：

```bash
kubectl get pod qos-besteffort qos-burstable qos-guaranteed -o jsonpath='{range .items[*]}{.metadata.name}{" => "}{.status.qosClass}{"\n"}{end}'
```

你要回答：

- 三种 QoS 是如何计算出来的？
- 哪类 Pod 在节点压力下更容易被驱逐？
- request 和 limit 相等就一定合理吗？

清理：

```bash
kubectl delete pod qos-besteffort qos-burstable qos-guaranteed
```

## 第 5 周：kubelet、CRI、Pod 生命周期与 probes

### 要理解什么

Pod 被调度到 Node 后，kubelet 才开始真正让容器运行起来。

你要理解：

- scheduler 只是决定 Pod 绑定到哪个 Node。
- kubelet 负责在节点上启动、停止、探测 Pod。
- kubelet 通过 CRI 与 containerd / CRI-O 交互。
- Pod sandbox 通常对应 pause container。
- CNI 在 Pod sandbox 网络准备阶段参与。
- liveness、readiness、startup probe 的作用不同。
- readiness 影响 Service endpoint，liveness 影响容器重启。

### 必读资料

- [Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Liveness, Readiness, and Startup Probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/)
- [Container Runtime Interface](https://kubernetes.io/docs/concepts/architecture/cri/)
- [Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
- [Debug Running Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod)

### 实验九：readiness 与 endpoint

创建一个 readiness 会失败的服务：

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ready-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ready-demo
  template:
    metadata:
      labels:
        app: ready-demo
    spec:
      containers:
      - name: nginx
        image: nginx
        readinessProbe:
          httpGet:
            path: /not-exist
            port: 80
          periodSeconds: 3
---
apiVersion: v1
kind: Service
metadata:
  name: ready-demo
spec:
  selector:
    app: ready-demo
  ports:
  - port: 80
    targetPort: 80
EOF
```

观察：

```bash
kubectl get pods -l app=ready-demo
kubectl describe pod -l app=ready-demo
kubectl get endpoints ready-demo
kubectl get endpointslice -l kubernetes.io/service-name=ready-demo
```

修复 readiness：

```bash
kubectl patch deployment ready-demo --type=json -p='[
  {"op":"replace","path":"/spec/template/spec/containers/0/readinessProbe/httpGet/path","value":"/"}
]'
kubectl rollout status deployment ready-demo
kubectl get endpoints ready-demo
```

你要回答：

- Pod Running 是否代表能接流量？
- readiness 失败时 endpoint 会怎样？
- liveness 失败和 readiness 失败的后果有什么不同？

清理：

```bash
kubectl delete deployment ready-demo
kubectl delete service ready-demo
```

### 实验十：CrashLoopBackOff 与日志

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: crash-demo
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo start; sleep 1; echo fail; exit 1"]
EOF
```

观察：

```bash
kubectl get pod crash-demo -w
kubectl describe pod crash-demo
kubectl logs crash-demo
kubectl logs crash-demo --previous
```

你要回答：

- CrashLoopBackOff 是 Pod phase 吗？
- `kubectl logs --previous` 为什么有用？
- 重启次数在哪里看？
- 如果容器启动太快退出，怎么抓证据？

清理：

```bash
kubectl delete pod crash-demo
```

### 实验十一：节点内观察容器运行时

在 kind 节点容器里观察：

```bash
docker exec -it k8s-deep-worker crictl ps
docker exec -it k8s-deep-worker crictl pods
docker exec -it k8s-deep-worker crictl images
```

你要回答：

- CRI 里 pod sandbox 和 container 有什么区别？
- kubelet 与 containerd 之间通过什么接口通信？
- `kubectl get pod` 与 `crictl ps` 看到的信息为什么不完全一样？

## 第 6 周：Service、EndpointSlice、kube-proxy、CoreDNS 与 CNI

### 要理解什么

Kubernetes 网络问题最容易“看起来像应用问题”。你需要把访问链路拆开：

```text
client Pod
    ↓
DNS 查询
    ↓
Service ClusterIP
    ↓
kube-proxy / dataplane
    ↓
EndpointSlice
    ↓
backend Pod IP
    ↓
container port
```

你要理解：

- Service 是稳定访问入口，不是后端本身。
- EndpointSlice 记录 Service 后端 endpoint。
- kube-proxy 监听 Service 和 EndpointSlice，编程节点数据面。
- CNI 负责 Pod 网络。
- CoreDNS 负责集群服务发现。
- NetworkPolicy 是否生效取决于 CNI 是否支持。

### 必读资料

- [Services, Load Balancing, and Networking](https://kubernetes.io/docs/concepts/services-networking/)
- [Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
- [Virtual IPs and Service Proxies](https://kubernetes.io/docs/reference/networking/virtual-ips/)
- [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Network Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
- [CoreDNS kubernetes plugin](https://coredns.io/plugins/kubernetes)
- [Debug Services](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)

### 实验十二：Service 访问链路

创建服务：

```bash
kubectl create deployment net-demo --image=nginx --replicas=2
kubectl expose deployment net-demo --port=80 --target-port=80
kubectl run curl --image=curlimages/curl --restart=Never -- sleep 3600
```

观察：

```bash
kubectl get svc net-demo -o wide
kubectl get endpoints net-demo
kubectl get endpointslice -l kubernetes.io/service-name=net-demo
kubectl get pods -l app=net-demo -o wide
kubectl exec curl -- curl -v http://net-demo.default.svc.cluster.local
```

制造 selector 不匹配：

```bash
kubectl patch service net-demo -p '{"spec":{"selector":{"app":"not-exist"}}}'
kubectl get endpoints net-demo
kubectl exec curl -- curl -v --max-time 3 http://net-demo.default.svc.cluster.local
```

恢复：

```bash
kubectl patch service net-demo -p '{"spec":{"selector":{"app":"net-demo"}}}'
```

你要回答：

- DNS 解析成功是否代表服务一定可用？
- Service 有 ClusterIP 但 endpoint 为空时会怎样？
- EndpointSlice 的 ready 状态与 readiness 有什么关系？

### 实验十三：CoreDNS 排障

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get svc -n kube-system kube-dns
kubectl get configmap -n kube-system coredns -o yaml
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100
kubectl exec curl -- cat /etc/resolv.conf
kubectl exec curl -- nslookup kubernetes.default.svc.cluster.local
kubectl exec curl -- nslookup net-demo.default.svc.cluster.local
```

你要回答：

- Pod 的 `/etc/resolv.conf` 里 nameserver 指向哪里？
- search domain 如何影响短域名解析？
- CoreDNS 挂了时，已有连接和新建连接分别可能怎样？

清理：

```bash
kubectl delete deployment net-demo
kubectl delete service net-demo
kubectl delete pod curl
```

## 第 7 周：稳定发布基线

### 要理解什么

稳定发布不是只会 `kubectl rollout restart`。你要能设计一套“服务上线不伤用户”的基本保护：

- readinessProbe：没准备好不接流量。
- startupProbe：慢启动服务不要被 liveness 误杀。
- livenessProbe：卡死后能自愈，但不能滥用。
- terminationGracePeriodSeconds：给服务优雅退出时间。
- preStop：配合连接排空。
- PDB：限制自愿中断数量。
- rollingUpdate：控制 maxUnavailable 和 maxSurge。
- HPA：根据指标水平扩缩容。
- request / limit：给调度和资源隔离提供依据。

### 必读资料

- [Disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
- [Specifying a Disruption Budget](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)
- [Deployments: Rolling Update Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Pod Lifecycle: Termination of Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Horizontal Pod Autoscaling](https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/)
- [Vertical Pod Autoscaling](https://kubernetes.io/docs/concepts/workloads/autoscaling/vertical-pod-autoscale/)
- [Node Autoscaling](https://kubernetes.io/docs/concepts/cluster-administration/node-autoscaling/)

### 实验十四：滚动发布与 PDB

创建 Deployment：

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rollout-demo
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: rollout-demo
  template:
    metadata:
      labels:
        app: rollout-demo
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          periodSeconds: 3
        lifecycle:
          preStop:
            exec:
              command: ["sh", "-c", "sleep 10"]
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: rollout-demo
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: rollout-demo
EOF
```

滚动升级：

```bash
kubectl set image deployment/rollout-demo nginx=nginx:1.27
kubectl rollout status deployment/rollout-demo
kubectl get pods -l app=rollout-demo -w
```

观察 PDB：

```bash
kubectl get pdb rollout-demo
kubectl describe pdb rollout-demo
```

你要回答：

- `maxUnavailable` 和 PDB 是同一个东西吗？
- PDB 是否会限制 Deployment 自己的滚动更新？
- preStop 和 terminationGracePeriodSeconds 如何配合？
- readiness 如何影响滚动发布的安全性？

清理：

```bash
kubectl delete deployment rollout-demo
kubectl delete pdb rollout-demo
```

### 实验十五：HPA 基础观察

kind 默认可能没有 metrics-server。你可以安装 metrics-server，或者只阅读 HPA 对象行为。

安装 metrics-server 的本地实验命令可能随版本变化，建议以官方项目 README 为准。安装后观察：

```bash
kubectl top nodes
kubectl top pods
```

创建一个 HPA：

```bash
kubectl create deployment hpa-demo --image=registry.k8s.io/hpa-example --requests=cpu=200m --replicas=1
kubectl autoscale deployment hpa-demo --cpu-percent=50 --min=1 --max=5
kubectl get hpa hpa-demo -w
```

你要回答：

- HPA 是按 Pod 数扩缩，还是调整单个 Pod 的 CPU / memory？
- HPA 依赖什么指标源？
- 没有 requests 时，CPU 利用率型 HPA 为什么容易出问题？
- HPA、VPA、Node Autoscaler 分别解决哪一层扩缩容？

清理：

```bash
kubectl delete hpa hpa-demo
kubectl delete deployment hpa-demo
```

## 第 8 周：CRD、Operator 与 Kubebuilder

### 要理解什么

CRD 和 Operator 是 Kubernetes 平台扩展的核心。你要理解：

- CRD 让你定义新的 API 类型。
- Controller watch 你的 CRD。
- Reconcile 把 CR 的期望状态转成真实资源。
- Status 记录观察结果。
- OwnerReference 让 Kubernetes 知道资源归属。
- Finalizer 让删除变成可控流程。
- Operator 不适合所有场景，简单模板用 Helm / Kustomize 可能更合适。

一个健康的 Operator 应该是：

- 幂等的。
- 可重试的。
- 不依赖事件只触发一次。
- 能处理对象被手动修改。
- 能处理外部资源失败。
- 能写清楚 status condition。
- 有明确权限边界。

### 必读资料

- [Kubernetes Operator Pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
- [Extend the Kubernetes API with CustomResourceDefinitions](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)
- [Kubebuilder Book: Getting Started](https://book.kubebuilder.io/getting-started)
- [Kubebuilder Book: What is in a controller?](https://book-v3.book.kubebuilder.io/cronjob-tutorial/controller-overview)
- [Kubebuilder Book: Using Finalizers](https://book.kubebuilder.io/reference/using-finalizers.html)
- [Operator SDK Go Operator Tutorial](https://sdk.operatorframework.io/docs/building-operators/golang/tutorial/)

### 实验十六：创建一个最小 Operator

安装 Kubebuilder 后：

```bash
mkdir service-operator
cd service-operator
go mod init example.com/service-operator
kubebuilder init --domain example.com --repo example.com/service-operator
kubebuilder create api --group platform --version v1alpha1 --kind AppService
```

目标设计：

- 用户创建 `AppService`。
- Operator 自动创建 Deployment。
- Operator 自动创建 Service。
- `spec.replicas` 控制副本数。
- `spec.image` 控制镜像。
- status 记录 observedGeneration、availableReplicas、Ready condition。

示例 CR：

```yaml
apiVersion: platform.example.com/v1alpha1
kind: AppService
metadata:
  name: demo
spec:
  image: nginx:1.27
  replicas: 2
  port: 80
```

你要实现的 reconcile 逻辑：

```text
读取 AppService
    ↓
如果不存在，说明被删除，退出
    ↓
确保 Deployment 存在
    ↓
确保 Service 存在
    ↓
如果 Deployment / Service 与 spec 不一致，更新
    ↓
读取 Deployment 状态
    ↓
更新 AppService status
```

你要回答：

- 为什么 reconcile 必须幂等？
- 如果 Deployment 被人手动删掉，Operator 应该怎样？
- 如果 Service 被人手动改端口，Operator 应该怎样？
- status 更新失败是否应该影响 spec reconcile？
- Operator 需要哪些 RBAC 权限？

### 实验十七：finalizer 思维练习

假设你的 Operator 会创建外部 DNS 记录或云资源。删除 CR 时不能直接让对象消失，否则外部资源会残留。

你要设计：

```text
用户删除 AppService
    ↓
metadata.deletionTimestamp 出现
    ↓
controller 发现 finalizer 存在
    ↓
删除外部资源
    ↓
确认删除成功
    ↓
移除 finalizer
    ↓
Kubernetes 真正删除 AppService
```

你要回答：

- finalizer 是谁加的？
- deletionTimestamp 出现后对象是否已经消失？
- 如果外部资源删除失败，应该移除 finalizer 吗？
- finalizer 卡住时如何排障？

## 第二阶段命令清单

API 与对象：

```bash
kubectl api-resources
kubectl api-versions
kubectl explain deployment.spec
kubectl get <resource> -o yaml
kubectl get <resource> -o json | jq
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl diff -f <file>
kubectl apply --server-side -f <file>
```

Workload：

```bash
kubectl get deployment,rs,pod
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name>
kubectl scale deployment/<name> --replicas=<n>
kubectl describe deployment <name>
kubectl describe pod <name>
kubectl logs <pod>
kubectl logs <pod> --previous
```

调度与资源：

```bash
kubectl get nodes -o wide
kubectl describe node <node>
kubectl top nodes
kubectl top pods
kubectl get pod <pod> -o jsonpath='{.status.qosClass}'
kubectl taint nodes <node> key=value:NoSchedule
kubectl label nodes <node> key=value
kubectl get events --field-selector reason=FailedScheduling
```

网络与服务：

```bash
kubectl get svc
kubectl describe svc <service>
kubectl get endpoints <service>
kubectl get endpointslice -l kubernetes.io/service-name=<service>
kubectl get pods -o wide
kubectl exec <pod> -- nslookup <service>
kubectl exec <pod> -- curl -v http://<service>
kubectl get configmap -n kube-system coredns -o yaml
kubectl logs -n kube-system -l k8s-app=kube-dns
```

控制面：

```bash
kubectl get pods -n kube-system
kubectl logs -n kube-system <control-plane-pod>
kubectl get --raw='/readyz?verbose'
kubectl get --raw='/livez?verbose'
kubectl get --raw='/metrics'
```

kind 节点观察：

```bash
docker ps
docker exec -it k8s-deep-control-plane crictl ps
docker exec -it k8s-deep-control-plane crictl pods
docker exec -it k8s-deep-worker crictl ps
docker exec -it k8s-deep-worker iptables-save
docker exec -it k8s-deep-worker nft list ruleset
```

Operator：

```bash
kubebuilder init
kubebuilder create api
make manifests
make install
make run
kubectl get crd
kubectl describe crd <name>
```

## 每周产出

第 1 周产出：

- 一份《Kubernetes API 对象生命周期笔记》。
- 内容包括 `spec`、`status`、`generation`、`observedGeneration`、`resourceVersion` 的例子。

第 2 周产出：

- 一份《apiserver / etcd / watch 链路笔记》。
- 内容包括控制面组件、watch 事件、etcd health、快照观察。

第 3 周产出：

- 一份《Deployment 到 Pod 的 controller 链路笔记》。
- 内容包括 Deployment、ReplicaSet、Pod、ownerReference、rollout、rollback。

第 4 周产出：

- 一份《Kubernetes 调度失败排查笔记》。
- 内容包括 Pending、request 过大、taint / toleration、QoS。

第 5 周产出：

- 一份《kubelet 与 Pod 生命周期笔记》。
- 内容包括 CrashLoopBackOff、readiness、liveness、CRI、sandbox。

第 6 周产出：

- 一份《Service 访问链路排障笔记》。
- 内容包括 DNS、Service、EndpointSlice、CoreDNS、kube-proxy / dataplane。

第 7 周产出：

- 一份《稳定发布基线模板》。
- 内容包括 Deployment strategy、readiness、startupProbe、preStop、PDB、HPA、资源配置。

第 8 周产出：

- 一个最小 Operator 项目。
- 内容包括 CRD、controller、Deployment / Service reconcile、status condition、README。

## 自测题

你应该能不用查资料回答：

- `kubectl apply` 后，容器是不是立刻由 kubectl 创建？
- `spec` 和 `status` 的职责分别是什么？
- `resourceVersion`、`generation`、`observedGeneration` 的区别是什么？
- controller 的 reconcile 为什么必须幂等？
- Deployment、ReplicaSet、Pod 的 ownerReference 是怎样的？
- 手动删除 Deployment 管理的 Pod，为什么会自动补回来？
- Pod Pending 时，先看 `kubectl logs` 有用吗？
- request 和 limit 分别影响什么？
- CPU limit 可能带来什么副作用？
- QoS 三种类别如何计算？
- readinessProbe 失败会不会重启容器？
- livenessProbe 失败会不会从 Service endpoint 摘除？
- CrashLoopBackOff 是不是 Pod phase？
- Service 有 ClusterIP 是否代表一定有后端？
- EndpointSlice 和 Service 的关系是什么？
- DNS 解析成功是否代表 HTTP 一定能访问？
- PDB 是否会阻止 Deployment 滚动发布？
- HPA、VPA、Node Autoscaler 各自扩的是什么？
- CRD 和 Operator 的区别是什么？
- finalizer 卡住时应该如何排查？

## 推荐阅读顺序

先读 Kubernetes API 与控制面：

- [Kubernetes API Concepts](https://kubernetes.io/docs/reference/using-api/api-concepts/)
- [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)
- [Kubernetes Controllers](https://kubernetes.io/docs/concepts/architecture/controller/)
- [Operating etcd clusters for Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)

再读 Workload 与调度：

- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Scheduling, Preemption and Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/)
- [Pod Quality of Service Classes](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/)

然后读网络与服务：

- [Services, Load Balancing, and Networking](https://kubernetes.io/docs/concepts/services-networking/)
- [Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
- [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Virtual IPs and Service Proxies](https://kubernetes.io/docs/reference/networking/virtual-ips/)
- [CoreDNS kubernetes plugin](https://coredns.io/plugins/kubernetes)

最后读扩展与 Operator：

- [Kubernetes Operator Pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
- [CustomResourceDefinitions](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)
- [Kubebuilder Book](https://book.kubebuilder.io/)
- [Operator SDK Go Tutorial](https://sdk.operatorframework.io/docs/building-operators/golang/tutorial/)

## 第二阶段完成标准

你可以认为第二阶段过关的标准是：

- 能画出 `kubectl apply -> apiserver -> etcd -> watch -> controller -> Pod` 的控制面链路。
- 能画出 `Pod -> DNS -> Service -> EndpointSlice -> backend Pod` 的数据面访问链路。
- 能解释一个 Pod 从创建到 Running 的完整流程。
- 能定位 Pending、CrashLoopBackOff、ImagePullBackOff、OOMKilled、NotReady。
- 能解释 Service 不通的分层排查路径。
- 能写出一个稳定发布 Deployment 模板。
- 能做一次 rollout、观察问题、回滚，并写清楚证据链。
- 能写一个最小 Operator，并让它 reconcile Deployment / Service。

这一阶段真正练出来的是“Kubernetes 系统级理解”：你不再只是记 YAML 字段，而是能知道每个对象背后是谁在 watch、谁在 reconcile、谁在节点上执行、谁在数据面转发。
