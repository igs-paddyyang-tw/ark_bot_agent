# CHANGELOG

所有版本的重要變更記錄於此，格式依 [Keep a Changelog](https://keepachangelog.com/zh-TW/1.0.0/)。

版本號遵循 [Semantic Versioning](https://semver.org/lang/zh-TW/)：`MAJOR.MINOR.PATCH`。
`0.x` 期間允許 breaking change，`1.0` 之後遵守 semver。

> ⚠️ **原始碼不在本 repo** —— 開發源是 `paddy-team-agent/src/ark_bot_agent/`。

---

## [0.2.0] — 2026-08-17

**首次發版。** M0–M4 完成：路徑層、設定層、三模式路由、報告管線。
`run_bot()` / `create_bot()` 仍是 stub —— M5（消費端化）才接上啟動流程。

wheel：113 個 `.py`，18 條依賴，兩個 extras（`search` / `skills`）。

### 端到端驗證（不是只看 import）

在乾淨 venv 裝 wheel + 建一個只有 `agents.yaml` / `bot.yaml` 的假消費端：

| 驗證 | 結果 |
|------|------|
| `get_home()` 定位到**消費端目錄**而非 site-packages | ✅ |
| 讀到消費端設定（`port: 9001`、`default: team`） | ✅ |
| `group_members()` / `resolve_agent("碼哥")` | ✅ |
| 報告管線產出 MD + HTML | ✅ |
| `ark-bot paths` CLI | ✅ |

### 主要內容

| 模組 | 說明 |
|------|------|
| `paths.py` | 路徑解析唯一出口。哨兵搜尋 `agents.yaml`，14 個衍生出口 |
| `config.py` | `agents.yaml` + `bot.yaml` → 10 個 dataclass。**agent 查詢唯一來源** |
| `bot/router.py` | `ModeRouter` 三模式路由，可注入。`MessageContext` 不綁 `telegram.Update` |
| `report/` | MD-first 管線（三模式共用） |
| `wiki/` `memory/` `llm/` `agent/` `skills/` `coordinator/` `server/` `bot/` | 自 nana-bot 平移，107 檔 |

### 平移時修掉的三個**靜默失效**

這三個都不拋例外、不寫 log，只讓行為安靜地錯：

1. **BM25 索引永遠讀不到** —— `layer1_bm25` 讀 `knowledge/.index`、
   `indexer` 寫 `knowledge/shared/.index`。搜尋一直 fallback 到 layer0。
2. **agent 的 `working_dir` 相對 cwd** —— `Path(info["dir"])` 從別的目錄啟動
   就找不到 agent 工作區，會安靜 fallback 到 `Path(".")`，
   agent 在錯誤目錄啟動（讀不到自己的 SOUL/skills/memory）。
3. **`TEMPLATES_DIR` 指到 site-packages** —— 消費端放的自訂模板永遠讀不到。

### 設計要點

- **路徑一律走 `paths.py`**：平移時消除 24 處 `parents[N]`（N 值 2/3 混用）
  與 37 處硬編相對路徑。套件安裝後 `__file__` 在 site-packages，數層數必然錯**且不報錯**。
- **設定檔分開**：`agents.yaml`（誰）／`bot.yaml`（怎麼跑）／`scheduler.yaml`（何時）。
  只有哨兵必要，**缺 `bot.yaml` 不該讓服務起不來**。
- **未知欄位忽略不報錯**：舊 yaml 留著已移除的欄位時服務仍要能起來。
- **報告失敗策略**：只有 MD 寫入失敗才拋（它是 source of truth）；
  HTML 與送檔失敗只反映在 `ReportResult`，並由 `user_note()` **把降級講出來**。
- **模式資訊必須寫進 prompt**：agent 只讀最終 prompt，看不到設定檔。
  但 👑 管家模式**刻意不注入** —— 它沒有替代行為，注入只是干擾。

### 發版機制

`packages/ark-bot-agent/pyproject.toml` 用 `package-dir` 指回 `../../src/`
（paddy 的 `src/` 下有多個套件，一個 pyproject 只能產一個 wheel）。

🔴 **只發 wheel，不要建 sdist** —— `--sdist` 會產生空殼且回報成功。

⚠️ **`readme` 不能指向 `../../README.md`** —— setuptools 的 `_assert_local`
拒絕存取 pyproject 目錄外的檔案。`package-dir` 可以跨目錄，`readme` 不行。

---

## [0.1.0] — 未發布

### M1 —— 骨架與路徑層

`paddy-team-agent/src/ark_bot_agent/` 已建立，19 個測試通過，但 `run_bot()`
與 `create_bot()` 還是 stub。**M3 才會有可用的 Bot。**

#### `paths.py` —— 路徑解析的唯一出口

套件化最大的技術障礙不是搬檔案，是路徑。實測 nana-bot 的 `src/` 有：

| 類別 | 處數 |
|------|:----:|
| 🔴 `Path(__file__).resolve().parents[N]` 定位專案根 | **24** |
| 🟠 硬編相對路徑（`Path("knowledge")` 等，依賴 cwd） | **37** |
| 🟡 環境變數當設定（應走設定檔） | **19** |

兩類路徑寫法在套件安裝後都會壞，而且**都不拋例外** ——
它們安靜地回傳錯路徑，下游變成「wiki 0 篇」「memory 讀不到」這種難查的症狀。
`parents[N]` 的 N 值還不一致（2 和 3 混用），本身就證明那寫法依賴檔案層數，
而套件化必然改變層數。

修法不是修數字，是**哨兵搜尋**：

```python
get_home()   # ARK_BOT_HOME → 往上找 agents.yaml → cwd
```

**哨兵用 `agents.yaml` 而不是新造 `bot.yaml`** —— 它本來就存在（agent 定義的
唯一來源），消費端不必為套件化多一個檔案；bot 層設定（server/modes/llm）也併進
同一份，一份 yaml 管完。

衍生出口全部集中在 `paths.py`（`get_state_dir` / `get_knowledge_dir` /
`get_agents_dir` / `get_steering` / `get_templates_dir`…），
其他模組**不自己組路徑**。

兩個刻意的設計選擇：

- `get_steering()` **不自動建目錄** —— 缺人格檔要讓呼叫端知道，
  自動建空目錄會讓「讀不到人格」看起來像「檔案是空的」
- `get_templates_dir()` **消費端優先** —— 要客製 Web UI 只需放同名檔案，不必 fork

#### `scripts/check_package_paths.py` —— CI 守門

掃 A/B/C 三類寫法，`--strict` 命中即 exit 1。

用 `ast` 精準排除 docstring 與多行字串 —— 因為說明文件常在 docstring 裡示範
「不該這樣寫」（`paths.py` 自己就有一張這種表格）。純文字比對會把說明當違規，
而**常駐假警報的代價不是雜訊，是維運開始習慣性忽略這個檢查**，等於把它廢掉。
