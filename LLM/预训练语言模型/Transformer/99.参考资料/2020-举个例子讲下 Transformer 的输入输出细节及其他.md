> [原文地址](https://zhuanlan.zhihu.com/p/166608727)

# 举个例子讲下 transformer 的输入输出细节及其他

最近由于工作需要，将 transformer 的相关资料看了下，网上很多关于 transformer 的讲解，但是很多都只讲了整个架构，涉及到的细节都讲的不是很清楚，在此将自己关于某些细节的体 会写出来，大家一起学习探讨下。

下图是 transformer 的原始架构图，就不细讲了。

![img](https://pic2.zhimg.com/80/v2-e7daa13fb15fb21fb133c2099361ccc9_1440w.webp)

主要讲下数据从输入到 encoder 到 decoder 输出这个过程中的流程（以机器翻译为例子）：

**1.encoder**

对于机器翻译来说，一个样本是由原始句子和翻译后的句子组成的。比如原始句子是： “我爱机器学习”，那么翻译后是 ’i love machine learning‘。则该一个样本就是由“我爱机器学习”和 "i love machine learning" 组成。

这个样本的原始句子的单词长度是 length=4,即‘我’ ‘爱’ ‘机器’ ‘学习’。经过 embedding 后每个词的 embedding 向量是 512。那么“我爱机器学习”这个句子的 embedding 后的维度是[4，512 ] （若是批量输入，则 embedding 后的维度是[batch, 4, 512]）。

**padding**

因为每个样本的原始句子的长度是不一样的，那么怎么能统一输入到 encoder 呢。此时 padding 操作登场了，假设样本中句子的最大长度是 10，那么对于长度不足 10 的句子，需要补足到 10 个长度，shape 就变为[10, 512], 补全的位置上的 embedding 数值自然就是 0 了

**Padding Mask**

对于输入序列一般我们都要进行 padding 补齐，也就是说设定一个统一长度 N，在较短的序列后面填充 0 到长度为 N。对于那些补零的数据来说，我们的 attention 机制不应该把注意力放在这些位置上，所以我们需要进行一些处理。具体的做法是，把这些位置的值加上一个非常大的负数(负无穷)，这样经过 softmax 后，这些位置的权重就会接近 0。Transformer 的 padding mask 实际上是一个张量，每个值都是一个 Boolean，值为 false 的地方就是要进行处理的地方。

**Positional Embedding**

得到补全后的句子 embedding 向量后，直接输入 encoder 的话，那么是没有考虑到句子中的位置顺序关系的。此时需要再加一个位置向量，位置向量在模型训练中有特定的方式，可以表示每个词的位置或者不同词之间的距离；总之，核心思想是在 attention 计算时提供有效的距离信息。

关于 positional embedding ，文章提出两种方法：

1.Learned Positional Embedding ，这个是绝对位置编码，即直接对不同的位置随机初始化一个 postion embedding，这个 postion embedding 作为参数进行训练。

2.Sinusoidal Position Embedding ，相对位置编码，即三角函数编码。

下面详细讲下 Sinusoidal Position Embedding 三角函数编码。

Positional Embedding 和句子 embedding 是 add 操作，那么自然其 shape 是相同的也是[10, 512] 。

Sinusoidal Positional Embedding 具体怎么得来呢，我们可以先思考下，使用**绝对位置编码，**不同位置对应的 positional embedding 固然不同，但是位置 1 和位置 2 的距离比位置 3 和位置 10 的距离更近，位置 1 和位置 2 与位置 3 和位置 4 都只相差 1。

这些关于位置的**相对含义，**模型能够通过绝对位置编码参数学习到吗？此外使用 Learned Positional Embedding 编码，位置之间没有约束关系，我们只能期待它隐式地学到，是否有更合理的方法能够显示的让模型理解位置的相对关系呢？

肯定是有的，首先由下述公式得到 Embedding 值:

![img](https://pic1.zhimg.com/80/v2-5589e776fd8510eab7a3d87de01580d4_1440w.webp)

对于句子中的每一个字，其位置 pos∈[0,1,2,…,9](假设每句话10个字), 每个字是 N（512）维向量，维度 i （i∈[ 0,1,2,3,4,..N]）带入函数

![img](https://pic4.zhimg.com/80/v2-d02e2346c76f5a829330d915cddd0403_1440w.webp)

**由于正弦函数能够表达相对位置信息**，那么对每个 positional embedding 进行 sin 或者 cos 激活，可能效果更好，那就再将偶数列上的 embedding 值用 sin()函数激活，奇数列的 embedding 值用 cos()函数激活得到的具体示意图如下:

![img](https://pic1.zhimg.com/80/v2-afa76dbf4afe436c658fdae9e72eb320_1440w.webp)

这样使用三角函数设计的好处是位置 i 处的单词的 psotional embedding 可以被位置 i+k 处单词的 psotional embedding 线性表示，反应两处单词的其相对位置关系。此外位置 i 和 i+k 的 psotional embedding 内积会随着相对位置的递增而减小，从而表征位置的相对距离。

但是不难发现，由于距离的对称性，Sinusoidal Position Encoding 虽然能够反映相对位置的距离关系，但是无法区分 i 和 i+j 的方向。即**pe(i)\*pe(i+j) =pe(i)\*pe(i-k)** (具体解释参见引用链接 1)

**另外，从参数维度上**，使用三角函数 Position Encoding 不会引入额外参数，Learned Positional Embedding 增加的参数量会随序列语句长度线性增长。**在可扩展性上**，Learned Positional Embedding 可扩展性较差，只能表征在 max*\*seq*\*length 以内的位置，而三角函数 Position Encoding 没有这样的限制，可扩展性更强。

**attention**

![img](https://pic3.zhimg.com/80/v2-8f0301ee897ea88a71e16312812c4b0e_1440w.webp)

关于 attention 操作，网上讲的很多，也很简单，就不写了。不过需要值得注意的一点是，单头 attention 的 Q/K/V 的 shape 和多头 attention 的每个头的 Qi/Ki/Vi 的大小是不一样的，假如单头 attention 的 Q/K/V 的参数矩阵 WQ/WK/WV 的 shape 分别是[512, 512](此处假设encoder的输入和输出是一样的shape)，那么多头 attention (假设 8 个头)的每个头的 Qi/Ki/Vi 的参数矩阵 WQi/WKi/WVi 大小是[512，512/8].

**FeedForward**

很简单，略

**add/Norm**

经过 add/norm 后的隐藏输出的 shape 也是[10,512]。（当然你也可以规定为[10, x]，那么 Q/K/V 的参数矩阵 shape 就需要变一下）

**encoder 输入输出**

让我们从输入开始，再从头理一遍单个 encoder 这个过程:

- 输入 x
- x 做一个层归一化： x1 = norm(x)
- 进入多头 self-attention: x2 = self_attention(x1)
- 残差加成：x3 = x + x2
- 再做个层归一化：x4 = norm(x3)
- 经过前馈网络: x5 = feed_forward(x4)
- 残差加成: x6 = x3 + x5
- 输出 x6

![img](https://pic1.zhimg.com/80/v2-19aeff0b0eabd2f25941718d1785a734_1440w.webp)

以上就是一个 Encoder 组件所做的全部工作了

---

**2.decoder**

![img](https://pic2.zhimg.com/80/v2-6456b10631f0cc97d70bee6f555cf941_1440w.webp)

**\*注意 encoder 的输出并没直接作为 decoder 的直接输入。\***

训练的时候，1.初始 decoder 的 time step 为 1 时(也就是第一次接收输入)，其输入为一个特殊的 token，可能是目标序列开始的 token(如<BOS>)，也可能是源序列结尾的 token(如<EOS>)，也可能是其它视任务而定的输入等等，不同源码中可能有微小的差异，其目标则是预测翻译后的第 1 个单词(token)是什么；2.然后<BOS>和预测出来的第 1 个单词一起，再次作为 decoder 的输入，得到第 2 个预测单词；3 后续依此类推；

具体的例子如下：

样本：“我/爱/机器/学习”和 "i/ love /machine/ learning"

**训练：**
\1. 把“我/爱/机器/学习”embedding 后输入到 encoder 里去，最后一层的 encoder 最终输出的 outputs [10, 512]（假设我们采用的 embedding 长度为 512，而且 batch size = 1),此 outputs 乘以新的参数矩阵，可以作为 decoder 里每一层用到的 K 和 V；

\2. 将<bos>作为 decoder 的初始输入，将 decoder 的最大概率输出词 A1 和‘i’做 cross entropy 计算 error。

\3. 将<bos>，"i" 作为 decoder 的输入，将 decoder 的最大概率输出词 A2 和‘love’做 cross entropy 计算 error。

\4. 将<bos>，"i"，"love" 作为 decoder 的输入，将 decoder 的最大概率输出词 A3 和'machine' 做 cross entropy 计算 error。

\5. 将<bos>，"i"，"love "，"machine" 作为 decoder 的输入，将 decoder 最大概率输出词 A4 和‘learning’做 cross entropy 计算 error。

\6. 将<bos>，"i"，"love "，"machine"，"learning" 作为 decoder 的输入，将 decoder 最大概率输出词 A5 和终止符</s>做 cross entropy 计算 error。

**Sequence Mask**

上述训练过程是挨个单词串行进行的，那么能不能并行进行呢，当然可以。可以看到上述单个句子训练时候，输入到 decoder 的分别是

<bos>

<bos>，"i"

<bos>，"i"，"love"

<bos>，"i"，"love "，"machine"

<bos>，"i"，"love "，"machine"，"learning"

那么为何不将这些输入组成矩阵，进行输入呢？这些输入组成矩阵形式如下：

【<bos>

<bos>，"i"

<bos>，"i"，"love"

<bos>，"i"，"love "，"machine"

<bos>，"i"，"love "，"machine"，"learning" 】

怎么操作得到这个矩阵呢？

将 decoder 在上述 2-6 步次的输入补全为一个完整的句子

【<bos>，"i"，"love "，"machine"，"learning"
<bos>，"i"，"love "，"machine"，"learning"
<bos>，"i"，"love "，"machine"，"learning"
<bos>，"i"，"love "，"machine"，"learning"
<bos>，"i"，"love "，"machine"，"learning"】

然后将上述矩阵矩阵乘以一个 mask 矩阵

【1 0 0 0 0

1 1 0 0 0

1 1 1 0 0

1 1 1 1 0

1 1 1 1 1 】

这样是不是就得到了

【<bos>

<bos>，"i"

<bos>，"i"，"love"

<bos>，"i"，"love "，"machine"

<bos>，"i"，"love "，"machine"，"learning" 】

这样的矩阵了 。着就是我们需要输入矩阵。这个 mask 矩阵就是 sequence mask，其实它和 encoder 中的 padding mask 异曲同工。

这样将这个矩阵输入到 decoder（其实你可以想一下，此时这个矩阵是不是类似于批处理，矩阵的每行是一个样本，只是每行的样本长度不一样，每行输入后最终得到一个输出概率分布，作为矩阵输入的话一下可以得到 5 个输出概率分布）。

这样我们就可以进行并行计算进行训练了。

**测试**

训练好模型，测试的时候，比如用 '机器学习很有趣'当作测试样本，得到其英语翻译。

这一句经过 encoder 后得到输出 tensor，送入到 decoder(并不是当作 decoder 的直接输入)：

1.然后用起始符<bos>当作 decoder 的 输入，得到输出 machine

2. 用<bos> + machine 当作输入得到输出 learning

3.用 <bos> + machine + learning 当作输入得到 is

4.用<bos> + machine + learning + is 当作输入得到 interesting

5.用<bos> + machine + learning + is + interesting 当作输入得到 结束符号<eos>

我们就得到了完整的翻译 'machine learning is interesting'

可以看到，在测试过程中，只能一个单词一个单词的进行输出，是串行进行的。
