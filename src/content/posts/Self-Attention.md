---
title: 注意力机制
published: 2026-01-03
description: 注意力机制让模型在处理每个 token 时能够动态地关注输入序列中不同位置的信息，解决了传统 RNN 难以捕捉长距离依赖的问题，是大模型理解复杂语义的核心基础。
tags: [AI, LLM]
category: AI
draft: false
---

## 什么是自注意力（Self-Attention）？
在处理序列数据时，自注意力机制允许模型在处理当前词时，去“看“句子中的其他所有词，并根据它们的相关性分配不同的权重。

解决的问题：
1. **突破长距离依赖：** 无论两个词隔得有多远，计算它们的关系只需一步。
2. **解决一词多义：** 比如“苹果公司”和“吃苹果”，通过关注周围的词（上下文），模型能动态调整“苹果”这个词最终的向量表示。


## Q、K、V
假设输入是一句话$X$（这句话里所有词的词向量拼在一起的矩阵）。模型里有三个可通过不断优化的权重矩阵：$W^Q$、$W^K$、$W^V$。
*   **Query ($Q$ - 查询向量): $Q = X W^Q$**。代表当前词去“询问其他词的意图”。
*   **Key ($K$ - 键向量): $K = X W^K$**。代表句子中每个词的“身份标签”，用来被别人查询。
*   **Value ($V$ - 值向量): $V = X W^V$**。代表句子中每个词本身携带的“真实信息内容”。

Q、K、V 本质上都是同一个输入矩阵 $X$ 经过不同线性变换（乘以不同权重矩阵）得到的。这就是它为什么叫自注意力（Self-Attention）——因为它是自己和自己算相关性。

## 核心计算流程：Scaled Dot-Product Attention（缩放点积注意力）
这是整个机制的灵魂，分为四个标准的数学步骤。

### 第一步：计算相似度得分（Dot Product）
**公式：**  $\text{Score} = Q K^T$。
* **做法：** 拿查询矩阵 $Q$ 乘以键矩阵 $K$ 的转置（$K^T$）。
* **意义：** 数学上，两个向量的点积越大，说明它们越相似/越相关。这一步算出了句子中所有词两两之间的原始相关性得分。

### 第二步：缩放（Scale）
**公式：** $\frac{Q K^T}{\sqrt{d_k}}$
* **做法：** 将得分除以 $\sqrt{d_k}$（$d_k$ 是 $K$ 向量的维度大小，比如 64，那根号下就是 8）
* **意义：** 如果向量维度很大，点积算出来的数值会非常大。数值太大丢进 Softmax 后，会导致梯度极度平缓（接近 0），也就是所谓的梯度消失，模型就学不动了。这一步是为了稳定训练。

### 第三步：归一化 (Softmax)
**公式：** $\text{Attention Weights} = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)$
* **做法：** 把缩放后的得分通过 Softmax 函数。
* **意义：** 将杂乱的分数转换成 $0 \sim 1$ 之间的概率分布，且每一行的权重之和等于 1。这就是 **“注意力权重表”**。

### 第四步：信息融合 (Multiply with V)
**公式：** $\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V$
* **做法：** 拿上一步算出来的权重矩阵，去乘以值矩阵 $V$。
* **意义：** 根据权重的大小，把各个词的真实信息 $V$ 加权求和，得到最终的输出矩阵。权重大的词，它的信息被保留得多；权重小的词，信息就被淡化了。

## 传统：MHA（Multi-Head Attention，多头注意力）
Transformer刚诞生时的经典设计。
*   **职场比喻：** 公司有8个老板（8个Query头，简称Q），每个人关注业务的不同维度。为了服务这8个老板，公司配了8个专属秘书（8个Key头和8个Value头，简称K和V）。老板1去提问，秘书1去查资料回报；老板2提问，秘书2去查....
*   **致命痛点：** 大模型在生成文本时是“一个字一个字蹦”的。为了不重复计算，模型会把以前算过的秘书资料（K和V向量）全部存在显存里，这叫KV Cache。当你上下文越来越长，或者并发用户越来越多时，这8个秘书积累的资料库会瞬间把显卡的显存撑爆，GPU算力还没用完，显存先OOM了（Out of Mermory）了。

### 为什么需要多头？（单头注意力的局限）
假设我们有一句及其经典的测试句：  
*"The animal didn't cross the street because it was too tired."  
（这只动物没有穿过街道，因为它太累了。）*

如果模型里只有**一个头（Single-Head Attention）**，它就像一个精力有限的读者，在阅读单词***it***的时候，它的注意力（Attention分数）只能集中在一件事上。比如，它可能只把注意力分配给了 *“animal”* （搞清了代词指代）。到那时，它就无暇顾及 *“tried”*（状态描述）了。

为了让模型能同时理解一句话中复杂的语法、情感、指代和逻辑，提出了MHA。

**类比：专家顾问团**
我们不再只雇佣一个读者，而是雇佣一个由 8 个不同领域专家组成的 **“剧本围读顾问团”**（8 个 Head）：

* 1号专家（语法头）： 专门负责找主谓宾，他看到 "it" 时，注意力会死死盯住 "cross"。
* 2号专家（指代头）： 专门负责找代词的主人，他看到 "it" 时，注意力会 100% 集中在 "animal" 上。
* 3号专家（情感头）： 专门负责感受情绪，他看到 "it" 时，会和 "tired" 产生强烈的注意力连接。

这 8 个专家（Head）**各自独立工作**，分别提取不同维度的信息，最后把各自的报告拼在一起（Concat），交给大老板（输出投影层）。这样，模型对这句话的理解就变得极其立体和丰满！ 

### MHA的核心数学公式
假设我们输入序列的词向量维度是 $d_{model} = 512$，我们准备使用 $h = 8$ 个头。
为了不增加整体的计算量，我们会把 512 维均分给 8 个头，所以每个头分配到的维度是 $d_k = 512 \div 8 = 64$ 维。

对于任意一个头 $i$（$i \in [1, h]$），它都有自己独立专属的三个小矩阵 $W_i^Q, W_i^K, W_i^V$：
1. **生成专属的Q、K、V：**
   $$Q_i = X W_i^Q, \quad K_i = X W_i^K, \quad V_i = X W_i^V$$
2. **各自独立计算注意力（缩放点积）：**
    $$\text{head}_i = \text{Attention}(Q_i, K_i, V_i) = \text{softmax}\left(\frac{Q_i K_i^T}{\sqrt{d_k}}\right) V_i$$
3. **拼接并融合：**
   所有专家把各自的报告$\text{head}_i$（每个都是 64 维）在最后一个维度上无缝拼接起来，正好变回原始的512维：
   $$\text{MultiHead}(Q, K, V) = \text{Concat}(\text{head}_1, \dots, \text{head}_h) W^O$$
   最后乘以一个融合矩阵$W^O$，把不同专家的意见进行一次综合评估，输出最终结果。

### MHA的代码表达
```
import torch
import torch.nn as nn
import math
import torch.nn.functional as F

class MultiHeadAttention(nn.Module):
    def __init__(self, hidden_size, num_heads):
        """
        初始化 MHA 模块
        Args:
            hidden_size: 模型的总隐藏层维度 (如 4096)
            num_heads: 头的总数 H (如 32)
        """
        super().__init__()
        self.hidden_size = hidden_size
        self.num_heads = num_heads
        
        # 必须保证总维度能被头数整除
        assert hidden_size % num_heads == 0
        self.head_dim = hidden_size // num_heads
        
        # 📚 注意看这里！MHA 的 Q, K, V 权重矩阵是等大的！
        # 相比于 GQA，这里的 Wk 和 Wv 巨大无比，这也是导致 KV Cache 爆炸的元凶
        self.Wq = nn.Linear(hidden_size, hidden_size, bias=False)
        self.Wk = nn.Linear(hidden_size, hidden_size, bias=False)
        self.Wv = nn.Linear(hidden_size, hidden_size, bias=False)
        
        # 输出融合矩阵
        self.Wo = nn.Linear(hidden_size, hidden_size, bias=False)

    def forward(self, x):
        """
        x 形状: [batch_size, seq_len, hidden_size]
        """
        batch_size, seq_len, _ = x.shape

        # 1. 一次性算出全量的 Q, K, V
        # 形状都是: [B, L, hidden_size]
        q = self.Wq(x)
        k = self.Wk(x)
        v = self.Wv(x)

        # 2. 空间魔术：拆分出多个头
        # 先 view 成 [B, L, num_heads, head_dim]
        # 再 transpose 变成 [B, num_heads, L, head_dim]
        q = q.view(batch_size, seq_len, self.num_heads, self.head_dim).transpose(1, 2)
        k = k.view(batch_size, seq_len, self.num_heads, self.head_dim).transpose(1, 2)
        v = v.view(batch_size, seq_len, self.num_heads, self.head_dim).transpose(1, 2)

        # 📚 对比 GQA：这里没有 repeat_kv() 函数！
        # 因为在 MHA 里，K 和 V 的头数本来就和 Q 一样多，每个 Q 头都有自己专属的 K 和 V。

        # 3. 计算缩放点积注意力 (Scaled Dot-Product Attention)
        # q: [B, H, L, head_dim] 与 k.T: [B, H, head_dim, L] 相乘 -> [B, H, L, L]
        scores = torch.matmul(q, k.transpose(-2, -1)) / math.sqrt(self.head_dim)
        
        # 计算注意力权重 (这里省略了掩码 mask 机制，专注展示核心结构)
        attn_weights = F.softmax(scores, dim=-1)
        
        # 权重乘以 V -> [B, H, L, head_dim]
        output = torch.matmul(attn_weights, v)

        # 4. 把多个专家的结果合并 (Concat)
        # transpose 把头数换回去: [B, L, H, head_dim]
        # contiguous 在内存中重新连续化，准备变形
        output = output.transpose(1, 2).contiguous()
        
        # view 直接把 H 和 head_dim 拍扁，变回 hidden_size
        # 形状回到: [B, L, hidden_size]
        output = output.view(batch_size, seq_len, self.hidden_size)

        # 5. 最后交由老板综合评估
        return self.Wo(output)

# ========== 测试一下 MHA ==========
if __name__ == "__main__":
    batch_size = 2
    seq_len = 10
    hidden_size = 4096
    num_heads = 32
    
    # 初始化 MHA
    mha = MultiHeadAttention(hidden_size, num_heads)
    dummy_x = torch.randn(batch_size, seq_len, hidden_size)
    
    out = mha(dummy_x)
    
    print("=== MHA 运行成功 ===")
    print(f"输入形状: {dummy_x.shape}")
    print(f"Q 权重形状: {mha.Wq.weight.shape}")
    print(f"K 权重形状: {mha.Wk.weight.shape} (注意：和 Q 完全一样大！)")
    print(f"输出形状: {out.shape}")
```


## 改进：MQA（Multi-Query Attention，多查询注意力）
为了解决显存爆炸，研究人员提出了极其激进的MQA。
*   **职场比喻：** 经济危机，公司大裁员，8个老板（8个Q头）全部保留，但全公司只留一个秘书（1个K头和1个V头）。8个老板不管问什么不同维度的问题，都由这1个可怜的秘书提供同一份基础资料。
*   **优势：** 秘书只剩1个了，KV Cache占用的显存直接缩小到了原来的1/8，推理速度加快了。
*   **代价：** 模型变笨了，因为1个秘书根本伺候不好8个性格迥异的老板，资料的精细度大幅下降，模型的理解能力和生成质量（特别是复杂逻辑）出现了明显的下滑。

## 完美的端水大师：GQA（Grouped Query Attention）
由Google研究人员提出，并被Llama 2彻底发扬光大。取一个折中办法。
*   **职场比喻：** 公司进行部门重组。把8个老板分成**2个业务组（Group）**，每组4个老板。然后给每个组配1个秘书。
    所以现在是：8个老板（Q），配了2个秘书（2个K和2个V）。
    第1～4号老板，共享1号秘书的资料；第5～8号老板，共享2号秘书的资料。
*   **优点：**
    * **省显存：** 虽然不如MQA那么极致，但相比于最原始的MHA，KV Cache的显存占用直接缩小了4倍，大大解决了访存墙问题。
    * **保智商：** 实验证明，GQA的模型质量和传统的MHA**几乎一摸一样**，完全没有MQA那种智商掉线的情况。

### GQA的数学表达
先定义几个关键变量：
* $H$：Query 头的总数量（例如 32 个老板）。
* $G$：KV 头的总数量，也就是分组的数量（例如 8 个秘书）。
* $g$：每个组里面包含的 Query 头数（$g = \frac{H}{G}$，例如 $32 \div 8 = 4$ 个老板共享一个秘书）。
* $D$：每个头的特征维度（Head Dimension）。
  
对于第 $i$ 个分组（$i \in [1, G]$），它里面有 $g$ 个 Query 头：$Q_{i,1}, Q_{i,2}, ..., Q_{i,g}$。这 $g$ 个 Query 头，共享唯一的一个 Key 头 $K_i$ 和一个 Value 头 $V_i$。

在计算注意力时，对于这个组里的第 $j$ 个 Query 头，它的计算公式为最经典的缩放点积注意力（Scaled Dot-Product Attention）：
$$\text{head}_{i,j} = \text{Attention}(Q_{i,j}, K_i, V_i) = \text{softmax}\left(\frac{Q_{i,j} K_i^T}{\sqrt{D}}\right) V_i$$
把所有组、所有头的结果算出来之后，把它们全部拼接（Concat）在一起，最后乘以一个输出权重矩阵 $W_O$，就完成了完整的一层前向传播：
$$\text{Output} = \text{Concat}(\text{head}_{1,1}, ..., \text{head}_{G,g}) W_O$$

### GQA的代码实现
为了追求极致的GPU并行效率，不会写8个`for`循环取分别算8个头。相反，会用一个巨大的线性层一次性算完，然后再用`view`和`transpose`函数，在张量的形状上玩“空间魔术”，把它们切分成多头。

```
import torch
import torch.nn as nn
import math
import torch.nn.functional as F

def repeat_kv(hidden_states: torch.Tensor, n_rep: int) -> torch.Tensor:
    """
    这是 GQA 的灵魂函数：把 K 和 V 的头复制 n_rep 次，以匹配 Q 的头数。
    Args:
        hidden_states: 形状为 [batch, num_kv_heads, seq_len, head_dim]
        n_rep: 每个 KV 头需要复制的次数 (即 H / G)
    """
    batch, num_kv_heads, slen, head_dim = hidden_states.shape
    if n_rep == 1:
        return hidden_states # 如果是 MHA (头数一样)，就不需要复制

    # 1. 增加一个维度: [batch, num_kv_heads, 1, seq_len, head_dim]
    # 2. 使用 expand 复制: [batch, num_kv_heads, n_rep, seq_len, head_dim]
    # 注意：expand 在 PyTorch 底层是不占用额外显存的，只是改变了视图(View)
    hidden_states = hidden_states[:, :, None, :, :].expand(
        batch, num_kv_heads, n_rep, slen, head_dim
    )
    
    # 3. 压平前两个维度，变成和 Q 一样的形状: [batch, num_kv_heads * n_rep, seq_len, head_dim]
    return hidden_states.reshape(batch, num_kv_heads * n_rep, slen, head_dim)


class GroupedQueryAttention(nn.Module):
    def __init__(self, hidden_size, num_q_heads, num_kv_heads):
        """
        初始化 GQA 模块
        Args:
            hidden_size: 模型的总隐藏层维度 (如 4096)
            num_q_heads: Query 头的数量 H (如 32)
            num_kv_heads: KV 头的数量 G (如 8)
        """
        super().__init__()
        self.num_q_heads = num_q_heads
        self.num_kv_heads = num_kv_heads
        
        # 确保 Q 的头数能被 KV 的头数整除
        assert self.num_q_heads % self.num_kv_heads == 0
        self.n_rep = self.num_q_heads // self.num_kv_heads
        
        # 每个头的维度
        self.head_dim = hidden_size // self.num_q_heads
        
        # 📚 这里的线性层输出维度不一样了！
        # Wq 的输出是 32 个头的大小
        self.Wq = nn.Linear(hidden_size, num_q_heads * self.head_dim, bias=False)
        # Wk 和 Wv 的输出只有 8 个头的大小 (极大节省了显存和计算量)
        self.Wk = nn.Linear(hidden_size, num_kv_heads * self.head_dim, bias=False)
        self.Wv = nn.Linear(hidden_size, num_kv_heads * self.head_dim, bias=False)
        
        self.Wo = nn.Linear(num_q_heads * self.head_dim, hidden_size, bias=False)

    def forward(self, x):
        """
        x 形状: [batch_size, seq_len, hidden_size]
        """
        batch_size, seq_len, _ = x.shape

        # 1. 线性投影得到 Q, K, V
        q = self.Wq(x) # [B, L, num_q_heads * head_dim]
        k = self.Wk(x) # [B, L, num_kv_heads * head_dim]
        v = self.Wv(x) # [B, L, num_kv_heads * head_dim]

        # 2. 变形为多头注意力所需要的形状
        q = q.view(batch_size, seq_len, self.num_q_heads, self.head_dim).transpose(1, 2)
        # Q 形状: [B, num_q_heads, L, head_dim]
        
        k = k.view(batch_size, seq_len, self.num_kv_heads, self.head_dim).transpose(1, 2)
        v = v.view(batch_size, seq_len, self.num_kv_heads, self.head_dim).transpose(1, 2)
        # K, V 形状: [B, num_kv_heads, L, head_dim]

        # 3. 核心步骤：给 K 和 V 强行“影分身”
        # 把 [B, 8, L, head_dim] 变成了 [B, 32, L, head_dim]
        k = repeat_kv(k, self.n_rep)
        v = repeat_kv(v, self.n_rep)

        # 4. 标准的 Scaled Dot-Product Attention
        # (此时 Q, K, V 的头数完全一样了，都是 32)
        scores = torch.matmul(q, k.transpose(-2, -1)) / math.sqrt(self.head_dim)
        attn_weights = F.softmax(scores, dim=-1)
        output = torch.matmul(attn_weights, v) # [B, num_q_heads, L, head_dim]

        # 5. 把多个头的结果重新拼装回去
        output = output.transpose(1, 2).contiguous() # [B, L, num_q_heads, head_dim]
        output = output.view(batch_size, seq_len, -1) # [B, L, hidden_size]

        # 6. 最终的线性映射
        return self.Wo(output)

# ========== 测试一下 GQA ==========
if __name__ == "__main__":
    batch_size = 2
    seq_len = 10
    hidden_size = 4096
    
    # 模拟老板和秘书的数量
    q_heads = 32
    kv_heads = 8
    
    # 初始化 GQA 模块
    gqa = GroupedQueryAttention(hidden_size, q_heads, kv_heads)
    
    # 构造假数据
    dummy_x = torch.randn(batch_size, seq_len, hidden_size)
    
    # 前向传播
    out = gqa(dummy_x)
    
    print("=== GQA 测试运行成功 ===")
    print(f"输入形状: {dummy_x.shape}")
    print(f"Q 的权重矩阵形状: {gqa.Wq.weight.shape} (全尺寸)")
    print(f"K 的权重矩阵形状: {gqa.Wk.weight.shape} (极大地缩小了)")
    print(f"输出形状: {out.shape}")
```