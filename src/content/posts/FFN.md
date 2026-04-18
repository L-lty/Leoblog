---
title: FFN前馈神经网络
published: 2026-01-05
description: 在大模型中，FFN（前馈神经网络）充当着存储海量事实性知识的“独立大脑”，它接收注意力机制梳理好的上下文信息，通过升维、非线性门控过滤再降维的机制，对每个Token进行互不干扰的深度特征提炼。
tags: [AI, LLM]
category: AI
draft: false
---

# 前馈神经网络（FFN）

## 一、核心定位：“大模型的独立大脑与知识库”
在Transformer架构中，如果说Self-Attention层是“开会交流”，那么FFN（Feed-Forward Network）层就是“闭门独立思考”。
*   **分工明确：** Attention负责在序列中寻找Token之间的关系（如语法结构，代词指代）；而FFN则负责对每一个梳理好关系的Token进行深度的特征提炼。
*   **知识的容器：** 业界普遍认为，大模型能背诵古诗、写代码、懂得物理定理，这些海量的**事实性世界知识（World Knowledge）**，绝大部分都储存在FFN那庞大的权重矩阵（Weights）中。它占据了整个模型大约2/3的参数量。
  
## 二、核心机制：升维和降维
FFN的标准称呼是**Position-wise FFN（逐位置前馈网络）**，这意味着它在处理数据时，对序列中的每一个词（Token）是**完全独立，互不干扰**的。
它的计算过程本质上是一次“空间映射”，分为三步：
1. **升维（Upscale）：** 用一个线性矩阵把输入特征的维度拉宽（传统架构通常拉宽到4倍）。在高维空间中，特征之间纠缠的复杂关系更容易被”解开“（即变得线性可分）。
2. **非线性激活（Activation）：** 引入非线性函数（如GELU、SiLU），赋予模型处理复杂逻辑的能力。如果没有这一步，再多的线性层叠加也等价于一层。
3. **降维（Downscale）：** 处理完毕后，用另一个线性矩阵将高位特征重新压缩回模型原本的维度（Hidden Size），方便传给下一层。

## 三、架构的演进与数学表达

### 第一阶段：古典派FFN（如BERT,GPT-2）
结构非常直接：一个输入矩阵，一个激活函数，一个输出矩阵。

**数学公式：**
$$
FFN(x) = \text{GELU}(x W_1 + b_1) W_2 + b_2
$$
(注：$W_1$ 负责升维，$W_2$ 负责降维)

### 第二阶段：现代派SwiGLU变体（如Llama，Qwen，DeepSeek）
现代大模型全面抛弃了古典FFN，转向带有 **门控机制（Gating）** 的SwiGLU结构。它额外增加了一条分支作为”智能保安“。

**数学公式：**
$$
SwiGLU(x) = (\text{SiLU}(x W_{\text{gate}}) \odot (x W_{\text{up}})) W_{\text{down}}
$$
* $W_{\text{up}}$：负责把原始信息升维。
* $W_{\text{gate}}$：负责计算出一个“门控信号”（权值在 0~1 之间波动）。
* $\odot$ (逐元素乘法)：核心神技！ 门控信号会和升维后的信息相乘。这就相当于模型在动态判断：“刚才提取出来的这些高维知识，哪些对当前语境有用（放行），哪些是没用的噪音（过滤掉）。”

## 四、核心代码
```js
import torch
import torch.nn as nn
import torch.nn.functional as F

# ==========================================
# 1. 经典老大哥：老式 Transformer 的 FFN
# ==========================================
class ClassicFFN(nn.Module):
    def __init__(self, hidden_size, intermediate_size):
        """
        hidden_size: 模型原本的维度 (如 1024)
        intermediate_size: 升维后的肚子维度 (通常是 4 * hidden_size = 4096)
        """
        super().__init__()
        self.linear1 = nn.Linear(hidden_size, intermediate_size)
        self.act = nn.GELU()  # 经典的非线性激活函数
        self.linear2 = nn.Linear(intermediate_size, hidden_size)

    def forward(self, x):
        # 简单粗暴的三步走：线性升维 -> 激活 -> 线性降维
        return self.linear2(self.act(self.linear1(x)))


# ==========================================
# 2. 现代当红炸子鸡：大模型标配的 SwiGLU FFN
# ==========================================
class SwiGLUFFN(nn.Module):
    def __init__(self, hidden_size, intermediate_size):
        """
        注意：为了保持总参数量和老式 FFN 一致，这里的 intermediate_size 通常取 8/3 倍左右。
        """
        super().__init__()
        # 变体核心：大模型通常去掉了 bias，且增加到了三个线性层
        
        # 1. 门控分支 (决定让哪些信息通过)
        self.gate_proj = nn.Linear(hidden_size, intermediate_size, bias=False)
        # 2. 内容分支 (真正的升维信息)
        self.up_proj = nn.Linear(hidden_size, intermediate_size, bias=False)
        # 3. 降维输出分支
        self.down_proj = nn.Linear(intermediate_size, hidden_size, bias=False)
        
        self.act = nn.SiLU()  # 换成了 SiLU 激活函数

    def forward(self, x):
        # 第一步：计算门控信号，并经过激活函数 (聪明的保安)
        gate = self.act(self.gate_proj(x)) 
        
        # 第二步：计算实际内容 (不经过激活函数)
        up = self.up_proj(x)
        
        # 第三步：门控信号 与 实际内容 做逐元素乘法 (核心 Gating 门控机制！)
        intermediate = gate * up
        
        # 第四步：线性降维输出
        return self.down_proj(intermediate)
```

## 五、考点/总结
### FFN与Attention的最大区别是什么？
   Attention跨Token处理数据（维度是L×L），解决上下文依赖；FFN独立处理单个Token（维度是D × 4D），解决特征提炼与知识存储。
### 为什么现在的模型不用ReLU而是用SwiGLU？
ReLU太过简单粗暴（小于0直接切断）。SwiGLU引入了门控乘法机制，不仅能平滑地传递梯度，还能动态过滤无用信息，在同等计算量下，模型的表达能力和准确率都有显著提升。