# 05 ICL-Router：给每个模型一份 500 题的成绩册，让 7B 路由器自己读

> **在主线上的位置：第 2 站（固定题单）之二。**
> 上一篇 [ZeroRouter](04_ZeroRouter_精读核验.md) 让新模型做固定锚点题，再用 IRT 把答题结果压成能力参数。ICL-Router 走另一条路：**不估参数**，把“这些题答对了、那些题答错了”的原始记录直接交给一个专门训练过的 LLM 路由器去读。新模型只要补一份成绩册，路由器不重训。
> 与 ZeroRouter 的共同点是题单固定、不随答题结果调整；不同的是题单规则是手工分层，而不是优化出来的。下一站 [RADAR](06_RADAR_精读核验.md) 让题单随答题结果动态变化。
>
> 原论文：[arXiv:2510.09719v3](https://arxiv.org/abs/2510.09719v3)（2025.10）· [中文译文](translations/05_ICL-Router_中文译文.md) · [中文 PDF](pdfs/zh/ICL-Router_Chinese.pdf) · [两阶段框架图](translations/05_ICL-Router_中文译文.md#fig-1)
> 全仓库统一记号见 [README](README.md#统一记号)。

## 链路速查

| 环节 | ICL-Router 的做法 |
|---|---|
| 候选池 | 初始 8 个 7B–9B 开源模型，逐步扩到 13 个（含 12B） |
| 问题表示 | 冻结的 Qwen3-Embedding-8B（4096 维）+ 两层 MLP 投影 |
| 模型表示 $z_m$ | 同一套 500 道题的“题目向量 + Yes/No”成绩册 |
| 质量预测 $\widehat P(q,m)$ | Qwen2.5-7B-Instruct 路由器读成绩册，输出答对概率 |
| 成本 | 没有 |
| 决策 | $\arg\max_m\widehat P(q,m)$，相当于统一记号里 $w=1$ |
| 新模型接入 | 新模型答完 500 道固定题，补一份成绩册，不重训 |
| 主指标 | 平均准确率 |

它针对的前序问题：RouterDC、EmbedLLM 为固定模型池学表示，加模型通常要重训；MODEL-SAT 允许动态接入，但依赖人工设计的能力描述。[中文方法概述](translations/05_ICL-Router_中文译文.md#s3-1)

<a name="query-reconstruction"></a>

## 1. 第一阶段：先让路由器读懂题目向量

几百道题的全文放进上下文太长，所以每道题先压成向量：

$$
\mathbf e_n=f_{\mathrm{emb}}(q_n),\qquad \mathbf v_n=f_{\mathrm{proj}}(\mathbf e_n).
$$

维度对上不代表路由器能理解。于是先做**查询重构训练（QRT）**：路由器只看投影后的向量，逐词复述原题。复述得出来，说明投影器保留了语义、路由器也学会了解读这种输入。[公式（1）–（3）](translations/05_ICL-Router_中文译文.md#s3-2)

$$
\mathcal L_{\mathrm{rec}}=-\frac{1}{NT_n}\sum_{n=1}^{N}\sum_{t=1}^{T_n}\log P\!\left(q_n^{(t)}\mid q_n^{(<t)},\mathbf v_n\right).
$$

分母里的 $T_n$ 同时出现在求和中，记号有歧义；这里保留原式，复现时要看实现中的 token mask 和 loss reduction。[公式提示](translations/05_ICL-Router_中文译文.md#eq-3-normalization-note)

训练：先单独训投影器 1 个 epoch（学习率 $2\times10^{-5}$），再联合训投影器与路由器 2 个 epoch（路由器学习率 $5\times10^{-6}$）；8B 嵌入模型始终冻结。[训练设置](translations/05_ICL-Router_中文译文.md#s4-2)

QRT 是让 7B 路由器读懂向量的工程步骤，与新模型接入逻辑无关，读的时候可以略过细节。

<a name="capability-profiles"></a>

## 2. 第二阶段：成绩册怎样决定路由

模型 $M$ 的成绩册：

$$
\mathbf P_M=\big((\mathbf v_1,c_{1M}),\ldots,(\mathbf v_K,c_{KM})\big),\qquad c_{kM}\in\{\mathrm{Yes},\mathrm{No}\}.
$$

$\mathbf v_k$ 是题目向量，$c_{kM}$ 是该模型在这题上的真实对错（原式在固定模型的档案中记作 $c_k$，下标 $M$ 是为区分模型加的）。所有候选用同一套 $K=500$ 道题，记录可以直接对照；路由器可以自己学“在哪类题上表现好”，不必人为命名能力维度。[能力档案与公式（4）](translations/05_ICL-Router_中文译文.md#s3-3)

**这 500 道题怎么选（主线上的又一种选题方法）。** 根据旧池 8 个模型的表现，从恰好 1、2、3、4 个模型答对的题中各取 125 道，都来自所有对照方法都能用的训练集。0 个或 5–8 个模型答对的题不进题单。作者的理由是全对、全错的题缺少区分度；但这个理由本身不能证明排除 5–7 个模型答对的题是最优选择。[附录 A.3：选题规则](translations/05_ICL-Router_中文译文.md#app-a-3)

**训练与决策。** 路由器读某个候选的成绩册和当前查询，预测这个候选能否答对；监督信号是真实答题结果，交叉熵训练 5 个 epoch（投影器学习率 $1\times10^{-5}$、路由器 $2\times10^{-6}$，batch 32）。

$$
\widehat P(q,M)=P_{\mathrm{router}}(\mathrm{Yes}\mid\mathbf P_M,q),\qquad M^*(q)=\arg\max_{M}\widehat P(q,M).
$$

路由时用的是预先建好的成绩册，不需要让候选先试答当前题。[训练损失（5）](translations/05_ICL-Router_中文译文.md#eq-5) · [决策公式（6）](translations/05_ICL-Router_中文译文.md#eq-6)

例：新题要分析一段程序的边界行为；A 的记录显示它在逻辑与程序题上较好，B 的常识题成绩更高。路由器可能给 A 0.78、B 0.42，于是选 A（示意数字，来自训练后的路由器预测，不是平均分代入手写规则）。

<a name="new-model-incorporation"></a>

## 3. 新模型接入：只补成绩册

新模型答完那 500 道固定题，得到 Yes/No，与已有题目向量组成新成绩册；投影器和 7B 路由器都不更新。题单提前定好，不随新模型前一题的表现改变。“无需重训”只针对接入这一步；初始两阶段训练和新模型答 500 题的开销仍在。[推理与接入流程](translations/05_ICL-Router_中文译文.md#s3-4)

初始池：DeepSeek-R1-Distill-Qwen-7B、Llama-3.1-8B-Instruct、Llama-3.1-Nemotron-Nano-8B-v1、Qwen2.5-7B-Instruct、cogito-v1-preview-llama-8B、Gemma-2-9B-IT、Internlm3-8B-Instruct、GLM-4-9B-Chat。依次加入 Falcon-H1-7B-Instruct、Gemma3-12B-IT、DeepSeek-R1-Llama-8B、OpenThinker3-7B、AceReason-Nemotron-1.1-7B。[初始候选](translations/05_ICL-Router_中文译文.md#table-1) · [新增模型](translations/05_ICL-Router_中文译文.md#s4-5)

五个模型陆续加入后：

| | 初始 | 加入 5 个后 |
|---|---:|---:|
| ICL-Router，ID | 76.30% | 79.90% |
| ICL-Router，OOD | 66.47% | 69.90% |
| MODEL-SAT，ID | 71.71% | 74.81% |
| MODEL-SAT，OOD | 62.99% | 66.34% |

与 IRT-Router 只报告新模型的预测误差不同，这里报告的是**接入后的路由准确率**。但新模型本身就会抬高池子的上限；要判断哪种接入方法更好，得固定候选池、题数和预算。[图 2、图 3 数值](translations/05_ICL-Router_中文译文.md#table-fig-2-3)

<a name="experimental-evidence"></a>

## 4. 主实验

ID：OlympiadBench、BBH、LogicBench、MMLUPro、MBPP，各按 7:3 划分；OOD：AIME、KORBench、MMLU-CF、AGIEval、HumanEval。不是全量数据，抽样规模和 few-shot 设置见[附录数据表](translations/05_ICL-Router_中文译文.md#table-a-1)。

基线：随机路由、LLM Router（凭自然语言画像选模型）、Max Expert（每个数据集固定选该集上最好的单模型，不是 Oracle）、RouterDC、EmbedLLM、MODEL-SAT。RouterDC 和 EmbedLLM 换用了 Qwen3 嵌入，MODEL-SAT 由作者重新实现。[基线定义](translations/05_ICL-Router_中文译文.md#s4-3) · [基线实现](translations/05_ICL-Router_中文译文.md#app-a-2)

| 方法 | ID 平均准确率 | OOD 平均准确率 |
|---|---:|---:|
| Random Router | 57.91% | 49.44% |
| LLM Router | 62.36% | 52.95% |
| Max Expert | 73.19% | 65.68% |
| RouterDC | 72.32% | 63.13% |
| EmbedLLM | 74.16% | 62.82% |
| MODEL-SAT | 71.71% | 62.99% |
| ICL-Router | **76.30%** | **66.47%** |

ID 比 EmbedLLM 高 2.14 个百分点、比 Max Expert 高 3.11；OOD 比 Max Expert 高 0.79。[ID 表](translations/05_ICL-Router_中文译文.md#table-1) · [OOD 表](translations/05_ICL-Router_中文译文.md#table-2)

三个消融：

- **去掉 QRT**：ID 76.30% → 74.01%，OOD 66.47% → 64.06%，各降约 2.3、2.4 个百分点。[表 4](translations/05_ICL-Router_中文译文.md#table-4)
- **成绩册长度**：与主线关系最紧的一项。[示例数量实验](translations/05_ICL-Router_中文译文.md#table-fig-4-5)

  | 每份成绩册的题数 | ID | OOD |
  |---|---:|---:|
  | 100 | 74.94% | 65.36% |
  | 300 | 75.87% | 65.75% |
  | 500 | **76.30%** | **66.47%** |
  | 1000 | 75.75% | 65.71% |

  500 在这四档里最好，1000 反而下降。作者推测是冗余或噪声，但没有隔离实验；500 也不是理论最优或必须遵守的常数。
- **嵌入模型**：mxbai-embed-large-v1（0.3B、1024 维）换成 Qwen3-8B-Embedding（8B、4096 维），ID/OOD 从 74.45%/64.55% 升到 76.30%/66.47%。[嵌入实验](translations/05_ICL-Router_中文译文.md#table-3)

<a name="limits-and-next-questions"></a>

## 5. 容易读错的地方

- “上下文学习”建立在两阶段训练过的路由器上，不是把成绩单丢给任意聊天模型就能选模型。
- 正文说 ID 比 RouterDC 高 4.08 个百分点，但表值是 76.30 − 72.32 = 3.98；比较时以表值为准。
- 3 个随机种子、每个被选中的模型以 temperature 0.3、top-p 1.0 采样 10 次取均值，但主表没有标准差或置信区间。[训练与解码设置](translations/05_ICL-Router_中文译文.md#s4-2-training-and-decoding)
- 决策里没有价格、延迟或预算，它证明的是准确率导向路由下这种模型表示的价值；不能和其他论文的综合 Reward 放在一起排名。
- “8 个候选只需生成约 8 个分类 token”说的是输出量；完整开销还有查询嵌入、投影、7B 路由器处理成绩册的上下文，附录没有完整的延迟、显存或费用表。[附录 A.4](translations/05_ICL-Router_中文译文.md#app-a-4)
- 8 张 A800 80GB 是 RouterDC 和 MODEL-SAT 基线的配置，不是 ICL-Router 主方法报告的设备。
- 作者承认实验集中在小参数模型，基准也不专门评价聊天或指令遵循。[原文局限](translations/05_ICL-Router_中文译文.md#app-a-5)

## 6. 对主线研究的用处

ICL-Router 给主线提供了一个**非参数**的对照：同样是固定题单，不估 $\theta$ 而直接读记录。可延伸的问题（研究建议，相关工作还需查新）：

- **同样题数，怎样选更好？** 比较原来的“1–4 个模型答对”规则、加入 5–7 个模型答对题的分层规则、随机、D-optimal、自适应选题，保持预算和上下文长度一致，看最终路由表现。排除某类题是否会漏掉强新模型的长处，目前只是假设。
- **更短的成绩册能否保住决策？** 在 100、200、500 题下比较不同的建册方式，并检查预测误差下降是否真的改变了最终选中的模型。
- **算上全部开销还剩多少收益？** 测 8、13、20 个候选时的预处理时间、路由延迟、显存，并在价格不一的混合池里比较同预算质量。

8GB 单卡上可以先用缓存的表示和答题结果做轻量评估，但这不等于复现了论文的联合训练。

**下一站：** [06 RADAR](06_RADAR_精读核验.md) 回到 IRT 框架，让下一道题由新模型当前的能力估计决定——这是主线上第一个**自适应**的选题方法。
