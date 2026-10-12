# LLM 路由六篇：来一个 query 怎样选模型，新模型怎样接入

六篇论文沿两条主线来读：

- **主线 A：来一个 query，怎样选到最合适的模型？** 质量预测、成本估计、决策规则。
- **主线 B：一个新模型上线，要测多少题、测哪些题，才能接进路由池？** 我们的研究问题在它的尽头。

两条线的关系是 **B 为主、A 为骨架**：要判断哪道测试题最值得测，先得知道路由器之后怎么用这个新模型。

## 从哪里开始读

- **想看全貌、六篇之间的关系和研究问题** → [00 综述](00_综述.md)（按问题组织，横向比较六篇）。
- **想精读某一篇** → 下面的笔记（按论文组织，每篇里同时讲它在 A、B 两条线上做了什么）。

| 笔记 | 在主线 A 上 | 在主线 B 上 | 中文材料 | PDF |
|---|---|---|---|---|
| [01 RouterBench](01_RouterBench_精读核验.md)（2024.03） | 定好尺子：Zero 基线、Oracle、AIQ | 不涉及；提供评测框架 | [精读材料](translations/01_RouterBench_中文精读材料.md)（非全文） | — |
| [02 RouteLLM](02_RouteLLM_精读核验.md)（2024.06） | 强弱二选一，偏好数据 + 阈值 | 0 题：直接换一对强弱模型 | [译文](translations/02_RouteLLM_中文译文.md) | [下载](pdfs/zh/RouteLLM_Chinese_Full.pdf) |
| [03 IRT-Router](03_IRT-Router_精读核验.md)（2025.06） | 能力 × 难度的可解释质量预测 | 0 题：凭一段模型简介 | [译文](translations/03_IRT-Router_中文译文.md) | [下载](pdfs/zh/IRT-Router_Chinese.pdf) |
| [04 ZeroRouter](04_ZeroRouter_精读核验.md)（2026.01） | 逐题估计成本和延迟 | 200 道 D-optimal 锚点题 | [译文](translations/04_ZeroRouter_中文译文.md) | [下载](pdfs/zh/ZeroRouter_Chinese.pdf) |
| [05 ICL-Router](05_ICL-Router_精读核验.md)（2025.10） | LLM 读成绩册，只看准确率 | 500 道固定题的对错记录 | [译文](translations/05_ICL-Router_中文译文.md) | [下载](pdfs/zh/ICL-Router_Chinese.pdf) |
| [06 RADAR](06_RADAR_精读核验.md)（2025.09） | 模型 × 推理预算；切比雪夫权衡 | Fisher 信息自适应逐题选题 | [译文](translations/06_RADAR_中文译文.md) | [下载](pdfs/zh/RADAR_Chinese_Body.pdf) |

编号按逻辑而非发表时间排，原因见[综述](00_综述.md#两条主线)。统一记号、对照总表、研究问题与论文路线、指标速查和补充阅读，都在综述里。

## 仓库结构

```
papers/llm-routing/
├── README.md                 ← 本页：入口
├── 00_综述.md                ← 两条主线、记号、对照、研究问题
├── 01_RouterBench_精读核验.md … 06_RADAR_精读核验.md   ← 精读笔记
├── translations/             ← 中文译文（编号与笔记一致）
├── pdfs/zh/                  ← 中文 PDF
└── assets/                   ← 笔记与译文用到的图表
```
