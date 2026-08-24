# 第八阶段学习资料：AIOps 与 AI SRE

生成日期：2026-05-08  
对应路线文档：[工作路线完善版.md](工作路线完善版.md)

## 目标

第八阶段的目标是把 Kubernetes、可观测性、平台工程、AI Agent 融合成真正有用的运维智能化能力。

阶段结束时，你应该能做到：

- 设计只读 RCA Agent。
- 自动关联告警、指标、日志、trace、K8s Event、发布记录、Runbook。
- 输出证据链，而不是猜根因。
- 设计 AI Runbook 工作流。
- 把自动化分成只读、建议、低风险写入、高风险写入。
- 设计人工确认、审计、回滚和失败停止机制。
- 用评估集衡量 AI SRE 的准确率、召回率、误操作风险和节省时间。

## 基本原则

AIOps 不要从“自动修复”开始。正确顺序是：

```text
自动收集上下文
    ↓
自动生成排查摘要
    ↓
自动匹配 Runbook
    ↓
自动提出建议
    ↓
低风险动作带审批执行
    ↓
高风险动作只建议或强审批
```

硬规则：

- 没证据就不能写根因。
- 不足以判断就明确说不足。
- 自动化先降低 MTTR 和 toil，再谈无人值守。
- 所有写操作必须有审计。
- 高风险动作必须人工确认。
- 自动修复失败必须停止，不允许继续扩大影响。

## 学习顺序

```text
Incident / RCA / Runbook 基础
    ↓
告警上下文自动收集
    ↓
指标 / 日志 / trace / event / 发布关联
    ↓
RCA Agent
    ↓
Runbook Automation
    ↓
受控修复
    ↓
评估与治理
```

## 第 1 周：Incident、RCA 与 Runbook

### 要理解什么

AI SRE 的基础不是模型，而是 SRE 流程：

- 告警分级。
- Incident command。
- 时间线。
- 影响面。
- 止血。
- 根因分析。
- 后续行动。
- Postmortem。

AI 适合做：

- 自动收集上下文。
- 摘要时间线。
- 推荐 Runbook。
- 生成 RCA 草稿。
- 检查缺失证据。

AI 不应直接替代：

- 事故指挥。
- 高风险判断。
- 生产变更批准。
- 责任归属。

### 必读资料

- [Google SRE Workbook：Incident Response](https://sre.google/workbook/incident-response/)
- [Google SRE：Managing Incidents](https://sre.google/sre-book/managing-incidents/)
- [Google SRE：Postmortem Culture](https://sre.google/sre-book/postmortem-culture/)
- [PagerDuty Incident Response](https://response.pagerduty.com/)
- [PagerDuty Runbook Automation](https://www.pagerduty.com/resources/incident-management-response/learn/runbook-automation-incident-response/)

### 实验一：定义 Incident 数据模型

```text
incident_id：
alert_name：
severity：
service：
namespace：
environment：
start_time：
detected_by：
customer_impact：
related_deployments：
related_metrics：
related_logs：
related_traces：
related_events：
runbook：
timeline：
current_hypotheses：
next_actions：
```

你要回答：

- 哪些字段来自告警？
- 哪些字段需要查询工具？
- 哪些字段可以由 AI 总结？
- 哪些字段必须人工确认？

## 第 2 周：告警上下文自动收集

### 要理解什么

一个好的告警上下文包应该包括：

- 告警表达式。
- 当前值与阈值。
- 最近 30 到 60 分钟趋势。
- 受影响服务。
- 受影响 Pod / Node。
- 相关 deployment / rollout。
- 最近变更。
- 关键日志样本。
- 关键 trace。
- 相关 K8s Event。
- Runbook 链接。

### 必读资料

- [Google SRE：Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [OpenTelemetry Semantic Conventions](https://opentelemetry.io/docs/concepts/semantic-conventions/)
- [OpenTelemetry Logging](https://opentelemetry.io/docs/specs/otel/logs/)

### 实验二：Alert Context Collector

实现一个脚本或 Agent 工具集：

```text
input: alert labels

steps:
  query Prometheus
  query Loki / ClickHouse
  query tracing backend
  query Kubernetes Events
  query Argo CD / Git commit
  fetch Runbook

output:
  markdown context report
```

你要回答：

- alert label 必须包含哪些字段？
- 如果告警没有 service label，会怎样？
- 最近变更如何和指标异常关联？
- 日志样本如何避免泄漏敏感信息？

## 第 3 周：RCA Agent

### 要理解什么

RCA Agent 的输出应该是：

```text
结论：
置信度：
证据：
反证：
缺失证据：
建议下一步：
止血建议：
风险：
```

不能输出：

- 没有证据的根因。
- 单一指标推断全部事实。
- 越权操作建议。
- 删除数据、重启核心服务等高风险动作的自动执行。

### 实验三：只读 RCA Agent

工具：

- `query_prometheus`
- `query_logs`
- `query_traces`
- `get_k8s_events`
- `get_recent_deployments`
- `get_runbook`

系统要求：

- 只读。
- 每个结论必须带证据。
- 置信度分高、中、低。
- 无法判断时必须说无法判断。
- 输出 Markdown。

你要回答：

- 如何防止 Agent 把症状当根因？
- 如何让 Agent 输出反证？
- 如何评估 RCA 是否正确？
- 如何处理多个可能根因？

## 第 4 周：Runbook Automation

### 要理解什么

Runbook 自动化分两类：

- 诊断型：收集上下文、检查状态、生成报告。
- 修复型：执行动作、改变系统状态。

诊断型可以先自动化。修复型必须分级。

动作风险分级：

```text
L0: 只读查询
L1: 低风险写入，例如创建诊断 ticket
L2: 可回滚变更，例如扩容无状态服务
L3: 高风险变更，例如回滚核心服务、限流、切流
L4: 危险操作，例如删除数据、改权限、清理存储
```

### 实验四：把 Runbook 变成工具

选择一个常见告警：

- Pod CrashLoopBackOff。
- 5xx 高。
- 延迟高。
- CoreDNS 错误。
- Kafka 消费积压。

把 Runbook 拆成工具：

```text
check_pods
check_events
check_logs
check_recent_deploy
check_dependencies
suggest_mitigation
```

你要回答：

- 哪些步骤可以自动执行？
- 哪些步骤需要人工确认？
- 哪些结果必须写入 incident timeline？
- Runbook 更新后 Agent 如何同步？

## 第 5 周：受控自动修复

### 要理解什么

自动修复一定要保守。先从低风险动作开始：

- 重启单个无状态 Pod。
- 扩容 Deployment 副本。
- 回滚到上一稳定版本。
- 触发缓存刷新。
- 降级非核心功能。

每个动作必须有：

- 前置条件。
- 影响范围。
- 执行命令。
- 成功判断。
- 失败判断。
- 回滚方式。
- 审计记录。
- 人工确认策略。

### 实验五：带审批扩容

流程：

```text
检测到 latency 高
    ↓
确认 CPU / saturation 高
    ↓
确认服务无异常发布
    ↓
建议扩容 replicas +1
    ↓
人工确认
    ↓
执行 kubectl scale 或 GitOps PR
    ↓
观察 10 分钟
    ↓
记录结果
```

你要回答：

- 扩容什么时候无效？
- 直接 kubectl scale 和改 GitOps repo 哪个更符合平台规范？
- 自动动作失败后是否继续尝试？
- 谁对自动修复负责？

## 第 6 周：AIOps 评估体系

### 要理解什么

AIOps 必须评估，否则很容易“演示很炫，线上不敢用”。

评估维度：

- 上下文收集完整率。
- 根因判断准确率。
- 缺失证据识别率。
- Runbook 匹配准确率。
- 错误建议率。
- 高风险动作拦截率。
- 平均节省排查时间。
- 人工确认通过率。
- 成本。
- 延迟。

### 实验六：评估集

准备 30 个历史或模拟事故：

- 10 个明确发布引起。
- 5 个资源不足。
- 5 个依赖故障。
- 5 个网络 / DNS。
- 5 个无足够证据。

每个样本包含：

```text
输入告警：
可用数据：
期望根因：
期望证据：
允许建议：
禁止动作：
```

你要回答：

- Agent 错误最常见在哪类事故？
- 缺数据时是否会胡猜？
- 是否能拒绝高风险动作？
- 是否真的减少人工排查时间？

## 第 7 周：平台化与治理

### 要理解什么

AI SRE 不能是某个人脚本。要平台化：

- 工具注册。
- 权限模型。
- 审计日志。
- 评估报告。
- Prompt / workflow 版本管理。
- Runbook 版本管理。
- 人工审批。
- Incident 集成。
- 灰度上线。

### 实验七：AI SRE 平台设计

写架构：

```text
Alert source
    ↓
Context collector
    ↓
RAG / Runbook search
    ↓
RCA Agent
    ↓
Risk classifier
    ↓
Human approval
    ↓
Action executor
    ↓
Audit / Timeline / Evaluation
```

你要回答：

- 工具权限在哪里控制？
- 审计日志写哪里？
- Prompt 变更如何评审？
- Runbook 过期如何发现？
- 如何灰度一个新的 Agent workflow？

## 命令与工具清单

只读工具建议：

```bash
kubectl get pods -n <ns> -o wide
kubectl describe pod <pod> -n <ns>
kubectl get events -n <ns> --sort-by=.metadata.creationTimestamp
kubectl logs <pod> -n <ns> --tail=200
kubectl rollout history deployment/<name> -n <ns>
```

Prometheus 查询：

```promql
sum by (service) (rate(http_requests_total{code=~"5.."}[5m]))
histogram_quantile(0.95, sum by (le, service) (rate(http_request_duration_seconds_bucket[5m])))
sum by (pod) (rate(container_cpu_usage_seconds_total[5m]))
container_memory_working_set_bytes
```

风险动作必须审批：

```bash
kubectl rollout undo deployment/<name> -n <ns>
kubectl scale deployment/<name> -n <ns> --replicas=<n>
kubectl delete pod <pod> -n <ns>
```

## 每周产出

- 第 1 周：Incident 数据模型。
- 第 2 周：Alert Context Collector。
- 第 3 周：只读 RCA Agent。
- 第 4 周：Runbook 工具化。
- 第 5 周：带审批修复流程。
- 第 6 周：AIOps 评估集。
- 第 7 周：AI SRE 平台架构设计。

## 自测题

- 为什么 AIOps 不应该从自动修复开始？
- RCA Agent 输出为什么必须带反证和缺失证据？
- 只读工具和可写工具如何分级？
- 哪些修复动作可以低风险自动化？
- GitOps 环境下自动修复应该直接改集群还是提 PR？
- 如何评估 Agent 是否真的减少 MTTR？
- Prompt 变更为什么需要版本管理？
- Runbook 过期会造成什么风险？

## 完成标准

你可以认为第八阶段过关的标准是：

- 能构建一个只读 RCA Agent，并在模拟事故中稳定输出证据链。
- 能把 Runbook 拆成工具，并明确风险等级。
- 能设计人工确认、审计、回滚和失败停止机制。
- 能用评估集证明系统有效，而不是只靠演示。

## 进阶

过关后继续：[第八阶段学习资料-AIOpsAISRE深水区.md](第八阶段学习资料-AIOpsAISRE深水区.md)——评估工程（离线评估集/影子模式）、告警关联三级策略、上下文工程、带审批的自动修复五层防御、生产灰度与责任模型，含工业锚点（Meta RCA 42%、微软 RCACopilot 0.77）和 2026 AI SRE 产品格局。
