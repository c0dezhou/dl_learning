# IRT-Router：把大模型当成“考生”，把问题当成“试题”

> 英文题名：**IRT-Router: Effective and Interpretable Multi-LLM Routing via Item Response Theory**  
> 中文阅读入口：[中文正文与原文附录合集（22页）](pdfs/zh/IRT-Router_Chinese.pdf#page=1)。正文已译；附录A/B/C保留英文。  
> 英文对照来源：[arXiv:2506.01048v2（英文原文）](https://arxiv.org/html/2506.01048v2)，2025 年版本。  
> 核验依据：库中英文 PDF 的方法、实验、附录，尤其 §3、§4、§5、§6、表 1–4、附录 A/C。  
> 证据标记：**[原文]** 论文直接支持；**[推导]** 源于公式或设计；**[待代码]** 正文不能验证实现细节；**[待实验]** 尚未证实。


## 原文定位速查（本次新增）

- 跳转说明：本页优先链接仓库内中文PDF；#page=N表示从1开始的物理页序。GitHub的PDF预览可能忽略页码片段；若未跳到指定页，请下载PDF，用阅读器打开并输入文中所列物理页码。页码已按PDF内容核对；GitHub预览的页码跳转尚未实测。范围链接跳到该范围首页，节号、公式/表格及段落关键词用于进一步定位。
- 页码口径：下文“中文PDF”专指本仓库所附《IRT-Router_Chinese.pdf》（原文件名《IRT-Router_中文译文_含图表与公式.pdf》），共22页；所有页码都是阅读器从1开始的物理页序。[中文PDF 第16–22页](pdfs/zh/IRT-Router_Chinese.pdf#page=16)为原英文第10–16页的附页，不能把两套页码混用。英文来源固定为 arXiv:2506.01048v2 /《IRT-Router.pdf》，共16页；已与官网下载PDF逐字节核对一致。
- 候选池与价格：[中文PDF 第9页](pdfs/zh/IRT-Router_Chinese.pdf#page=9)§5.2、第19页附录A.2/表5；[英文第13页](https://arxiv.org/pdf/2506.01048v2#page=13)。查“GPT-4o Mini+COT”“0.0137”。
- MIRT/NIRT参数与公式：[中文PDF 第6–8页](pdfs/zh/IRT-Router_Chinese.pdf#page=6)§4.1–4.2、Eq.(4–9)；[英文第5页](https://arxiv.org/pdf/2506.01048v2#page=5)。25维/训练设置在[中文PDF 第10页](pdfs/zh/IRT-Router_Chinese.pdf#page=10)§5.5；768维在[原文附录（该PDF内保留英文） 第19页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)表6。
- Warm-up：[中文PDF 第8页](pdfs/zh/IRT-Router_Chinese.pdf#page=8)Eq.(10)，第14页图6、第15页图7，第20页表9；[英文第6页](https://arxiv.org/pdf/2506.01048v2#page=6)与[第14页](https://arxiv.org/pdf/2506.01048v2#page=14)。
- 选择成本 vs 真实总费用：[中文PDF 第5页](pdfs/zh/IRT-Router_Chinese.pdf#page=5)§3.1 Eq.(1)及脚注1，对照第10页§5.4 Eq.(11–12)；[英文第4页](https://arxiv.org/pdf/2506.01048v2#page=4)、[第7页](https://arxiv.org/pdf/2506.01048v2#page=7)。
- 主结果：[中文PDF 第10–11页](pdfs/zh/IRT-Router_Chinese.pdf#page=10)表1（ID）、第11–12页表2（OOD）；[英文第7页](https://arxiv.org/pdf/2506.01048v2#page=7)。Top-k在[中文PDF 第13页](pdfs/zh/IRT-Router_Chinese.pdf#page=13)表3/[英文第8页](https://arxiv.org/pdf/2506.01048v2#page=8)；新模型在[中文PDF 第14页](pdfs/zh/IRT-Router_Chinese.pdf#page=14)表4/[英文第9页](https://arxiv.org/pdf/2506.01048v2#page=9)。
- 简介来源：[原文附录（该PDF内保留英文） 第19页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)附录A.1；完整简介第22页表10；[英文第13页](https://arxiv.org/pdf/2506.01048v2#page=13)、[第16页](https://arxiv.org/pdf/2506.01048v2#page=16)。
- 定位标记只担保对应的作者事实；“推导/教学例子/研究设想/待代码”仍是本文分析。Ctrl+F词优先给原文英文/数字；中文正文可用节号或对应中文标题，附页可直接搜英文。页码链接跳到所列范围的第一页。
- 校订：决策argmax同时涉及Eq.(1–2)，已补全；Warm-up的ID/OOD比较补充附录B表9。未核验官方代码，故实现约束仍保留待核验。

## 先理解研究问题

一位学生做题是否正确，除了题目难度，还取决于这个学生擅长什么。大模型也一样：有些擅长数学，有些擅长编程、中文或常识。**IRT-Router 借鉴教育测量学的项目反应理论（Item Response Theory，IRT），学习“题目属性、模型能力、回答质量”的关系**，再按质量与成本权衡选一个模型。 〔原文＋讲解定位：[中文PDF 第3页](pdfs/zh/IRT-Router_Chinese.pdf#page=3)、[中文PDF 第5–8页](pdfs/zh/IRT-Router_Chinese.pdf#page=5)，§1、§3.2、§4；[英文PDF 第2、4–6页](https://arxiv.org/pdf/2506.01048v2#page=2)；Ctrl+F「IRT」。考生/试题类比是对作者建模的教学转述。〕

它试图解决两个传统路由器的困难：一是模型 × 题目关系很难解释；二是很多做法只能在训练过的固定模型池上使用。IRT-Router 还使用**自然语言模型简介**来给没有参加训练的新模型预估能力。不过请先记住：**可以预测新模型，不等于已经证明新模型加入后能提高路由效用。** 〔原文＋边界判断定位：[中文PDF 第6页](pdfs/zh/IRT-Router_Chinese.pdf#page=6)、[中文PDF 第14页](pdfs/zh/IRT-Router_Chinese.pdf#page=14)，§4.1；§6.3 New LLM、表4；[英文PDF 第4–5、9页](https://arxiv.org/pdf/2506.01048v2#page=4)；Ctrl+F「New LLM」。新模型实验检验预测误差；未报告扩池后的路由曲线。〕

### 三条不同的信息，不要混淆

1. **题目特征**：题目是不是难、需要哪一种能力？ 〔讲解定位：[中文PDF 第5–8页](pdfs/zh/IRT-Router_Chinese.pdf#page=5)，§3.2、§4.2 Eq.(4–9)；[英文PDF 第4–5页](https://arxiv.org/pdf/2506.01048v2#page=4)；Ctrl+F「discrimination」。〕
2. **模型特征**：模型的能力画像是什么？ 〔讲解定位：[中文PDF 第6–8页](pdfs/zh/IRT-Router_Chinese.pdf#page=6)，§4.1–§4.2 Eq.(4)、(9)；[英文PDF 第4–5页](https://arxiv.org/pdf/2506.01048v2#page=4)；Ctrl+F「profile」。〕
3. **决策偏好**：更看重答对还是更看重便宜？ 〔讲解定位：[中文PDF 第5页](pdfs/zh/IRT-Router_Chinese.pdf#page=5)，§3.1 Eq.(1–2)；[英文PDF 第3–4页](https://arxiv.org/pdf/2506.01048v2#page=3)；Ctrl+F「trade-off」。〕

作者先学习 1 和 2 来预测成绩，再使用 3 作最终决策。 〔原文＋讲解定位：[中文PDF 第6页](pdfs/zh/IRT-Router_Chinese.pdf#page=6)、[中文PDF 第8页](pdfs/zh/IRT-Router_Chinese.pdf#page=8)，图3、§4.3；[英文PDF 第4–5页](https://arxiv.org/pdf/2506.01048v2#page=4)；Ctrl+F「Routing Decision」。〕

## ① 候选空间：20 个“待选项”，不一定全是不同底座模型

[原文，§5.2、附录 A.2] 使用 **20 个候选 LLM/配置**。作者将模型大致分为 API 模型、轻量/可量化模型、专门用途模型和增强版本，各类可重叠。输出价格跨度很大，示例从 **GLM-4-Flash 的 0.0137 美元/百万输出 token** 到 **GPT-4o 的 10 美元/百万输出 token**。 〔原文定位：[中文PDF 第9页](pdfs/zh/IRT-Router_Chinese.pdf#page=9)、[原文附录（该PDF内保留英文） 第19页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)，§5.2；附录A.2、表5的GLM-4-Flash/GPT-4o行；[英文PDF 第6、13页](https://arxiv.org/pdf/2506.01048v2#page=6)；Ctrl+F「0.0137」。第19页为中文PDF所附英文原页；价格是论文实验口径，非当前报价。 附录精确定位：[原文附录（该PDF内保留英文）A.2 Cadidate LLMs and their Pricing，第19页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)，分类段落“Cadidate LLMs are categorized”、末段“The first 20 LLMs”及表5；价格核对GLM-4-Flash/GPT-4o行。〕

需要记住一个容易被问到的例子：**GPT-4o Mini 与 GPT-4o Mini + CoT 被当作两个待选项**。底座模型可能相同，但提示方式不同，回答质量可能不同。由此可以看出，路由的选择对象未必只能是“模型名”，也可以是“模型 + 一种使用方式”。但是不能据此声称作者已经建立了 RADAR 那样完整的模型 × 思考预算配置研究。 〔原文＋推导定位：[原文附录（该PDF内保留英文） 第19页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)，附录A.2 Enhanced Models、表5两条GPT-4o Mini行；[英文PDF 第13页](https://arxiv.org/pdf/2506.01048v2#page=13)；Ctrl+F「Enhanced Models」。相同底座不同提示是作者设置；与RADAR研究范围的区别为本文比较。 附录精确定位：[原文附录（该PDF内保留英文）A.2 Cadidate LLMs and their Pricing，第19页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)，“Enhanced Models”项目及表5的GPT-4o Mini/GPT-4o Mini+COT两行。〕

本论文不需要自己的 GPU 部署完这 20 个模型才能理解其方法，因为作者使用构建好的逐题响应与评分数据；若要**从零重建实验数据**，仍需调用对应的模型或拥有完整响应缓存。 〔复现判断定位：[中文PDF 第8页](pdfs/zh/IRT-Router_Chinese.pdf#page=8)、[原文附录（该PDF内保留英文） 第19页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)，§5数据构造；附录A.3；[英文PDF 第6、13页](https://arxiv.org/pdf/2506.01048v2#page=6)；Ctrl+F「488600」。作者逐题生成20模型响应；“理解方法无须部署全部模型”是本文建议，缓存可用性未在此核实。 附录精确定位：[原文附录（该PDF内保留英文）A.3 Datasets Details，第19页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)，唯一段落“Table 7 shows”、数字“24430”“488600”；[表7，第20页](pdfs/zh/IRT-Router_Chinese.pdf#page=20)，核对数据集行及Evaluation Metric/Train Size/Test Size列。〕

## ② 题目表示：MIRT 和 NIRT 的区别从这里开始

[原文，§4.1–§4.2] 输入题目 `q` 由预训练 BERT（实验使用 `bert-base-uncased`，768 维）得到向量 `e_q`。之后通过可训练变换映射成 IRT 参数： 〔原文定位：[中文PDF 第6–8页](pdfs/zh/IRT-Router_Chinese.pdf#page=6)、[中文PDF 第10页](pdfs/zh/IRT-Router_Chinese.pdf#page=10)、[原文附录（该PDF内保留英文） 第19页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)，§4.1–4.2、§5.5；表6 bert-base-uncased行；[英文PDF 第4–5、7、13页](https://arxiv.org/pdf/2506.01048v2#page=4)；Ctrl+F「768」。 附录精确定位：[原文附录（该PDF内保留英文）B Sensitivity Analysis，第19页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)，“Embedding Models”段及表6的bert-base-uncased/768行。〕

| 属性 | MIRT-Router | NIRT-Router |
|---|---|---|
| 区分度 `a` | **25 维向量** | **1 个标量** 〔原文定位：[中文PDF 第7–8页](pdfs/zh/IRT-Router_Chinese.pdf#page=7)、[中文PDF 第10页](pdfs/zh/IRT-Router_Chinese.pdf#page=10)，Eq.(4)、(9)；§5.5 N=25；[英文PDF 第5、7页](https://arxiv.org/pdf/2506.01048v2#page=5)；Ctrl+F「Dimension」。〕 |
| 难度 `b` | **1 个标量** | **25 维向量** 〔原文定位：[中文PDF 第7–8页](pdfs/zh/IRT-Router_Chinese.pdf#page=7)、[中文PDF 第10页](pdfs/zh/IRT-Router_Chinese.pdf#page=10)，Eq.(4)、(9)；§5.5 N=25；[英文PDF 第5、7页](https://arxiv.org/pdf/2506.01048v2#page=5)；Ctrl+F「Dimension」。〕 |
| 题目能力相关性 `r` | 无单独输入 | **25 维**，标注/近邻构造 〔原文定位：[中文PDF 第7–8页](pdfs/zh/IRT-Router_Chinese.pdf#page=7)、[中文PDF 第10页](pdfs/zh/IRT-Router_Chinese.pdf#page=10)，§4.2.2 Relevance Vector、Eq.(9)；§5.5；[英文PDF 第5、7页](https://arxiv.org/pdf/2506.01048v2#page=5)；Ctrl+F「Relevance Vector」。〕 |
| 模型能力 `θ` | **25 维向量** | **25 维向量**（sigmoid 约束） 〔原文定位：[中文PDF 第7–8页](pdfs/zh/IRT-Router_Chinese.pdf#page=7)、[中文PDF 第10页](pdfs/zh/IRT-Router_Chinese.pdf#page=10)，Eq.(4)、(9)；§5.5；[英文PDF 第5、7页](https://arxiv.org/pdf/2506.01048v2#page=5)；Ctrl+F「sigmoid」。〕 |

**更好懂的区分**：MIRT 让题目有“偏重哪种能力”的 25 维权重，但只给它一个总体难度；NIRT 明确规定数学、编程等 25 种能力维度，题目可以对每种能力分别有难度，再用 `r` 表示实际需要哪些能力。原笔记对“25 维/标量”的更正是对的，不能写反。 〔原文＋讲解定位：[中文PDF 第7–8页](pdfs/zh/IRT-Router_Chinese.pdf#page=7)、[原文附录（该PDF内保留英文） 第20–21页](pdfs/zh/IRT-Router_Chinese.pdf#page=20)，§4.2；附录C.1的25项能力；[英文PDF 第5、14–15页](https://arxiv.org/pdf/2506.01048v2#page=5)；Ctrl+F「Predefined Abilities」。 附录精确定位：[原文附录（该PDF内保留英文）C.1 Predefined Abilities，第20–21页](pdfs/zh/IRT-Router_Chinese.pdf#page=20)，首段“We first predefine”与[第21页](pdfs/zh/IRT-Router_Chinese.pdf#page=21)“The 25 predefined”列表（0–24）。〕

NIRT 的相关性 `r` 如何拿到？[原文，§4.2.2] 先对训练题 BERT 表示做 UMAP 降维、HDBSCAN 聚类；从每类抽 5 道交给一个 LLM，判断涉及预先定义的 25 种能力；测试题不能偷偷看标签，而是找 5 道最近训练题，对它们的相关性向量求平均，然后归一化参与预测。这里引入额外的聚类、标注及能力定义依赖。 〔原文定位：[中文PDF 第7–8页](pdfs/zh/IRT-Router_Chinese.pdf#page=7)、[原文附录（该PDF内保留英文） 第21页](pdfs/zh/IRT-Router_Chinese.pdf#page=21)，§4.2.2、Eq.(9)；附录C.2提示词；[英文PDF 第5、15页](https://arxiv.org/pdf/2506.01048v2#page=5)；Ctrl+F「HDBSCAN」。测试相关性按5-NN均值近似；归一化具体为softmax。 附录精确定位：[原文附录（该PDF内保留英文）C.2 Prompt for Getting Relevance Vector，第21页](pdfs/zh/IRT-Router_Chinese.pdf#page=21)，唯一段落“In Section 4.2.2”含“GPT-4o Mini”，具体提示词见图9。〕

### Warm-up 不是重新训练路由器

[原文，§4.4] 对一条新题，找训练库里与它最相近的若干道题，把原来的 embedding 轻微往近邻平均表示靠拢： 〔原文定位：[中文PDF 第8页](pdfs/zh/IRT-Router_Chinese.pdf#page=8)，§4.4 Eq.(10)；[英文PDF 第6页](https://arxiv.org/pdf/2506.01048v2#page=6)；Ctrl+F「Warm-Up」。〕

$$
 e'_q=(1-\lambda)e_q+\lambda\,\overline e_{NN(q)}.
$$

例如 λ=0.3，表示保留 70% 原题向量，同时加入 30% 相似旧题的平均表示。它是**输入向量校正**，不是在线更新 θ，也不是新模型测试的 profiling。实验中这项调整对 OOD 效果更明显。 〔原文＋数值推导定位：[中文PDF 第8页](pdfs/zh/IRT-Router_Chinese.pdf#page=8)、[中文PDF 第14页](pdfs/zh/IRT-Router_Chinese.pdf#page=14)、[原文附录（该PDF内保留英文） 第20页](pdfs/zh/IRT-Router_Chinese.pdf#page=20)，Eq.(10)；§6.3图6；附录B表9；[英文PDF 第6、9、14页](https://arxiv.org/pdf/2506.01048v2#page=6)；Ctrl+F「Cold-Start Parameter」。70%/30%为代入式(10)；ID/OOD强弱比较看表9，§6.3图6本身仅给OOD。 附录精确定位：[原文附录（该PDF内保留英文）B Sensitivity Analysis，第20页](pdfs/zh/IRT-Router_Chinese.pdf#page=20)，“Cold-Start Parameter λ”段（含“both ID and OOD”）及表9的ID/OOD列。〕

## ③ 模型表示：只看“简历”就给模型估能力

[原文，§4.1、附录 A.1] 每个模型先有一段简介：发布时间、开发机构、类型、特点、简短描述等。**简介来源为使用 ChatGPT 的联网搜索，之后人工校正**。简介经过同一类 BERT 编码，得到 `e_M`，再映射到 25 维的模型能力 θ： 〔原文定位：[中文PDF 第6–7页](pdfs/zh/IRT-Router_Chinese.pdf#page=6)、[原文附录（该PDF内保留英文） 第19页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)、[原文附录（该PDF内保留英文） 第22页](pdfs/zh/IRT-Router_Chinese.pdf#page=22)，§4.1、Eq.(4)；附录A.1；表10；[英文PDF 第4–5、13、16页](https://arxiv.org/pdf/2506.01048v2#page=4)；Ctrl+F「manually」。25维设置另见[中文PDF 第10页](pdfs/zh/IRT-Router_Chinese.pdf#page=10)/英文第7页§5.5。 附录精确定位：[原文附录（该PDF内保留英文）A.1 LLM Profile，第19页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)，唯一段落以“Table 10 presents”开头、含“manually corrected”；完整简介见[表10，第22页](pdfs/zh/IRT-Router_Chinese.pdf#page=22)。〕

$$
 \boldsymbol\theta_M=W_\theta e_M.
$$

NIRT 的 θ 经过 sigmoid，映射到 (0,1)；MIRT 的 θ 没有对应的 sigmoid 形式。这里真正学到的是“简介文字特征和实际能力之间的统计联系”。 〔原文＋解释定位：[中文PDF 第7–8页](pdfs/zh/IRT-Router_Chinese.pdf#page=7)，Eq.(4)、(9)；[英文PDF 第5页](https://arxiv.org/pdf/2506.01048v2#page=5)；Ctrl+F「sigmoid」。〕

**优点**：一个新模型发布后，只要写得出简介，就可能生成 θ，不必先完成几百道测试题。 〔方法推导定位：[中文PDF 第6–7页](pdfs/zh/IRT-Router_Chinese.pdf#page=6)、[中文PDF 第14页](pdfs/zh/IRT-Router_Chinese.pdf#page=14)，§4.1、Eq.(4)；§6.3；[英文PDF 第4–5、9页](https://arxiv.org/pdf/2506.01048v2#page=4)；Ctrl+F「New LLM」。描述可直接映射能力；这不是作者报告的“0题预算实验”。〕

**代价与风险 [推导]**：简介中的宣传、发布时间、公司名称并不一定等于真实数学或编程能力；只有 20 个左右候选模型用于学习“简介到能力”的关系，可能很难稳定外推；不同 CoT 配置只靠细微描述差异也未必能学到效果变化。严格说，论文测试的是这个假设在给定数据上的效果，而不是证明它普适成立。 〔研究假设定位：[中文PDF 第6页](pdfs/zh/IRT-Router_Chinese.pdf#page=6)、[中文PDF 第14页](pdfs/zh/IRT-Router_Chinese.pdf#page=14)、[原文附录（该PDF内保留英文） 第19页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)、[原文附录（该PDF内保留英文） 第22页](pdfs/zh/IRT-Router_Chinese.pdf#page=22)，§4.1、§6.3；表5/10；[英文PDF 第4–5、9、13、16页](https://arxiv.org/pdf/2506.01048v2#page=4)；Ctrl+F「limited generalization」。简介依赖、候选规模及泛化有限有原文依据；宣传风险/CoT识别风险尚未实测。 附录精确定位：[原文附录（该PDF内保留英文）A.1 LLM Profile，第19页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)，唯一段落以“Table 10 presents”开头、含“manually corrected”；完整简介见[表10，第22页](pdfs/zh/IRT-Router_Chinese.pdf#page=22)；[原文附录（该PDF内保留英文）A.2 Cadidate LLMs and their Pricing，第19页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)，分类段落“Cadidate LLMs are categorized”、末段“The first 20 LLMs”及表5；价格核对GLM-4-Flash/GPT-4o行。〕

## ④ 成绩预测：两个版本的关键公式

### MIRT-Router

[原文 Eq. (4)(5)] 〔原文定位：[中文PDF 第7页](pdfs/zh/IRT-Router_Chinese.pdf#page=7)，§4.2.1 Eq.(4–5)；[英文PDF 第5页](https://arxiv.org/pdf/2506.01048v2#page=5)；Ctrl+F「Interactive Function」。〕

$$
\widehat P(q,M)=\sigma\!\left(\mathbf a_q^{\top}\boldsymbol\theta_M-b_q\right),\qquad
\sigma(z)=\frac{1}{1+e^{-z}}.
$$

读成一句话：**某题看重的能力 × 该模型的能力，加起来越高，相对题目难度越占优，预计表现就越好**。 〔公式讲解定位：[中文PDF 第7页](pdfs/zh/IRT-Router_Chinese.pdf#page=7)，Eq.(5)；[英文PDF 第5页](https://arxiv.org/pdf/2506.01048v2#page=5)；Ctrl+F「Interactive Function」。〕

教学示例：一题对数学能力权重较高，模型 A 数学强而模型 B 弱，即便两个模型整体排名差不多，A 的预测得分也可能更高。 〔教学例子定位：[中文PDF 第7页](pdfs/zh/IRT-Router_Chinese.pdf#page=7)，Eq.(5)；[英文PDF 第5页](https://arxiv.org/pdf/2506.01048v2#page=5)；Ctrl+F「MIRT-Router」。A/B数学特长为自拟示例；MIRT隐维不能无校准直接命名为数学维。〕

### NIRT-Router

[原文 Eq. (7)–(9)] 〔原文定位：[中文PDF 第7–8页](pdfs/zh/IRT-Router_Chinese.pdf#page=7)，§4.2.2 Eq.(7–9)；[英文PDF 第5页](https://arxiv.org/pdf/2506.01048v2#page=5)；Ctrl+F「Interactive Function」。〕

$$
\mathbf x_{qM}=\mathbf r_q\odot(\boldsymbol\theta_M-\mathbf b_q)\,a_q,
\qquad
\widehat P(q,M)=\sigma\!\left(\phi(W_1\mathbf x_{qM}^{\top}+\mathbf b_1)\right).
$$

`⊙` 为逐维相乘；`r` 表示这道题到底调用哪几种能力；`θ−b` 表示在这些能力维度上模型能否超过题目要求；最后的小网络学习更复杂的组合关系。 〔公式讲解定位：[中文PDF 第7–8页](pdfs/zh/IRT-Router_Chinese.pdf#page=7)，Eq.(7–9)；[英文PDF 第5页](https://arxiv.org/pdf/2506.01048v2#page=5)；Ctrl+F「Relevance Vector」。r经softmax后是连续权重，不宜理解成永远严格为0/1的掩码。〕

论文在有些地方讨论能力/难度的单调关系，但**仅靠 MIRT 公式并不能自动保证对 θ 的每个分量都单调增加**：还需要对应的区分度分量非负。原笔记声称“源码用了 softplus 保证 a≥0”，这一点**没有在正文公式或文字中直接验证**，不能没有代码证据就写成已确认事实。[待代码] 〔数学推导／待代码定位：[中文PDF 第5页](pdfs/zh/IRT-Router_Chinese.pdf#page=5)、[中文PDF 第7–8页](pdfs/zh/IRT-Router_Chinese.pdf#page=7)、[中文PDF 第15页](pdfs/zh/IRT-Router_Chinese.pdf#page=15)，§3.2；Eq.(4–5)、(9)；Limitations；[英文PDF 第4–5、9页](https://arxiv.org/pdf/2506.01048v2#page=4)；Ctrl+F「Monotonicity」。偏导为σ′(aᵀθ−b)·a的对应分量；公式未给softplus，实现尚未核验。〕

两种版本训练时使用每个模型在每道题上的实际评分 `y_{qM}` 和预测分数之间的**二元交叉熵**；公式接受 0 到 1 的软标签，所以某些数据集的质量分数也可以不是严格 0/1。 〔原文＋公式解释定位：[中文PDF 第7–8页](pdfs/zh/IRT-Router_Chinese.pdf#page=7)、[原文附录（该PDF内保留英文） 第20页](pdfs/zh/IRT-Router_Chinese.pdf#page=20)，Eq.(6)、NIRT Training；§5；表7 SQUAD f1行；[英文PDF 第5–6、14页](https://arxiv.org/pdf/2506.01048v2#page=5)；Ctrl+F「binary cross-entropy」。 附录精确定位：[原文附录（该PDF内保留英文）A.3 Datasets Details，第19页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)，唯一段落“Table 7 shows”、数字“24430”“488600”；[表7，第20页](pdfs/zh/IRT-Router_Chinese.pdf#page=20)，核对数据集行及Evaluation Metric/Train Size/Test Size列。〕

## ⑤ 成本：原文确实有“选择口径”和“评价口径”差异

[原文，§3.1、§5.4] 〔原文定位：[中文PDF 第5页](pdfs/zh/IRT-Router_Chinese.pdf#page=5)、[中文PDF 第10页](pdfs/zh/IRT-Router_Chinese.pdf#page=10)，§3.1、§5.4 Eq.(1)、(11)；[英文PDF 第3–4、7页](https://arxiv.org/pdf/2506.01048v2#page=3)；Ctrl+F「Total Cost」。〕

**用来选模型的成本**：按每百万输出 token 定价，把各模型价格线性缩放到 0–1。比如 GPT-4o 的输出单价 10 美元时归一化为 1；DeepSeek-Chat 的 0.28 美元归一化为 `0.28/10=0.028`。这个数**不随题目长度或实际输出长度变化**。 〔原文定位：[中文PDF 第5页](pdfs/zh/IRT-Router_Chinese.pdf#page=5)，§3.1 Eq.(1)与脚注1；[英文PDF 第3–4页](https://arxiv.org/pdf/2506.01048v2#page=3)；Ctrl+F「0.028」。〕

**最后展示的 Total Cost**：按每道测试题实际输入 token × 输入单价，加上实际输出 token × 输出单价，最后对测试集求和，单位美元。 〔原文定位：[中文PDF 第10页](pdfs/zh/IRT-Router_Chinese.pdf#page=10)，§5.4 Total Cost、Eq.(11)；[英文PDF 第7页](https://arxiv.org/pdf/2506.01048v2#page=7)；Ctrl+F「input_pricing」。〕

因此有潜在不一致：模型被选中时并没有使用测试报告中的那套完整逐题计费公式。[原文 + 推导] 这很适合设计验证，但不能未经实验就断言这种近似必然让性能下降。 〔两式比较／待实验定位：[中文PDF 第5页](pdfs/zh/IRT-Router_Chinese.pdf#page=5)、[中文PDF 第10页](pdfs/zh/IRT-Router_Chinese.pdf#page=10)，Eq.(1)成本说明 vs Eq.(11)；[英文PDF 第3–4、7页](https://arxiv.org/pdf/2506.01048v2#page=3)；Ctrl+F「fixed cost」。口径不同可直接核对；造成何种性能影响是本文研究假设。〕

## ⑥ 决策目标：为什么把 α 调了却可能选不出明显变化？

[原文，Eq. (1–2)] 〔原文定位：[中文PDF 第5页](pdfs/zh/IRT-Router_Chinese.pdf#page=5)，§3.1 Eq.(1–2)；[英文PDF 第3–4页](https://arxiv.org/pdf/2506.01048v2#page=3)；Ctrl+F「scoring function」。本段argmax合并了式(1)评分与式(2)选择。〕

$$
 M^*(q)=\arg\max_M\{\alpha\widehat P(q,M)-\beta C(M)\},
\qquad\alpha+\beta=1.
$$

实验只报告 `α=0.8/0.5/0.2` 三组权重，对应的 β 分别为 `0.2/0.5/0.8`。给定 `α=0.8`、DeepSeek-Chat `C=0.028`，它的成本罚分只有 `βC=0.0056`，的确可能比不同模型间的预测得分差更小。[数值推导] 〔原文＋数值推导定位：[中文PDF 第5页](pdfs/zh/IRT-Router_Chinese.pdf#page=5)、[中文PDF 第10–12页](pdfs/zh/IRT-Router_Chinese.pdf#page=10)，脚注1；表1–2三个α列组；[英文PDF 第4、7页](https://arxiv.org/pdf/2506.01048v2#page=4)；Ctrl+F「0.028」。0.0056=0.2×0.028，由本文计算；对α不敏感另见中文15/英文9 Limitations。〕

**不能直接得出的结论**：既然这款模型罚分很小，成本“对任何决策都没有作用”。是否翻转，取决于**两个候选项预测性能差和价格差的相对大小**；要看逐题选择变化统计，而不是只看一个模型的罚分。 〔数学推导定位：[中文PDF 第5页](pdfs/zh/IRT-Router_Chinese.pdf#page=5)，Eq.(1–2)；[英文PDF 第3–4页](https://arxiv.org/pdf/2506.01048v2#page=3)；Ctrl+F「scoring function」。比较两候选须比较αΔP与βΔC；原文没有本段逐题翻转实验。〕

## ⑦ 新模型接入：请分清“零题预测”和“零题证明收益”

[原文，§6.3、表 4] 作者以 `Claude 3.5 Haiku 20241022` 作为新模型，**不使用它参加路由训练的逐题表现，而根据简介预测其在 ID 测试题上的质量**。原文表 4 给出： 〔原文＋设置解释定位：[中文PDF 第14页](pdfs/zh/IRT-Router_Chinese.pdf#page=14)、[原文附录（该PDF内保留英文） 第19页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)，§6.3 New LLM、表4；附录A.2末句及表5末行；[英文PDF 第9、13页](https://arxiv.org/pdf/2506.01048v2#page=9)；Ctrl+F「Claude 3.5 Haiku」。 附录精确定位：[原文附录（该PDF内保留英文）A.2 Cadidate LLMs and their Pricing，第19页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)，末段“The first 20 LLMs”及表5末行“Claude 3.5 Haiku 20241022 (New)”。〕

| 方法 | RMSE ↓ | MAE ↓ | AUC ↑ | ACC ↑ |
|---|---:|---:|---:|---:|
| RouterBench 预测基线 | 0.62 | 0.55 | 0.50 | 0.34 〔原文定位：[中文PDF 第14页](pdfs/zh/IRT-Router_Chinese.pdf#page=14)，表4 RouterBench行；[英文PDF 第9页](https://arxiv.org/pdf/2506.01048v2#page=9)；Ctrl+F「0.55」。〕 |
| MIRT-Router | 0.45 | 0.43 | 0.62 | 0.67 〔原文定位：[中文PDF 第14页](pdfs/zh/IRT-Router_Chinese.pdf#page=14)，表4 MIRT-Router行；[英文PDF 第9页](https://arxiv.org/pdf/2506.01048v2#page=9)；Ctrl+F「0.43」。〕 |
| NIRT-Router | 0.45 | 0.42 | 0.62 | 0.68 〔原文定位：[中文PDF 第14页](pdfs/zh/IRT-Router_Chinese.pdf#page=14)，表4 NIRT-Router行；[英文PDF 第9页](https://arxiv.org/pdf/2506.01048v2#page=9)；Ctrl+F「0.42」。〕 |

这是“新模型表现预测”的回归/分类指标。**并没有据此报告“接入 Claude 后整条路由成本—质量曲线提升多少”**。作者自己承认对未见模型的能力泛化有限。因此“0 道题接入”只能表示算法可从简介构造一个模型表示，而不能说 0 道题已经实现可靠的路由最优化。 〔核验范围／方法推导定位：[中文PDF 第14页](pdfs/zh/IRT-Router_Chinese.pdf#page=14)，§6.3 New LLM、表4；[英文PDF 第9页](https://arxiv.org/pdf/2506.01048v2#page=9)；Ctrl+F「limited generalization」。本节仅四种预测指标；不能将其当作新模型路由收益实验。〕

## ⑧ 实验：结果要把 ID、OOD、总体准确度、Top-k 区分开

[原文，§5–§6] 数据为 12 个数据集 × 20 个模型。8 个 ID 集合：MMLU、CMMLU、ACLUE、ARC-Challenge、HotpotQA、SQuAD、MATH、MBPP，各自 70% 训练、30% 测试后合并；4 个 OOD 集合：CEVAL、CommonsenseQA、GSM8K、HumanEval。各数据集可能采用不同的题目评分方式。基线有 Small/large fixed LLM、HybridLLM、RouteLLM、RouterBench 的预测式路由。 〔原文定位：[中文PDF 第8–10页](pdfs/zh/IRT-Router_Chinese.pdf#page=8)、[原文附录（该PDF内保留英文） 第19–20页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)，§5–5.3；附录A.3、表7；[英文PDF 第6–7、13–14页](https://arxiv.org/pdf/2506.01048v2#page=6)；Ctrl+F「70%」。 附录精确定位：[原文附录（该PDF内保留英文）A.3 Datasets Details，第19页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)，唯一段落“Table 7 shows”、数字“24430”“488600”；[表7，第20页](pdfs/zh/IRT-Router_Chinese.pdf#page=20)，核对数据集行及Evaluation Metric/Train Size/Test Size列。〕

**把这两张表记准，千万不能写反**：

| 设置，α=0.8 | RouterBench Performance | MIRT Performance | NIRT Performance | MIRT 实际 Total Cost |
|---|---:|---:|---:|---:|
| **ID，表 1** | **80.01%** | **80.67%** | **80.69%** | **$0.42** 〔原文定位：[中文PDF 第10页](pdfs/zh/IRT-Router_Chinese.pdf#page=10)，表1 α=0.8，RouterBench/MIRT/NIRT行；[英文PDF 第7页](https://arxiv.org/pdf/2506.01048v2#page=7)；Ctrl+F「80.67%」。〕 |
| **OOD，表 2** | **85.50%** | **87.12%** | **87.37%** | **$0.14** 〔原文定位：[中文PDF 第11页](pdfs/zh/IRT-Router_Chinese.pdf#page=11)，表2 α=0.8，RouterBench/MIRT/NIRT行；[英文PDF 第7页](https://arxiv.org/pdf/2506.01048v2#page=7)；Ctrl+F「87.12%」。〕 |

为什么 OOD 分数高于 ID？**不能解释为“越分布外越准”**，因为测试题来自不同数据集、题型、评分方式，绝对百分数不能当作同条件泛化能力比较。看同一行下方法间的差值才较可比。 〔比较方法提醒定位：[中文PDF 第8–9页](pdfs/zh/IRT-Router_Chinese.pdf#page=8)、[原文附录（该PDF内保留英文） 第20页](pdfs/zh/IRT-Router_Chinese.pdf#page=20)，§5.1；表7的数据集与Evaluation Metric列；[英文PDF 第6、14页](https://arxiv.org/pdf/2506.01048v2#page=6)；Ctrl+F「SQUAD」。不同测试集的绝对分数不可直接当作同条件泛化提升。 附录精确定位：[原文附录（该PDF内保留英文）A.3 Datasets Details，第19页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)，唯一段落“Table 7 shows”、数字“24430”“488600”；[表7，第20页](pdfs/zh/IRT-Router_Chinese.pdf#page=20)，核对数据集行及Evaluation Metric/Train Size/Test Size列。〕

[原文，表 3] MIRT 对“最优模型”的选择准确率很低： 〔原文定位：[中文PDF 第13页](pdfs/zh/IRT-Router_Chinese.pdf#page=13)，§6.2 Routing Accuracy、表3；α=0.8，20候选；[英文PDF 第8页](https://arxiv.org/pdf/2506.01048v2#page=8)；Ctrl+F「Top-k」。〕

| 场景 | Top-1 | Top-3 | Top-5 |
|---|---:|---:|---:|
| ID | **2.72%** | 32.51% | 47.85% 〔原文定位：[中文PDF 第13页](pdfs/zh/IRT-Router_Chinese.pdf#page=13)，表3 ID行、Top-1/3/5列；[英文PDF 第8页](https://arxiv.org/pdf/2506.01048v2#page=8)；Ctrl+F「2.72%」。〕 |
| OOD | **2.15%** | 27.29% | 39.47% 〔原文定位：[中文PDF 第13页](pdfs/zh/IRT-Router_Chinese.pdf#page=13)，表3 OOD行、Top-1/3/5列；[英文PDF 第8页](https://arxiv.org/pdf/2506.01048v2#page=8)；Ctrl+F「2.15%」。〕 |

作者解释，目标同时考虑价格与质量、很多模型近似并列，严格的 Top-1 不一定等于实际效用差异大。但这仍说明，**“平均成绩高”和“准确识别逐题最优模型”是两个不同目标**，不能把 Top-1 的弱成绩省略。 〔原文＋解释定位：[中文PDF 第13–14页](pdfs/zh/IRT-Router_Chinese.pdf#page=13)，§6.2 Routing Accuracy、表3后解释；[英文PDF 第8–9页](https://arxiv.org/pdf/2506.01048v2#page=8)；Ctrl+F「lightweight」。作者另给轻量参数少的原因；效用损失需额外实验。〕

论文报告平均 Performance、美元 Total Cost、归一化后的 Reward 和 Top-k。没有提供覆盖整个连续成本偏好的 RouterBench 式凸包曲线；表 1–2 主要是在三个 α 下对比，未见对主要结果完整的独立种子方差报告。 〔报告范围核验定位：[中文PDF 第10–14页](pdfs/zh/IRT-Router_Chinese.pdf#page=10)、[原文附录（该PDF内保留英文） 第19–21页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)，§5.4；表1–4；附录B敏感性分析；[英文PDF 第7–9、13–15页](https://arxiv.org/pdf/2506.01048v2#page=7)；Ctrl+F「Reward」。主要表未给独立随机种子均值±方差，也未报告连续偏好凸包评估；限于本文版本。 附录精确定位：[原文附录（该PDF内保留英文）B Sensitivity Analysis，第19–20页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)，依次为“Embedding Models”“Dimension N”“Cold-Start Parameter λ”段及表6/8/9；B没有编号子节。〕

## 原稿纠错清单

| 需要核对的旧表述 | 判定 |
|---|---|
| MIRT 的 a=25 维，b=标量；NIRT 的 a=标量，b=25 维 | **正确** 〔原文定位：[中文PDF 第7–8页](pdfs/zh/IRT-Router_Chinese.pdf#page=7)、[中文PDF 第10页](pdfs/zh/IRT-Router_Chinese.pdf#page=10)，Eq.(4)、(9)；§5.5；[英文PDF 第5、7页](https://arxiv.org/pdf/2506.01048v2#page=5)；Ctrl+F「Dimension」。〕 |
| 模型简介经 ChatGPT 联网搜索生成后人工校正 | **原文附录 A.1 支持** 〔原文定位：[原文附录（该PDF内保留英文） 第19页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)，附录A.1 LLM Profile；[英文PDF 第13页](https://arxiv.org/pdf/2506.01048v2#page=13)；Ctrl+F「manually」。 附录精确定位：[原文附录（该PDF内保留英文）A.1 LLM Profile，第19页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)，唯一段落以“Table 10 presents”开头、含“manually corrected”；完整简介见[表10，第22页](pdfs/zh/IRT-Router_Chinese.pdf#page=22)。〕 |
| BERT 一定冻结 | **正文证据不足，应核对代码** 〔待代码定位：[中文PDF 第6页](pdfs/zh/IRT-Router_Chinese.pdf#page=6)、[中文PDF 第10页](pdfs/zh/IRT-Router_Chinese.pdf#page=10)，图3及§4.1、§5.5；[英文PDF 第4、7页](https://arxiv.org/pdf/2506.01048v2#page=4)；Ctrl+F「Fixed Parameters」。图3有固定/可学习参数图例；运行时冻结细节仍须代码核验，不据此断言没有任何图示线索。〕 |
| softplus 使 a 非负 | **正文未写，暂标为待代码** 〔待代码定位：[中文PDF 第7–8页](pdfs/zh/IRT-Router_Chinese.pdf#page=7)，Eq.(4)、(9)；[英文PDF 第5页](https://arxiv.org/pdf/2506.01048v2#page=5)；Ctrl+F「learnable」。〕 |
| α=0.8 时 MIRT 87.12% 是 ID | **错误；87.12% 是 OOD** 〔原文定位：[中文PDF 第11页](pdfs/zh/IRT-Router_Chinese.pdf#page=11)，表2 α=0.8 MIRT-Router行；[英文PDF 第7页](https://arxiv.org/pdf/2506.01048v2#page=7)；Ctrl+F「87.12%」。〕 |
| 新模型接入后实测路由收益提高 | **没有这个实验** 〔范围核验定位：[中文PDF 第14页](pdfs/zh/IRT-Router_Chinese.pdf#page=14)，§6.3 New LLM、表4；[英文PDF 第9页](https://arxiv.org/pdf/2506.01048v2#page=9)；Ctrl+F「RMSE」。此版本只报告新模型预测质量。〕 |
| 成本打分只用输出价格，评测考虑输入+输出 | **正确，需分口径说明** 〔原文定位：[中文PDF 第5页](pdfs/zh/IRT-Router_Chinese.pdf#page=5)、[中文PDF 第10页](pdfs/zh/IRT-Router_Chinese.pdf#page=10)，§3.1与§5.4，Eq.(1)、(11)；[英文PDF 第3–4、7页](https://arxiv.org/pdf/2506.01048v2#page=3)；Ctrl+F「input_pricing」。〕 |

## 候选破题点（不是已证明的 gap）

**IRT-1 [原文→待实验]：模型“简历”能否可靠估计能力？** 在仅凭描述的冷启动中，新模型 MAE 0.43、ACC 0.67，作者承认有余地。可以比较 0/8/16/32 道实测题校正 θ 的收益，但必须与后来用少样本 profiling 的论文比较，不能假称无人解决。 〔原文依据→研究设想定位：[中文PDF 第14页](pdfs/zh/IRT-Router_Chinese.pdf#page=14)，§6.3 New LLM、表4 MIRT-Router行；[英文PDF 第9页](https://arxiv.org/pdf/2506.01048v2#page=9)；Ctrl+F「0.43」。0/8/16/32为本文拟议对照，原文未做；新颖性尚待文献检索。〕

**IRT-2 [推导→待实验]：优化目标与计费口径不一致。** 测试逐题成本而选择使用模型级输出单价，可能在长提示/长回答上造成错选。下一步：以相同质量预测器替换“固定成本、估计逐题成本、真实成本上界”，测得路由翻转比例和质量—成本曲线。 〔推导→待实验定位：[中文PDF 第5页](pdfs/zh/IRT-Router_Chinese.pdf#page=5)、[中文PDF 第10页](pdfs/zh/IRT-Router_Chinese.pdf#page=10)、[中文PDF 第15页](pdfs/zh/IRT-Router_Chinese.pdf#page=15)，Eq.(1)、(11)；Limitations短查询段；[英文PDF 第3–4、7、9页](https://arxiv.org/pdf/2506.01048v2#page=3)；Ctrl+F「fixed cost」。替换成本和翻转实验为本文设计，非原文已证实gap。〕

**IRT-3 [原文→待实验]：Top-1 极低究竟是近似并列还是模型预测失误？** 固定成本系数，按 top-1 与最优模型的效用差 `U*(q)-U_selected(q)` 分桶；若低 Top-1 但效用损失小，确实无需大动；若损失集中在某些数据，则可定位真实的失败点。 〔原文依据→研究设想定位：[中文PDF 第13–14页](pdfs/zh/IRT-Router_Chinese.pdf#page=13)，§6.2 Routing Accuracy、表3及解释；[英文PDF 第8–9页](https://arxiv.org/pdf/2506.01048v2#page=8)；Ctrl+F「Top-1」。按效用损失分桶是本文提案，论文未给该分析。〕

**IRT-4 [待代码]：多维单调性与语义解释是否在实现中被约束？** 检查每个 a、θ、b 变换与激活函数，以及同一模型能力某分量增加时预测是否一定不减；再判断所谓“可解释能力维度”是否真能跨数据集稳定对应。 〔待代码／待实验定位：[中文PDF 第5页](pdfs/zh/IRT-Router_Chinese.pdf#page=5)、[中文PDF 第7–8页](pdfs/zh/IRT-Router_Chinese.pdf#page=7)、[中文PDF 第15页](pdfs/zh/IRT-Router_Chinese.pdf#page=15)、[原文附录（该PDF内保留英文） 第20–21页](pdfs/zh/IRT-Router_Chinese.pdf#page=20)，§3.2；Eq.(4–9)；Limitations；附录C.1；[英文PDF 第4–5、9、14–15页](https://arxiv.org/pdf/2506.01048v2#page=4)；Ctrl+F「constraints」。正文式子与语义定义是核验起点，不足以证明实现单调或跨域稳定。 附录精确定位：[原文附录（该PDF内保留英文）C.1 Predefined Abilities，第20–21页](pdfs/zh/IRT-Router_Chinese.pdf#page=20)，首段“We first predefine”与[第21页](pdfs/zh/IRT-Router_Chinese.pdf#page=21)“The 25 predefined”列表（0–24）。〕

## 老师可能问

1. **MIRT 和 NIRT 哪个更可解释？** NIRT 把 25 维对应预定义能力且显式引入相关向量；MIRT 的 25 维更偏隐空间。 〔原文＋讲解定位：[中文PDF 第7–8页](pdfs/zh/IRT-Router_Chinese.pdf#page=7)、[原文附录（该PDF内保留英文） 第20–21页](pdfs/zh/IRT-Router_Chinese.pdf#page=20)，§4.2、附录C.1；[英文PDF 第5、14–15页](https://arxiv.org/pdf/2506.01048v2#page=5)；Ctrl+F「Predefined Abilities」。 附录精确定位：[原文附录（该PDF内保留英文）C.1 Predefined Abilities，第20–21页](pdfs/zh/IRT-Router_Chinese.pdf#page=20)，首段“We first predefine”与[第21页](pdfs/zh/IRT-Router_Chinese.pdf#page=21)“The 25 predefined”列表（0–24）。〕
2. **IRT 的“难度”就是 embedding 范数吗？** 不是，是由模型学习的映射结果，含义来自 IRT 结构假设而不是天然物理量。 〔公式推导定位：[中文PDF 第7–8页](pdfs/zh/IRT-Router_Chinese.pdf#page=7)，Eq.(4)、(9)；[英文PDF 第5页](https://arxiv.org/pdf/2506.01048v2#page=5)；Ctrl+F「learnable weights」。〕
3. **新模型一题不做也能用吗？** 技术上能从描述估 θ，但论文检验的只是新模型成绩预测，准确度有限，不能把它当成可靠接入收益的证明。 〔方法推导＋证据边界定位：[中文PDF 第6–7页](pdfs/zh/IRT-Router_Chinese.pdf#page=6)、[中文PDF 第14页](pdfs/zh/IRT-Router_Chinese.pdf#page=14)，§4.1、Eq.(4)；§6.3表4；[英文PDF 第4–5、9页](https://arxiv.org/pdf/2506.01048v2#page=4)；Ctrl+F「New LLM」。〕
4. **什么叫基线复现成功？** 至少要核对 20 模型池、数据切分、评分、归一化以及 α 设置，不能只复用其中一张表。 〔复现检查建议定位：[中文PDF 第5页](pdfs/zh/IRT-Router_Chinese.pdf#page=5)、[中文PDF 第8–12页](pdfs/zh/IRT-Router_Chinese.pdf#page=8)、[原文附录（该PDF内保留英文） 第19–20页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)，§3.1；§5；表1–2/5/7；[英文PDF 第3–4、6–7、13–14页](https://arxiv.org/pdf/2506.01048v2#page=3)；Ctrl+F「Datasets Details」。 附录精确定位：[原文附录（该PDF内保留英文）A.2 Cadidate LLMs and their Pricing，第19页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)，分类段落“Cadidate LLMs are categorized”、末段“The first 20 LLMs”及表5；价格核对GLM-4-Flash/GPT-4o行；[原文附录（该PDF内保留英文）A.3 Datasets Details，第19页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)，唯一段落“Table 7 shows”、数字“24430”“488600”；[表7，第20页](pdfs/zh/IRT-Router_Chinese.pdf#page=20)，核对数据集行及Evaluation Metric/Train Size/Test Size列。〕

**原文定位**：问题定义及固定成本 Eq.(1) 在 §3.1；MIRT Eq.(4–6)、NIRT Eq.(7–9) 在 §4.2；Warm-up Eq.(10) 在 §4.4；总费用 Eq.(11) 在 §5.4；ID/OOD 表 1/2、Top-k 表 3、新模型表 4 在 §6；简介的制作方式在附录 A.1。 〔导航定位：[中文PDF 第5页](pdfs/zh/IRT-Router_Chinese.pdf#page=5)、[中文PDF 第7–8页](pdfs/zh/IRT-Router_Chinese.pdf#page=7)、[中文PDF 第10–14页](pdfs/zh/IRT-Router_Chinese.pdf#page=10)、[原文附录（该PDF内保留英文） 第19页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)，本段对应各节、公式及表；[英文PDF 第3–9、13页](https://arxiv.org/pdf/2506.01048v2#page=3)；Ctrl+F「Problem Definition」。具体同页表行与英文跳转见各段及上方速查。 附录精确定位：[原文附录（该PDF内保留英文）A.1 LLM Profile，第19页](pdfs/zh/IRT-Router_Chinese.pdf#page=19)，唯一段落以“Table 10 presents”开头、含“manually corrected”；完整简介见[表10，第22页](pdfs/zh/IRT-Router_Chinese.pdf#page=22)。〕
