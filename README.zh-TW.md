# graphify

[English](README.md) | [繁體中文](README.zh-TW.md) | [日本語](README.ja-JP.md) | [한국어](README.ko-KR.md)

[![CI](https://github.com/safishamsi/graphify/actions/workflows/ci.yml/badge.svg?branch=v3)](https://github.com/safishamsi/graphify/actions/workflows/ci.yml)
[![PyPI](https://img.shields.io/pypi/v/graphifyy)](https://pypi.org/project/graphifyy/)

**一個面向 AI 編碼助手的技能。** 在 Claude Code、Codex、OpenCode、OpenClaw、Factory Droid 或 Trae 中輸入 `/graphify`，它會讀取你的檔案、構建知識圖譜，並把原本不明顯的結構關係還給你。更快理解程式碼庫，找到架構決策背後的"為什麼"。

完全多模態。你可以直接丟進去程式碼、PDF、Markdown、截圖、流程圖、白板照片，甚至其他語言的圖片 —— graphify 會用 Claude vision 從這些內容中提取概念和關係，並把它們連線到同一張圖裡。

> Andrej Karpathy 會維護一個 `/raw` 資料夾，把論文、推文、截圖和筆記都丟進去。graphify 就是在解決這類問題 —— 相比直接讀取原始檔案，每次查詢的 token 消耗可降低 **71.5 倍**，結果還能跨會話持久儲存，並且會明確區分哪些內容是實際發現的，哪些只是合理推斷。

```
/graphify .                        # 可用於任意目錄：程式碼庫、筆記、論文都可以
```

```
graphify-out/
├── graph.html       可互動圖譜：可點節點、搜尋、按社群過濾
├── GRAPH_REPORT.md  God nodes、意外連線、建議提問
├── graph.json       持久化圖譜：數週後仍可查詢，無需重新讀原始檔案
└── cache/           SHA256 快取：重複執行時只處理變更過的檔案
```

## 工作原理

graphify 分兩輪執行。第一輪是確定性的 AST 提取，對程式碼檔案做結構分析（類、函式、匯入、呼叫圖、docstring、解釋性註釋），這一輪不需要 LLM。第二輪會並行呼叫 Claude 子代理處理文件、論文和圖片，從中提取概念、關係和設計動機。最後把兩邊結果合併到一個 NetworkX 圖裡，用 Leiden 社群發現演算法做聚類，並匯出成可互動 HTML、可查詢 JSON，以及一份人類可讀的審計報告。

**聚類是基於圖拓撲完成的，不依賴 embeddings。** Leiden 按邊密度發現社群。Claude 抽取出的語義相似邊（`semantically_similar_to`，標記為 `INFERRED`）本來就存在於圖中，所以會直接影響社群劃分。圖結構本身就是相似性訊號，不需要額外的 embedding 步驟，也不需要向量資料庫。

每條關係都會被標記為 `EXTRACTED`（直接在源材料中找到）、`INFERRED`（合理推斷，並附帶置信度分數）或 `AMBIGUOUS`（有歧義，需要複核）。所以你始終知道哪些是實際發現的，哪些是模型猜出來的。

## 安裝

**要求：** Python 3.10+，並且使用以下平臺之一：[Claude Code](https://claude.ai/code)、[Codex](https://openai.com/codex)、[OpenCode](https://opencode.ai)、[OpenClaw](https://openclaw.ai)、[Factory Droid](https://factory.ai) 或 [Trae](https://trae.ai)

```bash
pip install graphifyy && graphify install
```

> PyPI 包當前暫時叫 `graphifyy`，因為 `graphify` 這個名字還在回收中。CLI 命令和 skill 命令仍然都是 `graphify`。

### 平臺支援

| 平臺 | 安裝命令 |
|------|----------|
| Claude Code | `graphify install` |
| Codex | `graphify install --platform codex` |
| OpenCode | `graphify install --platform opencode` |
| OpenClaw | `graphify install --platform claw` |
| Factory Droid | `graphify install --platform droid` |
| Trae | `graphify install --platform trae` |
| Trae CN | `graphify install --platform trae-cn` |

Codex 使用者還需要在 `~/.codex/config.toml` 的 `[features]` 下開啟 `multi_agent = true`，這樣才能並行提取。OpenClaw 目前的並行 agent 支援還比較早期，所以使用順序提取。Trae 使用 Agent 工具進行並行子代理排程，**不支援** PreToolUse hook，因此 AGENTS.md 是其常駐機制。

然後開啟你的 AI 編碼助手，輸入：

```
/graphify .
```

### 讓助手始終優先使用圖譜（推薦）

圖構建完成後，在專案裡執行一次：

| 平臺 | 命令 |
|------|------|
| Claude Code | `graphify claude install` |
| Codex | `graphify codex install` |
| OpenCode | `graphify opencode install` |
| OpenClaw | `graphify claw install` |
| Factory Droid | `graphify droid install` |
| Trae | `graphify trae install` |
| Trae CN | `graphify trae-cn install` |

**Claude Code** 會做兩件事：
1. 在 `CLAUDE.md` 中寫入一段規則，告訴 Claude 在回答架構問題前先讀 `graphify-out/GRAPH_REPORT.md`
2. 安裝一個 **PreToolUse hook**（寫入 `settings.json`），在每次 `Glob` 和 `Grep` 前觸發

如果知識圖譜存在，Claude 會先看到：_"graphify: Knowledge graph exists. Read graphify-out/GRAPH_REPORT.md for god nodes and community structure before searching raw files."_ —— 這樣 Claude 會優先按圖譜導航，而不是一上來就 grep 整個專案。

**Codex** 會把規則寫入 `AGENTS.md`，並在 `.codex/hooks.json` 安裝 **PreToolUse hook**，在每次 Bash 工具呼叫前觸發，機制和 Claude Code 類似。

**OpenCode** 會把規則寫入 `AGENTS.md`，並安裝 **`tool.execute.before` plugin**（`.opencode/plugins/graphify.js` + `opencode.json` 註冊）；當知識圖譜存在時，它會在 bash 工具輸出前注入 graph reminder。

**OpenClaw、Factory Droid、Trae** 會把同樣的規則寫進專案根目錄的 `AGENTS.md`。這些平臺不支援 tool hook，所以 `AGENTS.md` 是它們的常駐機制。

🦀🦀🦀

### Crabyard + graphify

如果專案看起來像一個 [Crabyard](https://github.com/conscientiousness/crabyard) repo（例如存在 `crabyard/manifest.yaml`），`graphify * install` 會自動在產生的 `AGENTS.md` 或 `CLAUDE.md` 中加入更嚴格的 truth hierarchy：使用 graphify 來做架構導覽，但在實際採取行動前，仍以 `crabyard/specs/`、相關的 `crabyard/changes/<slug>/` bundle，以及 `crabyard/knowledge/` 作為對照與驗證來源。這只是偵測式整合。graphify 不需要依賴 Crabyard，Crabyard 也不需要知道 graphify 的存在。

### Crabyard repo mode

當你在包含 `crabyard/manifest.yaml` 的 repo root 執行 graphify 時，graphify 會在 detection 階段自動啟用 Crabyard repo mode：

- active change bundle 檔案優先排序
- `crabyard/specs/` 次之
- `crabyard/knowledge/` 再其次
- `.yaml` 與 `.yml` 會被視為 document，所以 `execution.yaml` 與 manifest 類檔案也可以進圖

這不會排除 repo 其餘部分。它只是調整 detection 順序與 extraction focus，讓最早進入圖譜的上下文更貼近目前的 Crabyard 工作。

建議和 Crabyard 搭配的循環：

1. 在 repo root 執行 `graphify . --wiki`
2. 先從 `graphify-out/GRAPH_REPORT.md` 或 `graphify-out/wiki/index.md` 開始做架構導覽
3. 進入當前工作時，再把 graph 給你的結論對照 `crabyard/changes/<slug>/`、`crabyard/specs/` 與 `crabyard/knowledge/`
4. 規劃、狀態、驗證、同步與封存仍交給 Crabyard
5. 在重要的程式碼或 spec 變更之後重新執行 graphify，讓下一次 session 仍然有可用的圖譜

實戰 workflow：

1. `research` 或 `explore`
   先從 `graphify-out/GRAPH_REPORT.md` 或 `graphify-out/wiki/index.md` 開始，理解目前架構、可能的 god nodes，以及跨模組依賴，再去讀 raw files。
2. `plan`
   當 graph 已經幫你縮小搜尋面之後，回到相關的 `crabyard/changes/<slug>/` bundle 與 `crabyard/specs/`。graphify 用來做導覽，Crabyard artifacts 才是規劃時的 truth。
3. `apply`
   依照 Crabyard 的 execution plan 實作。如果變更仍然局部，且 graph 還沒有失真，就繼續往前；如果這次修改重塑了架構、共享介面或 accepted specs，就在下一個重要決策點前更新 graph。
4. `review` 或 `debug`
   再次使用 graphify 觀察 impact radius、community boundaries 與不明顯的鄰近模組。之後再回到 code、tests、staged specs 與 Crabyard knowledge notes 驗證具體正確性。
5. `verify`、`sync`、`archive`
   這些仍然是 Crabyard 的責任。graphify 幫你更快理解 repo，但不取代 execution truth、verification gates 或 accepted-truth sync。

什麼時候該 refresh graphify outputs：

- 當架構、介面、`crabyard/specs/`、`crabyard/knowledge/` 或 active change bundle 有明顯變更時，就更新。
- 當你要開始新的深度 `explore`、`review` 或 `debug` session，而上一份 graph 已經過時時，就更新。
- 不要把 `graphify-out/wiki/` 當成手動維護的文件。wiki 是 graph 的衍生視圖。
- 不需要每次很小的 code edit 都重跑 `--wiki`。真正重要的是在決策點之前，讓 graph 保持合理新鮮，而不是每次儲存都重建所有衍生輸出。

🦀🦀🦀

解除安裝時使用對應平臺的 uninstall 命令即可（例如 `graphify claude uninstall`）。

**常駐模式和顯式觸發有什麼區別？**

常駐 hook 會優先暴露 `GRAPH_REPORT.md` —— 這是一頁式總結，包含 god nodes、社群結構和意外連線。你的助手在搜尋檔案前會先讀它，因此會按結構導航，而不是按關鍵字亂搜。這已經能覆蓋大部分日常問題。

`/graphify query`、`/graphify path` 和 `/graphify explain` 會更深入：它們會逐跳遍歷底層 `graph.json`，追蹤節點之間的精確路徑，並展示邊級別細節（關係型別、置信度、源位置）。當你想從圖譜裡精確回答某個問題，而不僅僅是獲得整體感知時，就該用這些命令。

可以這樣理解：常駐 hook 是先給助手一張地圖，`/graphify` 這幾個命令則是讓它沿著地圖精確導航。

<details>
<summary>手動安裝（curl）</summary>

```bash
mkdir -p ~/.claude/skills/graphify
curl -fsSL https://raw.githubusercontent.com/safishamsi/graphify/v3/graphify/skill.md \
  > ~/.claude/skills/graphify/SKILL.md
```

把下面內容加到 `~/.claude/CLAUDE.md`：

```
- **graphify** (`~/.claude/skills/graphify/SKILL.md`) - any input to knowledge graph. Trigger: `/graphify`
When the user types `/graphify`, invoke the Skill tool with `skill: "graphify"` before doing anything else.
```

</details>

## 用法

```
/graphify                          # 對當前目錄執行
/graphify ./raw                    # 對指定目錄執行
/graphify ./raw --mode deep        # 更激進地抽取 INFERRED 邊
/graphify ./raw --update           # 只重新提取變更檔案，併合併到已有圖譜
/graphify ./raw --cluster-only     # 只重新聚類已有圖譜，不重新提取
/graphify ./raw --no-viz           # 跳過 HTML，只生成 report + JSON
/graphify ./raw --obsidian         # 額外生成 Obsidian vault（可選）

/graphify add https://arxiv.org/abs/1706.03762        # 拉取論文、儲存並更新圖譜
/graphify add https://x.com/karpathy/status/...       # 拉取推文
/graphify add https://... --author "Name"             # 標記原作者
/graphify add https://... --contributor "Name"        # 標記是誰把它加入語料庫的

/graphify query "what connects attention to the optimizer?"
/graphify query "what connects attention to the optimizer?" --dfs   # 追蹤一條具體路徑
/graphify query "what connects attention to the optimizer?" --budget 1500  # 把預算限制在 N tokens
/graphify path "DigestAuth" "Response"
/graphify explain "SwinTransformer"

/graphify ./raw --watch            # 檔案變更時自動同步圖譜（程式碼：立即更新；文件：提醒你）
/graphify ./raw --wiki             # 構建可供 agent 抓取的 wiki（index.md + 每個 community 一篇文章）
/graphify ./raw --svg              # 匯出 graph.svg
/graphify ./raw --graphml          # 匯出 graph.graphml（Gephi、yEd）
/graphify ./raw --neo4j            # 生成給 Neo4j 用的 cypher.txt
/graphify ./raw --neo4j-push bolt://localhost:7687    # 直接推送到執行中的 Neo4j
/graphify ./raw --mcp              # 啟動 MCP stdio server

# git hooks - 跨平臺，在 commit 和切分支後重建圖譜
graphify hook install
graphify hook uninstall
graphify hook status

# 常駐助手規則 - 按平臺區分
graphify claude install            # CLAUDE.md + PreToolUse hook（Claude Code）
graphify claude uninstall
graphify codex install             # AGENTS.md（Codex）
graphify opencode install          # AGENTS.md（OpenCode）
graphify claw install              # AGENTS.md（OpenClaw）
graphify droid install             # AGENTS.md（Factory Droid）
graphify trae install              # AGENTS.md（Trae）
graphify trae uninstall
graphify trae-cn install           # AGENTS.md（Trae CN）
graphify trae-cn uninstall
```

支援混合檔案型別：

| 型別 | 副檔名 | 提取方式 |
|------|--------|----------|
| 程式碼 | `.py .ts .js .go .rs .java .c .cpp .rb .cs .kt .scala .php` | tree-sitter AST + 呼叫圖 + docstring / 註釋中的 rationale |
| 文件 | `.md .txt .rst .yaml .yml` | 透過 Claude 提取概念、關係和設計動機 |
| 論文 | `.pdf` | 引文挖掘 + 概念提取 |
| 圖片 | `.png .jpg .webp .gif` | Claude vision —— 截圖、圖表、任意語言都可以 |

## 你會得到什麼

**God nodes** —— 度最高的概念節點（整個系統最容易匯聚到的地方）

**意外連線** —— 按綜合得分排序。程式碼-論文之間的邊會比程式碼-程式碼邊權重更高。每條結果都會附帶一段人話解釋。

**建議提問** —— 圖譜特別擅長回答的 4 到 5 個問題。

**“為什麼”** —— docstring、行內註釋（`# NOTE:`、`# IMPORTANT:`、`# HACK:`、`# WHY:`）以及文件裡的設計動機都會被抽取成 `rationale_for` 節點。不只是知道程式碼“做了什麼”，還能知道“為什麼要這麼寫”。

**置信度分數** —— 每條 `INFERRED` 邊都有 `confidence_score`（0.0-1.0）。你不只知道哪些是猜出來的，還知道模型對這個猜測有多有把握。`EXTRACTED` 邊恆為 1.0。

**語義相似邊** —— 跨檔案的概念連線，即使結構上沒有直接依賴也能建立關聯。比如兩個函式做的是同一類問題但彼此沒有呼叫，或者某個程式碼類和某篇論文裡的演算法概念本質相同。

**超邊（Hyperedges）** —— 用來表達 3 個以上節點的群組關係，這是普通兩兩邊表達不出來的。比如：一組類共同實現一個協議、認證鏈路裡的一組函式、同一篇論文某一節裡的多個概念共同組成一個想法。

**Token 基準** —— 每次執行後都會自動列印。對混合語料（Karpathy 的倉庫 + 論文 + 圖片），每次查詢的 token 消耗可以比直接讀原檔案少 **71.5 倍**。第一次執行需要先提取並建圖，這一步會花 token；後續查詢直接讀取壓縮後的圖譜，節省會越來越明顯。SHA256 快取保證重複執行時只重新處理變更檔案。

**自動同步**（`--watch`）—— 在後臺終端裡跑著，程式碼庫一變化，圖譜就會跟著更新。程式碼檔案儲存會立刻觸發重建（只走 AST，不用 LLM）；文件/圖片變更則會提醒你跑 `--update` 進行 LLM 再提取。

**Git hooks**（`graphify hook install`）—— 安裝 `post-commit` 和 `post-checkout` hook。每次 commit 後、每次切分支後都會自動重建圖譜，不需要額外開一個後臺程序。

**Wiki**（`--wiki`）—— 為每個 community 和 god node 生成類似維基百科的 Markdown 文章，並提供 `index.md` 作為入口。任何 agent 只要讀 `index.md`，就能透過普通檔案導航整個知識庫，而不必直接解析 JSON。

## Worked examples

| 語料 | 檔案數 | 壓縮比 | 輸出 |
|------|--------|--------|------|
| Karpathy 的倉庫 + 5 篇論文 + 4 張圖片 | 52 | **71.5x** | [`worked/karpathy-repos/`](worked/karpathy-repos/) |
| graphify 原始碼 + Transformer 論文 | 4 | **5.4x** | [`worked/mixed-corpus/`](worked/mixed-corpus/) |
| httpx（合成 Python 庫） | 6 | ~1x | [`worked/httpx/`](worked/httpx/) |

Token 壓縮效果會隨著語料規模增大而更明顯。6 個檔案本來就塞得進上下文視窗，所以 graphify 在這種場景裡的價值更多是結構清晰度，而不是 token 壓縮。到了 52 個檔案（程式碼 + 論文 + 圖片）這種規模，就能做到 71x+。每個 `worked/` 目錄裡都帶了原始輸入和真實輸出（`GRAPH_REPORT.md`、`graph.json`），你可以自己跑一遍核對數字。

## 隱私

graphify 會把文件、論文和圖片的內容傳送給你所用 AI 編碼助手背後的模型 API 來做語義提取 —— 可能是 Anthropic（Claude Code）、OpenAI（Codex），或者你當前平臺使用的其他提供方。程式碼檔案則完全在本地透過 tree-sitter AST 處理，不會把程式碼內容發出去。專案本身沒有任何遙測、使用跟蹤或分析。唯一的網路請求就是語義提取階段呼叫你平臺自己的模型 API，使用的也是你自己的 API key。

## 技術棧

NetworkX + Leiden（graspologic）+ tree-sitter + vis.js。語義提取由 Claude（Claude Code）、GPT-4（Codex）或你當前平臺所執行的模型完成。不需要 Neo4j，不需要 server，整體是純本地執行。

<details>
<summary>貢獻</summary>

**Worked examples** 是最能建立信任的貢獻方式。對一個真實語料跑 `/graphify`，把輸出儲存到 `worked/{slug}/`，再寫一份誠實的 `review.md`，評價圖譜哪些地方做得對、哪些地方做得不對，然後提交 PR。

**提取 bug** —— 提 issue 時請附上輸入檔案、對應的快取項（`graphify-out/cache/`）以及它漏提取或瞎編了什麼。

模組職責和新增語言的方法見 [ARCHITECTURE.md](ARCHITECTURE.md)。

</details>
