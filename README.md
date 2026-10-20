# Tennis Momentum · 2024 MCM C

**English documentation:** see [README.en.md](README.en.md). English report and supporting documents are linked there; the Chinese originals remain available below.

**发球优势调整后的网球势头：从走势量化到可证伪的预测实验。**

基于 COMAP 官方的 **31 场比赛、7,284 个逐分记录**，完整实现在线发球基线、EWMA 势头、条件置换检验、逐分预测和未来反转预测。包含中文研究报告、真实运行结果、可复现代码和 GitHub Actions 配置。

这是独立的赛后研究示范，**不是获奖论文或正式参赛提交**。不预设“势头一定存在”，也不以复杂模型必然优于基线为目标。

[English report](docs/report.en.md) · [English PDF](docs/report.en.pdf) · [English offline HTML](docs/report.en.html) · [完整中文报告](docs/report.zh-CN.md) · [中文 PDF](docs/report.zh-CN.pdf) · [中文离线 HTML](docs/report.html) · [教练备忘录](docs/COACH_MEMO.md) · [官方赛题](https://www.contest.comap.com/undergraduate/contests/mcm/contests/2024/problems/2024_MCM_Problem_C.pdf)

![决赛的发球调整势头和逐分概率](results/figures/final_flow.png)

## 主要结果

| 问题 | 实际实验结论 |
| --- | --- |
| 能否量化走势？ | 可以：发球调整残差的 EWMA 给出方向和百分点尺度；决赛各盘平均为 −8.76、+5.15、+13.66、−6.74、+1.13。 |
| 能否证明非随机“势头”？ | 本检验不能：全局条件置换 p=0.405；62 项逐场检验经 BH 调整后无显著项。 |
| 历史信息改善下一分预测吗？ | 未见改善：测试 Log loss 从比分模型的 0.6434 增至历史模型的 0.6458，配对损失差 95% 区间跨零。 |
| 能预测未来反转吗？ | 结构模型测试 AUC=0.6096，扩展模型 AUC=0.5620；存在窗口重叠、标签高发生率和几乎一直报警的问题。 |

**重要解释：** 未拒绝零假设不等于证明势头不存在；看起来合理的走势不等于可预测的心理机制。图中红点是事后确认的指标反转，不是提前预测命中点。模型不用于未经验证的实时战术建议。

## 快速开始

建议使用 Python **3.12**。在仓库根目录执行：

```bash
python -m venv .venv
# Windows PowerShell
.venv\Scripts\Activate.ps1
# macOS / Linux 使用：source .venv/bin/activate
python -m pip install -r requirements.txt
python scripts/download_data.py
python -m src.analysis
python -m unittest discover -s tests -v
```

数据下载需要联网；数据就绪后，实验和测试可离线运行。默认执行 1,999 次置换、3,000 次比赛聚类自助抽样和 9 组参数比较。运行时长取决于 CPU 和数值库，程序会在结束时打印汇总。

仅检查代码而不下载数据时，八项测试中有一项真实数据规模测试会跳过，其他测试使用明确标注的合成夹具。合成数据不参与报告实验。

可用 `python -m src.analysis --permutations 99` 快速调试，**但这会覆盖结果文件，不能将调试结果与默认报告混用**。发布前重新运行默认命令。

## 模型与验证设计

1. **在线基线。** 训练轮次估计发球率；每位球员在当前比赛内用 Beta-Bernoulli 后验更新发球能力，只读取过去的分。
2. **势头。** `m[t] = (1-alpha)*m[t-1] + alpha*(y[t]-p[t])`。正值表示球员 1 超过基线；输出乘 100 后为百分点。
3. **状态与反转。** 势头超过 ±0.08 时进入对应状态，中间区域保持旧状态。预测未来 10 分内是否发生至少一次相反状态切换。
4. **随机性。** 在每场、每种发球身份内部置换得分，保留实际发球位置和分层得分数。检验一阶关联及累计偏离，统一校正 62 个比较。
5. **预测比较。** 正则化 Logistic 回归以在线基线的 log odds 为 offset，比较比分和历史信息；反转任务比较常数、结构和扩展模型。

| 划分 | 轮次 | 比赛 | 分数 | 用途 |
| --- | --- | ---: | ---: | --- |
| 训练 | 第 3 轮 | 16 | 3,788 | 估计基线与模型 |
| 验证 | 第 4 轮 | 8 | 1,919 | 选择 α、κ 和报警阈值 |
| 测试 | 四分之一决赛至决赛 | 7 | 1,577 | 最终报告，不参与选参 |

训练特征使用留一比赛的总体发球率；验证和测试使用训练集发球率。所有技术统计按比赛滞后，选参后不合并训练验证重训。跨轮次可能包含相同球员，因此不声称这是全新选手验证。

## 结果导航

- [机器可读汇总](results/summary.json)：数据规模、配置、区间、环境版本。
- [逐分模型指标](results/tables/point_metrics.csv) / [逐场指标](results/tables/point_by_match.csv)。
- [随机性检验](results/tables/permutation_tests.csv)：原始 p 值及 BH 调整值。
- [反转指标](results/tables/swing_metrics.csv) / [逐场反转指标](results/tables/swing_by_match.csv)。
- [逐分预测](results/tables/point_predictions.csv) / [反转窗口预测](results/tables/swing_predictions.csv)。
- [参数敏感性](results/tables/sensitivity.csv) / [反转定义敏感性](results/tables/swing_sensitivity_validation.csv)。
- [数据说明](docs/DATA_CARD.md) / [模型说明](docs/MODEL_CARD.md) / [AI 使用说明](docs/AI_USAGE.md)。

![逐分预测对照](results/figures/point_comparison.png)

## 仓库结构

```text
.
├── README.md
├── LICENSE                     # 原创代码与文档的 MIT 许可
├── CITATION.cff
├── requirements.txt
├── requirements-report.txt
├── .github/workflows/reproduce.yml
├── data/raw/README.md           # 原始 CSV 被 Git 忽略
├── src/
│   ├── model.py                # 因果特征、在线基线、Logistic
│   └── analysis.py             # 训练、检验、评估、图表
├── scripts/
│   ├── download_data.py        # 官方下载与 SHA-256 校验
│   ├── build_report.py         # Markdown / HTML / PDF
│   └── report_content.py       # 中文报告叙述与结果表
├── tests/test_models.py
├── results/
│   ├── summary.json
│   ├── tables/
│   └── figures/
└── docs/
    ├── report.zh-CN.md
    ├── report.zh-CN.pdf
    ├── report.html
    └── ...
```

## 重建报告

```bash
python -m pip install -r requirements-report.txt
python scripts/build_report.py
# Linux 或非默认中文字体环境：
python scripts/build_report.py --font /path/to/cjk-font.ttf
```

PDF 需要 TrueType 格式中文字体。Windows 默认查找微软雅黑/黑体；Linux 可用文泉驿正黑的 TTC。字体不随仓库分发。HTML 内嵌图像，可以离线阅读；GitHub 直接阅读 Markdown 最方便。

默认报告的叙述对应已经运行的固定实验。表格读取 CSV，但部分解释中的数字是固定叙述；更改模型或参数后须核对并修改 `scripts/report_content.py`，再重建三种报告。所有已交付结果对应默认 1,999 次置换。

## 来源、许可与披露

官方数据与题目来自 [COMAP](https://www.comap.org/membership/member-resources/item/momentum-in-tennis)。原始数据不在发布 ZIP 内，项目不重新授权第三方材料。原创代码、图表和分析文档使用 [MIT](LICENSE)。
