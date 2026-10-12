# 04 ZeroRouter：让新模型做 200 道固定“体检题”，估出能力再接入

> **在主线上的位置：第 2 站（固定题单）之一。**
> 上一篇 [IRT-Router](03_IRT-Router_精读核验.md) 凭简介猜新模型的能力，猜得不准。ZeroRouter 沿用 IRT 框架，但让新模型**真的做题**：先在旧模型上建好共享能力空间，新模型只需做一批预先选好的锚点题，就能估出自己的能力。锚点怎么选（D-optimal）是主线上出现的**第一个选题方法**。
> 它留给下一站的问题：题单在新模型答题前就定死了，前几题答得怎样不影响后面测什么。[ICL-Router](05_ICL-Router_精读核验.md) 是另一种固定题单；[RADAR](06_RADAR_精读核验.md) 则让题单随答题结果变化。
>
> 原论文：[arXiv:2601.06220v1](https://arxiv.org/abs/2601.06220v1)（2026.01.09；现有文件以参考文献结束，没有附录）· [中文译文](translations/04_ZeroRouter_中文译文.md#title) · [中文 PDF](pdfs/zh/ZeroRouter_Chinese.pdf)
> 全仓库统一记号见 [README](README.md#统一记号)。

## 链路速查

| 环节 | ZeroRouter 的做法 |
|---|---|
| 候选池 | 两个各 5 个模型的池分别路由；另有 50 个新模型用于动态接入 |
| 问题表示 | DistilBERT `[CLS]` + 11 个结构特征 → 题目参数 $\boldsymbol\alpha_q,\mathbf b_q$ |
| 模型表示 $z_m$ | 20 维能力 $\boldsymbol\theta_m$ |
| 质量预测 $\widehat P(q,m)$ | 多维二参数 IRT |
| 成本 $\widehat C(q,m)$ | **逐题**估计：输出长度 × 单价；另估延迟 $\widehat T$ |
| 决策 | $w_p\widehat P-w_c\widehat C-w_t\widehat T$，可加整批约束 |
| 新模型接入 | 做 200 道 D-optimal 锚点题，拟合 $\boldsymbol\theta_{\text{new}}$，不重训路由网络 |
| 主指标 | Reward（按三组权重分别计算） |

题名里的 zero-shot 指的是“不用重训路由预测网络”，**不是**新模型一题都不用测。

## 1. 先建共享坐标，再让新题、新模型进入坐标

**拟合坐标系。** 收集 Open LLM Leaderboard 上 200 个旧模型在 ID 数据集上的成绩，拟合多维二参数 IRT：每个模型一个 20 维能力 $\boldsymbol\theta_m$，每道题一个 20 维区分度 $\boldsymbol\alpha_q$ 和 20 维难度 $\mathbf b_q$：

$$
\widehat P(q,m)=\sigma\!\left(\boldsymbol\alpha_q^\top(\boldsymbol\theta_m-\mathbf b_q)\right).
$$

可以理解为：模型在各能力方向上离题目要求还差多远，区分度决定各方向的权重。与 IRT-Router 的 MIRT 相比，这里难度是向量、写在括号里；MIRT 的 $\sigma(a_q^\top\theta_m-b_q)$ 用标量难度。写法相近，参数含义不同。[概率公式与参数](translations/04_ZeroRouter_中文译文.md#s3-1-1-calibration)

**新题也要有坐标。** 用户新问的题没有历史成绩，于是再训练一个“题目文本 → 题目参数”的预测器：输入是微调后的 DistilBERT `[CLS]` 表示加 11 个结构特征（如可读性、句法树深度）；难度头预测相对平均难度的残差，区分度头把 20 维按相关性分组，用多个小专家分别预测。[混合特征](translations/04_ZeroRouter_中文译文.md#s3-2-1) · [两个预测头](translations/04_ZeroRouter_中文译文.md#s3-2-2-difficulty-head)

**前期训练并不轻。** IRT 部分用层次贝叶斯 + 随机变分推断（`py-irt`），6,000 个 epoch，Adam 初始学习率 0.1、每 100 个 epoch 乘 0.99；66M 参数的 DistilBERT 预测器微调 40 个 epoch，batch 32，学习率 $3\times10^{-5}$；单张 A800 80GB。“轻量接入”指的是坐标系建好之后的事。[训练配置](translations/04_ZeroRouter_中文译文.md#s4-1-implementation)

## 2. 锚点题怎么选：D-optimal

坐标建好后，新模型只缺一个 $\boldsymbol\theta_{\text{new}}$。问题变成：用哪一小批题测它，才能各个能力方向都测到、又不重复？ZeroRouter 选使区分度外积之和的对数行列式最大的一组：

$$
\mathcal A^*=\underset{\mathcal A\subset\mathcal P,\ |\mathcal A|=N}{\arg\max}\ \log\det\!\left(\sum_{q\in\mathcal A}\boldsymbol\alpha_q\boldsymbol\alpha_q^\top\right).
$$

直观地说，一堆近似题只能测清同一个方向，这个目标鼓励选出方向互补的题。实现是贪心：从 $\epsilon I$ 起，每次加入增益最大的候选，直到预算用完。[锚点选择与贪心算法](translations/04_ZeroRouter_中文译文.md#s3-1-1-anchor-selection)

**关键简化：去掉了 $p(1-p)$。** 一道题对能力估计的完整 Fisher 信息是 $p(1-p)\boldsymbol\alpha\boldsymbol\alpha^\top$，其中 $p$ 取决于新模型尚未知道的能力。为了能提前选题，作者把这个概率因子去掉，只用已校准的区分度。后果是：

- 好处：题单在新模型开始答题前就能确定；
- 代价：一道新模型几乎必对（或必错）的题，区分度再高也会被选上，而它其实提供不了多少信息。

[RADAR](06_RADAR_精读核验.md) 的自适应测验恰好保留了这一项。这是主线上 ZeroRouter → RADAR 最清楚的一条连线。[完整 Fisher 信息](translations/04_ZeroRouter_中文译文.md#eq-2) · [简化后的目标](translations/04_ZeroRouter_中文译文.md#eq-3)

**拟合能力。** 新模型做完锚点，固定题目参数，用二元交叉熵拟合 $\boldsymbol\theta_{\text{new}}$（得分 $y_k\in[0,1]$，允许非二值）。同一批调用还用来统计输出长度和延迟参数。论文实验预算为 200 题，约占训练可见数据的 1%。[能力拟合](translations/04_ZeroRouter_中文译文.md#eq-5) · [接入预算](translations/04_ZeroRouter_中文译文.md#s4-4-onboarding)

## 3. 从预测成绩到选模型：钱和时间也要逐题估

只看答对概率，路由器会总选最强最贵的模型。ZeroRouter 是六篇里唯一在**选模型时**就逐题估计费用和延迟的。

把概率式里的内积展开得 $\boldsymbol\alpha_q^\top\boldsymbol\theta_m-\boldsymbol\alpha_q^\top\mathbf b_q$，作者把 $s_q=\boldsymbol\alpha_q^\top\mathbf b_q$ 叫作任务感知难度，用它预测输出长度：把锚点题的 $s_q$ 分成 $K$ 个区间，记录每个模型在各区间的平均输出 token 数，新题落在哪个区间就查哪个均值。例：某模型低难度区平均输出 80 token、高难度区 400 token（教学例子）。输入长度用该模型的 tokenizer 直接算。[任务感知难度](translations/04_ZeroRouter_中文译文.md#eq-8) · [长度分桶](translations/04_ZeroRouter_中文译文.md#s3-1-2-calibration-bins)

$$
\widehat C(q,m)=\lambda_m^{\mathrm{in}}\ell_{mq}^{\mathrm{in}}+\lambda_m^{\mathrm{out}}\widehat\ell_{mq}^{\mathrm{out}},
\qquad
\widehat T(q,m)=\mathrm{TTFT}_m+\widehat\ell_{mq}^{\mathrm{out}}\mathrm{TPOT}_m.
$$

$\lambda$ 是 token 单价，TTFT 是首 token 时间，TPOT 是每个输出 token 的时间，后两者也由锚点校准。[费用公式](translations/04_ZeroRouter_中文译文.md#eq-6) · [延迟公式](translations/04_ZeroRouter_中文译文.md#eq-11)

**决策**是整批查询上的加权效用：

$$
\max_x\sum_q\sum_m x_{mq}\left(w_p\widehat P_{mq}-w_c\widehat C_{mq}-w_t\widehat T_{mq}\right),\qquad x_{mq}\in\{0,1\}.
$$

可选加总成本、总延迟、最低平均准确率约束。没有跨题约束时，各题独立选最优即可；有整批预算才需要联合求解。[路由目标](translations/04_ZeroRouter_中文译文.md#eq-17) · [可选全局约束](translations/04_ZeroRouter_中文译文.md#eq-18)

实验用三组权重 $(w_p,w_c,w_t)$：Max-Acc $(0.8,0.1,0.1)$、Min-Cost $(0.1,0.8,0.1)$、Min-Lat $(0.1,0.1,0.8)$。表里三列都是对应权重下的 Reward，不是准确率。[评测指标](translations/04_ZeroRouter_中文译文.md#eq-19) · [实验权重](translations/04_ZeroRouter_中文译文.md#s4-1-policy-weights)

## 4. 实验

两个 5 模型池，分别路由（不是 10 选 1）：[模型池](translations/04_ZeroRouter_中文译文.md#s4-1-model-pool)

| 小模型池 | 大模型池 |
|---|---|
| Gemma-3-1B | Gemma-3-27B |
| Gemma-3n-E4B | Qwen3-32B |
| DeepSeek-R1-0528-Qwen3-8B | Mixtral-8×7B-Instruct |
| Llama-3.1-8B-Instruct | DeepSeek-R1-Distill-Llama-70B |
| Phi-4 | Qwen3-235B-A22B |

另有 50 个训练截止后发布的模型用于动态接入，替换时保持 6 个候选。ID：IFEval、BBH、MATH、GPQA、MuSR、MMLU-Pro；OOD：ARC-Challenge、TruthfulQA、HumanEval。[数据集](translations/04_ZeroRouter_中文译文.md#s4-1-datasets)

基线：RouteLLM、FORC、GraphRouter、Model-SAT（正文也称 CIT-LLM-Routing）、随机路由、单模型。

- **主表**：小模型池 ID Max-Acc Reward 0.45（Model-SAT 0.43，GraphRouter 0.38）；大模型池 OOD Max-Acc Reward 0.68（Model-SAT 0.62）。[表 1](translations/04_ZeroRouter_中文译文.md#table-1)
- **新模型接入（同为 200 题）**：D-optimal 的 Max-Acc Reward 0.39，随机选题 0.27，Model-SAT 0.38；D-optimal 的 Min-Cost、Min-Lat 为 −0.34、−0.37。[表 2](translations/04_ZeroRouter_中文译文.md#table-2)

第二组是对主线最重要的结果：**同样 200 题，选哪些题会明显影响接入效果。** 表中 † 表示 Wilcoxon 符号秩检验 $p<0.05$，只适用于被标记的比较。

**动态池**：新模型依次替换旧模型，主路由器不重训（图 3(a)），比只预测新模型平均成绩更接近真实的持续更新。但没有比较“测完后拒绝接入”的策略。图 3(d) 显示任务感知难度与平均输出长度单调相关，支持这个成本信号有用，但不能说明逐题长度误差小。[动态池](translations/04_ZeroRouter_中文译文.md#s4-4-evolving-pool) · [图 3](translations/04_ZeroRouter_中文译文.md#fig-3)

## 5. 容易读错的地方

- “共研究 60 个模型”不等于部署时 60 选 1。
- 0.39 是 Reward，不是 39% 准确率，也不能和 ICL-Router 的准确率横比。
- 公式写的是总和、概率与费用量纲不同，但表值多为 0.x；单位、平均方式和缩放需查实现，不能自行补归一化系数去对齐。
- 200 是本文报告的预算，不是必须固定的数。
- 正文提到附录，但现有文件没有附录。

## 6. 对主线研究的用处

ZeroRouter 是我们研究问题最直接的出发点之一：它把“新模型接入”压缩成“在共享空间里估一个能力向量”，并且明确提出了一个选题目标。值得检验的地方（研究假说）：

- **估得准 ≠ 选得好。** D-optimal 优化的是能力识别所用的区分度矩阵；部署真正关心的是哪些题会因此改选模型、改选后质量和费用变多少。可以在同一反馈预算下比较固定 D-optimal、随机、以及面向路由决策边界的选题，看 Reward 或路由后悔值，而不只看 $\boldsymbol\theta$ 的 RMSE。
- **被省略的 $p(1-p)$。** 加回反馈信息就会碰到 RADAR 的自适应测验，单凭“加入反馈”不能算新意。
- **预算曲线。** 先画 16、32、64 到 200 题的收益曲线，再讨论是否需要按模型决定何时停测。
- **成本侧。** 先测输出长度 MAE、费用排序错误和最终候选翻转，再判断要不要换更复杂的回归头。

8GB 显存下的起点：保留预计算的题目特征与成绩缓存，先复现锚点选择、能力拟合和最终效用，再决定是否重训文本预测器。

**下一站：** [05 ICL-Router](05_ICL-Router_精读核验.md) 也用固定题单，但不估 IRT 参数，而是把新模型的答题记录直接交给一个训练过的 LLM 路由器去读。
