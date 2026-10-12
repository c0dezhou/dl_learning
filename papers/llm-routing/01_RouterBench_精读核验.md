# RouterBench：一篇教你“怎么公平比较路由器”的论文

同一道题可以交给不同的大模型，怎样判断一个路由器真的选得更好？只比较准确率，会把多花钱换来的质量算成路由能力；只比较省钱，又可能掩盖答案变差。已有路由方法缺少统一的比较尺度，RouterBench 因此把重点放在评测环节：预先收集各模型在每道题上的回答、质量和费用，让路由器在同一批记录上选模型，再比较共同成本范围内的质量。它展示的 kNN 和 MLP 先预测各模型的答题表现，把预测质量与价格合成一个分数，最终选分数最高的模型。论文要检验的是：这样逐题选择，究竟比固定用一个模型，或按固定概率混合几个模型，多带来了多少收益。[中文流程图](translations/01_RouterBench_中文精读材料.md#rb-fig-1)

原论文为 Hu 等人的 *RouterBench: A Benchmark for Multi-LLM Routing System*（2024 年 3 月 v2）。[原论文书目信息](https://arxiv.org/abs/2403.12031v2)。下文链接指向已有中文阅读笔记的 Markdown 转写，明确属于[中文精读材料，非全文译本](translations/01_RouterBench_中文精读材料.md#rb-title)。教学例子与研究建议会单独说明，不作为作者的实验结果。

## 为什么要保留每道题上每个模型的表现

假设两个模型平均分相同，价格也相同。它们仍可能各自擅长不同题目：模型 A 做错的题，模型 B 恰好会做。于是，即使总分看不出差别，逐题选模仍可能有收益。这个判断是从路由机制推出来的；真正能否利用这种互补性，还取决于路由器能不能在看到答案之前识别它。事后知道谁答得好，和事先能选对，是两回事。[逐题记录与用途](translations/01_RouterBench_中文精读材料.md#rb-sec-2)

RouterBench 保存了 405,467 条模型—题目结果记录，覆盖八类数据集、64 个任务。这里的 405,467 不是互不重复的题目数。数据集包括 MMLU、HellaSwag、WinoGrande、ARC-Challenge、GSM8K、MBPP、MT-Bench 和自建 RAG；RAG 有约 800 个带参考答案的用户问题。前五类采用 5-shot 与 Exact Match，后三类采用 0-shot 与 GPT-4 Judge；质量得分最终统一到 [0,1]。[数据与判分表](translations/01_RouterBench_中文精读材料.md#rb-table-1)

核心候选池有十一个模型：Llama-70B-chat、Mixtral-8×7B-chat、Yi-34B-chat、Code Llama-34B、Mistral-7B-chat、WizardLM-13B，以及 GPT-4、GPT-3.5-turbo、Claude-instant-v1、Claude-v1、Claude-v2。另有 You.com API、sonar-small-online 和 sonar-medium-online 用于 RAG。这不表示每项实验都在十四个候选中选；RAG 图中实际列的是五个候选。也不要把这篇 2024 年的 RouterBench 与独立的 LLMRouterBench（2026）混淆。[模型池说明](translations/01_RouterBench_中文精读材料.md#rb-supplement-models)

缓存让研究者可以反复训练、评价路由策略，不必每换一次策略就重新调用所有候选模型。但缓存省下的是离线实验的重复推理，线上选中的模型仍然需要生成答案、产生费用。复现时也必须说明，自己只是读取缓存、训练了路由器，还是重跑了模型生成和官方训练流程。[数据字段](translations/01_RouterBench_中文精读材料.md#rb-table-fields)

## 从问题文本到最终选模，中间发生了什么

预测式路由首先把问题编码成向量。kNN 找到训练集中最相似的 $k$ 道旧题，分别计算每个候选模型在这些题上的平均成绩，把它当作模型在当前题上的预测质量：

$$
\widehat P_m(x)=\frac{1}{k}\sum_{j\in N_k(x;D_{\mathrm{train}})}q_{jm}.
$$

这里 $q_{jm}$ 是模型 $m$ 在旧题 $j$ 上的观测质量。假设最近的五道数学题中，模型 A 答对四道、模型 B 答对两道，预测值就是 0.8 和 0.4。这是说明公式的自拟例子，不是 RouterBench 的实验数据；预测也不等于两个模型已经回答过当前题目。[KNN 公式与设置](translations/01_RouterBench_中文精读材料.md#rb-sec-5-2)

MLP 不显式查邻居，而是学一个从题目向量到模型预测质量的函数。两种方法都依赖训练题上的模型成绩；它们没有像 IRT 那样显式设定模型能力向量。kNN 依赖“相似文本上的模型表现可以迁移”，MLP 依赖“训练数据足以学到这种映射”。这些是理解方法的设计解释，不能单凭实验落后就断言哪一个假设一定出了问题。[MLP 公式与说明](translations/01_RouterBench_中文精读材料.md#rb-sec-5-3)

原论文并非没有交代 embedding。附录 B 列出三种 SentenceTransformers 编码器：all-MiniLM-L12-v2、all-mpnet-base-v2、all-distilroberta-v1。kNN 使用余弦相似度，比较 $k\in\{5,10,40\}$，最佳配置是 $k=40$ 加 all-MiniLM-L12-v2。MLP 比较一层或两层隐藏层，每层 100 个神经元，使用 ReLU、学习率 0.001；最佳配置是两层、每层 100，同样使用 all-MiniLM-L12-v2。预测式主实验在任务内按 70% / 30% 划分训练与测试，MT-Bench 因样本太少没有参加这一主对比。[超参数补注](translations/01_RouterBench_中文精读材料.md#rb-appendix-settings)；[数据划分](translations/01_RouterBench_中文精读材料.md#rb-supplement-split)

得到预测质量之后，还要考虑价格。论文的选择规则可写为

$$
m^*(x)=\arg\max_m\left[\lambda\widehat P_m(x)-\widehat c_m\right].
$$

$\lambda$ 表示 willingness to pay，即愿意为质量支付多少。它越大，选择就越偏向质量；这与 $Q-\lambda C$ 中 $\lambda$ 越大越重视省钱的方向不同。用一个自拟例子算一遍：A 的预测质量为 0.8、成本为 0.08，B 为 0.6、0.01，成本均用同一示意单位。当 $\lambda=0.2$，A、B 的分数分别是 0.08、0.11，选 B；当 $\lambda=1$，分数变成 0.72、0.59，选 A。同一个质量预测器可以因付费偏好不同而产生不同路由结果。[选择公式](translations/01_RouterBench_中文精读材料.md#rb-eq-6)

这里还有一个容易漏掉的区别：选模型时用的成本估计，不一定等于最终记分的逐题费用。论文在预测式选择中以每 token 定价近似成本；评价则读取按输入、输出和 API 定价计算的费用，开放权重模型参考 Together AI 价格。短选择题和长文档问题，即使用同一模型，每次请求的花费也可能不同。前一句是论文口径，后一句是解释这个区别的例子。复现时应把两种成本分别写清楚，也不能把当前价格冒充 2024 年原实验价格。[费用口径](translations/01_RouterBench_中文精读材料.md#rb-appendix-cost)

级联路由走的是另一条路径。它先调用便宜模型，检查生成的回答是否达到阈值；不够好且预算允许，再调用后续模型。因此一题可能产生多次调用，费用要累计，延迟结构也与生成前选模不同。RouterBench 用已知质量构造理想 Judge，再以概率 $\varepsilon$ 把评分 $g(o)$ 翻成 $1-g(o)$，模拟判断错误。它没有在这里证明现实中存在一个免费、准确的 Judge。[级联流程与扰动公式](translations/01_RouterBench_中文精读材料.md#rb-sec-5-4)

## 一条曲线比一个分数多告诉我们什么

改变 $\lambda$ 或级联预算，会得到多个成本—质量点。RouterBench 允许把两种策略按固定概率混合：若一半请求用策略 A，另一半用 B，平均成本与平均质量就在两个点的中间。这是期望的线性性，不保证每条请求都满足该平均预算，也不保证每次小样本实验恰好达到期望值。[随机混合公式](translations/01_RouterBench_中文精读材料.md#rb-eq-4)

由此，比较时不能只挑几个孤立点。若一个点能被成本不更高、质量不更低的策略或混合策略替代，它就不该代表最好的可行表现。论文取这些可实现点的非递减上包络，称为非递减凸包（NDCH）。中文材料用 A=(0.2,0.60)、B=(0.8,0.90) 作示意：等概率混合得到 (0.5,0.75)，优于同样成本、质量只有 0.70 的点。这里所有数字都是教学示意。[混合前沿图](translations/01_RouterBench_中文精读材料.md#rb-fig-2)

Zero Router 正是把这套操作应用于所有固定单模型点。它知道各模型总体上有多贵、平均答得多好，却不读取当前题目来决定模型；针对不同平均预算，采用不同的最优固定概率混合。因此它既不是均匀随机挑模型，也不是总选最弱模型。它提出了一个很实际的问题：不分析每道题，仅靠总体信息已经能做到这样，你的逐题预测还能多做多少？这里的 Zero Router 也不是 2026 年的 ZeroRouter 方法论文。[基线定义](translations/01_RouterBench_中文精读材料.md#rb-table-baselines)

另一端的 Oracle 提前知道每道题上所有模型的真实质量，选质量最高者，同分时选其中最便宜的。它让我们看到固定数据和候选池中存在多少事后可利用的空间，但线上没有这些未来答案，因此不能把 Oracle 与真实路由器的差距当作已经可实现的收益。这个 Oracle 也不是任意硬成本约束下完整的最优策略。[Oracle 的范围](translations/01_RouterBench_中文精读材料.md#rb-sec-4)

为了把整条前沿压成一个数，论文使用 AIQ：

$$
\operatorname{AIQ}(R)=\frac{1}{c_{\max}-c_{\min}}
\int_{c_{\min}}^{c_{\max}}\widetilde Q_R(c)\,dc.
$$

可以把它理解为共同成本区间内的平均质量高度。虽然名称是 Average Improvement in Quality，公式却没有减去 Zero；“提高多少”应在相同成本区间内比较 AIQ 差值或直接比较两条曲线。换了积分区间，绝对数值就不能直接横比。还要留意横轴：公式可以写成平均每题成本，原论文许多图用的却是评价集合总成本，跨题量比较需要换算。[AIQ 定义](translations/01_RouterBench_中文精读材料.md#rb-sec-3-4)；[成本与质量的定义](translations/01_RouterBench_中文精读材料.md#rb-sec-3-1)

## 实验真正支持了哪些结论

预测式路由在部分任务上能用更低或相近成本达到较强单模型的表现，但没有稳定、明显地胜过 Zero。下面保留原论文 Figure 4 的 AIQ，读的时候应在同一行内比较。[中文结果表与重绘图](translations/01_RouterBench_中文精读材料.md#rb-table-2)

| 任务 | Zero | kNN | MLP |
|---|---:|---:|---:|
| MMLU | 0.763 | 0.773 | 0.769 |
| ARC-Challenge | 0.877 | 0.871 | 0.875 |
| MBPP | 0.660 | 0.644 | 0.642 |
| HellaSwag | 0.588 | 0.588 | 0.588 |
| WinoGrande | 0.778 | 0.780 | 0.778 |
| GSM8K | 0.632 | 0.632 | 0.623 |

MMLU 上两种方法都高于 Zero；WinoGrande 只有 kNN 略高，MLP 持平；HellaSwag 三者相同；GSM8K 的 kNN 持平、MLP 更低；ARC-Challenge 和 MBPP 则是两种方法都低于 Zero。因此不能写成“kNN / MLP 全面优于静态基线”。作者关于“不显著超过”的表述是定性结论，不能擅自理解成这里报告了统计显著性检验。模型之间仍存在 Oracle 所展示的差距，但这张表本身无法确定预测方法失利的唯一原因。[实验解释及其限度](translations/01_RouterBench_中文精读材料.md#rb-sec-6-1)

级联的结果更依赖 Judge 条件。零错误模拟下，MMLU 的 Cascade AIQ 为 0.901，对应 Zero 为 0.763；MBPP 为 0.862 对 0.660。但 HellaSwag 上却是 0.469 对 0.607，级联反而更差。随着判断错误率升高，性能可能明显退化，论文特别讨论了 $\varepsilon>0.2$ 的情况。不同图中的 Zero 数值也并不完全相同，例如 GSM8K 的级联图为 0.628，不能拿预测式图的 0.632 直接拼成同一组比较。[级联完整数值与条件](translations/01_RouterBench_中文精读材料.md#rb-table-3)

RAG 提供了另一个有价值的观察。Zero 的 AIQ 为 0.632，Cascade（$\varepsilon=0.2$）、MLP、kNN 分别是 0.657、0.675、0.695。涉及近期事件的查询更需要搜索能力，这给路由器提供了可识别的选择依据。不过此时路由的对象包含基础模型和带检索服务的系统，差异不只是模型本身的能力；不能把它与纯模型问答合成一句“路由普遍有效”。[RAG 结果](translations/01_RouterBench_中文精读材料.md#rb-sec-6-3)

## 它怎样帮助研究新模型的少量反馈接入

RouterBench 能提供逐题数据、评价方法和强基线，但没有给出“新模型上线后，只获得少量反馈，怎样决定测哪些题并更新路由”的完整算法与实验。kNN 需要新模型在历史题上的成绩；MLP 也需要相应的训练或适配信息，这是由方法输入推出来的。需要多少反馈、是否必须重训、应该怎样选题，不能仅凭这一点下结论。附录 E 的跨任务泛化实验同样不等于新模型按时间接入。[研究设定的区别](translations/01_RouterBench_中文精读材料.md#rb-sec-7-1)；[补充实验的边界](translations/01_RouterBench_中文精读材料.md#rb-appendix-experiments)

因此，更稳妥的用法是把它当作评测框架与数据资源，再补上新模型到达、反馈预算和信息屏蔽协议。选题算法只能读取已经购买的反馈，最终测试标签只用于事后记分；还需按接入事件、数据来源和反馈重采样检查稳定性，避免把重复使用同一批题的实验格当成独立测试。它是否适合作为主方法对照，还要结合其他文献与实验证据判断。[复现核验清单](translations/01_RouterBench_中文精读材料.md#rb-sec-7-2)

读完这篇论文，值得继续验证的有三个问题。第一，在相同数据和价格体系下，把单模型、Zero、kNN / MLP 与 Oracle 放在一起，先判断逐题互补性到底有多少、能否从输入学出来。第二，把固定价格近似与逐题成本估计代入同一个路由器，检查选择翻转比例和最终曲线；不能预设更细的成本模型一定改善结果。第三，先明确业务约束是批量平均预算，还是每条请求的硬预算，再判断随机混合能否满足它。这些是从论文设计延伸出的研究建议，尚不是已经成立的缺陷或研究空白。[成本依据](translations/01_RouterBench_中文精读材料.md#rb-appendix-cost)；[随机混合的适用含义](translations/01_RouterBench_中文精读材料.md#rb-sec-3-2)

向别人介绍 RouterBench 时，可以抓住这一点：它把“路由比固定模型好”进一步追问为“路由是否胜过同预算下的强静态混合”。前面已有 FrugalGPT 等路由研究，因此它不能被说成最早证明路由有价值；它的贡献在于让这种比较有共同的数据、基线与评价规则。[与前序工作的关系](translations/01_RouterBench_中文精读材料.md#rb-supplement-position)
