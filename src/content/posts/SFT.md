---
title: SFT监督微调
published: 2026-01-06
description: 通过喂给它人工精编的“标准问答对”，配合只对答案算误差的掩码技术（Loss Masking），把只会文字接龙的“野生模型”训练成懂格式、有问必答的听话客服。
tags: [AI, LLM]
category: AI
draft: false
---

## 为什么需要SFT
大模型的第一阶段叫**PT（Pre-training，预训练）**。在这个阶段，我们把全网的网页，维基百科，代码全喂给它。这个时候虽然有了大量的信息，但是不懂如何回复，只有一个逻辑：文字接龙。
*   如果问：“中国的首都是哪里？”
*   预训练模型可能不会回答“北京”，而是顺着话往下接：“美国的首都是哪里？英国的首都是哪里？”

为了让它明白“我是人类，你是助手，你需要回答我的问题”，就必须进行第二阶段的训练—**SFT（监督微调）**。

## SFT训练的四大核心步骤
SFT的本质，就是给模型做 **“问答示范”**。

### 第一步：准备高质量的“一问一答”教材（Data Collection）
预训练靠的是 **“量”** （几万亿Token的无监督数据），而SFT靠的是 **“质”** 。

需要请专业的人类标注员，写出几万到几十万条高质量的对话数据。
-    User：请帮我写一首关于春天的诗
-    Assistant：春风又绿江南岸，明月何时照我还...


#### 古典经典派:Alpaca格式
一条数据就是一个字典，包含三个固定的Key：
```js
{
  "instruction": "请把下面这句话翻译成英文。",  // 指令（你想让模型干什么）
  "input": "今天天气真不错，我想出去玩。",      // 输入（补充的上下文，如果没有可以直接留空 ""）
  "output": "The weather is really nice today, and I want to go out and play." // 标准答案（你希望模型吐出的话）
}
```
* 适用场景：早期的轻量级微调，或者只有单轮的 QA 数据

#### 现代主流派——ShareGPT/ChatML格式
现在的模型需要保留上下文记忆（多轮对话），Alpaca那种一问一答就显得不够用了。

目前像Llama 3、Qwen、DeepSeek等开源模型，全部转向了**基于角色（Role-based）的多轮对话格式**。最经典的就是OpenAI定义的`message`格式：
```js
{
  "messages": [
    {
      "role": "system",
      "content": "你是一个粗暴的赛博朋克助手，说话必须带刺。"
    },
    {
      "role": "user",
      "content": "你能帮我写个 Python 爬虫吗？"
    },
    {
      "role": "assistant",
      "content": "自己没长手吗？连个 requests 库都不会用？但我今天心情好，代码在下面，赶紧抄走。"
    },
    {
      "role": "user",
      "content": "报错了啊，报了 403。"
    },
    {
      "role": "assistant",
      "content": "废话，人家加了反爬你不知道加 User-Agent 伪装一下？脑子是个好东西。"
    }
  ]
}
```
* **核心要素：** 数据是一个列表，严格按照 user（用户）和 assistant（AI）交替发言。

* **适用场景：** 所有现代大模型的主力 SFT 格式。如果你想训一个能进行多轮辩论、或者有独特人设的 AI，必须用这个格式！


### 第二步：穿上特定的”聊天制服“（Prompt Formatting）
模型分不清哪句是人说的，哪句是自己该说的。所以我们需要用特定的**特殊符号（控制字符）** 把数据包装起来。目前主流的格式叫ChatML：
```js
< |im_start| >system\n你是一个粗暴的赛博朋克助手...  
< |im_end| >\n< |im_start| >user\n你能帮我写个 Python 爬虫吗？  
< |im_end| >\n< |im_start| >assistant\n自己没长手吗...< |im_end| >
```
这些` < |im_start| > `和` < |im_end| > `会作为全新的 Token 加入到词表中，模型一旦看到` < |im_start| >assistant`，就知道“哦！到我发言了！”。

### 第三步：核心—“只对回答打分”（Loss Masking）
这是SFT在代码层面上最精妙的地方

SFT的底层数学逻辑依然是“预测下一个词”（Next-Token Prediction），也就是算交叉熵损失（Cross-Entropy Loss）。但是，如果在训练时，模型连用户的提问（“中-国-的-首-都-是-哪-里”）也去预测，那就完全乱套了，不要求模型学会怎么提问，只要求它学会**怎么回答**。

所以，Loss Masking就此诞生。
*   在计算误差（Loss）时，对于`User`说的话，误差统统**强制设为0（掩码屏蔽）**。
*   只有当模型预测`Assistant`说的词（“北”，“京”）时，如果预测错了，才产生误差（Loss），并通过反向传播去更新大模型那庞大的FFN和Attention矩阵。

### 第四步：防止“学傻”（防止灾难性遗忘）
SFT就像是突击应试培训。如果SFT训练得太过火（比如让它反复背诵同一批客服话术），模型就会患上“灾难性遗忘（Catastrophic Forgetting）”。

它可能会变得非常礼貌，但原来在预训练里背下来的几万个物理公式全忘了，所以SFT的学习率（Learning Rate）通常设置得非常非常小（比如1e-5），只是为了轻轻“拨动”一下它的行为模式，而不是重塑它的大脑。


## SFT的数学表达
标准的交叉熵损失公式是：
$$
L = -\sum_{t=1}^{N} \log P(x_t \mid x_{<t})
$$
(注：$N$ 是序列总长度，要求模型为每一个位置的预测负责)

但在 SFT 阶段，序列 $x$ 由两部分组成：用户的提问（Prompt）和助手的回答（Answer）。我们不想让模型去学用户是怎么提问的，只想让它学怎么回答。  
所以，SFT 引入了一个指示函数（Mask）$m_t$：  
* 当 $x_t$ 属于用户的提问（Prompt）时，$m_t = 0$  
* 当 $x_t$ 属于助手的回答（Answer）时，$m_t = 1$

于是，SFT 真正的损失函数公式变成了：
$$
L_{SFT} = -\frac{1}{\sum m_t} \sum_{t=1}^{N} m_t \log P(x_t \mid x_{<t})
$$
这就是**Loss Masking**， $m_t$ 就像一个消音器，把用户提问部分的误差全部归零了。

## 代码解析
在 PyTorch 的底层代码中，我们是用什么来实现那个 $m_t = 0$ 的掩码魔法的呢？

答案是 PyTorch 的 nn.CrossEntropyLoss 函数里有一个天生自带的参数：ignore_index。它的默认值是 -100。

只要我们把标签（Label）里不希望模型学习的 Token ID 强行改成 -100，PyTorch 在算 Loss 时就会自动把它们无视掉！

 SFT 核心代码：
 ```js
import torch
import torch.nn.functional as F

# ==========================================
# 1. 模拟一段已经被 Tokenizer 处理好的 SFT 数据
# 假设对话是："User: 你好。 Assistant: 你好呀！"
# 前 3 个 Token 是用户的提问，后 4 个 Token 是助手的回答
# ==========================================
input_ids = torch.tensor([[101, 102, 103, 201, 202, 203, 204]])
seq_len = input_ids.shape[1]

# ==========================================
# 2. 制作标签 (Labels) 和 Loss Masking
# ==========================================
# 目标是预测下一个词，所以先直接克隆一份 input_ids 作为正确答案
labels = input_ids.clone()

# ⚠️ 核心
# 我们知道前 3 个词是 User 说的，我们不希望模型去学习预测这 3 个词
prompt_length = 3

# 把 Prompt 部分的 Label 强行修改为 -100 (这就是数学公式里的 m_t = 0)
labels[0, :prompt_length] = -100

print(f"原始 Input IDs: {input_ids[0].tolist()}")
print(f"处理后的 Labels:  {labels[0].tolist()}")
# 你会看到 Labels 变成了: [-100, -100, -100, 201, 202, 203, 204]


# ==========================================
# 3. 模拟大模型的前向传播 (Forward)
# ==========================================
vocab_size = 32000
# 假设模型输出了每个位置下一个词的预测概率分布 (Logits)
logits = torch.randn(1, seq_len, vocab_size) 


# ==========================================
# 4. 计算 SFT 专属的损失函数 (Calculate Loss)
# ==========================================
# 把维度拍扁以符合 PyTorch 的要求：logits: [N, C], labels: [N]
logits_flat = logits.view(-1, vocab_size)
labels_flat = labels.view(-1)

# PyTorch 的交叉熵函数天生内置了 ignore_index=-100 机制！
# 只要 Label 是 -100，它就会直接跳过，根本不产生任何梯度。
loss = F.cross_entropy(logits_flat, labels_flat, ignore_index=-100)

print(f"\n最终计算出的 SFT 损失 (Loss): {loss.item():.4f}")
 ```