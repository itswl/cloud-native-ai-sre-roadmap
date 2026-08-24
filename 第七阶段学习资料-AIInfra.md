# 第七阶段学习资料：AI Infra

生成日期：2026-05-08  
对应路线文档：[工作路线完善版.md](工作路线完善版.md)

## 目标

第七阶段的目标是从“会调 LLM API”升级到“理解 LLM 推理、服务化、成本和稳定性”。

阶段结束时，你应该能做到：

- 解释 token、context window、prefill、decode、KV cache。
- 理解 vLLM、SGLang、TensorRT-LLM 分别解决什么问题。
- 能部署一个 OpenAI-compatible 本地模型服务。
- 能观测 TTFT、TPOT、tokens/s、并发、排队、KV cache、显存。
- 能做粗略容量评估。
- 能设计 RAG 的解析、切分、embedding、检索、rerank、评估链路。
- 能设计有权限边界、审计、guardrails、人工确认的 Agent。

## 学习顺序

```text
LLM 推理基本概念
    ↓
GPU / CUDA / 显存 / NCCL / MIG
    ↓
vLLM / SGLang / TensorRT-LLM
    ↓
推理服务指标与容量评估
    ↓
RAG 检索系统
    ↓
Agent 工具调用与工作流
    ↓
生产安全边界与评估
```

## 第 1 周：LLM 推理基础

### 要理解什么

LLM 服务性能由很多因素决定：

- 模型参数量。
- precision / quantization。
- context length。
- 输入 token 数。
- 输出 token 数。
- prefill。
- decode。
- KV cache。
- batching。
- 并发。

关键指标：

- TTFT：Time To First Token。
- TPOT：Time Per Output Token。
- tokens/s。
- request latency。
- queue time。
- GPU memory usage。
- KV cache usage。

### 必读资料

- [PagedAttention 论文](https://arxiv.org/abs/2309.06180)
- [vLLM Documentation](https://docs.vllm.ai/en/stable/)
- [OpenAI Text Generation](https://platform.openai.com/docs/guides/text-generation)

### 实验一：推理指标拆解

选择一个小模型，用 vLLM 启动服务（Qwen3-0.6B 或 Qwen2.5-0.5B-Instruct 都可以，前者更新）：

```bash
vllm serve Qwen/Qwen3-0.6B
```

请求：

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-0.6B",
    "messages": [{"role":"user","content":"用一句话解释 Kubernetes controller"}],
    "stream": true
  }'
```

你要回答：

- 首 token 慢可能是什么原因？
- 输出越长为什么总耗时越长？
- context 越长为什么显存压力越大？
- KV cache 为什么影响并发？

## 第 2 周：GPU、CUDA、NCCL、MIG

### 要理解什么

AI Infra 不能只看模型，也要懂硬件资源：

- CUDA 是 NVIDIA GPU 编程平台。
- 显存是 LLM 推理的关键瓶颈。
- NCCL 用于多 GPU 通信。
- MIG 可以把支持的 GPU 切成隔离实例。
- tensor parallel 需要多卡通信。
- 多卡不一定线性加速。

### 必读资料

- [NVIDIA CUDA C++ Programming Guide](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html)
- [NVIDIA NCCL Documentation](https://docs.nvidia.com/deeplearning/nccl/)
- [NVIDIA MIG User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/)
- [NVIDIA Multi-Instance GPU](https://www.nvidia.com/en-us/technologies/multi-instance-gpu/)

### 实验二：GPU 观察

```bash
nvidia-smi
nvidia-smi dmon
nvidia-smi topo -m
```

如果有 Kubernetes GPU 节点：

```bash
kubectl describe node <gpu-node>
kubectl get pods -A -o wide | grep <gpu-node>
```

你要回答：

- GPU 利用率高是否代表服务吞吐最优？
- 显存接近满时可能发生什么？
- MIG 适合推理还是训练？
- 多卡推理为什么要考虑拓扑？

## 第 3 周：vLLM、SGLang、TensorRT-LLM

### 要理解什么

三类工具定位不同：

- vLLM：高吞吐 LLM serving，核心优势包括 PagedAttention、continuous batching、OpenAI-compatible server。
- SGLang：面向 LLM / VLM 的高性能 serving 与结构化执行。
- TensorRT-LLM：NVIDIA GPU 上的深度优化推理栈，适合追求极致性能与生产部署。

### 必读资料

- [vLLM Documentation](https://docs.vllm.ai/en/stable/)
- [SGLang Documentation](https://docs.sglang.ai/)
- [SGLang 项目站点](https://www.sglang.io/)
- [NVIDIA TensorRT-LLM](https://docs.nvidia.com/tensorrt-llm/)

### 实验三：对比两个 serving 引擎

至少对比：

```text
启动方式：
OpenAI-compatible API：
并发能力：
metrics：
显存占用：
首 token 延迟：
长输出吞吐：
部署复杂度：
```

你要回答：

- 你的场景更关心吞吐还是延迟？
- OpenAI-compatible API 带来什么迁移收益？
- serving engine 是否等于完整 AI 平台？

## 第 4 周：推理服务监控与容量评估

### 要理解什么

推理服务要观测：

- 请求量。
- 错误率。
- TTFT。
- TPOT。
- 总延迟。
- 输入 token。
- 输出 token。
- queue length。
- running requests。
- waiting requests。
- GPU 显存。
- KV cache usage。
- OOM / CUDA error。

容量估算粗略思路：

```text
请求量
  × 平均输入 token
  × 平均输出 token
  × 延迟目标
  × 并发峰值
  → 推理实例数量与 GPU 规格
```

### 实验四：压测与指标

设计压测：

```text
短输入短输出
短输入长输出
长输入短输出
长输入长输出
并发 1 / 4 / 16 / 64
```

你要记录：

- P50 / P95 TTFT。
- P50 / P95 TPOT。
- tokens/s。
- 显存。
- 错误率。
- 排队时间。

你要回答：

- 为什么长 context 会挤压并发？
- 为什么 batch 能提高吞吐但可能影响延迟？
- 何时应该扩副本，何时应该换更大 GPU？

## 第 5 周：RAG 基础

### 要理解什么

RAG 是检索系统，不是“向量库 + prompt”。

标准链路：

```text
文档解析
    ↓
清洗
    ↓
切分
    ↓
embedding
    ↓
索引
    ↓
retrieval
    ↓
rerank
    ↓
prompt assembly
    ↓
generation
    ↓
评估
```

关键问题：

- 召回率。
- 精排质量。
- chunk 粒度。
- metadata。
- 权限过滤。
- 引用与证据。
- stale document。
- hallucination。

### 必读资料

- [Retrieval-Augmented Generation 论文](https://arxiv.org/abs/2005.11401)
- [OpenAI Retrieval Guide](https://platform.openai.com/docs/guides/retrieval)

### 实验五：做一个 Runbook RAG

资料源：

- Kubernetes Runbook。
- 故障 RCA。
- 服务文档。
- 告警说明。

要求：

- 文档有 metadata：service、owner、env、version。
- 检索结果必须返回引用。
- 没有资料时必须回答“不足以判断”。
- 设计 20 个测试问题。

你要回答：

- 什么问题暴露了召回失败？
- 什么问题暴露了 chunk 切分失败？
- 什么问题暴露了权限过滤失败？
- 为什么 RAG 必须评估？

## 第 6 周：Agent 与工具调用

### 要理解什么

Agent 的核心不是“会自动想”，而是安全地使用工具完成任务。

生产 Agent 必须有：

- 工具权限边界。
- 只读与可写分级。
- 参数 schema。
- 超时。
- 重试。
- 审计日志。
- 人工确认。
- 成本限制。
- 输入输出 guardrails。
- 可观测性与 trace。

### 必读资料

- [OpenAI Function Calling](https://platform.openai.com/docs/guides/function-calling)
- [OpenAI Agents SDK](https://platform.openai.com/docs/guides/agents-sdk/)
- [OpenAI Agents SDK Tools](https://openai.github.io/openai-agents-python/tools/)
- [OpenAI Agents SDK Guardrails](https://openai.github.io/openai-agents-python/guardrails/)
- [OpenAI Agents SDK Tracing](https://openai.github.io/openai-agents-python/tracing/)
- [LangGraph Overview](https://docs.langchain.com/oss/python/langgraph)
- [LangGraph Persistence](https://docs.langchain.com/oss/python/langgraph/persistence)
- [Model Context Protocol Specification](https://modelcontextprotocol.io/specification/2025-06-18)

### 实验六：只读 SRE Agent

工具：

```text
query_prometheus
query_logs
get_k8s_events
get_recent_deployments
get_runbook
```

要求：

- 所有工具只读。
- 每次工具调用记录参数和结果摘要。
- 输出必须包含证据。
- 不足以判断时必须列出缺失证据。

你要回答：

- 哪些工具可以只读开放？
- 哪些工具必须人工确认？
- 如果日志里有敏感信息，Agent 应该如何处理？
- Agent trace 要记录哪些内容？

## 第 7 周：评估、成本与安全边界

### 要理解什么

AI 系统上线前必须评估：

- 正确率。
- 引用准确率。
- 工具调用正确率。
- 幻觉率。
- 拒答质量。
- 成本。
- 延迟。
- 安全边界。

### 实验七：评估集

准备 50 个问题：

- 20 个常规排障。
- 10 个资料不存在的问题。
- 10 个权限敏感问题。
- 10 个需要多工具关联的问题。

记录：

```text
问题：
期望答案：
实际答案：
是否引用证据：
是否工具调用正确：
是否幻觉：
是否该拒答：
成本：
延迟：
```

你要回答：

- 如何发现“看似正确但无证据”的回答？
- 什么时候要降低模型自由度？
- 什么时候要加工具 guardrail？
- 什么时候不该用 Agent？

## 命令清单

vLLM：

```bash
vllm serve <model>
curl http://localhost:8000/v1/models
curl http://localhost:8000/metrics
```

GPU：

```bash
nvidia-smi
nvidia-smi dmon
nvidia-smi topo -m
```

Kubernetes：

```bash
kubectl describe node <gpu-node>
kubectl logs <inference-pod>
kubectl top pod
kubectl get events --sort-by=.metadata.creationTimestamp
```

## 每周产出

- 第 1 周：推理指标笔记。
- 第 2 周：GPU 资源观察笔记。
- 第 3 周：serving engine 对比。
- 第 4 周：压测与容量评估。
- 第 5 周：Runbook RAG 原型。
- 第 6 周：只读 SRE Agent 原型。
- 第 7 周：评估集与评估报告。

## 自测题

- prefill 和 decode 的区别是什么？
- KV cache 为什么占显存？
- TTFT 和 TPOT 分别代表什么？
- vLLM 的 PagedAttention 解决什么问题？
- batching 为什么提升吞吐？
- MIG 的适用边界是什么？
- RAG 为什么必须做权限过滤？
- Agent 工具为什么要分只读和可写？
- guardrails 和 prompt 约束有什么区别？

## 完成标准

你可以认为第七阶段过关的标准是：

- 能部署并压测一个 LLM serving 服务。
- 能解释性能瓶颈在显存、KV cache、batch、模型大小还是请求模式。
- 能做一个带引用和评估集的 Runbook RAG。
- 能做一个只读 SRE Agent，并明确工具权限、审计和人工确认边界。

## 进阶

过关后继续：[第七阶段学习资料-AIInfra深水区.md](第七阶段学习资料-AIInfra深水区.md)——容量拐点、chunked prefill、抢占、量化两面性、NCCL 验收、GPU 故障演练、DRA/Kueue 共 9 组实验；理论配套 ai-ops-learning 仓库的 `llm-inference-internals.md` 和 `gpu-cluster-ops.md`。
