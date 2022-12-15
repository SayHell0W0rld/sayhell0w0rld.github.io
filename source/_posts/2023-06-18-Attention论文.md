---
title: Attention Is All You Need 论文精读：我理解Transformer架构的全过程
date: 2023-06-18 10:00:00
tags:
  - Transformer
  - 深度学习
  - 论文
categories:
  - 学习笔记
---

## 写在前面

<!-- more -->

最近在学 Transformer 架构，绕不开这篇 2017 年 Google 团队发表的论文——*Attention Is All You Need*。说实话，第一次读这篇论文的时候，很多概念似懂非懂，尤其是 Multi-Head Attention 和 Positional Encoding 部分。后来翻了几篇解读博客，加上自己动手写了点伪代码，才算真正理清了整篇论文的脉络。这篇文章就是把我理解的过程记录下来，希望能帮到同样在啃这篇论文的人。

---

## 一、为什么需要 Transformer？

### RNN 的瓶颈

在 Transformer 出现之前，NLP 领域的主流方案是 RNN（循环神经网络）及其变体 LSTM、GRU。RNN 处理序列的方式是"一个词一个词地看"：每处理一个新词，就把之前所有词的信息压缩进一个固定长度的 hidden state。这种串行处理方式带来了两个根本性的问题：

1. **无法并行**：第 t 个词的计算必须等第 t-1 个词算完，训练速度极慢，多 GPU 也没用。
2. **长距离依赖**：句子越长，早期词汇的信息在 hidden state 里衰减越严重。比如一句话里"它"指代的名词在十个词之前，RNN 很难把这种关联记住。

CNN 在序列任务上也有尝试，但卷积核只能捕捉局部上下文，要处理长距离依赖就得堆很多层，效率同样不高。

### Transformer 的核心思路

Transformer 的想法很直接：既然 RNN 的问题是串行，CNN 的问题是局部，那就干脆不用 RNN 也不用 CNN，完全用 Attention 机制来建模序列中的依赖关系。Attention 的好处是，任意两个位置之间可以直接计算关联度，不需要经过中间步骤，天然支持并行。

---

## 二、Attention 机制详解

### 2.1 Scaled Dot-Product Attention

Attention 的核心公式：

```
Attention(Q, K, V) = softmax(QK^T / sqrt(d_k)) * V
```

这里涉及三个概念：

- **Q (Query)**：当前词"想要找什么信息"
- **K (Key)**：每个词"对外展示的标识"
- **V (Value)**：每个词"实际携带的信息"

计算过程分三步：

1. 用 Q 和所有 K 做点积，得到一个"相关性分数"向量
2. 除以 sqrt(d_k) 进行缩放（原因后面说），再做 softmax 归一化成权重
3. 用权重对所有 V 做加权求和，得到输出

**为什么要除以 sqrt(d_k)？** 假设 Q 和 K 的每个元素都是均值为 0、方差为 1 的随机变量，那么 Q 和 K 的点积的均值为 0，方差为 d_k。当 d_k 很大（比如 512）时，点积值可能很大，导致 softmax 输出接近 one-hot 分布——几乎所有权重都集中在某一个位置上，梯度接近 0，模型没法学。除以 sqrt(d_k) 把方差拉回 1，softmax 的输入落在合理范围内。

### 2.2 Self-Attention（自注意力）

Self-Attention 就是 Attention 机制的一种特殊应用：Q、K、V 全部来自同一个输入序列。

举个例子，处理句子 "The cat sat on the mat because it was tired"，当模型要理解 "it" 这个词时，Self-Attention 会让 "it" 同时去查看句子中所有其他词的 K，发现 "cat" 的 K 和自己的 Q 最匹配，于是从 "cat" 的 V 中提取信息。这样 "it" 的输出向量就融合了 "cat" 的语义。

### 2.3 Multi-Head Attention（多头注意力）

一个 Attention 头只能学到一种关联模式。但语言中的依赖关系是多样的：语法关系、语义关系、指代关系……需要多个"角度"来捕捉。

Multi-Head Attention 的做法：

1. 把 Q、K、V 分别用不同的权重矩阵投影到 d_model/h 维（h 是头数，论文中 h=8）
2. 每个头独立做一次 Scaled Dot-Product Attention
3. 把 h 个头的输出拼接起来，再用一个线性变换映射回 d_model 维

```python
# 伪代码
def multi_head_attention(x, num_heads=8):
    d_model = x.shape[-1]
    d_k = d_model // num_heads
    
    Q = linear_q(x)  # (batch, seq_len, d_model)
    K = linear_k(x)
    V = linear_v(x)
    
    # 拆成 h 个头
    Q = reshape(Q, (batch, seq_len, num_heads, d_k))  # (batch, seq_len, h, d_k)
    K = reshape(K, (batch, seq_len, num_heads, d_k))
    V = reshape(V, (batch, seq_len, num_heads, d_k))
    
    # 每个头独立计算 attention
    output = []
    for i in range(num_heads):
        attn_out = scaled_dot_product_attention(Q[:,:,i], K[:,:,i], V[:,:,i])
        output.append(attn_out)
    
    # 拼接并投影
    concat = concatenate(output, dim=-1)  # (batch, seq_len, d_model)
    return linear_o(concat)
```

---

## 三、Transformer 整体架构

Transformer 遵循经典的 Encoder-Decoder 框架，原论文用于机器翻译任务：Encoder 把源语言句子编码成上下文向量，Decoder 基于这些向量和已生成的目标语言词汇，逐词生成翻译结果。

### 3.1 Encoder

Encoder 由 N=6 个相同的层堆叠而成，每层包含两个子层：

1. **Multi-Head Self-Attention**：捕捉输入序列内部的全局依赖
2. **Position-wise Feed-Forward Network**：对每个位置独立做两层全连接网络

```
FFN(x) = max(0, x * W1 + b1) * W2 + b2
```

每个子层都带有 **残差连接（Residual Connection）** 和 **层归一化（Layer Normalization）**：

```
output = LayerNorm(x + Sublayer(x))
```

残差连接让梯度能直接跨层传播，缓解深层网络的退化问题。

### 3.2 Decoder

Decoder 同样由 N=6 个层堆叠，每层包含三个子层：

1. **Masked Multi-Head Self-Attention**：防止位置 i 关注到位置 i 之后的词（因为生成时还看不到未来的词）
2. **Multi-Head Cross-Attention**：Q 来自 Decoder 的上一层，K 和 V 来自 Encoder 的输出——这是 Decoder "查看"输入句子的地方
3. **Position-wise Feed-Forward Network**

同样每个子层都有残差连接 + 层归一化。

### 3.3 Positional Encoding（位置编码）

Self-Attention 本身对输入顺序是"无感"的——你把句子中的词重新排列，每个位置的输出只是跟着重排，不会有任何变化。但语言是有顺序的，"狗咬人"和"人咬狗"意思完全不同。

Transformer 的解决方案是给输入 embedding 加上一个位置编码向量。论文用的是正余弦函数：

```
PE(pos, 2i)   = sin(pos / 10000^(2i/d_model))
PE(pos, 2i+1) = cos(pos / 10000^(2i/d_model))
```

其中 pos 是词在句子中的位置，i 是维度索引。

这种设计有两个好处：
- 每个位置的编码是唯一的
- PE(pos+k) 可以表示为 PE(pos) 的线性函数，模型能学到相对位置关系

### 3.4 完整架构的伪代码

```python
class Transformer(nn.Module):
    def __init__(self, src_vocab_size, tgt_vocab_size, d_model=512, num_heads=8, num_layers=6):
        # Embedding 层
        self.src_embedding = Embedding(src_vocab_size, d_model)
        self.tgt_embedding = Embedding(tgt_vocab_size, d_model)
        self.pos_encoding = PositionalEncoding(d_model)
        
        # Encoder: N 层
        self.encoder_layers = [EncoderLayer(d_model, num_heads) for _ in range(num_layers)]
        
        # Decoder: N 层
        self.decoder_layers = [DecoderLayer(d_model, num_heads) for _ in range(num_layers)]
        
        # 输出层
        self.output_linear = Linear(d_model, tgt_vocab_size)
    
    def forward(self, src, tgt):
        # 编码
        x = self.pos_encoding(self.src_embedding(src))
        for layer in self.encoder_layers:
            x = layer(x)
        enc_output = x
        
        # 解码
        y = self.pos_encoding(self.tgt_embedding(tgt))
        for layer in self.decoder_layers:
            y = layer(y, enc_output)
        
        # 输出概率
        return softmax(self.output_linear(y))

class EncoderLayer(nn.Module):
    def forward(self, x):
        # Multi-Head Self-Attention + 残差 + LayerNorm
        x = layer_norm(x + multi_head_attention(x, x, x))
        # FFN + 残差 + LayerNorm
        x = layer_norm(x + feed_forward(x))
        return x

class DecoderLayer(nn.Module):
    def forward(self, x, enc_output):
        # Masked Multi-Head Self-Attention
        x = layer_norm(x + masked_multi_head_attention(x, x, x))
        # Cross-Attention: Q from decoder, K/V from encoder
        x = layer_norm(x + multi_head_attention(x, enc_output, enc_output))
        # FFN
        x = layer_norm(x + feed_forward(x))
        return x
```

---

## 四、为什么 Self-Attention 比 RNN 好？

论文中给出了一个对比表，核心指标：

| 指标 | RNN | CNN | Self-Attention |
|------|-----|-----|----------------|
| 每层复杂度 | O(n·d²) | O(k·n·d²) | O(n²·d) |
| 最短路径长度 | O(n) | O(log_k(n)) | O(1) |
| 可并行化程度 | 不可 | 部分 | 完全 |

- **最短路径长度**：Self-Attention 中任意两个位置的路径长度是 1，直接就能看到对方；RNN 需要经过 n 步才能从第一个词到达最后一个词。
- **并行化**：Self-Attention 的所有位置可以同时计算，训练效率远高于 RNN。

当然，Self-Attention 的复杂度是 O(n²·d)，序列长度 n 很大时计算量会暴增。这也是后续很多工作（比如 Linformer、Performer）试图解决的问题。

---

## 五、训练细节

论文中几个值得注意的训练技巧：

### 学习率调度

```python
lr = d_model^(-0.5) * min(step_num^(-0.5), step_num * warmup_steps^(-1.5))
```

前 warmup_steps 步学习率线性增大（warmup），之后按 step 的 -0.5 次方衰减。warmup 的作用是避免训练初期梯度不稳定时学习率过大导致模型震荡。

### 标签平滑（Label Smoothing）

训练时不是让模型输出 one-hot 分布（比如 [0, 0, 1, 0]），而是让它输出一个"软"分布（比如 [0.05, 0.05, 0.85, 0.05]）。这能提高模型的泛化能力，防止过拟合。

### 正则化

论文使用了 Residual Dropout 和 Attention Dropout，dropout rate 为 0.1。

---

## 六、实验结果

在 WMT 2014 英德翻译任务上，Transformer (big) 模型取得了 28.4 BLEU，比之前的 SOTA 模型高出 2.0 BLEU，而且训练成本只有前者的 1/40。

在英法翻译任务上，达到了 41.0 BLEU，同样大幅超越之前的所有单模型。

更值得注意的是在 English constituency parsing（句法分析）任务上的表现——这个任务要求模型理解句子的层级结构，和翻译任务差异很大。Transformer 在这个任务上同样表现优异，说明它学到的是语言的通用规律，而非翻译任务的专属模式。

---

## 七、写在最后

读完这篇论文，最大的感受是"简单但深刻"。整个架构的核心就是 Attention 一个机制，但通过精心设计（多头、残差、归一化、位置编码），让它在实际效果上碾压了之前的复杂架构。这也解释了为什么这篇论文的影响力如此之大——它不只是解决了一个问题，而是提供了一个通用的序列建模框架，后来的 BERT、GPT、LLaMA 等大模型全部建立在这个基础之上。

如果你也在读这篇论文，建议：
1. 先理解 Self-Attention 的 Q/K/V 是什么，这是所有后续内容的基础
2. 动手写一个最简版的 Self-Attention，几行代码就能跑通
3. 然后再看 Multi-Head 和 Encoder-Decoder，会清晰很多
4. 最后回到论文原文，对照着看，很多之前不理解的细节会豁然开朗

论文本身只有 11 页，公式也不多，值得反复读。

---

## 参考资料

1. Vaswani, A., et al. "Attention Is All You Need." NeurIPS 2017. [arXiv:1706.03762](https://arxiv.org/abs/1706.03762)
2. 周弈帆. "Attention Is All You Need (Transformer) 论文精读." 2022. [链接](https://zhouyifan.net/2022/11/12/20220925-Transformer)
3. ArthurChiao's Blog. "600 行 Python 代码实现 self-attention 和两类 Transformer." [链接](https://arthurchiao.art/blog/transformers-from-scratch-zh)
4. Datawhale. "Transformer 解读——深入浅出 PyTorch." [链接](https://datawhalechina.github.io/thorough-pytorch/第十章/Transformer%20解读.html)
