# Natural Speech 系列

## NaturalSpeech 1

## NaturalSpeech 2

## NaturalSpeech 3

NaturalSpeech 3 于 2024 年 03 月 05 日放出预印论文, 后经过两次修改.

尽管近期大规模文本转语音模型已经获得了重要进展, 但在音频质量, 相似性, 韵律上都仍表现得不够好.
由于音频包含多种复杂属性 (如内容, 韵律, 音色和其他声学细节) 使得生成面临着重大挑战, 一个自然的想法是将音频分解到单独的子空间以分别表示不同属性, 然后分别生成它们.
基于此, 以微软研究院领衔的研究团队提出了 NaturalSpeech 3, 一个基于新式分解扩散模型的文本转语音系统, 能够以零样本的方式生成自然的语音.
具体地, 有以下两点设计:
- 结合**分解向量量化 (Factorized Vector Quantization, FVQ)** 的神经编解码器将语音波形解耦到内容, 韵律, 音色和声学细节等子空间中;
- 提出分解扩散模型在每个子空间中按照对应提示词生成属性.

结合这种分解设计, NaturalSpeech 3 能够有效且高效地以分而治之的思路在解耦子空间中建模复杂音频.
实验证明 NaturalSpeech 3 超过了现有语音合成系统在音频质量, 相似性, 韵律和智能的表现, 并且达到了和人类录制相当的质量.
此外通过缩放到十亿参数和二十万小时的训练数据可以获得更好的性能.

注: 对比算法有 VALL-E, NaturalSpeech 2, Voicebox, Mega-TTS 2, UniAudio, StyleTTS 2, HierSpeech++.

![](../../Models/Diffusion/Images/2024.03.05_NaturalSpeech3.Fig.01a.png)

整体的结构如下:

NaturalSpeech 3 由神经语音编解码器 FACodec 和分解扩散模型组成.
FACodec 将原始波形分解出五种属性: 时长, 韵律, 内容, 声学细节和音色.
需要注意的是虽然时长可以作为韵律的一个方面, 但为了能非自回归式生成语音, 仍然选择显式建模它.
采用内部对齐工具来对齐语音和音素并获得音素级别的时长.
对于其他的属性, 则通过使用分解神经编解码器隐式地学习对应的属性子空间.
然后使用分解扩散模型用于生成每个语音属性表示.
最后应用编解码器的解码器结合生成的语音属性重建音频.

下面分别介绍两个部分:

### FACodec 

源代码于 2024 年 03 月 12 日开源于 [Github/Amphion](https://github.com/open-mmlab/Amphion/tree/main/models/codec/ns3_codec)

![](../../Models/Diffusion/Images/2024.03.05_NaturalSpeech3.Fig.02.png)

FACodec 由一个语音编码器, 音色提取器, 三个分别用于内容, 韵律, 声学细节的分解向量量化器 (FVQ), 语音解码器组成.
那么给定一个 16kHz 的语音数据 $x$,
1. 语音编码器中采用数个卷积块, 对输入进行下采样, 采样率为 200, 以获得量化前的隐变量 $h$;
2. 音色提取器采用 Transformer 编码器将 $h$ 转换成表示音色属性的全局向量 $h_t$;
3. 对于其他三个属性, 分别使用一个分解向量量化器 (FVQ) 捕获更细粒度的语音属性表示, 并获得对应的离散标识符 (Token);
4. 语音解码器采用语音编码器的镜像结构, 但使用更多的参数以确保高质量的语音重建. 首先将韵律, 内容和声学细节的表示相加然后使用条件层归一化融合音色信息以获得解码器所需的表示 $z$

上述过程的关键在于如何能够更好地解耦语音属性.

直接将语音分解到不同的子空间中并不能保证语音成功解耦. 为此采用了一些技术以获得更好的属性解耦效果.
- 信息瓶颈: 强制模型移除不必要的信息 (如内容子空间中的韵律信息). 具体的构造方式是在三个 FVQ 中将编码器输出映射到低维空间 (8 维) 然后在这个低维空间内进行量化. 这一技术能够确保每个编码嵌入包含更少的信息, 促进信息解耦. 在量化之后将这些量化向量映射回原始维度.
- 监督学习: 为每个属性引入监督任务作为辅助任务以获得高质量的语音解耦. 
  - 对于韵律, 音高是一个重要部分, 所以使用量化后的隐变量 $z_p$ 来预测音高信息. 
  - 对每帧提取 F0 特征然后使用归一化 F0 (z score) 作为目标; 
  - 对于内容, 直接使用音素标签作为目标 (使用内部对齐工具获得帧级别的音素标签); 
  - 对于音色, 对全局音色表示 $h_t$ 应用说话人分类来预测说话人 ID.
- 梯度反转: 防止信息泄露 (如韵律泄露到内容中) 能够增强解耦能力. 使用带梯度反转层 (Gradient Reversal Layer, GRL) 的对抗分类器在隐空间中消除不想要的信息.
  - 对于韵律, 应用音素 GRL (即预测音素标签的 GRL 层) 来消除内容信息;
  - 对于内容, 因为音高是韵律的一个重要方面, 所以简单地应用 F0-GRL 来减少韵律信息;
  - 对于声学细节, 应用音素 GRL + F0 GRL 来消除内容和韵律信息;
  - 对三者输出的和应用 Speaker-GRL 来消除音色信息.
- 细节失活: 
  - 经验上发现编解码器倾向于在声学细节空间中保留不想要的信息 (如内容韵律) 因为没有应用监督任务. 
  - 直觉上没有声学细节的话解码器依靠韵律内容和音色应该可以重建音频, 即便是低质量.
  根据以上两点设计了细节失活, 即在训练时对声学细节部分的输出按概率 $p$ 进行随机失活, 以获得解耦和重建质量的权衡.
  1. 编解码器能够充分利用韵律, 内容和音色信息来重建音频, 以确保解耦能力, 即便是低质量;
  2. 可以给定声学细节以获得高质量音频.

### Factorized Diffusion Model

![](../../Models/Diffusion/Images/2024.03.05_NaturalSpeech3.Fig.03.png)

采用离散扩散来生成语音以获得更高的质量.
具体有以下考虑:
- 将语音分解为时长, 韵律, 内容, 声学细节等属性, 然后用具体条件顺序地生成它们.
  首先前面提过由于非自回归生成设计, 首先生成时长;
  然后直觉上声学细节应该最后生成.
- 遵循语音分解的设计, 只为对应属性的提示词应用生成模型, 并在子空间中应用离散扩散.
- 为了促进扩散模型内的上下文学习, 使用编解码器将语音提示分解为属性提示并采用部分噪音化机制生成目标语音属性. 例如为了生成韵律, 直接将无噪声的韵律提示和带噪声的目标序列进行拼接, 然后逐渐从目标序列中移除噪声.

结合这些想法, 设计了分解扩散模型, 包含了音素编码器和四个具有相同离散扩散形式的语音属性扩散模块.
1. 结合时长提示和由音素编码器输出的音素级别的文本条件, 应用时长扩散以生成语音时长. 然后应用长度调节器以获得帧级别的音素条件 $c_{ph}$;
2. 结合韵律提示和音素条件 $c_{ph}$ 生成韵律 $z_{p}$;
3. 结合内容提示, 生成韵律 $z_{p}$, 音素条件 $c_{ph}$ 为条件生成内容韵律 $z_{c}$;
4. 结合声学细节提示, 生成韵律 $z_{p}$, 音素条件 $c_{ph}$ 和内容韵律 $z_{c}$ 生成声学细节 $z_{d}$;
5. 不显式生成音色属性, 而是直接从提示直接生成音色 (对应 FACodec 的设计).
6. 最后将属性 $z_p, z_c, z_d, h_t$ 结合起来并用编解码器的解码器进行解码.

下面详细介绍扩散形式.

#### 前向过程

$X = [x_{i}]_{i=1}^{N}$ 为目标离散标识符序列, 其中 $N$ 是序列长度;
$X^{p}$ 是提示词离散标识符序列;
$C$ 为条件.

在时间步 $t$ 的前向过程定义为对 $X$ 的子集用相应的二元掩膜 $M_t=[m_{t,i}]_{i=1}^{N}$ 进行遮盖, 即 $X_t=X\odot M_{t}$, 具体是当 $m_{t,i}=1$ 时将 $x_{i}$ 替换为 `[MASK]` 标识符, 否则当 $m_{t,i}=0$ 时保持 $x_{i}$ 不变, $m_{t,i}$ 独立同分布于伯努利分布 $\text{Bernoulli}(\sigma(t))$, $\sigma(t)\in (0,1]$ 是一个单调递减函数.

本文采用 $\sigma(t)=\sin((\pi t)/ (2 T))$.
特别地 $X_0=X$ 为原始标识符序列, 而 $X_{T}$ 为全掩膜序列.

#### 反向过程

反向过程通过从反向分布 $q(X_{t-\Delta t}|X_0, X_t)$ 采样来逐渐从全掩膜序列 $X_T$ 恢复为 $X_0$.
因为 $X_0$ 在推理时不可用, 所以使用由参数 $\theta$ 参数化的扩散模型 $p_{\theta}$, 基于条件 $X^{p}$ 和 $C$ 预测掩膜标识符, 记为 $p_{\theta}(X_0|X_t, X^p, C)$.
优化目标是掩膜标识符的负对数似然:

$$
    Loss_{mask} = E_{X\in \mathcal{D}, t\in [0,T]} \left[-\sum_{i=1}^N m_{t,i}\cdot \log (p_{\theta}(x_i|X_t, X^p, C))\right]
$$

然后就能够获得反向转移分布:

$$
    p(X_{t-\Delta t|X_0, X^p, C}) = E_{\hat{x}_0\sim p_{\theta}} q(X_{t-\Delta t|\hat{X}_0, X_t})
$$

#### 推理部分

在推理时, 逐渐从全掩膜序列 $X_T$ 替换被掩膜的标识符.
首先从 $p_{\theta}(X_0|X_t, X^p, C)$ 采样 $X'_0$;
然后从 $q_{X_{t-\Delta t}|X'_0, X_t}$ 采样 $X_{t-\Delta t}$, 涉及到对 $X'_0$ 中具有最低置信分数的 $\lfloor N\sigma(t-\Delta t)\rfloor$ 个标识符进行重掩膜. 其中置信分数定义为 $m_{t,i}=1$ 时 $p_{\theta}$, 否则为 1, 即标识符已经去掩膜, 无需重掩膜.

#### 无分类器指导

此外, 采用无分类器指导技术.
具体地, 在训练时不使用概率 $p_{cfg}=0.15$ 的提示词;
在推理时, 将模型的输出向基于提示 $g_{cond}=g(X|X^p)$ 的方向外推, 并远离无条件生成 $g_{uncond}=g(X)$, 即

$$
  g_{cfg} = g_{cond} + \alpha (g_{cond}-g_{uncond})
$$

指导缩放系数 $\alpha$ 基于实验结果选择.
然后通过 $g_{final}=std(g_{cond})\times g_{cfg} / std(g_{cfg})$ 进行重缩放.

### 和 NaturalSpeech 系列比较

NaturalSpeech3 和之前的 NaturalSpeech 工作相比, 有以下联系和不同:
- **目标**: NaturalSpeech 系列的目标是生成具有高质量和多样性的自然语音. 
  我们采用多阶段的方式解决:
  - 在单说话人场景下获得高质量的语音合成. 如 NaturalSpeech;
  - 在多风格多说话人多语种场景下获得高质量和多样性的语音合成. 如 NaturalSpeech 2 通过基于大规模多说话人数据集来探索零样本合成能力.
  - 在多说话人数据集上达到人类级别的自然度. 即 NaturalSpeech 3.
- **架构**: NaturalSpeech 系列共享的基本组件有编码器解码器用于波形重构, 时长预测用于非自回归语音生成.
  - NaturalSpeech 使用基于流模型的生成模型;
  - NaturalSpeech2 使用潜在扩散模型;
  - NaturalSpeech3 提出分解扩散模型以分而治之的方式生成每个被分解的语音属性.
- **语音表示**: 由于语音波形的复杂性, NaturalSpeech 系列使用编码器解码器用于获得音频隐变量以便进行高质量语音合成.
  - NaturalSpeech 使用基于 VAE 的连续表示;
  - NaturalSpeech2 使用基于残差向量量化器的神经编解码器得到的连续表示;
  - NaturalSpeech3 使用 FACodec 将语音信号转换到解耦子空间中, 并减少了语音建模复杂度.