---
title: Transformer原始架构
published: 2026-01-01
description: Transformer的架构由Encoder与Decoder组成。
tags: [AI, LLM]
category: AI
draft: false
---

## Transformer架构图
![Transformer架构图](./images/transformer.avif)

## 第一阶段：数据入场和预处理（Input Precessing）

### 词嵌入（Input Embedding）
*   **动作：** 查字典。把每一个词映射成一个几千维的浮点数向量。
*   **意义：** 赋予词语最初的数学含义。意思相近的词，在这个高维空间里的距离也会更近。

### 位置编码（Position Encoding）
*   **痛点：** Transformer极其粗暴，它是一口气把所有词同时吞进去的（完全并行），这导致它**分不清词的先后顺序**。
*   **解决方法：** 为了解决这个问题，科学家给每个词的位置分配了一个独特的“正弦/余弦波形信号”，强行加到的Embedding向量上。这就相当于给每个词贴上了一个极其精确的时间戳排队号码牌。
*   **公式：**
$$
PE_{(pos, 2i)} = \sin(pos / 10000^{2i/d_{model}})
$$
$$
PE_{(pos, 2i+1)} = \cos(pos / 10000^{2i/d_{model}})
$$

## 第二阶段：Encoder Block
入场完毕后，数据进入了左边的**Encoder**，它的任务是：**拥有全局视角，彻底搞定这句话的情景和深层含义。** 一个标准的Encoder层包含了两个子模块。

### 1.多头自注意力机制（Multi-Head Self-Attention）
*   这是关键，它让句子里的每一个词，都去和句子里的其他词“相亲”，算出彼此的相关度。
*   **运作机制：** 同一个输入 $x$，通过三个不同的Linear层，变成了三个化身：
    * $Q$ (Query / 诉求)：我这个词现在想寻找什么信息？
    * $K$ (Key / 招牌)：我这个词身上带有什么特征？
    * $V$ (Value / 内涵)：我这个词真正的深层含义是什么？
*   **注意力公式：**
$$
Attention(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V
$$
(大白话：用我的 $Q$ 去匹配所有人的 $K$，算出亲密度打分，然后根据分数把所有人的 $V$ 混合在一起，变成我最新的状态。)

### 2.前馈神经网络（Feed Forward Network，FFN）
*   注意力机制只负责找关系，而FFN负责记忆和加工特征。
*   数据在这里被放大4倍维度（升维），经过非线性激活函数后，再被压缩回来。这是Transformer存储世界知识的主力区。

### 3.Add & Norm
*   在Attention和FFN的周围，各包裹了一层残差连接（Add, $y = F(x) + x$）和层归一化（Layer Norm）。保证网路即使堆叠100层，信息也不会丢失，数值也不会爆炸。
*   
(经过 $N$ 层 Encoder 的锤炼后，输出了一份包含极致上下文理解的 **“终极记忆矩阵”**，把它打包送到右脑！)

## 第三阶段：Decoder Block
Decoder收到任务，准备开始翻译或生成回答。它的任务是：根据Encoder的记忆，一个词一个词地往外蹦。它比Decoder多了一个关键的模块：

### 1.掩码多头自注意力（Masked Multi-Head Self-Attention）
*   跟Encoder不同，右脑在生成第3个词时，**不能偷看**第4个词。
*   因此，在这个模块里加入了掩码（Mask，通常是把未来的注意力得分强行设为$-\infty$），强迫模型只能和前面已经生成的词建立关系。

### 2.交叉注意力机制（Cross-Attention）———Encoder Decoder联手的桥梁
*   **精妙的设计：** Decoder不能光顾自己接龙，还需要知道原话是什么意思。
*   在这个层里，$Q$（我要翻译什么）来自于 Decoder 自己的前一层输出。
*   而 $K$（源语言的特征）和 $V$（源语言的内涵）全部来自于 **Encoder 送过来的“终极记忆矩阵”**。
*   这就是Decoder在向Encoder索要信息：“我马上要生成下一个词了，根据我现在的进度（$Q$），请把你那边最相关的外语信息（$K, V$）传给我。“

### 3.FFN与Add & Norm
*   同Encoder一样，进行最后的知识检索与特征稳定加工。

## 第四阶段：Output Generation
当数据从Decoder的最后一层出来时，它依然是一串高维的浮点数向量。我们需要把它变成人类认识的文字。

### 1.线性映射投影曾（Linear/Im_head）
*   这是最宽的一个全连接矩阵，它把4096维的向量，直接映射到大模型的整个词汇表维度上（比如词表有 130,000 个词，输出的就是 130,000 维）。

### 2.Softmax概率化：
*   将那130000个毫无规律的打分，转化为加起来刚好等于100%的**概率分布。**
*   比如：输出”猫“的概率是85%，”狗“是10%，”杯子“是0.001%。
*   最后，系统根据这个概率，吐出最合理的词。

然后，这个新吐出来的词，又会被当成新的输入，重新喂进Decoder的最底部，开始下一轮的循环，这就是自回归生成（Auto-regressive Generation）。


## Transformer视频详解

<iframe width="100%" height="468" src="//player.bilibili.com/player.html?bvid=BV13z421U7cs&p=1&autoplay=0" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true" &autoplay=0> </iframe>