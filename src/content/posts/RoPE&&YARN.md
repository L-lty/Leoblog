---
title: RoPE旋转位置编码
published: 2026-01-02
description: 当 Agent 连接越来越多 MCP 工具时，传统“直接工具调用”的方式会带来严重的上下文开销；更高效的做法是让 Agent 在代码执行环境中，通过写代码来调用 MCP 工具。
tags: [AI, LLM]
category: AI
draft: false
---


# RoPE（Rotary Position Embedding）旋转位置编码 

## 为什么需要位置编码
在Transformer模型中，核心机制是**Self-Attention（自注意力）**。但是Attention有一个天生的致命缺陷：
它是脸盲的，完全不知道词的顺序。

对它来说，“狗咬人”和“人咬狗”，送进去的词向量是一摸一样的。为了让模型知道词的先后顺序，我们必须给每个词强行打上一个“位置戳”。

## 绝对位置和相对位置

- **绝对位置编码（如BERT，GPT-2）：** 相当于给每个词发一个固定的座位号，词A坐在1号词，词B坐在2号位。
  - 缺点：这种方式太死板。如果训练时只见过1000个座位，推理时突然来了第1001个词，模型就傻眼了（没有座位号发给它了）。
- **相对位置编码（如T5）：** 相当于不发座位号，只在两个词交流时，告诉它们”你俩隔了3个座位“。
  - 缺点：这种方法虽然符合直觉，但要在Attention的注意力矩阵里做非常复杂的加法运算，计算极其缓慢。

## ROPE：”指南针与相对角度“
RoPE的天才之处在于：**它在输入端用的是“绝对位置”，但在计算Attention时，奇迹般地变成了“相对位置”**。

比喻：

假如词汇表里的每个词（Token）都是一个手里拿着指南针的开会人员。
1. **绝对旋转（打下位置戳）：**  
   当第1个词进场时，我们让它的指南针顺时针旋转10&#176;（绝对角度）。
   当第2个词进场时，旋转20&#176;。
   当第m个词进场时，旋转m x 10&#176;。
   每个词的**绝对位置**，被转化为了一个**绝对的旋转角度**。
2. **相对交流（*Attention*的内积）：**  
    在*Attention*机制中，两个词是怎么交流的？是通过向量的**点积（*Dot Product*）**。在数学上，两个旋转过的向量做点积，它们的结果只和他们之间的**夹角差**有关。  
    第5个词（50&#176;）和第3个词（30&#176;）交流，夹角差是20&#176;。  
    第105个词（1050&#176;）和第103个词（1030&#176;）交流，夹角差依然是20&#176;。

**RoPE精髓：** 明明只给每个词单独旋转了它自己的绝对角度，但当它们两两做点积时，包含绝对位置的信息抵消了，**只留下了它们之间的相对距离**。

## 数学表达
假设有一个二维向量 $(q_1, q_2)$，它在位置 $m$ 的旋转变换公式为：
$$
\begin{pmatrix} \cos m\theta & -\sin m\theta \\ \sin m\theta & \cos m\theta \end{pmatrix} 
\begin{pmatrix} q_1 \\ q_2 \end{pmatrix}
$$
*(注：$\theta$ 是基于特征维度预设的基础旋转频率常量)*
   
## 为什么大模型全换成了RoPE？

1. **外推性极强（Length Extrapolation）：** 这是RoPE最牛的地方，如果训练时模型只见过4096长度的文章，推理时你塞给它8192长度的文章，虽然它没见过4097号座位，
但它懂得角度的相对关系，稍微改一改旋转的底数（比如说**NTK-Aware插值/Yarn算法**，也是长文本大模型的续命神技），模型就能轻松处理无限长的文本。
2. **不需要训练参数：** 这个旋转角度是固定的数学公式，直接算出来提前存好（Buffer）就行，不需要像以前那样去学习一堆位置参数。
3. 计算极快：相比于传统的相对位置编码，RoPE完全是乘法操作，非常契合GPU的并行计算。

## RoPE代码

```js
import torch

def precompute_freqs_cis(dim: int, end: int, theta: float = 10000.0):
    """
    预计算旋转的角度 (也就是你代码里的 freqs_cos 和 freqs_sin)
    dim: 每个头的维度大小
    end: 序列的最大长度 (比如 4096)
    theta: 旋转的基础频率常量
    """
    # 算出每个维度的旋转频率 (就是刚才比喻里的基础角度，有大有小)
    freqs = 1.0 / (theta ** (torch.arange(0, dim, 2)[: (dim // 2)].float() / dim))
    
    # 生成位置索引 [0, 1, 2, ..., end-1]
    t = torch.arange(end, device=freqs.device) 
    
    # 位置索引 乘以 频率，得到每个位置、每个维度的绝对旋转角度 (m * theta)
    freqs = torch.outer(t, freqs).float()  # 形状: [end, dim/2]
    
    # 算出这些角度的 cos 和 sin 值，存起来备用！
    # 这就是你代码里不需要梯度同步的那两个常量
    freqs_cos = torch.cos(freqs) 
    freqs_sin = torch.sin(freqs) 
    return freqs_cos, freqs_sin

def apply_rotary_emb(xq, freqs_cos, freqs_sin):
    """
    把预先算好的角度，真实地“旋转”到查询向量 Query 或键向量 Key 上
    """
    # xq 形状: [batch, seq_len, num_heads, head_dim]
    
    # 把最后一个维度两两分组，准备做二维旋转
    xq_ = xq.float().reshape(*xq.shape[:-1], -1, 2)
    
    # 取出两两分组里的 第一个数(x) 和 第二个数(y)
    xq_0, xq_1 = xq_[..., 0], xq_[..., 1]
    
    # 套用二维旋转矩阵的公式：
    # x_new = x * cos - y * sin
    # y_new = x * sin + y * cos
    xq_out_0 = xq_0 * freqs_cos - xq_1 * freqs_sin
    xq_out_1 = xq_0 * freqs_sin + xq_1 * freqs_cos
    
    # 把旋转后的结果拼回去，恢复原本的形状
    xq_out = torch.stack([xq_out_0, xq_out_1], dim=-1).flatten(3)
    return xq_out.type_as(xq)

# ========== 测试一下 ==========
if __name__ == "__main__":
    seq_len = 10
    head_dim = 64 # 注意：RoPE 是作用在每个 Head 上的
    
    # 1. 模型初始化时：预先算好角度缓存 (不参与梯度更新)
    freqs_cos, freqs_sin = precompute_freqs_cis(dim=head_dim, end=seq_len)
    
    # 2. 前向传播时：拿到输入向量 (比如 Q 或 K)
    # 假设 batch=1, seq_len=10, num_heads=4, head_dim=64
    q = torch.randn(1, 10, 4, 64) 
    
    # 3. 给 Q 向量强行施加“旋转魔法”
    # 注意把 cos 和 sin 的形状对齐
    q_rotated = apply_rotary_emb(q, freqs_cos.unsqueeze(0).unsqueeze(2), freqs_sin.unsqueeze(0).unsqueeze(2))
    
    print("原始 Query 形状:", q.shape)
    print("旋转后 Query 形状:", q_rotated.shape)
    print("RoPE 施加成功！现在这个 Q 已经自带相对位置信息了。")
```

# YaRN（Yet another RoPE extentioN method）

## 为什么有了RoPE，模型依然无法直接读长文本
RoPE就像是给每个词发了一个指南针。假设在训练模型时，最多只给它看过4096个词（也就是它只见过转了$4096 \times \theta$ 角度的指南针 ）。

如果推理时，突然塞给它8192个词，到了第4097个词，它拿到的旋转角度时它这辈子都没见到的。这个时候，Attention机制就会彻底崩溃，模型开始胡言乱语（就是外推行失效）。那怎么办呢？
## 失败的尝试：线性插值法（Linear Interpolation， PI）
最直观的想法：**强行缩放（把地图整体缩小一半）**。

想看8192个词？ok，把第2个词的指南针角度，当成第1个词用；把第8192个词的角度，压缩成4096的角度。相当于把刻度尺强行压扁。
* **惨痛后果：** 就像你在手机上把地图强行缩小了一倍。虽然你看到了 8km 的全局视野（长文本看懂了），但是局部的小街道（相邻的几个词）糊成了一团，完全看不清了！ 模型虽然能处理长文，但它对局部细节的理解能力直线下降。

## 过渡方案：NTK-Aware插值（区别对待）
研究人员发现，RoPE 里的那些旋转角度（频率）是有区别的：

* **高频维度（转得快的齿轮）：** 负责辨别距离极近的词（比如“我”和“爱”）。

* **低频维度（转得慢的齿轮）：** 负责辨别距离极远的词（比如第一段和最后一段）。

NTK 算法提出：**不能一刀切地缩小地图**， 对于局部的小街道（高频），我们保持原样不缩放，保证局部清晰；对于远处的宏观轮廓（低频），我们狠狠地把它压缩，让它能塞进 4096 的视野里。

## 最终方案：YaRN的分组处理与温度修复
YaRN是在NTK的基础上，把这种区别对待做到了极致和完美，它做了两件极其核心的事情：

### 第一件事：将RoPE的维度分为三组（频率分组）
1. **高频区（局部细节）：绝对不插值**，完全保持原样，让模型对近距离词汇的理解保持100%的精准度。
2. **低频区（长程宏观）：纯粹的线性插值**，直接按比例压缩，强行把远处的词拉回模型熟悉的视野范围内。
3. **中频区（过度地带）：** 使用⚙️平滑的过渡函数（平滑余弦插值），让高频的不变和低频的压缩在这里完美融合，防止出现断层。
   
### 第二件事：温度调节（Temperature Scaling）
研究人员在推导复杂的数学公式时发现了一个致命 Bug：当你把低频区强行压缩插值后，虽然距离对上了，但是 Attention 计算出来的注意力方差变小了（也就是说，原本清晰的注意力变得“涣散”了）。
为了解决这个问题，YaRN 强行在算出来的 Attention 分数上乘以一个**温度系数 $t$**（通常大于 1）。
这就相当于在看压缩后的地图时，**强行增加了一把“对比度”**，让涣散的注意力重新聚焦，彻底修复了插值带来的副作用。

## Yarn为什么受欢迎
性价比高，有了Yarn,如果手里有一个只能支持8K上下文的开源模型（比如最初的Llama 2），**几乎不需要重新训练**（或者只需要用几百条长文本稍微调几步），就能直接把它的上下文窗口无损扩展到32K甚至128K。

## Yarn的代码实现
```js
import torch
import math

def precompute_yarn_freqs_cis(
    dim: int, 
    end: int, 
    original_max_position: int = 4096, 
    scale: float = 8.0, 
    base: float = 10000.0,
    alpha: float = 1.0, 
    beta: float = 32.0
):
    """
    预计算 YaRN 版本的 RoPE 角度缓存
    Args:
        dim: 每个注意力头的特征维度
        end: 你想要扩展到的最终长度 (如 32768)
        original_max_position: 模型原本训练的最大长度 (如 4096)
        scale: 扩展倍数 s (如 32768 / 4096 = 8.0)
        base: RoPE 的基础频率 theta
        alpha, beta: YaRN 分区的边界参数
    """
    # 1. 基础频率计算 (和原版 RoPE 一样)
    inv_freq = 1.0 / (base ** (torch.arange(0, dim, 2).float() / dim))
    
    # 2. 计算每个维度的波长 (Wavelength)
    # 波长 = 2 * pi / 频率
    wavelengths = 2 * math.pi / inv_freq
    
    # 3. 计算 YaRN 的混合权重 gamma
    # clamp 保证 gamma 严格在 [0, 1] 之间
    gamma = (wavelengths - alpha * original_max_position) / (
        (beta - alpha) * original_max_position
    )
    gamma = torch.clamp(gamma, min=0.0, max=1.0)
    
    # 4. 根据三分天下规则，计算插值比例 (缩放系数)
    # 当 gamma=0(高频)时，scale_factor=1 (不压缩)
    # 当 gamma=1(低频)时，scale_factor=scale (按倍数压缩)
    scale_factor = (1 - gamma) * 1.0 + gamma * scale
    
    # 5. 修改原版频率，得到 YaRN 的新频率
    inv_freq_yarn = inv_freq / scale_factor
    
    # 6. 生成位置索引 [0, 1, ..., end-1]
    t = torch.arange(end, device=inv_freq.device, dtype=torch.float32)
    
    # 计算外积，得到绝对旋转角度
    freqs = torch.outer(t, inv_freq_yarn)
    
    # 7. 温度修复 (Temperature Scaling) 核心代码！
    # 算乘数放大因子 t，并对 Q 和 K 进行隐式放大
    t_factor = 0.1 * math.log(scale) + 1.0
    mscale = math.sqrt(t_factor)
    
    # 算出 cos 和 sin，并乘上放大因子
    freqs_cos = torch.cos(freqs) * mscale
    freqs_sin = torch.sin(freqs) * mscale
    
    return freqs_cos, freqs_sin

# ========== 测试一下这段代码 ==========
if __name__ == "__main__":
    head_dim = 128
    original_len = 4096
    target_len = 32768
    scale_ratio = target_len / original_len # 倍数为 8.0
    
    # 获取 YaRN 版本的 cos 和 sin
    freqs_cos, freqs_sin = precompute_yarn_freqs_cis(
        dim=head_dim, 
        end=target_len, 
        original_max_position=original_len, 
        scale=scale_ratio
    )
    
    print("=== YaRN 频率计算完成 ===")
    print(f"生成的 freqs_cos 形状: {freqs_cos.shape} (应为 [32768, 64])")
    print(f"高频区 (第0维) 前3个位置的 cos 值: {freqs_cos[:3, 0].tolist()}")
    # 你会发现得到的值幅值略大于 1，这就是温度修复 mscale 在起作用！
```