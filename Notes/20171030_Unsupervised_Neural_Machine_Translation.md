# [Unsupervised Neural Machine Translation]()

## 流程

1. 先用*单语语料*训练两个词向量空间
2. 然后用<font style="color:Red">无监督方法对齐</font>这两个空间(<font style="color:Red">具体做法参照 Learning bilingual word embeddings with (almost) no bilingual data</font>)。虽然说是无监督，其实也是有监督的：具体做法是用阿拉伯数字做种子词对，把两个空间里的这些种子词的向量学一个**线性变换对齐**。
3. 最后再迭代几轮就能把两个空间对齐得很好了，从空间导出的双语 lexicon 有大约 40% 的准确率。反正不管什么语言的单语语料里都很可能有数字，这种对齐词典的获取几乎是无成本的。

# 模型结构如下图，两种语言共用一个 encoder，各有各的 decoder

![](https://pic2.zhimg.com/50/v2-9ee6dff2563ce227774c498f43e1d5f8_hd.jpg)

用这个模型交替执行如下步骤：

## Denoising

就是把某种语言的句子<font style="color:Red">加一些噪声（随机交换一些词的顺序等）</font>，然后用 shared encoder <font style="color:Red">编码加噪声后的句子，最后用该语言的句子解码恢复它</font>。通过**最大化重构出的概率来训练 shared encoder 和该语言的 decoder**。加噪声的目的是想让 <font style="color:Red">encoder 学会分析语言的结构、提取语义特征，decoder 学一个好的语言模型，而不是仅仅学会复制粘贴</font>.

## Back-translation

语言 $L_1$ 的句子 $s_1$ 先用<font style="color:Red">编码器编码</font>，然后用 $L_2$ decoder greedy decoding出 $s_2$，这样就造出了伪平行句对 ($s_2$, $s_1$)，这时只做推断不更新模型参数；然后再用 shared encoder 编码 $s_2$，用 $L_1$ decoder 解码出 $s_1$，**这里通过最大化 P(s1|s2) 来训练模型（也就是 shared encoder 和 L1 decoder 的参数）**。

> Back-translation 是 Sennrich 15 年提出来的数据增广的技巧，详见论文 [1511.06709] Improving Neural Machine Translation Models with Monolingual Data 。具体做法是把单语语料用训好的机器学习模型翻译一遍做成伪平行语料，然后把这样的句对也当做训练数据来训模型，其实就是半监督学习的做法。叫 back-translation 是因为假如你增广语料的时候是把 s1 翻译成了 s2，那么训练的时候要用 s2 翻译出 s1 这个方向来训模型。其动机是，目标语言必须始终是真句子才能让翻译模型翻译的结果更流畅、更准确，而源语言即便有少量用词不当、语序不对、语法错误，只要不影响理解就无所谓。其实人做翻译的时候也是一样的：翻译质量取决于一个人译出语言的水平，而不是源语言的水平（源语言的水平只要足够看懂句子即可））

另外在训翻译模型的时候，词向量是固定住不变的，就用预训练得到的词向量。就通过这种神奇的方法，他竟然做到了还不错的翻译效果(BLEU>0.1)，并不是简单的逐词翻译！一些具体的例子比如（数字不准是 NMT 的老问题了）：
![](https://pic4.zhimg.com/50/v2-6928a6d3fd79bf22db9cccd6d2fc0136_hd.jpg)

# Remark

1. 整个模型相当于 <font style="color:Red">unsupervised cross-language embedding + denoising auto-encoder + dual learning</font> 合起来，但是效果这么好真是令人惊讶；

2. 其实这个不能算完全无监督，因为用了阿拉伯数字这个跨语言的 lexicon[词典]，<font style="color:Red">相当于利用了天然、廉价的对齐信息</font>；假设有一种语言不使用阿拉伯数字，那这篇文章的方法也行不通……当然，理论上说，其他无监督对齐词向量空间的方法也可以，比如说清华 ACL'17 那篇 <font style="color:Red">Adversarial training for unsupervised bilingual lexicon induction</font>，但是效果可能就要大很多折扣；

3. dual learning 那篇文章本来也想做完全无监督的 NMT，但是做不动，<font style="color:Red">最后还是用了少量平行语料</font>……这篇文章成功的原因一是利用了对齐的词向量空间（相当于凭空得到的先验），二是使用 shared embedding 减少参数，降低了学习难度；

4. 文章做的实验都是基于<font style="color:Red">英法或者英德的，都是比较接近的语言，在这些语言对之间共享 encoder 可能是合理的，因为可能存在一个统一的语义空间</font>；对于差异较大的语言，比如说中阿翻译，这么做可能是不行的。从 GNMT 的经验来看，多对一翻译是没问题的，因为翻译模型本质上是 conditional language model, which is a language model of the target language and conditions on the source language，多对一可以训练出一个更好的目标语言的语言模型；但是一对多经常会损害翻译质量，因为每个语言对对 encoder 提特征的方式的要求会有一些冲突。这篇文章却对两种语言使用同一个 shared encoder，在相差较大的语言上可能会难以学习；

5. 文章里 BPE 的效果不稳定，真是很神奇……在我做过的所有实验以及听到其他所有人的反馈里，BPE 都是妥妥地提升翻译质量好几个 BLEU 的；我觉得可能的原因是，不同语言形意关系不同……不同语言在单词级别往往可以对齐（假设交给人类双语者来对齐的话），但是 subword 仅仅是基于 char n-gram 的频率，其中有很多是没有语义的（比如说并非词根或者词缀），到底应该嵌入到 cross-lingual word embedding space 中的哪里呢？这一点其实不够明确；

6. 还有，我认为这篇文章的方法应该还是需要非常非常弱的监督信号的，就是需要假设两种语言的语料都是同一个领域的，至少是话题相近，不然可能很难构建统一的语义空间；

7. 再设想一下，假如我们有大量关于某外星语言的资料（类似七肢桶的语言那样），和人类语言的组织方式完全不同，甚至分词都没法分（不过这个可以靠 BPE 强撑，虽然效果不稳定），那就没法学了，因为两种语言的语义空间可能结构不一样，会导致词对齐不准，或者是 encoder 训练冲突。再假如说我们有大量某种古代失传的语言的资料，因为里面没有阿拉伯数字，所以在第一步生成 cross-lingual word embedding 的时候会遇到困难；另外由于古代文本描述的内容与今天的文本可能有较大差异（比如说古人没有电视没有火车没有议会，他们的生活跟我们今天的生活差太多了，有些概念可能是现代语言里不存在的，很难对齐到一个空间里），这一点也会导致学习困难。不过好在，人类破译一门古代语言一般都是从某一个子系统入手的，有一些关于语言的元信息，比如说破译玛雅语言就是先破译了它的历法和数字系统，又知道了文本的内容是古代帝王的生平，有了这些信息做初步的对齐就好办很多了。


# FAIR's work

**FAIR 的论文和这篇论文如出一辙：都是先训两个单语的词向量，然后对齐词向量空间，之后对齐 encoder 语义空间，两种语言各一个 decoder；用 denoising auto-encoder 训练单语语言模型，用 back-translation 造伪平行语料优化似然函数……** 个人感觉 FAIR 的文章更漂亮一些，因为它是真的完全无监督的，连阿拉伯数字这种天然跨语言的 token 都没用，并且工作的延续性更好，各个模块浑然一体。

主要区别点在于：

1. 使用 Word Translation Without Parallel Data 里的方法对齐两个词向量空间并导出初步的双语词表.
2. 具体做法是:
    - 用<font style="color:Red">对抗的方法训练一个生成器 G（原空间到目标空间的变换矩阵，要求正交），以及一个判别器 D（用来分辨一个给定的词向量是目标语言的还是源语言变换过来的），学好以后两个空间就对齐了</font>。
    - 在这之后可以继续 refine，就是找两种语言可以配对的高频词，以他们为种子，再次解析地求解两个空间之间的线性变换。
    - 这篇文章还有一个贡献就是提出了找配对词的指标，因为直接找最近邻可能有很多问题，比如说有些词孤零零的、有些词却是很多词的最近邻（这个问题叫 hubness），文章里的实验说明它提出的指标和 word translation accuracy 的相关性非常好，见下图（横轴是训练的 epoch 数，蓝线是 word translation accuracy，黑线是它提出的指标）：

![](https://pic4.zhimg.com/50/v2-46182692fc66d42609005ac9b9bfc7f7_hd.jpg)

从而可以用这个指标指示对抗训练什么时候停下来。论文最终展示的对齐效果拔群，跟有监督的方法效果差不多甚至更好（word translation 在有的语言对上能达到 70% 甚至 80% 的准确率）。而且对于字母表不同的、距离较遥远的语言，例如英汉，效果也非常好。模型结构图如下所示：
![](https://pic2.zhimg.com/50/v2-4910b992cb8e5002259e661c70dd75cb_hd.jpg)

- <font style="color:red">上半部分是 denoising auto-encoder，用于把一个 corrupted sentence 恢复出顺畅的源句子，corrupt 的方法是局部交换和扔掉个别词。</font>
- 下半部分是<font style="color:red">翻译模型，先用前一代的模型（蓝色模块）把语言 L1 的句子 s1 翻译成语言 L2 的句子 s2，然后优化 back-translation 的似然 P(s2|s1)</font>。

这里和 Cho 他们的模型是一样的。主要区别在于：<font style="color:red">Cho 的文章直接让两种语言共用一个 encoder；而这篇文章是两种语言各有各的 encoder，但是用一个判别器分辨 encoder 编码的结果来自哪种语言，通过对抗的方法让两个 encoder 编码的语义空间重合到一块儿去（和前面对齐词向量的方法一脉相承，工作的延续性更好一些，科科。而 Cho 那个拼接感有些强）</font>。

这一篇的词向量用步骤一中的词向量初始化，但是之后还会继续训练。而 Cho 的文章中则固定住了词向量。

在训练的时候，Cho 的文章是一步 denoise 一步 back-translation 交替训练，每次只训一个方向；而这篇是两个翻译方向的图中所有 loss 加起来训一个 batch，然后判别器 D 训一个 batch，这样交错下去。

3. 用生成初始翻译：

最后一点区别就是，Cho 那篇是从头开始训模型的；而 FAIR 这篇是用 word-by-word translation 生成第一轮时的初始翻译（其实这时候效果已经不错了，因为这篇文章用的词向量对齐质量很高，单是这种翻译 BLEU 就能达到 5 个点）。所以按照我的理解，FAIR 这篇第一轮迭代就能在比较靠谱的伪平行句子上进行了，而 Cho 那篇可能需要从一张白纸学起，训练初期的翻译结果估计不太能看……不过好在它固定了词向量，应该可以引导 shared encoder 快速构建语义空间。

所以，这两篇文章对于 cross-lingual embedding 的用法是不一样的。Cho 那篇仅仅是用来对齐两个空间；而这篇既用来对齐空间，还用来导出 translated lexicon，加速模型收敛。

4. 收敛条件

Cho 那篇没详细说，估计可能就是看损失函数吧。而 FAIR 这篇说，可以让 s1 翻译到 s2 再翻译回 s1'，计算 s1 和 s1' 的 BLEU 来做交叉验证，决定训练什么时候停止，并且说明这个指标跟真正平行语料上测出来的 BLEU 相关性很高（这里又能看到词向量对齐那篇提出的无监督交叉验证指标的影子）。如下图所示：
![](https://pic3.zhimg.com/50/v2-7c142e09be353426136c682737bb7619_hd.jpg)

不过看这个图的话，感觉其实没有多大必要引入这个指标……真正的 BLEU 看起来也是平稳收敛的，没出现拨动很大的情况，训得差不多了就可以停了，交叉验证意义不大。不像词对齐那个，不同 epoch 对齐准确率能从 80% 抖到 20%。
