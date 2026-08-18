# Changelog

> **日期以實際發生為準**（GitHub Release `published_at` / 本機 wheel 產出時間），
> 不是推估值。M0–M6 的套件化是 **2026-08-17 一天內**走完的，
> 所以 0.2.0 ~ 0.4.0 全部同日。
>
> ⚠️ **有 GitHub Release 的是 `v0.2.0` / `v0.4.1` / `v0.4.3`**（tag 三個）。
> **`0.3.0` / `0.3.1` / `0.4.0` / `0.4.2` / `0.5.0` / `0.5.1` 只有本機 wheel**，沒有發布 ——
> 讀這份的人會預設每一條都拿得到 asset，所以要講明。
> `v0.4.3` 的 release notes **同時涵蓋 0.4.2**（0.4.2 未單獨發版）。
>
> nana-bot 實裝 **0.4.3**（sha256 與 release asset 逐位元組核對一致）。

## [0.5.1] — 2026-08-18

**AgentPool 四缺口 + per-agent timeout**（M2）。純增益，無破壞性。

四個缺口都是「原本沒有」而不是「原本有別的值」：

| 缺口 | 原本 | 風險 |
|------|------|------|
| queue 上限 | `asyncio.Queue()` **無上限** | 無背壓，只會無聲堆積 |
| idle 回收 | 無 | 8 個常駐 kiro-cli 永不釋放 |
| health loop | 無 | 進程死了沒人知道 |
| crash 計數 | 無 | 崩潰迴圈偵測不到 |

> 不是新設計 —— `ark_team_agent` 的 lazy spawn + `idle_timeout_minutes`
> 在 slot 上驗證過（13 instances、閒置 30 分回收）。
> **兩個套件不共用程式碼，但該共用經驗。**

### Added

- `AgentProcess.idle_seconds` / `crash_count` / `max_queue`
- `agent.cli._health_loop()` —— 崩潰偵測 + idle 回收；`pool_status()` 給 API 用
- `backend.max_queue` / `backend.idle_timeout` / `backend.crash_window`（`bot.yaml` 驅動）
- `AgentDef.timeout` —— per-agent 逾時。實測 `data` 37s / `report` 50s /
  **`ai-dev` 123s**，全體共用一個值必然有一邊不對。
  解析順序：呼叫端 → per-agent → 全域；**未設定時 120s，與 0.4.3 完全同值**

### 三個關鍵決策

1. **queue 溢出丟最舊的，且被丟的那則必須收到結果** ——
   最新的是使用者剛送的，丟它使用者立刻感覺到沒反應。
   🔴 但被丟的 future 不能懸掛 —— 那是永不完成的 await，**比丟掉更糟**
2. **crash 用時間窗不用累計** —— 累計值會讓長時間運行的健康服務
   慢慢累積到誤觸上限（`ark_team_agent` 1.2.14 的教訓）
3. **health loop 出錯不停迴圈** —— 停掉等於回收與偵測都沒了，而且無聲

### 最重要的一條測試

`test_health_loop_recycles_idle_and_keeps_persistent` 測**整條路**：
閒置 → 回收 → **還叫得回來**。

> slot 踩過的坑是 `auto_start: false` 讓進程**連 lazy spawn 都叫不起來**
> （`send_message` 回 `ok:false`，instance 從團隊消失）。
> 回收機制不得重現那個形狀。

全庫 **1470 passed** / 6 skipped。

---

## [0.5.0] — 2026-08-18

**`CliResult` + 錯誤分類**（M1）。🔴 **破壞性**：`agent_cli_chat()` 回傳型別改變。

補的是「失敗看不出來」。舊回傳 `str | None`，呼叫端分不出四種情況，
而**失敗是用回傳值表示的，`try/except` 抓不到**：

| 實際發生 | 舊回傳 |
|---|---|
| agent 回了空字串 | `None` |
| 子進程崩潰 | `None` |
| 逾時 | `None` |
| CLI 根本沒安裝 | `None` |

前三種該重試，第四種重試一百次也不會好（那是設定問題）。

### Added

- `agent/result.py` —— `CliResult`（`status` / `output` / `error` / `retryable` /
  `elapsed_ms` / `backend` / `model`）。`partial` 用於「逾時但已有部分輸出」
- `agent/errors.py` —— `classify()` / `is_retryable()`。
  **`not_found` 與 `permission` 不可重試**
- `backend.timeout` / `backend.retries`（**retries 預設 0** ——
  0.4.3 沒有重試，開新行為要由設定明示，不能因升版就默默多花時間）

### 三個刻意的決策

1. **不叫 `AgentResult`** —— `llm/agent_loop.py` 已有同名類別，
   撞名會讓 `import` 拿到哪個變成靠順序決定
2. **不讓它當字串用** —— 不實作 `__str__` 回 output，漏改的呼叫點要立刻
   `TypeError` 而不是靜默送出物件字串。
   （`dataclass` 的 `__repr__` 含欄位值是**刻意保留**的，log 需要）
3. **分類順序：例外先判、`returncode` 最後** ——
   **逾時被 kill 的子進程也有非 0 returncode**，先看 returncode
   會把逾時誤分類成 crash，而兩者處置不同（逾時調 timeout、crash 查 agent）

### Changed

- 8 個呼叫點全部更新（`router` 1 · `team_flow` 3 · `handlers` 3 · `dispatch` 1）。
  另有 `test_all_call_sites_handle_cliresult()` 用 `ast` 掃描守門
- team_flow 的成員失敗現在**記得出是逾時還是崩潰**（先前一律回空字串）

### 遷移

舊呼叫端把 `reply` 改成 `result.reply_text()` 即語意等價（`None` → `""`），
判斷式（`if reply:`）不用動。

> ⚠️ **測試 stub 也要照契約回 `CliResult`** ——
> stub 比實作寬鬆就等於測不到型別改變（本次有 5 個整合測試因此先失敗）。

全庫 1425 → **1457 passed**。

---

## [0.4.3] — 2026-08-18

**`modes.chat.tools` 從裝飾變成真的白名單。** M5.6 遺留的最後一條。

### Fixed

- 🔴 **`modes.chat.tools` 沒有任何人讀** —— `agent_loop` 拿 `reg.all_schemas()`，
  七個工具全開；而設定只列四個，其中 `wiki_search` 這個名字**在系統裡不存在**
  （實際叫 `search_wiki`）。

  比「設定沒生效」嚴重的是，多出來的三個**有寫入副作用**：

  | 工具 | 副作用 |
  |---|---|
  | `save_to_wiki` | `filepath.write_text()` 直接寫共用知識庫 |
  | `save_memory` | 寫記憶 |
  | `execute_skill` | 執行 skill（含 `execute_code`） |

  讀設定的人會以為快答模式只能查不能寫。**它給人錯誤的安全感** ——
  這比沒有這個設定更糟。

### Changed

- `agent_loop(allowed_tools=...)` —— `None` = 全部（agent 模式的 Gemini fallback
  靠這個，職人本來就該能用寫入工具）；💬 快答的三個呼叫點
  （`router.run_chat` / handlers Path 3 / `/api/v1/chat`）傳 `modes.chat.tools`。
- **預設白名單只留唯讀 + 派工**：`search_wiki` / `recall_memory` /
  `web_search` / `dispatch_to_agent`。三個寫入工具改成 **opt-in** ——
  專案的知識庫規則是「wiki 只由 ingest 產出、禁止手寫」，
  讓對話模型自己寫進去與那條規則衝突，所以預設關不是預設開。
- `_pick_schemas()` 三種可見性：未知名稱 **warning 並列出可用名稱**、
  排除了哪些用 info 講、全部被過濾掉時 **ERROR 並退回全部**
  （安靜地變成無工具模式比報錯更糟）。

### ⚠️ 白名單是範圍控制，不是安全邊界

實測：叫娜娜寫知識庫 → 她**改用 `dispatch_to_agent` 請管家寫，
而管家走 kiro-cli 有全部權限，寫入成功**（頁數 54→55）。

要真的擋住寫入，得限制**被派工那一端**，不是娜娜的工具表。
這裡不假裝已經安全 —— 白名單解決的是「設定說謊」，不是「模型不能寫」。

### Added

- 8 個測試：過濾生效、`None`=全部、未知名稱 warning、排除有 log、
  全排除退回全部、空 registry 回 `None`、三個寫入工具不在預設、
  **預設白名單的每個名字都必須真的註冊得到**（原 bug 的直接守門）。

全庫 1425 passed / 6 skipped。

---

## [0.4.2] — 2026-08-18

**M5.6 實測產出。** 31 條結構驗證 + 三條真實路由，挖出四個「宣告了但沒接上」
與一個線上故障。四個都不拋例外、不寫 log。

### Fixed

- 🔴 **M4 報告管線在 TG 上從來沒執行過** —— `deliver_report()` 只被
  `ModeRouter._maybe_report()` 呼叫，而 **`ModeRouter.route()` 沒有任何呼叫點**
  （TG 走 `handlers.py` 自己那套 Path 1/2/3）。整個 M4 里程碑的產出在正式路徑上
  等於不存在。修法是 handlers 兩條收尾各呼叫 `_attach_report()`，
  而它**借用 router 的 `_maybe_report`** —— 不自己重寫判斷，否則門檻、
  標題規則、降級訊息會變成兩份各自漂移。
- 🔴 **清洗與報告的順序反了** —— 過濾只掛在 TG 層、報告管線在 router 裡 →
  MD/HTML 會存進工具雜訊（`I will run the following command: … (using tool: shell)`），
  **而畫面上看起來乾淨**，因為 TG 那層之後又清了一次。
  新增 `clean_cli_output()`，在 `run_agent` / `run_team` 建 `Reply` 時就清；
  報告也移到 3000 字截斷之前（截斷是 TG 訊息限制，不該切掉存檔內容）。
- 🔴 **BM25 索引路徑的第三個實例** —— `server/main.py` 的 index-status
  自己組路徑（少一層 `shared`）→ rebuild 明明成功、status 永遠回 `not_built`。
  `indexer` 與 `layer1_bm25` 一致（M2.1 的修正還在），只有端點沒走常數。
  **「模組不自己組路徑」要貫徹到端點層。**
- 🟠 `modes.chat.max_iterations` 三處硬編 `5` —— 剛好與設定同值，
  所以「沒接上」看不出來。
- ⚠️ **`google-genai` 下限提到 `>=2.0`** —— 這是線上故障：
  💬 快答模式只要命中任何工具就回 `400 INVALID_ARGUMENT —
  Function call is missing a thought_signature`。
  程式碼三層 plumbing 都寫好了，但 SDK **1.2.0 的 `Part` 沒有這個屬性**
  → `getattr` 永遠回 None → 送不回去。**升級解決不了，要釘下限** ——
  否則重建環境會再故障一次，而症狀看起來像 API 或金鑰問題。

### Added

- `bot.router.clean_cli_output()` —— kiro-cli 輸出清洗，可在 router 層使用
- `bot.handlers._attach_report()` —— TG 側的報告接點 + `DocumentSender` 注入
- `tests/ark_bot_agent/test_report_wiring.py`（4 條）——
  驗**可達性與副作用**，不驗函式存在

### 這批缺陷的共同形狀

`import` 掃得到、單元測試會過、模組載入得了，**就是沒有人呼叫**。
掃字面值與檢查符號存在都擋不住，只有行為測試釘得住。

### 驗證（nana-bot 實測）

```
BM25 index-status → {"status":"ok", page_count: 54, has_bm25: true}   ← 先前 not_built
快答工具往返      → search_wiki ×2 + web_search，無 400
Tier 0–3 全綠、ERROR 0
```

全庫 1379 passed / 6 skipped。

---

## [0.4.1] — 2026-08-18

### Removed
- `/recall` 指令已移除（import + BOT_COMMANDS + handler 三處清除）

## [0.4.0] — 2026-08-17

### Changed
- 輸出過濾重構為三層管線（`bot/output_filter.py`）
  - Layer 1: `clean_output()` — 移除 ANSI/XML/spinner/工具UI/shell/docker/pytest 噪音（80+ 規則）
  - Layer 2: `extract_final_reply()` — `> ` 精確提取 + [DONE] + reply() + fallback + 3500 字硬截斷
  - Layer 3: `_format_telegram()` — 既有 Markdown → TG HTML 轉換
- 移除舊單層 `_clean_output()` 與 `_TOOL_LINE_PREFIXES`

### Added
- `bot/output_filter.py` — 獨立可測試的過濾模組
- 25 個單元測試

## [0.3.1] — 2026-08-17

### Fixed
- **skill 註冊表全空** —— 平移漏改的第四個靜默失效。`server/main.py`、
  `bot/handlers.py`（×2）、`skills/registry.py` 仍把 **`"src.skills.internal"`**
  （消費端 M5 已刪的 layout）傳給 `importlib`，而 `auto_discover` 當時
  `except ImportError: return 0` **不留 log**：

  | 位置 | 症狀 |
  |---|---|
  | 啟動 log | `📦 Skills loaded: 0` |
  | `GET /api/v1/skills` | 回空表 |
  | 提詞組裝「可用技能」×2 | **agent 從來不知道自己有哪些技能** |
  | `hot_reload()` | **永遠回 `False`**（猜的 module 路徑不存在） |

  而啟動橫幅同時顯示「Internal 15」—— **數字對，事實錯**。

  **為什麼漏**：已有 `test_no_src_imports()` 守門，但它只看 `import` 語句，
  漏改的是**傳給 `importlib` 的字串**。守門條件要覆蓋動態 import 的字串；
  只驗寫法不驗實際值的檢查會給假綠燈。

- `hot_reload()` 改依 `packages()` 逐個套件找，不猜 module 路徑

### Added
- **共用 registry**（`get_registry()` / `register_package()`）——
  原本每個呼叫點各自 `SkillRegistry()` 再 discover 一次，於是
  `create_bot(skills=[...])` 注入的消費端 skill **只進得了 runner 那份統計表**，
  server 與提詞組裝看不到。**註冊表是全域事實，不該一個呼叫點一份。**
- `SkillRegistry.count_in()` / `packages()` —— 供橫幅分開報來源
- 4 個迴歸測試（字串型 module path 守門、共用 registry 可見性、`hot_reload`、
  注入失敗要可見）+ `conftest.py` autouse fixture 歸零模組級 registry
  （共用狀態要嘛不共用，要嘛測試層負責歸零）

### Changed
- `auto_discover()` import 失敗**留 log**（仍回 0 —— 消費端沒放 skill 是正常狀態）
- 啟動橫幅**內建與消費端分開報**，混成一個總數會藏住「消費端根本沒載到」：
  `📦 Skills: IDE 60 | 內建 12 | 消費端 3`。
  注入了卻一個都沒載到時直接明講
- 註解殘留的「專員」清成「管家」（6 處，純註解，使用者可見字串早已改名）

### 驗證（nana-bot 實測）

三個數字現在互相一致（先前橫幅 15 / server 0 / 端點空）：

```
📦 Skills: IDE 60 | 內建 12 | 消費端 3
📦 Skills loaded: 15（來源：ark_bot_agent.skills.internal、skills）
GET /api/v1/skills → 15（含消費端 news / news_renderer / news_scraper）
```

全庫 1348 passed / 7 skipped。

## [0.3.0] — 2026-08-17

### Added
- 設定層、三模式路由、報告管線初版
- MD-first 報告流程（MD → HTML → TG）

## [0.2.0] — 2026-08-17

### Added
- 初始發布：路徑層、三模式框架、Wiki/Memory 系統
