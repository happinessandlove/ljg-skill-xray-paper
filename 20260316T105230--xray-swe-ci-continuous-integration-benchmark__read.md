---
title:      xray-swe-ci-continuous-integration-benchmark
date:       2026-03-16 Sun 10:52
tags:       [read, xray, paper]
identifier: 20260316T105230
source:     https://arxiv.org/abs/2603.03823
authors:    Jialong Chen, Xander Xu, Hu Wei, Chuan Chen, Bing Zhao
venue:      arXiv preprint (2026.03)
---

# NAPKIN FORMULA

```
+----------------------------------------------------------+
|                                                          |
|   EvoScore = (1/Z) * SUM_t [ gamma^t * a(c_t) ]         |
|                                                          |
|   a(c) in [-1, 1]:  improvement / regression metric      |
|   gamma >= 1:  future-weighted, reward long-term stability|
|   Z = SUM gamma^t:  normalization factor                  |
|                                                          |
+----------------------------------------------------------+
```

代码维护能力不能用一次快照衡量, 需要在数十轮迭代中累积评估; gamma >= 1 意味着越到后期权重越大 -- 你前几轮修得再好, 后面引入回归也会被严厉惩罚.

# PROBLEM

**痛点定义**: 现有基准(HumanEval, SWE-bench)都是"快照式"一次性评估, 无法揭示 LLM 在长期代码维护中积累的退化问题 -- 而真实软件生命周期中, 维护占 60%-80% 的成本.

**前人困境**:
- SWE-bench 等基准给一个 issue, 出一个 patch, 完事. 这只测"修一个 bug"的能力, 不测"连续修几十个 commit 后还能不能保持代码质量"
- 没有基准能衡量"回归" -- 即修了 A 却破坏了 B 的能力退化问题
- 真实代码库的演化是动态的: 每一轮的需求取决于上一轮的代码状态, 不是预先写死的
- 缺乏统一指标来比较不同模型在长期维护中的表现

# INSIGHT

**核心直觉**: 把评估从"一次性考试"变成"学期制考核" -- 让 LLM 在真实代码库的连续演化历史(平均 233 天, 71 个 commit)中, 像真正的开发者一样迭代修改, 然后用未来加权的 EvoScore 衡量长期稳定性.

**关键步骤**:
1. *双智能体协议*: 将任务分为 Architect(分析测试失败 -> 定位代码问题 -> 设计改进方案, 限制最多 5 个最紧急需求) 和 Programmer(理解需求 -> 规划 -> 实现). Architect 看测试, Programmer 看需求文档而非直接看测试 -- 模拟真实开发中"产品经理/架构师 + 程序员"的分工
2. *归一化变化指标 a(c)*: 将每一步的改进/退化映射到 [-1, 1], 改进时按总差距归一化, 退化时按基线能力归一化, 确保不同起点的任务可以公平比较

# DELTA

**vs SOTA**:
- SWE-bench 是单次修复, SWE-CI 是连续 71 轮迭代维护, 评估维度完全不同
- 首次提出 EvoScore, 用 gamma 参数统一了"短期收益偏好 vs 长期稳定性偏好"的权衡
- 数据集规模: 100 个任务, 68 个真实仓库, 平均跨度 233 天, 每个任务 1000+ 行修改
- 消耗超过 100 亿 token 完成完整评估, 揭示了当前模型的真实维护极限
- Claude Opus 系列在全周期中保持领先, 是唯一零回归率超过 0.5 的模型家族

**新拼图**: 证明了"能修 bug != 能维护代码" -- 即使在 SWE-bench 上表现优秀的模型, 在连续迭代中也会频繁引入回归(大多数模型零回归率 < 0.25). 这为 LLM 编程能力的评估增加了"可持续性"这个全新维度.

# CRITIQUE

**隐形假设**:
- 双智能体协议(Architect + Programmer)本身就是一种设计选择, 不同的协议分工可能导致完全不同的评估结果 -- 但论文将其视为"理所当然"
- 测试套件的质量决定了 Architect 能否正确识别问题, 但论文假设所有仓库的测试都足够全面和正确
- 每轮最多 5 个需求的限制是人为设定的, 更激进或更保守的策略可能改变排名
- 仅评估 Python 仓库, 其他语言(尤其是静态类型语言)的回归模式可能截然不同
- pytest 3600 秒超时 + 最多 20 轮迭代 = 评估资源本身成为约束, 某些模型可能受限于时间而非能力

**未解之谜**:
- gamma 的最优值是什么? 论文展示了不同 gamma 下模型排名会变化, 但没有给出"正确"的 gamma -- 这是一个主观选择
- 零回归率如此之低(大部分 < 0.25), 是 LLM 的本质局限还是双智能体协议的设计缺陷?
- 100 亿 token 的评估成本是否可持续? 如何让更多研究者复现?
- 模型在"修到什么程度时停下来不修了"的决策能力完全未被评估 -- 有时不修比乱修更好

# LOGIC FLOW

```
                    Paper Logic Flow
  ================================================================

  [Observation] Existing benchmarks = one-shot snapshots
       |
       | But real software = continuous evolution (60-80% cost)
       v
  +---------------------------+
  | Step 1: Redefine Eval     |
  | Snapshot --> Evolution     |
  | One-shot --> Multi-round   |
  +---------------------------+
       |
       v
  +---------------------------+
  | Step 2: Build Dataset     |
  | 4923 repos --> filter -->  |
  | 100 tasks, 68 repos       |
  | Avg 233 days, 71 commits  |
  +---------------------------+
       |
       v
  +---------------------------+
  | Step 3: Dual-Agent Loop   |
  |                           |
  | Architect: test failures  |
  |   --> locate --> design   |
  |         |                 |
  | Programmer: understand    |
  |   --> plan --> implement  |
  |         |                 |
  | Repeat up to 20 rounds    |
  +---------------------------+
       |
       v
  +---------------------------+
  | Step 4: Measure           |
  | a(c) in [-1,1] per step   |
  | EvoScore = weighted sum   |
  | gamma >= 1: future-heavy  |
  +---------------------------+
       |
       v
  [Findings]
  - Newer models > older (especially post-2026)
  - Claude Opus leads across all gamma
  - Zero-regression rate < 0.25 for most models
  - "Can fix bugs" != "Can maintain code"
```

# NAPKIN SKETCH

```
  Traditional Benchmarks vs SWE-CI

  SWE-bench (snapshot):
  +-----------+         +-----------+
  | Bug       | ------> | Fix       |    Single shot, done.
  | Report    |  agent  | Patch     |
  +-----------+         +-----------+


  SWE-CI (evolution):
  +------+   +------+   +------+         +------+
  | C_0  |-->| C_1  |-->| C_2  |-->...-->| C_71 |
  +------+   +------+   +------+         +------+
     |          |          |                 |
     v          v          v                 v
   a(c_0)    a(c_1)    a(c_2)   ...      a(c_71)
     \          \          \                 /
      \          \          \               /
       +--- gamma-weighted aggregation ----+
                      |
                      v
                  EvoScore


  Zero-Regression Rate Across Models

  1.0 |
      |                                      ** Claude Opus
  0.5 |-------------------------------------- threshold
      |  *  *   *  *  *  *   *  *  *
  0.25|--*--*---*--*--*--*---*--*--*-- most models here
      |
  0.0 +-------------------------------------> Models
       GPT  Qwen  DS  Kimi GLM  ...  Claude

  Lesson: Fixing bugs is easy. NOT breaking things is hard.
```
