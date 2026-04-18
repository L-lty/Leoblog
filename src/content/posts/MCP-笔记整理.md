---
title: Code execution with MCP
published: 2026-01-01
description: 当 Agent 连接越来越多 MCP 工具时，传统“直接工具调用”的方式会带来严重的上下文开销；更高效的做法是让 Agent 在代码执行环境中，通过写代码来调用 MCP 工具。
tags: [AI, Agent, MCP]
category: AI
draft: false
---


# Code execution with MCP

## 一、文章主题

这篇文章讨论的是：

**当 Agent 连接越来越多 MCP 工具时，传统“直接工具调用”的方式会带来严重的上下文开销；更高效的做法是让 Agent 在代码执行环境中，通过写代码来调用 MCP 工具。**

---

## 二、为什么传统 MCP 工具调用会越来越低效

随着 MCP 生态发展，Agent 可能会连接：

- 成百上千个工具
- 数十个 MCP Server
- 大量外部系统与数据源

这时会出现两个核心问题。

### 1. 工具定义本身占用大量上下文

传统 MCP Client 往往会把所有工具定义一次性加载进模型上下文。

例如一个工具定义中可能包含：

- 工具名
- 描述
- 参数 schema
- 返回值说明

当工具很多时，模型在真正处理用户请求之前，就要先“读完”这些定义，导致：

- Token 消耗变大
- 响应变慢
- 成本增加

### 2. 中间结果不断穿过模型上下文

传统模式下，工具返回的中间结果需要经过模型，再交给下一个工具。

例如：

1. 从 Google Drive 取出会议纪要
2. 模型读取整段纪要
3. 再把整段纪要作为参数写给 Salesforce

这样的问题是：

- 大文本会被重复放进上下文
- 成本和延迟进一步上升
- 大文件甚至可能超过上下文窗口
- 模型在复制长文本时还可能出错

---

## 三、核心解决方案：Code execution with MCP

文章提出的核心思路是：

**不要让模型直接逐轮调用工具，而是让模型在代码执行环境里写代码，通过代码去调用 MCP 工具。**

也就是把 MCP Server 提供的工具，转换成一种“代码 API”的使用方式。

### 基本做法

可以把每个 MCP Server 的工具组织成文件树，例如：

```text
servers
├── google-drive
│   ├── getDocument.ts
│   └── index.ts
├── salesforce
│   ├── updateRecord.ts
│   └── index.ts
```

然后 Agent：

1. 先浏览文件树，找到需要的 server
2. 按需读取某个工具文件
3. 理解函数接口
4. 写代码组合调用多个工具

例如：

```ts
import * as gdrive from './servers/google-drive';
import * as salesforce from './servers/salesforce';

const transcript = (await gdrive.getDocument({ documentId: 'abc123' })).content;
await salesforce.updateRecord({
  objectType: 'SalesMeeting',
  recordId: '00Q5f000001abcXYZ',
  data: { Notes: transcript }
});
```

这样，整段转录文本只在代码执行环境里流动，不需要反复塞进模型上下文。

---

## 四、这种方案带来的核心收益

## 1. 按需加载工具（Progressive disclosure）

模型不需要一开始读完所有工具定义。

只需要：

- 找到相关 MCP Server
- 打开当前任务需要的那几个工具文件
- 读取必要的接口定义

这样可以显著减少上下文占用。

文章举例提到，这种方式可将 token 使用从：

- **150,000 tokens**
- 降到 **2,000 tokens**

节省约 **98.7%**。

## 2. 在代码里先处理数据，再返回结果

如果工具返回的是大数据集，例如：

- 10,000 行表格
- 大文档
- 多个系统返回的复杂 JSON

就可以先在执行环境里完成：

- 过滤
- 聚合
- 提取字段
- 拼接
- join

只把少量结果返回给模型。

这样比让模型直接处理全部原始数据高效得多。

## 3. 控制流更强、更省上下文

使用代码后，可以直接写：

- 循环
- 条件判断
- sleep / retry
- 错误处理

例如轮询 Slack 消息直到出现“deployment complete”。

这比让模型在多轮工具调用中不断判断“要不要继续”等逻辑，更自然也更便宜。

## 4. 更好地保护隐私

中间结果默认停留在执行环境里，而不是直接进入模型上下文。

只有显式 `log` 或 `return` 的内容，模型才会看到。

文章还举了一个更进一步的做法：

- MCP Client 自动对敏感数据做 tokenization
- 例如邮箱、电话、姓名用占位符替代
- 在真正写入目标系统时再反向还原

这样真实 PII 可以在系统间流转，但不暴露给模型。

## 5. 可以持久化状态和沉淀“技能”

有了文件系统和代码执行环境后，Agent 可以：

- 把中间结果写入文件
- 断点续做
- 保存成功的函数
- 形成可复用的 skill

例如把“将 Google Sheet 保存成 CSV”的逻辑保存成一个复用函数，后续任务直接调用。

这与更高层的 Agent Skills 思路是相通的。

---

## 五、适合哪些场景

这种模式特别适合：

- 工具很多
- 工具定义复杂
- 中间数据很大
- 需要多步编排
- 需要循环 / 条件 / 重试
- 涉及敏感数据流转
- 需要状态持久化和复用能力

典型例子包括：

- Google Drive → Salesforce 数据搬运
- 大表格筛选与处理
- Slack / 部署状态轮询
- 多系统数据联动
- 长流程自动化任务

---

## 六、代价与局限

文章也明确指出：

**代码执行不是“纯赚不亏”的方案。**

它会带来新的工程复杂度，包括：

- 需要安全沙箱（sandbox）
- 需要资源限制
- 需要监控与审计
- 需要防止 Agent 生成危险代码
- 运维复杂度高于直接 tool call

因此，这种方案的价值要和实现成本一起评估。

### 简单理解

- **工具少、流程简单**：直接 tool call 更简单
- **工具多、流程复杂、数据大**：代码执行 + MCP 更有优势

---

## 七、文章的核心思想

文章其实是在说：

虽然 Agent 场景里出现了很多看似新的问题，例如：

- 上下文管理
- 工具编排
- 状态持久化
- 数据流转

但这些问题在软件工程中早就有成熟思路。

而 **Code execution with MCP**，本质上就是把这些成熟的软件工程方法重新带回到 Agent 系统里。

也就是说：

- 不要让模型承担所有控制与搬运工作
- 让代码执行环境承担可编程、可复用、可组合的部分
- 让模型专注于理解任务和生成高层策略

---

## 八、我的理解

可以把两种方式对比成：

### 方式 A：传统 direct tool calling

模型负责：

- 看工具定义
- 决定调哪个工具
- 接收结果
- 再决定下一个工具
- 复制和组织中间数据

缺点是：

- 模型负担重
- 上下文很快膨胀
- 成本高
- 容易慢

### 方式 B：code execution with MCP

模型负责：

- 找到相关工具
- 写出组合逻辑代码

代码执行环境负责：

- 调工具
- 处理数据
- 控制流程
- 保存状态
- 管理隐私流转

优点是：

- 更省 token
- 更适合复杂任务
- 更像真正的软件系统

---

## 九、一句话总结

**MCP 解决的是“怎么把 Agent 接到外部工具和系统上”；而 code execution with MCP 解决的是“当工具很多、数据很大、流程很复杂时，如何让 Agent 更高效地使用这些工具”。**

---

## 十、提纲

### 关键词

- MCP
- Tool definitions
- Intermediate results
- Context window
- Code execution
- Progressive disclosure
- Privacy-preserving operations
- State persistence
- Skills

### 文章结论

1. MCP 工具多了以后，直接 tool calling 会变贵、变慢
2. 主要问题是工具定义和中间结果都占上下文
3. 更好的方法是让 Agent 在代码执行环境里调用 MCP 工具
4. 这样可以按需加载工具、在本地处理数据、增强控制流、保护隐私、持久化状态
5. 代价是系统更复杂，需要沙箱、安全和运维支持

