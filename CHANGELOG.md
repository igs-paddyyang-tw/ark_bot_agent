# Changelog

> **日期以實際發生為準**（GitHub Release `published_at` / 本機 wheel 產出時間），
> 不是推估值。M0–M6 的套件化是 **2026-08-17 一天內**走完的，
> 所以 0.2.0 ~ 0.4.0 全部同日。
>
> ⚠️ **有 GitHub Release 的是 `v0.2.0` / `v0.4.1` / `v0.4.3` / `v0.6.0` / `v0.7.0` / `v0.7.1`**（tag 六個）。
> **`0.7.2` 刻意不發** —— 工作區含另一位維護者標記 `[0.8.0]` 的功能，留給對方隨 0.8.0 出。
> **`0.3.0` / `0.3.1` / `0.4.0` / `0.4.2` / `0.5.0` / `0.5.1` / `0.5.2` 只有本機 wheel**，沒有發布 ——
> 讀這份的人會預設每一條都拿得到 asset，所以要講明。
> `v0.4.3` 的 notes 涵蓋 0.4.2；**`v0.6.0` 的 notes 涵蓋 0.5.0–0.6.0**。
>
> nana-bot 實裝 **0.4.3**（sha256 與 release asset 逐位元組核對一致）。

## [0.7.2] — **未發布**（工作區狀態）

> ⚠️ **這一版沒有 GitHub Release。** 工作區同時含另一位維護者標記為
> `[0.8.0]` 的 `team_backend` 整合（預設關、目前零測試涵蓋），
> **發成 0.7.2 會把 0.8.0 的功能塞進 patch 版號**，所以留給對方隨 0.8.0 一起出。
>
> 下列修正**已在 main**，會隨下一個發布版本出去。

### Fixed

- 🔴 **「少一層 `shared`」的第四個實例，而且是功能性缺陷** ——
  `skills/internal/wiki_distill` 把蒸餾產出寫進 `knowledge/wiki`（1 篇），
  而引擎的索引與搜尋讀 `knowledge/shared/wiki`（21 篇）
  → **蒸餾出來的知識永遠不會被索引，也搜不到**。

  四次全記錄（都是少一層、都不拋例外不寫 log）：

  | 次 | 位置 | 症狀 |
  |:-:|---|---|
  | 1 | `layer1_bm25` 讀 `knowledge/.index` | BM25 索引永遠讀不到，靜默 fallback |
  | 2 | `indexer` 與 `layer1` 讀寫不一致 | 同上 |
  | 3 | `server/main.py` 的 index-status 手組 | rebuild 回 `ok` 但 status 永遠 `not_built` |
  | 4 | **`wiki_distill` 寫 `knowledge/wiki`** | **蒸餾產出不進索引、搜不到** |

- 🔴 **常數只是半個 bug** —— `_DISTILL_PROMPT` **自己也告訴模型錯的路徑**
  （「存放到 `knowledge/wiki/`」）。只改常數沒有用，模型照 prompt 走。
  docstring / description / prompt 內的 `index.md` 與 `log.md` 共五處都改了。

  > **「同一事實寫兩處」的變體：其中一處是給模型看的自然語言，掃程式碼掃不到。**
  > 已加測試比對「prompt 內的路徑」與「常數」。

- `memory/indexer.DB_PATH` 與 `skills/internal/chat_history.DB_PATH`
  改 `_db_path()` 延遲計算 —— DB 是**寫入目標**，凍結錯的家等於資料寫到別處
  （與 `session` / `chat_trace` 同一處置）。
- 移除 `wiki_distill._RAW_DIR` / `_LOG_PATH` / `_INDEX_PATH` ——
  **死碼且路徑也錯**，留著只會被人照抄。

### Added

- **`paths.py` 五個正規存取器** —— `get_shared_dir()` /
  `get_shared_wiki_dir()` / `get_shared_raw_dir()` / `get_index_dir()` /
  `get_shared_tasks_dir()`。手組的 **13 處**全部收斂。

  > 💡 **修第 N 次都只是修那一次。** 四次同型 bug 的共同來源是
  > 「每個人自己組路徑」—— **加單一出口才是修這一類**。

- **`test_path_constants_sweep.py`** —— 守門改用 `ast` **自動掃描**，
  不再靠手寫清單（手寫清單不會涵蓋新增的常數）。驗六件事：
  都在 `get_home()` 底下／不准手組 `shared`／wiki 概念的常數互相一致／
  索引讀寫一致／prompt 與常數一致／死碼不復活。

  > **掃描一跑就抓到另外 7 處人工盤點漏掉的** —— 它們在**函式內**
  > 而不是模組常數，其中 `server/main.py` 有 4 處同樣的 `shared/wiki`。
  >
  > 💡 **人工盤點會漏，而且漏的地方沒有規律。**
  > 同型錯誤出現三次以上就該寫掃描，不要再盤一次。

### 順帶修掉的版號分岔

`pyproject` 已是 `0.7.2` 而 `__init__.py` 停在 `0.7.0`
—— **正是 0.4.1 加測試防的那個分岔**（該版 wheel 內部版號寫著 0.7.0）。已對齊。

> 🔴 **共用套件開發源時，動版號前先看 `pyproject` 與 `git log`。**
> 「已安裝版本 ≠ pyproject 版本」本身就是警訊。

ark_bot_agent **349 passed**。

---

## [0.7.1] — 2026-08-18

> 由另一位維護者發布（本檔先前缺此條目，事後依 commit 與程式碼補上）。

### Added

- **TG 檔案白名單** —— `report/sender.py` 的 `ALLOWED_FILE_TYPES = {".html", ".md"}`。
  資安限制：不在白名單的副檔名**拒絕傳送並 log**，不靜默略過。
  可用 `allowed_types` 參數覆寫。
- 長訊息分段 —— `bot/progress.py` 的 `_split_html_safe()`（TG 4096 字元限制）。
  **不在渲染層截斷**，由 `ProgressStack` 分段送出。

---

## [0.7.0] — 2026-08-18

**群組策略與觀測性補強**。🟠 **群組行為改變**（私訊不變）。

來源：`ninja-bot 客製化影響評估`。該報告基於 v0.3.0，**逐項查證後
3 項已完成、1 項說法不準**，剩下 5 項納入本版。

### Fixed

- 🔴 **套件在群組中會回應每一則訊息** ——
  `handlers.handle_message()` **零處 `chat.type` 判斷**，所有訊息走同一條路。
  私訊時是對的，群組時 bot 變成噪音來源，**群組立刻不能用**。

  新增 `access.group_policy`：

  | 值 | 行為 |
  |---|---|
  | `mention_only`（**預設**） | 只回被 @ 的；其餘**仍寫記憶**（群組是資訊來源） |
  | `all` | 每則都回（0.6.0 的行為） |
  | `off` | 群組完全不參與 |

  ⚠️ **升級注意**：預設值改變了群組行為。**要舊行為設 `group_policy: all`**
  —— 那也是回滾出口，不必降版。**私訊行為完全不變。**

  > 預設選 `mention_only` 而非沿用現行（ADR-001）：
  > **相容性保護的是「有人刻意依賴的行為」，不是「碰巧存在的缺陷」。**
  > 一個把每則訊息都回應的 bot 沒有人會刻意想要。

- 移除 `skills/internal/reaction_manager.py` —— **真死碼**：
  它的 `ReactionManager` **不是 `BaseSkill` 子類**（連 skill 都註冊不了），
  且除自己外零引用；而 `handlers.py` 有 12 處 `_set_reaction()` 在服役。

  > 選「刪」而非「接上」（ADR-006）：handlers 那套是**驗證過的**，
  > skill 版從未被呼叫 —— **把活的換成死的是用已知可用換未知**。
  >
  > `ark_team_agent` 有自己的 `reaction_manager.py`（實際在用），沒動。

- `memory/chat_trace.DB_PATH` → `_db_path()` 延遲計算。
  模組層常數在 import 時凍結 `get_state_dir()`，**誰先 import 就綁誰的家**
  （症狀：測試單獨跑會過、整檔跑會壞）。與 0.6.0 修的 session 那個同一類。

### Added

- **`backend.state_dir`** / **`ARK_BOT_STATE_DIR`** ——
  執行期狀態目錄可覆寫（env 給部署期、設定給版控）。
  未設時與 0.6.0 同值。**相對路徑相對 `get_home()` 不是 cwd**；
  指定路徑建不起來時 error log + 沿用預設。
  用途：消費端從舊 layout（`data/sessions.db`）遷移不必 symlink
  —— **但套件不搬檔，既有資料要自己搬**。
- **`chat_trace.stats(days)`** + **`GET /api/chat/stats`** ——
  路由決策聚合（等價 ninja 的 `/router_stats`）。
  純 SQL `GROUP BY`，**不加新儲存**（ADR-007）：資料已在收，
  缺的是出口不是資料，加第二套會讓兩份數字對不上。

  兩個判斷寫進實作：**pending 不進成功率分母**（剛送出的訊息不該拉低成功率）；
  空字串與 NULL 都歸「（未標記）」（分兩類會讓人以為是兩件事）。
- **`llm/prompt_budget.py`** —— chat context 預算量測（可重複執行）。

### 量測結果（`docs/reports/2026-08-18-chat-prompt-budget.md`）

每則訊息 **10,774 字元 ≈ 2,693 tokens**（「你好」也一樣），
而結果**推翻直覺**：最大的一塊不是記憶或知識庫，
是我們自己寫死的 `tone_rules`（**3,333 字元 = 34%**）。硬編字串合計 53%。

**結論：值得優化但不在本版做**（ADR-004）。報告列四項建議（可省 ~24%），
八成來自精簡 `tone_rules`，**但那項必須先有 A/B 對照實測** ——
語氣是 UX，退化了比多花 token 糟。

> 💡 這是 M3 教訓的複用：照計畫字面優化會改錯地方。
> **量測工具本身就是交付物** —— 沒有它，下次改動無從比較。

### 對來源報告的三處事實更新

| 報告項目 | 現況 |
|---|---|
| #7 多模型 MODEL_MAP ❌ | ✅ **0.5.2** `llm.models` 依用途 |
| #2 Permissions 降級 | ✅ **0.6.0** 等級 + 執行期增刪 + 持久化 |
| #5 Web UI 需覆寫 | ✅ **M1** `get_templates_dir()` 消費端優先 |

→ 消費端 pin 建議 **`>=0.7,<0.8`**。

ark_bot_agent **317 passed**。

---

## [0.6.0] — 2026-08-18

**AccessControl + verify hook + UI helpers**（M4）。🟠 設定格式擴充（向後相容）。

### Fixed

- 🔴 **`bot.yaml` 的 `access` 段從來沒有任何人讀** ——
  `handlers._ALLOWED_USERS` **只從環境變數 `ADMIN_CHAT_IDS` 取**，
  而 `.env` 沒設時它是空集合 → 「空＝不限制」→
  **設了白名單卻對所有人開放**。

  實測（0.5.2）：

  ```
  access.admin_chat_ids     = [937896656]
  handlers._ALLOWED_USERS   = set()
  _is_authorized(123456789) → True        ← 陌生人通過
  ```

  註解寫著「bot.yaml 的 `access.admin_chat_ids`；env 可覆寫」，
  **但實作只有 env 那一半**。

  > 這是「宣告了但沒接上」**最危險的形狀** ——
  > 不是功能沒生效，而是**安全設定沒生效而且看起來生效了**。
  > 與 `modes.chat.tools`（0.4.3）同型但後果更嚴重。

  ⚠️ **升級注意**：0.6.0 之後 `access` 段會**真的生效**。
  請先確認裡面的內容是你要的，否則升級後你可能把自己鎖在外面。

- 移除 `handlers._SOUL_DIR` —— **純死碼**（只有定義沒有引用），且寫死 admin-agent
- `session._DB_PATH` → `_db_path()` 延遲計算。模組層路徑常數在 import 時凍結，
  **誰先 import 就綁誰的家** —— 症狀是測試「單獨跑會過、完整跑會壞」

### Added

- **`access.py`** —— `AccessControl`（權限等級 + 執行期增刪 + 持久化）
  - 靜態（`bot.yaml` + env，**疊加**不是取代）+ 動態（`state/access.json`）
  - 舊格式 `admin_chat_ids: [int]` **載入時正規化**成 `{id: level}`，
    執行期只有一種形狀（兩種並存＝每個讀取點判斷兩次）
  - **不回寫 `bot.yaml`** —— 那會動到人寫的註解與排版
  - 空白名單維持「不限制」，但 **`is_admin()` 回 `False`** ——
    「不限制使用」不等於「都是管理員」
  - **只能移除動態加入的** —— 移除靜態項目會讓下次重啟又出現，
    **那種假成功比拒絕更糟**
  - 壞掉的 state 檔不讓 Bot 起不來，**但一定 log**（靜默忽略等於白名單被悄悄清空）
- `access.users`（新格式）· `modes.team.verify`
- **team_flow 選配 verify** —— 收斂後的可選後處理，**預設關**。
  有意見**附在後面不改寫答覆本身**（驗證者不是作者）；
  verify 失敗**不影響已完成的協作**；模型由 `llm.models.verify` 決定，**不寫死廠商**
- `bot/keyboard.py` · `bot/pagination.py`（自 ninja-bot 搬入，純 UI）

### 明確不取

**`tool_tracker.py`** —— 它靠 `feed(text)` 從**輸出流**解析進度，
而套件 `_execute()` 用 `communicate()` 等完整輸出 —— **沒有 stream 就沒得 feed**。
要它得先做 backend 的 stream 讀取，那是另一個範圍。
套件的 `ProgressStack` 是呼叫端驅動，兩者不是同一種東西 —— **但不並存**。

ark_bot_agent **283 passed**。

---

## [0.5.2] — 2026-08-18

**規劃不帶歷史 + 多模型分工**（M3）。純增益。

### ⚠️ 原計畫的前提不成立，量測後改了做法

計畫抄的是 ninja-bot 的「21k → 686 tokens（-97%）」，但那來自
Claude CLI 帶完整工具 schema 的規劃呼叫。本套件實測：

```
plan prompt        654 字元
describe_agents    203 字元   ← 原本要精簡這個
leader steering   3901 字元   ← 真正的量都在這
```

精簡成員描述最多省 ~100 字元，在 4555 字元總 context 裡是噪音。
**照計畫字面做會改錯地方，而且看起來有做事。**

真正的成本源：`leader` 是 `skip_resume: false` → 每次規劃都跑
`kiro-cli chat --resume`，**重播整段累積歷史**。

### Added

- **`agent_cli_chat(fresh=True)`** —— 不帶對話歷史的一次性呼叫。
  規劃階段的 prompt 是自足的（任務 + 成員清單），歷史純粹是成本。

  💡 **不會多付冷啟成本** —— `AgentProcess._execute()` 本來就是每則訊息
  spawn 一個新 process，所謂「常駐」是**佇列與設定**，不是長駐 REPL。
- **`llm/model_router.py`** —— `model_for(purpose)`。
  `llm.models` 以**用途**為 key（`chat` / `plan` / `verify` / `synthesis`），
  未指定 fallback 到 `llm.model`（**不可移除**，既有部署還在用）

  > key 用「用途」而非 ninja 的「節點名」：節點名與工作流拓樸綁死，
  > 而且無法判斷未知 key 是打錯還是新節點 —— 用途名可以 warning。

  **打錯的 key 會明講** —— `verfiy: strong` 靜默 fallback 會讓那個設定
  永遠不生效而行為看起來正常（與 `modes.chat.tools` 同型：設定說謊）。

含 `test_plan_prompt_stays_lean()` 釘住規模上限（現在 654，上限 1500）——
它現在是精簡的，別讓人往裡面塞東西。

全庫 1470 → **1479 passed**。

---

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
