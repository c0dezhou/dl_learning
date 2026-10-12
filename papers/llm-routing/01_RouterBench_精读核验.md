# 01 RouterBench：先定好尺子，再谈路由好不好

> **在两条主线上的位置**（两条主线见 [README](README.md#两条主线)）
>
> - **主线 A（来一个 query，怎样选模型）· 第 1 篇：定好尺子。** 路由器到底比什么强才算有用？RouterBench 给出逐题缓存、Zero 基线、Oracle 和 AIQ，后面五篇都在这把尺子上比较自己。它也给出最朴素的预测式路由（kNN、MLP），核心发现是：这类方法没有稳定赢过不看题目的 Zero 基线。
> - **主线 B（新模型怎样接入）· 起点。** RouterBench 不处理接入。它的预测靠每个模型在历史题上的成绩，**新模型没有历史成绩时怎么办？** 这就是主线 B 的出发点。它的缓存和指标则是做接入实验的评测框架。
>
> 原论文：Hu 等，*RouterBench: A Benchmark for Multi-LLM Routing System*，[arXiv:2403.12031v2](https://arxiv.org/abs/2403.12031v2)（2024.03）· [中文精读材料](translations/01_RouterBench_中文精读材料.md#rb-title)（不是全文译本）
> 全仓库统一记号见 [README](README.md#统一记号)。

## 链路速查

| 环节 | RouterBench 的做法 |
|---|---|
| 候选池 | 11 个核心模型（另有 3 个检索服务用于 RAG） |
| 问题表示 | SentenceTransformers 句向量，最佳为 all-MiniLM-L12-v2 |
| 模型表示 $z_m$ | 该模型在训练题上的成绩那一列 |
| 质量预测 $\widehat P(q,m)$ | kNN 或 MLP |
| 成本 $\widehat C$ | 每 token 定价，每个模型一个常数 |
| 决策 | $\lambda\widehat P-\widehat c$，$\lambda$ 越大越看重质量（即统一记号里的 $w$） |
| 新模型接入 | 不涉及 |
| 主指标 | AIQ（成本—质量曲线的平均高度） |

## 1. 为什么要逐题记录每个模型的表现

两个模型平均分相同，仍可能各擅胜场：A 做错的题，B 恰好会做。只要能在作答前认出这种题，逐题选模型就有收益。RouterBench 因此预先收集“每道题 × 每个模型”的质量与费用，让路由策略可以离线反复评测，不必每次重新调用模型。[逐题记录与用途](translations/01_RouterBench_中文精读材料.md#rb-sec-2)

数据规模：

- 405,467 条模型—题目记录（不是题目数），8 类数据集、64 个任务：MMLU、HellaSwag、WinoGrande、ARC-Challenge、GSM8K、MBPP、MT-Bench 和自建 RAG（约 800 个问题）。前五类 5-shot + Exact Match，后三类 0-shot + GPT-4 Judge，质量统一到 [0,1]。[数据与判分表](translations/01_RouterBench_中文精读材料.md#rb-table-1)
- 11 个核心模型：Llama-70B-chat、Mixtral-8×7B-chat、Yi-34B-chat、Code Llama-34B、Mistral-7B-chat、WizardLM-13B、GPT-4、GPT-3.5-turbo、Claude-instant-v1、Claude-v1、Claude-v2；RAG 另加 You.com API、sonar-small-online、sonar-medium-online。[模型池说明](translations/01_RouterBench_中文精读材料.md#rb-supplement-models)

这份缓存就是后续研究在 8GB 单卡上也能做方法对比的前提：路由与选题算法只读缓存，不需要部署模型。[数据字段](translations/01_RouterBench_中文精读材料.md#rb-table-fields)

## 2. 预测式路由：从问题到选中的模型

**第一步，预测质量。** kNN 找训练集中最相似的 $k$ 道旧题，取模型 $m$ 在这些题上的平均成绩：

$$
\widehat P(q,m)=\frac{1}{k}\sum_{j\in N_k(q)}y_{jm}.
$$

$y_{jm}$ 是模型 $m$ 在旧题 $j$ 上的实测质量。例：最近的 5 道数学题里，A 答对 4 道、B 答对 2 道，预测值就是 0.8 和 0.4（自拟例子）。MLP 则直接学一个“题目向量 → 各模型质量”的函数。[kNN](translations/01_RouterBench_中文精读材料.md#rb-sec-5-2) · [MLP](translations/01_RouterBench_中文精读材料.md#rb-sec-5-3)

超参数（附录 B）：kNN 用余弦相似度，比较 $k\in\{5,10,40\}$，最佳为 $k=40$；MLP 最佳为两层、每层 100 个神经元、ReLU、学习率 0.001。两者都按任务内 70%/30% 划分训练与测试，MT-Bench 因样本少不参加。[超参数](translations/01_RouterBench_中文精读材料.md#rb-appendix-settings) · [数据划分](translations/01_RouterBench_中文精读材料.md#rb-supplement-split)

**第二步，加入价格。**

$$
m^*(q)=\arg\max_m\left[\lambda\widehat P(q,m)-\widehat c_m\right].
$$

$\lambda$ 是愿付价（willingness to pay）。例：A 预测 0.8、成本 0.08，B 预测 0.6、成本 0.01。$\lambda=0.2$ 时分数为 0.08 对 0.11，选 B；$\lambda=1$ 时为 0.72 对 0.59，选 A。同一个质量预测器，换个付费偏好就会改选。[选择公式](translations/01_RouterBench_中文精读材料.md#rb-eq-6)

**一个口径差别。** 选模型时 $\widehat c_m$ 用每 token 定价近似；评测时记的是按实际输入、输出 token 算出的逐题费用（开放权重模型参考 Together AI 价格）。复现时两者要分开写清。[费用口径](translations/01_RouterBench_中文精读材料.md#rb-appendix-cost)

**另一条路：级联。** 先调用便宜模型，回答不合格再升级。一题可能调用多次，费用累加。论文用已知质量构造理想 Judge，再以概率 $\varepsilon$ 翻转判断来模拟误判。级联要先生成再判断，与两条主线关心的“生成前选模型”是两种设定。[级联流程](translations/01_RouterBench_中文精读材料.md#rb-sec-5-4)

## 3. 尺子本身：Zero 基线、Oracle 与 AIQ

这一节是整篇最重要的部分，后面五篇的比较都建立在这里。

**曲线而不是点。** 改变 $\lambda$ 得到一串成本—质量点。两种策略按固定概率混合，期望成本与质量落在两点连线上，所以只看孤立点不公平。论文取所有可实现点的非递减上包络（NDCH）。示意：A=(0.2, 0.60)、B=(0.8, 0.90)，各用一半得到 (0.5, 0.75)，优于同成本下质量 0.70 的点。[随机混合](translations/01_RouterBench_中文精读材料.md#rb-eq-4) · [混合前沿图](translations/01_RouterBench_中文精读材料.md#rb-fig-2)

**Zero 基线：不看题目的最优混合。** 只用各模型的平均质量和平均价格，按预算做最优固定概率混合。它回答的问题是：*不分析每道题，只凭总体信息已经能做到这样，逐题预测还能多带来多少？*（注意与 2026 年的 ZeroRouter 方法无关。）[基线定义](translations/01_RouterBench_中文精读材料.md#rb-table-baselines)

**Oracle：事后上界。** 预先知道每道题上所有模型的真实质量，选最好的、同分选最便宜的。它显示数据里有多少可利用的空间，但线上拿不到这些答案。[Oracle](translations/01_RouterBench_中文精读材料.md#rb-sec-4)

**AIQ：把曲线压成一个数。**

$$
\operatorname{AIQ}(R)=\frac{1}{c_{\max}-c_{\min}}\int_{c_{\min}}^{c_{\max}}\widetilde Q_R(c)\,dc,
$$

即共同成本区间内的平均质量高度。名字虽叫 Average Improvement in Quality，公式里并没有减去 Zero；“提升多少”要在同一区间内比 AIQ 差值。[AIQ 定义](translations/01_RouterBench_中文精读材料.md#rb-sec-3-4) · [成本与质量的定义](translations/01_RouterBench_中文精读材料.md#rb-sec-3-1)

## 4. 实验结论：预测式路由没有稳定赢过 Zero

Figure 4 的 AIQ，在同一行内比较：[结果表](translations/01_RouterBench_中文精读材料.md#rb-table-2)

| 任务 | Zero | kNN | MLP |
|---|---:|---:|---:|
| MMLU | 0.763 | 0.773 | 0.769 |
| ARC-Challenge | 0.877 | 0.871 | 0.875 |
| MBPP | 0.660 | 0.644 | 0.642 |
| HellaSwag | 0.588 | 0.588 | 0.588 |
| WinoGrande | 0.778 | 0.780 | 0.778 |
| GSM8K | 0.632 | 0.632 | 0.623 |

MMLU 上两种方法都高于 Zero；ARC-Challenge 和 MBPP 上都低于 Zero；其余基本持平。Oracle 显示模型之间确有逐题互补的空间，但从题目文本把它学出来并不容易。[实验解释](translations/01_RouterBench_中文精读材料.md#rb-sec-6-1)

另两组结果：

- **级联**在零误判时很强（MMLU 0.901 对 Zero 0.763；MBPP 0.862 对 0.660），但 HellaSwag 上是 0.469 对 0.607，反而更差；误判率 $\varepsilon>0.2$ 后明显退化。[级联结果](translations/01_RouterBench_中文精读材料.md#rb-table-3)
- **RAG** 上路由收益最清楚：Zero 0.632，Cascade（$\varepsilon=0.2$）0.657，MLP 0.675，kNN 0.695。涉及近期事件的问题需要搜索，路由器容易认出来。但这里的候选包含检索服务，差异不只是模型能力。[RAG 结果](translations/01_RouterBench_中文精读材料.md#rb-sec-6-3)

## 5. 容易读错的地方

- 405,467 是记录数，不是题目数；11 个核心模型不代表每项实验都在 14 个候选中选（RAG 图中是 5 个）。
- 不同图里的 Zero 数值不完全相同（GSM8K 在级联图中为 0.628，预测式图中为 0.632），不能跨图拼表。
- 作者说的“不显著超过”是定性表述，论文没有做统计显著性检验。
- 随机混合满足的是**批量平均**预算，不保证每条请求都在预算内。
- AIQ 的绝对值依赖积分区间；原论文许多图的横轴是评测集总成本而非每题成本。
- RouterBench（2024）与 LLMRouterBench（2026）是两篇不同的论文。
- 前面已有 FrugalGPT 等路由研究，RouterBench 的贡献是统一比较规则，而不是首次证明路由有价值。[与前序工作的关系](translations/01_RouterBench_中文精读材料.md#rb-supplement-position)

## 6. 对研究的用处

**对主线 A：** 值得先跑的三个检查见本节最后一段，它们对应 README “主线 A 留下的问题”中的成本口径和预算类型两项。

**对主线 B：** RouterBench 本身不处理新模型接入：kNN 需要新模型在历史题上的成绩，MLP 也需要相应训练数据。它对接入研究的用处是三样东西：[研究设定的区别](translations/01_RouterBench_中文精读材料.md#rb-sec-7-1)

1. **逐题缓存**，让选题实验可以离线做；
2. **Zero 基线**，判断一个接入方法是否真有逐题收益；
3. **AIQ 曲线指标**，比较“接入后路由好不好”，而不只是“能力估得准不准”。

用它做接入实验时，需要自己补上：新模型到达的时间设定、反馈预算 $B$、信息屏蔽（选题算法只能读已购买的反馈，测试标签只用于事后记分），并按接入事件、数据来源重采样检查稳定性。[复现核验清单](translations/01_RouterBench_中文精读材料.md#rb-sec-7-2)

值得先跑的三个检查（研究建议，非论文结论）：同一数据下把单模型、Zero、kNN/MLP、Oracle 放在一起，看逐题互补性能否从输入学出；把固定单价换成逐题成本，看选择翻转比例；明确业务约束是批量平均预算还是每条硬预算。[成本依据](translations/01_RouterBench_中文精读材料.md#rb-appendix-cost) · [随机混合的适用含义](translations/01_RouterBench_中文精读材料.md#rb-sec-3-2)

**接着读：** [02 RouteLLM](02_RouteLLM_精读核验.md)。

- 主线 A：它把多选一缩成强弱二选一，用偏好数据和阈值做决策，并显示训练数据覆盖对效果影响很大。
- 主线 B：它不去认识具体模型，只判断“这题要不要强模型”，所以换一对新模型也能直接用，一道题都不测。
