---
title: Anthropic Agent Harness 论文技术解读：我读完这几篇论文后的笔记
date: 2026-05-20 10:00:00
tags:
  - Anthropic
  - Agent
  - 论文
categories:
  - 学习笔记
---

## 背景：为什么关注 Agent Harness

最近在读 Anthropic 发布的一系列关于 Agent Harness 的工程博文，从 2024 年底的 *Building Effective Agents* 到 2025 年底的 *Effective Harnesses for Long-Running Agents*，再到 2026 年初的 *Harness Design for Long-Running Application Development*。这些文章串在一起看，能看到一条清晰的演进线索：模型能力在飞速进步，但真正决定 Agent 能不能落地做事的，是包裹在模型外面的那套工程系统。

LangChain 在 2026 年 3 月发了一篇 *The Anatomy of an Agent Harness*，里面引用了一个数据：同一个模型，换一套 Harness 架构，Terminal Bench 2.0 的通过率从 52.8% 拉到 66.5%。模型权重没动一个字节，纯靠外围工程改了跑分。这个数字挺震撼的，说明 Harness 不是什么锦上添花的东西，它是 Agent 能力的核心乘法因子。

下面是我读完这几篇博文后的技术梳理，侧重核心贡献、技术方案和实验结果。

---

## 1. Anthropic 的问题定义：长程 Agent 的四重困境

2025 年 5 月，Anthropic 内部想让 Claude 从零搭一个完整的 Web 应用——不是改 Bug，是搭一个产品。任务跑几个小时，单个上下文窗口根本装不下全程。每开一次新 Session，上一轮的记忆就清零。

他们最初按 Context Engineering 的思路来：用外部化记忆（进度文件、功能清单）+ 压缩策略。结果全面溃败，暴露了四种失败模式：

1. **提前交卷（Premature Completion）**：Agent 做了几个功能就宣布"项目完成"，它把已有的代码量当成了完工指标。
2. **环境盲区（Environmental Blindness）**：Agent 在写代码，但环境有 Bug，它写的东西跑不起来，自己不知道。
3. **虚标完成（False Completion）**：功能清单上标了 done，但实际功能是坏的。单元测试过了，端到端根本跑不通。
4. **失忆实习生（Memory Loss）**：每一轮新 Session 都花大量 Token 重新摸索项目结构，像新员工反复问"代码在哪个文件夹"。

这个发现很关键：Context Engineering 解决的是"信息存不住"的问题，但 Agent 的问题远不止存不住。它有时不翻记事本，翻了也不一定照做，做完了也没人验证。**记事本不是问题，管理金鱼翻记事本的流程才是问题。**

---

## 2. 解决方案一：双 Agent Harness（2025 年 11 月博文）

Anthropic 在 *Effective Harnesses for Long-Running Agents* 中提出了一个双 Agent 架构：

### Initializer Agent（初始化 Agent）

只跑一次，做环境搭建：
- 分析需求，拆出 200+ 个功能点
- 生成一份结构化的 JSON 功能清单（每条功能有 `"passes": false` 字段）
- 初始化 Git 仓库、进度文件 `claude-progress.txt`、启动脚本 `init.sh`

为什么用 JSON 而不是 Markdown？Anthropic 发现模型更不容易在 JSON 文件中做不恰当的修改。JSON 的结构化特性天然起到了"防作弊"的作用——Agent 只能改 `passes` 字段，不能删功能、不能改描述。

### Coding Agent（编码 Agent）

每次 Session 只做三件事：
1. **启动协议**：跑 `pwd`、读 `git log`、读 `progress.txt`、读功能清单、执行 `init.sh`
2. **增量开发**：每次只做一个功能，做完提交 Git，更新进度文件
3. **留好交接**：Session 结束时环境必须处于"可合并"状态——无大 Bug、代码有序、文档清晰

这个设计的核心思路是**让单次会话变得足够简单**。通过外部状态和任务分解，模型不需要同时记住所有事情，只需要看清单、选一个未完成的功能、做完、交班。

### 实验效果

改进后 Agent 能连续跑几个小时，逐步完成复杂功能。但仍然有局限：某些功能（比如涉及浏览器原生 alert 弹窗的 Bug）因为工具约束难以检测。

---

## 3. 解决方案二：三 Agent Harness（2026 年 3 月博文）

在 *Harness Design for Long-Running Application Development* 中，Anthropic 把架构进一步推到了三 Agent 模式，灵感来自 GAN（生成对抗网络）的反馈循环：

### 三个 Agent 的分工

1. **Planner（规划者）**：把短 Prompt 扩展成完整的产品 Spec，高层抽象，避免过早陷入技术细节。
2. **Generator（生成者）**：按 Sprint 实现 Spec。每个 Sprint 结束时自评，然后交给 QA 验证。用 Git 管版本。
3. **Evaluator（评估者/QA）**：用 Playwright MCP 和运行中的应用交互，按 Sprint Contract 测试，按产品深度、功能、设计、代码质量打分。**硬性阈值**——任何维度不及格就要求 Generator 返工。

### 关键设计决策

**分离生成和评估**。Anthropic 发现让 Agent 自己评价自己的工作效果很差——它对自己的产出过于宽容。把生成和评估拆开后，单独调优一个"严格"的 Evaluator 要比让 Generator 自我批判容易得多。

**Sprint Contract**。每次 Sprint 开始前，Generator 和 Evaluator 就"什么算完成"达成共识，定义可测试的行为。这避免了范围漂移。

**Context Reset vs Compaction**。对于 Claude Sonnet 4.5，彻底清空上下文、用结构化交接单启动新 Agent（Context Reset），效果优于摘要压缩（Compaction）。但到了 Opus 4.5，模型的长上下文能力更强了，可以用自动压缩做连续 Session。随着模型进步，Harness 的复杂度可以降低。

### 实验结果：Retro Video Game Maker

| 方案 | 耗时 | 成本 | 结果 |
|------|------|------|------|
| 单 Agent | 20 分钟 | $9 | 核心功能坏了，UI 差，空间浪费 |
| 三 Agent Harness | 6 小时 | $200 | 功能完整、UI 精致、集成了 AI 资产生成 |

Evaluator 提供了具体可操作的 Bug 报告（比如填充工具失效、快捷键逻辑错误、API 路由冲突），直接指导了修复。

---

## 4. 训练 Harness vs 生产 Harness：一个被忽视的陷阱

Han Lee 在 *Hidden Technical Debt of AI Systems: Agent Harness* 中提出了一个重要的区分：

| 维度 | 训练 Harness | 生产 Harness |
|------|-------------|-------------|
| 目标 | 探索动作空间，生成优化信号 | 约束动作空间，确保安全可靠 |
| 动作空间 | 最大化（让模型随便试） | 最小化（白名单，默认拒绝） |
| 工具 | 原始、底层、易扩展 | 封装、限定范围、有版本 |
| 失败处理 | 欢迎失败（优化信号） | 抑制失败（fail closed） |
| "好"的定义 | 策略在分布外改进 | 用户任务成功无事故 |

**关键结论**：训练 Harness 应该最宽，生产 Harness 应该最窄，两者之间的差距应该是有意设计、有审计的工程产物，而不是意外。

这个区分解释了为什么有些模型在研究环境中表现很好，部署后却一塌糊涂——他们用了训练时期的宽 Harness 直接上线。

---

## 5. Bitter Lesson 在 Harness 上的体现

Lee 还指出，Harness 中的复杂编排逻辑正在被模型的原生能力吸收。历史模式：

- **2023 年 RAG Pipeline**：Harness 管理分块、嵌入、向量库、重排器（因为模型上下文太小）
- **2024 年 Workflow + Tool Wrapper**：Harness 用 Planner-Executor 循环封装 API（因为工具调用不稳定）
- **2025-2026 年**：模型原生就能交错做规划、执行、反思，复杂的封装、Workflow、多 Agent 拓扑变成了开销

Hyung Won Chung 的那句话很到位：**"为当前算力水平添加结构，然后移除它，因为结构会成为下一个算力水平的瓶颈。"**

这解释了为什么 Anthropic 在 Opus 4.6 上可以直接去掉 Sprint 拆分——模型自身的规划和自评能力已经足够强，Harness 不需要再替它做这些事。

---

## 6. 实用模式总结

从这几篇博文里，我能提炼出几个反复出现的 Harness 模式：

### 状态外置（State Pinning）
不要信任模型的记忆。把任务进度、代码变更、测试结果都写到文件系统里（Git、JSON、txt），用版本控制做 checkpoint。文件系统有一种被低估的工程美德：稳定、透明、可恢复。

### 每次 Session 的启动协议
新 Agent 上线时强制执行：确认目录 → 读历史 → 读进度 → 启动环境 → 才开始干活。像工厂换班时翻交接簿。

### 生成-评估分离
自评效果差，单独做一个严格的 Evaluator 更好调。Evaluator 可以用 Playwright 等工具直接和运行中的应用交互，做端到端验证。

### Success is silent, failures are verbose
Hooks 在 Agent 执行关键操作后运行。类型检查通过了，Agent 听不到任何消息；失败了，错误信息注入上下文触发自我修正。

### 随模型进步做减法
Harness 的最佳实践不是一成不变的。模型每一代能力提升，Harness 的某些控制组件就该被移除——它们变成了瓶颈而非助力。

---

## 7. 对行业趋势的观察

读完这些材料，我有几点感受：

**Harness 正在成为产品本身。** 模型通过 API 谁都能拿到，但工具链、上下文管理、权限控制、验证器、恢复机制不是谁都能搭好的。LangChain 那个 Terminal Bench 2.0 的数据已经说明了：同一个 Opus，不同 Harness 差出十几个百分点。

**Harness 也是技术债务。** Lee 的文章提醒我们，当前 Harness 中的大部分控制逻辑会随着模型进步而过时。如果你把 Harness 当成永久产品表面来做，会花一年时间把它们全拆掉。好的设计是让 Harness 可拆卸、可演进。

**核心能力是乘法关系。** 任何一项接近零，总能力就接近零。模型能力、上下文管理、工具设计、验证器、恢复机制——它们不是加法关系。这也解释了为什么某些 benchmark 一年内出现断崖式提升：不是模型一夜聪明了几十倍，而是缺失的乘法因子被密集补齐了。

**Harness Engineering 不是在替代 Prompt Engineering，而是层层递进。** Prompt Engineering 管"怎么把话说对"，Context Engineering 管"模型开始思考之前，把需要的信息准备好"，Harness Engineering 管"完整的运行时治理"。三层是嵌套关系。

---

## 参考文献

1. Anthropic, *Effective Harnesses for Long-Running Agents*, Nov 2025
   https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents

2. Anthropic, *Harness Design for Long-Running Application Development*, Mar 2026
   https://www.anthropic.com/engineering/harness-design-long-running-apps

3. Anthropic, *Building Effective Agents*, Dec 2024
   https://www.anthropic.com/research/building-effective-agents

4. LangChain, *The Anatomy of an Agent Harness*, Mar 2026

5. Han Lee, *Hidden Technical Debt of AI Systems: Agent Harness*, May 2026
   https://leehanchung.github.io/blogs/2026/05/08/hidden-technical-debt-agent-harness

6. Addy Osmani, *Agent Harness Engineering*, Apr 2026
   https://addyosmani.com/blog/agent-harness-engineering

7. 36氪, *一文读懂 Harness Engineering*, Apr 2026
   https://m.36kr.com/p/3749464991187458

8. Survey, *Agent Harness for Large Language Model Agents: A Survey*, Apr 2026
   https://www.preprints.org/manuscript/202604.0428
