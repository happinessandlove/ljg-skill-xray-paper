---
title:      xray-dflash-block-diffusion-speculative-decoding
date:       2026-02-08 Sun 23:30
tags:       [read, xray, paper]
identifier: 20260208T233042
source:     https://arxiv.org/abs/2602.06036
authors:    Jian Chen, Yesheng Liang, Zhijian Liu
venue:      arXiv preprint (2026.02)
---

# NAPKIN FORMULA

```
+----------------------------------------------------------+
|                                                          |
|   Speedup = Diffusion(parallel draft | target features)  |
|             / AR(sequential draft)                       |
|           = 6x lossless acceleration                     |
|                                                          |
+----------------------------------------------------------+
```

用扩散模型替代自回归模型做投机解码的草稿生成, 一次并行出 16 个 token, 再让目标模型验证, 速度提 6 倍且输出无损.

# PROBLEM

**痛点定义**: 大语言模型自回归解码逐 token 串行生成, 延迟高, GPU 利用率低, 成为推理瓶颈.

**前人困境**:
- 投机解码(Speculative Decoding)是目前主流加速手段, 但草稿模型本身仍然是自回归的, 生成 gamma 个草稿 token 需要 gamma 次串行前向传播
- EAGLE 系列(EAGLE-1/2/3)用单层 AR 草稿模型, 加速比上限约 2-3x, 因为草稿生成本身就是瓶颈
- 独立的扩散语言模型(LLaDA, SDAR 等)虽能并行生成, 但质量不如 AR 模型, 需要多步去噪, 整体不划算
- 用大参数量(7B)扩散模型做草稿, 内存开销过大, 不实用

# INSIGHT

**核心直觉**: "The target knows best" -- 不要让小扩散模型独自猜, 而是把目标 AR 模型的隐藏特征直接注入扩散草稿模型的 KV 投影中, 让草稿模型站在巨人肩膀上并行生成.

**关键步骤**:
1. *KV 注入而非 token 嵌入拼接*: 从目标模型均匀采样若干层的隐藏表示, 通过投影层融合后直接注入草稿模型每一层的 Key-Value 投影, 避免信息在深层被稀释(EAGLE 的 token embedding 拼接方式会导致信息衰减)
2. *位置依赖的损失加权*: 块内靠前位置的 token 错误会导致后续所有 token 被拒绝, 因此用指数衰减 w_k = exp(-(k-1)/gamma) 给靠前位置更大权重, 训练时优先保证前几个位置的准确率

# DELTA

**vs SOTA**:
- 在 Qwen3-8B 上, DFlash 平均加速 4.86x, EAGLE-3 仅 1.76-2.02x, DFlash 是 EAGLE-3 的 2.4 倍
- 接受长度(acceptance length) DFlash 达 6.49, EAGLE-3 仅 2.96-3.40
- 在 SGLang 生产部署中, 单并发下加速 5.1x, 8 并发仍有 4.5x
- 非贪婪采样(temperature=1)下仍保持 4.03x 加速
- 在推理模式(thinking mode)下也维持 4.5x 加速

**新拼图**: 证明了扩散模型作为"轻量级块预测适配器"的全新定位 -- 扩散模型不必与 AR 模型在生成质量上正面竞争, 而是专注做并行草稿生成, 由 AR 模型验证保证质量. 这开辟了扩散语言模型的实用化路径.

# CRITIQUE

**隐形假设**:
- 依赖目标模型的 prefill 阶段提取隐藏特征, 假设 prefill 的计算开销可忽略或已被流水线掩盖; 在短 prompt 场景下 prefill 占比可能不高, 但论文未详细讨论 prefill overhead
- 块大小泛化是不对称的: 大块训练可泛化到小块推理, 反之不行. 这意味着必须用足够大的块训练, 增加了训练成本
- 假设 GPU 并行执行效率远高于串行(tparallel << gamma * tstep), 在小 batch 或非最优硬件上此假设可能不成立
- 仅与 EAGLE-3 对比, 未与 Medusa, Lookahead, REST 等其他投机解码方法比较, 基线不够全面

**未解之谜**:
- 高并发下性能显著下降(32 并发仅 2.8x), 如何在大规模 serving 场景保持加速比?
- 大模型(30B)上加速比降至 2.3-3.2x, 随模型规模增大加速比如何变化? 是否存在上限?
- 多步去噪(目前只用 1 步)是否能进一步提升草稿质量? 1 步 vs 多步的权衡点在哪?
- 与 KV cache 压缩、量化等其他推理优化技术是否正交可叠加?

# LOGIC FLOW

```
                         DFlash Inference Pipeline
  ===================================================================

  [Input Prompt]
       |
       v
  +--------------------+
  | Target Model       |     Prefill phase (frozen)
  | (e.g. Qwen3-8B)   |----> Extract hidden features
  +--------------------+      from L uniformly sampled layers
       |                          |
       | Generate bonus token     | Fused context features
       v                          v
  +---------+            +------------------+
  | Token_0 |----------->| KV Injection     |
  +---------+  anchor    | into ALL layers  |
                         | of draft model   |
                         +------------------+
                                |
                                v
                    +------------------------+
                    |  Block Diffusion Draft  |
                    |  (5-layer lightweight)  |
                    |                        |
                    |  [M] [M] [M] ... [M]   |  gamma masked positions
                    |   |   |   |       |    |
                    |   v   v   v       v    |  Single forward pass
                    |  [d1][d2][d3]...[d_g]  |  ALL tokens in parallel
                    +------------------------+
                                |
                                v
                    +------------------------+
                    | Target Model Verify    |
                    | (parallel check)       |
                    +------------------------+
                         |            |
                    accept k tokens   reject rest
                         |
                         v
                    [Token_0 ... Token_k] + bonus token
                         |
                         +-------> Next iteration (loop)
```

# NAPKIN SKETCH

```
   EAGLE-3 (Sequential Draft)        DFlash (Parallel Draft)
   ========================          ========================

   Target: [==========]              Target: [==========]
              |                                  |
              v                                  v
   Draft:  [t1]->[t2]->[t3]->...     Draft:  [t1 t2 t3 ... t16]
           step  step  step                   ONE step, all at once
           ~3 tokens accepted                 ~6.5 tokens accepted

   Time:   |||||||||||||||            Time:   |||||
           gamma * t_step                     t_parallel << gamma * t_step

   Speedup: ~2x                      Speedup: ~5x

   Secret sauce:
   +----------------------------------+
   | Target hidden features -------+  |
   |                               |  |
   |   Draft KV = proj(fused) ---->+  |  "The target knows best"
   |                                  |
   | Loss weight: w_k = exp(-(k-1)/g)|  Front positions matter more
   +----------------------------------+
```
