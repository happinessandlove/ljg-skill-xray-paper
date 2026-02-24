---
title:      xray-single-agent-skills-vs-multi-agent
date:       2026-02-24 Mon 00:26
tags:       [read, xray, paper]
identifier: 20260224T002627
source:     https://arxiv.org/abs/2601.04748
authors:    Xiaoxiao Li
venue:      arXiv preprint (2026.01)
---

# NAPKIN FORMULA

```
+----------------------------------------------------------+
|                                                          |
|   Acc(S) = alpha / (1 + (|S| / kappa)^gamma) - eps*I(S) |
|                                                          |
|   kappa ~ 50-100 skills (capacity wall)                  |
|   I(S) = semantic confusability (the real killer)        |
|                                                          |
+----------------------------------------------------------+
```

技能选择准确率不是随技能库增大而缓慢下降, 而是在容量阈值(~50-100)处发生相变式崩塌; 真正的杀手不是数量, 而是技能间的语义混淆度.

# PROBLEM

**痛点定义**: 多智能体系统(MAS)虽然能力强, 但 token 开销和延迟巨大(智能体间通信产生大量冗余 token). 能否用单智能体+技能库(SAS)替代, 同时保持性能?

**前人困境**:
- 多智能体系统中各智能体之间需要互相传递上下文, 产生 3-4 倍的 API 调用和大量重复 token
- 直觉上"把多智能体的能力封装成技能给单智能体用"应该可行, 但缺乏系统性的理论分析和实验验证
- 没人研究过技能库规模增大后会发生什么 -- 大家隐含假设"加更多技能总是好的"
- 认知科学早就知道人类选择会在选项过多时崩溃(Hick's Law), 但没人把这个框架迁移到 LLM 上

# INSIGHT

**核心直觉**: 多智能体 -> 单智能体+技能库的转换是可行的, 但有天花板. 这个天花板不是架构缺陷, 而是 LLM 在语义区分上的内禀认知极限 -- 就像人类面对太多相似选择时会"选择瘫痪"一样.

**关键步骤**:
1. *MAS-to-SAS 编译框架*: 形式化定义了三个可编译条件(通信可序列化、共享历史、同质骨干), 将多智能体系统的能力分解(capability decomposition) -> 后端绑定(backend assignment) -> 拓扑内化(topology internalization), 系统性地"压缩"为单智能体. 在 GSM8K/HumanEval/HotpotQA 上, 编译后准确率波动仅 +-2-4%, 但 token 减少 54%, 延迟减少 50%
2. *受控合成实验揭示相变*: 构建了 40 类 x 5 子类 = 200 个技能模板的合成库, 可精确控制库大小和语义相似度. 发现准确率在 kappa~50-100 处发生超线性衰减(gamma > 1), 且语义混淆(而非纯粹的数量)才是主因 -- 20 个高混淆技能比 100 个低混淆技能更致命

# DELTA

**vs SOTA**:
- 编译效率: 单智能体+技能 vs 多智能体, 准确率持平(+0.7%), token 用量减少 53.7%, 延迟减少 49.5%, API 调用从 3-4 次降至 1 次
- 缩放定律: 提出的 Acc = alpha/(1+(|S|/kappa)^gamma) 拟合 R^2 > 0.97, 首次量化了 LLM 技能选择的容量极限
- 混淆度实验: 在 |S|=20 时, 无竞争技能 100% 准确, 每技能 1 个竞争者降 7-30%, 2 个竞争者降 17-63% -- 证明语义混淆是主因
- 层级路由: 在 |S|=120(远超容量阈值)时, 扁平选择 45-63%, 层级路由恢复到 72-85%, 提升 37-40%

**新拼图**: 首次在 LLM 智能体设计与认知科学之间建立定量桥梁 -- 证明 LLM 的技能选择遵循与人类认知类似的容量限制(Hick's Law, 认知负荷理论, 基于相似性的干扰). 这意味着: (1) "无限技能"是幻觉, 必须管理技能库规模; (2) 层级组织对 AI 和人类一样有效; (3) 多智能体系统在技能空间本质上大且高混淆时仍有不可替代的价值.

# CRITIQUE

**隐形假设**:
- 仅在 GPT-4o 和 GPT-4o-mini 两个模型上测试, 容量阈值 kappa~83-92 是否对 Claude/Llama/Gemini 等模型通用? 不同架构的模型可能有完全不同的 kappa 值
- 使用合成技能库而非真实世界的技能分布, 真实场景中技能的语义重叠模式可能更复杂(长尾分布、领域聚集等)
- 编译条件 C1-C3(可序列化通信、共享历史、同质骨干)排除了大量真实 MAS 场景: 辩论系统、并行采样、带私有状态的智能体都无法编译, 但论文标题暗示了更广泛的适用性
- 技能选择准确率 != 端到端任务性能, 选错技能后的错误传播效应未被量化
- 策略复杂度(30 vs 100 vs 300 token)对准确率无显著影响, 这个"零结果"令人意外, 可能暗示实验设计存在天花板效应

**未解之谜**:
- kappa 是模型固有参数还是可以通过训练/微调提高? 如果可以, 这个"认知极限"就不是真正的极限
- 层级路由虽然恢复了 37-40%, 但仍未达到小规模库的水平, 是否存在更优的路由策略(如学习型路由器)?
- 语义混淆度 I(S) 的度量依赖嵌入空间, 不同嵌入模型会给出不同的混淆度, 如何标准化?
- 真实应用中技能库是动态增长的, 何时应该"停止加技能, 转向多智能体"? 论文未给出实用的判断准则

# LOGIC FLOW

```
                    Paper Logic Flow
  ================================================================

  [Question] Can single-agent + skills replace multi-agent systems?
       |
       v
  +---------------------------+
  | Phase 1: Compilation      |
  | MAS --> SAS + Skills      |
  |                           |
  | Conditions:               |
  |  C1: Serializable comm    |
  |  C2: Shared history       |
  |  C3: Homogeneous backbone |
  +---------------------------+
       |
       | YES: -54% tokens, -50% latency, ~same accuracy
       v
  [New Question] Does this scale to large skill libraries?
       |
       v
  +---------------------------+
  | Phase 2: Scaling Study    |
  | Vary |S| from 5 to 200   |
  +---------------------------+
       |
       | FINDING: Phase transition at kappa ~ 50-100
       v
  +---------------------------+
  | Phase 3: Root Cause       |
  | WHY does it break?        |
  |                           |
  | Not quantity alone -->    |
  | Semantic confusability!   |
  | (controlled experiments)  |
  +---------------------------+
       |
       v
  +---------------------------+
  | Phase 4: Mitigation       |
  | Hierarchical routing      |
  | Coarse --> Fine selection  |
  | Recovery: +37-40%         |
  +---------------------------+
       |
       v
  [Conclusion]
  SAS works for |S| < kappa with low confusability
  MAS still needed for large/confusable skill spaces
```

# NAPKIN SKETCH

```
   Accuracy
   100% |*****
        |     *****
        |          **                        <-- Phase transition
        |            *****                       (not gradual!)
        |                 **
        |                   ****
        |                       ***
    0%  +------|---------|----------|-----> |S|
              kappa/2   kappa    2*kappa
              (~50)     (~90)    (~180)


   The Real Story: It's Not About Size, It's About Confusion

   Library A (low confusion):          Library B (high confusion):
   +--------+--------+--------+       +--------+--------+--------+
   | Cook   | Debug  | Paint  |       | Debug  | Debug  | Debug  |
   | Recipe | Code   | Art    |       | Python | Java   | C++    |
   +--------+--------+--------+       +--------+--------+--------+
   Acc = 98% at |S|=60                Acc = 52% at |S|=20 !!


   Fix: Hierarchical Routing (like a menu tree)

   Flat:    [skill1] [skill2] [skill3] ... [skill120]  --> 45% acc
                        (too many choices!)

   Hierarchical:
   Level 1: [Category A] [Category B] [Category C] ... [Cat K]
                  |
   Level 2:   [A.1] [A.2] [A.3] [A.4] [A.5]          --> 82% acc
                        (manageable chunks!)
```
