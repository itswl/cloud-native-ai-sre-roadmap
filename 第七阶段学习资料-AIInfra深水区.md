# 第七阶段学习资料（进阶）：AI Infra 深水区实验

生成日期：2026-08-24
对应路线文档：[工作路线完善版.md](工作路线完善版.md)
前置：完成[第七阶段基础篇](第七阶段学习资料-AIInfra.md)；理论配套阅读 ai-ops-learning 仓库的 `llm-inference-internals.md` 与 `gpu-cluster-ops.md`

## 目标

基础篇解决"能部署、能观测"。本篇通过 9 组实验，把三句话变成肌肉记忆：

- prefill 是算力瓶颈，decode 是带宽瓶颈，KV cache 是并发瓶颈。
- GPU 是会坏的生产资料，故障处置要流水线化。
- 调度器（推理引擎的和 K8s 的）的每个参数都对应一条 SLO 曲线。

阶段结束时，你应该能做到：

- 用压测数据画出一个推理服务的"容量拐点"，并解释拐点由哪个资源决定。
- 给任意 vLLM 服务做 TTFT 分解（排队 vs prefill）和 ITL 毛刺归因（prefill 干扰 vs 抢占）。
- 解释量化在什么流量形态下加速、什么形态下失效，并有自己的实测数据。
- 用 nccl-tests 验收一台多卡机器，并解读 busbw。
- 写出 GPU 故障自愈流水线的判定逻辑（Xid 分级 → 动作）。
- 在无 GPU 的 kind 集群上演示 DRA 的对象模型和 Kueue 的 gang 调度。

## 环境

- 一张 24G+ 的卡（4090/A10/L20 都行）跑实验 1-5；实验 6 需要多卡机；实验 7 需要任意 NVIDIA 卡；实验 8 不需要 GPU。
- vLLM 最新稳定版、DCGM（`datacenter-gpu-manager`）、nccl-tests、kind、kubectl。
- 模型建议 `Qwen/Qwen3-8B`（FP16 与 AWQ 两个版本都下好）。

---

## 实验一：找到你的容量拐点

目的：亲手复现"QPS 涨延迟不动 → 再涨一点全面雪崩"的容量悬崖，并归因。

```bash
# 起服务（参数刻意保守，让拐点来得早一点）
vllm serve Qwen/Qwen3-8B --max-num-seqs 64 --gpu-memory-utilization 0.9

# 梯度压测（vLLM 内置压测器；老版本用 benchmarks/benchmark_serving.py 等价）
for rate in 1 2 4 8 16 32; do
  vllm bench serve \
    --model Qwen/Qwen3-8B \
    --dataset-name random --random-input-len 1024 --random-output-len 256 \
    --num-prompts 300 --request-rate $rate \
    --save-result --result-filename rate_${rate}.json
done
```

每一轮同时抓四个指标：P95 TTFT、P95 ITL、`vllm:num_requests_waiting`、`vllm:gpu_cache_usage_perc`。

你要回答：

- 拐点出现时，先饱和的是哪个信号？waiting 队列、KV 水位，还是 ITL？
- 拐点前后，吞吐（总 token/s）分别怎么变？为什么拐点后吞吐反而可能下降？
- 把 `--max-num-seqs` 改成 16 重跑：拐点位置和拐点后的行为有什么区别？哪种配置更适合在线服务？

产出：一张"request rate → P95 TTFT / P95 ITL / 吞吐"三线图，标注拐点和归因。

## 实验二：chunked prefill 的 TTFT/ITL 跷跷板

目的：理解 `--max-num-batched-tokens` 是延迟形态的第一旋钮。

```bash
# 三种预算分别起服务压同一份混合负载（短请求 + 10% 的 8K 长 prompt）
vllm serve Qwen/Qwen3-8B --max-num-batched-tokens 512
vllm serve Qwen/Qwen3-8B --max-num-batched-tokens 2048
vllm serve Qwen/Qwen3-8B --max-num-batched-tokens 8192
```

你要回答：

- 长 prompt 的 TTFT 随预算怎么变？短请求的 P99 ITL 随预算怎么变？
- 为什么这两条曲线方向相反？（提示：一步 batch 的执行时间由谁决定）
- 你的业务如果是"对话为主 + 偶发文档分析"，选哪档？为什么根治要靠 PD 分离？

## 实验三：前缀缓存命中与路由亲和

```bash
# 负载 A：所有请求共享同一个 2K token 的 system prompt
# 负载 B：每个请求的 prompt 完全随机
# 分别压测，对比 TTFT 分布和 /metrics 里的前缀缓存命中指标
```

你要回答：

- 命中与未命中的 TTFT 差多少？这个差值和"2K token 的 prefill 时间"对得上吗？
- 把服务扩成 2 副本、轮询路由，命中率变成多少？为什么？
- 由此推导：多副本 LLM 服务的负载均衡策略应该按什么设计？（对照 Gateway API Inference Extension 的做法）

## 实验四：制造并观察抢占

目的：见过抢占的人才会给抢占配告警。

```bash
# 故意把 KV 池挤小：大 max-num-seqs + 小显存预算 + 长输出
vllm serve Qwen/Qwen3-8B --gpu-memory-utilization 0.7 --max-num-seqs 128
# 压测：并发 100，--random-output-len 1024
```

观察 `vllm:num_preemptions_total` 和被抢占请求的 ITL 时间线。

你要回答：

- 抢占发生时 `gpu_cache_usage_perc` 在什么水位？
- 被抢占的请求，客户端视角是什么体验？（流式输出中间停顿多久）
- 写出你的抢占告警规则（阈值、for 窗口、severity），以及触发后的三个处置选项。

## 实验五：量化的两面性

目的：验证"W4A16 单流快、大 batch 可能失效"。

```bash
# 同一张卡分别起 FP16 和 AWQ 版本
# 场景 A：并发 1，测单流 t/s
# 场景 B：并发 64，测聚合吞吐 token/s
```

你要回答：

- 场景 A 里 AWQ 快多少？用"带宽 ÷ 权重字节"的 roofline 解释这个比例。
- 场景 B 里差距还在吗？如果 AWQ 反而不占优，瓶颈变成了什么？
- 算出两种配置各自的 MBU，哪个更接近硬件极限？

## 实验六：多卡机器的 NCCL 验收（需多卡）

```bash
nvidia-smi topo -m          # 先画出这台机器的拓扑
git clone https://github.com/NVIDIA/nccl-tests && cd nccl-tests && make
./build/all_reduce_perf -b 8 -e 4G -f 2 -g <卡数>

# 对照实验：屏蔽 P2P，强制走系统内存
NCCL_P2P_DISABLE=1 ./build/all_reduce_perf -b 8 -e 4G -f 2 -g <卡数>
```

你要回答：

- 大消息段的 busbw 是多少？两次对照差几倍？这个差值对 TP 推理意味着什么？
- `NCCL_DEBUG=INFO` 的输出里，如何确认它选择了 NVLink 还是 PCIe 还是 Socket？
- 给这台机型写一条"验收基线"：busbw 低于多少直接拒收。

## 实验七：GPU 故障处置演练

```bash
# 1. 分级体检
dcgmi diag -r 1 && dcgmi diag -r 2      # r3 留给维护窗口
# 2. 查 ECC 与坏行
nvidia-smi -q -d ECC,ROW_REMAPPER
# 3. 桌面推演（Xid 无法安全注入，用推演代替）：
#    抽三张卡片：Xid 31 / Xid 63 Pending / Xid 79
#    对每张写出：判定级别 → 是否排水 → 是否 reset → 是否 RMA → 通知谁
# 4. 把处置写成脚本：输入 Xid 码，输出动作建议 + 自动 cordon（干跑模式）
```

产出：一份《GPU 故障处置 runbook》+ 一个 `gpu-triage.sh`（读 dmesg 的 Xid → 按分级表输出处置建议）。

## 实验八：无 GPU 也能玩的调度实验（kind）

### 8a. DRA 对象模型

```bash
# dra-example-driver 在 kind 里模拟 GPU 设备，专为学习 DRA 设计（不需要真卡）
git clone https://github.com/kubernetes-sigs/dra-example-driver
cd dra-example-driver
# 按仓库 README 的 demo 脚本走：建 kind 集群 → 构建并安装示例驱动
kubectl get resourceslices          # 驱动上报的"设备清单"
kubectl apply -f demo/gpu-test1.yaml
kubectl get resourceclaims          # 观察 claim 的分配过程
```

你要回答：

- ResourceSlice / DeviceClass / ResourceClaim / ResourceClaimTemplate 各自由谁创建、生命周期如何？
- 和 device plugin 的 `nvidia.com/gpu: 1` 相比，claim 里多了哪些表达能力？
- 什么场景下你会推动生产集群迁移到 DRA？什么场景下按兵不动？

### 8b. Kueue 的 gang 与配额

```bash
kubectl apply --server-side -f https://github.com/kubernetes-sigs/kueue/releases/latest/download/manifests.yaml
# 建一个 ClusterQueue（配额设为 8 个假 GPU）+ LocalQueue
# 提交两个各需 6 GPU 的 Job：观察第二个 Job 被整体挂起而不是拿 2 个 GPU 干等
```

你要回答：

- 没有 Kueue 时，这两个 Job 会发生什么？（描述死锁形成）
- ClusterQueue 的 borrowing 解决什么问题？借出去的额度怎么收回？
- 训练平台的队列设计：按团队分还是按优先级分？说出你的方案和理由。

## 实验九：综合作业——容量评估答辩

不查资料完成，然后用实验验证：

```
题目：单卡 H100 80G（或按你手上的卡换算），Qwen2.5-72B-AWQ，
      流量画像：平均输入 3K token、输出 400 token、P95 ITL < 60ms
求：最大安全并发、预期单流 t/s、聚合吞吐、每百万 token 成本
   （GPU 按 ¥18/时全成本计），以及"何时必须加第二张卡"的触发指标
交付：一页 A4，公式过程 + 压测验证误差 + 误差原因分析
```

## 自测题

- 容量拐点由 KV 预算决定和由算力决定，压测曲线形态有什么区别？
- `max-num-batched-tokens` 调大，谁受益谁受损？
- 前缀缓存命中率在扩副本后下降的机制是什么？三种解法？
- 抢占率告警应该设在多少？触发后的处置优先级？
- W4A16 在高并发下失效的原因？此时想继续提吞吐该换什么量化路线？
- busbw 和 algbw 的换算关系？验收看哪个？
- Xid 48 之后为什么必须回滚 checkpoint？
- DRA 的 ResourceClaim 和 PVC 的设计同构性体现在哪？

## 完成标准

- 有一张自己压出来的容量拐点图，能当众讲清拐点归因。
- 有一份带数据的量化选型结论（单流/高并发两种形态）。
- 有一台机器的 NCCL 验收记录和基线值。
- 有一份 Xid 分级处置 runbook 和配套脚本。
- 能在 kind 上演示 DRA 和 Kueue，并说清生产迁移判断。
- 实验九的纸面估算与实测误差 < 30%，且能解释误差来源。

这一阶段练出来的是"AI Infra 的定量直觉"：别人说"加卡"，你说"先看是 KV 瓶颈还是带宽瓶颈，这两种加卡的方式不一样"。
