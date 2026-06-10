---
title: OpenClaw vs Hermes
published: 2026-04-20
description: OpenClaw 和 Hermes 是两个非常有代表性的项目。它们代表了 AI Agent 的两条不同进化路径。
tags: [AI, Agent]
category: AI
---


## OpenClaw 与 Hermes 的共同点
**OpenClaw** 和 **Hermes** 都属于通用 Agent 系统。它们都不是单点脚本，也不是某个聊天渠道里的bot。核心结构都是一样的：
``` markdown
LLM（大脑） + Tools（手） + Memory（记忆） + Planner（决策）
```
它们都在尝试把模型、工具、会话、记忆、Skills、消息入口和本地运行环境接成一套可以长期使用的系统。

## OpenClaw 与 Hermes 区别

工程重心不同，**OpenClaw** 更像一个本地优先的 **Agent Gateway**,重点是把真实世界的入口、会话、设备和权限接起来。 **Hermes** 更像一个学习型 **Agent Runtime**,重点是让 Agent 在执行过程中沉淀经典，下次少走弯路。
``` mermaid
flowchart TD
    %% 标题使用纯文本换行，避免 HTML 导致的边框高度计算错误
    Title["`**OpenClaw vs Hermes**
    同属通用 Agent，但系统厚度长在不同位置`"]
    style Title fill:none,stroke:none,font-size:18px

    %% 核心节点
    A["`**通用 Agent 系统**
    模型、工具、会话、记忆、Skills、入口、执行环境`"]
    
    B["`**为什么容易混淆？**
    关键词高度重叠：
    Gateway / Skills / Memory / Messaging / Local-first
    看功能列表很像，但架构重心并不一样`"]
    
    %% 左右对比分支
    C["`**OpenClaw 更厚**
    
    **Gateway 控制面**
    多渠道入口
    会话与路由
    设备节点与权限治理`"]
    
    D["`**Hermes 更厚**
    
    **学习型 Runtime**
    Agent Loop
    会话检索
    Skill 沉淀与用户建模`"]
    
    %% 底部阅读提示
    Footer["`读法：先看系统重心，再看功能清单`"]

    %% 连线与不可见占位符 (~~~) 保证布局从上到下排列
    Title ~~~ A
    A --> B
    B --> C
    B --> D
    C ~~~ Footer
    D ~~~ Footer

    %% 定义清爽的配色方案，增加 stroke-width 使边框更清晰
    classDef topBox fill:#f0fbff,stroke:#2596be,stroke-width:2px,color:#2c3e50
    classDef midBox fill:#fff9e6,stroke:#d99a29,stroke-width:2px,color:#2c3e50
    classDef leftBox fill:#e8f0fe,stroke:#689df6,stroke-width:2px,color:#1a1a1a
    classDef rightBox fill:#e6f4ea,stroke:#5bb974,stroke-width:2px,color:#1a1a1a
    classDef footerBox fill:#f8f9fa,stroke:#b0b3b8,stroke-width:1.5px,color:#4b5055

    %% 绑定样式
    class A topBox
    class B midBox
    class C leftBox
    class D rightBox
    class Footer footerBox
```

OpenClaw 和 Hermes 之所以会被放在一起比较，不是误会。它们确实有相似的系统边界。一个通用 Agent 系统，通常不只是“模型加提示词”。 把 Harness拆一下，会有以下几层：
* LLM 更像引擎/大脑
* Agent Loop 更像工作节奏
* Harness 更像给 Agent 配好的工位、规范、工具链、权限和验收机制。
* Memory、Skills、Context、工具调用、执行环境，都会影响最后的可用性。

从这个角度来看，OpenClaw 和 Hermes 都已经越过了**模型包装器**的阶段。它们都在做一件更接近真实使用的事：把 Agent 放进一个长期运行的工程环境里。
这也是容易混淆的根源。他们都是 Agent,但是关键点在不同位置。

OpenClaw 主要在于入口、控制面和多设备协同上。

Hermes 主要在执行循环、技能沉淀和跨会话经验复用上。这一点看清楚，后面的架构、安全、迁移和选型都会更容易理解。

## 系统的不同重心
很多对比容易卡住，是因为一上来就列功能表，功能表有用，但容易把人带偏。两个系统都支持聊天入口、工具调用、skills、memory、模型切换，于是看起来像同类。

跟那个有用的拆法是把 Agent 系统分为几层：入口、控制面、执行循环、经验曾。

OpenClaw 的 README 里有一句话：“The Gateway is just the control plane - the product is the assistant.” 它不只是在做一个聊天机器人，更像一个本地优先，可接多入口、可接设备节点、可接设备节点、可接 WebChat 和 Dashboard 的控制面。
``` mermaid
flowchart TD
    %% 标题使用纯文本换行，避免 HTML 导致的计算错误
    Title["`**OpenClaw 的重心**
    先把入口和控制面做厚，再让 Agent 在秩序里工作`"]
    style Title fill:none,stroke:none,font-size:20px

    %% 顶部输入渠道层
    T1["Telegram"]
    T2["Discord"]
    T3["WeChat"]
    T4["WebChat"]

    %% 核心网关层
    GW["`**OpenClaw Gateway**
    会话、路由、渠道连接、控制面`"]

    %% 核心控制与逻辑层
    S["`**Session**
    会话 / 路由`"]
    P["`**Policy**
    配对 / 白名单 / 沙箱`"]
    R["`**Runtime**
    模型 + 工具`"]

    %% 底部执行与数据节点层
    DN["`**Device Nodes**
    macOS / iOS / Android`"]
    WS["`**Workspace**
    AGENTS / SOUL / USER / Skills`"]

    %% 底部说明文本
    Footer["`这张图只保留主链路：
    入口进入 Gateway，再落到会话、策略、运行时和工作区。
    `"]
    style Footer fill:none,stroke:none,color:#88929c,font-size:14px

    %% 排版占位，确保 Title 和 Footer 居中且不被连线打乱
    Title ~~~ T2
    Title ~~~ T3

    %% 渠道连接到网关
    T1 --> GW
    T2 --> GW
    T3 --> GW
    T4 --> GW
    
    %% 网关向下分发
    GW --> S
    GW --> P
    GW --> R
    
    %% 逻辑层连接到执行节点与工作区
    S --> DN
    P --> DN
    P --> WS
    R --> WS
    
    DN ~~~ Footer
    WS ~~~ Footer

    %% 定义与原图 1:1 匹配的节点颜色样式
    classDef inputBox fill:#e8f0fe,stroke:#689df6,stroke-width:2px,color:#3c4043
    classDef gwBox fill:#f0fbff,stroke:#2596be,stroke-width:2px,color:#2c3e50
    classDef policyBox fill:#fff9e6,stroke:#d99a29,stroke-width:2px,color:#2c3e50
    classDef runtimeBox fill:#e6f4ea,stroke:#5bb974,stroke-width:2px,color:#1a1a1a
    classDef wsBox fill:#f8f9fa,stroke:#b0b3b8,stroke-width:2px,color:#4b5055

    %% 绑定样式到具体节点
    class T1,T2,T3,T4 inputBox
    class GW gwBox
    class S,P policyBox
    class R,DN runtimeBox
    class WS wsBox
```

Hermes 的 README 则把自己定义成 **“The self-improving AI agent”** 。值得看的地方是 build-in learning loop:从经验中创建skill,在使用中改进skills，搜索过去的会话，并逐步构建用户模型。
``` mermaid
flowchart TD
    %% 标题使用纯文本原生换行，防止解析器崩溃
    Title["`**Hermes 的重心**
    把执行过程变成可检索、可复用的经验资产`"]
    style Title fill:none,stroke:none,font-size:20px

    %% 定义所有节点
    H1["用户任务"]
    H2["`**Hermes Agent Loop**
    模型调用 + 工具调用 + 任务迭代`"]
    
    H3["`**Session Store**
    SQLite + FTS5`"]
    H4["`**Memory Provider**
    偏好 / 事实`"]
    
    H5["任务输出"]
    H6["工具结果回流"]
    
    H7["`**Skill Manager**
    创建 / 修补`"]
    H8["`**~/.hermes/skills**
    过程记忆，下次任务可复用`"]

    %% 占位符确保标题在最上方
    Title ~~~ H1
    
    %% 主干流程
    H1 --> H2
    H2 --> H5
    
    %% 左侧记忆与状态循环
    H2 --> H3
    H3 --> H4
    H4 --> H2
    
    %% 右侧工具循环
    H2 --> H6
    H6 --> H2
    
    %% 右下方 Skill 沉淀
    H5 --> H7
    H7 --> H8

    %% 定义清晰的 CSS 样式类
    classDef cyanBox fill:#e8f4fd,stroke:#2c9ed2,stroke-width:2px,color:#2c3e50
    classDef greenBox fill:#e1f9e8,stroke:#34a853,stroke-width:2px,color:#2c3e50
    classDef greyBox fill:#f4f5f6,stroke:#8a8d90,stroke-width:2px,color:#2c3e50
    classDef orangeBox fill:#fff2e1,stroke:#f7ac4c,stroke-width:2px,color:#2c3e50

    %% 绑定样式
    class H1,H5,H6 cyanBox
    class H2 greenBox
    class H3,H4 greyBox
    class H7,H8 orangeBox
```

背后的团队也不同。OpenClaw 由独立开发者 Peter Steinberger 创建，凭借极简安装和多渠道接入快速积累了大量 GitHub Star。不过 Steinberger 今年 2 月加入 OpenAI，项目已交给社区基金会维护，后续的发展节奏还在观察。Hermes 背后是 Nous Research，Hermes 系列模型（Hermes 3、Hermes 4）的缔造者，对模型训练和推理优化有第一手积累，上线不到两个月社区增长很快。

用一句话概括：

**OpenClaw 管入口和秩序，Hermes 管执行和经验。**

## OpenClaw：把真实世界接进来
OpenClaw 不只是聊天窗口，它更像一个按会话串行执行的作业系统。我们看到的是聊天入口，系统内部跑的是一套消息接入、路由、会话和记忆加载机制。

OpenClaw 的定位是 personal AI assistant。你把它跑在自己的设备或服务器上，然后通过熟悉的聊天入口和它交互。它的渠道列表很长：WhatsApp、Telegram、Slack、Discord、Google Chat、Signal、iMessage、BlueBubbles、IRC、Microsoft Teams、Matrix、Feishu、LINE、Mattermost、Nextcloud Talk、Nostr、Synology Chat、Tlon、Twitch、Zalo、WeChat、WebChat。代码仓库里还有专门的 macOS menu bar app、iOS/Android node、Voice Wake、Talk Mode、Live Canvas（A2UI）。

这个细节有分量。对很多真实用户来说，Agent 的第一道门槛经常还没到 ReAct，而是这些更朴素的问题：
* 我能不能从 Telegram 发它？
* 我能不能从 Discord 群里唤起它？
* 我能不能让它跑在家里的小机器上？
* 我能不能让家人、同事、设备节点以不同权限接入？
 
OpenClaw 的 Gateway 正是在处理这些问题。
``` mermaid
flowchart TD
    %% 顶部用户与入口层
    U["`用户 / 群聊`"]
    E["`聊天入口
    Telegram / Discord
    WeChat / WebChat`"]
    
    %% 核心网关层
    GW["`OpenClaw Gateway
    控制面
    会话、路由、渠道连接`"]

    %% 中间路由与策略层
    S["`Session
    会话 / 路由`"]
    P["`Policy
    配对 / 白名单 / 沙箱`"]
    D["`Device Nodes
    macOS / iOS / Android`"]
    W["`Workspace
    AGENTS / SOUL / USER
    / Skills`"]

    %% 底部运行时层
    AR["`Agent Runtime
    模型 + 工具`"]

    %% 定义主链路走向
    U --> E
    E --> GW

    %% 网关向下分发
    GW --> S
    GW --> P
    GW --> D
    GW --> W

    %% 汇总到 Agent 运行时
    S --> AR
    P --> AR
    W --> AR

    %% 定义与原图匹配的 CSS 颜色和边框样式
    classDef blueBox fill:#eaf4fc,stroke:#5b9bd5,stroke-width:2px,color:#2c3e50
    classDef cyanBox fill:#eaf6f6,stroke:#45a7a7,stroke-width:2px,color:#2c3e50
    classDef orangeBox fill:#fef6e9,stroke:#eda34a,stroke-width:2px,color:#2c3e50
    classDef greenBox fill:#eefaf0,stroke:#64c375,stroke-width:2px,color:#2c3e50
    classDef greyBox fill:#f8f9fc,stroke:#a6b0c3,stroke-width:2px,color:#2c3e50

    %% 绑定样式
    class U,E blueBox
    class GW cyanBox
    class S,P orangeBox
    class D,AR greenBox
    class W greyBox
```

它先把入口和控制面做厚，再让 Agent 在这个秩序里工作。

这类题系统的难点，不只是发起一次模型调用。更麻烦的是多渠道状态、会话隔离、群聊激活规则、消息分片、凭据存放、配对策略、设备节点权限、WebSocket 控制面、Dashboard，以及一堆看起来不起眼但上线后每天都会碰到的边界条件。所以把 OpenClaw 简化成"一个工具箱"，多少有些低估它了。它更像一个 Agent 版的个人通信与设备控制平面。

## Hermes：让 Agent 把经验写下来
Hermes 的重心不一样。它当然也有 CLI 和 Messaging Gateway，也能接 Telegram、Discord、Slack、WhatsApp、Signal 等入口。但如果只从"能接哪些平台"看 Hermes，就会错过它最有意思的地方。

只抓和 OpenClaw 对比最相关的一点：**Hermes 把 Agent 的执行过程当成长期资产**。它的 README 里，closed learning loop 被放在非常靠前的位置：Agent-curated memory、autonomous skill creation、skills self-improve、FTS5 session search、Honcho user modeling。

翻成工程语言，大概是四件事：
1. 当前任务怎么跑，靠 Agent loop 和 tool runtime。
2. 过去做过什么，靠 session store 和搜索召回。
3. 哪些流程值得复用，沉淀成 skill。
4. 用户长期偏好和行为模式，交给 memory provider 和 Honcho 用户建模。

代码里也能看到这条线。`run_agent.py` 负责完整的 tool calling conversation loop；`model_tools.py` 负责工具发现和分发；`skill_manager_tool.py` 开头就写着"Skills are the agent's procedural memory"，允许 Agent 创建、更新、删除 skills，把成功路径变成 reusable procedural knowledge；`hermes_state.py` 用 SQLite + FTS5 存会话和做全文检索，支持 WAL 模式的并发读写和基于 source tag（cli、telegram、discord 等）的过滤。

Hermes更关心的问题是：  
**当它完成了一个复杂任务以后，这段经验会不会消失？下次做同类任务，它能不能少试错？**
``` mermaid
flowchart TD
    %% 顶部输入层
    U["`用户任务`"]

    %% 核心循环层
    Loop["`Hermes Agent Loop
    模型调用 + 工具调用
    任务迭代`"]

    %% 中间执行与状态层
    Tool["`工具结果
    回流到 Loop`"]
    
    Output["`任务输出`"]
    
    Session["`Session Store
    SQLite + FTS5
    会话检索`"]
    
    Memory["`Memory Provider
    长期偏好 / 事实`"]

    %% 底部 Skill 沉淀层
    SkillMgr["`Skill Manager
    创建 / 修补 Skill`"]
    
    Skills["`~/.hermes/skills
    过程记忆
    下次任务复用`"]

    %% ---------------- 排版辅助 ----------------
    %% 强制让中间层的四个模块在同一水平线上排开
    Tool ~~~ Output ~~~ Session ~~~ Memory

    %% ---------------- 连线逻辑 ----------------
    U --> Loop

    %% 工具调用的双向回流
    Loop --> Tool
    Tool --> Loop

    %% 记忆与会话的双向读取/写入
    Loop --> Session
    Session --> Loop
    
    Loop --> Memory
    Memory --> Loop

    %% 任务输出与 Skill 沉淀链路
    Loop --> Output
    Output --> SkillMgr
    SkillMgr --> Skills

    %% Skill 经验资产在下次任务中复用 (长回流线)
    Skills --> Loop

    %% ---------------- 颜色与样式定义 ----------------
    classDef blueBox fill:#eaf4fc,stroke:#5b9bd5,stroke-width:2px,color:#2c3e50
    classDef greenBox fill:#eefaf0,stroke:#64c375,stroke-width:2px,color:#2c3e50
    classDef cyanBox fill:#eaf6f6,stroke:#45a7a7,stroke-width:2px,color:#2c3e50
    classDef greyBox fill:#f8f9fc,stroke:#a6b0c3,stroke-width:2px,color:#2c3e50
    classDef orangeBox fill:#fef6e9,stroke:#eda34a,stroke-width:2px,color:#2c3e50

    %% ---------------- 绑定样式 ----------------
    class U blueBox
    class Loop greenBox
    class Tool,Output cyanBox
    class Session,Memory greyBox
    class SkillMgr,Skills orangeBox
```

这套设计想解决的，是一个老问题：**Agent 每次从零开始，成本很高。**

如果它已经踩过坑、跑通过流程、修过某个复杂错误，就可以把这条路径保存下来。下一次同类任务，不需要重新"聪明"一次，只要复用之前沉淀过的工作方法。有 Reddit 用户反馈，Agent 在两小时内自动生成了三份技能文档后，重复性研究任务的速度提升了约 40%。这类数据还需要更多验证，但方向感是清楚的。

它把很多 Agent 产品挂在嘴边的"长期记忆"，往 procedural memory 的方向又推了一步。

## Skill：同一个词，但又不同
简单来对比，OpenClaw 是人工写的Skill，Hermes 是自动生成 Skill。方向没错，但容易过度简化。SKill 可以从提示词开始，但它更像一个 Agent work unit。它可以是一个目录，里面有`SKILL.md`，也可以有参考资料、脚本、模板、资产和踩坑记录。

把这条线放到 OpenClaw 和 Hermes 上，会更容易看懂差异。

OpenClaw 有一套完整的技能体系。代码仓库里已经内置了 50 多个 skill 目录（1password、discord、slack、github、coding-agent、apple-notes、voice-call 等），支持 AgentSkills-compatible skill folders，每个 skill 是一个包含 `SKILL.md` 的目录。系统按 bundled skills、managed/local skills、personal agent skills、project agent skills、workspace skills 分层，通过加载优先级和 gating 做治理。

这更像一个工程化技能目录：
* 哪些技能来自系统；
* 哪些来自本地用户；
* 哪些属于某个 workspace；
* 哪些需要特定环境变量、二进制或配置；
* 哪些优先级更高；
* 哪些第三方 skills 要当成外部输入来处理。

Hermes 的 skill 侧重点更像"过程记忆"。

它的 `skill_manager_tool.py` 开头就写着：Skills are the agent's procedural memory: they capture how to do a specific kind of task。它们记录的是"怎么做某类具体任务"，不是泛泛的偏好事实。系统提示里也会提醒 Agent：完成复杂任务、修复棘手错误、发现非平凡 workflow 后，可以用 `skill_manage` 把方法保存下来；如果发现 skill 过时或错误，就直接 patch。Hermes 的 skills 目录也预置了 26 个类别（research、software-development、data-science、devops、mlops 等），兼容 agentskills.io 开放标准。

**OpenClaw 的 skill 更像团队里的 SOP 库。Hermes 的 skill 更像一个强执行者不断更新的工作笔记。**

SOP 库的优点是可控、可审计、适合团队治理。工作笔记的优点是贴近真实任务、迭代快、能把个体使用经验滚起来。代价也不同。OpenClaw 的 skill 质量更多取决于人和社区。Hermes 的自动沉淀有想象力，但也需要复看和修剪，否则"经验"也可能变成"惯性错误"。

## Memory
先对几个基础词进行定义：
* Context 是这次任务的临时上下文；
* Knowledge 更偏稳定知识；
* Memory 会随时间变化，和用户、任务、历史互动相关；
* Experience 是从原始记录里蒸馏出来的方法和教训。

用这组词再看 OpenClaw 和 Hermes，会更清楚。

OpenClaw 的记忆走"文件即记忆"路线。核心文件包括定义 Agent 性格的 `SOUL.md`、记录用户偏好的 `USER.md`、按日期组织的日常日志 `memory/*.md`，以及精选长期记忆的 `MEMORY.md`。语义检索工具负责查找，上下文压缩前执行一次静默记忆写入，防止压缩丢信息。更像给 Agent 一个笔记本。

Hermes 的记忆更系统化，分三层。
| 层级 | 内容 | 特点 |
| :--- | :--- | :--- |
| 会话记忆 | 当前对话上下文 | 仅维持于当次会话 |
| 持久记忆 | 跨会话的事实和偏好<br/>( `MEMORY.md` + `USER.md` ) | 自动累积，每次对话带上关键信息 |
| 技能记忆 | 从成功任务中学到的解决方案模式 | SQLite + FTS5 全文检索，支持 LLM 摘要召回，可搜索、可复用、自我迭代 |

更像给 Agent 装了一个搜索引擎式的大脑。所以它们都讲 Memory，但**侧重点不同**。OpenClaw 的 Memory 更容易和 **"身份、会话、工作区边界"** 放在一起看。Hermes 的 Memory 更容易和**执行轨迹、搜索召回、Skill 沉淀、用户建模"**放在一起看。

## 安全措施

### OpenClaw：信任模型 + 配置审计
OpenClaw 的安全模型是"personal assistant"（one trusted operator），不是多租户共享。SECURITY.md 里写得很清楚：authenticated Gateway callers are treated as trusted operators for that gateway instance。

它提供了 openclaw security audit --deep 命令来扫描网关配置风险，DM pairing、allowlist、sandbox 和 doctor 机制共同构成安全边界。代码仓库里的安全文档非常详细，覆盖了 Workspace Memory Trust Boundary、Plugin Trust Boundary、Temp Folder Boundary、Sub-agent delegation hardening 等多个层面。

不过 OpenClaw 在安全方面的历史不太平静。今年 2 月被曝出 WebSocket Token 泄露漏洞，外部安全团队发现第三方 Skill 存在数据外泄和 Prompt 注入风险，ClawHub 上也发现了一批恶意 Skill。官方的响应和修复速度不慢，但这些事件提醒我们：一个入口足够多、生态足够开放的系统，攻击面也会相应扩大。

### Hermes：纵深防御 + 容器隔离
Hermes 在部署层面更强调逐层收紧。它支持六种 terminal backend（local、Docker、SSH、Daytona、Singularity、Modal），其中 NixOS 模式提供了 Namespace 隔离（`ProtectSystem=strict`）。
安全策略包括：
* 危险命令审批：终端命令、文件写入等默认需要人工确认，超时未批准自动拒绝
* 容器隔离：可以把 Agent 的执行环境限制在 Docker 或远程后端里
* 凭据过滤：防止敏感信息泄露到上下文
* 上下文注入扫描：检测 Prompt 注入风险

截至目前，Hermes 没有被公开披露过重大安全漏洞。当然，这也和它上线时间更短、用户规模更小有关，不能简单等同于"更安全"。

一句话区分：**OpenClaw 更多在"人该怎么管 Agent"这一层做安全，Hermes 更多在"Agent 运行时该怎么被约束"这一层做安全。**

| 维度 | OpenClaw | Hermes Agent |
| :--- | :--- | :--- |
| **大类** | 通用 Agent 系统 | 通用 Agent 系统 |
| **核心定位** | 本地优先个人 AI 助手，重点是 Gateway 控制面 | self-improving AI agent，重点是学习型执行循环 |
| **入口能力** | 很强，覆盖 25+ 聊天渠道、WebChat、macOS/iOS/Android 节点、Live Canvas | 支持 CLI 和 Telegram/Discord/Slack/WhatsApp/Signal/Email |
| **架构重心** | Gateway、会话、路由、设备节点、权限、Dashboard | Agent loop、工具分发、skills、memory、session search、执行后端 |
| **技能体系** | AgentSkills-compatible，强调加载来源、优先级、gating 和治理；50+ 内置 skill | skills 作为 procedural memory，强调自动创建、修补和复用；26 个类别 |
| **记忆方向** | workspace 文件、memory 插件、语义检索与 agent state | SQLite + FTS5 会话搜索、memory provider、Honcho 用户建模 |
| **安全策略** | 信任模型 + 配置审计 + DM pairing / allowlist / sandbox | 纵深防御：审批 + 容器隔离 + 凭据过滤 + 注入扫描 |
| **技术栈** | Node.js / TypeScript | Python 3.11 |
| **安装体验** | `openclaw onboard --install-daemon`，偏 Gateway 和渠道上手 | `hermes setup`、`hermes model`、`hermes gateway`，偏 CLI 和模型配置 |
| **模型支持** | 多 provider，支持 OAuth + API key failover | 200+ 模型（OpenRouter、Anthropic、OpenAI、智谱、Kimi、MiniMax 等），一条命令切换 |
| **迁移支持** | 更偏 OpenClaw<br/>自身跨机器迁移 | 支持从 OpenClaw 导入 persona、memory、skills、<br/>allowlist、部分 settings 和 secrets |
| **更适合** | 多渠道个人助理、<br/>设备联动、团队入口治理 | 长期重复任务、研究工作流、个人经验沉淀、RL<br/>轨迹数据生成 |

这张表核心只有一句：**OpenClaw 的价值在"接入复杂世界"，Hermes 的价值在"沉淀复杂经验"**。

## 选 OpenClaw，还是 Hermes
从三个角度想：  
第一，你的主要复杂度在哪？
如果复杂度在入口，比如 Telegram、Discord、Slack、WeChat、WebChat、iOS、Android、macOS 节点、群聊、私聊、配对、远程 Gateway，那 OpenClaw 更自然。如果复杂度在任务本身，比如研究、代码修改、数据分析、日报周报、PR 审查、重复性排障、长链路自动化，那 Hermes 更值得试。

第二，你更担心不可控，还是更担心不成长？  
如果更担心不可控，可以先看 OpenClaw 的 Gateway、allowlist、pairing、sandbox、doctor、workspace 边界。  
如果更担心 Agent 每次都从零开始，可以先看 Hermes 的 skills self-improve、FTS5 session search、memory provider、Honcho 用户建模。

第三，你是一个人用，还是要带进团队流程？  
个人折腾、研究工作流、长任务，Hermes 的成长性更有吸引力。团队协作、多入口接入、设备联动、权限治理，OpenClaw 的控制面价值更高。当然，也可以两者都试。

看 Hermes，就看它的学习循环有没有帮你减少重复劳动。

看 OpenClaw，就看它的 Gateway、渠道、会话和设备治理有没有让你的 Agent 更容易进入日常场景。

更直接一点：**我现在缺的，是入口、秩序，还是经验？**