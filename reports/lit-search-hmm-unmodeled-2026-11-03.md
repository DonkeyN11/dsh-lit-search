# 文献调研报告：HMM 中"真值不可表示"的建模与地图匹配侧的现状

- 委托：Donkey.Ning（宁宸章）
- 执行：DSH `@copylee/dsh-academic@0.5.1` + 用户级技能 `sjtu-lit-search`
- 日期：2026-11-03
- 工具链：`paper_search` / `paper_citations` / `paper_get` / `paper_cite` / `reference_verify`
- 参考文献核验：`reference_verify` **5/5 VERIFIED**

---

## 1. 检索日志

| # | 查询 | 源 | 参数 | 结果 |
|---|---|---|---|---|
| 1 | hidden Markov model misspecification unmodeled hidden state robustness | openalex, crossref, semanticscholar, arxiv | limit 6 | 6 条；arXiv 429、S2 首轮 429 |
| 2 | map matching HMM likelihood confidence uncertainty quantification | openalex, crossref, semanticscholar | limit 6 | 6 条；S2 恢复 |
| 3 | background outlier component mixture HMM robust likelihood misspecification | openalex, crossref, semanticscholar | limit 6 | 6 条 |
| 4 | map matching missing road network candidate generation failure confidence | openalex, crossref, semanticscholar | limit 6 | 6 条（噪声大） |
| 5 | hidden Markov model background state unmodeled state map matching misspecification | openalex, crossref, semanticscholar, **googlescholar** | limit 8, sort=citations | 8 条；**Scholar 经系统代理 7890 成功** |
| 6 | 地图匹配 置信度 隐马尔可夫 不确定性 | **cnki** | limit 5 | 0 条（查询过窄） |
| 7 | 地图匹配 置信度 | **cnki** | limit 5 | 5 条（均为定位/SLAM 学位论文） |
| 引文追溯 A | Newson & Krumm 2009 的被引 | openalex | limit 6 | **1060 篇**；最高被引全是轨迹挖掘/交通预测 |
| 引文追溯 B | Quddus 2006 的被引 | openalex | limit 8 | DOI 解析到另一篇论文（见 §6 更正） |

**未答源**：arXiv（两次 429）、Semantic Scholar（首轮 429，带 key 后仍受 ~1 req/s 限制）。

## 2. 选中语料

| 文献 | 年份/venue | 被引 | 与本问题的关系 |
|---|---|---|---|
| [Extra hidden states under emission misspecification in HMM](https://doi.org/10.1111/stan.70042) | 2026 · Statistica Neerlandica | 0 | **核心支撑**：额外隐状态可补偿发射分布失配；组件过少时观测律 KL 损失随序列长至少线性增长；精确细化或聚合混合发射可消除缺口；同时警告"拟合状态数 ≠ 真实 regime 数" |
| [Doubly Outlier-Robust Online Infinite HMM](https://doi.org/10.48550/ARXIV.2604.14322) | 2026 · arXiv | 0 | 数据含离群且模型失配时的稳健在线更新（广义贝叶斯 + 后验影响函数） |
| [Semiparametric Robust Estimation of Population Location](https://doi.org/10.48550/ARXIV.2512.03021) | 2025 · arXiv | 0 | 主成分参数化、背景完全非参数化，给出"说不清的部分整体交给非参数"的范式 |
| [Hidden Markov map matching through noise and sparseness](https://doi.org/10.1145/1653771.1653818) | 2009 · ACM SIGSPATIAL | 1060（被引） | HMM 地图匹配基准；其被引前六名中 5 篇是轨迹挖掘/交通预测、1 篇是交互投票匹配（Yuan 2010），**无一做匹配不确定度** |
| [Integrity of map-matching algorithms](https://doi.org/10.1016/j.trc.2006.08.004) | 2006 · TR-C | 108 | 地图匹配侧最接近"可信性"的工作，但其完整性是算法级指标，不是后验校准 |
| [Quantification of Map Matching Credibility Based on GNSS Positioning Covariance](https://doi.org/10.1007/978-981-95-6366-1_24) | 2026 · LNEE | 0 | 作者本人前作：用 GNSS 协方差量化匹配可信度，是本文的直接基线 |

中文侧（CNKI，5 条）集中在定位/SLAM 学位论文（4 篇：鲁棒 SLAM、点云-IMU-激光融合定位、轻量化语义地图定位、融合定位避障）与 1 篇地图图片匹配（SAM + Gluestick），年份 2024–2025，来自哈工大/东南/华南理工/兰州交大/重庆大学，**没有检索到专门处理"地图匹配置信度/失配态"的中文文献**。

## 3. 主题综合：三种处理"真值不可表示"的方式

1. **加状态（state refinement）**——用一个显式的额外隐状态吸收未被模型覆盖的变异。Nicosia 2026 给出了它的理论代价与合法性：组件不足时 KL 损失线性增长，而精确细化或聚合混合发射可以消除这一近似缺口；但"状态数"本身不能解释为"机制数"。
2. **稳健化似然/更新（robustification）**——不改状态空间，改推断规则。Yiu 2026 用广义贝叶斯与后验影响函数，让 iHMM 在离群与失配下仍收敛；代价是需要调节稳健性强度。
3. **非参数背景成分**——把无法参数化的部分整体交给非参数项（Barua & Basu 2025），主成分仍参数化，兼顾可解释与稳健。

地图匹配侧：HMM 匹配器（Newson & Krumm 及其 1060 篇后继）默认**候选集完整、发射模型正确**；路网缺失或尾部误差被当作匹配失败而非一个可建模状态。Quddus 2006 的完整性指标与作者前作的协方差可信度，是仅有的两块"承认不确定"的工作，但都不是"把失配建模成状态再回灌管线"。

## 4. 缺口与候选贡献

- **没有**工作在地图匹配 HMM 中显式引入"背景态 / 失配态"来承载路网缺失与尾部误差；
- **没有**工作把这些未建模隐状态放回管线、重跑实验并以校准指标（ECE / Brier / 可靠性图）度量改善；
- **没有**中文文献触及该问题。

这三点组合起来正是可主张的贡献；Nicosia 2026 可作为"加状态"的理论依据，Yiu 2026 可作稳健化对照，Quddus 2006 与前作构成基线谱系。

## 5. 参考文献（`paper_cite` 生成，`reference_verify` 5/5 通过）

1. A. Nicosia, "Extra hidden states under emission misspecification in hidden Markov models," *Statistica Neerlandica*, vol. 80, no. 4, Oct. 2026, doi: 10.1111/stan.70042.
2. H. Yiu, L. Sánchez-Betancourt, Á. Cartea, and G. Duran-Martin, "Doubly Outlier-Robust Online Infinite Hidden Markov Model," 2026, arXiv. doi: 10.48550/ARXIV.2604.14322.
3. P. Newson and J. Krumm, "Hidden Markov map matching through noise and sparseness," in *Proc. 17th ACM SIGSPATIAL Int. Conf. Advances in Geographic Information Systems*, 2009, pp. 336–343. doi: 10.1145/1653771.1653818.
4. M. A. Quddus, W. Y. Ochieng, and R. B. Noland, "Integrity of map-matching algorithms," *Transportation Research Part C: Emerging Technologies*, vol. 14, no. 4, pp. 283–302, Aug. 2006. doi: 10.1016/j.trc.2006.08.004.
5. A. Barua and A. Basu, "Semiparametric Robust Estimation of Population Location," 2025, arXiv. doi: 10.48550/ARXIV.2512.03021.

## 6. 方法与局限（含一处自我更正）

- **更正**：本报告初稿曾把 Quddus 2006 的 DOI 写作 `10.1016/j.trc.2006.05.005`（凭记忆），`paper_citations` 把它解析成了另一篇交通流论文《Macroscopic traffic flow propagation stability…》。改用 `paper_get` 按标题检索得到正确 DOI `10.1016/j.trc.2006.08.004`，并重新生成、核验参考文献。这正是"DOI 只能来自工具、不能来自记忆"的实例。
- **代理**：Google Scholar 与 CNKI 走插件自带浏览器，跟随 Windows 系统代理（Clash mixed 7890），未固化任何环境变量；Node 侧开放索引直连可用。
- **限流**：arXiv 两次 429；Semantic Scholar 首轮 429，带 key 后仍受免费档速率限制，两轮之间需间隔。
- **CNKI**：第一次查询过窄返回 0 条，缩减关键词后返回 5 条；仅题录，未取全文。
- **未做的部分**：未对 Nicosia 与 Yiu 做全文精读（仅摘要 + `paper_get` 元数据）；未追 Quddus 的被引（DOI 更正后未重跑，属下一轮动作）。
