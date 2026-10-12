# RADAR：不仅选哪个模型，还选让它思考多久

> 英文题名：**RADAR: Reasoning-Ability and Difficulty-Aware Routing for Reasoning LLMs**  
> 中文阅读入口：[中文正文PDF（15页）](pdfs/zh/RADAR_Chinese_Body.pdf#page=1)。附录A–E未译、未收录。  
> 英文对照来源：[arXiv:2509.25426v3（英文原文）](https://arxiv.org/html/2509.25426v3)，**ICLR 2026** 会议论文。  
> 核验材料：库中 `radar.pdf` v3 的 §3–§4、附录 B/D，重点为 Eq.(4–9)、表 1–3、表 18、表 20–23。  
> 标签：**[原文]** 作者给出的定义/数字；**[推导]** 从公式分析得出；**[待实验]** 合理但尚未成立的问题。与之前讲稿不同，这份核验直接用了英文原文。


## 原文定位速查（本次新增）

- 跳转说明：本页优先链接仓库内中文PDF；#page=N表示从1开始的物理页序。GitHub的PDF预览可能忽略页码片段；若未跳到指定页，请下载PDF，用阅读器打开并输入文中所列物理页码。页码已按PDF内容核对；GitHub预览的页码跳转尚未实测。范围链接跳到该范围首页，节号、公式/表格及段落关键词用于进一步定位。
- 页码口径：下文“中文PDF”专指本仓库所附《RADAR_Chinese_Body.pdf》（原文件名《RADAR_中文译本_正文.pdf》），共15页；页码均为阅读器从1开始的物理页序。英文固定为 arXiv:2509.25426v3 /《radar.pdf》，共29页；已与官网下载PDF逐字节核对一致。两者不能套用同一页码。
- [中文PDF 第15页](pdfs/zh/RADAR_Chinese_Body.pdf#page=15)明确写明“附录A—E未收录”。本笔记涉及的表14/18/20–23、4096维、训练参数、A100硬件等附录事实，必须打开英文原文；下文均标“附录未译、中文PDF未收录”。
- 35配置：[中文PDF 第4页](pdfs/zh/RADAR_Chinese_Body.pdf#page=4)§3.1定义、第8页§4配置列表；[英文第4页](https://arxiv.org/pdf/2509.25426v3#page=4)、[第8页](https://arxiv.org/pdf/2509.25426v3#page=8)。查“35”“256”。
- 冻结embedding、标量a/b/θ、2PL：[中文PDF 第5–6页](pdfs/zh/RADAR_Chinese_Body.pdf#page=5)§3.3；[英文第5–6页](https://arxiv.org/pdf/2509.25426v3#page=5)。具体Qwen3-Embedding-8B/4096维与训练参数：附录未译、中文PDF未收录，[英文第29页附录D.5](https://arxiv.org/pdf/2509.25426v3#page=29)。
- Chebyshev：[中文PDF 第5页](pdfs/zh/RADAR_Chinese_Body.pdf#page=5)Eq.(4)/[英文第5页](https://arxiv.org/pdf/2509.25426v3#page=5)；固定平均输出成本：[中文PDF 第6页](pdfs/zh/RADAR_Chinese_Body.pdf#page=6)Eq.(6)/[英文第6页](https://arxiv.org/pdf/2509.25426v3#page=6)。
- 自适应选题：[中文PDF 第6–7页](pdfs/zh/RADAR_Chinese_Body.pdf#page=6)§3.5 Eq.(7–9)/[英文第6–7页](https://arxiv.org/pdf/2509.25426v3#page=6)。5000题与Qwen3-14B在[中文PDF 第9页](pdfs/zh/RADAR_Chinese_Body.pdf#page=9)§4.2/[英文第9页](https://arxiv.org/pdf/2509.25426v3#page=9)。查“Fisher”“5k training”。
- 主表1–3全在[中文PDF 第8页](pdfs/zh/RADAR_Chinese_Body.pdf#page=8)；英文表1第7页，表2–3第8页。FRAMES OOD的9.99%不在中文主表，见[英文第19页表14](https://arxiv.org/pdf/2509.25426v3#page=19)。
- Fisher vs 随机：附录未译、中文PDF未收录；[英文第20页附录B.7](https://arxiv.org/pdf/2509.25426v3#page=20)、[第21页表20](https://arxiv.org/pdf/2509.25426v3#page=21)、[第22页表21–23](https://arxiv.org/pdf/2509.25426v3#page=22)。主表/附表数字不一致请对照同一场景，勿合并。
- 硬件/基线/评估细节：附录未译、中文PDF未收录；[英文第27页D.1–D.2](https://arxiv.org/pdf/2509.25426v3#page=27)、[第28页D.3–D.4](https://arxiv.org/pdf/2509.25426v3#page=28)、[第29页D.5–D.6](https://arxiv.org/pdf/2509.25426v3#page=29)。时延6.89±0.53ms在[中文PDF 第10页](pdfs/zh/RADAR_Chinese_Body.pdf#page=10)§4.3/英文第10页。
- “推导/教学例子/研究设想/待代码”不声称作者逐字提出；Ctrl+F英文词用于原文，中文可按节号/译名或数字查找。页码链接跳到范围第一页。
- 校订：新模型实验为§4.2；Eq.(5)是交叉熵，2PL概率式未单独编号；D.4是QA提示词，完整评估设置在D.1、IRT实现D.5、指标D.6。已纠正原笔记相关定位。

## 先说清楚它在解决什么新问题

假设一个推理模型可以选择“不思考直接答”“最多想 256 token”“最多想 4096 token”。同一个模型面对一道简单题，可能不用长篇思考也能答对；碰到难题则可能需要更长的思考预算。但预算越大，平均响应成本和时间也越高，有时过度思考还会降低效果。 〔原文＋教学情景定位：[中文PDF 第1–2页](pdfs/zh/RADAR_Chinese_Body.pdf#page=1)、[中文PDF 第4页](pdfs/zh/RADAR_Chinese_Body.pdf#page=4)，§1；§3.1；[英文PDF 第1–2、4页](https://arxiv.org/pdf/2509.25426v3#page=1)；Ctrl+F「over-think」。256/4096是讲解情景；预算不等于实际固定输出长度。〕

**RADAR 认为路由选项应该是“模型 + 推理预算”的组合**，而不是单独的模型名。它先用教育测量学的 IRT 模型预测每个组合做对某题的概率，再把概率与估计费用合在一起选择配置。想象学校组织考试：不是只决定派哪位同学，还要决定给这位同学多少答题时间。 〔原文＋教学类比定位：[中文PDF 第3–6页](pdfs/zh/RADAR_Chinese_Body.pdf#page=3)，图2；§3.1–3.4；[英文PDF 第3–6页](https://arxiv.org/pdf/2509.25426v3#page=3)；Ctrl+F「configuration」。〕

它最值得我们研究的另一点是：当新模型出现，**不是固定测一套题，而是看它前面答得怎样，决定下一题测什么**，用尽可能少的题估计其能力。但请注意：它优化的目标是**把能力估计准确**，并没有直接在选题公式里优化最终路由是否会换成更好的模型。 〔原文＋目标分析定位：[中文PDF 第6–7页](pdfs/zh/RADAR_Chinese_Body.pdf#page=6)，§3.5 Eq.(7–9)；[英文PDF 第6–7页](https://arxiv.org/pdf/2509.25426v3#page=6)；Ctrl+F「Fisher」。最大Fisher是θ估计准则；与部署价值的区别为本文推导。〕

## ① 候选空间：为什么是 35，而不是 7？

[原文，§3.1、§4、附录 D.1] 〔原文定位：[中文PDF 第4页](pdfs/zh/RADAR_Chinese_Body.pdf#page=4)、[中文PDF 第8页](pdfs/zh/RADAR_Chinese_Body.pdf#page=8)，§3.1；§4实现设置段；[英文PDF 第4、8页](https://arxiv.org/pdf/2509.25426v3#page=4)；Ctrl+F「35 configurations」。附录D.1另在英文27页，附录未译、中文PDF未收录。 附录精确定位：[附录未译：D.1 Evaluation Setup and Implementation Details，第27页](https://arxiv.org/pdf/2509.25426v3#page=27)，唯一长段“We conduct both”，其中“We route over 35 configurations”及“eight runs”分别定位配置与AIME重复运行。〕

| 底座模型 | 可用推理预算 | 形成的配置数 |
|---|---|---:|
| OpenAI `o4-mini` | low、medium、high | 3 〔原文定位：[中文PDF 第8页](pdfs/zh/RADAR_Chinese_Body.pdf#page=8)，§4表3下配置列表，o4-mini三档；[英文PDF 第8页](https://arxiv.org/pdf/2509.25426v3#page=8)；Ctrl+F「low」。〕 |
| `Qwen3-0.6B` | 0、256、512、1k、2k、4k、8k、16k | 8 〔原文定位：[中文PDF 第8页](pdfs/zh/RADAR_Chinese_Body.pdf#page=8)，§4表3下Qwen3配置列表；[英文PDF 第8页](https://arxiv.org/pdf/2509.25426v3#page=8)；Ctrl+F「256」。〕 |
| `Qwen3-1.7B` | 同上 | 8 〔原文定位：[中文PDF 第8页](pdfs/zh/RADAR_Chinese_Body.pdf#page=8)，§4表3下Qwen3配置列表；[英文PDF 第8页](https://arxiv.org/pdf/2509.25426v3#page=8)；Ctrl+F「1.7B」。〕 |
| `Qwen3-4B` | 同上 | 8 〔原文定位：[中文PDF 第8页](pdfs/zh/RADAR_Chinese_Body.pdf#page=8)，§4表3下Qwen3配置列表；[英文PDF 第8页](https://arxiv.org/pdf/2509.25426v3#page=8)；Ctrl+F「4B」。〕 |
| `Qwen3-8B` | 同上 | 8 〔原文定位：[中文PDF 第8页](pdfs/zh/RADAR_Chinese_Body.pdf#page=8)，§4表3下Qwen3配置列表；[英文PDF 第8页](https://arxiv.org/pdf/2509.25426v3#page=8)；Ctrl+F「8B」。〕 |
| **合计** | | **35** 〔原文＋计数定位：[中文PDF 第8页](pdfs/zh/RADAR_Chinese_Body.pdf#page=8)，§4实现设置段；[英文PDF 第8页](https://arxiv.org/pdf/2509.25426v3#page=8)；Ctrl+F「35」。3+4×8=35；5个底座由该列表数得。〕 |

这不是 35 个相互独立的底座模型，而是 **5 个底座模型、35 种模型配置**。Qwen 的数值是思考 token 数量的上限，超过时插入中断指令与特殊 token，要求停止思考并输出答案；不是提前保证模型一定生成恰好 256/512 个思考 token。[原文，§3.1] 〔原文＋计数定位：[中文PDF 第4页](pdfs/zh/RADAR_Chinese_Body.pdf#page=4)、[中文PDF 第8页](pdfs/zh/RADAR_Chinese_Body.pdf#page=8)，§3.1预算终止机制；§4配置列表；[英文PDF 第4、8页](https://arxiv.org/pdf/2509.25426v3#page=4)；Ctrl+F「interruption message」。〕

这项“改变决策对象”的贡献很实在：在相同底座下，RADAR 可以在“低预算/高预算”间切换。但也随之带来大量配置离散化选择问题：预算取点过稀可能错过合适范围，过密则采集成绩和建立配置档案成本上升。[推导] 〔推导／待实验定位：[中文PDF 第4页](pdfs/zh/RADAR_Chinese_Body.pdf#page=4)，§3.1离散配置集合；[英文PDF 第4页](https://arxiv.org/pdf/2509.25426v3#page=4)；Ctrl+F「discretization」。取点稀密影响是本文研究假设，非作者实验结论。〕

## ② 题目表示：怎样知道什么题比较难？

[原文，§3.3、附录 D] 使用冻结的 `Qwen3-Embedding-8B` 把问题转换为 **4096 维向量** `e_q`，然后两个可学习的线性映射分别得到： 〔原文定位：[中文PDF 第5–6页](pdfs/zh/RADAR_Chinese_Body.pdf#page=5)，§3.3线性映射；模型名/4096维见附录D.5；[英文PDF 第5、29页](https://arxiv.org/pdf/2509.25426v3#page=5)；Ctrl+F「4096」。D.5附录未译、中文PDF未收录；冻结在中文5–6页可核对。 附录精确定位：[附录未译：D.5 IRT Implementation Details in RADAR，第29页](https://arxiv.org/pdf/2509.25426v3#page=29)，唯一段落“We employ”，嵌入句含“frozen-weight”“4096”，训练句含“100 epochs”。〕

$$
 a_q=\mathbf w_a^{\top}\mathbf e_q,\qquad
 b_q=\mathbf w_b^{\top}\mathbf e_q.
$$

**a 是区分度，b 是难度，二者都是标量。** 〔原文定位：[中文PDF 第6页](pdfs/zh/RADAR_Chinese_Body.pdf#page=6)，§3.3两条未编号线性映射；[英文PDF 第5页](https://arxiv.org/pdf/2509.25426v3#page=5)；Ctrl+F「scalar-valued」。〕

通俗解释：`b_q` 越高，表示按这种统计建模这题所需能力越高；`a_q` 表示这道题能在多大程度上区分不同能力的配置。题目文本不是直接输入一个巨大的 LLM Router，而是经过 embedding 后压缩为两个数，再进入概率模型。 〔原文＋讲解定位：[中文PDF 第5–6页](pdfs/zh/RADAR_Chinese_Body.pdf#page=5)，§3.3 difficulty/discrimination及2PL公式；[英文PDF 第5页](https://arxiv.org/pdf/2509.25426v3#page=5)；Ctrl+F「difficulty」。〕

要小心：“可解释”是结构赋予变量的含义，不意味着预测出的 b 完全等于人类判断的难度。它仍依赖训练数据、文本编码器和 2PL 假设。 〔解释边界定位：[中文PDF 第9–10页](pdfs/zh/RADAR_Chinese_Body.pdf#page=9)，§4.3、图3；难度Pearson相关0.509；[英文PDF 第9页](https://arxiv.org/pdf/2509.25426v3#page=9)；Ctrl+F「0.509」。作者报告中等相关，不能说与人类难度完全等同。〕

## ③ 模型表示：每个配置只用一个能力 θ

[原文，§3.3] 每个配置 `g` 有一个实数能力参数 `θ_g`，从“训练题 × 配置”的做题成绩中与题目参数联合学习，不需要模型名字或说明文字，也不靠参数量直接推断能力。 〔原文＋公式解释定位：[中文PDF 第6页](pdfs/zh/RADAR_Chinese_Body.pdf#page=6)，§3.3标量θ与训练矩阵、Eq.(5)；[英文PDF 第5–6页](https://arxiv.org/pdf/2509.25426v3#page=5)；Ctrl+F「binary-valued」。〕

这里和 IRT-Router、ZeroRouter 很不同：

- **IRT-Router**：简介 → 25 维能力向量。 〔跨文原文定位：[IRT-Router中文PDF 第6–7页](pdfs/zh/IRT-Router_Chinese.pdf#page=6)§4.1、Eq.(4)，[IRT-Router中文PDF 第10页](pdfs/zh/IRT-Router_Chinese.pdf#page=10)§5.5的25维；[英文v2第5页](https://arxiv.org/pdf/2506.01048v2#page=5)、[第7页](https://arxiv.org/pdf/2506.01048v2#page=7)；Ctrl+F「profile」「Dimension」。〕
- **ZeroRouter**：测试数据 → 20 维能力向量。 〔跨文原文定位：[ZeroRouter中文PDF 第4页](pdfs/zh/ZeroRouter_Chinese.pdf#page=4)§3.1.1 Eq.(1)、[ZeroRouter中文PDF 第9页](pdfs/zh/ZeroRouter_Chinese.pdf#page=9)Implementation Details；[英文v1第2页](https://arxiv.org/pdf/2601.06220v1#page=2)、[第5页](https://arxiv.org/pdf/2601.06220v1#page=5)；Ctrl+F「D = 20」。这里的“测试数据”指用于能力校准的历史测评响应，非泄露下游测试标签。〕
- **RADAR**：测试数据 → **一个标量能力**。 〔原文定位：[中文PDF 第6页](pdfs/zh/RADAR_Chinese_Body.pdf#page=6)，§3.3标量θ、Eq.(5)；[英文PDF 第5–6页](https://arxiv.org/pdf/2509.25426v3#page=5)；Ctrl+F「scalar-valued」。〕

**为什么一个数也有价值？** 如果题目主要沿着一条“总体难度轴”变化，一个数会很省参数、容易排序；但如果能力之间存在严重互补，例如 A 擅长代码、B 擅长数学，同一条轴可能无法很好表达“谁在哪些题上突出”。这属于**模型容量/假设上的可能局限 [推导]**，不能在没做对照前说 scalar IRT 必然不如多维。 〔作者动机＋研究假设定位：[中文PDF 第6页](pdfs/zh/RADAR_Chinese_Body.pdf#page=6)，§3.3与MIRT的参数比较；[英文PDF 第5–6页](https://arxiv.org/pdf/2509.25426v3#page=5)；Ctrl+F「fewer」。标量的省参数/排序是作者解释；代码/数学互补失败例是本文假设。〕

**另一个问题 [推导]**：同一 Qwen 底座的不同预算作为独立配置学 θ。方法并没有把预算之间的依赖关系写成层次模型；因此即使所有配置共享底座，较少预算的数据也不一定被充分用于较大预算的能力估计。注意作者也讨论过“预算增大有时会过度思考”，不能机械要求能力随预算严格单调。 〔模型结构推导定位：[中文PDF 第4页](pdfs/zh/RADAR_Chinese_Body.pdf#page=4)、[中文PDF 第6页](pdfs/zh/RADAR_Chinese_Body.pdf#page=6)，§3.1配置定义；§3.3每配置θ；[英文PDF 第4–6页](https://arxiv.org/pdf/2509.25426v3#page=4)；Ctrl+F「configuration」。未在显示模型中建层次共享；overthinking背景见中文1–2/英文1–2，实际实现待代码。〕

## ④ 答对概率：2PL 的核心公式

[原文，§3.3 的未编号概率式；Eq.(5)为训练损失] 〔原文定位：[中文PDF 第6页](pdfs/zh/RADAR_Chinese_Body.pdf#page=6)，§3.3未编号的2PL概率式；Eq.(5)后再次定义pij；[英文PDF 第5–6页](https://arxiv.org/pdf/2509.25426v3#page=5)；Ctrl+F「sigmoid」。校订：式(5)编号属于交叉熵损失，概率式本身未单独编号。〕

$$
\widehat P(y_{qg}=1)=\sigma\!\left(a_q(\theta_g-b_q)\right),
\qquad \sigma(z)=\frac{1}{1+e^{-z}}.
$$

逐字解释：如果配置能力 θ 比题目难度 b 大，且区分度 a 为正，括号里的值通常更大，预测答对的概率也更高。 〔公式推导定位：[中文PDF 第6页](pdfs/zh/RADAR_Chinese_Body.pdf#page=6)，§3.3 2PL概率式；[英文PDF 第5页](https://arxiv.org/pdf/2509.25426v3#page=5)；Ctrl+F「sigmoid」。单调判断明确要求a>0。〕

**教学示例（非论文实测）**：一道题 `a=2`、`b=0.5`；某配置 `θ=1.0`，得到 `σ(2×0.5)=σ(1)≈0.731`；另一个配置 `θ=0`，得到 `σ(-1)≈0.269`。一个问题和一个配置之间，只需要这 3 个数就能算出预测概率。 〔自拟数值例定位：[中文PDF 第6页](pdfs/zh/RADAR_Chinese_Body.pdf#page=6)，§3.3 2PL概率式；[英文PDF 第5页](https://arxiv.org/pdf/2509.25426v3#page=5)；Ctrl+F「sigmoid」。0.731/0.269由公式计算，非实验表数。〕

[原文，Eq.(5)、附录 D] 从全部已知的配置—题目结果格子优化平均交叉熵，训练 100 epochs，学习率 `5×10⁻⁴`、batch size 32。正文没有在显示的线性映射式中给 `a` 一个明确的非负约束，因此“θ 越高一定越容易答对”的逐题数学单调性，**从印出的式子本身尚不能保证**；是否有未写出的实现约束，需要检查官方代码。[推导 + 待代码] 〔原文＋推导／待代码定位：[中文PDF 第5–6页](pdfs/zh/RADAR_Chinese_Body.pdf#page=5)，§3.3、Eq.(5)；训练参数见附录D.5；[英文PDF 第5–6、29页](https://arxiv.org/pdf/2509.25426v3#page=5)；Ctrl+F「100 epochs」。D.5附录未译、中文PDF未收录；印刷映射未强制a非负，不等于已证明代码无约束。 附录精确定位：[附录未译：D.5 IRT Implementation Details in RADAR，第29页](https://arxiv.org/pdf/2509.25426v3#page=29)，唯一段落“We employ”，嵌入句含“frozen-weight”“4096”，训练句含“100 epochs”。〕

## ⑤ 成本：注意它不是按每道题预测输出长度

[原文，§3.4、Eq.(6)] 〔原文定位：[中文PDF 第6页](pdfs/zh/RADAR_Chinese_Body.pdf#page=6)，§3.4 Eq.(6)；[英文PDF 第6页](https://arxiv.org/pdf/2509.25426v3#page=6)；Ctrl+F「COST PREDICTION」。〕

$$
\widehat C(g)=\frac{1}{|\mathcal Q_{train}|}\sum_{q\in\mathcal Q_{train}}
(n_{gq}^{reason}+n_{gq}^{answer})\,t_g.
$$

这里 `t_g` 是对应底座模型的输出 token 单价，思考 token 和作答 token 都纳入。把训练集里各题的实际输出长度求平均，就得到**该配置的一个平均输出成本常数**，然后对配置之间成本使用 min-max 归一化到 `[0,1]`。 〔原文定位：[中文PDF 第6页](pdfs/zh/RADAR_Chinese_Body.pdf#page=6)，§3.4 Eq.(6)及其后min-max说明；[英文PDF 第6页](https://arxiv.org/pdf/2509.25426v3#page=6)；Ctrl+F「min-max」。〕

**它不随当前题目变化，并且这个启发式公式没有计入输入 token 费用**。比如一条普通数学题与一条包含十几篇文章的长上下文题，若分配到同一配置，在路由打分时使用的是同一个配置成本估计。长上下文上的成本误差可能更值得检查；但要有实际逐题 token 数据才能确定影响程度。[原文 + 推导] 〔公式推导／待实验定位：[中文PDF 第6页](pdfs/zh/RADAR_Chinese_Body.pdf#page=6)，§3.4 Eq.(6)；[英文PDF 第6页](https://arxiv.org/pdf/2509.25426v3#page=6)；Ctrl+F「averaged」。训练集平均与未计输入可从公式核对；长提示影响是研究假设。〕

这也解释为什么不能把本文成本预测写成 ZeroRouter 式的“每题输入+输出长度估计”。二者明显不同。 〔跨文比较：RADAR[中文PDF 第6页](pdfs/zh/RADAR_Chinese_Body.pdf#page=6)Eq.(6)/[英文v3第6页](https://arxiv.org/pdf/2509.25426v3#page=6)；[ZeroRouter中文PDF 第5页](pdfs/zh/ZeroRouter_Chinese.pdf#page=5)§3.1.2 Eq.(6–10)/[英文v1第3页](https://arxiv.org/pdf/2601.06220v1#page=3)，Ctrl+F「Model Inference Cost Estimation」。后者明确输入/输出token与输出长度分桶。〕

## ⑥ 路由目标：RADAR 的 Chebyshev 到底改变了什么？

[原文，§3.2] 〔原文定位：[中文PDF 第5页](pdfs/zh/RADAR_Chinese_Body.pdf#page=5)，§3.2.1–3.2.2，Eq.(2–4)；[英文PDF 第5页](https://arxiv.org/pdf/2509.25426v3#page=5)；Ctrl+F「Chebyshev」。〕

普通线性加权选择： 〔原文定位：[中文PDF 第5页](pdfs/zh/RADAR_Chinese_Body.pdf#page=5)，Eq.(3)；[英文PDF 第5页](https://arxiv.org/pdf/2509.25426v3#page=5)；Ctrl+F「LSP」。〕

$$
 g^*(q)=\arg\max_g\left[w\widehat P(q,g)-(1-w)\widehat C(g)\right].
$$

RADAR 重点使用的加权 Chebyshev 标量化： 〔原文定位：[中文PDF 第5页](pdfs/zh/RADAR_Chinese_Body.pdf#page=5)，Eq.(4)；[英文PDF 第5页](https://arxiv.org/pdf/2509.25426v3#page=5)；Ctrl+F「CSP」。〕

$$
 g^*(q)=\arg\min_g\max\left\{w\left|1-\widehat P(q,g)\right|,
(1-w)\widehat C(g)\right\},\qquad 0\leq w\leq1.
$$

**解释**：理想点是“正确率 1，成本 0”。线性式把质量和价格直接相加；Chebyshev 式看加权后最差的那一项，即“离理想目标差得最严重的维度”。改变 w 会走向不同配置。 〔原文＋讲解定位：[中文PDF 第5页](pdfs/zh/RADAR_Chinese_Body.pdf#page=5)，§3.2.2 Eq.(4)前后说明；[英文PDF 第5页](https://arxiv.org/pdf/2509.25426v3#page=5)；Ctrl+F「ideal reference point」。〕

原文强调线性加权可能找不到非凸 Pareto 前沿的某些离散点，而 Chebyshev 可以找到。要注意两层条件： 〔原文定位：[中文PDF 第5页](pdfs/zh/RADAR_Chinese_Body.pdf#page=5)，§3.2.2 Eq.(4)前说明；[英文PDF 第5页](https://arxiv.org/pdf/2509.25426v3#page=5)；Ctrl+F「non-convex」。〕

1. **作者论文的离散配置集合下**，这个论点有数学依据；并在 OOD 的附录表 18 进行了对比。 〔原文定位：[中文PDF 第5页](pdfs/zh/RADAR_Chinese_Body.pdf#page=5)、[中文PDF 第9页](pdfs/zh/RADAR_Chinese_Body.pdf#page=9)，§3.2、§4.1消融段；附录B.5表18；[英文PDF 第5、20–21页](https://arxiv.org/pdf/2509.25426v3#page=5)；Ctrl+F「Table 18」。表18附录未译、中文PDF未收录；其结论有任务/指标例外，例如LSAT Hypervolume线性略高。 附录精确定位：[附录未译：B.5 Ablation Study on Scalarization，第20页](https://arxiv.org/pdf/2509.25426v3#page=20)，唯一段落“See Table 18”；数值见[表18，第21页](https://arxiv.org/pdf/2509.25426v3#page=21)，LS/CS列，含LSAT例外。〕
2. **若允许按概率随机混合配置**，期望成本—质量点会形成凸组合，那些不在上凸包的点可能不再是批量平均意义下的最优点。此时“能找到凹部点”不自动等于部署收益更高；需要明确用户到底要求单请求确定性约束还是长期平均预算。这是研究设计层面的**推导**，不是原文实验已证实的不足。 〔本文数学推导定位：[中文PDF 第5页](pdfs/zh/RADAR_Chinese_Body.pdf#page=5)，Eq.(3–4)的离散g选择；[英文PDF 第5页](https://arxiv.org/pdf/2509.25426v3#page=5)；Ctrl+F「Chebyshev」。随机混合后的凸包与部署条件分析并非作者在此给出的实验结论。〕

## ⑦ 新模型接入：最值得精读的“自适应考试”

[原文，§3.5，Eq.(7–9)] 新来一个配置，保留已经学到的题目 `a_q,b_q`，只估它自己的 θ。 〔原文定位：[中文PDF 第6–7页](pdfs/zh/RADAR_Chinese_Body.pdf#page=6)，§3.5 Eq.(7–9)；[英文PDF 第6–7页](https://arxiv.org/pdf/2509.25426v3#page=6)；Ctrl+F「learned query parameters」。〕

**一轮过程**：

1. 先设置新配置当前能力猜测 `θ̂₀=0`；已测题集为空。 〔原文定位：[中文PDF 第7页](pdfs/zh/RADAR_Chinese_Body.pdf#page=7)，§3.5 Eq.(8)前初始化；[英文PDF 第6页](https://arxiv.org/pdf/2509.25426v3#page=6)；Ctrl+F「initialize」。〕
2. 对所有尚未测的题，计算当前 θ 下的 Fisher 信息： 〔原文定位：[中文PDF 第7页](pdfs/zh/RADAR_Chinese_Body.pdf#page=7)，§3.5 Eq.(9)；[英文PDF 第6页](https://arxiv.org/pdf/2509.25426v3#page=6)；Ctrl+F「Fisher information」。〕

$$
 I(q;\theta)=a_q^2\,p_q(\theta)[1-p_q(\theta)],
 \quad p_q(\theta)=\sigma(a_q(\theta-b_q)).
$$

3. 选择信息量最大的那道题，让新配置真实作答并获得对/错标签。 〔原文定位：[中文PDF 第7页](pdfs/zh/RADAR_Chinese_Body.pdf#page=7)，§3.5 Eq.(8)及之后观察响应步骤；[英文PDF 第6–7页](https://arxiv.org/pdf/2509.25426v3#page=6)；Ctrl+F「highest Fisher」。〕
4. 用迄今已测的全部结果重新做最大似然估计，得到更新后的 `θ̂_t`。 〔原文定位：[中文PDF 第7页](pdfs/zh/RADAR_Chinese_Body.pdf#page=7)，§3.5 Eq.(7)、能力重新估计式；[英文PDF 第6–7页](https://arxiv.org/pdf/2509.25426v3#page=6)；Ctrl+F「update」。〕
5. 继续下一轮，直到预算用完。 〔原文定位：[中文PDF 第7页](pdfs/zh/RADAR_Chinese_Body.pdf#page=7)，§3.5停止条件T；[英文PDF 第7页](https://arxiv.org/pdf/2509.25426v3#page=7)；Ctrl+F「budget」。〕

**为什么 Fisher 最大不一定是“最难的题”？** 信息来自 `a² × p(1−p)`。假设区分度相同，一道估计有 50% 正确概率的题比几乎肯定做对/做错的题更能区别“能力究竟在哪个位置”。而区分度越大，通常更有信息，所以还要乘 `a²`。 〔数学推导定位：[中文PDF 第7页](pdfs/zh/RADAR_Chinese_Body.pdf#page=7)，Eq.(9)；[英文PDF 第6页](https://arxiv.org/pdf/2509.25426v3#page=6)；Ctrl+F「Fisher」。固定a时p(1−p)在p=0.5最大；关于a的比较仍须同时考虑p变化。〕

[原文，§4.2、表 3、附录表 20–23] 加入 `Qwen3-14B` 的 8 档预算配置。主要新配置校准使用 **5000 道题，约训练集的 12%**；附录还比较 100、500、5000 道下 Fisher 选题与随机选题。**这 5000 道是校准一个新配置能力的采样规模，不能写成只花 5000 道就完成 8 个配置的全部评测。** 〔原文＋预算口径解释定位：[中文PDF 第8–9页](pdfs/zh/RADAR_Chinese_Body.pdf#page=8)，§4.2、表3；附录B.7表20–23；[英文PDF 第8–9、20–22页](https://arxiv.org/pdf/2509.25426v3#page=8)；Ctrl+F「5k training」。表20–23附录未译、中文PDF未收录。5000按§3.5的单配置算法理解；作者未将8配置调用合计写成5000。 附录精确定位：[附录未译：B.7 Ablation Study on Adaptive Testing，第20–21页](https://arxiv.org/pdf/2509.25426v3#page=20)，首段“We conduct an ablation study”与第二段“For example”；[表20，第21页](https://arxiv.org/pdf/2509.25426v3#page=21)为ID Hypervolume，[表21–23，第22页](https://arxiv.org/pdf/2509.25426v3#page=22)依次为ID CPT、OOD Hypervolume、OOD CPT，核对Rnd/Fshr各预算列。〕

### 它真正优化什么？

- **显式的选题目标**：尽快减少 θ 的估计不确定性，即“测准这个配置的能力”。 〔目标解释定位：[中文PDF 第6–7页](pdfs/zh/RADAR_Chinese_Body.pdf#page=6)，§3.5 Eq.(7–9)；[英文PDF 第6–7页](https://arxiv.org/pdf/2509.25426v3#page=6)；Ctrl+F「Fisher」。信息准则针对标量θ，不显式包含路由后悔值。〕
- **最终报告的指标**：接入后的成本—质量权衡指标（主表为 Hypervolume，附录还给 CPT）。 〔原文定位：[中文PDF 第8–9页](pdfs/zh/RADAR_Chinese_Body.pdf#page=8)，§4.2、表3的Hypervolume；附录B.7表20–23；[英文PDF 第8–9、21–22页](https://arxiv.org/pdf/2509.25426v3#page=8)；Ctrl+F「RADAR++」。直接新增模型主表指标为Hypervolume，附录还给CPT；附录未译、中文PDF未收录。 附录精确定位：[附录未译：B.7 Ablation Study on Adaptive Testing，第20–21页](https://arxiv.org/pdf/2509.25426v3#page=20)，首段“We conduct an ablation study”与第二段“For example”；[表20，第21页](https://arxiv.org/pdf/2509.25426v3#page=21)为ID Hypervolume，[表21–23，第22页](https://arxiv.org/pdf/2509.25426v3#page=22)依次为ID CPT、OOD Hypervolume、OOD CPT，核对Rnd/Fshr各预算列。〕
- **可能出现的错位 [推导]**：一个配置的 θ 被测得很准，但如果它在大多数题目上都比旧池已有配置贵且弱，那么路由收益仍可能几乎为零；反之某个新配置只在狭窄的低成本决策边界上有用，可能不必准确估计其全局能力。 〔研究假设定位：[中文PDF 第5–7页](pdfs/zh/RADAR_Chinese_Body.pdf#page=5)，Eq.(4)、(7–9)；[英文PDF 第5–7页](https://arxiv.org/pdf/2509.25426v3#page=5)；Ctrl+F「Fisher」。贵且弱、局部决策边界情景为本文构造，作者未用其证明错位。〕

这是“**能力估计最优** 不一定等于 **部署决策最优**”的候选研究问题。但要证明它是可投稿的 gap，必须在强基线上复现、检查新文献，并用相同预算的效用实验验证。 〔候选研究问题定位：[中文PDF 第6–9页](pdfs/zh/RADAR_Chinese_Body.pdf#page=6)，§3.5与§4.2、表3的目标/评测对照；[英文PDF 第6–9页](https://arxiv.org/pdf/2509.25426v3#page=6)；Ctrl+F「RADAR++」。“可投稿gap”与新颖性均未获得论文或本次定位工作证明。〕

## ⑧ 评测：至少背下来这四个数字和条件

[原文，§4、表 1–3、附录] 〔原文定位：[中文PDF 第7–9页](pdfs/zh/RADAR_Chinese_Body.pdf#page=7)，§4、表1–3；[英文PDF 第7–9页](https://arxiv.org/pdf/2509.25426v3#page=7)；Ctrl+F「Benchmarks」。〕

**数据**：8 个推理基准，包括 GPQA-Diamond、MMLU、MMLU-Redux、MMLU-Pro、LSAT、AIME、MATH-500、FRAMES；一共 **50,139 个不同问题、约 175 万条二值回答记录**。ID 与 OOD 按基准划分，OOD 还考虑重叠数据的排除条件。常规问题/配置的输出评测一般只执行一次；**AIME 因测试题较少进行了多次运行**。所以旧稿“只跑一次”作为对整篇实验的判断不成立。 〔原文定位：[中文PDF 第7–8页](pdfs/zh/RADAR_Chinese_Body.pdf#page=7)，§4 Benchmarks与实现设置，表3下段；[英文PDF 第7–8页](https://arxiv.org/pdf/2509.25426v3#page=7)；Ctrl+F「50,139」。AIME为8次运行均值；附录D.1英文27页重复说明，附录未译、中文PDF未收录。 附录精确定位：[附录未译：D.1 Evaluation Setup and Implementation Details，第27页](https://arxiv.org/pdf/2509.25426v3#page=27)，唯一长段“We conduct both”，其中“We route over 35 configurations”及“eight runs”分别定位配置与AIME重复运行。〕

**评价指标**：

- **Hypervolume**：概括性能—成本前沿的覆盖面积，越大越好。它不是单个成本点的正确率；不同论文的归一化若不同，也不能直接跨论文比绝对数值。 〔原文＋比较提醒定位：[中文PDF 第7页](pdfs/zh/RADAR_Chinese_Body.pdf#page=7)，§4 Metrics；附录D.6；[英文PDF 第7、29页](https://arxiv.org/pdf/2509.25426v3#page=7)；Ctrl+F「hypervolume」。D.6附录未译、中文PDF未收录；跨论文尺度警告为本文分析。 附录精确定位：[附录未译：D.6 Metrics，第29页](https://arxiv.org/pdf/2509.25426v3#page=29)，唯一长段“Our formulation”，Hypervolume释义查“area under”，CPT定义查“Given a performance threshold”。〕
- **CPT(90%)**：达到参考配置 `o4-mini/high` 平均性能的 90%，最少需要其多少比例的费用，越小越好。如果原模型准确率是 80%，目标是 72%，**不是 90% 的绝对准确率**。 〔原文＋数值例定位：[中文PDF 第7页](pdfs/zh/RADAR_Chinese_Body.pdf#page=7)，§4 Metrics；附录D.6；[英文PDF 第7、29页](https://arxiv.org/pdf/2509.25426v3#page=7)；Ctrl+F「CPT」。80%×90%=72%是自拟算例；D.6附录未译、中文PDF未收录。 附录精确定位：[附录未译：D.6 Metrics，第29页](https://arxiv.org/pdf/2509.25426v3#page=29)，唯一长段“Our formulation”，Hypervolume释义查“area under”，CPT定义查“Given a performance threshold”。〕
- **Oracle**：使用测试时每题真实成绩事后选最佳配置，是理想上界，不是可部署方法。 〔原文定位：[中文PDF 第7页](pdfs/zh/RADAR_Chinese_Body.pdf#page=7)，§4 Baselines；附录D.3；[英文PDF 第7、28页](https://arxiv.org/pdf/2509.25426v3#page=7)；Ctrl+F「Oracle」。D.3附录未译、中文PDF未收录；选最佳表现中最便宜配置的事后上界。 附录精确定位：[附录未译：D.3 Baselines，第28页](https://arxiv.org/pdf/2509.25426v3#page=28)，唯一长段“For RouterBench”；IRT适配定位“We adapt a concurrent”，Oracle定位“The Oracle router”。〕

**主表部分数据**：

| 场景与量 | 原文数字 | 正确解释 |
|---|---:|---|
| MATH-500，ID CPT(90%) | **1.31%** | 达到 o4-mini/high 质量的 90%，所需成本相对于它的比例 〔原文定位：[中文PDF 第8页](pdfs/zh/RADAR_Chinese_Body.pdf#page=8)，表2 MATH-500行、RADAR列；[英文PDF 第8页](https://arxiv.org/pdf/2509.25426v3#page=8)；Ctrl+F「1.31%」。〕 |
| FRAMES，OOD CPT(90%) | **9.99%** | 长上下文 OOD 场景下约十分之一的参考成本 〔原文定位：附录未译、中文PDF未收录，附录B.3 表14 FRAMES行、RADAR列；[英文PDF 第19页](https://arxiv.org/pdf/2509.25426v3#page=19)；Ctrl+F「9.99%」。也见附录B.5表18英文21页；不能在中文主表2找此OOD值。 附录精确定位：[附录未译：B.3 Results on OOD Queries，表14，第19页](https://arxiv.org/pdf/2509.25426v3#page=19)，B.3首段“Table 13 and Table 14 report”；表14表题“Routing performance on OOD queries”、CPT(90%)，FRAMES行/RADAR列“9.99%”；[附录未译：B.5 Ablation Study on Scalarization，第20页](https://arxiv.org/pdf/2509.25426v3#page=20)，唯一段落“See Table 18”；数值见[表18，第21页](https://arxiv.org/pdf/2509.25426v3#page=21)，LS/CS列，含LSAT例外。〕 |
| GPQA-Diamond，ID Hypervolume | RADAR **0.7513**，IRT 基线 **0.6942** | RADAR 在该指标、该设置下较好 〔原文定位：[中文PDF 第8页](pdfs/zh/RADAR_Chinese_Body.pdf#page=8)，表1 GPQA-Diamond行、IRT/RADAR列；[英文PDF 第7页](https://arxiv.org/pdf/2509.25426v3#page=7)；Ctrl+F「0.7513」。IRT基线经适配，非原始MIRT直接照搬；适配细节见英文28页附录D.3，附录未译、中文PDF未收录。 附录精确定位：[附录未译：D.3 Baselines，第28页](https://arxiv.org/pdf/2509.25426v3#page=28)，唯一长段“For RouterBench”；IRT适配定位“We adapt a concurrent”，Oracle定位“The Oracle router”。〕 |
| MATH-500，ID Hypervolume | **RouterBench 0.9528**，RADAR **0.9449** | **不是 RADAR 每个基准都第一** 〔原文定位：[中文PDF 第8页](pdfs/zh/RADAR_Chinese_Body.pdf#page=8)，表1 MATH-500行；[英文PDF 第7页](https://arxiv.org/pdf/2509.25426v3#page=7)；Ctrl+F「0.9528」。〕 |
| 新增 Qwen3-14B，GPQA OOD Hypervolume | **0.7466 → 0.7463** | **略微下降**，不是接入后所有集合必增 〔原文定位：[中文PDF 第8页](pdfs/zh/RADAR_Chinese_Body.pdf#page=8)，表3 GPQA-Diamond行、右侧OOD两列；[英文PDF 第8页](https://arxiv.org/pdf/2509.25426v3#page=8)；Ctrl+F「0.7463」。〕 |
| 新增 Qwen3-14B，FRAMES OOD Hypervolume | **0.8865 → 0.8931** | 有一定改善，但不能概括成普遍大幅提升 〔原文定位：[中文PDF 第8页](pdfs/zh/RADAR_Chinese_Body.pdf#page=8)，表3 FRAMES行、右侧OOD两列；[英文PDF 第8页](https://arxiv.org/pdf/2509.25426v3#page=8)；Ctrl+F「0.8931」。〕 |

**附录表格核验提醒**：主表 1 的 ID MMLU RADAR 值是 **0.8720**、FRAMES 是 **0.8762**；附录表 20 标称“ID”的基线列却出现 MMLU **0.8609** 和 FRAMES **0.8865**，与主表不一致，后者还与表 3 的 OOD 基线数字一致。**原文内部存在表格口径不一致的疑点，引用主要结论时优先标清采用主表，不能私自将不同表的数混合。** 〔逐表核验定位：[中文PDF 第8页](pdfs/zh/RADAR_Chinese_Body.pdf#page=8)，主表1及表3；对照附录B.7表20；[英文PDF 第7–8、21页](https://arxiv.org/pdf/2509.25426v3#page=7)；Ctrl+F「0.8609」。表20附录未译、中文PDF未收录；可确认数字不一致，无法仅凭此确认成因。 附录精确定位：[附录未译：B.7 Ablation Study on Adaptive Testing，第20–21页](https://arxiv.org/pdf/2509.25426v3#page=20)，首段“We conduct an ablation study”与第二段“For example”；[表20，第21页](https://arxiv.org/pdf/2509.25426v3#page=21)为ID Hypervolume，[表21–23，第22页](https://arxiv.org/pdf/2509.25426v3#page=22)依次为ID CPT、OOD Hypervolume、OOD CPT，核对Rnd/Fshr各预算列。〕

**实验成本与设备**：[原文，附录 D.2] 开源 Qwen 模型采用 `vLLM` 部署，每个模型用一张 **NVIDIA A100 80GB**；路由时延实验报告约 **6.89 ± 0.53 ms** 的附加时间。这意味着单卡 8GB 显存机器不适合按原方式从零生成全量 175 万条结果；**但不能因此说 RADAR 理论、选题公式或路由后处理不可复现**。可将“结果生成”与“利用公开缓存做方法复现”拆开评估；缓存是否完整开放还需单独核对。 〔原文＋复现判断定位：[中文PDF 第10页](pdfs/zh/RADAR_Chinese_Body.pdf#page=10)，§4.3时延；硬件在附录D.2；[英文PDF 第10、27页](https://arxiv.org/pdf/2509.25426v3#page=10)；Ctrl+F「6.89」。D.2附录未译、中文PDF未收录。时延为MATH-500的500题×3次；8GB可行性/公开缓存完备性未实测。 附录精确定位：[附录未译：D.2 Hardware，第27页](https://arxiv.org/pdf/2509.25426v3#page=27)，唯一段落“For all open-source models”，含“vLLM”“NVIDIA A100 80GB”“single such GPU”。〕

## 对旧笔记的逐项更正

| 旧笔记内容 | 处理 |
|---|---|
| 35 个“模型 × 预算”配置 | **正确，但明确为 5 个底座模型的 35 个配置** 〔原文＋计数定位：[中文PDF 第8页](pdfs/zh/RADAR_Chinese_Body.pdf#page=8)，§4配置列表；[英文PDF 第8页](https://arxiv.org/pdf/2509.25426v3#page=8)；Ctrl+F「35」。〕 |
| 每道题由 4096 维 embedding → a、b 两个标量 | **正确** 〔原文定位：[中文PDF 第5–6页](pdfs/zh/RADAR_Chinese_Body.pdf#page=5)，§3.3；4096维在附录D.5；[英文PDF 第5、29页](https://arxiv.org/pdf/2509.25426v3#page=5)；Ctrl+F「4096」。D.5附录未译、中文PDF未收录。 附录精确定位：[附录未译：D.5 IRT Implementation Details in RADAR，第29页](https://arxiv.org/pdf/2509.25426v3#page=29)，唯一段落“We employ”，嵌入句含“frozen-weight”“4096”，训练句含“100 epochs”。〕 |
| 每配置一个能力 θ | **正确** 〔原文定位：[中文PDF 第6页](pdfs/zh/RADAR_Chinese_Body.pdf#page=6)，§3.3的θ定义；[英文PDF 第5页](https://arxiv.org/pdf/2509.25426v3#page=5)；Ctrl+F「scalar-valued」。〕 |
| 输出平均 token 成本，不计输入 | **正确，但应强调只是训练集平均固定成本** 〔公式核验定位：[中文PDF 第6页](pdfs/zh/RADAR_Chinese_Body.pdf#page=6)，§3.4 Eq.(6)；[英文PDF 第6页](https://arxiv.org/pdf/2509.25426v3#page=6)；Ctrl+F「averaged」。〕 |
| Adaptive testing 的目标是估 θ | **正确，是与新选题最有关的点** 〔目标解释定位：[中文PDF 第6–7页](pdfs/zh/RADAR_Chinese_Body.pdf#page=6)，§3.5 Eq.(7–9)；[英文PDF 第6–7页](https://arxiv.org/pdf/2509.25426v3#page=6)；Ctrl+F「ability」。〕 |
| Fisher 选题达到 5000 时一定显著优于随机 | **过强；附录各数据集有打平/随机略优例子** 〔反例核验定位：附录未译、中文PDF未收录，附录B.7 表20–23的Rnd5000/Fshr5000列；[英文PDF 第21–22页](https://arxiv.org/pdf/2509.25426v3#page=21)；Ctrl+F「Rnd 5000」。表20 GPQA两者同0.7535；MMLU随机0.8745>Fisher0.8731，未见显著性检验支持“一定显著”。 附录精确定位：[附录未译：B.7 Ablation Study on Adaptive Testing，第20–21页](https://arxiv.org/pdf/2509.25426v3#page=20)，首段“We conduct an ablation study”与第二段“For example”；[表20，第21页](https://arxiv.org/pdf/2509.25426v3#page=21)为ID Hypervolume，[表21–23，第22页](https://arxiv.org/pdf/2509.25426v3#page=22)依次为ID CPT、OOD Hypervolume、OOD CPT，核对Rnd/Fshr各预算列。〕 |
| Radar 所有基准均优于最强基线 | **错误，MATH-500 的 ID Hypervolume 是反例** 〔反例核验定位：[中文PDF 第8页](pdfs/zh/RADAR_Chinese_Body.pdf#page=8)，表1 MATH-500行；[英文PDF 第7页](https://arxiv.org/pdf/2509.25426v3#page=7)；Ctrl+F「0.9528」。〕 |
| 只跑一次 | **不准确：AIME 和时延等实验有多次运行** 〔原文定位：[中文PDF 第8页](pdfs/zh/RADAR_Chinese_Body.pdf#page=8)、[中文PDF 第10页](pdfs/zh/RADAR_Chinese_Body.pdf#page=10)，§4实现设置；§4.3时延；[英文PDF 第8、10页](https://arxiv.org/pdf/2509.25426v3#page=8)；Ctrl+F「eight runs」。AIME8次，时延3次。〕 |
| 有非负 a 约束/没有非负 a 约束 | **正文没写明实现约束；应分“公式没有强制”与“代码如何实现”** 〔待代码定位：[中文PDF 第5–6页](pdfs/zh/RADAR_Chinese_Body.pdf#page=5)，§3.3线性映射及2PL式；附录D.5；[英文PDF 第5–6、29页](https://arxiv.org/pdf/2509.25426v3#page=5)；Ctrl+F「linear transformations」。D.5附录未译、中文PDF未收录；不把公式缺省当作代码层面的否定事实。 附录精确定位：[附录未译：D.5 IRT Implementation Details in RADAR，第29页](https://arxiv.org/pdf/2509.25426v3#page=29)，唯一段落“We employ”，嵌入句含“frozen-weight”“4096”，训练句含“100 epochs”。〕 |

## 候选破题点

**R-1 [原文+推导→待实验]：Fisher 估能力，未直接以路由后悔值选题。** 设计决策价值导向的主动测试（如 expected utility improvement、VoI），对比同预算 Fisher、随机、D-optimal 和难度分层。但要先检查收益是否可能被新模型“不值得接入”的情形冲淡。 〔原文依据→研究设想定位：[中文PDF 第6–9页](pdfs/zh/RADAR_Chinese_Body.pdf#page=6)，§3.5 Eq.(7–9)；§4.2表3；[英文PDF 第6–9页](https://arxiv.org/pdf/2509.25426v3#page=6)；Ctrl+F「Fisher」。VoI、D-optimal等同预算比较为本文方案，非作者实验或已核实的新颖性。〕

**R-2 [原文→待实验]：每配置固定输出成本，长上下文与输出长度动态性未建模。** 用真实逐题输入/输出费用替换常数成本，测 Pareto 曲线和决策翻转；若收益主要来自费用而非能力预测，论文贡献应诚实说明。 〔原文依据→待实验定位：[中文PDF 第6页](pdfs/zh/RADAR_Chinese_Body.pdf#page=6)，§3.4 Eq.(6)及future work句；[英文PDF 第6页](https://arxiv.org/pdf/2509.25426v3#page=6)；Ctrl+F「elaborate cost」。动态费用与翻转分析由本文提出。〕

**R-3 [推导→待实验]：同一底座不同预算没有共享能力结构。** 尝试层次先验或“底座能力 + 预算增量”共享统计强度，尤其在少量校准反馈下比较。但不能假设能力对预算严格单调，需要保留 overthinking 的例外。 〔模型结构推导→待实验定位：[中文PDF 第4页](pdfs/zh/RADAR_Chinese_Body.pdf#page=4)、[中文PDF 第6页](pdfs/zh/RADAR_Chinese_Body.pdf#page=6)，§3.1、§3.3每配置θ；[英文PDF 第4–6页](https://arxiv.org/pdf/2509.25426v3#page=4)；Ctrl+F「configuration」。层次先验/预算增量为本文研究方案，非原文方法。〕

**R-4 [原文→待实验]：未讨论在何时提前停测或拒绝接入。** 5000 是实验配置，不等于每个新模型都值得花足预算。先测试顺序停测策略是否能节省评测成本、且不损失后续部署收益；也要查后续 adaptive profiling 文献确认新颖性。 〔报告范围→研究设想定位：[中文PDF 第7页](pdfs/zh/RADAR_Chinese_Body.pdf#page=7)、[中文PDF 第9页](pdfs/zh/RADAR_Chinese_Body.pdf#page=9)，§3.5固定预算T停止；§4.2 5k实验；[英文PDF 第7、9页](https://arxiv.org/pdf/2509.25426v3#page=7)；Ctrl+F「budget」。本文版本未给基于路由价值的提前停止/拒接规则；不表示整个相关领域未研究。〕

## 老师可能问

1. 为什么 5 个底座模型却有 35 个候选？因为路由决策对象是“模型与推理预算”的组合。 〔原文＋计数定位：[中文PDF 第4页](pdfs/zh/RADAR_Chinese_Body.pdf#page=4)、[中文PDF 第8页](pdfs/zh/RADAR_Chinese_Body.pdf#page=8)，§3.1；§4配置列表；[英文PDF 第4、8页](https://arxiv.org/pdf/2509.25426v3#page=4)；Ctrl+F「35」。〕
2. Fisher 选题是不是随机挑最难的？不是，它最大化在当前能力估计位置的信息量，考察区分度及不确定性。 〔原文＋讲解定位：[中文PDF 第7页](pdfs/zh/RADAR_Chinese_Body.pdf#page=7)，§3.5 Eq.(8–9)；[英文PDF 第6页](https://arxiv.org/pdf/2509.25426v3#page=6)；Ctrl+F「Fisher」。〕
3. 5000 是零样本接入吗？不是；新配置确实需要运行大量校准题，零样本不成立。 〔原文＋判断定位：[中文PDF 第9页](pdfs/zh/RADAR_Chinese_Body.pdf#page=9)，§4.2；与§3.5测评流程对照；[英文PDF 第6–7、9页](https://arxiv.org/pdf/2509.25426v3#page=6)；Ctrl+F「5k training」。〕
4. Hypervolume 好不代表所有用户都收益更好吗？是的；要结合预算偏好、Pareto 曲线和具体成本点。 〔指标解释定位：[中文PDF 第7页](pdfs/zh/RADAR_Chinese_Body.pdf#page=7)，§4 Metrics；Eq.(4)偏好权重在中文5页；[英文PDF 第5、7页](https://arxiv.org/pdf/2509.25426v3#page=5)；Ctrl+F「CPT」。〕
5. 笔记中的 “deployment-aware profiling” 能否直接称创新？不能；当前只是候选问题，必须用强基线、文献排重与部署效用实验共同证实。 〔研究建议定位：[中文PDF 第6–9页](pdfs/zh/RADAR_Chinese_Body.pdf#page=6)，§3.5与§4.2/表3；[英文PDF 第6–9页](https://arxiv.org/pdf/2509.25426v3#page=6)；Ctrl+F「RADAR++」。deployment-aware profiling为本文候选方向，文中无相同创新性证明。〕

**快速定位**：§3.1 配置定义；§3.2 Chebyshev Eq.(4)；§3.3 性能预测未编号概率式、训练损失 Eq.(5)；§3.4 成本 Eq.(6)；§3.5 Fisher Eq.(7–9)；表 1–3 主实验；附录 B.7 自适应选题对照、B.8 新模型扩展；附录 D.2 硬件、D.3 基线、D.4 QA 提示词、D.1 评估设置、D.5 IRT 实现、D.6 指标。 〔导航定位：[中文PDF 第4–9页](pdfs/zh/RADAR_Chinese_Body.pdf#page=4)，方法§3.1–3.5；实验§4.2及表1–3；[英文PDF 第4–9页](https://arxiv.org/pdf/2509.25426v3#page=4)；Ctrl+F「ADAPTIVE TESTING」。附录定位：B.7英文20–22、B.8英文21–23；D.2英文27、D.3–D.4英文28–29、D.5–D.6英文29；均附录未译、中文PDF未收录。 附录精确定位：[附录未译：D.2 Hardware，第27页](https://arxiv.org/pdf/2509.25426v3#page=27)，唯一段落“For all open-source models”，含“vLLM”“NVIDIA A100 80GB”“single such GPU”；[附录未译：D.3 Baselines，第28页](https://arxiv.org/pdf/2509.25426v3#page=28)，唯一长段“For RouterBench”；IRT适配定位“We adapt a concurrent”，Oracle定位“The Oracle router”；[附录未译：D.4 QA Prompts，第28–29页](https://arxiv.org/pdf/2509.25426v3#page=28)，按“For AIME and MATH”“For GPQA”“For MMLU Pro”“For FRAMES”各提示段定位；FRAMES代码续至第29页；[附录未译：D.5 IRT Implementation Details in RADAR，第29页](https://arxiv.org/pdf/2509.25426v3#page=29)，唯一段落“We employ”，嵌入句含“frozen-weight”“4096”，训练句含“100 epochs”；[附录未译：D.6 Metrics，第29页](https://arxiv.org/pdf/2509.25426v3#page=29)，唯一长段“Our formulation”，Hypervolume释义查“area under”，CPT定义查“Given a performance threshold”；[附录未译：B.7 Ablation Study on Adaptive Testing，第20–21页](https://arxiv.org/pdf/2509.25426v3#page=20)，首段“We conduct an ablation study”与第二段“For example”；[表20，第21页](https://arxiv.org/pdf/2509.25426v3#page=21)为ID Hypervolume，[表21–23，第22页](https://arxiv.org/pdf/2509.25426v3#page=22)依次为ID CPT、OOD Hypervolume、OOD CPT，核对Rnd/Fshr各预算列；[附录未译：B.8 Model Scalability and Generalization Evaluation of RADAR，第21页](https://arxiv.org/pdf/2509.25426v3#page=21)，唯一段落“We evaluate the scalability”含“8 new model configurations”“5k training queries”；[图4，第23页](https://arxiv.org/pdf/2509.25426v3#page=23)为FRAMES接入前后路由比例。〕
