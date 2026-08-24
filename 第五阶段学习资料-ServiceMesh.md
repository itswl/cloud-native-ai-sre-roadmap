# 第五阶段学习资料：Service Mesh

生成日期：2026-05-08  
对应路线文档：[工作路线完善版.md](工作路线完善版.md)

## 目标

第五阶段的目标不是“安装 Istio”，而是理解服务间通信治理。

阶段结束时，你应该能做到：

- 解释 Service Mesh 解决什么问题，也能判断什么时候不该上 Mesh。
- 理解 Envoy 的 listener、route、cluster、endpoint、filter chain。
- 理解 xDS：LDS、RDS、CDS、EDS。
- 理解 Istio sidecar 模式与 ambient 模式的差异。
- 理解 ztunnel、waypoint、HBONE、L4 / L7 分层。
- 能配置 mTLS、AuthorizationPolicy、timeout、retry、circuit breaker、fault injection。
- 能用 Gateway API 做流量路由。
- 能排查“路由没生效”“mTLS 失败”“waypoint 没接管 L7”“Envoy 配置不符合预期”。

## 建议环境

- kind / k3d 测试集群。
- Istio 最新稳定版本。
- istioctl。
- Gateway API CRD。
- httpbin、sleep、bookinfo 或你自己的两个 HTTP 服务。
- kiali 可选。
- Prometheus / Grafana 可选，但建议接入。

安装 Istio Ambient 的命令以官方文档为准，实验前先确认版本：

```bash
istioctl version
kubectl version
```

## 学习顺序

```text
Service Mesh 解决的问题
    ↓
Envoy 基础模型
    ↓
Istio 控制面与数据面
    ↓
Ambient：ztunnel / waypoint
    ↓
mTLS 与身份
    ↓
流量治理：timeout / retry / circuit breaking
    ↓
Gateway API 与路由
    ↓
Mesh 排障
```

## 第 1 周：Service Mesh 边界

### 要理解什么

Service Mesh 通常解决：

- 服务间 mTLS。
- 统一身份。
- L4 / L7 访问控制。
- 流量路由。
- 重试、超时、熔断。
- 可观测性。

它不应该被当成万能药：

- 应用自身超时和重试仍要合理。
- 数据库连接池仍要治理。
- Service Mesh 增加复杂度和数据面开销。
- 小系统或低复杂度系统不一定需要 Mesh。

### 必读资料

- [Istio Traffic Management](https://istio.io/latest/docs/concepts/traffic-management/)
- [Istio Security](https://istio.io/latest/docs/concepts/security/)
- [Istio Ambient Overview](https://istio.io/latest/docs/ambient/overview/)

### 实验一：画出服务调用链路

选择两个服务：

```text
frontend
    ↓
backend
```

画出无 Mesh、sidecar Mesh、ambient Mesh 三种路径。

你要回答：

- 流量在哪里被代理？
- mTLS 在哪里终止？
- L7 路由在哪里执行？
- 哪种模式对应用 Pod 侵入更小？

## 第 2 周：Envoy 基础

### 要理解什么

Envoy 的核心抽象：

- Listener：监听入口。
- Filter chain：连接处理链。
- Route：HTTP 路由规则。
- Cluster：上游服务池。
- Endpoint：具体后端实例。
- xDS：动态下发配置。

### 必读资料

- [Envoy Architecture Overview](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/arch_overview)
- [Envoy xDS Protocol](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol)
- [Envoy Listeners](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/listeners/listeners)

### 实验二：查看 Envoy 配置

如果使用 sidecar 模式：

```bash
istioctl proxy-status
istioctl proxy-config listeners <pod>.<namespace>
istioctl proxy-config routes <pod>.<namespace>
istioctl proxy-config clusters <pod>.<namespace>
istioctl proxy-config endpoints <pod>.<namespace>
```

你要回答：

- 某个 Service 对应哪个 cluster？
- 某条 HTTPRoute / VirtualService 是否进入 Envoy 配置？
- endpoint 是否和 Kubernetes EndpointSlice 一致？
- Envoy 配置不符合预期时，先查 Istio 还是先查应用？

## 第 3 周：Istio Ambient 基础

### 要理解什么

Ambient 模式把能力拆成两层：

- ztunnel：每节点 L4 安全覆盖层，处理 mTLS、L4 auth、L4 telemetry。
- waypoint：Envoy 代理，处理 L7 route、L7 auth、HTTP metrics、fault injection 等。

ztunnel 不解析 HTTP header。需要 L7 能力时才引入 waypoint。

### 必读资料

- [Istio Ambient Overview](https://istio.io/latest/docs/ambient/overview/)
- [Istio Ambient Data Plane](https://istio.io/latest/docs/ambient/architecture/data-plane/)
- [Istio：Configure waypoint proxies](https://istio.io/latest/docs/ambient/usage/waypoint/)
- [Istio：Add workloads to the mesh](https://istio.io/latest/docs/ambient/usage/add-workloads/)

### 实验三：加入 Ambient Mesh

示例流程以官方文档为准，核心动作：

```bash
kubectl create namespace mesh-demo
kubectl label namespace mesh-demo istio.io/dataplane-mode=ambient
kubectl -n mesh-demo apply -f samples/httpbin/httpbin.yaml
kubectl -n mesh-demo apply -f samples/curl/curl.yaml
```

注意：Istio 1.23 起官方示例 `sleep` 已更名为 `curl`（`samples/curl/curl.yaml`），老教程里的 `samples/sleep/sleep.yaml` 在新版本发行包里已不存在。

测试：

```bash
kubectl -n mesh-demo exec deploy/curl -- curl -sS http://httpbin:8000/get
istioctl ztunnel-config workloads
```

你要回答：

- namespace label 改变了什么？
- 加入 ambient 后 Pod 是否注入 sidecar？
- ztunnel 如何知道 workload？
- 不配置 waypoint 时，哪些 L7 能力不可用？

## 第 4 周：mTLS、身份与授权

### 要理解什么

Mesh 安全重点是：

- workload identity。
- mTLS。
- PeerAuthentication。
- AuthorizationPolicy。
- L4 与 L7 policy 的差异。

Ambient 中 L4 策略可由 ztunnel 执行，L7 策略需要 waypoint。

### 必读资料

- [Istio Authorization Policy](https://istio.io/latest/docs/reference/config/security/authorization-policy/)
- [Istio PeerAuthentication](https://istio.io/latest/docs/reference/config/security/peer_authentication/)
- [Istio Ambient L4 Policy](https://istio.io/latest/docs/ambient/usage/l4-policy/)

### 实验四：只允许 curl 访问 httpbin

目标：

- 默认拒绝。
- 只允许 `curl` workload 调用 `httpbin`。
- 验证非授权 Pod 被拒绝。

你要回答：

- policy 绑定到 source 还是 destination？
- L4 policy 能不能按 HTTP path 判断？
- 按 path / header 做授权时为什么需要 waypoint？
- mTLS 成功是否代表一定授权成功？

## 第 5 周：流量治理

### 要理解什么

流量治理能力包括：

- timeout。
- retry。
- circuit breaking。
- fault injection。
- traffic splitting。
- outlier detection。

关键原则：

- 重试不能无限放大流量。
- timeout 要和上下游预算一致。
- 熔断是保护系统，不是修复系统。
- fault injection 是验证韧性，不是线上乱试。

### 必读资料

- [Istio Request Timeouts](https://istio.io/latest/docs/tasks/traffic-management/request-timeouts/)
- [Istio Circuit Breaking](https://istio.io/latest/docs/tasks/traffic-management/circuit-breaking/)
- [Istio Fault Injection](https://istio.io/latest/docs/tasks/traffic-management/fault-injection/)
- [Gateway API HTTP Routing Guide](https://gateway-api.sigs.k8s.io/guides/http-routing/)

### 实验五：timeout 与 fault injection

目标：

- 给 backend 注入 2 秒延迟。
- 给 frontend 设置 1 秒 timeout。
- 观察错误率和 trace。

你要回答：

- timeout 是谁执行的？
- 应用自己的 timeout 和 mesh timeout 冲突时会怎样？
- 重试会不会让下游压力更大？
- 哪些接口不应该自动重试？

## 第 6 周：Gateway API 与流量分割

### 要理解什么

Gateway API 提供比 Ingress 更丰富的 API：

- GatewayClass。
- Gateway。
- HTTPRoute。
- ReferenceGrant。
- Listener。
- BackendRef。

Ambient 中 L7 路由通常使用 Gateway API 表达。

### 必读资料

- [Gateway API Overview](https://gateway-api.sigs.k8s.io/docs/concepts/api-overview/)
- [Gateway API Spec Reference（Gateway / HTTPRoute / ReferenceGrant）](https://gateway-api.sigs.k8s.io/reference/api-spec/)
- [HTTP Routing Guide](https://gateway-api.sigs.k8s.io/guides/http-routing/)
- [Traffic Splitting Guide](https://gateway-api.sigs.k8s.io/guides/traffic-splitting/)

### 实验六：双版本分流

目标：

```text
v1: 90%
v2: 10%
```

然后逐步调整到：

```text
v1: 50%
v2: 50%
```

你要回答：

- 分流规则最终会下发到哪里？
- 如何验证流量比例？
- 如果 v2 错误率高，回滚动作是什么？
- Service Mesh 分流和 Argo Rollouts canary 如何配合？

## 第 7 周：Mesh 可观测性与排障

### 要理解什么

Mesh 排障常见问题：

- 服务没加入 mesh。
- waypoint 未绑定。
- AuthorizationPolicy 绑定错对象。
- HTTPRoute parentRef 错。
- DestinationRule / VirtualService 冲突。
- Envoy 配置未下发。
- mTLS 模式不匹配。
- NetworkPolicy 或 CNI 层阻断。

### 实验七：排障 checklist

```bash
istioctl analyze
istioctl proxy-status
istioctl ztunnel-config workloads
istioctl ztunnel-config services
kubectl get httproute,gateway,authorizationpolicy,peerauthentication -A
kubectl logs -n istio-system -l app=ztunnel
kubectl logs -n istio-system deploy/istiod
```

你要回答：

- 如何判断流量是否经过 waypoint？
- 如何判断是 L4 policy 拒绝还是 L7 policy 拒绝？
- 如何判断路由规则有没有生效？
- 如何从 Envoy cluster 找到真实 endpoint？

## 命令清单

```bash
istioctl version
istioctl analyze
istioctl proxy-status
istioctl proxy-config listeners <pod>.<ns>
istioctl proxy-config routes <pod>.<ns>
istioctl proxy-config clusters <pod>.<ns>
istioctl proxy-config endpoints <pod>.<ns>
istioctl ztunnel-config workloads
istioctl ztunnel-config services
kubectl get gateway,httproute,grpcroutes -A
kubectl get authorizationpolicy,peerauthentication -A
kubectl logs -n istio-system deploy/istiod
kubectl logs -n istio-system -l app=ztunnel
```

## 每周产出

- 第 1 周：三种服务调用链路图。
- 第 2 周：Envoy xDS 配置观察笔记。
- 第 3 周：Ambient 入 mesh 实验记录。
- 第 4 周：mTLS 与授权策略实验。
- 第 5 周：timeout / retry / fault injection 实验。
- 第 6 周：Gateway API 分流实验。
- 第 7 周：Mesh 排障 checklist。

## 自测题

- ztunnel 和 waypoint 分别负责什么？
- Ambient 模式为什么可以不注入 sidecar？
- L4 policy 和 L7 policy 的区别是什么？
- Envoy listener、route、cluster、endpoint 的关系是什么？
- xDS 的 LDS、RDS、CDS、EDS 分别是什么？
- timeout 和 retry 怎么避免放大故障？
- Gateway API 与 Ingress 的核心差异是什么？
- 如何判断一个服务是否真的需要 Mesh？

## 完成标准

你可以认为第五阶段过关的标准是：

- 能解释一次请求在 ambient mesh 中的完整路径。
- 能配置并验证 mTLS、授权、路由、超时、重试和熔断。
- 能用 Gateway API 做双版本分流。
- 能用 istioctl 从控制面和数据面排查配置问题。
