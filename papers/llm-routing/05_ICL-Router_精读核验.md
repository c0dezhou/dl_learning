# ICL-Router：给每个模型一份 500 道题的“成绩册”，让 7B 路由器自己学会读

> 英文题名：**ICL-Router: In-Context Learned Model Representations for LLM Routing**  
> 来源：[arXiv:2510.09719v3（英文原文）](https://arxiv.org/html/2510.09719v3)。  
> 核验依据：库中 `ICL-Router.pdf`，§3–§4、图 1–5、表 1–4、附录 A.2–A.5。  
> 标记：**[原文]** 直接证据；**[推导]** 根据方法/公式解释；**[待实验]** 候选问题；**[待文献]** 需检索最新相关工作。

## 原文定位导航（本次增补）

- 中译依据：Library 中现有 `ICL-Router_中文译本_含完整图表公式.pdf`，15 页，非官方中文学习译本，首页明确按 v3 翻译。
- 英文核验：`ICL-Router.pdf`，11 页；首页为 **arXiv:2510.09719v3 [cs.LG], 14 Nov 2025**。[固定版本原文 PDF](https://arxiv.org/pdf/2510.09719v3#page=1)。
- 页码口径：下文“中译 PDF 第 N 页”和“原文 PDF 第 N 页”都指从首页起 **1-based 物理页**。中译正文页脚1–11及附录14–15与物理页一致；中译第12–13页标作 References 1/2、2/2，不能把它们当全文页码1/2。英文页码按文件物理位置计，不与中译混用。
- 原文章号、式号、图表号均沿用英文 v3。附录 A.1 的数据表在英文重新编号为 **Table 1**（物理第10页），中译叫 **表A.1**（物理第14页），与正文 Table 1 区分。中译第9–10页的图数值对照表是译者整理，主要数值已经对照英文原图。
- “原文事实”只表示作者写明；“本文讲解/推导”是解释或公式推论；“范围核验”限于所列本版章节；“待实验/待文献”尚未成立。Ctrl+F中文词用于中译，英文词用于原文。

| 要找什么 | 中译物理页 | 英文一键定位 | 章、式、图表或关键词 |
|---|---:|---|---|
| QRT与重构损失 | 4 | [原文第3–4页](https://arxiv.org/pdf/2510.09719v3#page=3) | §3.2；Eq.(1–3) |
| 能力档案、Yes/No预测与接入 | 4–5 | [原文第4页](https://arxiv.org/pdf/2510.09719v3#page=4) | §3.3–3.4；Eq.(4–6) |
| 冻结8B embedding、7B router与训练 | 5–6 | [原文第4–5页](https://arxiv.org/pdf/2510.09719v3#page=4) | §4.2；Implementation Settings |
| 初始8模型与ID/OOD主表 | 6–7 | [原文第5页](https://arxiv.org/pdf/2510.09719v3#page=5) | Tables 1–2 |
| 新增5模型、76.3→79.9 | 8–9 | [原文第6页](https://arxiv.org/pdf/2510.09719v3#page=6) | §4.5；Figures 2–3 |
| 4096维、题数消融、QRT消融 | 10–11 | [原文第7页](https://arxiv.org/pdf/2510.09719v3#page=7) | Table 3；Figures 4–5；Table 4 |
| 数据规模、500题选法、开销与局限 | 14–15 | [原文第10–11页](https://arxiv.org/pdf/2510.09719v3#page=10) | Appendix A.1–A.5；125；8 tokens |


## 先用一个故事理解

有 8 个擅长领域各异的同学，你不知道谁适合某道新题。传统方法可能把每个人压成一个“数学 80、编程 70”的短向量。ICL-Router 另辟一条路：**让每个人完成同一套 500 道有区分度的题，保留“什么题、答对还是答错”的记录，交给一个经过训练的 LLM 路由器阅读。**

【定位·原文设定＋教学类比：中译 PDF 第 5 页；原文 §3.3 ICL Model Routing Training，Eq.(4–5)，[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“能力画像”／原文“capability profile”；另见：中译 PDF 第 15 页；原文 Appendix A.3 Challenging Query Set Construction，[原文 PDF 第 11 页](https://arxiv.org/pdf/2510.09719v3#page=11)；Ctrl+F：中译“125”／原文“125”。8位同学、数学80/编程70均为自拟比喻，非论文数据。】

一个模型在逻辑题上对得多，另一个在数学题上对得多，7B 路由器不必先得到明确命名的“逻辑能力维度”，它尝试从整份成绩册中学习模式，然后判断新题交给谁。

【定位·本文讲解：中译 PDF 第 5 页；原文 §3.3 ICL Model Routing Training，Eq.(4–5)，[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“能力画像”／原文“capability profile”；另见：中译 PDF 第 5 页；原文 §4.2 Implementation Settings（QRT设置），[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“两层”／原文“two-layer”。逻辑与数学成绩模式为示意；7B模型型号是作者设置。】

亮点是**新模型只需要完成同一套题并建立成绩册，不用重新训练核心路由器**。代价是：路由时要运行一个相对昂贵的 7B 路由器，并读取一套比 IRT 向量大得多的能力表示。本文几乎不研究费用—质量权衡，主要追求准确率。

【定位·原文事实＋本文开销判断：中译 PDF 第 5 页；原文 §3.4 Inference and Scalable Model Incorporation，Eq.(6)，[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“同一个查询集合”／原文“same query set”；另见：中译 PDF 第 15 页；原文 Appendix A.4 Cost Evaluation，[原文 PDF 第 11 页](https://arxiv.org/pdf/2510.09719v3#page=11)；Ctrl+F：中译“8 个词元”／原文“8 tokens”。7B运行与档案处理是结构事实；“相对昂贵”未有本文与IRT的同硬件量化对照，不作为实验胜负结论。】

## ① 候选空间：初始 8 个接近同尺寸模型，再加入 5 个

[原文，§4.2、表 1]

**初始 8 个模型**：DeepSeek-R1-Distill-Qwen-7B、Llama-3.1-8B-Instruct、Llama-3.1-Nemotron-Nano-8B、Qwen2.5-7B-Instruct、cogito-v1-preview-llama-8B、Gemma-2-9B-IT、InternLM3-8B-Instruct、GLM-4-9B-Chat。它们大多为 7B–9B 级别的开源模型。

【定位·原文事实：中译 PDF 第 6 页；原文 Table 1（ID准确率与原始8模型），[原文 PDF 第 5 页](https://arxiv.org/pdf/2510.09719v3#page=5)；Ctrl+F：中译“76.30”／原文“76.30”。表1的正式型号为Llama-3.1-Nemotron-Nano-8B-v1。】

**逐步加入的 5 个新模型**：[原文，§4.5] Falcon-H1-7B-Instruct、Gemma3-12B-IT、DeepSeek-R1-Llama-8B、OpenThinker3-7B、AceReason-Nemotron-1.1-7B。

【定位·原文事实：中译 PDF 第 8–9 页；原文 §4.5 Scalability to New Models，Figures 2–3，[原文 PDF 第 6 页](https://arxiv.org/pdf/2510.09719v3#page=6)；Ctrl+F：中译“Gemma3-12B”／原文“Gemma3-12B”。新增列表位于§4.5文字，不在初始8模型Table 1中。】

所以旧笔记“8 个都是 7–9B”可以作为初始池描述；**不能说加入新模型后所有候选仍都为 7–9B**，因为新模型中明确包含 12B。

【定位·本文集合范围澄清：中译 PDF 第 6 页；原文 Table 1（ID准确率与原始8模型），[原文 PDF 第 5 页](https://arxiv.org/pdf/2510.09719v3#page=5)；Ctrl+F：中译“76.30”／原文“76.30”；另见：中译 PDF 第 8–9 页；原文 §4.5 Scalability to New Models，Figures 2–3，[原文 PDF 第 6 页](https://arxiv.org/pdf/2510.09719v3#page=6)；Ctrl+F：中译“Gemma3-12B”／原文“Gemma3-12B”。12B是新增Gemma3，不是初始八模型之一。】

**重要实验边界**：作者选择的是小参数规模相近的模型池，并未展示强烈不同价位的闭源 API 模型之间的成本最优化。因此这篇论文的目标不是 ZeroRouter 那样同时最优化质量、美元和延迟。[原文，附录 A.5]

【定位·原文范围＋跨论文目标对照：中译 PDF 第 15 页；原文 Appendix A.5 Limitations，[原文 PDF 第 11 页](https://arxiv.org/pdf/2510.09719v3#page=11)；Ctrl+F：中译“小参数规模”／原文“small-parameter”；另见：中译 PDF 第 5 页；原文 §3.4 Inference and Scalable Model Incorporation，Eq.(6)，[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“同一个查询集合”／原文“same query set”；另见：另文 `ZeroRouter_Chinese_Full_Translation.pdf`：中译 PDF 第 7 页；原文 Multi-Objective Optimization，Eq.(17–18)；中译§3.3.1，[原文 PDF 第 4 页](https://arxiv.org/pdf/2601.06220v1#page=4)；Ctrl+F：中译“可选全局约束”／原文“optional global”。A.5只承认小参数池；没有API价格跨度实验是本版实验范围核验，不是A.5逐字结论。】

## ② 题目表示：第一阶段为什么要“复述题目”？

[原文，§3.2、§4.2]

对每道题 `q`，先用**冻结的 `Qwen3-Embedding-8B`**编码为约 **4096 维**语义向量 `e(q)`，再由**两层 MLP 投影器**映射到 `Qwen2.5-7B-Instruct` 路由器的输入表示空间：

【定位·原文事实：中译 PDF 第 5 页；原文 §4.2 Implementation Settings（QRT设置），[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“两层”／原文“two-layer”；另见：中译 PDF 第 10 页；原文 §4.6，Table 3（8B/4096），[原文 PDF 第 7 页](https://arxiv.org/pdf/2510.09719v3#page=7)；Ctrl+F：中译“8B/4096”／原文“8B/4096”。4096维在Table 3而不是§4.2明确列出；embedding冻结，路由器并非冻结。】

$$
\mathbf e=f_{emb}(q),\qquad\mathbf v=f_{proj}(\mathbf e).
$$

【定位·原文公式改记号：中译 PDF 第 4 页；原文 §3.2 Query Reconstruction Training，Eq.(1–3)，[原文 PDF 第 3–4 页](https://arxiv.org/pdf/2510.09719v3#page=3)；Ctrl+F：中译“重构损失”／原文“reconstruction loss”。对应Eq.(1–2)。】

为什么不直接把 4096 维向量塞给路由器？因为 embedding 模型的坐标系和 7B 自回归模型的内部词向量空间不一定一致。**第一阶段 Query Reconstruction Training（QRT）就像让路由器学习一种“向量方言”**：投影器给它一个题目向量，它要把原题文本重新生成出来。这不是研究目标本身，而是让投影器和路由器掌握题目语义的预训练任务。

【定位·原文动机＋教学类比：中译 PDF 第 4 页；原文 §3.2 Query Reconstruction Training，Eq.(1–3)，[原文 PDF 第 3–4 页](https://arxiv.org/pdf/2510.09719v3#page=3)；Ctrl+F：中译“重构损失”／原文“reconstruction loss”。“向量方言”是本笔记比喻，不是论文术语。】

[原文，Eq.(3)] 重建阶段的损失是逐 token 自回归交叉熵，目标是原问题中的真实词：

$$
\mathcal L_{rec}=-\frac{1}{N}\sum_n\frac{1}{T_n}\sum_{t=1}^{T_n}
\log P(q_n^{(t)}\mid q_n^{(<t)},\mathbf v_n).
$$

【定位·本文规范化示意，非原式照抄：中译 PDF 第 4 页；原文 §3.2 Query Reconstruction Training，Eq.(3)，[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“重构损失”／原文“reconstruction loss”。原文Eq.(3)将1/(N T_n)放在求和外。这里将1/T_n放入n求和，是解释性改写；不能据此声称loss reduction与原实现相同。】

上面按“每题平均 token 损失、再平均题目”写清常见规范化意图；论文印刷公式的前因子排版略紧凑，**复现时应以实际代码的数据和 token mask 为准**，不要仅凭这个说明重写完全一致的 loss reduction。

【定位·范围核验／待代码：中译 PDF 第 4 页；原文 §3.2 Query Reconstruction Training，Eq.(3)，[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“重构损失”／原文“reconstruction loss”。已目视原文第4页Eq.(3)；归一化歧义客观存在，尚未查看实现的mask与reduction。】

训练配置：[原文，§4.2] 先单独训练 projector **1 个 epoch**（学习率 `2×10⁻⁵`），再与 7B 路由器联合训练 **2 个 epoch**（路由器学习率 `5×10⁻⁶`）。上述 1+2 epoch 都属于第一阶段 QRT；第二阶段另外训练 5 个 epoch，才学“该不该选这个候选模型”。

【定位·原文事实＋校订：中译 PDF 第 5 页；原文 §4.2 Implementation Settings（QRT设置），[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“两层”／原文“two-layer”。1 epoch仅投影器、随后2 epoch联合训练均在第一阶段QRT内；真正第二阶段见中译第6页/原文第5页的5 epochs。】

## ③ 模型表示：不是 25 维 θ，而是同一套 500 条 Yes/No 记录

[原文，§3.3、Eq.(4)] 每个模型 M 的档案为：

$$
\mathbf P_M=\big((\mathbf v_1,c_{1M}),\ldots,(\mathbf v_{500},c_{500,M})\big),
\qquad c_{kM}\in\{\text{Yes},\text{No}\}.
$$

【定位·原文公式代入实验K值：中译 PDF 第 5 页；原文 §3.3 ICL Model Routing Training，Eq.(4–5)，[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“能力画像”／原文“capability profile”；另见：中译 PDF 第 15 页；原文 Appendix A.3 Challenging Query Set Construction，[原文 PDF 第 11 页](https://arxiv.org/pdf/2510.09719v3#page=11)；Ctrl+F：中译“125”／原文“125”。Eq.(4)写一般K；K=500来自§4.2与A.3，不是架构恒定长度。】

其中 `v_k` 是第 k 道测试题被 embedding+projector 变换后的向量，`Yes/No` 表示模型回答是否正确。**所有模型使用相同的 500 道题**，这很重要：因为分母、考题、比较尺度一致，成绩记录可对齐。

【定位·原文事实＋本文解释：中译 PDF 第 5 页；原文 §3.3 ICL Model Routing Training，Eq.(4–5)，[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“能力画像”／原文“capability profile”；另见：中译 PDF 第 5 页；原文 §3.4 Inference and Scalable Model Incorporation，Eq.(6)，[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“同一个查询集合”／原文“same query set”；另见：中译 PDF 第 15 页；原文 Appendix A.3 Challenging Query Set Construction，[原文 PDF 第 11 页](https://arxiv.org/pdf/2510.09719v3#page=11)；Ctrl+F：中译“125”／原文“125”。same query set Q写在§3.4；“比较尺度可对齐”为本文解释。】

### 这 500 道题是怎么挑出来的？

[原文，附录 A.3] 不是简单挑“8 个模型分歧最大”的前 500，也不是从全部训练题中简单均匀随机得到的题单。作者从旧池 8 个模型的训练题表现中建立四组：

【定位·原文事实＋边界：中译 PDF 第 15 页；原文 Appendix A.3 Challenging Query Set Construction，[原文 PDF 第 11 页](https://arxiv.org/pdf/2510.09719v3#page=11)；Ctrl+F：中译“125”／原文“125”。原文只规定每组125道，没有披露每组内部究竟如何抽取；不能由此排除组内随机采样。】

| 一道题被 8 个旧模型中几人答对 | 抽取数量 |
|---|---:|
| **恰好 1 人** | 125  【定位·原文数值：中译 PDF 第 15 页；原文 Appendix A.3 Challenging Query Set Construction，[原文 PDF 第 11 页](https://arxiv.org/pdf/2510.09719v3#page=11)；Ctrl+F：中译“125”／原文“125”】 |
| **恰好 2 人** | 125  【定位·原文数值：中译 PDF 第 15 页；原文 Appendix A.3 Challenging Query Set Construction，[原文 PDF 第 11 页](https://arxiv.org/pdf/2510.09719v3#page=11)；Ctrl+F：中译“125”／原文“125”】 |
| **恰好 3 人** | 125  【定位·原文数值：中译 PDF 第 15 页；原文 Appendix A.3 Challenging Query Set Construction，[原文 PDF 第 11 页](https://arxiv.org/pdf/2510.09719v3#page=11)；Ctrl+F：中译“125”／原文“125”】 |
| **恰好 4 人** | 125  【定位·原文数值：中译 PDF 第 15 页；原文 Appendix A.3 Challenging Query Set Construction，[原文 PDF 第 11 页](https://arxiv.org/pdf/2510.09719v3#page=11)；Ctrl+F：中译“125”／原文“125”】 |
| **总计** | **500**  【定位·原文数值：中译 PDF 第 15 页；原文 Appendix A.3 Challenging Query Set Construction，[原文 PDF 第 11 页](https://arxiv.org/pdf/2510.09719v3#page=11)；Ctrl+F：中译“125”／原文“125”】 |

这意味着**恰好 5、6、7、8 人答对的题，以及 0 人答对的题，都不在这套选题规则里**。论文认为只要大家都对或都错，往往难以区分能力；但为什么不选“恰好 5–7 人答对”的题？作者没有用系统的对照充分解释。它是一个可验证的设计选择，不等于所有分歧题都同等有价值。[原文 + 推导]

【定位·原文规则的推论＋范围核验：中译 PDF 第 15 页；原文 Appendix A.3 Challenging Query Set Construction，[原文 PDF 第 11 页](https://arxiv.org/pdf/2510.09719v3#page=11)；Ctrl+F：中译“125”／原文“125”。排除0及5–8由所列四组推出；为什么排除5–7以及它们是否更有效，A.3未给对照，本文保留为问题。】

相比其他论文，这个“模型表示”的信息量较大：IRT/RADAR 几十个或单个参数，ICL 则保留 500 个题目语义和答题结果，使 7B 网络自己综合。**表达力可能更强，但读取成本随示例数和模型池增长。**

【定位·跨论文结构对照＋待实验：中译 PDF 第 5 页；原文 §3.3 ICL Model Routing Training，Eq.(4–5)，[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“能力画像”／原文“capability profile”；另见：中译 PDF 第 15 页；原文 Appendix A.3 Challenging Query Set Construction，[原文 PDF 第 11 页](https://arxiv.org/pdf/2510.09719v3#page=11)；Ctrl+F：中译“125”／原文“125”；另见：另文 `IRT-Router_中文译文_含图表与公式.pdf`：中译 PDF 第 10 页；原文 IRT-Router §5.5 Implementation Details，[原文 PDF 第 7 页](https://arxiv.org/pdf/2506.01048v2#page=7)；Ctrl+F：中译“25”／原文“25”；另见：另文 `RADAR_中文译本_正文.pdf`：中译 PDF 第 6–7 页；原文 RADAR §3.5，Eq.(7–9)，[原文 PDF 第 6–7 页](https://arxiv.org/pdf/2509.25426v3#page=6)；Ctrl+F：中译“当前能力估计”／原文“Fisher information”。25维及标量分别来自IRT-Router与RADAR；更强表达力、实际读取开销增幅均非本文已测结论，需控制缓存/批处理。】

## ④ 性能预测：不是 IRT 公式，而是 7B 路由器判断 Yes 的概率

[原文，§3.3–§3.4、Eq.(5)(6)] 第二阶段把某个模型的 `500 条成绩册 + 当前问题的向量` 输给 `Qwen2.5-7B-Instruct`，训练它判断：“这个模型回答这条新题，会正确吗？”

【定位·原文事实：中译 PDF 第 5 页；原文 §3.3 ICL Model Routing Training，Eq.(4–5)，[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“能力画像”／原文“capability profile”；另见：中译 PDF 第 5 页；原文 §3.4 Inference and Scalable Model Incorporation，Eq.(6)，[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“同一个查询集合”／原文“same query set”；另见：中译 PDF 第 5 页；原文 §4.2 Implementation Settings（QRT设置），[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“两层”／原文“two-layer”。Eq.(5)为分类交叉熵，Eq.(6)以Yes概率选择模型。】

$$
\widehat P(M,q)=P_{router}(\text{Yes}\mid\mathbf P_M,q),
\qquad
M^*(q)=\arg\max_M\widehat P(M,q).
$$

【定位·原文决策式的解释性记号：中译 PDF 第 5 页；原文 §3.4 Inference and Scalable Model Incorporation，Eq.(6)，[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“同一个查询集合”／原文“same query set”。能力画像P_M写入条件是根据§3.4文字展开，原式条件写M_t,q′。】

它**并非在测试时先让全部候选 LLM 试答当前题**；7B 路由器读的是事先建立的档案和当前题表示。只执行最终选择的候选生成正式答案。

【定位·原文流程解读：中译 PDF 第 5 页；原文 §3.4 Inference and Scalable Model Incorporation，Eq.(6)，[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“同一个查询集合”／原文“same query set”。区分预先能力测评和当前查询正式生成；不是让候选全部作答后再选。】

[原文，§4.2] 第二阶段训练 **5 个 epoch**，同时更新投影器和路由器，二者学习率分别 `1×10⁻⁵` 和 `2×10⁻⁶`，batch 32。不要误记成“仅训练分类头，7B 冻结”。

【定位·原文事实：中译 PDF 第 6 页；原文 §4.2 Implementation Settings（续页），[原文 PDF 第 5 页](https://arxiv.org/pdf/2510.09719v3#page=5)；Ctrl+F：中译“五个epoch”／原文“five epochs”。训练描述接在原文第4页之后的第5页，五个epoch属第二阶段。】

### 一个简单的理解例子（不是原文数据）

问：“证明一段程序在边界输入下不会死循环”。模型 A 的成绩册显示它在程序验证、逻辑推理相似题上大多正确；模型 B 虽然普通常识题正确率更高，在这些题上却经常错。7B 路由器可能输出 `P(Yes|A,q)=0.78`、`P(Yes|B,q)=0.42`，于是选择 A。这里 0.78/0.42 是模型直接输出的预测概率，不是从 a、b、θ 的显式公式计算。

【定位·教学示例：中译 PDF 第 5 页；原文 §3.4 Inference and Scalable Model Incorporation，Eq.(6)，[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“同一个查询集合”／原文“same query set”。程序验证情境及0.78/0.42均非论文数据，也不证明概率已经校准。】

## ⑤ 成本：本文没有优化 API 费用，不能用“8 token”概括全部开销

[原文，§3.4、附录 A.4] 论文的选模型公式是 `argmax P(Yes)`，**没有在路由打分里加入输入价、输出价或推理延迟**。作者讨论开销时称，8 个候选模型大约只需要输出 8 个分类 token，相比正式 LLM 回答往往输出数百/数千 token，开销可以接受。

【定位·原文事实＋作者开销主张：中译 PDF 第 5 页；原文 §3.4 Inference and Scalable Model Incorporation，Eq.(6)，[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“同一个查询集合”／原文“same query set”；另见：中译 PDF 第 15 页；原文 Appendix A.4 Cost Evaluation，[原文 PDF 第 11 页](https://arxiv.org/pdf/2510.09719v3#page=11)；Ctrl+F：中译“8 个词元”／原文“8 tokens”。A.4的acceptable range是作者定性说明，不是美元/延迟测量表。】

**重要区别 [推导]**：

- **生成 8 个 token**，说的是分类器最后写出的答案 token 数。
- **计算 7B 模型对 8 个候选的条件概率**，还涉及 embedding、投影、每个候选 500 个示例向量的上下文处理和大量前向算力。

【定位·本文计算流程推导：中译 PDF 第 5 页；原文 §4.2 Implementation Settings（QRT设置），[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“两层”／原文“two-layer”；另见：中译 PDF 第 5 页；原文 §3.3 ICL Model Routing Training，Eq.(4–5)，[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“能力画像”／原文“capability profile”；另见：中译 PDF 第 15 页；原文 Appendix A.4 Cost Evaluation，[原文 PDF 第 11 页](https://arxiv.org/pdf/2510.09719v3#page=11)；Ctrl+F：中译“8 个词元”／原文“8 tokens”。每候选条件评估的表达不限定必须串行执行独立前向；缓存/批处理实现需另查。】

输出 token 很少，**并不等于路由运行只消耗 8 token 对应的全部计算量**。论文没有提供足以直接量化这些 prefill 与 GPU 占用的完整延迟/美元成本表。因此不能把“成本低”当作严格实验结论，也不能凭此直接声称其开销必然高于某个其他算法，须同设备实测。[原文范围 + 推导]

【定位·范围核验＋待实验：中译 PDF 第 15 页；原文 Appendix A.4 Cost Evaluation，[原文 PDF 第 11 页](https://arxiv.org/pdf/2510.09719v3#page=11)；Ctrl+F：中译“8 个词元”／原文“8 tokens”。检查本版§4 Tables 1–4、Figures 2–5及A.4：结果为准确率、结构和定性token说明，未给完整prefill/GPU/美元测量。不能推断实际高于特定算法。】

## ⑥ 决策：只最大化“谁更可能答对”

[原文，Eq.(6)]

$$
M^*(q)=\arg\max_{M\in\mathcal M}\widehat P(M,q).
$$

【定位·原文公式改记号：中译 PDF 第 5 页；原文 §3.4 Inference and Scalable Model Incorporation，Eq.(6)，[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“同一个查询集合”／原文“same query set”。对应Eq.(6)。】

没有 α、β 的质量—成本权重，没有每题预算上限，没有延迟参数。因此**它与 RADAR、ZeroRouter、IRT-Router 不直接是同一个目标函数**。在比较“谁最好”时需要先把目标对齐：

【定位·原文目标＋本文比较原则：中译 PDF 第 5 页；原文 §3.4 Inference and Scalable Model Incorporation，Eq.(6)，[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“同一个查询集合”／原文“same query set”；另见：另文 `ZeroRouter_Chinese_Full_Translation.pdf`：中译 PDF 第 7 页；原文 Multi-Objective Optimization，Eq.(17–18)；中译§3.3.1，[原文 PDF 第 4 页](https://arxiv.org/pdf/2601.06220v1#page=4)；Ctrl+F：中译“可选全局约束”／原文“optional global”。此处不把其他论文带成本的目标式当成ICL自身目标。】

- 如果只看准确率，ICL-Router 可以直接与其他准确率导向路由比较。
- 如果研究成本—质量权衡，就必须补全候选的实际调用成本或额外成本预测器，并对比同预算下的质量，否则拿它的 76.3% 与 ZeroRouter 的 0.39 Reward 比较毫无意义。

【定位·原文数值口径＋本文建议：中译 PDF 第 6 页；原文 Table 1（ID准确率与原始8模型），[原文 PDF 第 5 页](https://arxiv.org/pdf/2510.09719v3#page=5)；Ctrl+F：中译“76.30”／原文“76.30”；另见：另文 `ZeroRouter_Chinese_Full_Translation.pdf`：中译 PDF 第 12 页；原文 Table 2（新模型锚点采样消融），[原文 PDF 第 7 页](https://arxiv.org/pdf/2601.06220v1#page=7)；Ctrl+F：中译“D-最优性”／原文“D-optimality”；另见：另文 `ZeroRouter_Chinese_Full_Translation.pdf`：中译 PDF 第 8–9 页；原文 Evaluation Metrics，Eq.(19)；中译§4.1，[原文 PDF 第 5 页](https://arxiv.org/pdf/2601.06220v1#page=5)；Ctrl+F：中译“总奖励”／原文“total reward”。76.30是准确率百分数；0.39是加权Reward。需先统一候选、预算、指标和评测协议。】

添加一个线性成本惩罚虽容易，但它本身不是足够强的算法创新。真正的问题是：对成本差异很大的新模型，怎样以更少的测试题、可控的路由器开销得到稳定的最终效用收益。[推导]

【定位·本文研究判断：中译 PDF 第 5 页；原文 §3.4 Inference and Scalable Model Incorporation，Eq.(6)，[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“同一个查询集合”／原文“same query set”；另见：中译 PDF 第 15 页；原文 Appendix A.4 Cost Evaluation，[原文 PDF 第 11 页](https://arxiv.org/pdf/2510.09719v3#page=11)；Ctrl+F：中译“8 个词元”／原文“8 tokens”。创新性不能由本篇单独证明；需要新近文献和实验，不是作者结论。】

## ⑦ 新模型接入：固定成绩册，核心网络不重训

[原文，§3.4、§4.5] 新模型来了，使用**原来同一套 500 道固定题**测试它，获得 500 个 Yes/No 标签，与对应题目向量组合成为新档案。之后将档案加入路由器的候选集合，投影器与 7B 路由器不用因这个新模型再做参数训练。

【定位·原文事实：中译 PDF 第 5 页；原文 §3.4 Inference and Scalable Model Incorporation，Eq.(6)，[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“同一个查询集合”／原文“same query set”；另见：中译 PDF 第 15 页；原文 Appendix A.3 Challenging Query Set Construction，[原文 PDF 第 11 页](https://arxiv.org/pdf/2510.09719v3#page=11)；Ctrl+F：中译“125”／原文“125”。固定题集与无需重训明确在§3.4；500为实验采用的K。】

这里和另两篇一眼看出区别：

| 方法 | 新模型需要回答几题？ | 是否根据新模型前面答案实时改选下一题？ | 核心路由器是否重训？ |
|---|---:|---|---|
| **ICL-Router** | **500** | **否**，提前定好 | 否  【定位·原文事实：中译 PDF 第 5 页；原文 §3.4 Inference and Scalable Model Incorporation，Eq.(6)，[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“同一个查询集合”／原文“same query set”；另见：中译 PDF 第 15 页；原文 Appendix A.3 Challenging Query Set Construction，[原文 PDF 第 11 页](https://arxiv.org/pdf/2510.09719v3#page=11)；Ctrl+F：中译“125”／原文“125”。500为主设置。】 |
| **ZeroRouter** | **200** | **否**，D-optimal 预选 | 否  【定位·另篇原文事实：另文 `ZeroRouter_Chinese_Full_Translation.pdf`：中译 PDF 第 4 页；原文 Information-Theoretic Anchor Selection，Eq.(2–4)；中译§3.1.1，[原文 PDF 第 2–3 页](https://arxiv.org/pdf/2601.06220v1#page=2)；Ctrl+F：中译“贪心前向选择”／原文“greedy forward”；另见：另文 `ZeroRouter_Chinese_Full_Translation.pdf`：中译 PDF 第 11 页；原文 Efficient Onboarding with Minimal Data；中译§4.4，[原文 PDF 第 6 页](https://arxiv.org/pdf/2601.06220v1#page=6)；Ctrl+F：中译“200 个查询”／原文“200”。200为其报告预算。】 |
| **RADAR** | **5000/新配置（主设置）** | **是**，Fisher 更新 | 否  【定位·另篇原文事实＋算法口径：另文 `RADAR_中文译本_正文.pdf`：中译 PDF 第 6–7 页；原文 RADAR §3.5，Eq.(7–9)，[原文 PDF 第 6–7 页](https://arxiv.org/pdf/2509.25426v3#page=6)；Ctrl+F：中译“当前能力估计”／原文“Fisher information”；另见：另文 `RADAR_中文译本_正文.pdf`：中译 PDF 第 9 页；原文 RADAR §4.2 Model Scalability and Generalization Evaluation，[原文 PDF 第 9 页](https://arxiv.org/pdf/2509.25426v3#page=9)；Ctrl+F：中译“5,000”／原文“5k”。5,000见§4.2，单配置迭代见§3.5；不要误当全部8个新配置合计5,000次回答。】 |

**新模型实验 [原文，图 2/3]**：逐步加入 5 个模型后，ID 平均准确率从 **76.3% → 79.9%**，OOD 平均从约 **66.4% → 69.9%**。这是接入后真实路由准确率的变化，比“只评价新模型 θ 或 P 的预测误差”多了一步部署验证。但是**候选池新增模型本来就可能提高 Oracle 的上界**，要证明 profiling 方法优秀，还需在相同池与同一预算下比较其他 onboarding 方法。

【定位·原文数值＋本文对照建议：中译 PDF 第 8–9 页；原文 §4.5 Scalability to New Models，Figures 2–3，[原文 PDF 第 6 页](https://arxiv.org/pdf/2510.09719v3#page=6)；Ctrl+F：中译“Gemma3-12B”／原文“Gemma3-12B”。原文§4.5文字将起点写66.4%，Table 2与图3为66.47%；终点69.90%。Oracle上界与同预算对照属于本文解释。】

## ⑧ 评测：结果亮眼，也有真实局限

[原文，§4、表 1–4]

**ID 的 5 个数据集**：OlympiadBench、BBH、LogicBench、MMLU-Pro、MBPP，分别按 7:3 训练测试；**OOD 的 5 个数据集**：AIME、KORBench、MMLU-CF、AGIEval、HumanEval，完全作为未见任务测试。实验样本数及 few-shot 细节见附录 A.1，不应默认使用各 benchmark 的全量原始数据。

【定位·原文事实：中译 PDF 第 5 页；原文 §4.1 Datasets，[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“7:3”／原文“7:3”；另见：中译 PDF 第 14–15 页；原文 Appendix A.1（英文附录Table 1；中译表A.1），[原文 PDF 第 10 页](https://arxiv.org/pdf/2510.09719v3#page=10)；Ctrl+F：中译“1,080”／原文“1,080”。英文附录表1与正文表1重号；此处数据规模请查物理第10页的附录表。】

**对照算法**：Random Router、让 Qwen2.5-7B 通过文字档案选模型的 `LLM Router`、Max Expert、RouterDC、EmbedLLM、MODEL-SAT。注意 **Max Expert 选择的是“每个数据集最好的固定单模型”**，并不是“每道题事先知道答案的 Oracle”。作者为了比较公平对一些 baseline 调整了编码器；对 MODEL-SAT 进行了重新实现，不能简单理解为所有基线毫无改动地复用了原作者成绩。

【定位·原文事实＋定义澄清：中译 PDF 第 7 页；原文 §4.3 Baselines，[原文 PDF 第 5 页](https://arxiv.org/pdf/2510.09719v3#page=5)；Ctrl+F：中译“每个数据集”／原文“each dataset”；另见：中译 PDF 第 15 页；原文 Appendix A.2 Baselines，[原文 PDF 第 11 页](https://arxiv.org/pdf/2510.09719v3#page=11)；Ctrl+F：中译“八张”／原文“eight NVIDIA”。Max Expert是per-dataset最佳；A.2明确替换编码器与重实现MODEL-SAT。】

**原文主要结果**：

| 指标 | ICL-Router | 强参照 | 结论 |
|---|---:|---:|---|
| ID 平均准确率 | **76.30%** | EmbedLLM **74.16%**；Max Expert **73.19%** | 本设置下路由有收益  【定位·原文数值：中译 PDF 第 6 页；原文 Table 1（ID准确率与原始8模型），[原文 PDF 第 5 页](https://arxiv.org/pdf/2510.09719v3#page=5)；Ctrl+F：中译“76.30”／原文“76.30”】 |
| OOD 平均准确率 | **66.47%** | Max Expert **65.68%**；EmbedLLM **62.82%** | 分布外也改善，但提升幅度依对照而异  【定位·原文数值：中译 PDF 第 7 页；原文 Table 2（OOD准确率），[原文 PDF 第 5 页](https://arxiv.org/pdf/2510.09719v3#page=5)；Ctrl+F：中译“66.47”／原文“66.47”】 |
| 去掉查询复述预训练，ID | **74.01%** | 完整方法 **76.30%** | QRT 在实验里有作用  【定位·原文数值：中译 PDF 第 11 页；原文 §4.6 Effects of Query Reconstruction Training，Table 4，[原文 PDF 第 7 页](https://arxiv.org/pdf/2510.09719v3#page=7)；Ctrl+F：中译“74.01”／原文“74.01”】 |
| 去掉查询复述预训练，OOD | **64.06%** | 完整方法 **66.47%** | OOD 下降更明显  【定位·原文数值：中译 PDF 第 11 页；原文 §4.6 Effects of Query Reconstruction Training，Table 4，[原文 PDF 第 7 页](https://arxiv.org/pdf/2510.09719v3#page=7)；Ctrl+F：中译“74.01”／原文“74.01”】 |

**档案题数消融**：[原文，图 4/5] ID 100 条 **74.94%**、500 条 **76.30%**、1000 条反而 **75.75%**；OOD 100 条 **65.36%**、500 条 **66.47%**、1000 条 **65.71%**。所以“越多题越好”并不成立。500 是论文试过的一个较好配置，**不是理论上永远最优或所有新模型都必须测试的硬性常数**。

【定位·原文数字＋范围澄清：中译 PDF 第 10 页；原文 §4.6 Effects of In-Context Exemplar Quantity，Figures 4–5，[原文 PDF 第 7 页](https://arxiv.org/pdf/2510.09719v3#page=7)；Ctrl+F：中译“74.94”／原文“74.94”。作者还测试300条；500最优仅针对图4–5列出的设置，不是理论最优。】

**稳定性细节**：[原文，§4.2] 使用 3 个随机训练种子；为了平均被选择模型回答的随机性，测试时每个被选 LLM 以 temperature=0.3、top-p=1.0 生成 10 次并取平均。但主表没有给出每个方法完整的标准差/置信区间，所以不能用 3 个种子这个事实替代显著性证明。

【定位·原文事实＋范围核验：中译 PDF 第 6 页；原文 §4.2 Implementation Settings（续页），[原文 PDF 第 5 页](https://arxiv.org/pdf/2510.09719v3#page=5)；Ctrl+F：中译“五个epoch”／原文“five epochs”；另见：中译 PDF 第 6 页；原文 Table 1（ID准确率与原始8模型），[原文 PDF 第 5 页](https://arxiv.org/pdf/2510.09719v3#page=5)；Ctrl+F：中译“76.30”／原文“76.30”；另见：中译 PDF 第 7 页；原文 Table 2（OOD准确率），[原文 PDF 第 5 页](https://arxiv.org/pdf/2510.09719v3#page=5)；Ctrl+F：中译“66.47”／原文“66.47”。三种子和每次10采样来自§4.2；Table 1–2未列逐方法方差/置信区间。显著性保留判断为本文解释。】

**硬件现实**：正文训练 7B 路由器和投影器，使用冻结的 8B embedding 模型生成表示；附录对某些增强基线还使用 **8 张 A800 80GB**。在单卡 8GB 环境，先做缓存驱动的轻量 profiling/评测通常更可控；复现实验要区分“能读现成数据预测”与“能完整联合微调大模型”。

【定位·原文硬件事实＋本文建议：中译 PDF 第 5 页；原文 §4.2 Implementation Settings（QRT设置），[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“两层”／原文“two-layer”；另见：中译 PDF 第 15 页；原文 Appendix A.2 Baselines，[原文 PDF 第 11 页](https://arxiv.org/pdf/2510.09719v3#page=11)；Ctrl+F：中译“八张”／原文“eight NVIDIA”。八张A800指A.2的RouterDC及MODEL-SAT基线，不能当作ICL主方法的明确设备报告；8B embedding本身不训练。】

## 对原笔记的核对与纠错

| 旧笔记说法 | 结论 |
|---|---|
| 8 个原始 7–9B 模型，后加 5 个 | **正确，新增列表含 12B**  【定位·原文事实：中译 PDF 第 6 页；原文 Table 1（ID准确率与原始8模型），[原文 PDF 第 5 页](https://arxiv.org/pdf/2510.09719v3#page=5)；Ctrl+F：中译“76.30”／原文“76.30”；另见：中译 PDF 第 8–9 页；原文 §4.5 Scalability to New Models，Figures 2–3，[原文 PDF 第 6 页](https://arxiv.org/pdf/2510.09719v3#page=6)；Ctrl+F：中译“Gemma3-12B”／原文“Gemma3-12B”】 |
| embedding 用冻结的 Qwen3-Embedding-8B | **正确**  【定位·原文事实：中译 PDF 第 5 页；原文 §4.2 Implementation Settings（QRT设置），[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“两层”／原文“two-layer”】 |
| 第一阶段先复述题目再学分类 | **正确，QRT 不是最终路由损失**  【定位·原文事实：中译 PDF 第 4 页；原文 §3.2 Query Reconstruction Training，Eq.(1–3)，[原文 PDF 第 3–4 页](https://arxiv.org/pdf/2510.09719v3#page=3)；Ctrl+F：中译“重构损失”／原文“reconstruction loss”】 |
| 第二阶段同时训练投影器和 7B 路由器 | **正确**  【定位·原文事实：中译 PDF 第 6 页；原文 §4.2 Implementation Settings（续页），[原文 PDF 第 5 页](https://arxiv.org/pdf/2510.09719v3#page=5)；Ctrl+F：中译“五个epoch”／原文“five epochs”】 |
| 500 题分成 1/2/3/4 人答对、各 125 | **正确，别漏掉排除了 5–7 人答对**  【定位·原文规则＋推论：中译 PDF 第 15 页；原文 Appendix A.3 Challenging Query Set Construction，[原文 PDF 第 11 页](https://arxiv.org/pdf/2510.09719v3#page=11)；Ctrl+F：中译“125”／原文“125”】 |
| 只看准确率，不在打分中考虑美元成本 | **正确**  【定位·原文公式：中译 PDF 第 5 页；原文 §3.4 Inference and Scalable Model Incorporation，Eq.(6)，[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“同一个查询集合”／原文“same query set”】 |
| 每候选一个 7B 前向，路由总开销等于输出 token 数 | **后半句错误；prefill 等仍有计算**  【定位·本文开销解释：中译 PDF 第 5 页；原文 §3.3 ICL Model Routing Training，Eq.(4–5)，[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“能力画像”／原文“capability profile”；另见：中译 PDF 第 15 页；原文 Appendix A.4 Cost Evaluation，[原文 PDF 第 11 页](https://arxiv.org/pdf/2510.09719v3#page=11)；Ctrl+F：中译“8 个词元”／原文“8 tokens”】 |
| 3 随机种子意味着论文给出可靠方差区间 | **没有对应完整区间**  【定位·范围核验：中译 PDF 第 6 页；原文 §4.2 Implementation Settings（续页），[原文 PDF 第 5 页](https://arxiv.org/pdf/2510.09719v3#page=5)；Ctrl+F：中译“五个epoch”／原文“five epochs”；另见：中译 PDF 第 6 页；原文 Table 1（ID准确率与原始8模型），[原文 PDF 第 5 页](https://arxiv.org/pdf/2510.09719v3#page=5)；Ctrl+F：中译“76.30”／原文“76.30”；另见：中译 PDF 第 7 页；原文 Table 2（OOD准确率），[原文 PDF 第 5 页](https://arxiv.org/pdf/2510.09719v3#page=5)；Ctrl+F：中译“66.47”／原文“66.47”】 |

## 候选破题点

**ICL-1 [原文→待实验]：固定选 1–4 人答对的题，漏掉 5–7 人答对的题，是否丢失对强新模型最有价值的信号？** 对能力明显高于旧池的新模型，旧模型大多答对的题也可能产生有用差异；要通过相同题数、同分布预算下的分层选题消融验证，不能预设会提高。

【定位·原文选题规则→待实验：中译 PDF 第 15 页；原文 Appendix A.3 Challenging Query Set Construction，[原文 PDF 第 11 页](https://arxiv.org/pdf/2510.09719v3#page=11)；Ctrl+F：中译“125”／原文“125”。5–7人组被排除可推出；对强新模型更有价值是未验证假说。】

**ICL-2 [原文→待实验]：500 条不是越多越好，那究竟如何选择更好的少量档案？** 相同 100/200/500 预算，比较随机、作者规则、D-optimal、基于模型回答反馈的自适应规则，同时控制路由器上下文长度与训练是否公平。

【定位·原文消融→本文拟议实验：中译 PDF 第 15 页；原文 Appendix A.3 Challenging Query Set Construction，[原文 PDF 第 11 页](https://arxiv.org/pdf/2510.09719v3#page=11)；Ctrl+F：中译“125”／原文“125”；另见：中译 PDF 第 10 页；原文 §4.6 Effects of In-Context Exemplar Quantity，Figures 4–5，[原文 PDF 第 7 页](https://arxiv.org/pdf/2510.09719v3#page=7)；Ctrl+F：中译“74.94”／原文“74.94”。100/300/500/1000为作者消融；加入200或其他选题法属本文建议，不在原文结果中。】

**ICL-3 [原文范围→待实验]：只报告准确率，路由器本身的 7B 计算成本没有系统量化。** 使用真实 wall-clock/显存/能耗或 API 等效价，衡量 8/13/20 候选时预处理和推理开销，寻找是否存在更轻量的压缩档案替代方案。

【定位·本版报告范围→待实验：中译 PDF 第 15 页；原文 Appendix A.4 Cost Evaluation，[原文 PDF 第 11 页](https://arxiv.org/pdf/2510.09719v3#page=11)；Ctrl+F：中译“8 个词元”／原文“8 tokens”；另见：中译 PDF 第 6 页；原文 Table 1（ID准确率与原始8模型），[原文 PDF 第 5 页](https://arxiv.org/pdf/2510.09719v3#page=5)；Ctrl+F：中译“76.30”／原文“76.30”；另见：中译 PDF 第 7 页；原文 Table 2（OOD准确率），[原文 PDF 第 5 页](https://arxiv.org/pdf/2510.09719v3#page=5)；Ctrl+F：中译“66.47”／原文“66.47”。8/13来自原始池及新增5模型；20候选、能耗和压缩档案方案为拟议评测，非原文数据。】

**ICL-4 [推导→待文献/实验]：目标是预测每个模型答对概率，而部署可能只关心“新模型是否改变最终最佳决策”。** 可将固定能力档案或动态选题与路由效用联系起来，但需对照近期决策导向实验设计方法，不能只在 profiling 上换个选题算法就直接声称创新。

【定位·本文研究假说／待文献：中译 PDF 第 5 页；原文 §3.3 ICL Model Routing Training，Eq.(4–5)，[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“能力画像”／原文“capability profile”；另见：中译 PDF 第 5 页；原文 §3.4 Inference and Scalable Model Incorporation，Eq.(6)，[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“同一个查询集合”／原文“same query set”。Eq.(5–6)是推导起点；论文没有证明所提决策导向设计更新颖或更优。】

**ICL-5 [原文→待实验]：模型尺寸较集中，价位差异未系统报告，成本异质池泛化还没证明。** 在多价位 API 模型加开源模型的混合池中，比较完整预测与成本权衡及实际 regret；与 IRT/RADAR 设定对齐后再说优势。

【定位·原文范围→待实验：中译 PDF 第 15 页；原文 Appendix A.5 Limitations，[原文 PDF 第 11 页](https://arxiv.org/pdf/2510.09719v3#page=11)；Ctrl+F：中译“小参数规模”／原文“small-parameter”；另见：中译 PDF 第 6 页；原文 Table 1（ID准确率与原始8模型），[原文 PDF 第 5 页](https://arxiv.org/pdf/2510.09719v3#page=5)；Ctrl+F：中译“76.30”／原文“76.30”；另见：中译 PDF 第 8–9 页；原文 §4.5 Scalability to New Models，Figures 2–3，[原文 PDF 第 6 页](https://arxiv.org/pdf/2510.09719v3#page=6)；Ctrl+F：中译“Gemma3-12B”／原文“Gemma3-12B”。校订：论文支持“模型尺寸较集中”，没有报告价格表来证明“价位集中”；异质成本池结果仍待验证。】

## 老师可能问

1. 为什么要先训练“复述题目”？因为投影向量与 7B 路由器的内部表示有分布鸿沟，QRT 帮助学会解读向量，而不仅是尺寸匹配。 【定位·原文动机＋讲解：中译 PDF 第 4 页；原文 §3.2 Query Reconstruction Training，Eq.(1–3)，[原文 PDF 第 3–4 页](https://arxiv.org/pdf/2510.09719v3#page=3)；Ctrl+F：中译“重构损失”／原文“reconstruction loss”】
2. 500 道是新模型的新训练集吗？不是；是供新模型测评的固定代表性题单，拿到结果后建立它的上下文档案，**不重新训练主路由器**。 【定位·原文事实：中译 PDF 第 5 页；原文 §3.4 Inference and Scalable Model Incorporation，Eq.(6)，[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“同一个查询集合”／原文“same query set”；另见：中译 PDF 第 15 页；原文 Appendix A.3 Challenging Query Set Construction，[原文 PDF 第 11 页](https://arxiv.org/pdf/2510.09719v3#page=11)；Ctrl+F：中译“125”／原文“125”】
3. 为什么 Max Expert 不是 Oracle？前者每个数据集固定选一个最好模型；后者每道题都事先知道所有模型答得怎样。 【定位·原文定义＋本文对照：中译 PDF 第 7 页；原文 §4.3 Baselines，[原文 PDF 第 5 页](https://arxiv.org/pdf/2510.09719v3#page=5)；Ctrl+F：中译“每个数据集”／原文“each dataset”】
4. 既然输出 8 token，为什么仍可能有不小开销？因为 7B 计算预填充档案与输入的前向时间并没有被输出 token 数描述。 【定位·本文计算流程解释：中译 PDF 第 5 页；原文 §3.3 ICL Model Routing Training，Eq.(4–5)，[原文 PDF 第 4 页](https://arxiv.org/pdf/2510.09719v3#page=4)；Ctrl+F：中译“能力画像”／原文“capability profile”；另见：中译 PDF 第 15 页；原文 Appendix A.4 Cost Evaluation，[原文 PDF 第 11 页](https://arxiv.org/pdf/2510.09719v3#page=11)；Ctrl+F：中译“8 个词元”／原文“8 tokens”】
5. 为什么 1000 条不如 500？论文给出下降现象，但“冗余/噪声/上下文负担”都只是待进一步验证的解释，不能据此推断单一原因。 【定位·原文现象＋解释边界：中译 PDF 第 10 页；原文 §4.6 Effects of In-Context Exemplar Quantity，Figures 4–5，[原文 PDF 第 7 页](https://arxiv.org/pdf/2510.09719v3#page=7)；Ctrl+F：中译“74.94”／原文“74.94”。原文也用likely提出冗余/噪声解释；没有隔离机制实验，因此不能作为已证实原因。】

**原文定位**：§3.2 重建 Eq.(1–3)；§3.3 档案 Eq.(4) 与分类损失 Eq.(5)；§3.4 推理 Eq.(6)；§4.2 训练细节；§4.4 表 1/2；§4.5 图 2/3；§4.6 图 4/5、表 3/4；附录 A.2 baseline 修改，A.3 500 题规则，A.4 成本，A.5 局限。

【定位·导航补充：中译 PDF 第 4 页；原文 §3.2 Query Reconstruction Training，Eq.(1–3)，[原文 PDF 第 3–4 页](https://arxiv.org/pdf/2510.09719v3#page=3)；Ctrl+F：中译“重构损失”／原文“reconstruction loss”。各式/表的精确物理页见篇首导航与上文邻接定位；不要将附录Table 1和正文Table 1混同。】