---
title: NLP基石：Transformer
date: 2021-09-01
draft: false
mathjax: true
toc: true
#tocBorder: true
---

2017年11月，是计算机人工智能领域一个重要的时间节点。因为在那个时候，论文《[Attention Is All You Need][Attention is all you need]》发布了。Transformer的发布，为NLP领域新的研究范式（预训练+微调）奠定了模型基础，也成为了人工智能众多领域的一大基石，成千上万的前沿研究或多或少都发现它的身影。

## Model Architecture

### 1. Encoder & Decoder 

Encoder由6个独立层组成，每层都由多头注意力子层（Multi-Head Attention Sublayer）和前馈网络子层（Feed-Forward Network Sublayer）组成。每个子层之后都应用了残差连接和LayerNorm，这两个小模块的引入都是为了使模型在训练过程中更好地管理梯度流，从而实现高效训练，大大减少模型训练时间。

Decoder也是由6个独立层组成，每层在Encoder的两个子层之前增加一个Mask Multi-Head Attention Sublayer，主要是因为在顺序decoder模式中，当前词的context只能包含input sentence的全部和output sentence的上文，不能包含下文。

### 2. Attention

具体图示详情可参见[Blog][illustrated-transformer]

#### 2.1 Scaled Dot-Product Attention

引入注意力机制，可以采用加性注意力或者点积注意力，文章比较两者的优缺点之后选择后者，因为高度优化后的矩阵乘法计算更快以及空间利用效率高。当${d_k}$较小时，两种注意力表现相似，但较大时，加性注意力明显表现更佳，这是因为点积注意力的值随着${d_k}$的增大而增大，从而推向softmax的低梯度区域，不利于训练，于是在点积的基础上城上一个放缩系数，保证点积后的结果不会太大。至于为什么乘${\frac{1}{\sqrt{d_k}}}$而不是其他系数，这是因为在Q、K中每个元素都服从均值为0，方差为1的假设下，点积的结果服从均值为0，方差为${d_k}$，为了维持结果方差为1，就必须乘上${\frac{1}{\sqrt{d_k}}}$。

训练的decoder阶段存在Mask，则在softmax之前将被Mask部分的值设置为负无穷，以实现“只看到上文，看不到下文”的效果。预测阶段由于不存在下文，所以也不用Mask，只需要每次decoder出来一个词，就将这个词放在之前decoder出来的所有词后面，再次推入Decoder中，取当前位置的隐层向量经过softmax层获得下一个词的分布概率。

$$
Attention(Q,K,V) = softmax(\frac{QK^T}{\sqrt{d_k}})V
$$

#### 2.2 Multi-Head Attention

使用多头注意力机制而不是全连接单头注意力，主要有以下几个原因：

* 多头注意力机制允许模型从不同表征空间联合抽取信息（似乎有一种ensemble的意味）
* 多头允许模型并行化，加速模型训练和预测
* 多头注意力和全连接单头注意力在计算量上是规模相同的

$$
\begin{align}
MultiHead(Q,K,V) &= Concat(head_1,...,head_n)W^O \\\\
head_i &= Attention(QW_i^Q,KW_i^K,VW_i^V) \\\\
W_i^Q, W_i^K &\in d_{model}*d_k, W_i^v \in d_{model}*d_v \\\\
d_k &= d_v = 64, h = 8, d_{model} = 512 \\\\
\end{align}
$$

#### 3. Position-wise Feed-Forward Networks

全连接前馈神经网络中包含两个线性变换层和一个ReLU激活层。值得注意的是，每个位置独立进行FFN且参数是相同的，但是不同层的FFN参数都是不一样的。相当于同一层中不同位置使用相同参数进行FFN，不同层使用不同参数进行FFN。
$$
\begin{align}
FFN(x) &= max(0, xW_1 + b_1 )W_2 + b_2 \\\\
d_{model} &= 512, d_{ff} = 2048
\end{align}
$$

#### 4. Embeddings and Softmax

embedding层和decoder之后的线性层（线性层后就是softmax层）共享相同的参数，维度都是${d_{vocab}*d_{model}}$。值得注意的是，inputids过embedding层时，会乘上一个系数${\sqrt{d_{model}}}$。

#### 5. Positional Encoding

由于模型没有采用RNN或者CNN架构，为了识别token在文本中的位置并利用文本的顺序信息，需要引入相对位置或者绝对位置关系编码。文中位置编码采用sin/cos形式向量，维度和embedding层保持一致，方便直接相加。采用这个函数进行位置编码，是因为作者假设模型能够通过相对位置很容易地学习到attention矩阵：对于固定的位置偏置k，${PE_{pos+k}}$能够表示为${PE_{pos}}$的线性函数。

位置编码的选择有两种，一是随着模型一起学习的参数，二是能够直接计算得到的固定参数。实验比较发现，两者效果上是差不多的，于是选择固定形式，这样一方面减少了不必要的参数，另一方面在模型在预测阶段可以突破训练阶段数据的最大长度。
$$
\begin{align}
PE(pos,2i) &= sin(pos/10000^{2i/d_{model}}) \\\\
PE(pos,2i+1) &=cos(pos/10000^{2i/d_{model}})
\end{align}
$$

## Why Self-Attention

作者对RNN、CNN、Transformer为基础架构的seq2seq模型进行横向比较，具体来说，从每层的计算复杂度、可并行计算的能力、最大路径依赖长度（用于衡量长距离依赖能力，总所周知，长距离依赖是众多序列任务的一个关键瓶颈）三方面进行比较。Transformer充分吸收了其他两种架构的优点，既能像CNN一样并行计算，又能像RNN一样减少计算复杂度，最重要的是，在最大路径依赖长度方面，Transformer通过attention做到了${O(1)}$级别，远远超过RNN的${O(n)}$和CNN的${O(log_kn)}$，从而实现了真正意义上的良好长距离依赖性能。

就像word2vec一样，Transformer也有一些副产品能够对理解“黑箱”一样的神经网络模型作出贡献。将encoder最后一层的Attention Map抽取出来，会发现一些词attend到与之最相关的其他词，对比不同head出来的不同Attention Map，会发现不同head会在不同方面进行attend，也即是相同的词在不同head中attend到的词集合是不同的。这一点在后续的相关研究中会继续发挥作用。

## Training

### 1. Setting

* Optimizer：Base版本训练英-德数据共100k steps，其中warmup_steps为4k（占比4%）

* Regularization：残差连接之后采用dropout，embeddings和position encodings相加之后也会采用dropout，dropout=0.1

* Label Smoothing：训练阶段采用参数值为0.1的Label Smoothing，这样会造成PPL的下降，因为模型会学习得更加不确定，但是能提高ACC和BLEU值

* Checkpoint：机器翻译任务中，选择last 5/20 checkpoint进行平均得到最终模型；预测时设置最大长度比训练时大一些

### 2. Model Variations

作者做了一系列实验，探究head数量、$d_k$大小、layer数量等模型结构超参数对模型效果的影响，发现以下几点：

* 参数量一定的情况下，head越多或者越少，都会造成模型效果的下降；head越少，损失不同表征空间信息，head越多，用于计算注意力的参数就太少，导致不足以计算得到正确的attention结果
* 正确注意力的计算并不是一件简单的事情，采用更加复杂的注意力计算方式似乎会更加有益
* 模型越大，dropout越高，越能缓解过拟合问题


[Attention is all you need]: https://arxiv.org/pdf/1706.03762.pdf
[illustrated-transformer]: https://jalammar.github.io/illustrated-transformer/

