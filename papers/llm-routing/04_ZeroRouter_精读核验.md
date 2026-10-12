# ZeroRouter：用 200 道固定“体检题”让新模型进入旧路由池

ZeroRouter 研究的是：路由系统已经训练好，后来又发布了一个新模型，怎样用较少测试把它接进来？如果路由器只会输出训练时那几个模型的编号，候选池一变，原来的预测网络往往也要跟着改。ZeroRouter 把这个接口换成了共享能力空间：题目用“难度和区分度”表示，模型用“能力”表示，两者组合后预测作答表现。新模型先完成一批预选锚点题，系统据此估计它的能力、输出长度和延迟；用户的新问题到来时，再综合各候选的预期表现、费用和时间选择模型。论文中的新模型实验用了 200 条反馈，因此这里的 zero-shot 不能理解为“新模型一题也不用测”。它省去的是重新训练整套路由预测网络的过程。[方法总览](translations/04_ZeroRouter_中文译文.md#fig-2) · [200 题接入实验](translations/04_ZeroRouter_中文译文.md#s4-4-onboarding)

全文可读版：[中文译文，含公式、图表和参考文献](translations/04_ZeroRouter_中文译文.md#title)。本文依据 2026 年 1 月 9 日的 arXiv:2601.06220v1；现有中英文文件都以参考文献结束，没有附录。正文虽提到附录，本笔记不把这句话当成附录入口。

## 1. 先建立共同坐标，再让新题和新模型进入这个空间

作者从 Open LLM Leaderboard 收集 200 个旧模型在分布内数据集上的表现，先用多维二参数 IRT 拟合一个共同坐标系。每个模型有 20 维能力向量 $\boldsymbol\theta_m$；每道题有 20 维区分度 $\boldsymbol\alpha_q$ 和难度 $\mathbf b_q$。作答成功概率由下面的式子给出：

$$
\widehat P_{mq}
=\sigma\!\left(\boldsymbol\alpha_q^\top
(\boldsymbol\theta_m-\mathbf b_q)\right).
$$

可以把它理解成：模型在各个潜在能力方向上，距离这道题的要求还有多远，再由区分度决定这些方向对作答结果有多大影响。这些维度是从成绩中拟合出来的，不能直接把每一维都当成已经验证的“数学能力”或“编程能力”。与 IRT-Router 的 MIRT 写法相比，这里的难度 $\mathbf b_q$ 是向量，进入内积；后者的 $\sigma(\mathbf a_q^\top\boldsymbol\theta_m-b_q)$ 使用标量难度。两种写法相近，但参数含义并不相同。[概率公式与参数定义](translations/04_ZeroRouter_中文译文.md#s3-1-1-calibration)

旧题有成绩数据，可以拟合坐标；用户刚提出的新题没有这些数据。为此，ZeroRouter 又训练了一个“文本到题目坐标”的预测器。输入一方面来自微调后的 DistilBERT `[CLS]` 语义表示，另一方面是 11 个结构特征，例如可读性和句法树深度。共享网络融合它们后，难度头预测相对平均难度的残差，区分度头则把 20 个维度按相关性分组，用多个小专家分别预测。这一步把“在历史成绩上拟合题目”接到了“只读新题文字也能估计题目属性”。[混合特征](translations/04_ZeroRouter_中文译文.md#s3-2-1) · [两个预测头](translations/04_ZeroRouter_中文译文.md#s3-2-2-difficulty-head)

训练并不轻到可以忽略：IRT 部分采用层次贝叶斯设定和随机变分推断，用 `py-irt` 实现，训练 6,000 个 epoch，Adam 初始学习率 0.1，每 100 个 epoch 乘 0.99；66M 参数的 DistilBERT 预测器微调 40 个 epoch，batch 为 32，学习率为 $3\times10^{-5}$。作者报告使用单张 A800 80GB。所谓轻量接入，是这个公共空间和预测器已经建立之后的工作量。[训练配置](translations/04_ZeroRouter_中文译文.md#s4-1-implementation)

## 2. 200 道锚点怎样选，为什么不算自适应测试？

共享坐标系建好后，新模型只需要补上自己的能力向量。问题变成：拿哪一小批题来测它，才能减少重复、尽量覆盖不同的能力方向？ZeroRouter 使用 D-optimal 设计，选择让区分度外积之和的对数行列式最大的锚点集合：

$$
\mathcal A^*=\underset{\mathcal A\subset\mathcal P,\ |\mathcal A|=N}
{\arg\max}\ \log\det\!\left(
\sum_{q\in\mathcal A}\boldsymbol\alpha_q\boldsymbol\alpha_q^\top
\right).
$$

直观地说，反复出很多近似题，可能只测清了同一个方向；这个目标鼓励所选题在潜在能力方向上提供互补信息。实际算法从 $\epsilon I$ 开始，逐题加入使目标增益最大的候选，直到达到预算。[锚点选择与贪心算法](translations/04_ZeroRouter_中文译文.md#s3-1-1-anchor-selection)

这里有一个重要简化。完整 Fisher 信息中的单题贡献是 $p(1-p)\boldsymbol\alpha\boldsymbol\alpha^\top$，其中 $p$ 依赖新模型的未知能力。为了提前选题，作者去掉了概率因子，只用已经校准好的区分度向量。因此，这批锚点在新模型开始回答前就能确定，上一题答对还是答错不会改变下一题。它与 RADAR 根据最新能力估计逐步选题的做法不同；也不能把区分度矩阵的覆盖，直接等同于对所有新模型都最大的真实 Fisher 信息。[完整 Fisher 信息](translations/04_ZeroRouter_中文译文.md#eq-2) · [简化后的目标](translations/04_ZeroRouter_中文译文.md#eq-3)

新模型完成锚点后，系统固定题目参数，通过最小化二元交叉熵拟合 $\boldsymbol\theta_{\mathrm{new}}$。原文将锚点得分记为 $y_k\in[0,1]$，并非要求每个数据源都只能提供严格的二值分数。这些调用同时用来统计输出长度和延迟参数。论文报告的新模型实验预算为 200 题，约占训练可见数据的 1%；这是实验设置，不意味着所有部署都必须固定用 200 题。[能力拟合](translations/04_ZeroRouter_中文译文.md#eq-5) · [接入预算](translations/04_ZeroRouter_中文译文.md#s4-4-onboarding)

## 3. 从预测成绩走到真正选模型，还要估算钱和时间

只估计答对概率，路由器容易总选能力最强、也最贵的模型。ZeroRouter 用同一个题目空间补上成本和延迟估计。把概率式中的内积展开，可得到 $\boldsymbol\alpha_q^\top\boldsymbol\theta_m-\boldsymbol\alpha_q^\top\mathbf b_q$；作者把 $s_q=\boldsymbol\alpha_q^\top\mathbf b_q$ 称为任务感知难度，并用它预测输出长度。[任务感知难度](translations/04_ZeroRouter_中文译文.md#eq-8)

方法是为每个模型建立一张表：先把锚点题的 $s_q$ 分成 $K$ 个区间，再记录每个区间内的平均输出 token 数。新题落在哪个区间，就取该模型对应的均值作为长度估计。作为教学例子，如果某模型低难度区平均输出 80 token，高难度区平均输出 400 token，新题被分入高难度区后便按 400 估算；这只是经验均值，不是对单次回答长度的保证。输入长度则可直接用该模型的 tokenizer 计算。[长度分桶与查表](translations/04_ZeroRouter_中文译文.md#s3-1-2-calibration-bins)

据此，费用与时间分别估计为：

$$
\widehat C_{mq}
=\lambda_m^{\mathrm{in}}\ell_{mq}^{\mathrm{in}}
+\lambda_m^{\mathrm{out}}\widehat\ell_{mq}^{\mathrm{out}},
\qquad
\widehat T_{mq}
=\mathrm{TTFT}_m+\widehat\ell_{mq}^{\mathrm{out}}\mathrm{TPOT}_m.
$$

$\lambda$ 是 API 的 token 单价，TTFT 是首 token 时间，TPOT 是每个输出 token 的生成时间；后两者也用锚点校准。路由发生在候选模型实际生成之前，所以这些量是在估计未来费用与时间。[费用公式](translations/04_ZeroRouter_中文译文.md#eq-6) · [延迟公式](translations/04_ZeroRouter_中文译文.md#eq-11)

最终，每道题都必须选一个模型，优化目标是整个查询集合上的加权效用：

$$
\max_x\sum_q\sum_m x_{mq}
\left(w_p\widehat P_{mq}-w_c\widehat C_{mq}-w_t\widehat T_{mq}\right),
\qquad x_{mq}\in\{0,1\}.
$$

论文还给出总成本、总延迟和最低平均准确率的可选约束。从这个公式可以推得：没有跨题约束时，各题独立选择效用最大的模型即可；有整批预算时，才需要联合安排。不能仅凭作者写了整数规划，就认定每个实验都启用了全局预算限制。[路由目标](translations/04_ZeroRouter_中文译文.md#eq-17) · [可选全局约束](translations/04_ZeroRouter_中文译文.md#eq-18)

实验中的三组权重 $(w_p,w_c,w_t)$ 分别为：准确率优先 $(0.8,0.1,0.1)$、成本优先 $(0.1,0.8,0.1)$、延迟优先 $(0.1,0.1,0.8)$。所以表里的 Max-Acc、Min-Cost、Min-Lat 都是对应策略下的 Reward，不能把 Max-Acc 列直接读成准确率。概率、费用和时间的量纲不同，公式又写成总和，而表值多为 0.x；复现时仍需核实实现中的单位、平均方式和缩放，不能自行补一个归一化系数来对齐结果。[评测指标](translations/04_ZeroRouter_中文译文.md#eq-19) · [实验权重](translations/04_ZeroRouter_中文译文.md#s4-1-policy-weights)

## 4. 实验支持了什么，哪些数字最值得记？

主实验的 10 个核心模型分成两个各含 5 个模型的池，分别路由，并非一次 10 选 1：

| 小模型池 | 大模型池 |
| --- | --- |
| Gemma-3-1B | Gemma-3-27B |
| Gemma-3n-E4B | Qwen3-32B |
| DeepSeek-R1-0528-Qwen3-8B | Mixtral-8×7B-Instruct |
| Llama-3.1-8B-Instruct | DeepSeek-R1-Distill-Llama-70B |
| Phi-4 | Qwen3-235B-A22B |

另外 50 个训练截止后发布的模型用于动态接入评估，动态替换时保持 6 个候选。因而“共研究 60 个模型”不能写成“部署时同时 60 选 1”。分布内测试为 IFEval、BBH、MATH、GPQA、MuSR、MMLU-Pro；分布外测试为 ARC-Challenge、TruthfulQA、HumanEval。这里的 ID/OOD 是本论文的训练划分。[模型池](translations/04_ZeroRouter_中文译文.md#s4-1-model-pool) · [数据集](translations/04_ZeroRouter_中文译文.md#s4-1-datasets)

比较对象包括 RouteLLM、FORC、GraphRouter、Model-SAT、随机路由和单模型；正文基线段称能力指令微调方法为 CIT-LLM-Routing，表格使用 Model-SAT 名称。两组结果能帮助抓住论文的主张：

- 主表中，小模型池的 ID Max-Acc Reward 为 0.45，Model-SAT 为 0.43，GraphRouter 为 0.38；大模型池的 OOD Max-Acc Reward 为 0.68，Model-SAT 为 0.62。[表 1](translations/04_ZeroRouter_中文译文.md#table-1)
- 在同为 200 题的新模型接入实验中，D-optimal 的 Max-Acc Reward 为 0.39，随机选题为 0.27，Model-SAT 为 0.38。D-optimal 的 Min-Cost、Min-Lat Reward 分别为 −0.34、−0.37。[表 2](translations/04_ZeroRouter_中文译文.md#table-2)

第二组结果支持锚点选择确实影响接入效果，但 0.39 不是 39% 准确率，也不能与 ICL-Router 报告的准确率直接横比。表中的 † 来自 $p<0.05$ 的 Wilcoxon 符号秩检验，只适用于被标记的比较，不代表每个单元格都显著，更不是换一个模型池仍能复现的保证。

动态实验让新模型依次替换旧模型，并保持主路由器不重训；图 3(a) 展示了这一过程中的表现。这比只预测新模型的平均成绩更接近持续更新候选池，但该实验没有单独比较“测完后拒绝接入新模型”的策略。图 3(d) 还展示任务感知难度与平均输出长度的单调关系，它支持这个成本估计信号有用，却不足以证明逐题长度误差很小、或候选费用排序总是正确。[动态池实验](translations/04_ZeroRouter_中文译文.md#s4-4-evolving-pool) · [图 3](translations/04_ZeroRouter_中文译文.md#fig-3)

## 5. 如果沿着它做研究，先把选题目标和路由收益连起来

这篇论文留下的一个值得检验的问题是：把新模型能力估得更准，是否一定带来更好的路由？D-optimal 优化的是能力识别所用的区分度矩阵，部署时真正关心的却是哪些题会因此改选模型、改选后质量和费用改变多少。可以在相同反馈预算下比较固定 D-optimal、随机选题，以及面向路由决策边界的选题，再看 Reward 或路由后悔值，不能只看能力向量的 RMSE。这是从估计目标与决策目标的差别提出的研究假说，论文还没有证明边界题一定更好。[选题目标](translations/04_ZeroRouter_中文译文.md#eq-3) · [路由目标](translations/04_ZeroRouter_中文译文.md#eq-17)

另一个起点是检验被省略的 $p(1-p)$：某个新模型若几乎总能答对一道题，这道题即使区分度大，也未必对它的能力估计有很多信息。比较静态设计与反馈引导选题是合理实验，但会直接涉及 RADAR 已有的自适应测试，不能仅凭“加入反馈”就声称新颖。预算方面也应先画 16、32、64 到 200 题的收益曲线，再讨论是否需要按模型决定停止时间；200 只是本文已报告的接入预算。

成本侧可以先测输出长度 MAE、费用排序错误和最终候选翻转，再判断是否值得换更复杂的回归头。现有的均值相关图不能证明长度误差已经主导路由失败。Reward 单位和归一化则是需要先查实现的复现问题，不宜包装成算法贡献。

如果只有 8GB 显存，更实际的起点是保留预计算题目特征和成绩缓存，先复现锚点选择、能力拟合及最终效用，再决定是否重训文本预测器。这是复现路径建议，不是论文验证过的最低硬件要求。向导师概括时，最值得说明的变化是：ZeroRouter 把新模型接入压缩为共享空间中的能力估计，再将它接到质量、费用和时间的统一决策里；固定锚点是否最有利于最终决策，才是后续需要用实验回答的问题。
