# 03 IRT-Router：把模型当考生、问题当试题，凭简介猜新模型的能力

> **在两条主线上的位置**（两条主线见 [README](README.md#两条主线)）
>
> - **主线 A（来一个 query，怎样选模型）· 第 3 篇：给质量预测加上结构。** 上一篇 [RouteLLM](02_RouteLLM_精读核验.md) 只做强弱二选一。IRT-Router 回到多模型选择，引入 ZeroRouter、RADAR 都会沿用的框架——**项目反应理论（IRT）**：题目有难度和区分度，模型有能力，两者配对就能预测作答表现，再与成本一起打分。它留下的问题：平均性能领先，Top-1 却只有 2.72%；选模型时用常数成本，评测时用逐题费用。
> - **主线 B（新模型怎样接入）· 第 1 站（零反馈）之二。** 新模型的能力从一段**模型简介**映射出来，仍是零题接入。但简介未必反映真实能力，新模型的预测 AUC 只有 0.62。**既然凭空猜不准，能不能让新模型做几道题？** 这是 [ZeroRouter](04_ZeroRouter_精读核验.md) 的出发点。
>
> 原论文：[arXiv:2506.01048v2](https://arxiv.org/abs/2506.01048v2)（2025.06）· [中文译文](translations/03_IRT-Router_中文译文.md) · [中文 PDF](pdfs/zh/IRT-Router_Chinese.pdf)
> 全仓库统一记号见 [README](README.md#统一记号)。

## 链路速查

| 环节 | IRT-Router 的做法 |
|---|---|
| 候选池 | 20 个模型或配置（如 GPT-4o Mini 与 GPT-4o Mini + CoT 算两个） |
| 问题表示 | `bert-base-uncased` 768 维 → 题目参数 $a_q,b_q$ |
| 模型表示 $z_m$ | 模型简介的文本向量 → 25 维能力 $\theta_m$ |
| 质量预测 $\widehat P(q,m)$ | MIRT 或 NIRT |
| 成本 $\widehat C(m)$ | 归一化的输出 token 单价，每模型一个常数 |
| 决策 | $\alpha\widehat P-\beta C$，$\alpha+\beta=1$（$\alpha$ 即统一记号的 $w$） |
| 新模型接入 | 写一段简介，0 题 |
| 主指标 | Performance、Total Cost、Reward、Top-k 准确率 |

## 1. 题目和模型怎样变成能力参数

一个学生能否答对，取决于题目考什么、他擅长什么。IRT-Router 把这个关系用到模型上：

- **题目**：`bert-base-uncased` 编码为 768 维 $e_q$，再映射成区分度 $a_q$、难度 $b_q$。[嵌入方法](translations/03_IRT-Router_中文译文.md#p-s4-1-01)
- **模型**：先写一份自然语言简介（发布时间、开发机构、类型、特点、简短描述，由 ChatGPT 联网生成后人工校正），编码为 $e_M$，再映射成能力 $\theta_M$。能力不是由参数量决定的，而是学“简介文字与实际成绩之间的关系”。[简介来源，英文附录 A.1](translations/03_IRT-Router_中文译文.md#app-a-1)

两个版本的参数形状不同（都用 25 维）：

| 参数 | MIRT-Router | NIRT-Router |
|---|---|---|
| 模型能力 $\theta_M$ | 25 维潜在向量 | 25 维向量，经 sigmoid |
| 题目区分度 $a_q$ | 25 维向量 | 标量 |
| 题目难度 $b_q$ | 标量 | 25 维向量 |
| 能力相关性 $r_q$ | 无 | 25 维，对应预定义能力 |

**MIRT：**

$$
\widehat P(q,M)=\sigma\!\left(a_q^\top\theta_M-b_q\right),\qquad \theta_M=W_\theta e_M.
$$

题目看重的能力方向与模型能力越匹配、难度越低，预测分数越高。例：A 擅长数学、B 擅长代码，平均成绩相近，一道数学权重高的题更适合 A（教学例子）。这 25 维是学出来的潜在方向，不能直接叫作“数学能力”“代码能力”。[MIRT 公式](translations/03_IRT-Router_中文译文.md#p-s4-2-1-01)

**NIRT：** 预先规定 25 个维度的含义，再加相关性向量：

$$
x_{qM}=r_q\odot(\theta_M-b_q)a_q,\qquad
\widehat P(q,M)=\sigma\!\left(\phi(W_1x_{qM}^{\top}+b_1)\right).
$$

$r_q$ 表示题目需要哪些能力，$\theta_M-b_q$ 表示模型在各维度上是否达到要求，后面的网络学习组合方式。[NIRT 公式](translations/03_IRT-Router_中文译文.md#eq-7)

$r_q$ 的来源：训练题先做 UMAP 降维 + HDBSCAN 聚类，每簇抽 5 道让 LLM 判断涉及哪些预定义能力；测试题取 5 个最近训练邻居的 $r$ 均值再归一化，不读测试题的真实标签。[训练题相关性](translations/03_IRT-Router_中文译文.md#p-s4-2-2-03) · [测试题近邻估计](translations/03_IRT-Router_中文译文.md#p-s4-2-2-04)

两个版本都用逐题实际评分 $y_{qM}\in[0,1]$ 和二元交叉熵训练，允许软标签。[训练目标](translations/03_IRT-Router_中文译文.md#eq-6)

## 2. 一道新问题怎样被路由

**查询预热。** 找训练库中最相近的 5 道题，把新题向量往它们的均值挪一点：

$$
e'_q=(1-\lambda)e_q+\lambda\,\overline e_{NN(q)}.
$$

$\lambda=0.3$ 即保留 70% 原向量。这一步只校正题目输入，不改模型能力，与新模型接入无关。消融显示它在 OOD 上更有用，对 NIRT 影响尤其明显。[查询预热](translations/03_IRT-Router_中文译文.md#eq-10) · [预热实验](translations/03_IRT-Router_中文译文.md#p-s6-3-02)

**决策：**

$$
M^*(q)=\arg\max_M\left\{\alpha\widehat P(q,M)-\beta C(M)\right\},\qquad \alpha+\beta=1,
$$

实验取 $\alpha=0.8、0.5、0.2$。[决策公式](translations/03_IRT-Router_中文译文.md#eq-2) · [候选配置，英文附录 A.2](translations/03_IRT-Router_中文译文.md#app-a-2)

**成本的两个口径。** 选模型时 $C(M)$ 是归一化的输出 token 单价，不随题目变（例：GLM-4-Flash 每百万输出 token 0.0137 美元，GPT-4o 10 美元，DeepSeek-Chat 0.28 美元映射为 0.028）。最终报告的 Total Cost 则按每题实际输入、输出 token 数分别计价再求和。[成本定义](translations/03_IRT-Router_中文译文.md#p-s3-1-02) · [价格示例](translations/03_IRT-Router_中文译文.md#fig-1) · [实际总费用](translations/03_IRT-Router_中文译文.md#eq-11)

$\alpha=0.8$ 时 DeepSeek-Chat 的成本罚分只有 $0.2\times0.028=0.0056$。罚分小，并不等于成本对决策无效；会不会换模型，还要看两个候选的质量差与价格差。

## 3. 主实验

12 个数据集、20 个候选：8 个 ID 数据集（MMLU、CMMLU、ACLUE、ARC-Challenge、HotpotQA、SQuAD、MATH、MBPP）各按 70%/30% 划分后合并；4 个 OOD 数据集（CEVAL、CommonsenseQA、GSM8K、HumanEval）。基线有固定小/大模型、HybridLLM、RouteLLM 和 RouterBench 预测式路由。[数据划分](translations/03_IRT-Router_中文译文.md#p-s5-1-01) · [基线](translations/03_IRT-Router_中文译文.md#s5-3)

$\alpha=0.8$ 时：

| 场景 | RouterBench | MIRT | NIRT | MIRT 实际总费用 |
|---|---:|---:|---:|---:|
| ID | 80.01% | 80.67% | 80.69% | 0.42 美元 |
| OOD | 85.50% | 87.12% | 87.37% | 0.14 美元 |

数值见[表 1](translations/03_IRT-Router_中文译文.md#table-1) 和[表 2](translations/03_IRT-Router_中文译文.md#table-2)。ID、OOD 是不同题目和评分，只能在同一行内比方法。

**平均分好看，但很少选中“最优”那个。** MIRT 在 20 个候选中的 Top-1 路由准确率只有 ID 2.72%、OOD 2.15%；Top-3 为 32.51%、27.29%，Top-5 为 47.85%、39.47%。作者解释说很多模型的综合得分很接近，没选中第一名不一定损失很大。但这仍提醒我们：**平均成绩高**和**逐题选对模型**是两个不同的目标。[Top-k 结果](translations/03_IRT-Router_中文译文.md#p-s6-2-06)

## 4. 新模型只凭简介：能走多远

用 Claude 3.5 Haiku 20241022 测试：写简介 → 得到能力 → 预测它在 ID 测试题上的表现。[新模型实验](translations/03_IRT-Router_中文译文.md#p-s6-3-01) · [表 4](translations/03_IRT-Router_中文译文.md#table-4)

| 方法 | RMSE ↓ | MAE ↓ | AUC ↑ | ACC ↑ |
|---|---:|---:|---:|---:|
| RouterBench | 0.62 | 0.55 | 0.50 | 0.34 |
| MIRT-Router | 0.45 | 0.43 | 0.62 | 0.67 |
| NIRT-Router | 0.45 | 0.42 | 0.62 | 0.68 |

比基线好，但 AUC 0.62 离可靠还远。更关键的是，论文报告的是**预测误差**，没有报告加入 Claude 之后整条路由曲线提升多少。“零题得到能力表示”不等于“零题证明接入有收益”。

从方法上看，公司名、发布时间、宣传文字未必反映数学和编程能力；用约 20 个候选学这种映射，外推到新模型可能不稳；同一底座加不加 CoT 的差别，靠描述也很难识别。这些是待验证的风险。

**这正是主线 B 的转折点**：凭简介猜不准，下一步自然是让新模型做一些题、用答题结果来估能力。

## 5. 容易读错的地方

- 87.12% 是 OOD 成绩；OOD 分数更高不代表“越分布外越准”，因为题目不同。
- 查询预热不是新模型 profiling，它只改题目向量。
- 候选含“GPT-4o Mini + CoT”这种使用方式，但还不是 [RADAR](06_RADAR_精读核验.md) 那种完整的“模型 × 推理预算”。
- 主表只比较三组 $\alpha$，没有 RouterBench 式的连续成本—质量曲线，也没有完整的多种子方差报告。
- MIRT 要保证“能力越高预测越好”，需要 $a_q$ 各分量非负；论文写了单调性，但实现是否有 softplus 等约束、BERT 是否冻结，需要查代码。

## 6. 对研究的用处

作者自己承认：带标准答案的短问题不能覆盖真实查询分布，模型对成本权重不够敏感，也没有能力排序约束。[原文局限](translations/03_IRT-Router_中文译文.md#limitations) 在此基础上可以做的实验（研究建议）：

**对主线 A：**

- **看错选损失而不只看 Top-1**：计算 $U^*(q)-U_{\text{selected}}(q)$。错选若多是近似并列，损失很小；若集中在某类任务，才能找出机制问题。
- **拆开成本的作用**：固定质量预测器，只换成本口径（固定单价、预测逐题成本、真实费用上界），比较决策翻转比例和曲线。
- **检查实现**：参数变换、激活函数、单调性，以及能力维度的语义是否跨数据集稳定。

**对主线 B：**

- **简介 + 少量实测**：先用简介得到初始 $\theta$，再给新模型 0、8、16、32 道题校正，看路由效用是否改善，并与 ZeroRouter、RADAR 的选题方法同台比较。这能回答“简介能省掉多少测试题”。2026 年的 [RouteProfile](https://arxiv.org/abs/2605.00180) 用模型卡、公开榜单分数等更丰富的公开信号建画像，可作为“零反馈”一端更强的基线。

复现配置：Adam，学习率 0.002，batch size 512，一张 A100 40GB。分析路由器不必在本地部署 20 个模型，但从零重建数据需要模型调用或完整响应缓存。[训练配置](translations/03_IRT-Router_中文译文.md#p-s5-5-01) · [映射公式](translations/03_IRT-Router_中文译文.md#eq-4)

**接着读：** [04 ZeroRouter](04_ZeroRouter_精读核验.md)。

- 主线 A：同样用 IRT 预测质量，但成本和延迟改为逐题估计，并支持整批约束。
- 主线 B：新模型的能力不再从简介猜，而是让它做 200 道事先选好的锚点题来估。
