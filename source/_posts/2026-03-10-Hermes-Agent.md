---
title: 用 Hermes Agent 搭自动化工作流：记忆和技能是怎么工作的
date: 2026-03-10 10:00:00
tags:
  - Hermes Agent
  - 记忆系统
  - AI
categories:
  - 工具
---

## 为什么 AI Agent 总是"失忆"

<!-- more -->

用过 Claude、ChatGPT 的人都知道一个痛点：每次新开一个对话，之前的上下文全没了。上周跟它讲过的技术栈选型、代码规范、部署偏好，它一概不记得。你得重新说一遍，像跟一个每天失忆的同事合作。

这不是模型的问题，模型在单次对话里记忆力很好。问题在于 Agent 框架层面没有设计持久化的记忆机制。大部分 AI 助手还是"无状态"的——对话结束，记忆归零。

Hermes Agent 解决这个问题的方式很直接：把记忆拆成两个文件（MEMORY.md 和 USER.md），持久化到磁盘，每次新会话启动时注入 system prompt。同时用 SQLite + FTS5 建立所有历史会话的全文索引，需要的时候可以搜索过去任何一次对话的内容。加上一个技能系统（Skills），让 Agent 能从经验中提取可复用的操作流程。

下面我详细讲讲这套机制是怎么运转的。

---

## 记忆系统：两层文件 + 全文搜索

Hermes Agent 的记忆分三个层级：

### 第一层：MEMORY.md（Agent 的工作笔记）

这个文件存 Agent 自己的观察和经验，比如：

- 用户的项目用什么技术栈
- 这台机器装了什么工具
- 上次部署踩了什么坑
- 某个命令在什么环境下会出错

容量限制 2,200 字符（约 800 tokens）。Hermes Agent 通过内置的 `memory` tool 来管理这个文件，支持 `add`、`replace`、`remove` 三种操作。写入时会做安全扫描，防止 prompt injection 和凭证泄露。

关键设计：memory 是有上限的。当容量超过 80%，Agent 需要主动合并重叠条目或删除过时信息。这不是 bug，是有意为之——有上限迫使 Agent 只保留真正重要的信息，避免记忆膨胀成一堆无用的噪音。

### 第二层：USER.md（用户画像）

这个文件存关于用户的信息：偏好、沟通风格、工作习惯。容量 1,375 字符。

举个例子，如果我总是在让 Hermes 帮我写代码时要求"简洁回答，不要解释原理"，它会把这条偏好记到 USER.md。下次新会话开始，它就知道该怎么回复我。

### 第三层：FTS5 全文搜索（历史会话检索）

MEMORY.md 和 USER.md 是"常驻记忆"——每次会话都会加载到 system prompt。但有些信息不需要每次都加载，比如"上周三我们讨论过的那个 Redis 连接池配置"。

Hermes Agent 把所有历史会话存进 SQLite 数据库，用 FTS5 做全文索引。需要回忆某个具体细节时，Agent 可以调用 `session_search` tool 搜索过去的对话，检索速度大约 20ms。搜索结果由 LLM 做摘要，只返回相关部分。

这三层配合的效果是：

| 记忆类型 | 容量 | 用途 | 成本 |
|---------|------|------|------|
| MEMORY.md | 2,200 chars | 关键环境事实、教训 | 每次会话固定消耗 |
| USER.md | 1,375 chars | 用户偏好、风格 | 每次会话固定消耗 |
| session_search | 无限 | 历史对话回溯 | 按需调用，不消耗 prompt token |

MEMORY.md 和 USER.md 合计约 1,300 tokens，占用 system prompt 的固定开销。这个开销换来的是 Agent 每次都能"记得"关键上下文，不用从零开始。

### 记忆管理：写入审批机制

Hermes Agent 支持一个 `write_approval` 配置项。设为 `true` 时，Agent 写入记忆前需要用户确认。在 CLI 模式下会弹出内联确认；在 Telegram/Discord 等消息平台上，写入会暂存为 "staged" 状态，用户通过 `/memory pending` 查看待确认的写入，用 `/memory approve <id>` 批准。

这个机制对团队协作场景很有用——你不希望 Agent 自动把生产环境的密码写进记忆文件。

---

## 技能系统：从经验中提取操作流程

记忆解决的是"记住事实"的问题，技能解决的是"记住做法"的问题。

Hermes Agent 的 Skills 系统把可复用的操作流程打包成 Markdown 文件，存储在 `~/.hermes/skills/` 目录下。每个 Skill 是一个 SKILL.md 文件，包含：

```yaml
---
name: my-deploy-checklist
description: 我的部署检查清单
version: 1.0.0
metadata:
  hermes:
    tags: [deploy, checklist]
    category: devops
---

# 部署检查清单

## When to Use
用户要求部署项目时

## Procedure
1. 检查 Git 工作区是否干净
2. 跑单元测试 `make test`
3. 构建 Docker 镜像
4. 推送到 registry
5. 更新 Kubernetes deployment
6. 验证 health check 通过

## Pitfalls
- 如果 `make test` 失败，不要跳过直接部署
- Docker build cache 可能导致旧代码被缓存，加 `--no-cache`
```

### 技能是怎么被创建的

Hermes Agent 不需要你手动写技能。当你完成一个复杂任务、遇到错误后恢复、或者纠正了 Agent 的行为时，Agent 会自动判断这个流程是否值得保存为技能。

触发条件包括：

1. **完成了复杂任务**：比如第一次搭建一个完整的 CI/CD pipeline
2. **遇到错误并恢复**：Agent 从失败中学到了排错步骤
3. **收到用户纠正**：比如"不对，我们的部署方式是 git push 而不是 CLI 命令"
4. **发现非平凡工作流**：某个操作序列不是简单的单步调用

创建的 Skill 会保存到 `~/.hermes/skills/` 目录。下次遇到类似场景，Agent 会自动加载对应的 Skill，按照保存的流程执行。

### Progressive Disclosure：按需加载

不是所有技能都同时加载到 prompt 里。Hermes Agent 用三级渐进加载（Progressive Disclosure）来控制 token 消耗：

- **Level 0**：只加载技能列表（名称、描述、分类），约 3k tokens
- **Level 1**：Agent 判断需要某个技能时，加载完整内容
- **Level 2**：加载技能的特定引用文件

这意味着即使你有几十个技能，也不会把 prompt 撑爆。Agent 只在需要时才加载具体技能的内容。

### 技能自维护：Curator

Hermes Agent 内置一个 Curator，在后台运行，负责：

- 合并内容重叠的技能
- 归档长期未使用的技能
- 不会自动删除任何技能（只归档）

这防止了技能目录随着时间膨胀成一堆冗余文件。

---

## 记忆 + 技能 = 自我改进循环

记忆和技能不是孤立工作的，它们构成一个闭环：

```
执行任务 → 验证结果 → 保存持久教训到记忆 → 如果流程可复用，创建/更新技能
```

### 举个具体例子

假设我让 Hermes Agent 帮我部署一个 Node.js 项目。

**第一次**：
- Agent 不知道项目细节，问了很多问题
- 找到部署方式是 `git push` 触发 CI，不是手动部署
- 部署过程中遇到 `node_modules` 缓存问题，加了 `--no-cache`
- 部署成功

**记忆更新**：
- MEMORY.md 写入："项目用 git push 触发 CI 部署，不是 CLI 命令"
- 创建 Skill `deploy-checklist`：包含完整的部署步骤和 `--no-cache` 的坑

**第二次**：
- Agent 自动加载 MEMORY.md，知道用 git push 方式
- 加载 `deploy-checklist` skill，按流程执行
- 直接部署成功，不需要我再解释

**这就是自我改进的实际含义。** 不是模型被重新训练了，而是 Agent 层面的上下文管理变得更好了。

---

## 实际搭建：我怎么配置的

### 安装

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
```

安装后跑 `hermes setup --portal`，一次 OAuth 完成模型配置和工具网关（web search、image generation、TTS、browser）。

### 记忆配置

```yaml
# ~/.hermes/config.yaml
memory:
  memory_enabled: true
  user_profile_enabled: true
  memory_char_limit: 2200
  user_char_limit: 1375
  write_approval: false  # 改成 true 可以开启写入审批
```

### 技能目录

默认技能在 `~/.hermes/skills/`。可以添加外部目录：

```yaml
# ~/.hermes/config.yaml
skills:
  external_dirs:
    - ~/team-skills
    - /home/shared/common-skills
```

### 自动化调度

Hermes Agent 内置 cron，可以定时执行任务并推送到 Telegram/Discord 等平台。比如每周五自动跑一次代码质量检查，结果推送到 Slack：

```
hermes cron add "每周五下午5点跑代码检查" --schedule "0 17 * * 5" --skill code-quality --deliver slack
```

---

## 跟其他方案的对比

| 特性 | Hermes Agent | OpenAI Assistants API | Claude Code | 传统 chatbot |
|------|-------------|----------------------|-------------|-------------|
| 跨会话记忆 | ✅ MEMORY.md + USER.md + FTS5 | ✅ 有限 | ❌ | ❌ |
| 自主技能创建 | ✅ | ❌ | ❌ | ❌ |
| 记忆容量管理 | ✅ 有上限，强制精简 | 无上限 | N/A | N/A |
| 历史会话搜索 | ✅ FTS5 全文索引 | 有限 | ❌ | ❌ |
| 部署灵活性 | 本地/服务器/云端 | 仅 OpenAI 云 | 本地 | 取决于实现 |
| 开源 | ✅ MIT | ❌ | ❌ | 取决于实现 |

Hermes Agent 的定位不是"更好的 chatbot"，是一个会积累经验的协作伙伴。它的记忆和技能系统设计得比较克制——有容量上限、有安全扫描、有归档机制——这些都是为了防止 Agent 的记忆变成垃圾场。

---

## 遇到的问题

1. **记忆写太多垃圾**：刚开始用的时候，我让 Agent 记了很多临时信息（比如某个 PR 的编号）。几天后 MEMORY.md 塞满了过时数据。教训：只记持久性事实，临时信息走 session_search。

2. **技能没有验证步骤**：最早创建的几个技能只有操作步骤，没有"怎么确认做对了"。结果 Agent 按流程跑完但结果是错的，它自己不知道。后来我都加了 Verification 部分。

3. **多技能冲突**：创建了太多细分技能后，遇到类似场景时 Agent 会加载多个相关技能，内容有矛盾。Curator 的合并功能解决了大部分，但偶尔还是需要手动清理。

---

## 总结

Hermes Agent 的核心思路是：**模型不变，用架构层的改进让 Agent 越用越好。** 记忆系统管事实，技能系统管流程，两者配合形成自我改进循环。

如果你的使用场景是长期跟一个 AI Agent 协作（而不是一次性问答），这套机制能省下大量重复解释的时间。一周的真实使用下来，Agent 需要的纠正次数会明显减少。

安装和配置都不复杂，主要花时间在"教 Agent 正确的东西"上——而这个过程本身就是自我改进循环的一部分。
