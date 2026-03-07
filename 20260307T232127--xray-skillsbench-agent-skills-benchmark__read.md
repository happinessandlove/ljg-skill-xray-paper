---
title:      xray-skillsbench-agent-skills-benchmark
date:       2026-03-07 Sat 23:21
tags:       [read, xray, paper]
identifier: 20260307T232127
source:     https://arxiv.org/abs/2602.12670
authors:    Xiangyi Li, Wenbo Chen, Yimin Liu et al. (41 authors)
venue:      arXiv preprint (2026.02)
---

# NAPKIN FORMULA

```
+----------------------------------------------------------+
|                                                          |
|   Delta(skills) = f(domain, harness, curation)           |
|                                                          |
|   curated >> self-generated                              |
|   focused(2-3) >> comprehensive(4+)                     |
|   specialized_domain >> general_domain                   |
|                                                          |
+----------------------------------------------------------+
```

技能的有效性不取决于"有没有技能", 而取决于三个交互变量: 领域专业度、执行框架(harness)、以及人工策展质量. 自生成技能不仅无用, 反而有害.

# PROBLEM

**痛点定义**: Agent Skills(结构化的过程性知识包)被广泛用于增强 LLM 智能体, 但缺乏系统性的跨领域评估——没人知道 Skills 到底在什么条件下有效、什么条件下无效.

**前人困境**: 此前的评估要么局限于单一领域(如 SWE-bench 只看软件工程), 要么没有对比有/无 Skills 的配对实验. 没有统一的 benchmark 来回答"Skills 到底值不值得投入"这个根本问题.

# INSIGHT

**核心直觉**: 把 Skills 本身当作实验变量, 设计配对实验(有 Skills vs 无 Skills vs 自生成 Skills), 横跨 11 个领域、84 个任务、7 种模型-框架组合, 用确定性程序化验证(非 LLM-as-judge)来量化 Skills 的真实增量.

**关键步骤**:
1. 构建三条件配对评估框架: No Skills / Curated Skills / Self-Generated Skills, 在相同任务上对比, 隔离 Skills 的因果效应
2. 使用确定性断言而非 LLM 评判, 消除评估噪声; 同时设置反作弊机制防止模型走捷径

# DELTA

**vs SOTA**: 这是首个系统评估 Agent Skills 的跨领域 benchmark. 7,308 条有效评估轨迹, 覆盖 Claude Code / Gemini CLI / Codex CLI 三大主流 harness.

**新拼图**:
- 人工策展的 Skills 平均提升 16.2pp, 但分布极不均匀(医疗 +51.9pp vs 软件工程 +4.5pp)
- 自生成 Skills 平均降低 1.3pp, 证明模型无法为自己写出有效的过程性知识
- 2-3 个聚焦 Skills 最优(+18.6pp), 4+ 个 Skills 收益骤降(+5.9pp)
- 小模型(Haiku)+Skills 可以追平大模型(Opus)无 Skills 的表现, Skills 部分补偿模型容量
- 不同 harness 对 Skills 的利用率差异巨大: Claude Code 最稳定, Codex CLI 经常忽略 Skills

# CRITIQUE

**隐形假设**:
- 假设终端容器化环境可代表所有智能体场景, 但 GUI 交互、多智能体协作等场景未覆盖
- 注入 Skills 必然增加 context 长度, 无法完全分离"结构化过程知识"与"更多上下文信息"的效应
- 84 个任务虽跨 11 领域, 但每个领域样本量有限(部分领域仅几个任务), 领域级结论的统计置信度存疑
- 确定性验证偏向可程序化评估的任务, 排除了需要主观判断的真实场景

**未解之谜**:
- 为什么自生成 Skills 反而有害? 是元认知缺陷还是 context 污染? 论文未深入分析机制
- Skills 与 RAG / Few-shot 等其他知识注入方式的对比缺失
- 最优 Skills 的设计原则(长度、粒度、示例比例)仍是经验性的, 缺乏理论框架
- 负面案例(如 taxonomy-tree-merge -39.3pp)的根因分析不够深入

# LOGIC FLOW

```
+------------------+     +-------------------+     +------------------+
|  OBSERVATION     |     |  BENCHMARK DESIGN |     |  EVALUATION      |
|  Skills widely   |---->|  84 tasks x       |---->|  7,308 valid     |
|  used but never  |     |  11 domains x     |     |  trajectories    |
|  systematically  |     |  3 conditions x   |     |  deterministic   |
|  evaluated       |     |  7 model-harness  |     |  assertions      |
+------------------+     +-------------------+     +------------------+
                                                          |
                                                          v
+------------------+     +-------------------+     +------------------+
|  IMPLICATIONS    |     |  KEY FINDINGS     |     |  ANALYSIS        |
|  - Curate, don't |<----|  - Curated +16pp  |<----|  - Domain split  |
|    auto-generate |     |  - Self-gen -1.3pp|     |  - Harness split |
|  - Focus > flood |     |  - 2-3 optimal    |     |  - Scale split   |
|  - Domain matters|     |  - Domain varies  |     |  - Failure modes |
+------------------+     +-------------------+     +------------------+
```

# NAPKIN SKETCH

```
         SKILLS EFFECTIVENESS MAP

  Impact |
  (pp)   |
   +50   | *Healthcare
         |
   +40   | *Manufacturing
         |
   +30   |
         |  *Cybersecurity
   +20   |  *NatSci
         |              <-- Sweet spot: 2-3 focused skills
   +10   |
         |  *Math
    +5   |  *SWE
     0   |------------------------------------------
         |  *** Self-generated (avg -1.3pp)
    -5   |
         +------------------------------------------
              Specialized <-------> General
              domains               domains

  TAKEAWAY: Skills = domain-specific amplifier,
            NOT universal booster
```
