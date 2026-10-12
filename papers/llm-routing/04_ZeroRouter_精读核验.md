# ZeroRouter：用 200 道固定“体检题”让新模型进入旧路由池

> 英文题名：**Breaking Model Lock-in: Cost-Efficient Zero-Shot LLM Routing via a Universal Latent Space**  
> 来源：[arXiv:2601.06220v1（英文原文）](https://arxiv.org/html/2601.06220v1)，2026 年 1 月 9 日版本。  
> 核验依据：库中 `ZeroRouter.pdf` 原文，§Methodology、§Experiments、图 2/3、表 1/2。  
> 证据标记：**[原文]** 原文事实；**[推导]** 从公式推导；**[待代码]** 必须查看实现；**[待实验]** 研究假说而非已证实结论。

## 原文定位导航（本次增补）

- 中译依据：Library 中现有 `ZeroRouter_Chinese_Full_Translation.pdf`，14 页，非作者官方译稿；首页标明 arXiv:2601.06220v1，2026-01-09。
- 英文核验：`ZeroRouter.pdf`，8 页；首页实际版本为 **arXiv:2601.06220v1 [cs.LG], 9 Jan 2026**。[固定版本原文 PDF](https://arxiv.org/pdf/2601.06220v1#page=1)。本次全部定位锁定 v1。
- 页码口径：下文“中译 PDF 第 N 页”和“原文 PDF 第 N 页”均从文件首页起按 **1-based 物理页**计数。中译页脚 1–14 与物理页一致；第 10 页为旋转排版的表 1。不要把英文页数套到中译上。英文原文的这些章节标题不带数字；下文“中译§”是译稿的导航编号，不冒充原文章号。
- 证据口径：“原文事实”表示作者在本版写明；“本文推导/讲解”仅给推导起点；“范围核验”是检查指定章节/图表后的有限结论；“待代码/待实验”仍未证实。教学数字和研究建议没有对应的作者结论。英文 Ctrl+F 可在原文中使用；中译关键词仅在对应中译文件搜索。

| 要找什么 | 中译物理页 | 英文一键定位 | 公式、图表或关键词 |
|---|---:|---|---|
| 20维 IRT 与训练配置 | 4、9 | [原文第2页](https://arxiv.org/pdf/2601.06220v1#page=2)；[第5页](https://arxiv.org/pdf/2601.06220v1#page=5) | Eq.(1)；Implementation Details |
| 固定 D-optimal 与能力拟合 | 4–5 | [原文第2–3页](https://arxiv.org/pdf/2601.06220v1#page=2) | Eq.(2–5)；Information-Theoretic Anchor Selection |
| 成本、长度分桶、延迟 | 5–6 | [原文第3–4页](https://arxiv.org/pdf/2601.06220v1#page=3) | Eq.(6–11)；Output Token Estimation |
| 语义/结构与两预测头 | 6–7 | [原文第4页](https://arxiv.org/pdf/2601.06220v1#page=4) | Eq.(12–16)；Unified Prediction Model |
| ILP 与 Reward / 权重 | 7–9 | [原文第4–5页](https://arxiv.org/pdf/2601.06220v1#page=4) | Eq.(17–19)；Evaluation Metrics |
| 两组各5模型、主结果 | 8、10 | [原文第5–6页](https://arxiv.org/pdf/2601.06220v1#page=5) | LLM Candidates；Table 1 |
| 200题/1%、6模型动态池 | 11–12 | [原文第6–7页](https://arxiv.org/pdf/2601.06220v1#page=6) | Efficient Onboarding；Figure 3(a)；Table 2 |

注意：英文第 5 页提到“see the Appendix”，但本次核验的 8 页 v1 PDF 在第 8 页以参考文献结束，未附该附录；没有为其补造定位。

## 一句话先懂

很多传统路由器只认识训练时出现过的那几个模型，一加入新模型就得改最后的分类层或重新采数据。**ZeroRouter 的解决方案是：把“题目属性”和“模型本事”拆到一个共享的 20 维能力空间；这个空间建立好后，新模型只做一小批预选的代表性题，就能给它估能力、费用和延迟，直接加入路由器。**

【定位·原文事实＋本文概括：中译 PDF 第 2 页；原文 Introduction；中译§1，[原文 PDF 第 1 页](https://arxiv.org/pdf/2601.06220v1#page=1)；Ctrl+F：中译“模型锁定”／原文“model lock-in”；另见：中译 PDF 第 4 页；原文 Model Inference Accuracy Estimation，Eq.(1)；中译§3.1.1，[原文 PDF 第 2 页](https://arxiv.org/pdf/2601.06220v1#page=2)；Ctrl+F：中译“层次贝叶斯”／原文“hierarchical Bayesian”；另见：中译 PDF 第 9 页；原文 Implementation Details；中译§4.1，[原文 PDF 第 5 页](https://arxiv.org/pdf/2601.06220v1#page=5)；Ctrl+F：中译“6,000”／原文“6,000”。20维为实验设置，不是方法只允许20维。】

它不是“完全不需要测试新模型”，而是“**不必为新模型重训整个题目路由网络，仍需 200 条锚点反馈**”。题名中的 zero-shot 更多体现跨任务/新问题泛化与避免固定模型分类式重训，不能照字面理解成新模型 0 题即可可靠接入。

【定位·原文事实＋术语解读：中译 PDF 第 11 页；原文 Efficient Onboarding with Minimal Data；中译§4.4，[原文 PDF 第 6 页](https://arxiv.org/pdf/2601.06220v1#page=6)；Ctrl+F：中译“200 个查询”／原文“200”；另见：中译 PDF 第 2 页；原文 Introduction；中译§1，[原文 PDF 第 1 页](https://arxiv.org/pdf/2601.06220v1#page=1)；Ctrl+F：中译“模型锁定”／原文“model lock-in”。200是本版实验预算；“zero-shot”的边界解释为本文澄清，不表示作者将该术语严格等同于零反馈。】

完整流程：大量旧模型测题数据 → IRT 联合学习题目坐标与模型坐标 → 从题目中选 200 个覆盖面广的锚点 → 新模型做这 200 题 → 拟合其能力向量并统计费用/延迟 → 用户新题进入时预测题目坐标、对候选模型逐个算成绩/成本/延迟 → 按偏好选模型。

【定位·本文流程概括：中译 PDF 第 3 页；原文 Figure 2；中译§3 方法，[原文 PDF 第 3 页](https://arxiv.org/pdf/2601.06220v1#page=3)；Ctrl+F：中译“框架概览”／原文“Figure 2”；另见：中译 PDF 第 4 页；原文 Information-Theoretic Anchor Selection，Eq.(2–4)；中译§3.1.1，[原文 PDF 第 2–3 页](https://arxiv.org/pdf/2601.06220v1#page=2)；Ctrl+F：中译“贪心前向选择”／原文“greedy forward”；另见：中译 PDF 第 11 页；原文 Efficient Onboarding with Minimal Data；中译§4.4，[原文 PDF 第 6 页](https://arxiv.org/pdf/2601.06220v1#page=6)；Ctrl+F：中译“200 个查询”／原文“200”。能力、费用、延迟分别还见下文 Eq.(5–11)。】

## ① 候选空间：不是 10 个一起路由

[原文，§Experiments、表 1] 核心验证使用 **10 个模型**，但是分两组实验，每组 **5 个**：

【定位·原文事实：中译 PDF 第 10 页；原文 Table 1（两组各5模型及显著性表注），[原文 PDF 第 6 页](https://arxiv.org/pdf/2601.06220v1#page=6)；Ctrl+F：中译“Wilcoxon”／原文“Wilcoxon”。表中分组标题明确写各5模型；型号全名对照中译第8页/原文第5页 LLM Candidates。】

| 小模型池（5 个） | 大模型池（5 个） |
|---|---|
| Gemma-3-1B | Gemma-3-27B  【定位·原文事实：中译 PDF 第 10 页；原文 Table 1（两组各5模型及显著性表注），[原文 PDF 第 6 页](https://arxiv.org/pdf/2601.06220v1#page=6)；Ctrl+F：中译“Wilcoxon”／原文“Wilcoxon”】 |
| Gemma-3n-E4B | Qwen3-32B  【定位·原文事实：中译 PDF 第 10 页；原文 Table 1（两组各5模型及显著性表注），[原文 PDF 第 6 页](https://arxiv.org/pdf/2601.06220v1#page=6)；Ctrl+F：中译“Wilcoxon”／原文“Wilcoxon”】 |
| DeepSeek-R1-0528-Qwen3-8B | Mixtral-8×7B-Instruct  【定位·原文事实：中译 PDF 第 10 页；原文 Table 1（两组各5模型及显著性表注），[原文 PDF 第 6 页](https://arxiv.org/pdf/2601.06220v1#page=6)；Ctrl+F：中译“Wilcoxon”／原文“Wilcoxon”】 |
| Llama-3.1-8B-Instruct | DeepSeek-R1-Distill-Llama-70B  【定位·原文事实：中译 PDF 第 10 页；原文 Table 1（两组各5模型及显著性表注），[原文 PDF 第 6 页](https://arxiv.org/pdf/2601.06220v1#page=6)；Ctrl+F：中译“Wilcoxon”／原文“Wilcoxon”】 |
| Phi-4 | Qwen3-235B-A22B  【定位·原文事实：中译 PDF 第 10 页；原文 Table 1（两组各5模型及显著性表注），[原文 PDF 第 6 页](https://arxiv.org/pdf/2601.06220v1#page=6)；Ctrl+F：中译“Wilcoxon”／原文“Wilcoxon”】 |

此外还有**50 个训练截止后发布的模型**用于动态接入评估；动态替换实验中的候选池设置为 6 个。不要写“每次选择 10 个核心模型中的最佳一个”；那会改变竞争程度，也会影响所有绝对成绩。这里的“60 个模型”是跨不同验证阶段的整体研究覆盖，不等于同一时刻一个 60 选 1 的实际候选池。

【定位·原文事实＋范围澄清：中译 PDF 第 8 页；原文 Experiments / LLM Candidates；中译§4.1，[原文 PDF 第 5 页](https://arxiv.org/pdf/2601.06220v1#page=5)；Ctrl+F：中译“另外50”／原文“50 distinct”；另见：中译 PDF 第 11–12 页；原文 Robustness in Real-world Evolving Model Pool，Figure 3(a)；中译§4.4，[原文 PDF 第 6–7 页](https://arxiv.org/pdf/2601.06220v1#page=6)；Ctrl+F：中译“固定规模”／原文“fixed-size pool”。60为两个验证集合的覆盖数；不是Table 1的同时候选数。】

## ② 题目表示：语义 + 结构的“双路输入”

[原文，§Context-Aware Latent Space Coordinate Prediction] 一道题同时提取两种信息：

1. **语义向量**：经过 `DistilBERT-base-uncased`，取 `[CLS]` 表示。该编码器在实验中**微调**，不是冻结不动。
2. **11 个手工结构特征**：例如可读性、句法树深度等；这些特征根据其与 IRT 参数的相关性筛选。

【定位·原文事实：中译 PDF 第 6 页；原文 Hybrid Feature Representation，Eq.(12–14)；中译§3.2，[原文 PDF 第 4 页](https://arxiv.org/pdf/2601.06220v1#page=4)；Ctrl+F：中译“结构特征”／原文“Structural Features”。DistilBERT-base-uncased型号与训练超参数另见中译第9页 Implementation Details。】

两类特征拼接后通过共享网络，再产生两个预测头：**难度头**预测相对平均难度的残差、再加回均值；**区分度头**将 20 个预测维度按相关关系分组，使用多组小专家分别学习。[原文]

【定位·原文事实：中译 PDF 第 6–7 页；原文 Unified Prediction Model，Eq.(14–16)；中译§3.2.2，[原文 PDF 第 4 页](https://arxiv.org/pdf/2601.06220v1#page=4)；Ctrl+F：中译“残差向量”／原文“Residual Prediction”；另见：中译 PDF 第 9 页；原文 Implementation Details；中译§4.1，[原文 PDF 第 5 页](https://arxiv.org/pdf/2601.06220v1#page=5)；Ctrl+F：中译“6,000”／原文“6,000”。Eq.(15)是残差加均值；Eq.(16)为分簇专家输出的拼接。】

**为什么要这样设计？** 文本语义编码器擅长表示“在问什么”，结构特征可能补充“题目组织复杂不复杂”。残差估计可能比直接预测高维难度更稳定；专家结构则希望捕捉区分度维度之间的相关性。这些是设计动机，不是跨领域泛化成功的数学保证。

【定位·本文讲解：中译 PDF 第 6–7 页；原文 Unified Prediction Model，Eq.(14–16)；中译§3.2.2，[原文 PDF 第 4 页](https://arxiv.org/pdf/2601.06220v1#page=4)；Ctrl+F：中译“残差向量”／原文“Residual Prediction”。稳定性和泛化保证的保留判断是本文解读，不是额外定理。】

## ③ 模型表示：20 维能力 θ 从哪里学？

[原文，§Universal Latent Space、Implementation Details] 先拿到训练阶段约 **200 个旧模型**的 Open LLM Leaderboard 表现数据，通过多维二参数 Logistic IRT 构建共同坐标系。对每个模型拟合一个 20 维 `θ_m`，每道题也有 20 维区分度 `α_q`、20 维难度 `b_q`。作者采用层次贝叶斯设定和随机变分推断（SVI），实现采用 `py-irt`。

【定位·原文事实：中译 PDF 第 4 页；原文 Model Inference Accuracy Estimation，Eq.(1)；中译§3.1.1，[原文 PDF 第 2 页](https://arxiv.org/pdf/2601.06220v1#page=2)；Ctrl+F：中译“层次贝叶斯”／原文“hierarchical Bayesian”；另见：中译 PDF 第 9 页；原文 Implementation Details；中译§4.1，[原文 PDF 第 5 页](https://arxiv.org/pdf/2601.06220v1#page=5)；Ctrl+F：中译“6,000”／原文“6,000”。200旧模型与新模型200道题是两个不同数量，不能混同。】

**训练数字**：潜在维数 `D=20`，IRT 参数训练 6000 epoch、Adam 初始学习率 0.1，逐渐衰减；题目参数预测器为 66M 参数的 DistilBERT 结构，微调 40 epoch，batch 32，学习率 `3×10⁻⁵`。论文报告在**单张 NVIDIA A800 80GB**上完成训练。[原文，Implementation Details]

【定位·原文事实：中译 PDF 第 9 页；原文 Implementation Details；中译§4.1，[原文 PDF 第 5 页](https://arxiv.org/pdf/2601.06220v1#page=5)；Ctrl+F：中译“6,000”／原文“6,000”。原文还写每100 epoch将学习率乘0.99。】

这本质上是 **先校准，再学习文字到题目坐标的预测器**：

- 第一步，成绩数据告诉你旧题的“真实”潜在坐标（统计拟合的潜变量）。
- 第二步，训练神经网络学习“只读新题文字，能否预测它在这个坐标系的位置”。

【定位·本文流程解读：中译 PDF 第 4 页；原文 Model Inference Accuracy Estimation，Eq.(1)；中译§3.1.1，[原文 PDF 第 2 页](https://arxiv.org/pdf/2601.06220v1#page=2)；Ctrl+F：中译“层次贝叶斯”／原文“hierarchical Bayesian”；另见：中译 PDF 第 6–7 页；原文 Unified Prediction Model，Eq.(14–16)；中译§3.2.2，[原文 PDF 第 4 页](https://arxiv.org/pdf/2601.06220v1#page=4)；Ctrl+F：中译“残差向量”／原文“Residual Prediction”。“真实”在此仅指已拟合的潜变量标签，不是可直接观测的真能力。】

这样新模型和新问题都可以落在同一共享空间里比较，而不是学习一个写死了输出 5/10 个候选 ID 的分类头。

【定位·本文方法解读：中译 PDF 第 3 页；原文 Figure 2；中译§3 方法，[原文 PDF 第 3 页](https://arxiv.org/pdf/2601.06220v1#page=3)；Ctrl+F：中译“框架概览”／原文“Figure 2”；另见：中译 PDF 第 5 页；原文 Lightweight Profiling for New Models，Eq.(5)；中译§3.1.1，[原文 PDF 第 3 页](https://arxiv.org/pdf/2601.06220v1#page=3)；Ctrl+F：中译“二元交叉熵”／原文“Lightweight Profiling”。由模型能力与查询参数分离得出。】

## ④ 成绩预测：为什么它与 IRT-Router MIRT 公式不完全相同？

[原文，Eq.(1)]

$$
\widehat P_{mq}=\sigma\left(\boldsymbol\alpha_q^\top(\boldsymbol\theta_m-\mathbf b_q)\right).
$$

【定位·原文公式：中译 PDF 第 4 页；原文 Model Inference Accuracy Estimation，Eq.(1)；中译§3.1.1，[原文 PDF 第 2 页](https://arxiv.org/pdf/2601.06220v1#page=2)；Ctrl+F：中译“层次贝叶斯”／原文“hierarchical Bayesian”。这里只更换模型/查询下标并写成sigmoid记号。】

这一公式和 IRT-Router 的 MIRT 版本很像，却不能说完全一样：

$$
\underbrace{\sigma(\mathbf a_q^\top\boldsymbol\theta_m-b_q)}_{\text{IRT-Router MIRT，难度 b 为标量}}
\quad\text{vs.}\quad
\underbrace{\sigma(\boldsymbol\alpha_q^\top(\boldsymbol\theta_m-\mathbf b_q))}_{\text{ZeroRouter，难度 b 为 20 维}}.
$$

【定位·跨论文原文公式对照：中译 PDF 第 4 页；原文 Model Inference Accuracy Estimation，Eq.(1)；中译§3.1.1，[原文 PDF 第 2 页](https://arxiv.org/pdf/2601.06220v1#page=2)；Ctrl+F：中译“层次贝叶斯”／原文“hierarchical Bayesian”；另见：另文 `IRT-Router_中文译文_含图表与公式.pdf`：中译 PDF 第 6–7 页；原文 IRT-Router §4.2.1 MIRT-Router，Eq.(4–5)，[原文 PDF 第 5 页](https://arxiv.org/pdf/2506.01048v2#page=5)；Ctrl+F：中译“难度bi”／原文“difficulty”。IRT-Router的标量b只指MIRT版本；不要推广到它的NIRT版本。】

**直观理解**：ZeroRouter 认为题目可能有 20 个方面的难度，模型也有对应的 20 种能力；一个模型数学强、另一个代码强，可以在不同方向上与题目产生不同的匹配。`α` 指示哪些能力维度更影响作答结果。

【定位·本文直观解释：中译 PDF 第 4 页；原文 Model Inference Accuracy Estimation，Eq.(1)；中译§3.1.1，[原文 PDF 第 2 页](https://arxiv.org/pdf/2601.06220v1#page=2)；Ctrl+F：中译“层次贝叶斯”／原文“hierarchical Bayesian”；另见：中译 PDF 第 9 页；原文 Implementation Details；中译§4.1，[原文 PDF 第 5 页](https://arxiv.org/pdf/2601.06220v1#page=5)；Ctrl+F：中译“6,000”／原文“6,000”。20个维度是潜在坐标，不等于作者逐个命名的20门学科；数学/代码为示意。】

在代数上，ZeroRouter 的括号展开：

$$
\boldsymbol\alpha_q^\top\boldsymbol\theta_m-\boldsymbol\alpha_q^\top\mathbf b_q.
$$

【定位·本文代数推导：中译 PDF 第 4 页；原文 Model Inference Accuracy Estimation，Eq.(1)；中译§3.1.1，[原文 PDF 第 2 页](https://arxiv.org/pdf/2601.06220v1#page=2)；Ctrl+F：中译“层次贝叶斯”／原文“hierarchical Bayesian”。只是Eq.(1)内积展开，不是作者另外编号的公式。】

所以 `s_q=α_qᵀ b_q` 可以看作加权后的**任务感知难度**。作者之后又用它来估计输出 token 长度；这让成绩预测与成本预测分享一个题目难度信号。

【定位·原文事实＋推导：中译 PDF 第 5 页；原文 Model Inference Cost Estimation，Eq.(6–10)；中译§3.1.2，[原文 PDF 第 3 页](https://arxiv.org/pdf/2601.06220v1#page=3)；Ctrl+F：中译“查找表”／原文“lookup table”。Eq.(8)明确定义task-aware difficulty；Eq.(9–10)用于预测输出长度。】

## ⑤ 成本与延迟：本论文最应该掌握的工程细节

[原文，Eq.(6)–(11)] 对每个候选模型 m、每道题 q 估计：

$$
\widehat C_{mq}=\lambda_m^{in}\ell_{mq}^{in}
+\lambda_m^{out}\widehat\ell_{mq}^{out},
$$

【定位·原文公式的预测写法：中译 PDF 第 5 页；原文 Model Inference Cost Estimation，Eq.(6–10)；中译§3.1.2，[原文 PDF 第 3 页](https://arxiv.org/pdf/2601.06220v1#page=3)；Ctrl+F：中译“查找表”／原文“lookup table”。Eq.(6)原文写实际成本C与长度；此处结合Eq.(10)用预测输出长度，故加帽。】

$$
\widehat T_{mq}=TTFT_m+\widehat\ell_{mq}^{out}\cdot TPOT_m.
$$

【定位·原文公式：中译 PDF 第 6 页；原文 Model Inference Latency Estimation，Eq.(11)；中译§3.1.3，[原文 PDF 第 4 页](https://arxiv.org/pdf/2601.06220v1#page=4)；Ctrl+F：中译“首token 时间”／原文“Time to First Token”。TTFT/TPOT按锚点校准，预测值不等于未来每题实测。】

这里 `λ` 是输入/输出 token 的 API 单价，不是 RouterBench 决策式中的支付偏好权重。`TTFT` 是首 token 时间，`TPOT` 是每个输出 token 的时间。

【定位·原文符号释义：中译 PDF 第 5 页；原文 Model Inference Cost Estimation，Eq.(6–10)；中译§3.1.2，[原文 PDF 第 3 页](https://arxiv.org/pdf/2601.06220v1#page=3)；Ctrl+F：中译“查找表”／原文“lookup table”；另见：中译 PDF 第 6 页；原文 Model Inference Latency Estimation，Eq.(11)；中译§3.1.3，[原文 PDF 第 4 页](https://arxiv.org/pdf/2601.06220v1#page=4)；Ctrl+F：中译“首token 时间”／原文“Time to First Token”。这里以ZeroRouter自身Eq.(6)(11)的定义为准；λ是价格。】

**输入 token**：作者使用各模型自己的 tokenizer 直接计算，是确定性的（给定文本和 tokenizer）。**输出 token**：由于尚未真的运行候选模型，长度需要估计。

【定位·原文事实：中译 PDF 第 5 页；原文 Model Inference Cost Estimation，Eq.(6–10)；中译§3.1.2，[原文 PDF 第 3 页](https://arxiv.org/pdf/2601.06220v1#page=3)；Ctrl+F：中译“查找表”／原文“lookup table”。Eq.(7)给定tokenizer计数；Eq.(10)给输出长度估计。】

为了避免给每个模型训练一个额外长度回归网络，作者做了一张查找表：

1. 算每道训练/锚点题的 `s_q=α_qᵀ b_q`。
2. 把 s 按区间分成 `K` 档。
3. 对每个模型、每一档，记录锚点题的平均输出 token 数。
4. 新题属于第 k 档时，就拿对应平均长度作为预测。

【定位·原文流程概括：中译 PDF 第 5 页；原文 Model Inference Cost Estimation，Eq.(6–10)；中译§3.1.2，[原文 PDF 第 3 页](https://arxiv.org/pdf/2601.06220v1#page=3)；Ctrl+F：中译“查找表”／原文“lookup table”。用于分桶及均值校准的是锚点集合A；新题用预测α、b进入该分桶。】

教学示例：某模型在低难度区平均输出 80 token、高难度区平均输出 400 token。新题预测为高难度区，就用 400 估计输出费和生成时间。**这不保证该模型每次都会生成 400 token，只是经验均值。**

【定位·教学示例：中译 PDF 第 5 页；原文 Model Inference Cost Estimation，Eq.(6–10)；中译§3.1.2，[原文 PDF 第 3 页](https://arxiv.org/pdf/2601.06220v1#page=3)；Ctrl+F：中译“查找表”／原文“lookup table”。80和400均为本笔记自拟数字，不是论文实测值；仅演示Eq.(9–10)。】

同样地，TTFT 与 TPOT 在锚点题上统计。[原文] 原文没有给出充分的输出长度逐题预测误差对比，因此不能认定它的成本/延迟估计总能精确排序不同候选。[待实验]

【定位·原文事实＋范围核验：中译 PDF 第 6 页；原文 Model Inference Latency Estimation，Eq.(11)；中译§3.1.3，[原文 PDF 第 4 页](https://arxiv.org/pdf/2601.06220v1#page=4)；Ctrl+F：中译“首token 时间”／原文“Time to First Token”；另见：中译 PDF 第 12–13 页；原文 Validation of Task-Aware Difficulty，Figure 3(d)；中译§4.5，[原文 PDF 第 7 页](https://arxiv.org/pdf/2601.06220v1#page=7)；Ctrl+F：中译“平均输出token 长度”／原文“average output token length”。已查本版Model Inference Cost/Latency Estimation及Figure 3(d)和Table 1–2；图3(d)报告的是分桶平均长度相关性，未给逐题MAE或候选排序误差。不能从未报告推定误差大。】

## ⑥ 最终选择：统一考虑成绩、钱、时间

[原文，Eq.(17)–(19)] 对一批查询，给每条题 q、模型 m 一个决策变量 `x_mq∈{0,1}`，每题必须选恰好一个。目标写为：

$$
\max_{x}\sum_q\sum_m x_{mq}
\big(w_p\widehat P_{mq}-w_c\widehat C_{mq}-w_t\widehat T_{mq}\big).
$$

【定位·原文公式改记号：中译 PDF 第 7 页；原文 Multi-Objective Optimization，Eq.(17–18)；中译§3.3.1，[原文 PDF 第 4 页](https://arxiv.org/pdf/2601.06220v1#page=4)；Ctrl+F：中译“可选全局约束”／原文“optional global”。对应Eq.(17)，约束见Eq.(18)；Eq.(19)是评测Reward，不是这一决策式本身。】

作者将它表述为整数规划（ILP），还写了可选的**整批总成本限制、总延迟限制和最低平均准确率要求**。

【定位·原文事实：中译 PDF 第 7 页；原文 Multi-Objective Optimization，Eq.(17–18)；中译§3.3.1，[原文 PDF 第 4 页](https://arxiv.org/pdf/2601.06220v1#page=4)；Ctrl+F：中译“可选全局约束”／原文“optional global”。原文明确称optional global constraints。】

关键数学细节：[推导] **如果没有启用跨查询的全局约束**，所有题的选择彼此独立，各自选择加权分数最高的模型即可，并不需要复杂的全局整数规划；若真的有总预算，题目之间才需要联合分配。

【定位·本文数学推导：中译 PDF 第 7 页；原文 Multi-Objective Optimization，Eq.(17–18)；中译§3.3.1，[原文 PDF 第 4 页](https://arxiv.org/pdf/2601.06220v1#page=4)；Ctrl+F：中译“可选全局约束”／原文“optional global”。独立性结论以只保留每题恰选一个的约束为前提，作者未在此给出这一简化定理。】

**三类实验权重**：[原文]

| 偏好 | 质量 `w_p` | 成本 `w_c` | 延迟 `w_t` |
|---|---:|---:|---:|
| 准确率优先 Max-Acc | 0.8 | 0.1 | 0.1  【定位·原文数值：中译 PDF 第 9 页；原文 Evaluation Metrics，三组权重；中译§4.1，[原文 PDF 第 5 页](https://arxiv.org/pdf/2601.06220v1#page=5)；Ctrl+F：中译“0.8, 0.1, 0.1”／原文“0.8, 0.1, 0.1”】 |
| 成本优先 Min-Cost | 0.1 | 0.8 | 0.1  【定位·原文数值：中译 PDF 第 9 页；原文 Evaluation Metrics，三组权重；中译§4.1，[原文 PDF 第 5 页](https://arxiv.org/pdf/2601.06220v1#page=5)；Ctrl+F：中译“0.8, 0.1, 0.1”／原文“0.8, 0.1, 0.1”】 |
| 延迟优先 Min-Lat | 0.1 | 0.1 | 0.8  【定位·原文数值：中译 PDF 第 9 页；原文 Evaluation Metrics，三组权重；中译§4.1，[原文 PDF 第 5 页](https://arxiv.org/pdf/2601.06220v1#page=5)；Ctrl+F：中译“0.8, 0.1, 0.1”／原文“0.8, 0.1, 0.1”】 |

**风险点 [推导]**：概率、美元成本和秒级时间单位并不相同，直接线性组合前必须弄清实际缩放规则。文中主公式列出了它们，但对于表里 0.x 的具体 Reward 数值如何从“求和”变成相应尺度，文字不够透明。复现时应核对代码或作者给出的完整标准化方式，不允许随手填个归一化系数对齐结果。

【定位·待代码／范围核验：中译 PDF 第 7 页；原文 Multi-Objective Optimization，Eq.(17–18)；中译§3.3.1，[原文 PDF 第 4 页](https://arxiv.org/pdf/2601.06220v1#page=4)；Ctrl+F：中译“可选全局约束”／原文“optional global”；另见：中译 PDF 第 8–9 页；原文 Evaluation Metrics，Eq.(19)；中译§4.1，[原文 PDF 第 5 页](https://arxiv.org/pdf/2601.06220v1#page=5)；Ctrl+F：中译“总奖励”／原文“total reward”；另见：中译 PDF 第 10 页；原文 Table 1（两组各5模型及显著性表注），[原文 PDF 第 6 页](https://arxiv.org/pdf/2601.06220v1#page=6)；Ctrl+F：中译“Wilcoxon”／原文“Wilcoxon”。本版公式是求和，主表为小数；指定正文未写出把两者接通的完整缩放协议。此处提出复现问题，未证实代码遗漏或结果错误。】

## ⑦ 新模型接入：固定 200 题，D-optimal 不是自适应 Fisher

[原文，Eq.(2)(3)(5)、表 2] 为新模型选择 **200 道 anchor queries**，约占训练可见题目的 1%。关键是**锚点在新模型开始做题之前就选好**，新模型第 1 道的答案不会影响第 2 道选什么，因此它属于固定设计，不是 RADAR 的 sequential adaptive testing。

【定位·原文事实＋方法对照：中译 PDF 第 4 页；原文 Information-Theoretic Anchor Selection，Eq.(2–4)；中译§3.1.1，[原文 PDF 第 2–3 页](https://arxiv.org/pdf/2601.06220v1#page=2)；Ctrl+F：中译“贪心前向选择”／原文“greedy forward”；另见：中译 PDF 第 11 页；原文 Efficient Onboarding with Minimal Data；中译§4.4，[原文 PDF 第 6 页](https://arxiv.org/pdf/2601.06220v1#page=6)；Ctrl+F：中译“200 个查询”／原文“200”；另见：另文 `RADAR_中文译本_正文.pdf`：中译 PDF 第 6–7 页；原文 RADAR §3.5，Eq.(7–9)，[原文 PDF 第 6–7 页](https://arxiv.org/pdf/2509.25426v3#page=6)；Ctrl+F：中译“当前能力估计”／原文“Fisher information”。200和1%见Efficient Onboarding；固定性由Eq.(3–4)仅依赖旧题α推得；RADAR的反馈后更新另见其Eq.(8–9)。】

### 为什么叫 D-optimal？

如果题目区分度是 20 维向量 `α_q`，我们希望选的这批题能够在尽量多的方向上区别模型能力，而不是只重复测试同一方面：

$$
\mathcal A^*=\arg\max_{\substack{\mathcal A\subseteq\mathcal Q\\|\mathcal A|=N}}
\log\det\left(\sum_{q\in\mathcal A}\boldsymbol\alpha_q\boldsymbol\alpha_q^\top\right).
$$

【定位·原文公式：中译 PDF 第 4 页；原文 Information-Theoretic Anchor Selection，Eq.(2–4)；中译§3.1.1，[原文 PDF 第 2–3 页](https://arxiv.org/pdf/2601.06220v1#page=2)；Ctrl+F：中译“贪心前向选择”／原文“greedy forward”。对应Eq.(3)，数值稳定的εI初始化与贪心实现见Eq.(4)。】

完整的 Fisher 信息其实应该是 `p(1−p) ααᵀ`，但新模型还没做题时 θ 不知道，p 也就不确定；作者为了可提前选题，**去掉了概率因子，只保留区分度矩阵**，再采用贪心近似选择。

【定位·原文事实：中译 PDF 第 4 页；原文 Information-Theoretic Anchor Selection，Eq.(2–4)；中译§3.1.1，[原文 PDF 第 2–3 页](https://arxiv.org/pdf/2601.06220v1#page=2)；Ctrl+F：中译“贪心前向选择”／原文“greedy forward”。Eq.(2)保留p(1−p)，其后文字明确因未知θ而简化为Eq.(3)。】

**易懂例子**：一套考试只出 200 道几乎一样的数学题，即使能准确区分数学能力，也可能完全测不出编程和语言能力；D-optimal 希望覆盖不同“能力方向”。

【定位·教学示例：中译 PDF 第 4 页；原文 Information-Theoretic Anchor Selection，Eq.(2–4)；中译§3.1.1，[原文 PDF 第 2–3 页](https://arxiv.org/pdf/2601.06220v1#page=2)；Ctrl+F：中译“贪心前向选择”／原文“greedy forward”。数学/编程/语言的例子不是原文实验；D-optimal最大化信息矩阵体积，不保证题库未包含的能力也得到覆盖。】

新模型完成 200 题后，以其对错反馈拟合 θ，同时用这些锚点校准该模型的输出长度分段表及延迟参数。**不重训路由器的题目编码网络**，只是增加模型档案。[原文]

【定位·原文事实：中译 PDF 第 5 页；原文 Lightweight Profiling for New Models，Eq.(5)；中译§3.1.1，[原文 PDF 第 3 页](https://arxiv.org/pdf/2601.06220v1#page=3)；Ctrl+F：中译“二元交叉熵”／原文“Lightweight Profiling”；另见：中译 PDF 第 5 页；原文 Model Inference Cost Estimation，Eq.(6–10)；中译§3.1.2，[原文 PDF 第 3 页](https://arxiv.org/pdf/2601.06220v1#page=3)；Ctrl+F：中译“查找表”／原文“lookup table”；另见：中译 PDF 第 6 页；原文 Model Inference Latency Estimation，Eq.(11)；中译§3.1.3，[原文 PDF 第 4 页](https://arxiv.org/pdf/2601.06220v1#page=4)；Ctrl+F：中译“首token 时间”／原文“Time to First Token”；另见：中译 PDF 第 11 页；原文 Efficient Onboarding with Minimal Data；中译§4.4，[原文 PDF 第 6 页](https://arxiv.org/pdf/2601.06220v1#page=6)；Ctrl+F：中译“200 个查询”／原文“200”。Eq.(5)拟合新θ；Eq.(9)长度表；Eq.(11)延迟参数。原文锚点得分yk∈[0,1]，并不将所有数据源都严格限定为二值。】

### 论文新模型实验结果要如何正确读？

[原文，表 2] 在 **Max-Acc 加权 Reward** 这一列：D-optimal 为 **0.39**，随机选题 **0.27**，Model-SAT 基线 **0.38**。数字 0.39 **不是 39% 的题目准确率**，只是作者按特定权重构造的效用指标。Min-Cost、Min-Lat 对应的表格值甚至可能为负，也说明不能把它们简单当“正确率”。

【定位·原文数值＋指标释义：中译 PDF 第 12 页；原文 Table 2（新模型锚点采样消融），[原文 PDF 第 7 页](https://arxiv.org/pdf/2601.06220v1#page=7)；Ctrl+F：中译“D-最优性”／原文“D-optimality”；另见：中译 PDF 第 8–9 页；原文 Evaluation Metrics，Eq.(19)；中译§4.1，[原文 PDF 第 5 页](https://arxiv.org/pdf/2601.06220v1#page=5)；Ctrl+F：中译“总奖励”／原文“total reward”。Table 2的三列对应不同权重策略下Reward；不将Max-Acc列名当作纯准确率。】

作者还安排新模型陆续替换旧模型，固定 6 模型池并保持旧路由器不重训；这是比“仅预测新模型成绩”更接近部署接入的实验，但没有报告“如果拒绝这个新模型、不把它加入池子”的专门策略对比。

【定位·原文事实＋范围核验：中译 PDF 第 11–12 页；原文 Robustness in Real-world Evolving Model Pool，Figure 3(a)；中译§4.4，[原文 PDF 第 6–7 页](https://arxiv.org/pdf/2601.06220v1#page=6)；Ctrl+F：中译“固定规模”／原文“fixed-size pool”。Figure 3(a)比较动态池表现；在该实验文字、图3(a)及Table 2中未见专门的拒绝接入策略。后半是本文实验范围判断。】

## ⑧ 评测：结果好坏是有前提的

[原文，§Experiments、表 1/2]

**6 个 ID 基准**：IFEval、BBH、MATH、GPQA、MuSR、MMLU-Pro；**3 个 OOD 基准**：ARC-Challenge、TruthfulQA、HumanEval。

【定位·原文事实：中译 PDF 第 8 页；原文 Experiments / Datasets；中译§4.1，[原文 PDF 第 5 页](https://arxiv.org/pdf/2601.06220v1#page=5)；Ctrl+F：中译“六个分布内”／原文“Six ID benchmarks”。只指本版采用的ID/OOD划分，非这些基准普遍的固定身份。】

**基线**：RouteLLM、FORC、GraphRouter、Model-SAT（亦称 CIT-LLM-Routing）、随机选择与单模型。作者对多个对比结果使用 **Wilcoxon 符号秩检验**标注显著性，门槛 `p<0.05`；但显著性是对其定义的实验比较与样本而言，不自动保证新场景复现。

【定位·原文事实＋统计解释：中译 PDF 第 8 页；原文 Experiments / Baselines；中译§4.1，[原文 PDF 第 5 页](https://arxiv.org/pdf/2601.06220v1#page=5)；Ctrl+F：中译“CIT-”／原文“CIT-LLM-Routing”；另见：中译 PDF 第 10 页；原文 Table 1（两组各5模型及显著性表注），[原文 PDF 第 6 页](https://arxiv.org/pdf/2601.06220v1#page=6)；Ctrl+F：中译“Wilcoxon”／原文“Wilcoxon”；另见：中译 PDF 第 12 页；原文 Table 2（新模型锚点采样消融），[原文 PDF 第 7 页](https://arxiv.org/pdf/2601.06220v1#page=7)；Ctrl+F：中译“D-最优性”／原文“D-optimality”。正文Baselines称CIT-LLM-Routing，表格称Model-SAT；显著性符号须逐格读，不代表所有单元格都显著。】

**主表的一个示例**：小模型池，ID 的 Max-Acc Reward：ZeroRouter **0.45**，Model-SAT **0.43**，GraphRouter **0.38**；大模型池，OOD 的 Max-Acc Reward：ZeroRouter **0.68**，Model-SAT **0.62**。这些仍是带权重的 Reward，而不是可以与 ICL-Router 的 76.3% 准确率直接比较的数值。

【定位·原文数值：中译 PDF 第 10 页；原文 Table 1（两组各5模型及显著性表注），[原文 PDF 第 6 页](https://arxiv.org/pdf/2601.06220v1#page=6)；Ctrl+F：中译“Wilcoxon”／原文“Wilcoxon”；另见：中译 PDF 第 8–9 页；原文 Evaluation Metrics，Eq.(19)；中译§4.1，[原文 PDF 第 5 页](https://arxiv.org/pdf/2601.06220v1#page=5)；Ctrl+F：中译“总奖励”／原文“total reward”。小模型ID对应0.45/0.43/0.38，大模型OOD对应0.68/0.62；跨ICL准确率的比较是指标口径提醒。】

[原文] 作者报告在单张 A800 80GB 上完成潜在参数校准和题目预测器训练。对个人 8GB 显存机器，更现实的复现起点是保留预计算题目特征或缓存成绩、先复现 D-optimal 与路线效用；不能据此宣称已经独立重训复现了全文。

【定位·原文事实＋本文复现建议：中译 PDF 第 9 页；原文 Implementation Details；中译§4.1，[原文 PDF 第 5 页](https://arxiv.org/pdf/2601.06220v1#page=5)；Ctrl+F：中译“6,000”／原文“6,000”。A800 80GB是作者报告的训练设备；单卡8GB路线是本笔记建议，不是作者验证的最低硬件要求。】

## 原笔记纠错清单

| 原笔记中的说法 | 处理 |
|---|---|
| 核心 10 模型分两个 5 模型池 | **正确**  【定位·原文事实：中译 PDF 第 10 页；原文 Table 1（两组各5模型及显著性表注），[原文 PDF 第 6 页](https://arxiv.org/pdf/2601.06220v1#page=6)；Ctrl+F：中译“Wilcoxon”／原文“Wilcoxon”】 |
| 新增 50 个较新模型 | **原文支持**，但不是一次 60 选 1  【定位·原文事实：中译 PDF 第 8 页；原文 Experiments / LLM Candidates；中译§4.1，[原文 PDF 第 5 页](https://arxiv.org/pdf/2601.06220v1#page=5)；Ctrl+F：中译“另外50”／原文“50 distinct”】 |
| DistilBERT 微调，另有 11 个结构特征 | **正确**  【定位·原文事实：中译 PDF 第 6 页；原文 Hybrid Feature Representation，Eq.(12–14)；中译§3.2，[原文 PDF 第 4 页](https://arxiv.org/pdf/2601.06220v1#page=4)；Ctrl+F：中译“结构特征”／原文“Structural Features”】 |
| 20 维 α、b、θ，贝叶斯 IRT | **正确**  【定位·原文事实：中译 PDF 第 4 页；原文 Model Inference Accuracy Estimation，Eq.(1)；中译§3.1.1，[原文 PDF 第 2 页](https://arxiv.org/pdf/2601.06220v1#page=2)；Ctrl+F：中译“层次贝叶斯”／原文“hierarchical Bayesian”；另见：中译 PDF 第 9 页；原文 Implementation Details；中译§4.1，[原文 PDF 第 5 页](https://arxiv.org/pdf/2601.06220v1#page=5)；Ctrl+F：中译“6,000”／原文“6,000”】 |
| 200 个锚点使用 D-optimal，提前一次性选好 | **正确**  【定位·原文事实＋推导：中译 PDF 第 4 页；原文 Information-Theoretic Anchor Selection，Eq.(2–4)；中译§3.1.1，[原文 PDF 第 2–3 页](https://arxiv.org/pdf/2601.06220v1#page=2)；Ctrl+F：中译“贪心前向选择”／原文“greedy forward”；另见：中译 PDF 第 11 页；原文 Efficient Onboarding with Minimal Data；中译§4.4，[原文 PDF 第 6 页](https://arxiv.org/pdf/2601.06220v1#page=6)；Ctrl+F：中译“200 个查询”／原文“200”】 |
| 200 个锚点“自适应测试” | **错误：本文选择不依赖新模型反馈**  【定位·本文由公式判别：中译 PDF 第 4 页；原文 Information-Theoretic Anchor Selection，Eq.(2–4)；中译§3.1.1，[原文 PDF 第 2–3 页](https://arxiv.org/pdf/2601.06220v1#page=2)；Ctrl+F：中译“贪心前向选择”／原文“greedy forward”】 |
| 0.39 表示正确率 39% | **错误：它是 Max-Acc Reward**  【定位·原文指标释义：中译 PDF 第 12 页；原文 Table 2（新模型锚点采样消融），[原文 PDF 第 7 页](https://arxiv.org/pdf/2601.06220v1#page=7)；Ctrl+F：中译“D-最优性”／原文“D-optimality”；另见：中译 PDF 第 8–9 页；原文 Evaluation Metrics，Eq.(19)；中译§4.1，[原文 PDF 第 5 页](https://arxiv.org/pdf/2601.06220v1#page=5)；Ctrl+F：中译“总奖励”／原文“total reward”】 |
| 三目标 ILP 就说明实验做了总预算的全局分配 | **证据不足：可选约束与实际启用不同**  【定位·范围核验：中译 PDF 第 7 页；原文 Multi-Objective Optimization，Eq.(17–18)；中译§3.3.1，[原文 PDF 第 4 页](https://arxiv.org/pdf/2601.06220v1#page=4)；Ctrl+F：中译“可选全局约束”／原文“optional global”】 |
| 延迟估计是 `TTFT + 输出长度 × TPOT` | **正确，是近似模型，不是实测每题未来耗时**  【定位·原文事实：中译 PDF 第 6 页；原文 Model Inference Latency Estimation，Eq.(11)；中译§3.1.3，[原文 PDF 第 4 页](https://arxiv.org/pdf/2601.06220v1#page=4)；Ctrl+F：中译“首token 时间”／原文“Time to First Token”】 |

## 候选破题点

**Z-1 [原文+推导→待实验]：去掉 Fisher 概率因子后，锚点主要优化方向覆盖，不一定对新模型实际有信息。** 一个远远超过某题难度的强新模型几乎总答对，这题的 α 再大也未必有识别价值。可以比较静态 D-optimal 与先验能力/反馈引导的 sequential selection，但这会直接碰到 RADAR，需先确定新颖目标。

【定位·原文起点→本文推导→待实验：中译 PDF 第 4 页；原文 Information-Theoretic Anchor Selection，Eq.(2–4)；中译§3.1.1，[原文 PDF 第 2–3 页](https://arxiv.org/pdf/2601.06220v1#page=2)；Ctrl+F：中译“贪心前向选择”／原文“greedy forward”；另见：另文 `RADAR_中文译本_正文.pdf`：中译 PDF 第 6–7 页；原文 RADAR §3.5，Eq.(7–9)，[原文 PDF 第 6–7 页](https://arxiv.org/pdf/2509.25426v3#page=6)；Ctrl+F：中译“当前能力估计”／原文“Fisher information”。校订：“只保证方向多样性”过强，改为优化方向覆盖；实际Fisher还有p(1−p)，饱和时信息变小由Eq.(2)推得。改进有效性与新颖性均未证实。】

**Z-2 [推导→待实验]：把能力估得准，不一定将决策边界估得准。** 当新模型在旧模型附近只存在少数“能改变最优路由”的题时，可能更值得选这些边界题，而非整体最能缩小 θ 方差的题。要设计相同反馈预算的真实路由 Regret/Reward 比较，**不可以只报告 θ 的 RMSE**。

【定位·本文研究假说：中译 PDF 第 4 页；原文 Information-Theoretic Anchor Selection，Eq.(2–4)；中译§3.1.1，[原文 PDF 第 2–3 页](https://arxiv.org/pdf/2601.06220v1#page=2)；Ctrl+F：中译“贪心前向选择”／原文“greedy forward”；另见：中译 PDF 第 7 页；原文 Multi-Objective Optimization，Eq.(17–18)；中译§3.3.1，[原文 PDF 第 4 页](https://arxiv.org/pdf/2601.06220v1#page=4)；Ctrl+F：中译“可选全局约束”／原文“optional global”；另见：中译 PDF 第 8–9 页；原文 Evaluation Metrics，Eq.(19)；中译§4.1，[原文 PDF 第 5 页](https://arxiv.org/pdf/2601.06220v1#page=5)；Ctrl+F：中译“总奖励”／原文“total reward”。能力估计目标与路由效用来自这些公式；“边界题更值得选”不是原文发现，需在同池同预算实验检验。】

**Z-3 [原文范围→待实验]：新模型接入主实验采用固定 200 题预算。** 是否 16、32、64 道就够？是否需要因模型与旧池差距不同而调整停止时间？先比较多种预算下的收益曲线，再考虑最优停止机制。

【定位·原文范围→待实验：中译 PDF 第 11 页；原文 Efficient Onboarding with Minimal Data；中译§4.4，[原文 PDF 第 6 页](https://arxiv.org/pdf/2601.06220v1#page=6)；Ctrl+F：中译“200 个查询”／原文“200”。校订：200仅是报告的新模型接入实验预算；不能推出方法要求所有模型、所有用户永远用200。16/32/64及最优停止为本文拟议实验。】

**Z-4 [原文+推导→待实验]：输出长度分桶误差可能主导成本选错。** 先衡量每道题输出长度的 MAE、成本排序错误、候选翻转，再判断是否需要复杂神经回归头。不能凭“没有报告误差”直接声称误差很大。

【定位·原文起点→待实验：中译 PDF 第 5 页；原文 Model Inference Cost Estimation，Eq.(6–10)；中译§3.1.2，[原文 PDF 第 3 页](https://arxiv.org/pdf/2601.06220v1#page=3)；Ctrl+F：中译“查找表”／原文“lookup table”；另见：中译 PDF 第 12–13 页；原文 Validation of Task-Aware Difficulty，Figure 3(d)；中译§4.5，[原文 PDF 第 7 页](https://arxiv.org/pdf/2601.06220v1#page=7)；Ctrl+F：中译“平均输出token 长度”／原文“average output token length”；另见：中译 PDF 第 7 页；原文 Multi-Objective Optimization，Eq.(17–18)；中译§3.3.1，[原文 PDF 第 4 页](https://arxiv.org/pdf/2601.06220v1#page=4)；Ctrl+F：中译“可选全局约束”／原文“optional global”。论文给长度均值相关图，并未建立分桶误差主导路由错误的因果证据。MAE、候选翻转为拟议指标。】

**Z-5 [待代码]：Reward 量纲与归一化细节。** 主公式求和而表值 0.x，需要核实每样本平均、成本/时间单位及是否进一步缩放。属于复现协议问题，不是默认算法贡献。

【定位·待代码：中译 PDF 第 8–9 页；原文 Evaluation Metrics，Eq.(19)；中译§4.1，[原文 PDF 第 5 页](https://arxiv.org/pdf/2601.06220v1#page=5)；Ctrl+F：中译“总奖励”／原文“total reward”；另见：中译 PDF 第 10 页；原文 Table 1（两组各5模型及显著性表注），[原文 PDF 第 6 页](https://arxiv.org/pdf/2601.06220v1#page=6)；Ctrl+F：中译“Wilcoxon”／原文“Wilcoxon”；另见：中译 PDF 第 12 页；原文 Table 2（新模型锚点采样消融），[原文 PDF 第 7 页](https://arxiv.org/pdf/2601.06220v1#page=7)；Ctrl+F：中译“D-最优性”／原文“D-optimality”。未检查实现，不能给出实际标准化方案。】

## 老师可能问

1. **为什么 ZeroRouter 叫 zero-shot 却测 200 题？** 它并非新模型零反馈，更准确是新模型无需重训主路由器，且测试 OOD 查询泛化。 【定位·原文事实＋术语解读：中译 PDF 第 11 页；原文 Efficient Onboarding with Minimal Data；中译§4.4，[原文 PDF 第 6 页](https://arxiv.org/pdf/2601.06220v1#page=6)；Ctrl+F：中译“200 个查询”／原文“200”；另见：中译 PDF 第 2 页；原文 Introduction；中译§1，[原文 PDF 第 1 页](https://arxiv.org/pdf/2601.06220v1#page=1)；Ctrl+F：中译“模型锁定”／原文“model lock-in”】
2. **D-optimal 与 RADAR Fisher 选题差在哪？** 前者提前最大化区分度矩阵的覆盖，后者用新模型最新反馈更新 θ，实时挑最大 Fisher 信息题。 【定位·跨论文对照：中译 PDF 第 4 页；原文 Information-Theoretic Anchor Selection，Eq.(2–4)；中译§3.1.1，[原文 PDF 第 2–3 页](https://arxiv.org/pdf/2601.06220v1#page=2)；Ctrl+F：中译“贪心前向选择”／原文“greedy forward”；另见：另文 `RADAR_中文译本_正文.pdf`：中译 PDF 第 6–7 页；原文 RADAR §3.5，Eq.(7–9)，[原文 PDF 第 6–7 页](https://arxiv.org/pdf/2509.25426v3#page=6)；Ctrl+F：中译“当前能力估计”／原文“Fisher information”】
3. **ZeroRouter 的 b 为什么是向量？** 它沿多个潜在能力维度描述题目难度；IRT-Router MIRT 的 b 只是一个数。 【定位·跨论文公式对照：中译 PDF 第 4 页；原文 Model Inference Accuracy Estimation，Eq.(1)；中译§3.1.1，[原文 PDF 第 2 页](https://arxiv.org/pdf/2601.06220v1#page=2)；Ctrl+F：中译“层次贝叶斯”／原文“hierarchical Bayesian”；另见：另文 `IRT-Router_中文译文_含图表与公式.pdf`：中译 PDF 第 6–7 页；原文 IRT-Router §4.2.1 MIRT-Router，Eq.(4–5)，[原文 PDF 第 5 页](https://arxiv.org/pdf/2506.01048v2#page=5)；Ctrl+F：中译“难度bi”／原文“difficulty”】
4. **为什么预测输出 token？** 路由时还没执行最终候选，无法观测未来输出长度，只能估计未来费用和延迟。 【定位·原文事实：中译 PDF 第 5 页；原文 Model Inference Cost Estimation，Eq.(6–10)；中译§3.1.2，[原文 PDF 第 3 页](https://arxiv.org/pdf/2601.06220v1#page=3)；Ctrl+F：中译“查找表”／原文“lookup table”】
5. **论文最强的是路由结构还是选择预算？** 两者都有贡献，必须分别对比分桶/表征、锚点策略、权重和基线，不能把整体 Reward 改善全归因于 D-optimal。 【定位·本文评估建议：中译 PDF 第 12 页；原文 Table 2（新模型锚点采样消融），[原文 PDF 第 7 页](https://arxiv.org/pdf/2601.06220v1#page=7)；Ctrl+F：中译“D-最优性”／原文“D-optimality”；另见：中译 PDF 第 6–7 页；原文 Unified Prediction Model，Eq.(14–16)；中译§3.2.2，[原文 PDF 第 4 页](https://arxiv.org/pdf/2601.06220v1#page=4)；Ctrl+F：中译“残差向量”／原文“Residual Prediction”。Table 2只隔离锚点策略，不能代替全部组件归因。】

**定位原文**：IRT Eq.(1)；D-optimal Eq.(2)(3)；锚点拟合 Eq.(5)；成本 Eq.(6–10)；延迟 Eq.(11)；题目参数预测 Eq.(12–16)；ILP Eq.(17–18)；Reward Eq.(19)；主表 Table 1、锚点消融 Table 2、Figure 3。

【定位·导航补充：中译 PDF 第 3 页；原文 Figure 2；中译§3 方法，[原文 PDF 第 3 页](https://arxiv.org/pdf/2601.06220v1#page=3)；Ctrl+F：中译“框架概览”／原文“Figure 2”。精确页码见篇首导航及各公式旁的定位；Figure 3与Table 2均在中译第12页、原文第7页。】