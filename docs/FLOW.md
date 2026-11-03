# sjtu-lit-search 的构成、依赖与检索流程

面向想复用这套能力的人。作者：Donkey.Ning（宁宸章）。最后更新 2026-11-03。

---

## 1. 它由什么组成

`sjtu-lit-search` **本身只是一个技能文件**（`~/.dsh/skills/sjtu-lit-search/SKILL.md`，约 110 行）。它不联网、不含代码、不存密钥，作用是把"怎么查文献"写成 DSH 能自动加载的操作规程。

真正干活的是三样东西：

| 层 | 是什么 | 负责 |
|---|---|---|
| 技能 | `sjtu-lit-search/SKILL.md`（用户级，所有工作区生效） | 拆题、选源、定预算、判停、输出格式、失败处理 |
| 插件 | `@copylee/dsh-academic@0.5.1`（npm，MIT，SLSA provenance） | 19 个工具：多源检索、详情、引文、全文、引文格式、参考文献核验、Zotero |
| 浏览器 | `dsh-browser-cdp`（`bcdp_*` 工具） | 无 API 的源：Google Scholar、CNKI、出版商全文、交大书目 |

## 2. 依赖与前提

- **DSH 桌面版 ≥ 0.2.0-rc.2**（实测 0.2.1-rc.5）；插件 `peerDependencies` 只要求 `cordis` 与 `schemastery`，无构建脚本。
- **Chrome 或 Edge**：Google Scholar 与 CNKI 由插件在独立浏览器窗口中检索（不读取你日常浏览器的 cookie）。
- **网络**：开放索引（OpenAlex/Crossref/PubMed/Europe PMC/DBLP/Semantic Scholar）直连可用；**校园网 IP 即可**访问 CNKI、IEEE、Elsevier、Wiley、Springer（机构订阅按 IP 授权）；**只有 Google Scholar 需要代理**（Clash，系统代理 mixed 端口，如 `127.0.0.1:7890`）。
- **可选密钥**：Semantic Scholar / OpenAlex / NCBI 的 API key 在插件设置页填写，存进 DSH 凭据库；不填也能用，只是限流更紧。
- **不要**把 `HTTP_PROXY` / `NODE_USE_ENV_PROXY` 固化成用户环境变量：Clash 关闭时会让所有 Node 请求先撞死代理，反而拖垮本来直连可用的源。
- **交大图书馆**：`lib.sjtu.edu.cn` 是 JavaScript 应用，无公开检索 API；机构资源靠 CARSI/jAccount 登录或校园 IP。技能规定：遇到登录页或人机验证就交给使用者，不绕过。

## 3. 完整流程（8 步）

1. **预检**：确认哪些源可达、代理是否在、图书馆登录态是否有效。不可达的源要显式记录，不能当"没有这篇"。
2. **拆题**：两层分解——应用域（2–4 个）× 研究挑战（每个域 3–5 个），生成英文与中文两套关键词。
3. **双主源检索**：
   - 结构化索引（OpenAlex + Crossref + Semantic Scholar，按学科加 arXiv/PubMed/Europe PMC/DBLP）——一次 `paper_search` 调用内部并行分发，拿 DOI 与期刊元数据；
   - 相关性补充（Google Scholar，必要时 `sort="citations"`）——串行，一次一句；
   - 中文侧（CNKI）——中文关键词，串行，最多 2 次。
4. **引文追溯**：对 3–5 篇种子做 references / citations / related，最多两轮。
5. **筛选**：按期刊层级、年份、被引筛；按 DOI → arXiv id → PMID → 标题去重；用 `paper_get` 查撤稿与元数据。
6. **精读**：短名单先 `paper_read outline=true`，再读单节或用 `query` 定位段落；付费墙读不了就只据摘要，并说明。
7. **机构全文**：对短名单里被付费墙挡住的论文，用浏览器按 DOI 打开出版商页面（IEEE/Elsevier/Wiley/Springer/ACM）或走图书馆数据库导航；串行，遇到登录/验证码交给人。
8. **产出**：检索日志（源、查询、命中数、失败源）→ 语料表（题名/年份/期刊/层级/被引/是否开放/入选理由）→ 主题综合 → 缺口与候选贡献 → 参考文献（只能由 `paper_cite` 生成，再跑 `reference_verify`）。

## 4. 并行还是串行？——混合，且有明确预算

**不是**"五个渠道全并行然后合并"，**也不是**"搜到一个程度就自己停"。真实规则是：

- **一次 `paper_search` 内部并行**：一个调用同时打多个开放索引，返回合并去重后的记录。这是唯一的"真并行"。
- **渠道之间串行**：Scholar、CNKI 各自开浏览器窗口，慢（10–20 秒/次）、易撞反爬、可能要人过验证，所以一次一句。
- **浏览器任务可有界并发**：子智能体已被实测可以驱动 `bcdp_*`；允许最多 **2 个**并发浏览器会话、不同站点、同站串行；多于此就排队。API 侧不限并发。

停止条件（写死在技能 §2b）：

| 预算 | 上限 |
|---|---|
| 关键词维度 | ≤3 |
| 开放索引调用 | ≤6 |
| Google Scholar 查询 | ≤3（串行） |
| CNKI 查询 | ≤2（串行） |
| arXiv | 每次 ≤1，间隔 ≥3 秒，429 等 60 秒重试一次 |
| 引文轮次 | ≤2；最后一轮新增 <20% 即提前停 |
| 摘要级筛选 | ≤15 篇 |
| 全文精读 | ≤5 篇 |
| 机构全文取回 | ≤5 篇（仅短名单内付费墙论文） |

任一预算耗尽，或连续两步没有新记录，就停止并写报告，同时列出"哪些维度因预算或限流没覆盖"。

## 5. 以"地图匹配"为例

| 步骤 | 具体动作 | 并行/串行 |
|---|---|---|
| 拆题 | 域：城市峡谷 GNSS、车道级地图、室内/停车楼；挑战：候选集缺失、发射模型失配、置信度校准 | — |
| 关键词 | EN: `map matching HMM uncertainty calibration` / `candidate set missing` / `posterior confidence`；ZH: `地图匹配 置信度` / `隐马尔可夫 匹配` | — |
| 结构化索引 | 1 次 `paper_search(sources=[openalex,crossref,semanticscholar])`，必要时加 `dblp`/`arxiv` | 调用内并行 |
| 相关性补充 | 1–3 次 `sources=["googlescholar"]`，逐条 | 串行 |
| 中文侧 | 1–2 次 `sources=["cnki"]`，中文词 | 串行 |
| 引文追溯 | 对 Newson & Krumm 2009 等种子做 citations/references | 串行（每轮一次调用） |
| 筛选+精读 | 去重 → 期刊层级 → `paper_get` 查撤稿 → 最多 5 篇读全文 | 串行 |
| 机构全文 | 对 IEEE/Elsevier 的付费论文按 DOI 开浏览器窗口 | 串行，最多 2 路并发 |
| 产出 | 报告 + 参考文献（`paper_cite` → `reference_verify`） | — |

实测样例见 `reports/lit-search-hmm-unmodeled-2026-11-03.md`：7 组检索 + 2 次引文追溯，5 条参考文献全部核验通过。

## 6. 失败与边界

- **arXiv 429**：突发限流，不是墙。串行 + ≥3 秒间隔 + 退避 60 秒重试一次；三次串行实测均 200。
- **出版商 403/202**：Cloudflare/CloudFront 的机器人检测，**不是没权限**。换真实浏览器 + 校园 IP 即可；技能要求记为"反爬墙，需浏览器"，不得写成"无访问权"。
- **Google Scholar**：无官方 API，抓取型方案随时可能触发人机验证；一次一句，验证交给使用者。
- **CNKI**：只回题录，链接需你本人在浏览器打开；不做无头浏览器绕过。
- **图书馆**：无检索 API，走数据库导航与 OPAC；机构登录态由使用者完成。
- **合规**：只通过开放获取或使用者自身的机构订阅取全文，不包含任何绕过付费墙的代码。
