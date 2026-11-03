# 调研结果的整理、分级与导入（Zotero / 小绿鲸）

设计目标：每轮检索结束后，结果不再停留在对话里，而是自动分级、落盘成可审计的清单，并导入文献管理软件。

## 1. 现状（实测 2026-11-03）

| 组件 | 状态 | 依据 |
|---|---|---|
| `@copylee/dsh-academic` | 已装 0.5.1，含 8 个 `zotero_*` 工具与 `zoteroWrite` 开关 | 插件设置类型定义 |
| Zotero | **本机未安装**（注册表、Program Files、D 盘均无；23119 未监听） | `reg query` + `netstat` |
| 小绿鲸英文文献阅读器 | **已安装**：`D:\Software\小绿鲸\xljsci`，用户数据 `%APPDATA%\xljsci` | 开始菜单快捷方式解析 |
| 小绿鲸的 Zotero 通道 | 其 `resources/app/src/createZoteroData.js` **直接读 `zotero.sqlite`**（WASM SQLite），并把标注转 XFDF | 源码读取 |

结论：**Zotero 是中枢，小绿鲸是阅读端**。把文献写进 Zotero，小绿鲸就能读到；没有 Zotero 时退化为 RIS 手工导入。

## 2. 架构

```
检索（paper_* / bcdp） → 分级（T1/T2/T3） → manifest.json（唯一事实源）
                                             ├── Zotero（zotero_add / organize / note）
                                             │      └── 小绿鲸读取 zotero.sqlite
                                             └── import.ris / import.bib（无 Zotero 时）
```

manifest 先行，其余产物都从它生成：这样重跑可幂等（已导入的 DOI 跳过），也便于人工复核。

## 3. 分级模型

| 级别 | 含义 | 处理 |
|---|---|---|
| **T1 核心** | 直接基线/竞品，必须引用 | 入库 + 笔记写明"为什么必须引" |
| **T2 支撑** | 方法或相邻领域，支撑论证 | 入库 + 标签 |
| **T3 背景** | 背景/综述/拓展阅读 | 只记 manifest 或入库不打笔记 |

评分输入：期刊层级（SCI Q1 / CCF-A / 领域旗舰）、被引数、与问题的直接性、是否开放获取、是否撤稿（`paper_get`）。分级理由必须能一句话说清。

## 4. 写入 Zotero 的做法

前置条件：

1. 安装 Zotero（插件文档要求 7 或更高；**写入功能据其文档需要 Zotero 10 或更高**，以插件文档为准）。
2. Zotero → 设置 → 高级 → 勾选“允许此计算机上的其他应用程序与 Zotero 通讯”。
3. 插件设置 → 学术 → 打开 `zoteroWrite`（默认关闭）。
4. 首次写入时 Zotero 会弹窗确认。

写入顺序：`zotero_add`（按 DOI/arXiv 入库，自动去重）→ `zotero_organize`（建分类 `LitSearch/<topic>-<YYYYMMDD>`，加标签 `lit:T1|T2|T3`、`lit:<topic>`）→ `zotero_note`（写分级理由与入选原因）。

不删除任何条目；失败项留在 manifest 里，报告写“未导入”，不谎报成功。

## 5. 没有 Zotero 时：小绿鲸手工导入

- 用 `paper_cite` 生成 `import.ris`（必要时加 `import.bib`），放到 `lit-imports/<topic>-<YYYYMMDD>/`。
- 开放获取的 PDF 用 `paper_download` 落到同目录 `papers/`，便于整包导入。
- 小绿鲸对 RIS/BibTeX 的 UI 入口以实际版本为准；`%APPDATA%\xljsci` 是它的用户数据目录。
- 中文题录（CNKI 只有题录、无 DOI）可能无法自动匹配，按标题手工补齐。

## 6. 目录与产物

```
lit-imports/<topic>-<YYYYMMDD>/
├── manifest.json      # DOI/arXiv、题名、年份、期刊、T1-T3、标签、理由、OA 链接
├── manifest.csv       # 供 Excel / 人工复核
├── import.ris         # 无 Zotero 时给 小绿鲸
├── import.bib         # 同上（可选）
└── papers/            # 开放获取 PDF（可选）
```

## 7. 插件 vs 技能

**先扩技能**（本方案）：`@copylee` 已经提供 Zotero 的增/标签/分类/笔记工具，缺的只是“分级 + manifest + 幂等 + 导入编排”这套流程，写进 `sjtu-lit-search` 即可，零新代码。

**何时才需要新插件**：需要无人值守（定时跑、检索完自动导入）、或需要读取 小绿鲸 的 `zotero.sqlite` 反查已读状态时。那时写一个 host 插件：监听 manifest → 调 Zotero 本地 API（`http://127.0.0.1:23119/api/users/0/...`，带 `Zotero-Allowed-Request` 头）→ 生成 RIS；仍不直接改 小绿鲸 数据库（避免破坏其数据）。

## 8. 管线验证记录（2026-11-03，实测通过）

环境：Zotero 安装在 `D:\ProgramData\zotero`，profile `%APPDATA%\Zotero\Zotero\Profiles\2o4cxx4d.default\prefs.js`；插件 `@copylee/dsh-academic 0.5.1`。

| 步骤 | 结果 |
|---|---|
| 本地 API | 初始 `403 Local API is not enabled`；写入 `extensions.zotero.httpServer.localAPI.enabled=true` 并重启后 `GET /api/users/0/items` 返回 `200 []` |
| 建分类 | `LitSearch`（`zotero://user/0/collection/TI3S5UFG`）→ 子分类 `map-matching-unmodeled-states-20261103`（`JSIBF6QV`，5 items） |
| 入库 | 5 篇全部 added（2 期刊、1 会议、2 预印本），DOI 未重复 |
| 分级标签 | `lit:T1` ×3、`lit:T2` ×1、`lit:T3` ×1，另一个公共标签 `lit:map-matching`、`lit:2026-11` |
| 笔记 | 在 Nicosia 条目下建分级理由笔记 `4FGBPI48` |
| 回读 | `zotero_search tags=["lit:T1"]` 返回 3 篇，DOI/标签一致 |

结论：**技能 §7 描述的管线可运行**。剩余一步是 小绿鲸 侧的读取：它通过 `createZoteroData.js` 读 `zotero.sqlite`，需在 小绿鲸 UI 里触发“从 Zotero 导入”；本仓库不改它的数据库。

回滚方式：删除分类 `LitSearch / map-matching-unmodeled-states-20261103` 即可清除本次 5 条测试记录；Zotero prefs 的备份在同目录 `prefs.js.bak-20261103`。



## 9. 风险与边界

- Zotero 未安装时整条链不成立：先装再谈自动入库。
- 插件文档里的“Zotero 10”与实际版本号可能不一致，安装前先核对；若写入不可用，退回 RIS。
- 不写入密码、token；不删除条目；不修改 小绿鲸 的数据库文件。
- 入库是写操作：每次运行前确认目标分类名与标签命名，避免污染既有文库。
