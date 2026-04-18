---
title: Norm归一化
published: 2026-01-04
description: 防止数据在几十层 Transformer 流水线里经过成千上万次乘法后发生“数值爆炸”（变成无限大）。它强行把每一层的数据按比例缩放回一个健康、稳定的区间，保证模型训练时不崩溃。
tags: [AI, LLM]
category: AI
draft: false
---

# RMSNorm LayerNorm BatchNorm

# 为什么神经网络需要“归一化”
在深度学习中，数据（张量）在经过一层层的矩阵乘法后，数值很容易变得极大或极小，也就是梯度爆炸或梯度消失。为了让训练稳定，我们需要在每一层后面加一个“稳压器”，把数据重新拉回一个标准的范围内，这个操作就叫做归一化。

# LayerNorm

在早期的Transformer模型（比如BERT，GPT-2）中，大家用的都是LayerNorm。

假设一组数据传入了LayerNorm，就像老师拿到了一份全班成绩单（比如有考10分，有考100分的，极度不平衡）。LayerNorm会做两部极其严格的数学操作：

1. **平移（减去均值，Mean-centering）：** 老师会先算出全班的平均分（比如60分），然后让每个人的分数都减去60分。这样全班的平均分就变成了完美的0（有正有负）。
2. **缩放（除以方差）：** 老师再算出现在分数的波动范围（方差/标准差），把分数除以这个标准差。这样全班分数的波动幅度就被压缩到了1。

经过LayerNorm处理后，数据的均值一定是0，方差一定是1。模型训练变得非常稳定。

## LayerNorm的实现
**核心逻辑：** 跨特征（跨科目），在同一个样本（同一个学生）上做归一化
  
*   **老师的做法：** 班主任不管其他学生考得怎么样。他把你单独叫到办公室，把你考的C门课的成绩加起来，算出你个人的平均分和你个人成绩的方差（看看你是偏科还是均衡）。然后用你自己的平均分和方差，把你的各科成绩重新缩放。
*   **数学实现：** 对于第i个样本（学生），它的均值和方差计算如下：
    对于第 $i$ 个样本（学生）：
    $$
    \mu_i = \frac{1}{C} \sum_{j=1}^C x_{ij}
    $$
    $$
    \sigma_i^2 = \frac{1}{C} \sum_{j=1}^C (x_{ij} - \mu_i)^2
    $$
*   **应用场景：** **自然语言处理（Transformer / RNN）。** 每一个词（Token）的向量都是独立表达一个意思的。LayerNorm 只对当前这个词的向量维度做归一化，完全不依赖 Batch Size 有多大。
*   **致命弱点：** 计算量偏大。每次都要算均值，再算方差，非常吃内存带宽。
  
```js
import torch
import torch.nn as nn

class LayerNorm(nn.Module):
    def __init__(self, hidden_size, eps=1e-5):
        """
        初始化 LayerNorm 模块
        Args:
            hidden_size (int): 输入张量的特征维度大小 (即词向量的维度)
            eps (float): 防止分母为 0 的极小值 (PyTorch 默认通常是 1e-5)
        """
        super().__init__()
        # 老师的两件法宝：
        # 1. 缩放参数 gamma (weight)：初始化为全 1，用于调整方差
        self.weight = nn.Parameter(torch.ones(hidden_size))
        # 2. 平移参数 beta (bias)：初始化为全 0，用于调整均值
        # 注意：RMSNorm 里是没有这个 bias 的！
        self.bias = nn.Parameter(torch.zeros(hidden_size))
        self.eps = eps

    def forward(self, x):
        """
        前向传播函数
        Args:
            x (torch.Tensor): 形状通常为 [batch_size, seq_len, hidden_size]
        """
        # 第一步：算“个人平均分”（平移准备）
        # 沿着最后一个维度 (hidden_size) 求均值
        mean = x.mean(dim=-1, keepdim=True)
        
        # 第二步：算“个人成绩方差”（缩放准备）
        # 每个分数减去平均分后，求平方，再求均值
        # (x - mean) 就是我们在理论部分说的“减去均值，强制均值归零”的操作
        var = (x - mean).pow(2).mean(dim=-1, keepdim=True)
        
        # 第三步：标准化（归一化）
        # 减去均值后，除以标准差 (方差开根号)
        # 这里同样加上了 eps 防止除以零
        x_normed = (x - mean) / torch.sqrt(var + self.eps)
        
        # 第四步：老师的最终微调（乘以 gamma，加上 beta）
        return self.weight * x_normed + self.bias

# ========== 测试一下这段代码 ==========
if __name__ == "__main__":
    # 模拟一个 Batch Size 为 2，序列长度为 3，特征维度为 4 的输入
    dummy_input = torch.randn(2, 3, 4)
    
    # 初始化 LayerNorm，特征维度填 4
    layernorm = LayerNorm(hidden_size=4)
    
    # 将输入送入模型
    output = layernorm(dummy_input)
    
    print("=== LayerNorm 测试 ===")
    print("输入张量形状:", dummy_input.shape)
    print("输出张量形状:", output.shape)
    print("\n输出结果展示:\n", output)
    
    # 验证一下：LayerNorm 处理后，最后一个维度的均值应该极度接近 0，方差接近 1
    print("\n验证均值 (应该接近0):", output.mean(dim=-1)[0, 0].item())
    print("验证方差 (应该接近1):", output.var(dim=-1, unbiased=False)[0, 0].item())
```

# RMSNorm
随着大模型越来越大，研究人员开始精打细算，LayerNorm真的有必要这么严格？特别是在GPU上计算那个“减去均值”的操作，需要先把所有数据遍历一遍去减，非常浪费内存带宽和计算时间。

于是RMSNorm提出了：**不要均值，不要平移，只做缩放** RMSNorm认为，神经网络在归一化时，真正起作用的是“缩放（让数据别太大）”，而“平移”没什么用。
## RMSNorm的实现
直接计算所有数值的均方根（Root Mean Square， RMS），然后把所有数值除以这个均方根，**核心逻辑：** 跨样本（跨学生），在**同一个样本**上做归一化。公式：


*   **老师的做法：** 追求效率的班主任觉得调分的根本目的只是为了防止分数大得离谱。直接把你各科成绩求个平方，算个均值，再开个根号（即均方根 RMS）。然后把你的成绩除以这个 RMS 就完事了，不需要算平均分。
*   **数学实现：**
    直接计算第 $i$ 个样本的均方根：
    $$
    RMS_i = \sqrt{\frac{1}{C} \sum_{j=1}^C x_{ij}^2}
    $$
    然后直接缩放数据（$\gamma$ 为可学习的权重系数）：
    $$
    \hat{x}_{ij} = \frac{x_{ij}}{RMS_i} \cdot \gamma_j
    $$
*   **应用场景：** **现代大语言模型（Llama, Qwen, DeepSeek 等）。** 
*   **核心优势：** 不用算均值 $\mu$，也不用每个数去减均值，计算速度比 LayerNorm 快了 **10% ~ 50%**，而最终的模型收敛效果几乎一致。

```js
import torch
import torch.nn as nn

class RMSNorm(nn.Module):
    def __init__(self, hidden_size, eps=1e-6):
        """
        初始化 RMSNorm 模块
        Args:
            hidden_size (int): 输入张量的特征维度大小 (比如 512, 4096 等)
            eps (float): 一个极小的值，加在分母里防止除以 0 导致报错
        """
        super().__init__()
        # 可学习的缩放参数 gamma (在代码中通常命名为 weight)
        # 初始化为全 1，意味着刚开始不改变原始数据的大小
        self.weight = nn.Parameter(torch.ones(hidden_size))
        self.eps = eps

    def forward(self, x):
        """
        前向传播函数
        Args:
            x (torch.Tensor): 输入张量，形状通常为 [batch_size, seq_len, hidden_size]
        """
        # 1. 计算均方 (Mean Square)
        # 沿着最后一个维度 (hidden_size) 求平方的均值
        # keepdim=True 是为了保持维度不变，方便后面做广播除法
        variance = x.pow(2).mean(-1, keepdim=True)
        
        # 2. 计算均方根 (Root Mean Square) 并取倒数，然后乘以原始输入 x
        # 注意：这里使用 torch.rsqrt (平方根的倒数) 而不是先算 sqrt 再算除法
        # 这是底层性能优化的基操，rsqrt 在 GPU 上的计算速度更快
        x_normed = x * torch.rsqrt(variance + self.eps)
        
        # 3. 乘以可学习的缩放系数 gamma (即 weight)
        # 数据类型转换是为了确保混合精度训练时的稳定性（通常转回原本的数据类型）
        return self.weight * x_normed

# ========== 测试一下这段代码 ==========
if __name__ == "__main__":
    # 模拟一个 Batch Size 为 2，序列长度为 3，特征维度为 4 的输入
    dummy_input = torch.randn(2, 3, 4)
    
    # 初始化 RMSNorm，特征维度填 4
    rmsnorm = RMSNorm(hidden_size=4)
    
    # 将输入送入模型
    output = rmsnorm(dummy_input)
    
    print("输入张量形状:", dummy_input.shape)
    print("输出张量形状:", output.shape)
    print("\n输出结果展示:\n", output)

```


# BatchNorm

**核心逻辑：** 跨样本（跨学生），在**同一个特征（同一门课）** 上做归一化。

*   **老师的做法：** 数学教研组长算出全班数学的平均分（$\mu$）和方差（$\sigma^2$），然后把每个人的数学成绩都减去平均分，再除以标准差。
*   **数学实现：** 
    对于第 $j$ 个特征（科目）：
    $$
    \mu_j = \frac{1}{N} \sum_{i=1}^N x_{ij}
    $$
    $$
    \sigma_j^2 = \frac{1}{N} \sum_{i=1}^N (x_{ij} - \mu_j)^2
    $$
    然后对数据进行标准化，并乘以可学习的参数 $\gamma$，加上 $\beta$。
*   **应用场景：** **计算机视觉（CNN）。** 因为同一张图片的同一个通道在不同图片之间是有可比性的。
*   **致命弱点：** 极度依赖班级人数（Batch Size）。如果你这批只抽了 2 个学生算数学平均分，绝对有偏差。在 NLP 中，每个句子的长度不一样（有的学生缺考了某几门课），BatchNorm 很难处理。



# 核心对比速查表

| 特性 | BatchNorm | LayerNorm | RMSNorm |
| :--- | :--- | :--- | :--- |
| **计算维度** | 跨样本，按**特征（列）** 算 | 跨特征，按**样本（行）** 算 | 跨特征，按**样本（行）** 算 |
| **均值中心化** | 有 (减去 $\mu$) | 有 (减去 $\mu$) | **无** (不做平移) |
| **对 Batch Size 的要求**| 要求较大 | 无所谓 (Batch 为 1 也能算) | 无所谓 |
| **主要舞台** | CV (如 ResNet) | NLP (如 BERT, GPT-2) | 现代大模型 (如 Llama) |

# 为什么 RMSNorm 更适合大模型推理？

在模型落地部署（推理，Inference）阶段，RMSNorm 的优势会被彻底放大，它是现代大模型推理加速框架（如 vLLM、TensorRT-LLM）能跑出极限速度的关键基石之一。

核心原因可以归结为以下三点：

## 1. 突破大模型推理的死穴：“访存墙”（Memory Wall）
在大模型推理（尤其是逐字生成的 Decode 阶段）时，GPU 最大的瓶颈往往不是计算能力（FLOPs），而是**显存带宽**（将数据从显存搬运到计算核心的速度）。

*   **LayerNorm（高延迟）：** GPU 需要先读取一次当前 Token 的特征向量算出均值 $\mu$；然后再把这个向量**重新读取一遍**，让每个数减去均值并算出方差。这种对同一块内存**反复多次的读写（I/O）**会严重拖累推理速度。
*   **RMSNorm（高吞吐）：** 只需要计算平方和。这意味着 GPU 只需要把数据从显存里**顺滑地扫一遍（仅读取一次）**，就能算出 RMS 并直接完成缩放。它极大地降低了显存读写压力，省下宝贵的带宽去搬运庞大的模型权重。

## 2. 算子融合（Kernel Fusion）的“最佳伴侣”
工业界极度依赖**算子融合**技术来加速推理（例如将“残差相加 Add”和“归一化 Norm”捏成一个底层指令 `AddNorm`，省去中间结果写回显存的步骤）。

*   **LayerNorm（难融合）：** 减去均值的操作引入了强制的“同步点”。必须等所有维度的特征全部读取完毕并算出 $\mu$ 之后，才能处理具体的数字。
*   **RMSNorm（易融合）：** 数学形式只有乘法和加法，没有任何复杂的平移依赖。底层工程师可以极其轻松地写出极高效率的 `Add_RMSNorm` 融合算子。

## 3. 极大地降低单步延迟（Latency）
大模型生成回答是一个**自回归**过程，生成每一个 Token 都要把模型里的几十层 Norm 全部走一遍。

假设一个拥有 80 层的超大模型：
*   **LayerNorm：** 生成每一个词，这 80 层都要去算 80 次均值，再做 80 次减法运算。
*   **RMSNorm：** 直接略过了这 80 次毫无意义的减均值操作。

在微秒级别的延迟竞争中，当生成一段 1000 个字的回答时，RMSNorm 省下来的微小时间会累积成巨大的**首字响应延迟（TTFT）**和**每秒生成词数（Tokens/s）**的显著提升。

---

# 💡 总结

对于大模型推理来说：**数学公式越简单，底层硬件就越喜欢。**

RMSNorm 的出现，精准命中了现代 GPU 架构在推理时的最大痛点：**减少了内存读写，消除了同步依赖，完美支持底层算子融合。** 这就是开源大模型全面拥抱 RMSNorm 的根本原因。
