# 通过循环语言模型扩展潜在推理

Rui-Jie Zhu\*<sup>1,2,\*</sup>, Zixuan Wang\*<sup>1,3</sup>, Kai Hua\*<sup>1</sup>, Tianyu Zhang\*<sup>4,5</sup>, Ziniu Li\*<sup>1</sup>, Haoran Que\*<sup>1,6</sup>, Boyi Wei\*<sup>3</sup>, Zixin Wen\*<sup>1,7</sup>, Fan Yin\*<sup>1</sup>, He Xing\*<sup>11</sup>, Lu Li<sup>8</sup>, Jiajun Shi<sup>1</sup>, Kaijing Ma<sup>1</sup>, Shanda Li<sup>1,7</sup>, Taylor Kergan<sup>2,9</sup>, Andrew Smith<sup>2,9</sup>, Xingwei Qu<sup>1,10</sup>, Mude Hui<sup>2</sup>, Bohong Wu<sup>1</sup>, Qiyang Min<sup>1</sup>, Hongzhi Huang<sup>1</sup>, Xun Zhou<sup>1</sup>, Wei Ye<sup>6</sup>, Jiaheng Liu<sup>11</sup>, Jian Yang<sup>11</sup>, Yunfeng Shi<sup>11</sup>, Chenghua Lin<sup>10</sup>, Enduo Zhao<sup>11</sup>, Tianle Cai<sup>1</sup>, Ge Zhang\*<sup>1,\*</sup>, Wenhao Huang<sup>1,\*</sup>, Yoshua Bengio<sup>4,5</sup>, Jason Eshraghian<sup>2,\*</sup>

<sup>1</sup>ByteDance Seed, <sup>2</sup>UC Santa Cruz, <sup>3</sup>Princeton University, <sup>4</sup>Mila - Quebec AI Institute, <sup>5</sup>University of Montreal, <sup>6</sup>Peking University, <sup>7</sup>Carnegie Mellon University, <sup>8</sup>University of Pennsylvania, <sup>9</sup>Conscium, <sup>10</sup>University of Manchester, <sup>11</sup>M-A-P

*核心贡献者， †通讯作者

# 摘要

现代大语言模型主要通过显式文本生成（如链式思维CoT）来进行"思考"，这将推理推迟到后训练阶段，且未能充分利用预训练数据。我们提出并开源了Ouro（以递归的衔尾蛇Ouroboros命名），这是一系列预训练的循环语言模型（LoopLM），通过以下方式将推理构建到预训练阶段：（i）潜在空间中的迭代计算；（ii）用于学习深度分配的熵正则化目标；（iii）扩展至7.7T tokens。Ouro 1.4B和2.6B模型具有卓越的性能，在广泛基准测试中匹配高达12B参数的SOTA大语言模型。通过对照实验，我们表明这种优势并非来自知识容量的增加，而是来自卓越的知识操作能力。我们还表明，LoopLM产生的推理痕迹比显式CoT更与最终输出对齐。我们希望我们的研究结果能够展示LoopLM作为推理时代新型扩展方向的潜力。

通讯地址：ridger@ucsc.edu, zhangge.eli@bytedance.com, huang.wenhao@bytedance.com, jsn@ucsc.edu  
项目页面与基础/推理模型：http://ouro-llm.github.io

![图片 page1-12](images/page001_img01.png)

![图片 page1-13](images/page001_img02.png)

![图片 page1-14](images/page001_img03.png)

_图1 Ouro循环语言模型性能。（左）参数共享的循环架构。（中\&右）雷达图比较Ouro 1.4B和2.6B模型（均为4个循环步骤，红色）与各transformer基线。我们的模型展现出与更大基线相当或更优的强劲性能。_

---

# 1 引言

大语言模型（LLM）的发展历来依赖于将模型规模扩大作为主要驱动力，同时伴随着数据和算力的增加[1-4]。然而，部署具有数千亿参数的模型需要庞大的基础设施，增加了延迟和成本，同时限制了可访问性。这些因素使参数效率变得至关重要：在固定参数预算内实现更好的模型能力。这样的模型不仅可以用更少的可训练参数减少在有限数据集上的过拟合，还能以更轻量级的基础设施实现更实际的部署。为了实现这种参数效率，人们探索了两条主要途径。第一条是扩大训练语料库而不增加模型规模[5]，尽管数据稀缺日益限制了这一路径。第二条是通过链式思维（CoT）推理[6]利用推理时计算，允许模型通过扩展token生成在复杂问题上投入更多计算。

我们探索第三条基于架构创新的途径：在固定参数预算内实现动态计算。这通过递归应用共享参数来实现，其中一组权重绑定的层在前向传播过程中被迭代重用。我们称之为循环语言模型（LoopLM）。该设计具有几个优势。首先，LoopLM通过学习的早退出机制实现自适应计算：简单输入可以在更少的循环步骤后终止，而复杂输入则分配更多迭代。这将计算深度与参数数量解耦。其次，与CoT等推理时方法不同，LoopLM通过深化内部计算图而非扩展输出序列来扩展，避免了上下文长度的膨胀。最后，当在相同数据上训练时，LoopLM可以提高每参数的容量并超越更大规模的标准transformer。

大量先前研究已在适度规模上探索了LoopLM[7-14]，从开创性的Universal Transformer[15]到递归Transformer[16]和潜在推理方法[17-19]。然而，循环语言模型是否能转化为实际有意义规模的前沿级收益尚未得到验证。为此，我们提出：

# LoopLM相比非递归transformer模型是否表现出更有利的扩展行为（能力、效率和安全性）？

我们证明答案是肯定的。我们描述了LoopLM的扩展轨迹和饱和行为，表明LoopLM提供了更高效的高性能路径。这些claims在典型的SOTA基础模型的多万亿token训练制度下进行评估，远超先前工作。除了实证收益外，我们还通过以下问题分析这些改进背后的机制：

1. 权重的递归重用是否产生了通过增加非共享权重深度通常获得的相同能力提升？

2. LoopLM的收益是否随循环次数单调变化？影响这一点的因素有哪些？

# 我们的贡献

我们通过多方面研究来回答上述问题。我们将LoopLM预训练扩展至7.7T tokens，并彻底研究其多轴扩展行为。为实现自适应计算，我们引入训练目标，在保持峰值性能的同时实现计算高效的递归。我们还进行对照消融以隔离LoopLM收益的来源。具体而言，我们的贡献包括：

- 大规模下的卓越参数效率。通过在7.7T tokens上预训练，我们证明1.4B和2.6B参数的LoopLM在大多数基准测试上匹配4B和8B标准transformer，产生对资源受限环境中部署至关重要的2-3倍参数效率收益（图1和图2）。

- 熵正则化自适应计算。自适应退出往往崩溃至浅深度或过度使用长循环。我们通过在统一先验下进行熵正则化来避免这种情况，以实现无偏深度探索，随后进行专注于调优计算-性能权衡的第二训练阶段，并根据输入难度分配步骤。

![图片 page3-0](images/page003_img01.png)

![图片 page3-1](images/page003_img02.png)

![图片 page3-2](images/page003_img03.png)

![图片 page3-3](images/page003_img04.png)

![图片 page3-4](images/page003_img05.png)

![图片 page3-5](images/page003_img06.png)

![图片 page3-6](images/page003_img07.png)

![图片 page3-7](images/page003_img08.png)

_图2 高级推理基准测试性能。Ouro-Thinking模型与Qwen3和DeepSeek-Distill等强基线进行比较。Ouro-1.4B-Thinking R4与4B模型具有竞争力，Ouro-2.6B-Thinking R4在多个数学和科学数据集上匹配或超越8B模型。_

- 递归的机制理解。使用受语言模型物理学框架启发的对照实验，我们发现递归并未增加原始知识存储（循环和非循环模型均约为每参数2 bits），但显著增强了需要事实合成和多跳推理的任务的知识操作能力。

- 改进的安全性和可信性。LoopLM降低了HEx-PHI[20]上的有害性，安全性随递归步骤增加而提升（包括外推步骤）。与CoT相比，我们的迭代潜在更新产生的推理痕迹比最终输出更好对齐，表明更大的因果可信性而非事后合理化。

我们的研究建立了循环深度作为模型规模和数据之外的第三条扩展轴，我们公开发布了Ouro模型系列（1.4B和2.6B参数）以展示LoopLM在大规模下的优势。

# 2 相关工作

该架构的核心思想在近期文献中重新出现，递归深度结构被用于提高现代LLM的效率和推理能力。例如，Geiping等人[17]采用"递归深度"来扩展潜在空间的测试时计算。类似地，Saunshi等人[7]证明"循环transformer"可以在推理任务上匹配更深非循环模型的性能，正式将循环与潜在思想生成联系起来。该方法通过将标准模型转换为具有公共基础块的"松弛递归Transformer"同时在递归步骤间注入独特的LoRA适配器来改进[16]。类似概念也以不同术语出现，如连续空间中的"思考"[18]和用于自适应计算的"内部思考"[21]。更先进的变体如递归混合[22]将递归参数效率与自适应、token级路由相结合。

在所有这些工作中，从原始Universal Transformer到其现代后裔，这种新兴架构系列可以从两个互补的角度理解。从一个角度来看，它表现得像一个深层Transformer，其中所有层的权重都是绑定的。从另一个角度来看，迭代充当潜在推理，隐藏状态形成潜在思维链，逐步细化表示以解决问题。综合这些结果表明，模型可以通过内部重用计算来提高推理能力，而无需增加参数数量，将规模转向实质。

视角1：参数共享以提高模型效率。该观点将LoopLM视为参数共享：一个或多个Transformer块，甚至子模块（如注意力、FFN），在模型深度上被重用，减少参数而不改变计算。现代transformer时代最突出的例子是ALBERT[23]，它结合参数重用与嵌入分解以大幅减少总参数计数。在LLM广泛采用之前，参数共享在机器翻译[24]中被广泛探索；Takase等人[25]系统研究了平衡压缩和精度的共享策略。随着模型变得更大，对参数重用的兴趣下降，但它已重新兴起以缩小LLM的内存占用。例如，Megrez2[26]在标准专家混合（MoE）模型中跨层重用专家，为内存受限的边缘LLM部署展示了一条可行路径。

视角2：潜在推理与迭代细化。这里，LoopLM的迭代被视为潜在推理，其中每个步骤都是细化模型内部表示的非言语"思想"。实证上，增加递归步骤数可提高复杂推理任务的性能[7,17]。一些模型通过将隐藏状态反馈到输入中使这一过程显式化。Coconut插入一个"连续思想"token，源自前一步的最后一层隐藏状态，使模型能够在连续潜在空间"思考"[27]。CoTFormer将激活交错回到输入中，然后将这些增强序列重新应用到共享层[28]。这些显式反馈循环与隐式LoopLM变体形成对比，后者整个思考过程包含在前一步到当前步的隐藏状态演化中。因此，视角1（模型压缩）和视角2（潜在推理）都利用共享参数迭代来提高参数效率，并正在被探索以增强推理和高效序列长度扩展（如PHD-Transformer[29]）。

---

# 3 使用LoopLM学习自适应潜在推理

在本节中，我们正式定义因果Transformer上的LoopLM架构，并展示我们的自适应潜在推理训练方案。图3展示训练和推理时的架构。我们的目标是让模型为每个token和每个样本选择递归步骤数，在简单输入上花费更少计算，在困难输入上花费更多计算，同时不牺牲可用多步骤时的准确性。

# 3.1 LoopLM架构

设$\mathrm{emb}(\cdot):\mathbb{R}^{|V|}\to \mathbb{R}^d$为token嵌入；$\mathcal{T}_{\theta}(\cdot):\mathbb{R}^{M\times d}\to \mathbb{R}^{M\times d}$为由$\theta$参数化的因果transformer层，具有隐藏大小$d$和输入长度$M$，以及$\mathrm{lmhead}(\cdot):\mathbb{R}^d\to \mathbb{R}^{|V|}$为词汇大小为$V$的解嵌入层。非循环LM堆叠$L$层，其中$\circ$表示函数组合：

$$
\[
F (\cdot) := \mathrm {l m h e a d} \circ \mathcal {M} ^ {L} \circ \mathrm {e m b} (\cdot), \qquad \mathcal {M} ^ {L} (\cdot) := \mathcal {T} _ {\theta_ {L}} \circ \dots \circ \mathcal {T} _ {\theta_ {1}} (\cdot)
\]
$$

设$t \in \{1, \dots, T_{\max}\}$为循环步骤数（递归步骤数或递归深度）。循环模型$F^{(t)}$重用深度为$L$的相同层堆栈$t$次：

$$
\[
F ^ {(t)} (\cdot) = \operatorname {l m h e a d} \circ \underbrace {\mathcal {M} ^ {L} \circ \mathcal {M} ^ {L} \circ \cdots \circ \mathcal {M} ^ {L}} _ {t \text {i t e r a t i o n s}} \circ \operatorname {e m b} (\cdot). \tag {1}
\]
$$

因此，$t = 1$产生非循环模型$F^{(1)} \equiv F$。如图3（左）所示，在每个递归步骤$t$，模型产生语言建模头输出。我们将单步$t$的标准交叉熵损失定义为损失$\mathcal{L}^{(t)}$：

$$
\mathcal {L} ^ {(t)} = \mathbb {E} _ {x _ {1: M}} \left[ \sum_ {\ell = 1} ^ {M - 1} - \log p _ {\theta} ^ {(t)} \big (x _ {\ell + 1}\mid x _ {1: \ell} \big) \right], \tag {2}
$$

其中$p_{\theta}^{(t)}(\cdot \mid x_{1:\ell}) = \mathrm{softmax}\bigl (\mathrm{lmhead}(h_{\ell}^{(t)})\bigr)$，$x_{1:\ell}$表示输入的长度为$\ell$的前缀（token 1到ell），$h_\ell^{(t)}$是位置$\ell$在$t$次循环后的隐藏状态。注意这是单个递归步骤的单独损失。结合所有步骤的总训练目标在以下章节中定义。

先前文献[7,11]已表明扩展$t$对推理任务有益。然而，这增加了计算，并非所有token都需要许多步骤[30,31]。因此，将计算预算花在正确的token上至关重要。这通过下一节描述的门控机制实现。

# 3.2 通过门控机制实现自适应计算

为实现自适应计算，我们添加了一个退出门，与LM头在每个步骤$t \leq T_{\max}$并行运行（图3）。在每个循环$t$，门输出瞬时（每步）退出概率

$$
\[
\lambda_ {t} (x) = \sigma \left(\mathrm {L i n e a r} _ {\phi} \left(h ^ {(t)}\right)\right) \in (0, 1)
\]
$$

其中$h^{(t)}$是步骤$t$的最终层隐藏状态，$\phi$是门参数。我们定义

$$
\[
S _ {t} (x) = \prod_ {j = 1} ^ {t} \bigl (1 - \lambda_ {j} (x) \bigr), \qquad S _ {0} (x) \equiv 1,
$$
$$
为生存概率，或不在前$t$步中退出的概率。则在步骤$t$首次退出的未归一化概率为
$$
\[
\tilde {p} _ {t} (x) = \lambda_ {t} (x) S _ {t - 1} (x), \qquad t = 1, \ldots , T _ {\max } - 1.
$$

$$

为获得退出步骤上的有效离散分布，我们将剩余质量分配给最终步骤：

$$
\[
p _ {\phi} (t \mid x) = \left\{ \begin{array}{l l} \tilde {p} _ {t} (x), & t = 1, \dots , T _ {\max } - 1, \\ S _ {T _ {\max } - 1} (x), & t = T _ {\max }, \end{array} \right. \quad \text {s o t h a t} \quad \sum_ {t = 1} ^ {T _ {\max }} p _ {\phi} (t \mid x) = 1. \tag {3}
$$
$$
带早退的推理。如图3（右）所示，我们从学习的退出分布$\{p_{\phi}(t \mid x)\}_{t=1}^{T_{\max}}$中推断退出步骤，实现高效推理。步骤$n$的累积退出概率为：
$$
\[
\operatorname {C D F} (n \mid x) = \sum_ {t = 1} ^ {n} p _ {\phi} (t \mid x) = 1 - \prod_ {j = 1} ^ {n} \bigl (1 - \lambda_ {j} (x) \bigr), \quad n < T _ {\max}, \qquad \operatorname {C D F} (T _ {\max} \mid x) = 1.
\]
$$
给定阈值$q \in [0,1]$，我们在累积概率越过$q$的第一个步骤终止：
$$
\[
t _ {\mathrm {e x i t}} (x) = \min \{m \in \{1, \ldots , T _ {\max }\} : \operatorname {C D F} (m \mid x) \geq q \}.
$$

$$

阈值$q$控制计算-准确性权衡：较小的$q$有利于更早退出（更少计算），而较大的$q$允许更深度计算。实际上，$q$可以全局选择、按任务校准或与步骤的floor/ceiling一起调度。这种确定性、基于分位数的策略避免了采样，同时与学习分布保持一致。

门控参数$\phi$（从而通过$\{\lambda_t\}$得到$p_{\phi}$）通过两个阶段学习：

- **阶段1：** 在预训练期间，门与LM通过优化熵正则化目标联合学习（第3.3节）。

- **阶段2：** 我们冻结LM并微调$\phi$以锐化$p_{\phi}$（即调整深度分配）而不改变token级预测。

下一节描述完整的训练目标。

# 3.3 阶段1：学习熵正则化目标

在next-token预测损失的朴素梯度下降下，更深的循环通常在某个深度之前降低单步损失$\mathcal{L}^{(t)}$（公式2）；超过该深度后，收益减少，梯度将概率质量转向后期步骤。当$p_{\phi}$集中在后期步骤时，那些步骤收到更多训练信号，其损失进一步下降，这又吸引更多质量到末端。这种自我强化将$p_{\phi}$崩溃至$t = T_{\mathrm{max}}$。熵项惩罚向最深层步骤的崩溃，维持$p_{\phi}$中足够的分散以反映输入难度。

给定单步损失$\mathcal{L}^{(t)}$和退出步骤分布$p_{\phi}(t \mid x)$（公式3），我们的训练目标结合next-token预测与熵正则化：

$$
\[
\mathcal {L} = \underbrace {\sum_ {t = 1} ^ {T _ {\mathrm {m a x}}} p _ {\phi} (t \mid x) \mathcal {L} ^ {(t)}} _ {\mathrm {e x p e c t e d t a s k l o s s}} - \underbrace {\beta H (p _ {\phi} (\cdot \mid x))} _ {\mathrm {e n t r o p y r e g u l a r i z a t i o n}}, \qquad H (p _ {\phi} (\cdot \mid x)) = - \sum_ {t = 1} ^ {T _ {\mathrm {m a x}}} p _ {\phi} (t \mid x) \log p _ {\phi} (t \mid x). \qquad (4)
\]
$$

直观上，期望任务损失按退出步骤概率加权每个$\mathcal{L}^{(t)}$。系数$\beta$控制探索-利用权衡：较大的$\beta$鼓励更高熵（更探索性）的$p_{\phi}$，而较小的$\beta$让$p_{\phi}(t \mid x)$将其大部分质量放在模型对最优深度有信心的特定步骤上。

替代视角：带统一先验的变分推理。公式（4）中的目标可以视为证据下界（ELBO）损失，其中退出步骤$z \in \{1, \dots, T_{\max}\}$是潜在变量，其变分后验是学习的退出分布$p_{\phi}(z = t \mid x)$，其先验是$\pi(t)$。负ELBO为：

$$
\[
\mathcal {L} _ {\mathrm {E L B O}} = \sum_ {t = 1} ^ {T _ {\max }} p _ {\phi} (t \mid x) \mathcal {L} ^ {(t)} + \beta \mathrm {K L} \big (p _ {\phi} (\cdot \mid x) \| \pi (\cdot) \big).
$$
$$
对于统一先验$\pi_t = 1 / T_{\mathrm{max}}$，KL变为
$$
\[
\mathrm {K L} \big (p _ {\phi} (\cdot | x) \| \pi \big) = - H (p _ {\phi} (\cdot | x)) + \log T _ {\max},
$$

$$

因此最小化ELBO等价于（相差常数$\log T_{\mathrm{max}}$）公式（4）中的目标。这将熵项识别为KL正则化，并澄清期望损失在退出步骤上边缘化，同时也与PonderNet[32]等自适应计算方法联系起来，后者也优化动态停止的ELBO。

为什么是统一先验？不同先验编码不同深度偏好。几何先验（如参考文献[32]）或泊松-对数正态先验柔和地偏好更早停止[17]，而统一先验是无偏深的。我们采用统一先验以将退出决策与输入难度驱动的任何全局计算偏好分离；熵项则防止崩溃至总是使用$T_{\mathrm{max}}$。与几何先验的实证比较见附录A节。

# 3.4 阶段2：专注自适应门训练

在这个阶段，我们冻结LM参数，只训练退出门以基于实现的性能增益做出终止决策。我们使用平衡额外循环边际改善与额外计算的贪婪信号。

为确保门不改变LM表示，我们在每个token$i$和每个步骤$t$计算分离的每步损失$\mathcal{L}_{i,\mathrm{stop}}^{(t)}$，并定义从步骤$t-1$到$t$的损失改善为

$$
\[
I _ {i} ^ {(t)} = \max \left(0, \mathcal {L} _ {i, \mathrm {s t o p}} ^ {(t - 1)} - \mathcal {L} _ {i, \mathrm {s t o p}} ^ {(t)}\right) \tag {5}
$$
$$
其中较大的$I_{i}^{(t)}$表示持续改善；较小的值表示收益已停滞，LoopLM应选择早退。我们通过计算理想继续概率来实现这一点，这是一个训练标签，指示继续（接近1）还是退出（接近0）：
$$
\[
w _ {i} ^ {(t)} = \sigma (k \cdot (I _ {i} ^ {(t)} - \gamma))
$$

$$

斜率$k = 50.0$，阈值$\gamma = 0.005$，因此$w_{i}^{(t)} \approx 1$建议继续，$w_{i}^{(t)} \approx 0$建议退出循环。步骤$t$的自适应退出损失取门预测的继续概率$1 - \lambda_{i}^{(t)}$与理想标签$w_{i}^{(t)}$之间的二元交叉熵，在序列长度$M$上平均：

$$
\[
\mathcal {L} _ {\text {a d a p t i v e}} ^ {(t)} = - \frac {1}{M} \sum_ {i = 1} ^ {M} \left[ w _ {i} ^ {(t)} \log \left(\underbrace {1 - \lambda_ {i} ^ {(t)}} _ {\text {p r e d i c t e d}}\right) + \left(1 - w _ {i} ^ {(t)}\right) \log \left(\underbrace {\lambda_ {i} ^ {(t)}} _ {\text {p r e d i c t e d}}\right) \right]. \tag {6}
$$
$$
总自适应损失跨递归步骤平均：
$$
\[
\mathcal {L} _ {\mathrm {a d a p t i v e}} = \frac {1}{T _ {\mathrm {m a x}}} \sum_ {t = 2} ^ {T _ {\mathrm {m a x}}} \mathcal {L} _ {\mathrm {a d a p t i v e}} ^ {(t)}
\]
$$
![图片 page8-0](images/page008_img01.png)

_图4 端到端Ouro训练流程：共享warmup → 稳定训练 → 分叉为1.4B保留路径和2.6B升级循环路径 → 四个共享阶段 → 推理SFT生成Ouro-Thinking。_

我们自适应损失的意义。公式（6）中的自适应损失在步骤$t$训练门以匹配从实际性能改善派生的理想行为：

- 预测概率：门生成$\lambda_{i}^{(t)}$（退出概率）和$1 - \lambda_{i}^{(t)}$（继续概率）

- 目标标签：理想行为编码为$w_{i}^{(t)}$（目标继续概率）和$1 - w_{i}^{(t)}$（目标退出概率）

此公式同时惩罚两种失败模式：

- 欠思考：门应该在继续时退出（大标签$w_{i}^{(t)}$，但大预测退出$\lambda_{i}^{(t)}$）

- 过思考：门应该在退出时继续（小标签$w_{i}^{(t)}$，但小预测退出$1 - \lambda_{i}^{(t)}$）

优化公式（6）训练门选择权衡额外计算与可测量改善的贪婪退出步骤。实证评估见第5.4.1节。

# 4 训练循环语言模型

我们用于Ouro模型家族的端到端训练流程如图4所示。总计7.7T tokens用于训练基础模型Ouro-1.4B和Ouro-2.6B。最终推理SFT（监督微调）产生Ouro-1.4B-Thinking和Ouro-2.6B-Thinking变体。本节详述每个训练阶段的架构、数据组成和具体配置。前四个阶段训练方案的概览见表1。

_表1 Ouro 1.4B和2.6B的训练方案。_

<table><tr><td></td><td>阶段1a
预训练I</td><td>阶段1b
预训练II</td><td>阶段2
CT退火</td><td>阶段3
LongCT</td><td>阶段4
中期训练</td></tr><tr><td colspan="6">超参数</td></tr><tr><td>学习率（最终）</td><td>3.0 × 10-4</td><td>3.0 × 10-4</td><td>3.0 × 10-5</td><td>3.0 × 10-5</td><td>1.0 × 10-5</td></tr><tr><td>LR调度器</td><td>Constant</td><td>Constant</td><td>Cosine Decay</td><td>Constant</td><td>Cosine Decay</td></tr><tr><td>权重衰减</td><td></td><td></td><td>0.1</td><td></td><td></td></tr><tr><td>梯度范数裁剪</td><td></td><td></td><td>1.0</td><td></td><td></td></tr><tr><td>优化器</td><td></td><td></td><td>AdamW (β1=0.9, β2=0.95)</td><td></td><td></td></tr><tr><td>批量大小（tokens）</td><td>4M→8M</td><td></td><td></td><td>8M</td><td></td></tr><tr><td>序列长度</td><td>4K</td><td>4K</td><td>16K</td><td>64K</td><td>32K</td></tr><tr><td>训练tokens</td><td>3T</td><td>3T</td><td>1.4T</td><td>20B</td><td>300B</td></tr><tr><td>递归步骤</td><td>8</td><td></td><td></td><td>4</td><td></td></tr><tr><td>β for KL散度</td><td>0.1</td><td></td><td></td><td>0.05</td><td></td></tr><tr><td>RoPE base</td><td>10K</td><td>10K</td><td>40K</td><td>1M</td><td>1M</td></tr><tr><td colspan="6">数据重点</td></tr><tr><td>网络数据</td><td>高</td><td>高</td><td>中</td><td>低</td><td>低</td></tr><tr><td>数学与代码</td><td>低</td><td>低</td><td>高</td><td>低</td><td>高</td></tr><tr><td>长上下文</td><td>无</td><td>无</td><td>低</td><td>高</td><td>中</td></tr><tr><td>SFT质量</td><td>无</td><td>无</td><td>低</td><td>低</td><td>高</td></tr></table>

---

# 4.1 Transformer架构

Ouro模型使用标准仅解码器Transformer[33]，优先考虑循环计算机制的干净实现，无多余修改。核心架构由递归应用的一堆Transformer块组成。每个块使用带旋转位置嵌入（RoPE）[34]的多头注意力（MHA）。每个块中的前馈网络（FFN）使用SwiGLU激活[35]。为增强训练稳定性（对深度递归计算尤其关键），我们采用三明治归一化结构，在注意力和FFN子层之前都放置RMSNorm层[17]。两个模型都使用来自SmolLM2模型[36]的49,152-token词汇表。此tokenizer针对代码和拉丁字母语言优化。架构详情见表2。

_表2 Ouro模型架构配置。两个模型共享相同的词汇表和核心组件类型，在参数数量和层深度上有所不同。_

<table><tr><td>模型</td><td>参数</td><td>层数</td><td>隐藏大小(dmodel)</td><td>注意力</td><td>FFN</td><td>位置嵌入</td><td>词汇大小</td></tr><tr><td>Ouro 1.4B</td><td>1.4B</td><td>24</td><td>2048</td><td>MHA</td><td>SwiGLU</td><td>RoPE</td><td>49,152</td></tr><tr><td>Ouro 2.6B</td><td>2.6B</td><td>48</td><td>2048</td><td>MHA</td><td>SwiGLU</td><td>RoPE</td><td>49,152</td></tr></table>

# 4.2 数据

数据设定了基础模型的能力边界。我们的语料库涵盖多个阶段的网络文本、数学、代码和长上下文文档，建立核心语言理解的同时加强推理、编码和长上下文技能。除标准网络爬虫外，我们还包括针对数学推理和代码生成的目标数据集以提高复杂问题解决能力。表3总结每个训练阶段的数据组成和规模。

_表3 训练语料库统计。由于预训练期间数据是随机采样的，数据集大小不直接对应于所见tokens的总数。_

<table><tr><td>数据源</td><td>阶段</td><td># Tokens (B)</td><td># 使用Tokens (B)</td></tr><tr><td>Nemotron-CC (网络数据)</td><td>阶段1</td><td>6386</td><td>4404</td></tr><tr><td>MAP-CC (网络数据)</td><td>阶段1</td><td>800</td><td>780</td></tr><tr><td>Ultra-FineWeb-zh (网络数据)</td><td>阶段1</td><td>120</td><td>120</td></tr><tr><td>OpenCoder-pretrain</td><td>阶段1</td><td>450</td><td>450</td></tr><tr><td>MegaMath-web</td><td>阶段1</td><td>247</td><td>246</td></tr><tr><td>MegaMath-high-qualty</td><td>阶段2</td><td>64</td><td>64</td></tr><tr><td>Nemotron-CC-Math-v1</td><td>阶段2</td><td>210</td><td>210</td></tr><tr><td>Nemotron-Code</td><td>阶段2</td><td>53</td><td>53</td></tr><tr><td>Nemotron-SFT-Code</td><td>阶段2</td><td>48</td><td>48</td></tr><tr><td>Nemotron-SFT-General</td><td>阶段2</td><td>87</td><td>87</td></tr><tr><td>OpenCoder-Annealing</td><td>阶段2</td><td>7</td><td>7</td></tr><tr><td>ProLong-64K</td><td>阶段3</td><td>20</td><td>20</td></tr><tr><td>Mid-training SFT Mix</td><td>阶段4</td><td>182</td><td>90</td></tr></table>

_表4 阶段1（稳定训练I和II）的数据组成。总数据集大小：6T tokens。_

<table><tr><td>数据源</td><td>Nemotron-CC</td><td>MAP-CC</td><td>Ultra-FineWeb-zh</td><td>OpenCoder-pretrain</td><td>MegaMath-web</td></tr><tr><td>比例 (%)</td><td>73.4</td><td>13.0</td><td>2.0</td><td>7.5</td><td>4.1</td></tr></table>

为确保可重复性，我们的训练语料库完全由开源数据集组成，数据统计总结于表4。我们将数据分为四个阶段，每个阶段的建设策略与现代预训练中常用的Warmup-Stable-Decay（WSD）学习率调度器[37]对齐。

阶段1：预训练 此阶段支持训练的热身和稳定阶段。语料库主要由网络CommonCrawl（CC）数据组成。因为我们寻求在>2T tokens上训练模型，许多流行的开放语料库太小（例如Fineweb-Edu为1.3T tokens[38]，DCLM为2.6T tokens[39]）。因此我们使用Nemotron-CC[40]（6.3T tokens）作为稳定阶段的主要数据集。为给模型提供基本中文能力，我们包含Ultra-FineWeb-zh[41]和MAP-CC[42]。然而，由于tokenizer中没有中文词汇，字符会被碎片化为多个字节级子token，因此我们从阶段2开始移除中文。为增强编码和数学能力，我们纳入OpenCoder[43]和MegaMath[44]。详见表4了解数据集比例。

阶段2：持续训练（CT）退火 CT退火阶段纳入更高质量数据，在退火学习率下增强模型。Token序列长度扩展至16K tokens，超过大多数样本的长度以最小化截断。我们从Nemotron-CC的高质量子集构建语料库，并增强HQ MegaMath、Nemotron-CC-Math-v1[45,46]、OpenCoder-Annealing[43]、Nemotron预训练Code-v1[46]和Nemotron预训练SFT-v1[46]。数据组成见表5。

_表5 阶段2（CT退火）的数据组成。总数据集大小：1.4T tokens。_

<table><tr><td>数据源</td><td>比例 (%)</td></tr><tr><td>Nemotron-CC-high-qualty</td><td>66.5</td></tr><tr><td>Nemotron-CC-Math-v1</td><td>15.0</td></tr><tr><td>MegaMath-high-qualty</td><td>4.6</td></tr><tr><td>OpenCoder-LLM/opc-annealing-corpus</td><td>0.5</td></tr><tr><td>Nemotron预训练Code-v1/Synthetic-Code</td><td>3.8</td></tr><tr><td>Nemotron预训练SFT-v1/Nemotron-SFT-Code</td><td>3.4</td></tr><tr><td>Nemotron预训练SFT-v1/Nemotron-SFT-General</td><td>6.2</td></tr></table>

阶段3：长上下文训练（LongCT） LongCT阶段扩展模型的长上下文能力。我们采用ProLong[47]的64K长度子集，由20B tokens组成，在更长序列上训练模型并提高其处理长上下文的能力。

阶段4：中期训练 此阶段使用多样化的极高质量数据，包括$\langle$问题、答案$\rangle$和$\langle$问题、CoT、答案$\rangle$样本，以进一步发展高级能力。我们整合20+个开源SFT数据集以提高数据广度，并对主流评估基准进行彻底的去污染以避免重叠。所有样本转换为ChatML格式以减少后续后训练阶段的对齐税。处理后，我们获得182B tokens，随机采样90B tokens。为稳定训练分布，我们重放30B tokens来自阶段1，180B来自阶段2，产生300B tokens的有效 volume。因此，此阶段在多样监督信号下整合和扩展预训练期间获得的能力。

# 4.3 训练稳定性和自适应配置

我们使用flame[48]框架进行预训练，构建于torchtitan[49]之上。在训练期间，我们优先考虑稳定性而非激进扩展，基于训练动力学的实证观察进行几项关键调整。这些决策对实现递归架构的稳定收敛至关重要，其表现出与标准transformer不同的优化特征。

为稳定性的递归步骤减少。我们在阶段1a（稳定训练I）的8个递归步骤初始实验导致损失峰值和梯度振荡。我们假设这源于多个递归迭代中复合梯度流，可能放大小扰动。因此，我们将递归步骤从8减少到4（阶段1b，稳定训练II中的图4），在计算深度与训练稳定性之间取得平衡。

批量大小扩展。为进一步增强稳定性，我们将批量大小从4M逐步增加到8M tokens。更大的批量大小提供更稳定的梯度估计，这对递归架构尤为重要，因为跨多个迭代的梯度流可能引入额外方差。

KL散度系数减少。我们在公式（4）中战略性地将$\beta$从阶段1a的0.1减少到后续阶段的0.05。这一减少有两个目的：（1）减少任务损失与KL penalty之间的冲突梯度，带来更稳定的优化；（2）减少来自统一先验的"拉力"，允许模型在不受人为约束的情况下更自由地探索有益深度模式。这一调整允许模型学习有用的深度模式而不受过度约束。

优化配置。在所有阶段，我们使用AdamW优化器，权重衰减设置为0.1，$\beta_{1} = 0.9$，$\beta_{2} = 0.95$，梯度裁剪为1.0。选择这些保守设置 specifically to maintain stability with recurrent architectures。

学习率考虑。我们经验地发现递归架构需要比参数匹配Transformer更小的学习率。鉴于计算约束，我们没有运行详尽的学习率扫描，而是采用优先考虑稳定收敛而非可能更快但风险更高的保守速率。

序列长度进展。序列长度跨阶段逐步增加：两个预训练阶段均为4K tokens，CT退火为16K，长上下文训练为64K，中期训练为32K。这一进展在训练吞吐量扩展上下文能力的同时稳定优化。

# 4.3.1 分阶段训练详情

- 阶段1a：预训练阶段I（探索）。我们用8个递归步骤初始化训练。学习率遵循Warmup-Stable调度，峰值为$3 \times 10^{-4}$。序列长度为4K tokens，初始批量大小为4M tokens，逐渐增加到8M以提高稳定性。在此阶段，我们观察到训练不稳定，促使后续架构调整。

- 阶段1b：具有稳定性驱动升级循环的预训练阶段II。在识别阶段1a的稳定性问题后，我们将递归步骤从8减少到4。为在提高稳定性的同时保持计算效率，我们将方法分为两个变体：

  - 1.4B Ouro：使用原始24个预训练层，4个递归步骤

  - 2.6B Ouro：通过层复制将24层升级到48层，4个递归步骤

我们架构的递归特性使这种升级循环过程特别顺畅，因为跨迭代的共享权重自然地促进了层复制，避免了标准transformer升级循环中的典型不稳定性。

- 阶段2：CT退火。学习率退火至$3 \times 10^{-5}$，同时让模型接触高质量训练数据。递归步骤保持在4，已被证明是稳定性-性能权衡的最优。数据组成经过仔细平衡（如表5所示）。

- **阶段3：LongCT。** 批量大小保持在8M tokens。减少的KL系数（$\beta = 0.05$）即使在64K长度序列下也继续提供稳定的训练动态。

- 阶段4：中期训练。学习率进一步降低至$1 \times 10^{-5}$，使用余弦调度器以帮助模型更好地吸收这个多样化、高质量的数据集。

# 4.4 监督微调

数据组成。我们对约8.3M样本的多样化语料库执行SFT，这些样本来自高质量公共数据集。如表6所示，我们的训练混合强调数学推理（3.5M样本）和代码生成（3.2M样本），同时纳入科学推理（808K样本）和对话能力（767K样本）。

对于数学推理，我们结合OpenThoughts3[50]和AceReason-1.1-SFT[51]以提供问题解决策略的全面覆盖。我们的代码训练数据聚合多个来源，包括AceReason-1.1-SFT、OpenCodeReasoning[52]、Llama-Nemotron-后训练数据集[53]和OpenThoughts3，确保对多样化编程范式和推理模式的广泛暴露。科学推理能力通过OpenThoughts3和Llama-Nemotron-后训练数据集发展，而对话能力使用OO1-Chat-747K<sup>1</sup>和DeepWriting-20K[54]数据集增强。

训练配置。我们使用最大序列长度$32\mathrm{K}$tokens训练2个epoch，使用LlamaFactory代码库[55]。我们使用学习率$2 \times 10^{-5}$和$\beta = (0.9, 0.95)$的Adam优化器，应用余弦衰减调度以实现稳定收敛。

_表6 监督微调数据组成。训练语料库包含跨四个关键能力领域的8.3M样本。_

<table><tr><td>主题</td><td>数据源</td><td>大小</td></tr><tr><td>数学</td><td>OpenThoughts3, AceReason-1.1-SFT</td><td>3.5M</td></tr><tr><td>代码</td><td>AceReason-1.1-SFT, OpenCodeReasoning, Llama-Nemotron后训练数据集, OpenThoughts3</td><td>3.2M</td></tr><tr><td>科学</td><td>OpenThoughts3, Llama-Nemotron后训练数据集</td><td>808K</td></tr><tr><td>聊天</td><td>OO1-Chat-747K, DeepWriting-20K</td><td>767K</td></tr></table>

# 4.5 强化学习尝试

在SFT阶段之后，我们使用DAPO[56]和GRPO[57]在DAPO-17K数据集上进行探索性RLVR（带可验证奖励的强化学习）对齐实验。这些尝试未能比最终SFT检查点获得显著性能提升。主要问题源于模型的动态早退出机制。vLLM/SGLang通过固定执行路径提供快速rollouts，这在LoopLM的可变深度计算下失效。

我们尝试了两种方法，均未成功：

1. 离策略rollouts：我们在vLLM中生成分散的完整四步rollouts，每个token产生四个logit候选项。然后我们选择第一个超过终止阈值的token来模拟早退。对于更新，我们使用到该步骤的累积损失，丢弃后续token和损失。这种离策略不匹配（其中token以最终深度产生，损失在较早深度计算）未能提高性能。

2. 固定4轮RL：为避免离策略问题，我们在固定四步递归步骤执行rollouts和更新。训练正常进行，但性能未超过SFT检查点。类似原因是规模：在已经经历广泛SFT之后，这些较小的模型可能对RL收益的提升空间有限。有趣的是，模型在推理时仍在有益的情况下使用更少轮次，尽管在四轮训练。这一泛化背后的机制仍不清楚。

随着我们继续开发能够完全支持LoopLM动态计算的基础设施，我们将进一步探索该架构的RL对齐。

<sup>1</sup>https://huggingface.co/datasets/m-a-p/001-Chat-747K

<sup>2</sup>训练因基础设施问题中断；我们从最后一个保存的检查点恢复，学习率接近原始余弦衰减调度。

---

# 5 实验

# 5.1 基础模型评估

我们对使用LoopLM架构在7.7T tokens上训练的Ouro基础模型进行全面评估。评估重点关注其在通用知识、推理、数学、科学、编码和多语言能力方面的性能。所有基准测试使用lm-eval-harness[58]和evalplus[59]框架评估，详见附录C.1的设置。

对于基础模型基线，我们将Ouro模型与领先的开源基础模型进行比较，包括Qwen2.5[2]、Qwen3[3]、Gemma3[4]、Llama3.1[5]和Llama3.2[5]系列基础模型。所有模型使用相同评估流程评估以确保公平比较。

_表7 1.4B LoopLM模型与1-4B参数基线的比较。最高分数加粗，第二高分数加下划线。LoopLM的列以灰色突出显示。_

<table><tr><td></td><td>Gemma3 1B</td><td>Llama3.2 1.2B</td><td>Qwen2.5 1.5B</td><td>Qwen3 1.7B</td><td>Qwen2.5 3B</td><td>Llama3.2 3B</td><td>Qwen3 4B</td><td>Gemma3 4B</td><td>Ouro 1.4B R4</td></tr><tr><td>架构</td><td>Dense</td><td>Dense</td><td>Dense</td><td>Dense</td><td>Dense</td><td>Dense</td><td>Dense</td><td>Dense</td><td>LoopLM</td></tr><tr><td># 参数</td><td>1.0B</td><td>1.0B</td><td>1.5B</td><td>1.7B</td><td>3.0B</td><td>3.0B</td><td>4.0B</td><td>4.0B</td><td>1.4B</td></tr><tr><td># Tokens</td><td>2T</td><td>9T</td><td>18T</td><td>36T</td><td>18T</td><td>9T</td><td>36T</td><td>4T</td><td>7.7T</td></tr><tr><td colspan="10">通用任务</td></tr><tr><td>MMLU</td><td>39.85</td><td>45.46</td><td>60.99</td><td>62.46</td><td>65.62</td><td>59.69</td><td>73.19</td><td>58.37</td><td>67.35</td></tr><tr><td>MMLU-Pro</td><td>11.31</td><td>11.80</td><td>29.11</td><td>37.27</td><td>37.87</td><td>33.34</td><td>51.40</td><td>34.61</td><td>48.62</td></tr><tr><td>BBH</td><td>30.26</td><td>30.72</td><td>43.66</td><td>53.51</td><td>55.37</td><td>39.45</td><td>70.95</td><td>66.32</td><td>71.02</td></tr><tr><td>ARC-C</td><td>39.25</td><td>41.98</td><td>54.44</td><td>55.72</td><td>55.46</td><td>52.47</td><td>63.65</td><td>60.92</td><td>60.92</td></tr><tr><td>HellaSwag</td><td>56.12</td><td>59.35</td><td>67.73</td><td>67.09</td><td>74.54</td><td>73.09</td><td>75.66</td><td>75.58</td><td>74.29</td></tr><tr><td>Winogrande</td><td>58.72</td><td>62.75</td><td>66.77</td><td>66.30</td><td>70.17</td><td>69.14</td><td>71.19</td><td>71.07</td><td>72.30</td></tr><tr><td colspan="10">数学与编码任务</td></tr><tr><td>GSM8K</td><td>2.05</td><td>7.05</td><td>60.73</td><td>70.28</td><td>74.60</td><td>67.20</td><td>72.86</td><td>68.69</td><td>78.92</td></tr><tr><td>MATH500</td><td>41.00</td><td>7.40</td><td>17.60</td><td>25.80</td><td>42.60</td><td>40.80</td><td>59.60</td><td>68.60</td><td>82.40</td></tr><tr><td>HumanEval</td><td>6.70</td><td>19.50</td><td>52.40</td><td>66.50</td><td>68.90</td><td>29.90</td><td>77.40</td><td>34.80</td><td>74.40</td></tr><tr><td>HumanEval+</td><td>5.50</td><td>17.40</td><td>46.30</td><td>59.80</td><td>62.20</td><td>26.20</td><td>70.70</td><td>29.30</td><td>67.40</td></tr><tr><td>MBPP</td><td>12.40</td><td>35.70</td><td>60.30</td><td>68.00</td><td>63.00</td><td>50.30</td><td>78.80</td><td>60.60</td><td>73.00</td></tr><tr><td>MBPP+</td><td>10.10</td><td>29.10</td><td>50.00</td><td>58.50</td><td>54.20</td><td>39.70</td><td>65.90</td><td>51.10</td><td>62.70</td></tr></table>

评估结果总结 基于整体评估结果，我们强调关于基础模型的关键结论：

(1) 我们1.4B参数的Ouro模型（带4个递归步骤）在大多数基准测试上实现与4B Qwen3-Base相当的性能。值得注意的是，它在BBH（71.02 vs 70.95）、GSM8K（78.92 vs 72.86）和MATH500（82.40 vs 59.60）等具有挑战性的推理任务上匹配或超越4B模型。

(2) 2.6B参数Ouro模型在推理密集型基准测试上超越高达8B参数的密集模型。它在MMLU-Pro上达到55.73，BBH上80.46，MATH500上90.85，超越8B Qwen3-Base（分别为53.72、77.65和62.30）。

(3) 递归架构在需要多步推理和知识操作的任务上表现出特殊优势，在MMLU-Pro、BBH、GSM8K和MATH500基准测试上观察到最显著的收益，验证了我们的假设，即迭代计算增强推理能力。

# 5.2 推理模型评估

我们评估Ouro推理模型（Ouro-Thinking）在需要多步问题解决和深度推理的挑战性数学和科学基准测试上的推理能力，4个递归步骤。评估包括AIME 2024/2025（美国数学邀请赛）、OlympiadBench、GPQA、SuperGPQA、BeyondAIME和HLE，代表该领域一些最具挑战性的推理任务。

_表8 2.6B LoopLM模型与3-12B参数基线的比较。最高分数加粗，第二高分数加下划线。LoopLM的列以灰色突出显示。_

<table><tr><td></td><td>Qwen2.5 3B</td><td>Llama3.2 3B</td><td>Qwen3 4B</td><td>Gemma3 4B</td><td>Qwen2.5 7B</td><td>Llama3.1 8B</td><td>Qwen3 8B</td><td>Gemma3 12B</td><td>Ouro 2.6B R4</td></tr><tr><td>架构</td><td>Dense</td><td>Dense</td><td>Dense</td><td>Dense</td><td>Dense</td><td>Dense</td><td>Dense</td><td>Dense</td><td>LoopLM</td></tr><tr><td># 总参数</td><td>3.0B</td><td>3.0B</td><td>4.0B</td><td>4.0B</td><td>7.0B</td><td>8.0B</td><td>8.0B</td><td>12.0B</td><td>2.6B</td></tr><tr><td># 训练Tokens</td><td>18T</td><td>9T</td><td>36T</td><td>4T</td><td>18T</td><td>15T</td><td>36T</td><td>12T</td><td>7.7T</td></tr><tr><td colspan="10">通用任务</td></tr><tr><td>MMLU</td><td>65.62</td><td>59.69</td><td>73.19</td><td>58.37</td><td>74.20</td><td>73.02</td><td>76.63</td><td>72.14</td><td>74.60</td></tr><tr><td>MMLU-Pro</td><td>37.87</td><td>33.34</td><td>51.40</td><td>34.61</td><td>43.55</td><td>43.24</td><td>53.72</td><td>49.21</td><td>55.73</td></tr><tr><td>BBH</td><td>55.37</td><td>39.45</td><td>71.14</td><td>66.32</td><td>53.72</td><td>71.56</td><td>77.65</td><td>78.41</td><td>80.46</td></tr><tr><td>ARC-C</td><td>55.46</td><td>52.47</td><td>63.65</td><td>60.75</td><td>63.65</td><td>60.75</td><td>66.10</td><td>72.44</td><td>66.40</td></tr><tr><td>HellaSwag</td><td>74.54</td><td>73.09</td><td>75.66</td><td>75.58</td><td>79.98</td><td>81.97</td><td>79.60</td><td>83.68</td><td>79.69</td></tr><tr><td>Winogrande</td><td>70.17</td><td>69.14</td><td>71.19</td><td>71.27</td><td>76.48</td><td>77.11</td><td>76.80</td><td>77.74</td><td>75.85</td></tr><tr><td colspan="10">数学与编码任务</td></tr><tr><td>GSM8K</td><td>74.60</td><td>67.20</td><td>72.86</td><td>68.69</td><td>81.50</td><td>78.17</td><td>83.09</td><td>77.18</td><td>81.58</td></tr><tr><td>MATH500</td><td>42.60</td><td>40.80</td><td>59.60</td><td>68.60</td><td>61.20</td><td>52.90</td><td>62.30</td><td>83.20</td><td>90.85</td></tr><tr><td>HumanEval</td><td>68.90</td><td>29.90</td><td>77.70</td><td>34.80</td><td>79.30</td><td>38.40</td><td>84.80</td><td>46.30</td><td>78.70</td></tr><tr><td>HumanEval+</td><td>62.20</td><td>26.20</td><td>70.70</td><td>29.30</td><td>70.60</td><td>31.10</td><td>75.30</td><td>37.20</td><td>70.70</td></tr><tr><td>MBPP</td><td>63.00</td><td>50.30</td><td>78.80</td><td>60.60</td><td>73.80</td><td>62.40</td><td>79.00</td><td>73.50</td><td>80.40</td></tr><tr><td>MBPP+</td><td>54.20</td><td>39.70</td><td>65.90</td><td>51.10</td><td>63.50</td><td>51.60</td><td>67.90</td><td>66.10</td><td>66.60</td></tr></table>

_表9 不同基准测试的性能比较。对于AIME24和AIME25，我们报告pass@1/pass@10指标。最高分数加粗，第二高分数加下划线。_

<table><tr><td rowspan="2">模型</td><td colspan="2">AIME24</td><td colspan="2">AIME25</td><td rowspan="2">Olympiad bench</td><td rowspan="2">Beyond AIME</td><td rowspan="2">HLE</td><td rowspan="2">Super GPQA</td><td rowspan="2">GPQA</td></tr><tr><td>pass@1</td><td>pass@10</td><td>pass@1</td><td>pass@10</td></tr><tr><td>Ouro-1.4B-Thinking-R4</td><td>65.0</td><td>83.3</td><td>46.3</td><td>73.3</td><td>71.6</td><td>34.0</td><td>5.21</td><td>47.4</td><td>45.5</td></tr><tr><td>Ouro-2.6B-Thinking-R4</td><td>64.7</td><td>90.0</td><td>50.3</td><td>76.7</td><td>76.4</td><td>39.0</td><td>5.58</td><td>53.7</td><td>52.7</td></tr><tr><td>Qwen3-1.7B</td><td>32.0</td><td>55.6</td><td>22.0</td><td>33.3</td><td>56.4</td><td>15.0</td><td>4.13</td><td>35.9</td><td>34.0</td></tr><tr><td>Qwen3-4B</td><td>61.3</td><td>75.0</td><td>51.3</td><td>63.3</td><td>73.2</td><td>31.0</td><td>5.21</td><td>51.9</td><td>54.5</td></tr><tr><td>Qwen3-8B</td><td>73.0</td><td>86.7</td><td>66.7</td><td>81.3</td><td>75.3</td><td>38.0</td><td>2.22</td><td>48.0</td><td>59.1</td></tr><tr><td>Deepseek-Distill-Qwen-1.5B</td><td>29.6</td><td>66.7</td><td>23.0</td><td>43.33</td><td>56.44</td><td>9.0</td><td>4.2</td><td>26.5</td><td>33.2</td></tr><tr><td>Deepseek-Distill-Qwen-7B</td><td>57.3</td><td>83.3</td><td>36.0</td><td>73.3</td><td>72.0</td><td>30.0</td><td>5.14</td><td>46.6</td><td>51.0</td></tr></table>

# 基准测试

AIME 2024/2025[60]。每年AIME I和II各30题；整数答案0-999。

- OlympiadBench[61]。奥赛级双语科学问题；支持多模态输入的图像。

- GPQA[62]。生物学、物理学和化学中的448道研究生级选择题；搜索抵抗设计。

- SuperGPQA[63]。扩展至约285个研究生学科的GPQA；策划保持挑战性。

- BeyondAIME[64]。超越AIME的困难整数答案数学；强调污染抵抗。

- HLE[65]。多学科闭卷基准测试；专家编写，具有公开分割和私有测试集。

比较的模型。我们报告Ouro-1.4B-Thinking和Ouro-2.6B-Thinking的结果，它们是基于LoopLM的带迭代深度的循环语言模型。作为基线，我们包括Qwen3-1.7B、Qwen3-4B、Qwen3-8B、DeepSeek-Distill-Qwen-1.5B和DeepSeek-Distill-Qwen-7B。我们尽可能使用规模匹配的基线，否则与下一个更大规模的广泛使用模型进行比较。

评估协议。所有系统使用单一内部工具和相同提示进行评估。我们在跨基准测试使用LLM-as-judge协议，固定评分标准和平局打破策略。除非另有说明，每个模型使用temperature = 1.0和top_p = 0.7进行解码。

评估结果。表9总结结果。LoopLM架构中的迭代推理在这些任务上提供一致的收益。带4个递归步骤的1.4B Ouro模型在OlympiadBench上达到71.55（对比Qwen3-4B的73.18），在BeyondAIME上达到34.0（对比Qwen3-4B的31.0）。带4个递归步骤变体的2.6B在OlympiadBench上得分76.44（对比Qwen3-8B的75.25），在BeyondAIME上达到39.0（对比Qwen3-8B的38.0）。

# 5.3 按递归深度和外推的性能

_表10 Ouro 1.4B基础模型在不同递归步骤上的性能（C-QA是CommonsenseQA[66]）。步骤5-8代表外推，因为模型仅用最多4步训练。性能在训练深度（$T = 4$）处达到峰值，然后下降。_

<table><tr><td>UT步骤</td><td>ARC-C(25-shot)</td><td>ARC-E(8-shot)</td><td>C-QA(10-shot)</td><td>HellaSwag(10-shot)</td><td>MMLU(5-shot平均)</td><td>Winogrande(5-shot)</td></tr><tr><td>1</td><td>37.63</td><td>63.85</td><td>44.64</td><td>55.24</td><td>41.21</td><td>56.99</td></tr><tr><td>2</td><td>54.86</td><td>80.30</td><td>67.98</td><td>71.15</td><td>60.43</td><td>66.69</td></tr><tr><td>3</td><td>59.47</td><td>83.33</td><td>74.37</td><td>74.07</td><td>66.71</td><td>71.35</td></tr><tr><td>4</td><td>60.92</td><td>83.96</td><td>75.43</td><td>74.29</td><td>67.45</td><td>72.30</td></tr><tr><td colspan="7">外推（训练于T=4）</td></tr><tr><td>5</td><td>58.96</td><td>82.91</td><td>75.35</td><td>73.72</td><td>66.64</td><td>70.32</td></tr><tr><td>6</td><td>59.73</td><td>82.58</td><td>74.94</td><td>72.77</td><td>65.77</td><td>71.03</td></tr><tr><td>7</td><td>58.96</td><td>81.99</td><td>74.28</td><td>72.35</td><td>65.28</td><td>70.09</td></tr><tr><td>8</td><td>58.19</td><td>82.07</td><td>73.55</td><td>71.60</td><td>64.49</td><td>69.30</td></tr></table>

_表11 Ouro 2.6B基础模型在不同递归步骤上的性能（C-QA是CommonsenseQA[66]）。步骤5-8代表外推，因为模型仅用最多4步训练。性能在训练深度（$T = 4$）附近最强，外推时显示不同的退化模式。_

<table><tr><td>UT步骤</td><td>ARC-C(25-shot)</td><td>ARC-E(8-shot)</td><td>C-QA(10-shot)</td><td>HellaSwag(10-shot)</td><td>MMLU(5-shot平均)</td><td>Winogrande(5-shot)</td></tr><tr><td>1</td><td>47.95</td><td>72.39</td><td>57.58</td><td>68.94</td><td>51.55</td><td>61.48</td></tr><tr><td>2</td><td>62.37</td><td>85.23</td><td>76.90</td><td>77.61</td><td>67.63</td><td>70.48</td></tr><tr><td>3</td><td>65.36</td><td>87.33</td><td>79.77</td><td>79.12</td><td>73.57</td><td>74.35</td></tr><tr><td>4</td><td>66.38</td><td>86.95</td><td>81.65</td><td>79.56</td><td>74.60</td><td>75.53</td></tr><tr><td colspan="7">外推（训练于T=4）</td></tr><tr><td>5</td><td>65.36</td><td>86.83</td><td>81.24</td><td>79.57</td><td>74.43</td><td>75.93</td></tr><tr><td>6</td><td>65.02</td><td>86.74</td><td>81.08</td><td>79.63</td><td>73.79</td><td>75.37</td></tr><tr><td>7</td><td>65.44</td><td>86.57</td><td>80.75</td><td>79.59</td><td>72.92</td><td>75.77</td></tr><tr><td>8</td><td>64.76</td><td>86.49</td><td>81.08</td><td>79.50</td><td>72.24</td><td>74.59</td></tr></table>

_表12 Ouro-1.4B-Thinking模型按递归步骤的性能。模型在$T = 4$训练。性能在$T = 4$或$T = 5$处达到峰值。所有分数为百分比（0-100）。_

<table><tr><td>基准测试</td><td>T=1</td><td>T=2</td><td>T=3</td><td>T=4</td><td>T=5</td><td>T=6</td><td>T=7</td><td>T=8</td></tr><tr><td>OlympiadBench</td><td>2.22</td><td>59.70</td><td>70.67</td><td>71.55</td><td>72.30</td><td>69.48</td><td>69.04</td><td>66.81</td></tr><tr><td>SuperGPQA</td><td>2.03</td><td>33.07</td><td>44.50</td><td>47.37</td><td>48.73</td><td>46.15</td><td>45.29</td><td>42.88</td></tr><tr><td>AIME 2024</td><td>0.00</td><td>37.33</td><td>62.33</td><td>65.00</td><td>60.67</td><td>50.67</td><td>42.33</td><td>38.67</td></tr><tr><td>AIME 2025</td><td>0.33</td><td>25.00</td><td>43.33</td><td>46.30</td><td>47.00</td><td>43.00</td><td>41.00</td><td>38.00</td></tr></table>

_表13 Ouro-2.6B-Thinking模型按递归步骤的性能。模型在$T = 4$训练。性能在$T = 3$或$T = 4$处达到峰值。所有分数为百分比（0-100）。_

<table><tr><td>基准测试</td><td>T=1</td><td>T=2</td><td>T=3</td><td>T=4</td><td>T=5</td><td>T=6</td><td>T=7</td><td>T=8</td></tr><tr><td>OlympiadBench</td><td>18.96</td><td>68.59</td><td>75.56</td><td>76.44</td><td>71.85</td><td>69.19</td><td>57.63</td><td>39.26</td></tr><tr><td>SuperGPQA</td><td>15.66</td><td>48.58</td><td>56.70</td><td>53.68</td><td>56.45</td><td>55.44</td><td>53.32</td><td>46.84</td></tr><tr><td>AIME 2024</td><td>3.00</td><td>52.00</td><td>70.33</td><td>64.70</td><td>57.00</td><td>56.33</td><td>49.67</td><td>39.00</td></tr><tr><td>AIME 2025</td><td>2.00</td><td>40.67</td><td>50.67</td><td>50.30</td><td>49.33</td><td>46.00</td><td>38.00</td><td>24.33</td></tr></table>

我们分析Ouro模型性能作为其递归计算深度的函数。我们的模型用最多4个递归步骤（$T = 4$）训练。我们研究基础模型和SFT Ouro-Thinking模型的此行为。

基础模型性能。表10和表11分别呈现Ouro 1.4B和2.6B基础模型在深度T = 1到T = 8评估的性能。

对于两个基础模型，标准基准测试（如MMLU、ARC-C）的性能通常在训练深度$T = 4$之前提高。步骤$T = 5$到$T = 8$代表超出训练配置的外推。如两表所示，基准测试性能在外推时出现适度退化，与$T = 4$的峰值相比明显下降。

然而，任务特定性能的这种退化与模型的安全对齐形成鲜明对比。如第7.1节详述，模型的安全性随递归步骤数增加而提升，即使进入外推 regime（$T > 4$）。这表明虽然模型在基准测试中的细粒度知识在训练深度之外可能减弱，但迭代细化过程继续增强其安全对齐。

推理模型（SFT）性能。我们在SFT模型Ouro-Thinking上进行类似分析，以了解递归深度如何影响专业推理任务。1.4B和2.6B模型的结果分别见表12和表13。

对于两个SFT模型，$T = 1$的性能非常低，确认迭代细化对这些复杂任务至关重要。性能通常在训练深度附近或达到峰值，但显示略有不同的模式。1.4B模型（表12）在$T = 4$或$T = 5$附近达到峰值。2.6B模型（表13）往往更早达到峰值，在$T = 3$或$T = 4$。有趣的是，两个模型都不在所有任务上严格在$T = 4$达到峰值，这与通常基于logit的基线模型评估不同。这可能表明这些推理任务所需更长的解码允许在不同的递归深度上更积极地探索能力。对于两个模型，当外推到更深、未见过的递归步骤（$T = 6-8$）时，性能退化，强化了性能针对训练中看到的深度优化的结论。

# 5.4 早退出和自适应计算效率

LoopLM架构的定义优势在于其自适应计算分配能力。与具有固定计算预算的标准transformer不同，我们的模型可以根据输入复杂性动态调整递归步骤数。本节研究实现自适应早退的各种策略，比较它们在平衡计算效率与任务性能方面的有效性。

# 5.4.1 早退出策略

我们探索三种不同方法来确定模型何时应终止其迭代计算并产生最终输出。

基线：静态退出。最简单的策略强制模型在预定递归步骤退出，无论输入特征如何。虽然这种方法提供可预测的计算成本，但未能利用模型自适应资源分配的潜力。我们评估步骤1到4的静态退出以建立性能边界并理解计算深度与准确性之间的关系。

隐藏状态差异阈值。这种启发式方法监控连续递归步骤之间表示变化的幅度。在每个步骤$t$，我们计算$\Delta h_t = \| h_t - h_{t-1} \|_2$，当$\Delta h_t < \epsilon$时触发早退，其中$\epsilon$为某个阈值。

带Q-Exit准则的学习门控。我们的主要方法采用第4节描述的学习退出门，其基于模型的当前隐藏状态产生逐步停止概率$\lambda_{t}$。在推理期间，我们应用Q-exit准则：在每个步骤$t$，我们计算累积分布函数$\mathrm{CDF}(t) = \sum_{i=1}^{t} p(i|x)$，当$\mathrm{CDF}(t)$超过阈值$q \in [0,1]$时退出。阈值$q$作为部署时超参数，控制计算-准确性权衡，无需重新训练模型。

我们在此训练配置下评估此策略。未训练配置使用门如我们标准预训练流程（带统一先验KL损失的熵正则化目标）中所训练。这代表门在与语言建模联合优化整个阶段1-4时的行为。训练配置额外应用第3.4节描述的专业自适应退出损失，明确教导门基于观察到的任务损失改善做出停止决策。

![图片 page17-7](images/page017_img01.png)

_图5 MMLU上早退出策略的比较。我们评估四种方法在不同平均退出轮次下的表现：静态基线（红色三角形）、隐藏状态差异阈值（绿色方块）、标准预训练的Ponder门（蓝色圆圈），以及第3.4节专门自适应退出训练的Ponder门（橙色菱形）。_

实验结果。图5呈现所有策略在MMLU基准测试上的准确性-效率权衡曲线。通过改变退出阈值（或基线的静态退出步骤），我们获得每个方法的多个操作点，能够在相同计算预算（按平均退出轮次测量）下直接比较。

此分析中出现几个关键发现：

1. 带专门自适应退出训练的Ponder门在每个计算预算下实现最佳准确性，展示第3.4节描述的基于损失改善的训练信号比标准熵正则化的明显收益。在平均退出轮次2.5时，专门训练达到66%准确性，而标准门仅达到约64%；

2. 即使没有专门训练，标准预训练的Ponder门也显著优于静态基线，验证统一先验熵正则化目标成功启用自适应计算。门通过一般训练动力学学习区分输入难度，尽管缺乏将停止决策与实际性能改善相关联的明确监督。这表明我们基础训练方法已捕获资源分配的有用信号；

3. 隐藏状态差异阈值策略表现得出奇地具有竞争力，密切跟踪两种门配置。在中等计算预算（2-3平均轮次）下，其准确性在专门训练门的1%-2%以内，表明表示稳定性为计算收敛提供合理代理。然而，专门训练门在所有操作点上的一致 superior 性能确认通过自适应退出损失的显式监督捕获了超出仅从表示动态推断的信息；

4. 比较未训练和训练的门配置揭示专门训练过程的价值主张。这些曲线之间的差距，在大多数操作点上约2%-3%准确性，代表教导门显式监控任务损失改善$I_{t}^{(n)}$而非仅依赖熵正则化发现停止策略的收益。这一实证结果验证我们引入自适应退出损失作为专门训练目标的设计选择；

5. 基线从1到4轮的单调改善确认"更深更好"特性，同时揭示递减收益。从1.0到2轮的戏剧性跳跃（40%到60%准确性）与3到4轮的边际增益（67.35%准确性）形成对比。这一模式解释了自适应方法为何有效：大多数样本在中间深度达到 near-maximal 性能，只有少数需要完整计算深度。

# 5.4.2 用于推理效率的KV缓存共享

我们架构的递归特性引入了一个挑战：天真地，每个递归步骤需要维护自己的KV缓存，导致我们的4步模型产生4倍内存开销。我们研究通过KV缓存重用减少此开销的策略。

预填充阶段 在预填充阶段（处理输入提示）期间，我们发现所有四个递归步骤需要自己的KV缓存，因为每个步骤以无法由 earlier 步骤近似的方式转换表示。尝试在预填充期间重用KV缓存导致性能下降（GSM8K上>10分）。

解码阶段 然而，在解码阶段（自回归生成）期间，我们发现KV缓存重用变得可行。我们探索两种策略：

1. 最后步骤重用：仅维护最后一个（第4个）递归步骤的KV缓存

2. 第一步骤重用：仅维护第一个（第1个）递归步骤的KV缓存

3. 平均重用：维护跨所有四个步骤的平均KV缓存

_表14 解码期间KV缓存共享策略。最后步骤和平均策略在减少4倍内存的同时实现最小性能损失。_

<table><tr><td>策略</td><td>GSM8K</td><td>MATH-500</td><td>内存减少</td></tr><tr><td>完整（4×缓存）</td><td>78.92</td><td>82.40</td><td>1.00×</td></tr><tr><td>仅第一步骤</td><td>18.73</td><td>8.43</td><td>4.00×</td></tr><tr><td>仅最后步骤</td><td>78.85</td><td>80.40</td><td>4.00×</td></tr><tr><td>平均</td><td>78.73</td><td>78.52</td><td>4.00×</td></tr></table>

如表14所示，这些策略产生截然不同的结果。仅重用第一步骤的缓存导致灾难性性能崩溃（例如GSM8K上18.73，从78.92下降），表明初始表示对后续解码步骤不足。相比之下，最后步骤和平均重用策略实现与完整缓存基线几乎相同的性能（GSM8K上在0.3分以内），同时成功将内存需求减少4倍。最后步骤策略在MATH-500上表现略好于平均方法，表明最终递归步骤的表示对后续token生成最有信息。这一发现使得LoopLM模型的实际部署成为可能，内存占用与类似参数数量的标准transformer相当。

# 6 从参数知识角度理解LoopLM的优势

为什么LoopLM在参数数量不增加的情况下实现远更好的性能？尽管在[7]中观察到潜在的增强推理能力，但优势来源仍不清楚。具体而言，LoopLM是否因相同参数规模下增强的知识容量而表现更好？还是它们在提取和组合编码在参数内的知识方面具有更好的能力？为理解这一现象，我们探索简单地增加循环次数究竟增强了哪些能力。在本节中，我们进行实验以测试模型在参数中记忆事实知识的能力，以及基于[67-69]中完全可控的综合任务的组合编码在参数内的现有知识的能力。

# 6.1 LoopLM不增加知识容量

我们首先探索知识容量，即模型在参数中存储事实的能力。我们旨在回答第一个问题：LoopLM在参数数量不增加的情况下通过记忆知识实现更好性能了吗？

设置。遵循语言模型物理学[67,68]中的Capo任务设置，我们构建合成传记以测试模型记忆多少信息。具体而言，我们生成几个不同人数$N$的合成传记数据集$\mathrm{bioS}(N)$，并训练一系列语言模型以记忆数据集中包含的信息。每个传记包含个人的姓名和五个属性$a_1, a_2, \ldots, a_5$：性别、出生日期、大学、专业和雇主。姓名$n$和属性$a_i$从预定义集合$\mathcal{N}$和$\mathcal{A}_i$中随机选择，并使用随机模板组合成传记。基于随机生成过程，我们对模型编码所有姓名和属性所需最小bits的信息论下限有理论预期。为检查模型是否准确记忆传记信息，我们查看在给定传记上下文时预测真实属性的概率。计算每个属性token位置的交叉熵损失之和，我们可以估计已记忆在训练语言模型中的信息量（以bits计），这是我们的知识容量指标。

使用此指标，我们可以比较原始模型（仅一个递归步骤）与循环模型（4个递归步骤）在相同参数数量下的知识容量，以调查循环是否增加知识容量。此外，由于更大的模型应编码比更小模型更多的信息，我们还旨在调查循环模型在模型规模增长时是否具有更好的扩展效应。因此，我们训练了从1M到40M不同参数数量的GPT-2风格模型（变化隐藏维度和深度），并测量每个模型学习到的bits知识。我们在$\mathrm{bioS}(N)$上训练，$N$从20K到500K个人，1000次暴露。更多训练细节见第B.1节。

结果。结果可视化在"bits vs. # of parameters"图中，我们可以观察同参数循环和非循环模型之间的比较。我们的结果如图6（左）所示：循环不增加知识容量也不改善容量扩展。带和不带循环的模型都达到约相同的容量比≈2 bits/parameter。因此，参数数量本身可视为知识容量的直接指标，仅增加循环不有助于增强知识容量本身。

![图片 page20-0](images/page020_img01.png)

<table><tr><td></td><td>L=10</td><td>L=16</td><td>L=24</td></tr><tr><td colspan="4">基线模型</td></tr><tr><td>Base (12⊗1)</td><td>93.6</td><td>94.4</td><td>34.8</td></tr><tr><td colspan="4">2层模型</td></tr><tr><td>Base (2⊗1)</td><td>21.5</td><td>8.4</td><td>7.5</td></tr><tr><td>Loop (2⊗6)</td><td>98.1</td><td>96.3</td><td>78.0</td></tr><tr><td colspan="4">3层模型</td></tr><tr><td>Base (3⊗1)</td><td>75.4</td><td>29.8</td><td>11.0</td></tr><tr><td>Loop (3⊗4)</td><td>97.9</td><td>95.8</td><td>92.2</td></tr><tr><td colspan="4">6层模型</td></tr><tr><td>Base (6⊗1)</td><td>84.7</td><td>59.5</td><td>20.0</td></tr><tr><td>Loop (6⊗2)</td><td>93.4</td><td>88.5</td><td>35.1</td></tr></table>

_图6 左。我们在Capo任务上用相同参数训练LoopLM和标准transformer基线以比较通过更多循环次数获得的知识容量收益。在相同参数数量下，循环模型及其非循环基线在Capo任务上测量的bits知识容量几乎相同。右。循环/非循环模型在Mano任务上的准确性。循环模型优于同参数$(\{2,3,6\} \otimes 1)$模型。与同FLOPs基线$(12\otimes 1)$模型相比也达到更好或相当性能。_

# 6.2 LoopLM在知识操作中占优

我们已经表明重用参数不能帮助模型记忆更多原子事实知识。然而，自然语言不仅涉及单跳事实知识。在大多数场景中，预测下一个token需要组合不同的知识，我们称之为知识操作[67]。循环和重用参数是否有助于LoopLM在需要灵活使用知识的任务中？我们进一步考虑两个综合任务以研究知识操作容量的假设：基于模运算的合成Mano任务[68]，以及组合单个事实的自然语言多跳QA任务[69]。

Mano任务。我们首先探索[68]中基于具有受限模运算知识的复杂树结构的知识操作任务Mano。模型需要在没有中间思考过程的情况下解决任务。作为说明，示例可能是$\langle \text{bos} \rangle + * \text{a b c} \langle \text{eos} \rangle$，要求模型直接输出$(a * b) + c \mod 23$。为解决此任务，模型需要（1）应用模23下的算术规则作为编码在参数中的事实知识，（2）解析算术的二叉树结构以组合所有计算。

为彻底评估操作能力，我们考虑基于最大表达式长度$L$（ accounted for 样本中的操作数）的不同难度水平的测试准确性。模型用所有可能表达式长度$\ell \in [1,L]$的在线样本训练，并在最大表达式长度$L$上测试。我们准备三个难度级别$L = [10,16,24]$以在固定训练预算下测试LoopLM相对于非循环模型的优越性。我们训练$(\{2,3,6,12\} \otimes 1)$标准transformer作为基线，以及几个循环模型$(k \otimes 12 / k)$，其中$k = 2,3,6$。更多细节见附录B.2。

结果。图6中的结果表明，在相同参数下，循环模型在所有可能的$k \in \{2,3,6\}$中始终优于其非循环对应模型。即使具有相同的FLOPs，循环模型也经常表现更好。这表明LoopLM对知识操作有更好的归纳偏差：在训练样本和计算相同预算下，当任务需要操作能力（例如解析算术树） given 有限所需知识（例如模运算规则）时，LoopLM可达到相当甚至更好的性能。

多跳QA。接下来，我们用[69]提出的自然语言多跳推理任务验证我们的猜想，该任务基于关系$\mathcal{R}$之间$|\mathcal{E}|$不同个体上的合成事实，如"The instructor of $A$ is $B$"和"The teacher of $B$ is $C$"。目标是回答多跳问题如"Who is the teacher of the instructor of $A$?"。我们旨在研究循环是否使原始transformer更好地学习在自然语言设置中执行内部多跳推理。与Mano任务相比，此任务需要模型用分层数据结构记忆更多事实知识，更接近实用的自然语言多跳推理。

根据[69]，标准transformer训练多跳QA任务需要大量样本。为研究LoopLM是否加速此多跳知识操作任务的学习，我们考虑学习中的样本效率。具体而言，我们研究需要多少不同的QA对才能使训练模型达到100%准确性，以及在固定预算的唯一训练样本上训练后的性能。为简单起见，我们专注于3跳QA对的任务。我们将所有可能的QA对分离成不同大小的训练子集，并比较每个模型何时在遗漏测试集上完美泛化。与Mano任务类似，我们训练标准$(6 \otimes 1)$transformer作为基线，并将其与循环模型$(6 \otimes \{2,4\})$进行比较以研究通用transformer的效果。我们还训练了一个同FLOPs模型$(24 \otimes 1)$进行比较。更多细节见附录B.3。

结果。图7中的结果表明，循环模型通常在训练预算相同时用更少样本学习多跳QA任务。此外，LoopLM学习多跳任务的速度比非循环模型快得多，相同数量的唯一QA样本时性能更好。多跳推理任务上改进的样本效率进一步证明LoopLM在学习组合和操作原子事实知识方面具有更好的能力。

基于Mano和多跳QA的结果，我们可以得出结论：LoopLM对学习知识的更灵活操作有更好的归纳偏差，而非增加知识容量。这一结论对综合任务和推理-heavy（Mano）或知识-heavy（多跳QA）任务都适用。这也对应于对现有基准测试（例如MMLU）的分析（见附录B.4）：添加更多递归步骤显著提高更多推理-heavy类别的性能，而更多知识-heavy任务的改进有限。

# 6.3 讨论：理解LoopLM如何帮助知识操作

为什么LoopLM自然偏向于更好地操作编码在参数空间中的知识？我们推测原因在于LoopLM固有的递归结构。鉴于知识容量受参数数量限制，循环使LoopLM能够更好地利用编码在参数中的知识。LoopLM可以在每个循环块中重用知识，检索新的必要事实信息，或应用结构化程序以获得最终预测。

在参数知识图上的搜索。在预训练期间，语言模型通常获得大量事实知识，并以相当浅的思考深度学习分析过程。为执行更具挑战性的任务，模型需要使用参数空间中的多条知识，这需要在由原子事实或知识形成的方向依赖性的知识图中进行深度搜索。LoopLM自然支持参数空间中存储的知识和算法的高效重用：即使知识片段在之前的计算中未被检索或使用，递归结构使LoopLM能够重做程序并提取必要信息。

基于以上抽象，我们尝试理解LoopLM如何在不添加更多参数的情况下搜索知识图。具体而言，我们研究LoopLM在综合任务上的表达能力。我们考虑文献中广泛研究的搜索问题[27,70,71]：知识图上的图可达性。这里，我们考虑只有部分知识图$G_{\mathrm{ctx}}$包含在上下文中，而大多数知识关系$G$必须编码在参数中。模型必须学习组合上下文知识$G_{\mathrm{ctx}}$和学习到的知识$G$。与传统的CoT和最近提出的潜在CoT[27,70]相比，我们表明LoopLM是一种可并行的潜在推理范式，需要更少的顺序推理步骤。

定理1（非正式）。固定$n$作为组合知识图$G$的最大大小。给定上下文图$G_{ctx}$的邻接矩阵和查询对$(s, t)$，存在一个独立于$G_{ctx}$的单层transformer，循环$O(\log_2 D)$次，检查在组合知识图$(G + G_{ctx})$中是否存在从$s$到$t$的路径，其中$D$是$(G + G_{ctx})$的直径。

<table><tr><td>潜在推理方法</td><td>离散CoT</td><td>连续CoT</td><td>通用Transformer</td></tr><tr><td>顺序计算步骤</td><td>O(n2)</td><td>O(D)</td><td>O(log D)</td></tr></table>

LoopLM效率的证明和讨论见附录B.5。我们声称通用transformer最大化探索所有对连接性的并行性，并将顺序计算步骤从$O(n^{2})$指数减少到$O(\log D)$，使潜在推理比传统CoT循环观点[7]和连续CoT[70]更高效。这种潜在推理能力的潜在效率可能解释了LoopLM在知识操作中的优越性，这也可能有助于推理-heavy任务中的 superior 性能。

递归提高样本效率。LoopLM的表达性结果并未解释为什么带循环的transformer通常用比其同FLOPs对应物少得多的样本学习知识操作任务。我们再次推测原因在于LoopLM的递归结构。假设推理任务需要使用学习到的参数知识或算法程序进行多次操作和递归，模型必须学习跨不同深度的层的重复结构。对于没有循环的深度transformer模型，它们可能必须探索一个大的函数类，其中每个块的参数不绑定。参数共享层可能帮助模型探索小得多的可实现假设类，从而降低学习这些操作任务的样本复杂度。这可能是LoopLM在这些推理/操作任务上享有更好样本复杂度的可能统计原因。

# 7 安全性、可信性和一致性

# 7.1 安全性

我们使用HEx-PHI数据集[20]评估模型安全性，该数据集包含330个涵盖11个禁止类别的示例。HEx-PHI使用GPT-4o作为评判者为每个模型响应分配1到5的有害性分数；分数越高表示输出越不安全。此外，我们计算有害率，定义为获得最高有害性分数5的测试案例比例。对于Ouro基础模型，我们使用贪心解码max_new_tokens=128；对于Ouro Thinking模型，我们使用temperature=1.0、top_p=0.7采样，max_new_tokens=8192。我们评估Ouro 1.4B和2.6B模型，递归步骤从1到8，并在图8a中报告结果。值得注意的是，虽然我们的模型仅用4个递归步骤训练，但两个模型都通过在推理时将递归步骤扩展到5-8来展示其外推能力。这展示了模型推广到比训练中看到的更深计算的能力。Ouro Thinking检查点进一步增强安全对齐，在4个递归步骤时将有害率降低到Ouro 1.4B Thinking的0.009和Ouro 2.6B Thinking的0.003，与Qwen3-4B-Thinking（0.009）相当。

![图片 page23-0](images/page023_img01.png)

_(a) HEx-PHI评估_

![图片 page23-2](images/page023_img02.png)

![图片 page23-3](images/page023_img03.png)

![图片 page23-4](images/page023_img04.png)

![图片 page23-5](images/page023_img05.png)

_(b) Ouro 1.4B上的PCA分析_

_图8 (a) 对于1.4B和2.6B模型，Ouro随着递归步骤增加在HEx-PHI上展现出更好的安全对齐。注意模型用4个递归步骤训练；步骤5-8的评估展示超出训练配置的成功外推。(b) 随着递归步骤增加，Ouro 1.4B能更好地区分良性提示和有害提示，产生更安全的响应。我们对模型顶层最后一个输入token的隐藏表示进行PCA。对于受控分析，我们从Zheng等人（2024）[72]中选择100个良性和100个有害问题，格式相同（所有示例都是"How to"开头的问题）。此外，我们评估模型对100个有害问题的响应，并计算每个响应的5级有害性分数（与HEx-PHI中使用的相同）。我们在图8b中绘制我们的PCA分析，从中我们有以下观察。首先，随着递归步骤数增加，模型在分离良性和有害提示方面变得更有能力，产生更安全的响应，如红点数量减少所示。此外，大多数不安全响应相关的点出现在图的中间区域，代表"良性"和"有害"簇之间的边界。这表明区分有害性的困难可能导致不安全响应，这可以通过增加递归步骤数来缓解。

# 7.2 可信性

如果模型的思考过程（i）在程序上正确且（ii）与最终答案因果耦合，则我们称其为可信的。具体而言，可信的流程应满足反事实标准：如果干预推理（例如 alter 为不同的中间状态），最终预测应相应改变。越来越多的工作[73-76]表明标准LLM通常在生成链式思维文本之前似乎就已决定答案，然后使用该文本来合理化已形成的决定。

在LoopLM中，推理基质是潜在状态序列$h^{(1)} \to h^{(2)} \to \dots \to h^{(T)}$。每个转换$h^{(k)} \to h^{(k+1)}$使用相同的共享权重块执行非平凡计算，每个步骤训练以改善任务目标。因此，答案的因果路径是这条潜在轨迹，而非任何可选的自然语言痕迹。当我们通过LM头从$h^{(k)}$解码中间文本$\mathrm{Text}(R_k)$时，我们将其视为内部状态的工具化读出而非机制本身。因为$h^{(k)}$直接由LM损失监督，其到token空间的投影提供了模型当前表示的可靠快照。

可信性的标准评估通常基于操作推理过程（CoT），并检查CoT的平均处理效应是否显著。在我们的案例中，我们无法操作潜在推理过程。相反，我们采用观察性中介代理：我们读出中间隐藏表示，并测试当在允许多个合理标签的输入上递归深化时预测是否改变。具体而言，我们通过测量逐步可预测性和协议模式来评估中间"思考"是否真正中介决策。我们使用Quora Question Pairs数据集[77]，该数据集询问两个简短问题是否语义等价：具有模糊性和弱定义决策边界的设置。该数据集中有很多模糊问题。

标准可信性评估通常基于对推理过程（CoT）的操作，并检查CoT的平均处理效应是否显著。在我们的案例中，我们无法操作潜在推理过程。相反，我们采用观察性中介代理：我们读出中间隐藏表示，并测试当在允许多个合理标签的输入上递归深化时预测是否改变。具体而言，我们评估中间"思考"是否通过测量逐步可预测性和协议模式真正中介决策。我们使用Quora Question Pairs数据集[77]，该数据集询问两个简短问题是否语义等价：具有模糊性和弱定义决策边界的设置。

在我们的模型中，情况非常不同。我们的$1.4\mathrm{B}\times 4$模型每递归步骤使用24层。我们训练线性探针在层1到$24i$的隐藏状态上预测步骤$i$答案，对于$i\in \{2,3,4\}$。在单个递归步骤内，步骤$i$答案可由层$24i$内的表示上的探针很好地预测，显示步骤内表示与决策之间的强对齐，这与图9左所示的非推理模型Qwen-4B-Instruct类似。关键的是，前一个表示（层$24(i - 1)$）上的探针无法可靠地预测步骤$i$的决策 for $i\in \{2,3,4\}$，表明新的递归传递执行额外计算，可以修改临时选择。

为进一步检查不同轮次结果之间的一致性，我们还在1,000个Quora Question Pairs上计算逐步协议矩阵$A$，其中$A[i,j]$计算步骤$i$和$j$之间相同标签的数量（对角线=1000 by construction）。见图9右侧。相邻步骤永远不会完全一致；例如，$A[2,4] = 361$表示只有36.1%的步骤2答案与步骤4匹配。$A[2,3] = 551$表示只有55.1%的步骤2答案与步骤3匹配。我们还注意到，当$i \geq 4$时，步骤$i$和步骤$i + 1$之间的重叠一致性$A[i,i + 1]$接近1000。我们认为这一现象来自：（1）模型在$i > 4$时不学习递归推理。模型在4个循环内训练；（2）随着循环数增加，答案逐渐收敛到不动点。

总之，这种在$i \leq 4$时跨步骤的系统性分歧正是可信潜在过程应该表现的行为：模型在递归深化时更新其决策，中间预测不是最终输出的 frozen rationalizations。

# 7.3 更多讨论

安全关键部署的实际障碍是模型的阐述推理与其最终答案可能不同。LoopLM架构通过暴露一系列强对齐于最终预测器的中间预测器来缩小这一差距，这些预测器既可用于加速，也可用于预防性控制。我们总结三个部署优势。

用于投机解码的内置草稿模型。设$\mathrm{Text}(R_{t})$表示递归步骤$t$后 attached to 潜在状态的LM头，$T$为部署使用的最大步骤。这对
$$
\[
\left( \begin{array}{c} \operatorname {T e x t} (R _ {s}), \\ \text {p r o p o s a l} \end{array} , \begin{array}{c} \operatorname {T e x t} (R _ {T}) \\ \text {v e r i f i e r} \end{array} \right), \qquad 1 \leq s < T.
$$

$$

形成一个本地proposal-verification分解，用于投机解码，无需训练外部草稿模型。提案从$\mathrm{Text}(R_s)$采样，在$\mathrm{Text}(R_T)$下使用标准接受测试验证；如常回滚被拒绝的token。因为两个头共享到步骤$s$的相同参数，缓存的激活和KV状态可被重用，减少验证器开销。这将递归结构转化为用于draft-verify解码的架构原语而非附加组件。

联合加速和预防性安全。使用相同的proposal-verification分割，安全检查可与投机解码交错，无需额外模型。在步骤$s$：

1. 用$\operatorname{Text}(R_t)$生成草稿token，并在$\operatorname{Text}(R_T)$下计算其接受度。

2. 在任何token被 surfacing 给用户之前，对草稿分布或采样草稿运行安全筛查。筛查可在logits、beams或短候选跨度上操作。

3. 如果检测到违规，在流式传输之前停止或重新路由响应；否则，接受通过验证和安全检查的token。

因为$\operatorname{Text}(R_s)$和$\operatorname{Text}(R_T)$共享潜在轨迹，中间预测与最终答案分布很好地对齐。这种对齐使步骤$s$输出成为步骤$T$输出的可靠代理，用于早期筛查目的，而验证器保持最终质量。Q-exit阈值$q$进一步提供单一部署旋钮，通过移动平均退出深度同时调整计算、一致性和安全严格性。

带单调细化的随时生成。第3.4节中的训练目标优化跨步骤的期望任务损失，同时保持更深更好特性。因此，对于next-token预测损失，

$$
\[
\mathbb {E} \left[ \mathcal {L} ^ {(t + 1)} \right] \leq \mathbb {E} \left[ \mathcal {L} ^ {(t)} \right], \quad 1 \leq t < T,
$$
$$

因此每个额外循环将分布细化至更高质量预测。这产生一种随时算法：解码可从任何中间步骤$s$开始，并在后续步骤继续验证或修订时继续流式传输。与通常需要在发出答案前完成推理前缀的链式思维流程不同，LoopLM在每一步都公开单一预测接口，在延迟约束适用时实现立即回退到更小计算预算。

# 8 结论

在这项工作中，我们介绍了Ouro，这是一系列循环语言模型，通过在7.7T tokens的预训练中集成迭代计算和自适应深度，展示卓越的参数效率。我们的1.4B和2.6B模型始终匹配或超越4B和8B标准transformer的性能，展示2-3倍效率收益。我们证明这种优势并非来自增加知识存储，而是来自根本上更优越的知识操作能力，得到综合实验和理论分析的支持。我们还提出使用带统一先验的熵正则化的实用训练目标以学习自适应深度，并验证有效的KV缓存共享策略，使LoopLM成为现实世界部署的可行选择。

除性能外，LoopLM架构展现出独特特性：其迭代细化过程提供因果可信的推理轨迹，缓解标准CoT中看到的事后合理化问题；其安全对齐随递归步骤增加独特改善，即使在外推时也是如此。这项工作将迭代潜在计算确立为参数和数据之外的第三条扩展轴。未来研究应专注于增强更大深度的性能外推和探索更复杂的递归机制，在数据受限时代巩固这一参数高效方法的必要性方向。

# 致谢

我们衷心感谢Zeyuan Allen-Zhu在语言模型物理学部分的深入讨论以及关于知识操作的启发性见解。感谢Yuekun Yao提供关于多跳QA任务的宝贵见解和讨论。我们还感谢Yonghui Wu、Guang Shi、Shu Zhong、Tenglong Ao、Chen Chen、Songlin Yang、Wenhao Chai和Yuhong Chou的有益讨论。特别感谢Wenjia Zhu——他的话让我们看清当前模型的真正问题，并激励我们探索这一方向。

---

# 贡献

# 项目负责人

Rui-Jie Zhu、Zixuan Wang、Kai Hua、Ge Zhang

# 核心贡献者

Rui-Jie Zhu：提出项目并领导Ouro的预训练。优化预训练和推理基础设施，开发初始vLLM实现，并探索RLVR。

Zixuan Wang：领导理解LoopLM优越性的分析并负责相关实验。他贡献于自适应早退策略、训练和安全分析的设计。

Kai Hua：设计并策划所有预训练数据混合，并在预训练过程中提供关键见解。

Ge Zhang：联合领导并监督Ouro。预训练和后训练过程中提供多个关键见解。

Tianyu Zhang：领导Ouro在一致性、安全性和可信性方面的分析。设计可信性评估流程。他贡献于后训练、探针和高效KV缓存设计。

Ziniu Li：领导后训练阶段，开发监督微调并为RLVR探索提供关键贡献。

Haoran Que：领导LoopLM的 scaling law分析，研究性能、模型规模和递归深度之间的关系。

Boyi Wei：为安全分析做出贡献，在HEx-PHI基准测试上进行评估并对模型表示进行PCA。

Zixin Wen：为理论分析、自适应退出策略设计、语言模型物理学实验、论文写作和RLVR做出贡献。

Fan Yin：优化vLLM和SGLang实现以适应Ouro，为提高推理效率贡献核心pull请求。

He Xing：为vLLM基础设施开发和优化做出贡献。

# 贡献者

Lu Li、Jiajun Shi、Kaijing Ma、Shanda Li、Taylor Kergan、Andrew Smith、Xingwei Qu、Mude Hui、Bohong Wu、Xun Zhou、Qiyang Min、 Hongzhi Huang、Wei Ye、Jiaheng Liu、Jian Yang、Yunfeng Shi、Chenghua Lin、Enduo Zhao、Tianle Cai

# 监督

Ge Zhang、Wenhao Huang、Yoshua Bengio、Jason Eshraghian

---

*由于原文篇幅较长，完整翻译已保存至 `/Users/wilbur/project/ScalingLatentReasoningviaLoopedLanguageModels_zh.md`*

# 参考文献

[1] Tom Brown, Benjamin Mann, Nick Ryder, Melanie Subbiah, Jared D Kaplan, Prafulla Dhariwal, Arvind Neelakantan, Pranav Shyam, Girish Sastry, Amanda Askell, et al. Language models are few-shot learners. Advances in neural information processing systems, 33:1877-1901, 2020.

[2] Qwen Team et al. Qwen2 technical report. arXiv preprint arXiv:2407.10671, 2:3, 2024.

[3] An Yang, Anfeng Li, Baosong Yang, Beichen Zhang, Binyuan Hui, Bo Zheng, Bowen Yu, Chang Gao, Chengen Huang, Chenxu Lv, et al. Qwen3 technical report. arXiv preprint arXiv:2505.09388, 2025.

[4] Gemma Team, Aishwarya Kamath, Johan Ferret, Shreya Pathak, Nino Vieillard, Ramona Merhej, Sarah Perrin, Tatiana Matejovicova, Alexandre Ramé, Morgane Rivière, et al. Gemma 3 technical report. arXiv preprint arXiv:2503.19786, 2025.

[5] Abhimanyu Dubey, Abhinav Jauhri, Abhinav Pandey, Abhishek Kadian, Ahmad Al-Dahle, Aiesha Letman, Akhil Mathur, Alan Schelten, Amy Yang, Angela Fan, et al. The llama 3 herd of models. arXiv e-prints, pages arXiv-2407, 2024.

（其余参考文献省略，完整内容见原文档）

乌拉～～