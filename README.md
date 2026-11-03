# dsh-lit-search

Donkey.Ning 的 DSH 文献调研能力打包：一个**学术检索插件**的选择与配置说明，加一个**用户级技能**，再加一份完整的检索报告作为验收样例。

## 组成

| 部分 | 内容 | 来源 |
|---|---|---|
| 检索底座插件 | `@copylee/dsh-academic@0.5.1` | npm（MIT，SLSA provenance） |
| 用户级技能 | `skills/sjtu-lit-search/SKILL.md` | 本仓库作者，改编自 `lit-search-skill` |
| 验收样例 | `reports/lit-search-hmm-unmodeled-2026-11-03.md` | 本仓库作者 |

## 为什么是"插件 + 技能"两层

- **插件**负责“能查到”：19 个工具覆盖 arXiv / OpenAlex / Crossref / Semantic Scholar / PubMed / Europe PMC / DBLP，并把 Google Scholar 与 CNKI 放进真实 Chrome/Edge 窗口检索（无官方 API）；另有引文核验与 Zotero 打通。
- **技能**负责“怎么查”：把原 `lit-search-skill` 的四阶段流程（拆题 → 双主源检索 → 引文追溯 → 筛选精读 → 结构化输出）映射到 DSH 的工具面，并补上交大图书馆的开源缺口。
- **两者都不含需要保密的凭据**：Semantic Scholar / OpenAlex / NCBI 的 key 走 DSH 设置页，落在 DSH 凭据库。

## 安装

1. 安装检索插件（DSH 桌面版：插件 → 添加插件，或命令行）：

   ```bash
   dsh plugin --profile <profile> add @copylee/dsh-academic@latest
   ```

2. 安装用户级技能（所有工作区生效）：

   ```bash
   mkdir -p "$HOME/.dsh/skills/sjtu-lit-search"
   cp skills/sjtu-lit-search/SKILL.md "$HOME/.dsh/skills/sjtu-lit-search/SKILL.md"
   ```

   Windows PowerShell：

   ```powershell
   New-Item -ItemType Directory -Force "$HOME\.dsh\skills\sjtu-lit-search" | Out-Null
   Copy-Item "skills\sjtu-lit-search\SKILL.md" "$HOME\.dsh\skills\sjtu-lit-search\SKILL.md"
   ```

3. 可选：在 DSH 设置 → 学术里填 Semantic Scholar / OpenAlex / NCBI 的 API key（不填也可用，但会受公共限流）。

## 网络前提

- **Google Scholar 与 CNKI**：走插件自带的浏览器窗口，跟随 Windows 系统代理。开 Clash 并把系统代理指向 mixed 端口（如 `127.0.0.1:7890`）即可。
- **不要把 `HTTP_PROXY` / `HTTPS_PROXY` / `NODE_USE_ENV_PROXY` 固化成用户环境变量**：Clash 关闭时会让所有 Node 请求先撞死代理，反而破坏本来直连可用的开放索引。需要时只在单条命令里临时设置。
- **交大图书馆**（`lib.sjtu.edu.cn`）没有公开检索 API，机构资源通过 CARSI/jAccount 登录 + 校园网 IP 访问；技能中给出的是“用浏览器打开数据库导航/OPAC，登录态由用户自己完成，遇验证码交给人”的路径，不绕过任何付费墙。

## 验收样例

`reports/lit-search-hmm-unmodeled-2026-11-03.md` 是一次全管线运行：7 组检索 + 2 次引文追溯，5 条参考文献全部经 `reference_verify` 核验通过，并记录了一处 DOI 自我更正（凭记忆写错 → 用 `paper_get` 按标题找回正确 DOI）。

## 许可

MIT。

## 已知限制

- Google Scholar 与 CNKI 无官方接口，抓取型方案随时可能被限流或要求人机验证；一次一句，不要并发。
- Semantic Scholar 免费档约 1 请求/秒，带 key 也需间隔。
- 出版商全文仅通过用户自己的机构权限获取；本仓库不包含任何绕过付费墙的代码。
