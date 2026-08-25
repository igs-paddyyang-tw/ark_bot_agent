# Changelog

> **發版狀態不是每版都有。** 有 GitHub Release 的是
> `v0.2.0` · `v0.4.1` · `v0.4.3` · `v0.6.0` · `v0.7.0` · `v0.7.1` · `v0.8.0`
> · `v0.11.3` · `v0.11.4` · `v0.11.5`（依 Release API 核對）。
> 其餘版本只有本機 wheel —— 讀本檔不要預設每一條都拿得到 asset。
>
> 0.4.2 – 0.7.2 的條目是 2026-08-19 依 git commit 回填的（當時只寫到 0.4.1）。
> 頂部曾有 0.10.1 / 0.7.2 的重複摘要（`ccac2b1` 補條目時錯插）+ 0.11.5 掉標題，
> 2026-08-25 已重排修正（完整版在下方各自版號處）。

## [1.0.1] — 2026-08-25 · 記憶落點收乾淨：steering 與 knowledge 也跟著「家」走

1.0.0 只把 **memory** 的落點收進 `paths.get_agent_memory_dir()` 單一出口，
但 **steering 與 knowledge** 仍有多處自組 `agents/<name>/...` —— manager
（`dir: "."`）因此**人格讀根層、記憶／知識卻讀 `agents/manager-agent/`**，
同一個 agent 的家被解析成兩個。這一版把剩下的收乾淨。

### 🔴 症狀：manager 讀不到自己剛寫的記憶

1.0.0 讓 manager 的 daily log **寫**到根層 `memory/`，但 `context_builder`
組提示時仍去 `agents/manager-agent/memory/` **讀** memory.md / recent.md
→ 寫在一處、讀在另一處，manager 的持久記憶與最近經驗**永遠讀不到**。
steering（SOUL.md）與 knowledge（歷史記憶搜尋、save_memory）同型。

### 修法：新增 `get_agent_knowledge_dir()`，10 處全接上出口

`paths.py` 補上 knowledge 的單一出口（與 memory/steering 同一個底座
`get_agent_home`：家在專案根 → 根層 `knowledge/`）。收斂的呼叫點：

| 類 | 檔案 | 改為 |
|----|------|------|
| memory | `bot/handlers.py`（memory.md / recent.md）· `memory/api.py`（daily） | `get_agent_memory_dir()` |
| steering | `bot/handlers.py`（`_load_soul` + fallback）· `llm/tools/dispatch.py` · `agent/cli.py` | `get_agent_steering_dir()` |
| knowledge | `wiki/engine.py`（raw/wiki）· `bot/handlers.py`（`_search_memory`）· `agent/memory.py`（`save_memory`） | `get_agent_knowledge_dir()` |

`dispatch.py` 順帶把兩層 SOUL fallback 收成一行（出口內部已處理 id／`-agent` 後綴）。

### 守門：memory / steering / knowledge 三類掃描 + 三家合一

`tests/ark_bot_agent/test_manager_memory_home.py` 擴充：
- `test_three_homes_are_one` —— 同一個 agent 的 memory／steering／knowledge 必解析到同一個家
- steering／knowledge 各一條 `ast` 掃描（禁止再自組），比照原本的 memory 掃描
- 行為層 `test_save_memory_lands_in_root_for_manager` —— 真的寫一次，驗不再生孤兒目錄
- **432 passed**（無回歸）

### 已知豁免（不在本次範圍）

`server/main.py` 的 wiki 瀏覽 API 收的是**瀏覽樹傳來、已含 `agents/{name}`
前綴的相對路徑**（Web UI 內部自洽的路徑，非用 agent_id 解析家）——
要收斂需與 `scan_dir` 一起改，屬 Web UI 子系統重構，測試已標記豁免。

## [1.0.0] — 2026-08-25 · 記憶落點跟著「家」走 · 首個 1.x

### 為什麼是 1.0.0 而不是 0.12.1

單看程式碼這是一個路徑解析的修正，但它補上的是 **0.12.0 那次架構收斂的最後一塊**：
0.12.0 讓三個模式各自綁定自己的 `.kiro` 來源（chat=admin-agent、agent=根目錄），
本版讓**記憶落點也照同一條判準**。三模式的「家」從此是一個一致的概念 ——
steering、工作目錄、記憶三者同源，不再各自解析。

行為有變（見下方遷移），所以不是 patch；架構前提定型，所以離開 0.x。

### 🔴 `dir: "."` 的 agent 記憶被寫到孤兒目錄

`write_daily_log()` 一律組 `agents/<name>/memory/daily/`，於是
`agents.yaml` 裡 `dir: "."` 的 agent（manager／👑 管家）——
人格、steering、工作目錄全在專案根 —— **記憶卻另立門戶**，
生出一個除了 `memory/` 什麼都沒有的 `agents/<id>-agent/`。

語意矛盾：manager 是「根目錄的化身」，只有記憶不在根目錄。
而它**不拋例外、不寫 log**，與本套件記載過的「少一層 `shared`」同型 ——
**每個模組自己組路徑，就一定會有人組錯**。

實測本部署：`agents/paddy-agent/` 底下只有 `memory/daily/` 三個檔，
同一天的對話因此散在兩個檔案裡（根層一份、孤兒目錄一份）。

### 修法：`paths.get_agent_memory_dir()` 單一出口

判準是**「這個 agent 的家在哪」，不是「它叫什麼」**：

| agent | `dir` | 記憶落點 |
|---|---|---|
| `_default` | — | `memory/` |
| manager（👑 管家）| `.` | `memory/`　← 本版改變 |
| admin / leader / worker | `agents/<name>-agent` | `agents/<name>-agent/memory/`（不變）|

**沒有寫死 `role == "manager"`** —— 真正的不變量是「工作目錄 == 專案根」。
綁 role 的話，改天有人把 role 改成別的、`dir` 還是 `.`，就會靜默退回舊行為。

收斂的呼叫點五處（原本各組各的）：`memory/daily_log`、`memory/indexer`、
`llm/context_builder`、`llm/tools/memory_tools`、`bot/handlers`（recent.md 與昨日 log）。

### 遷移：`scripts/migrate_manager_memory.py`

既有檔案不會自己搬。預設 dry-run，`--apply` 才動檔案：

- 同名檔**按 `## HH:MM` 條目合併後排序**，不覆寫、不丟內容
- 只有真的搬空才移除孤兒目錄；非空一律保留並回報
- 本部署實測：搬移 2、合併 1，條目數 5+5 → 10（零損失）

### 同批對齊：規格文件 ↔ 程式碼（三處矛盾）

拿 `docs/reports/2026-08-25-nana-bot-three-mode-architecture.md` 逐條核對開發源，
挖出三處「文件的設計主張，程式碼沒做到」：

**① 「記憶落點跟著家走」原本只實作了一半。**
初版只有「家在專案根」那一支照 `dir` 解析，其餘仍拿名字拼：

```
設定的家 (dir)     : agents/foo-workspace
記憶落點 (resolver): agents/bar/memory        ← 與家不一致
```

改成 `get_agent_home()` 當底座，記憶（`get_agent_memory_dir`）與人格
（`get_agent_steering_dir`）都從它長出來 —— **人格與記憶不該解析到兩個家**。

> 💡 兩個部署的 `dir` 剛好都是 `agents/{id}-agent` 所以沒踩到。
> **那是巧合不是保證，而巧合不會出現在測試結果裡。**
> 順帶暴露一個既有隱患：`skill_manage.py` 傳的 `proposal["agent"]`
> 可能是 bare id，舊寫法會寫進 `agents/<bare-id>/`（設定裡不存在的目錄）。

**② 「`source_agent` 是部署配置、非源碼硬編」原本不成立 —— 有四份實作。**

| 模組 | 函式名 | fallback |
|---|---|---|
| `bot/handlers.py` | `_chat_source_agent` | `"admin-agent"` |
| `llm/context_builder.py` | `_chat_source_agent` | `""` |
| `llm/tools/memory_tools.py` | `_chat_memory_agent` | `"admin-agent"` |
| `llm/tools/wiki_search.py` | `_chat_wiki_agent` | `"admin-agent"` |

三種函式名、**兩種不一致的 fallback** —— 同一個問題在同一次對話裡
會得到兩種答案。收斂成 `config.chat_source_agent()` 單一出口，
fallback 統一為**空字串**（語意：不指定某個 agent → 專案根）。

> 🔴 **fallback 硬編別的部署的 agent 名，比沒有 fallback 更糟 —— 它看起來有根據。**
> 這三處正是 `check_hardcoded_names.py` 長期的紅燈，收斂後轉綠（exit 0）。

**③ `/status` 面板宣稱 chat 會派工。**

```
• 派工：dispatch_to_agent（7 Agents）      ← 修正前
```

`dispatch_to_agent` 在 0.12.0 就移出 chat 白名單，而「7」是別的部署的數字
（nana-bot 10 個、paddy 6 個）。改成從設定推導 Chat 工具清單與可派工數量。
**狀態面板說謊比沒有狀態面板更糟。**

### 守門（`tests/ark_bot_agent/test_manager_memory_home.py`，17 個）

兩層，且**每條都做過反證**（退回舊寫法確認會紅 → 3 failed / 4 failed 兩批）：

1. **行為層** —— 真的呼叫 `write_daily_log()` 驗檔案落在哪，
   並驗孤兒目錄沒有長回來。只驗「函式存在」或掃字面值會給假綠燈。
2. **掃描層** —— `ast` 掃全套件，禁止再出現 `get_agents_dir() / x / "memory"`
   （出口自己除外）、禁止再硬編 chat fallback、禁止面板寫死 agent 數量。
   附一條「用故意違規的程式碼證明掃描器會紅」——
   **掃描器的涵蓋範圍本身要被測試**。

## [0.12.0] — 2026-08-25 · 三模式 .kiro 來源統一

三個對話模式各自綁定不同的 `.kiro` 來源，職責分離。

### 架構

| 模式 | 引擎 | `.kiro` 來源 | 記憶 / 產出 |
|------|------|-------------|------------|
| 💬 chat | Gemini ReAct | `agents/admin-agent/`（管家人格） | 只看 admin-agent 自己 |
| 👑 agent | kiro-cli（cwd=根目錄） | 根目錄 `.kiro/`（= Kiro IDE 預設） | role=manager，常駐 |
| ⚔️ team | kiro-cli | `agents/leader-agent/` | leader 統籌，常駐 |

### 變更

- **chat 模式取消派工**：拿掉 `dispatch_to_agent`，改為純自答（查知識庫／記憶／web）。
  要動手做的事 → 提醒使用者切模式或用 `@代號`。派工需求集中到 ⚔️ team。
- **chat 記憶只看自己**：`recall` 與 `search_wiki` 用 admin-agent 身份
  （wiki 查詢順序：admin-agent 私有 → 共用）。
- **`agents.yaml`**：新增 `manager`（dir=`.`, role=manager, persistent）；
  `admin` 保留給 chat（role=admin）。
- **`bot.yaml`**：`default_agent: manager`、新增 `chat.source_agent: admin-agent`。
- **`ChatModeConfig.source_agent`**：chat 讀哪個 agent 的人格/記憶，改為設定檔驅動
  （部署配置，非源碼硬編）。
- **`list_descriptions(only=...)`**：prompt 只列白名單內的工具，
  不再誤導 LLM 以為能呼叫實際被擋掉的工具。

## [0.11.5] — 2026-08-24 · ✅ Release

TG 送檔白名單與 `ark_team_agent` 1.5.5 同步。

### 白名單重排：交付物可以，程式碼與設定／機密不行

原本只有 `.html` / `.md` —— agent 產出的圖表、資料表、PDF 報告全部送不出去，
而它只看得到「檔案類型不允許傳送」，看不出是白名單的問題。

| | 放行 |
|---|---|
| 報告 | `.html` `.md` `.txt` `.pdf` |
| 資料 | `.csv` `.xlsx` |
| 圖片 | `.png` `.jpg` `.jpeg` |
| 文件 | `.docx` `.pptx` |

程式碼（`.py` `.sh` …）、設定（`.json` `.yaml` `.env` …）、憑證（`.pem` `.key`）、
封裝（`.zip` `.tar`）仍然擋著。

> 🔴 **`.json` 刻意不放行** —— 它不是「程式碼」，但機密最常長這樣
> （service account key、token）。判準是「會不會外洩祕密」，不是
> 「是不是程式碼」。封裝格式擋著的理由不同：**內容不可檢查，
> 放行等於整個白名單失效。**

### 🔴 這份清單有孿生的一份，而且在此之前沒有任何測試在管

`ark_bot_agent/report/sender.py`（本檔）與 `ark_team_agent/telegram.py` 各一份。
兩邊內容碰巧相同，是因為**沒人動過** —— 不是因為有機制保證。
只改一邊，症狀是「同一種檔案在 A bot 傳得出去、在 B bot 傳不出去」，
而兩邊的 log 都只說「不在白名單」。

> 💡 **碰巧一致不算一致。** 與本專案修過的 TEAM.md 工具表、代號表、
> BM25 路徑 ×4 同型。

守門 `tests/test_file_whitelist.py` 43 個（**跨兩個套件**）：兩份逐項比對、
四類擋掉的各自釘住、`.json` 單獨一條、格式正規化（少點或大寫會永遠比不中
且不報錯）。比對那條做過反證 —— 只改一邊確實變紅。

> ⚠️ 送檔分流（圖片走 `send_photo`）是 **`ark_team_agent` 側**的改動，
> 本套件沒有那條路徑（`deliver_report` 只送報告）。

## [0.11.4] — 2026-08-24 · ✅ Release

面板真 bug + 介面文字去硬編人名。

> ⚠️ 本段原本誤寫進 0.11.3 —— 那一版 **08-22 就已發布**
> （`igs-paddyyang-tw/ark_bot_agent` 的 `v0.11.3`）。本機 git tag 是
> `bot-v0.10.0`（開發源 repo 的 tag 系統），與發版 repo 的 tag 不同套，
> 只看本機 tag 會誤判成「未發布」。已發布的版本不改寫內容。

### 💬 快答模式的模式表硬編了別的部署的人名

`context_builder` §1a（「你現在是快答模式」那張表）是**手寫字串**，
寫著「👑 管家 = **admin** agent」「⚔️ 團隊 = **隊長**」—— 那是 nana-bot 的編制。
paddy 部署實際上 Agent 模式的對象是 `paddy`（admin 是「🛠️ 維運管家」）、
leader 代號叫「軍師」→ **快答模式會把使用者指向錯的人**。

改成 `_mode_labels()` 從 `modes.default_agent` / `modes.team_leader` 取
`AgentDef.display`，與同檔 §7b 的派工名單同一個原則。§1b 語氣示範與 §7b
派工舉例裡的代號（「碼哥」「阿智」）一併泛化 —— 它們也是別的部署才有的人。

**讀不到設定時退回泛稱，絕不填具體人名**：填錯的名字比不填更糟，
它看起來有根據，使用者會照著去找不存在的人。

> 🔴 **為什麼既有守門（0.11.x 的 `test_three_mode_dynamic.py` A 條）沒抓到**：
> A 掃的是 agent **id**（`admin-agent`），§1a 硬編的是**代號**（「管家」「隊長」）
> —— **兩種寫法互相掃不到**。已補 D 條掃代號，並用 ast 排除 docstring
> （說明文字要能舉例）。

### 🔴 `/agents` 面板在快答模式顯示 `🤖 *None*`

`_build_agents_text` 的第二個分支是 `if kind == "team":`（**不是 `elif`**）
→ chat 模式先在第一個 if 設好標題，接著掉進那個 `else`，而 chat／team 模式的
`session.agent_name` 是 `None`：

```
修前  chat  → 🎯 目前模式：🤖 *None*
修後  chat  → 🎯 目前模式：💬 *快答*
      team  → 🎯 目前模式：⚔️ *團隊*（🧠 軍師統籌）
```

### 介面文字全面去人名（面板 · /start · 派工提詞）

`_MODE_INFO` 是模組層 dict，硬編「管家」「隊長」「娜娜」，而面板下半部
還有**第二份**手寫的三模式說明（同一事實兩處）。`/start` 寫死
「我是娜娜，你的雲端助理」+「AWS 維護」。這些全是使用者在 TG 上直接
看到的文字 —— 比 prompt 更外顯，而每個部署都長一樣。

改成 `_mode_info()` 函式，label／desc 取自 `modes.default_agent` /
`modes.team_leader`；三模式說明改由它產生（第二份手寫清單刪除）；
`/start` 的自我介紹取 `bot.yaml` 的 `codename`；`@代號` 範例取自實際
dispatchable 成員。`team_flow` 收斂提詞與 `spec_executor` 的 `ROLE_MAP`
同批處理 —— 後者硬編 8 個 agent 名（含 `pm-agent`），漏改的症狀是
**在錯的目錄 spawn kiro-cli**（讀不到那個 agent 的 SOUL／skills／memory，
不拋例外，只是換了個人格回話）。

## [0.11.3] — 2026-08-22 · ✅ Release

修 Agent 模式慢的**真正根因**：常駐 Agent 池從沒啟動。

### `_post_init` 沒啟動常駐池 → 每則訊息都 cold-start

上一版（0.11.2）以為 Agent 模式慢是「context 大 + timeout 太短」，放寬了
paddy timeout 到 300s。但那**沒治到根因** —— 實測 paddy-bot 進程樹是空的，
**根本沒有常駐 kiro-cli**。

`start-bot.py`（生產入口）→ `create_app()` → `_post_init`，而 `_post_init`
**從沒呼叫 `start_all_agents()`**。那個函式只在 `bot/run.py` 與 `server/main.py`
（都沒被用到）被呼叫。後果：`_agents` dict 恆空 → `_cli_attempt` 找不到常駐 proc
→ 每則 Agent 訊息都走 subprocess fallback，付全套 cold-start（spawn + MCP 初始化 +
讀整個 steering + `--resume`），且吃 subprocess 的 120s timeout ——
**`persistent: true` 和 0.11.2 加的 `timeout: 300` 全都沒生效（設在沒開的池上）**。

修法：`_post_init` 開頭呼叫 `start_all_agents()`（比照 `bot/run.py`），
註冊常駐池；首則訊息 spawn 並 warm，之後每則省掉 spawn/MCP/resume。
加 `_post_shutdown` 呼叫 `stop_all_agents()` 優雅收屍。

### `start-bot.py` 沒初始化 logging → INFO 全丟

順帶發現：`start-bot.py` 從沒呼叫 `setup_logging()` → root logger 無 handler →
INFO 全被丟棄，只有 WARNING+ 靠 lastResort 到 stderr。「常駐池啟動了幾個」
「agent 就緒」這些關鍵 INFO 全看不到（與 ark_team_agent 早期 `run_team()`
沒初始化 logging 同型 —— 照文件做的部署反而沒日誌）。已在 `main()` 補上。

> 💡 0.11.2 的教訓延伸：**timeout 設對層之前，先確認那一層有在跑。**
> persistent/timeout 都設在常駐池上，而池沒開 → 設定看起來對、實際全空轉。

## [0.11.2] — 2026-08-22 · 未發布

修 Agent 模式對話崩潰（逾時被偽裝成通用錯誤）+ paddy 逾時放寬。

### `CliResult` 缺 `timed_out` 屬性 → AttributeError

`handlers.py` Path 2（Agent/Team 的 kiro-cli 回覆）讀 `cli_result.timed_out`
分辨「逾時」與「其他失敗」給不同訊息，但 `CliResult` **根本沒有這個屬性**
（它用 `status` Literal + `error` 字串）→ 每次 kiro-cli 逾時就
`AttributeError: 'CliResult' object has no attribute 'timed_out'`
→ 落到通用「抱歉，我剛剛出了點狀況 😵」，逾時被偽裝成未知錯。

修法：`CliResult` 加 `timed_out` property（`error.startswith("逾時")` 判斷，
逾時走 `CliResult.fail(describe("timeout", ...))` 的 error 就是「逾時（Ns）」）。
守門測試 `test_timed_out_property` 固定此行為。

### paddy 逾時 120s → 300s

paddy（manager）工作目錄是專案根，讀整個 `.kiro/steering`（全局視野 →
context 大 → kiro-cli 回應慢），實測撞 `backend.timeout` 的 120s。
agents.yaml 給 paddy 顯式 `timeout: 300`（persistent 常駐、不頻繁 spawn，
互動入口多等可接受）。其他 agent 維持 120s 不變。

> 根因鏈：120s 逾時是**真的慢**（大 context），但使用者看到的「出了點狀況」
> 是**假訊息**（AttributeError 蓋掉真相）。兩者都修 —— 放寬 timeout 治本，
> 修 property 讓萬一再逾時時使用者看到的是「⏱️ 處理超時」而非未知錯。

## [0.11.1] — 2026-08-22 · 未發布

三模式團隊認知動態化 + 移除無家的 Gemini team 工具。
**依版號規則歸 Patch**：把手寫字串換成既有唯一來源（動態化，行為修正）+ 移除
沒有正確歸屬的工具，無新公開 API。原標 0.12.0 與內容不符（0.11.0 未發過），
收斂為 0.11.1。

三模式獨立化 + 動態團隊認知。依 plan「ark_bot_agent 三模式獨立化」，
把三處手寫字串換成既有的**唯一來源**（`config.describe/dispatchable/group_members`）。

**共同主題（v1.6.0 同型）：基礎設施早就有單一來源，缺口全在手寫字串繞過它。**

### A — context_builder §7b 手寫 8-agent 代號表 → 動態

`build_default_system_prompt`（快答/agent 模式）§7b 是**獨立手寫**的代號對照表，
沒用 `config.describe()`；新增 agent 必漏改。改為 `config.describe(config.dispatchable())`
動態產生。同時把「派工判斷樹」收斂為快答模式原則（**明確點名或非那人不可才派**，
跨職能建議切 ⚔️ 團隊），與 §1a 一致 —— 快答本就不該積極派工。

### B — dispatch.register_tools fallback 死清單 → warning

主路徑 `get_dispatchable_agents()` 已動態，但空清單時塞一份 8-agent 硬編 fallback
→ 那份必然漂移。改為 **log warning 不假造**（空清單只有「沒設 dispatchable」或
「設定沒載入」兩因，都該被看見）；`target_agent` 的 `enum` 改條件加入
（空名單不放空 enum —— 空 enum 會禁掉所有值，比不限定更糟）。

### C — team 狀態工具：不做 Gemini 版（⚔️ Team 全走 ark-agent-cli）

原 v1.7.0 曾把 `query_team_status` / `list_tasks` 做成 Gemini function-calling 工具，
但 **Paddy 拍板：Team 模式全走 ark-agent-cli**（`run_team_flow` → `agent_cli_chat`），
根本不吃 Gemini 工具；而 💬 Chat / 👑 Agent 依「Team 才拿到」不該有團隊狀態工具
→ 這兩個 Gemini 工具**沒有正確的家，已移除**（`team_status.py` 刪除、`tools/__init__` 不再註冊）。

> team 認知交給 kiro-cli 的 MCP/steering（team-agent 側 `team_mcp` 提供）。
> 決策記錄見 `docs/plans/2026-08-22-ark-bot-team-awareness-plan.md` §六。

### D — 守門測試（`tests/ark_bot_agent/test_three_mode_dynamic.py`）

用 ast 掃 context_builder / dispatch 的**字串字面值**不得再含 agent id
（排除 docstring/註解）；驗 Gemini registry **不得**含 `query_team_status` / `list_tasks`、
且 `team_status.py` 已刪除（留著會被下一個人接回去）。



依 `docs/specs/ark-bot-agent-spec.md` + `docs/designs/ark-bot-agent-design.md`
做 code 與規格的差異比對，八項待實作全部收掉。

**共同主題：這八項都不是功能寫錯，是寫好了沒有人呼叫，或呼叫得到但在這個環境不生效。**

### 🔴 N-8 最嚴重：消費端注入的 router 從來沒有被執行過

```python
create_bot(router=NinjaRouter())     # 存進 Bot.router
bot.run()                            # uvicorn.run("ark_bot_agent.server.main:app")
                                     #             ↑ import string！
```

`run()` 用 **import string** 啟 uvicorn，uvicorn 會**重新 import 模組**建 app
—— `Bot` 實例（和它身上的 router）在那條路徑上完全看不到。而全套件
**沒有任何地方讀 `Bot.router`**。

所以 `ninja-bot` 的 `NinjaRouter.route()`（**LLM 意圖分流，它的招牌功能**）
在結構上不可能被執行過。

`ModeRouter.route()` 的「零呼叫點」從 0.4.2 就記錄過，但當時的結論是
「TG 走 handlers 自己那套」——
**真正的後果不是套件少一個功能，是消費端的擴充點是假的。**

> 💡 **「零呼叫點」要往上追一層問「那誰該呼叫它」。**
> 只看套件內部會停在「TG 有自己的路徑，所以沒差」。

修法：`bot/router.py` 加模組層 registry（`set_router` / `get_router`），
`create_bot()` 註冊進去，`handlers` 在 `has_custom_route()` 為真時委派。
這與 0.4.1 修 skill 註冊表同一個理由：**註冊表是全域事實，不該一個呼叫點一份**，
而跨 import 邊界的東西只能走模組層。

**最小接法**：只在真的覆寫 `route()` 時委派，沒覆寫就走原路徑，
**預設行為逐位元組不變**。只覆寫 `run_*()` 的子類**不算** custom
（那種消費端要換的是某一段行為，不是接管整條路由）。
自訂 router 拋例外時**退回預設路徑**，不是把 stack trace 給使用者。

### 🔴 N-5：TG 斷線時健康檢查全綠

實測：某部署把 TG 拆到另一個進程後那個進程沒人啟動，
**TG 入口斷 6.5 小時、3 則訊息卡在 Telegram**，而 `/health` 全程回 `{"status":"ok"}`。

**壞掉的東西不在被觀測的範圍內，所以沒有任何訊號。**

`/health` 與 `/api/v1/health` 現在帶 `telegram` 欄位，三態：

| `status` | 意思 | 算不算健康 |
|---|---|---|
| `disabled` | 沒設 token，這個部署刻意不要 TG | ✅ |
| `polling` | updater 正在跑 | ✅ |
| `stalled` | **設了 token 但 updater 沒在跑** | ❌ → `degraded` |

> 🔴 `disabled` 與 `stalled` 必須分開 —— 少了這個區分，
> 「刻意不用 TG 的部署」會永遠顯示不健康，
> 而**常駐假警報會讓人開始忽略這個欄位**。
> 健康檢查**不對外發請求**（不該依賴外部服務的可用性），有測試釘住。

### N-1 / N-2：`team_backend` 三態 + `url` 不給預設

依使用者拍板「預設不開，要在裝 `ark-team-agent` 後才會開啟」。

```yaml
team_backend:
  enabled: auto     # auto（預設）| true | false
  url: ""           # 空 = 未設定 → 不啟用
```

`auto` 用 **`importlib.util.find_spec("ark_team_agent")`** 判斷 ——
🔴 **不能 `import`**。套件的硬約束是「`ark_bot_agent` 不得 import
`ark_team_agent`」，那是「只裝一個也能跑」的地基；`find_spec` 只查 metadata
不執行模組，兩者不衝突。**有測試攔截 import 來釘住這件事。**

`url` 預設從 `http://localhost:13030` 改成**空字串** —— 那是 **nana 的 port**，
每個部署都得覆寫，**忘了就派工到別的團隊而且不會報錯**。
現在未設就不啟用，`enabled: true` 卻沒 url 會 warning 指出要設哪個欄位。

⚠️ **`.enabled` 的讀取點全部改走 `is_on()`** —— 直接讀會拿到字串 `"auto"`，
而**非空字串是 truthy**，「預設關閉」會變成「預設開啟」。
守門測試用 `ast` 找真正的屬性存取（不是掃字面值，否則會被 warning 訊息自己誤報）。

### N-3：HTTP 派工失敗退回內建 team

`run_team()` 的降級邏輯**本來就完整**（含 `queued` 不可降級 ——
訊息已進對方 overflow 佇列，再跑一次內建 team 就是同一件事做兩遍）。
它進不去的原因是 N-8。**N-3 是被 N-8 卡住的，不是沒寫。**

### N-6：群組未授權改靜默

白名單檢查排在 `group_policy` **之前**，所以 0.7.0 的群組策略擋不到它 ——
任何非白名單成員 @ 一下就多一則「🔒 需要權限 + 你的 chat id」。
私訊時那是對的（對方需要知道去哪要權限），**群組時它讓 bot 變噪音來源**。

### N-7：啟動偵測 privacy mode

`group_policy` 的「不回應但仍寫記憶」需要 privacy mode 關閉才完整生效。
開著時 bot **根本收不到**一般群組訊息 → `_record_only()` 永遠走不到。

🔴 **這個開關不在程式碼也不在 `bot.yaml`，在 BotFather** ——
程式碼怎麼讀都看不出來，只能啟動時問 Telegram 一次。
與 `ark-team-agent` v1.3.3 的 `restart.flag`（實作存在但在這個環境沒被啟用）同型。

`group_policy: off` 時不提示；偵測失敗不影響啟動。

### N-4：`agents.yaml` ↔ `team.yaml` 對映守門

同一批 agent 目錄被**兩個控制面**驅動，兩種命名、兩份宣告，先前無比對機制。

🔴 **錨點是目錄不是名字。** 直覺會比對「`ai-dev` 去尾等於 `ai-dev-agent`」，
但真正決定行為的是**兩邊指到同一個工作目錄** —— kiro-cli 在哪啟動就載入
那裡的 `.kiro/steering/`，也就是決定「它是誰」。
**名字對得上而目錄指錯，agent 會用錯的人格回話而不會有任何錯誤訊息。**

`tests/test_agent_config_mapping.py`（8 測試）驗：孤兒目錄、TG 叫不到的
instance、名字與目錄對映不一致、目錄實際存在、管家必須跑在專案根、
bot 側不得有重複目錄。實測故意改壞一個 `dir` → 三條從不同角度同時變紅。

**刻意容許的不對稱**：`paddy`（`dir: "."`，管家跑專案根有全局視野）
沒有 team 對應；`admin-agent` 有自己的目錄但 bot 那邊不用它。

### 🟠 團隊進度訊息硬編稱呼與 emoji —— 畫面在說謊

`bot/team_flow.py` 四處寫死：

```python
await _tick("🎯 隊長規劃中⋯")                        # L306
await _tick(f"⚔️ {names} 執行中⋯（{len(items)} 人）")   # L359（names 只有 codename）
await on_progress("⚔️ " + "　".join(done_names) + tail) # L250（逐一更新）
await _tick("🎯 隊長彙整中⋯")                        # L383
```

而 `modes.team_leader` 可以指向任何 agent，`agents.yaml` 也早就給了
每個人自己的 `emoji` 與 `codename`。

**後果不是難看，是說謊** —— 畫面說「🎯 隊長規劃中」，
實際在規劃的是 **🧠 軍師**；三個成員各有 🤖 💻 🧪，畫面全部套一個 ⚔️。

> 🔴 與 `GEMINI_MODEL` 三處硬編是同一個形狀：**設定改了，那幾處不會變。**
> 而且同一個畫面出現兩種稱呼法（階段 2 的標題套 ⚔️、逐一更新也套 ⚔️
> 但成員名字沒帶 emoji）。

### 🔴 拆成兩個出口：`stage_label` 與 `member_label`

第一版我做成單一個 `agent_label()`，統籌階段也換成 leader 自己的 emoji
（`🧠 軍師規劃中`）。**使用者指出那少掉了一層區分**，改成兩個出口：

| 出口 | 回什麼 | 標的是 |
|---|---|---|
| `stage_label(leader)` | `🎯 軍師` | **流程走到哪** ← 🎯 固定 |
| `member_label(agent)` | `🤖 博士` | **誰在做** ← 各自的 emoji |

```
🎯 軍師規劃中⋯                          ← 階段標記
🤖 博士　💻 工匠　🧪 稽查 執行中⋯（3 人）   ← 成員身份
🎯 軍師彙整中⋯                          ← 階段標記
```

> 💡 **兩者是不同的東西，所以是兩個函式而不是同一個加參數。**
> 全部換成 emoji + 名字的話，四行變成同一種語彙 ——
> 看不出「這是誰」與「這是階段」的差別。

四處收斂結果：

| | 改前 | 改後 |
|---|---|---|
| ① 規劃 | `🎯 隊長規劃中⋯` | `🎯 軍師規劃中⋯` |
| ② 執行 | `⚔️ 博士、工匠、稽查 執行中⋯（3 人）` | `🤖 博士　💻 工匠　🧪 稽查 執行中⋯（3 人）` |
| ③ 逐一 | `⚔️ 稽查 ✓　工匠 ✓　⏳ 還有 1 人` | `🧪 稽查 ✓　💻 工匠 ✓　⏳ 還有 1 人` |
| ④ 彙整 | `🎯 隊長彙整中⋯` | `🎯 軍師彙整中⋯` |

**認不出的 id 回 id 本身，不補假名字**（例如 `🎯 leader-x`）——
進度訊息要能指出「是哪個 id 沒設 codename」，
那比顯示一個好看但查不到的稱呼有用。沒有 emoji 時不留前導空白。

⚠️ **`stage_label` 刻意不切 `member_label()` 的輸出字串** ——
`.split(" ")[-1]` 遇到 codename 是「大 軍師」就只拿到「軍師」，
而那種 bug 只在特定設定下出現。兩者共用 `_persona()`。

守門測試用 `ast` 只掃「**執行時會送出的字串**」（`_tick(...)` /
`on_progress(...)` 內的字面值，含 f-string 的固定片段），
不掃 prompt 與 docstring —— 那些引用舊寫法是為了說明為什麼改掉。
實測把舊寫法塞回去 → 直接指出行號與具體字串。

### 🔴 兩個 DB schema 漂移 —— 兩個不同根因，症狀都是靜默

#### ① 跨套件撞表名 → bot 的 session 完全存不進去

| 套件 | 檔案 | 表 | 欄位 |
|---|---|---|---|
| `ark_team_agent.session` | `state/sessions.db` | `sessions` | chat_id / topic_id / instance / session_id … |
| `ark_bot_agent.agent.session` | **同一個** | **同一張** | current_agent / mode / history |

`CREATE TABLE IF NOT EXISTS` **先建的贏**，另一邊之後每次讀寫都失敗
—— 而失敗被 `except` 吞掉只留一行 log。

實測 paddy（同時消費兩個套件）：team daemon 先建表 →
**bot 的 session 一筆都沒寫成功過**。症狀是「`/agents` 選的模式重啟後
回到預設、對話歷史不見」，**沒有任何錯誤畫面**。

改成 `state/ark_bot_sessions.db`。**沒有資料遷移問題** ——
舊檔從頭到尾就是 team-agent 的表。

> 🔴 **兩個獨立套件不能靠「記得不要撞」來避免衝突** ——
> 它們互不知道對方存在。共用 `state/` 的檔名一律帶套件前綴。

#### ② 同套件內加欄位，既有 DB 沒跟上 → 路由決策從沒寫成功過

```
實際:  trace_id, timestamp, user_input_summary, target_agent,
       route_path, reply_summary, success
期望:  ↑ 再加上 ark_decision
```

`ark_decision` 是後來才加進 `CREATE TABLE` 的，而那句對**已存在的表
完全不做事** → 每次 `update_trace_decision()` 都
`no such column: ark_decision`，同樣被 `except` 吞掉。

加 `_ensure_columns()`：連線時比對 `PRAGMA table_info` 並 `ALTER TABLE` 補齊。
刻意保守 —— **只加不減、不改型別、不搬資料**（刪欄位在 SQLite 需重建表，
風險遠高於留著一個沒人讀的欄位）。

> 💡 **①②是同一個家族：schema 宣告與磁碟上的表不一致，而寫入失敗是靜默的。**
> 差別只在一個是別人先建表，一個是自己改了宣告沒管既有的表。

#### 順手：`MemorySearch` 的 cwd 相對預設路徑

`db_path: str = "data/sessions.db"` —— 兩個問題：cwd 相對，
且 basename 與 team 的 `sessions.db` 相同。

🔴 **`check_package_paths` 抓不到它** —— 掃描器只掃 `Path("...")`，
**建構子的字串預設值溜過去了**。改走 `get_state_dir()`。

⚠️ 這個類目前**套件內零建構點**（ninja-bot 有自己的一份拷貝）。
**沒有移除** —— `ModeRouter.route()` 的教訓是「零呼叫點」不等於死碼。

#### 守門：`tests/test_state_file_isolation.py`（8 測試）

掃**兩個套件 `.db` 檔名集合的交集**（比 basename，因為同名不同 schema
就是下一次事故的形狀）、`_TRACE_COLUMNS` 與 `CREATE TABLE` 一致、
缺欄位會被補上、**寫進去讀得回來**、多餘欄位留著不動、migration 冪等、
`MemorySearch` 換 cwd 結果不變。

> ⚠️ **本檔第一版被自己的 docstring 誤報** —— `memory_search.py` 的說明區塊
> 引用舊檔名來解釋為什麼改掉，而我只跳過 `#` 開頭的行。
> 已改用 `ast` 排除 docstring（`check_package_paths.py` 有同名 helper，同一個理由）。
> **這正是本專案記過的那條：說明文件常示範「不該這樣寫」。**

### 測試

`tests/test_v0_11_wiring.py`（39）+ `tests/test_agent_config_mapping.py`（8）。
每一條驗的都是**副作用與實際值** —— 掃字面值與檢查符號存在都會給假綠燈
（`ModeRouter` 有 213 個測試全綠而 `route()` 從沒被呼叫過）。

### 💡 三態改造踩到三個既有測試，而它們斷言的是「表示法」不是「行為」

```python
assert tb.enabled is False           # ← 舊測試
assert tb.is_on() is False           # ← 改成這樣
```

三個都在斷言 `enabled` 這個欄位長什麼樣，而它們的 docstring 說的是
「預設關，不該打外部 HTTP」—— **那個行為在新語意下完全成立**。

其中一個更明顯：`test_team_backend_partial_config_keeps_defaults` 斷言
`tb.url` 為 truthy，也就是**它在保護「url 有預設值」這個 bug**。
docstring 的原意是「部分設定不會把其他欄位清成 `None`」，那件事仍然成立
（`url == ""` 不是 `None`），所以改成驗那個，並順手釘住「有 url 才會真的開」。

> 🔴 **斷言表示法而不是行為，會在正確的改動上變紅。**
> 判斷「該改測試還是該改實作」的方法是讀 docstring 的意圖：
> 意圖仍成立就改斷言，意圖被破壞才是實作錯了。
> 這三個都屬前者 —— 而如果只看紅燈就把 `is_on()` 改回布林，
> 就會把 nana 的 port 那個地雷放回去。

## [0.10.1] — 2026-08-21 · 未發布

**0.10.0 加的「啟動通知 admin 私訊」從進來就沒有送出過一次。**

```python
admin_chat_id = os.getenv("ADMIN_CHAT_ID") or os.getenv("TELEGRAM_CHAT_ID")
if admin_chat_id:
    await application.bot.send_message(...)
```

實測 nana-bot：

```
.env  ADMIN_CHAT_ID / TELEGRAM_CHAT_ID / ADMIN_CHAT_IDS  → 三個都沒設
      → admin_chat_id 恆為 None → if 永遠不成立 → 一次都沒送出
bot.yaml  access.admin_chat_ids = [937896656]            → 有設，但沒人讀
```

### 🔴 同一族的第三個實例

| 版本 | 位置 | 症狀 |
|---|---|---|
| 0.6.0 | `handlers._ALLOWED_USERS` | 設了白名單卻對所有人開放 |
| 0.9.0 | `tg_handlers.ADMIN_CHAT_IDS` | 同上 + 審批卡片送不出去 |
| **0.10.1** | `bot/main.py` 啟動通知 | **功能從沒執行過** |

三次都是同一句話：**設定寫了，但那條路只讀 env。**
而這次是 `check_package_paths.py` 的 `# env-ok:` 機制抓到的 ——
0.9.0 把已知的 15 處標成豁免之後，掃描器安靜了；
08-20 新增的這 2 處就成了唯一的紅點。

> 💡 **這就是「不製造常駐假警報」的報酬**：噪音清乾淨之後，
> 新出現的問題才會顯眼。若當時把那 15 處留著紅，這 2 處會被淹掉。

### 改法

收件人改為 `access.get_access().admin_ids()`，env 降為覆寫：

- 支援**多個 admin**（原本只能一個）
- env 是非數字時**退回 `bot.yaml`** —— 舊寫法會拿垃圾值去 `int()` 再被
  `except: pass` 吞掉，看起來和「沒有收件人」一模一樣
- 🔴 **`except: pass` 改成 `log.warning`** —— 送失敗與沒有收件人看起來相同，
  正是這個功能壞掉沒人發現的原因之一
- 沒有 admin 時 `log.info` 明講「去設 `bot.yaml` 的 `access.admin_chat_ids`」

### ⚠️ 順手修掉一個會炸的東西

`bot/main.py` **原本沒有 logger** —— 我加 `log.warning` 時 `ast.parse` 通過
（語法完全合法），但執行到那行會 `NameError`。已補 `log = logging.getLogger(__name__)`。

> 這與 1.4.4 那次「把方法插進 `__init__` 中間吞掉後面三行」同一類：
> **語法檢查通過不代表結構正確。** 抓到它的是「載入模組後檢查 `hasattr(m, 'log')`」，
> 不是語法檢查。

### 測試

`tests/test_startup_notify.py`（10 測試），**驗副作用不驗函式存在** ——
`send_message` 有沒有被呼叫、送給誰。涵蓋：env 沒設走 `bot.yaml`、多 admin、
`user` 等級不打擾、env 覆寫、舊 env 名相容、垃圾值退回、沒有 admin 要留 log、
送失敗要 warning 且不阻塞啟動、指令選單仍要註冊。

### 📌 版號說明

`packages/ark-bot-agent/pyproject.toml` 在本次之前**已被提前 bump 到 0.10.1**
（無對應程式碼、無 CHANGELOG 條目、`__init__.py` 仍是 0.10.0）——
兩個守門測試都抓到那個分岔。本版補上內容與 `__init__.py`，兩處現在一致。

## [0.10.0] — 2026-08-20

### Added
- **TG 回覆分級** — ≤200 字直發、201-4000 摘要+存 raw、>4000 摘要+報告+HTML 附件
- **summarizer.py** — 從長文提取 ≤200 字摘要（結論+數字+下一步）
- **長回覆自動入知識庫** — `_save_reply_to_raw()` 自動存 knowledge/raw/（含 frontmatter）
- **/restart 指令** — admin only + Inline Button 確認 + restart.flag + graceful shutdown

### Fixed
- **超時明確回饋** — 不再靜默，回覆「⏱️ 處理超時」或具體錯誤
- **截斷統一** — 移除三層疊加截斷，output_filter 改為 50000 防爆（交給分級邏輯）

### Changed
- 非常駐 agent timeout 180→120s
- BOT_COMMANDS menu 加入 /restart

## [0.9.1] — 2026-08-20

### Fixed
- **卡住明確回饋** — 超時回覆「⏱️ 處理超時」、失敗回覆具體錯誤（非靜默）
- **截斷統一** — 移除 handlers 的 `[-3000:]` 硬截斷，統一在 output_filter 4000 字截一次
- **截斷提示優化** — 改為「📎 完整回覆已存為報告」引導看全文

### Changed
- 非常駐 agent timeout 180→120s
- _MAX_REPLY_CHARS 3500→4000

## [0.9.0] — 2026-08-19 · 未發布

環境變數當設定用的 16 處，逐一看過後：**10 處是正確模式，6 處是真的沒接上。**

### 🔴 `memory/tg_handlers.py` 的白名單只讀 env —— 0.6.0 那個缺陷的孿生兄弟

```python
ADMIN_CHAT_IDS: set[int] = set()
_admin_env = os.getenv("ADMIN_CHAT_IDS", "")     # ← 完全不讀 bot.yaml
```

實測（nana-bot，`.env` 沒有 `ADMIN_CHAT_IDS`）：

```
bot.yaml  access.admin_chat_ids  = [937896656]
access（0.6.0 修好的那條路）      = [937896656]   ✅
tg_handlers.ADMIN_CHAT_IDS        = set()         ❌
```

而三個使用點都是「**空集合 = 放行**」：

| 使用點 | 後果 |
|---|---|
| `_require_auth`（`/recall` `/skills` `/consolidate`） | 對所有人開放 |
| `callback_skill_approval` | **任何人可核准 skill** —— 等於把程式碼裝進 agent |
| `send_approval_card` | 沒有收件人 → **審批卡片從來送不出去** |

> 🔴 0.6.0 修的是 `handlers._ALLOWED_USERS`，**同一個錯法在另一個模組活了下來**。
> 「設了白名單卻對所有人開放，而且看起來生效了」—— 這類缺陷的嚴重度要看
> **沒接上之後，讀設定的人會做出什麼錯誤決定**。

修法：白名單唯一來源是 `access.get_access()`，並在 `AccessControl` 加
`admin_ids()`（「要主動推訊息給管理者」的正規出口）。

**兩個使用點刻意用不同的判準**：

| 使用點 | 判準 | 理由 |
|---|---|---|
| `_require_auth` | `is_allowed()` | 查詢自己的記憶，不是特權操作 → **維持既有語意**（空白名單 = 不限制） |
| `callback_skill_approval` | `is_admin()` | 🔴 **刻意收緊** —— 空白名單時擋下 |

> 「空白名單 = 任何人都能核准 skill」不是有人刻意依賴的行為，是碰巧存在的缺陷。
> 空白名單時的錯誤訊息會直接告訴你去設 `bot.yaml`，而不是只說「沒權限」。

### 🟠 `GEMINI_MODEL` 三處硬編，而它們寫的版本和設定不一樣

```
bot.yaml  llm.model = gemini-3.5-flash
三處硬編             = gemini-2.5-flash      ← 改設定完全沒有用
```

`llm/gemini_chat.py` · `wiki/engine.py` · `wiki/search/layer3_rerank.py`。
**同一事實寫三處，而且三處都和第四處（設定）不一致。**

而出口 `model_for(purpose)` **0.5.2 就存在了** —— 只是這三處沒接上。
補 `wiki` / `rerank` 兩個用途到 `KNOWN_PURPOSES`，不併進 `chat`，
理由正是該模組自己的設計前提：**成本結構不同**
（rerank 輸出極短該用最便宜的；wiki 問答要組答案並標來源，比 rerank 重比 chat 輕）。

### 🟠 `coordinator/a2a/server.py` 的 AgentCard 名稱硬編 `"ai-bot"`

`BotConfig.name` 存在，但 AgentCard 不看它 → 設了 `name` 也不會出現在對外的卡片上。

### ⚪ `skills/internal/openai_chat.py` 是合理的例外

它**固定走 OpenAI**，與 `llm.provider` 無關 —— 不能用 `model_for()`／`llm.model`，
那可能是 Gemini 的模型名（預設 `gemini-3.5-flash`），送給 OpenAI 會直接失敗。
標記 `# env-ok:` 並把理由寫在程式碼裡。

### 🔴 修完之後：讓掃描器分得出「正確模式」與「沒接上」

剩下 10 處是 `os.getenv(X) or <config>` —— **config 為主、env 覆寫，這是對的**。
純 regex 分不出它和 `os.getenv(X, "字面值")` 的差別，就這樣放著等於
**製造常駐假警報，而那會讓維運開始習慣性忽略整個檢查**。

`check_package_paths.py` 新增行尾標記 `# env-ok: <理由>`，三個設計決定：

1. **沒寫理由就不算豁免** —— 否則它會變成 `# noqa`：貼上去就安靜，沒人知道為什麼。
2. **豁免項目會被列出來並計數**，不是靜默跳過 ——
   靜默忽略正是 `modes.chat.tools` 那個錯名躺很久的原因。
3. **不自動放行「看起來有 fallback」的寫法** ——
   `os.getenv(X, "字面值")` 的 fallback 也是 fallback，而那正是要抓的東西。

結果：`--strict src/` 從 **16 處命中** → **0 命中 ｜ 15 處豁免（每個都有理由）**。

### 測試

`tests/test_env_config_convergence.py`（17 測試）。除了驗六處修正，
**也驗豁免機制自己**：空理由不得豁免、豁免不得被靜默吞掉、
不傳 `suppressed` 時要相容。豁免機制本身就是新的風險面。

### ⚠️ 部署注意

nana-bot 跑的是 site-packages 的 wheel（**不是** editable），所以這些修正
要 build wheel → 安裝 → **重啟服務**才會生效。
`memory/tg_handlers.py` 的授權變更屬行為改變，重啟後才套用。

## [0.8.0] — 2026-08-19 · ✅ Release

⚔️ 團隊模式可改走外部 `ark-team-agent` 的 headless API（`team_backend`），
**預設關**，本地 `team_flow` 仍是降級路徑。含 0.7.2 未發布的路徑常數收斂。

### 🔴 Fixed — `team_backend` 從加進來就無法啟用，而啟用會讓團隊模式崩潰

`config.load()` 明確組裝了每一個嵌套區塊（`server` / `modes` / `llm` /
`backend` / `report` / `features` / `access`）—— **獨漏 `team_backend`**。

而 `team_backend` 是 `BotConfig` 的已知欄位，所以 `_pick()` 會把 `bot.yaml`
的**原始 dict 直接塞進去**，不轉成 dataclass。同一個漏改因此有兩種症狀，
兩種都不像壞掉：

| bot.yaml 有沒有寫 `team_backend:` | 結果 |
|---|---|
| 沒寫 | 拿到 dataclass 預設值 `enabled=False` → **功能永遠無法啟用** |
| 寫了 | `cfg.team_backend` 是 **dict** → `router.run_team` 碰 `.enabled` 就 `AttributeError` → **⚔️ 團隊模式壞掉** |

```python
>>> C.load(bot_path=...).team_backend        # bot.yaml 裡寫了 team_backend
{'enabled': True, 'url': 'http://localhost:33333', 'timeout': 60}   # ← dict
>>> _.enabled
AttributeError: 'dict' object has no attribute 'enabled'
```

> 🔴 **「宣告了但沒接上」的又一個實例**（同 `access` 段從來沒人讀、
> `modes.chat.tools` 白名單沒接上），而這次的變體是**設了才會壞** ——
> 讀設定的人以為自己啟用了一個功能，實際得到的是一個崩潰。

守門用掃描而非具名：`test_config_wiring.py` 比對「`BotConfig` 的嵌套 dataclass
欄位」與「`load()` 裡明確組裝的關鍵字」，**新增任何區塊而忘了接上都會紅**。
（已實測：把那一行拿掉，4 個測試立刻失敗。）

### Added

- `agent/team_client.py` — `TeamClient` / `DispatchResult`
  - `status` / `retryable` / `http_status` / `reply_text()`，**刻意不實作 `__str__`**
    （形狀對齊 `CliResult`，理由同 0.5.0：`str | None` 讓呼叫端分不出失敗種類）
  - 傳輸層與服務端失敗分開歸類：`unreachable` / `timeout` / `bad_response`
    vs `unknown_agent` / `send_failed` / `queued` / `internal_error`
  - 相容舊服務端（`ark_team_agent` ≤ 1.4.7 的回應沒有 `status` 欄位）：
    由 HTTP code 推導，**不留 `unknown`** —— 否則「200 帶 error」那種會被判成
    可重試 → 無意義的重試風暴
- `config.TeamBackendConfig` 接上 `bot.yaml`（見上），並補進 `__all__`
- 測試 +34：`test_team_client.py` 18 · `test_team_backend_route.py` 10 ·
  `test_config_wiring.py` 6（`team_backend` 這條路徑原本**零涵蓋**）

### Changed

- `router.run_team`：`status == "queued"` **不降級走本地 `team_flow`**

  訊息已進 team-agent 的 overflow 佇列（agent 未就緒），會在就緒後投遞 ——
  再跑一次本地就是**同一件事做兩遍**。這是唯一「失敗但不該降級」的分類，
  `ok=False` 分不出來，只能靠 `status`。

- 降級的 log 改帶 `status` / `retryable` / `http`，
  只印 `error` 字串分不出「服務沒開」還是「agent 沒回」

### 相依

需要 `ark_team_agent` **≥ 1.4.8** —— `/api/v1/dispatch` 與 `/api/v1/agents`
在 1.4.7 及之前分別回「HTTP 200 帶 error」與 HTTP 500（都呼叫了 `Daemon`
不存在的方法）。1.4.7 之後的服務端才有 `status` 欄位與正確的 HTTP status code。

## [0.7.2] — 2026-08-19 · 未發布（內容併入 0.8.0）

### Fixed

- 🔴 **「少一層 `shared`」的第四個實例，而且這次是功能性缺陷**

  `skills/internal/wiki_distill` 把蒸餾產出寫進 `knowledge/wiki`（1 篇），
  而引擎的索引與搜尋讀 `knowledge/shared/wiki`（21 篇）
  → **蒸餾出來的知識永遠不會被索引，也搜不到**。四次全記錄：

  | 次 | 位置 | 症狀 |
  |:-:|---|---|
  | 1 | `layer1_bm25` 讀 `knowledge/.index` | BM25 索引讀不到，靜默 fallback 到 layer0 |
  | 2 | `indexer` 與 `layer1` 讀寫不一致 | 同上 |
  | 3 | `server/main.py` 的 index-status 手組路徑 | rebuild 回 ok 但 status 永遠 `not_built` |
  | 4 | `wiki_distill` 寫 `knowledge/wiki` | **蒸餾產出不進索引、搜不到** |

- 💡 **常數只是半個 bug —— prompt 自己也講錯路徑。**
  `_DISTILL_PROMPT` 寫著「存放到 `knowledge/wiki/`」。只改常數沒有用，
  **模型照 prompt 走**。docstring / description / prompt 裡共五處都要改。
  這類「同一事實寫兩處」的變體，其中一處是**給模型看的自然語言** ——
  掃程式碼掃不到它。
- TG 長訊息在 4096 字元被硬截斷 → `md_to_tg_html()` 移除硬截斷，
  新增 `_split_html_safe()` 在段落邊界切割，`ProgressStack.complete()`
  改為 edit 首段 + send 後續段

### Changed

- `paths.py` 新增正規存取器 `get_shared_dir` / `get_shared_wiki_dir` /
  `get_shared_raw_dir` / `get_index_dir` / `get_shared_tasks_dir`，
  **手組路徑的 13 處全部收斂**

  > 💡 四次同型 bug 的共同來源是「每個人自己組路徑」。
  > 修第 N 次都只是修那一次；**加單一出口才是修這一類。**

- `memory/indexer` 與 `chat_history` 的 `DB_PATH` 改延遲計算 ——
  DB 是**寫入目標**，凍結錯的家等於資料寫到別處

### Removed

- `wiki_distill._RAW_DIR` / `_LOG_PATH` / `_INDEX_PATH` ——
  死碼且路徑也錯，留著只會被人照抄

### 守門

`PATH_CONSTANTS` 原本是**手寫清單**（新增的常數不會被涵蓋）→ 改成 `ast`
自動發現所有模組層路徑常數，並驗六件事（都在 home 底下／不准手組 shared／
wiki 概念的常數互相一致／索引讀寫一致／prompt 與常數一致／死碼不復活）。

**掃描一跑就抓到另外 7 處人工盤點漏掉的** —— 它們在**函式內**不是模組常數，
其中 `server/main.py` 有 4 處同樣的 `shared/wiki`。

> 💡 人工盤點會漏，而且漏的地方沒有規律。凡是同型錯誤已出現三次以上，
> 就該寫掃描而不是再盤一次。

測試 317 → 349。

## [0.7.1] — 2026-08-18 · ✅ Release

### Security

- TG 檔案傳送白名單 —— `sender.ALLOWED_FILE_TYPES` + `send_report` 檢查，
  `handlers.send_document` 雙重守門。**預設只允許 `.html` / `.md` 報告檔**

## [0.7.0] — 2026-08-18 · ✅ Release

用 skill 產文件、再用 skill 跑 plan（`ark-project-planning` →
`ark-superpowers` → `ark-spec-executor`）：5 Phase / 20 任務 / 15 AC 全過。
測試 283 → 317。

### 🔴 Fixed — 套件在群組會回應每一則訊息

`handlers.handle_message()` **零處 `chat.type` 判斷** —— 所有訊息走同一條路。
私訊時是對的，**群組時 bot 變成噪音來源，群組立刻不能用**。

新增 `access.group_policy`（`mention_only` 預設 / `all` / `off`）。
未回覆的群組訊息**仍寫記憶** —— 群組是資訊來源，不回話是禮貌，不聽是浪費。

> 🔴 **預設值改了群組行為，這是刻意的**：相容性保護的是
> 「有人刻意依賴的行為」，不是「碰巧存在的缺陷」。一個把每則訊息都回應的
> bot 沒有人會刻意想要。私訊行為完全不變，`group_policy: all` 就是回滾出口。

### Added

- **R3 `state_dir` 可設定** —— `ARK_BOT_STATE_DIR`（env）→ `backend.state_dir`
  （設定）→ `get_home()/state`（預設，與 0.6.0 同值）
  - **相對路徑相對 `get_home()` 不是 cwd**（相對 cwd 是本套件修掉最多次的
    bug 類型，M2 平移時 37 處，症狀是「從別的目錄啟動就讀不到且不報錯」）
  - 指定路徑建不起來時 error log + 沿用預設，不讓 Bot 起不來 ——
    但一定要講，否則部署者不知道資料實際去哪了
  - **套件不搬檔**，既有資料要自己搬
- **R5 `chat_trace.stats()`** + `GET /api/chat/stats?days=7` ——
  純 SQL GROUP BY，不加新儲存：**資料已經在收，缺的是出口不是資料**
  （加第二套會讓兩份數字對不上）

### Removed

- `skills/internal/reaction_manager.py`（**獨立 commit**，以便單獨 revert）

  實跑確認是死碼：`ReactionManager` 不是 `BaseSkill` 子類 → 連 skill 都
  註冊不了（`get_registry()` 的 12 個 skill 裡沒有 reaction），全套件零引用。
  而 `handlers.py` 有 12 處 `_set_reaction()` 在服役 —— 同一件事兩套實作。

  > 選「刪」而非「接上」：handlers 那 12 處是**服役中且驗證過的**，
  > skill 版從未被呼叫。**把活的換成後者是用已知可用換未知。**

### 💡 量測推翻直覺：最大的成本是我們自己寫的字

每則 chat 訊息 10774 字元 ≈ 2693 tokens（「你好」也一樣）：

| 區塊 | 字元 | 佔比 |
|---|---:|---:|
| **`tone_rules`（硬編字串）** | **3333** | **34%** |
| `memory_md` | 1511 | 15% |
| `soul_and_other` | 1371 | 14% |
| `recent_md` | 1311 | 13% |

**硬編字串合計 53%，資料類只 47%** —— 直覺會以為記憶與知識庫是大頭。
結論：值得優化但不在本版做，且精簡 `tone_rules` 必須先有 A/B 實測
（語氣是 UX，退化了比多花 token 糟）。

> 🔴 **量測工具自己也要能被檢查。** 初版憑印象寫區塊標記（`## 搜尋`），
> 實際是 `## 知識庫與搜尋` → 找不到標記就歸到前一段 → `recent_md` 量到 2799
> 而程式明明截在 1500。修法是加「各段加總 == 總數」的帳目核對。
> **量測工具算錯比沒有量測更糟 —— 它會讓人去優化錯的地方。**

### ⚠️ 發版踩點：刪檔案的版本，build 前必須清 `build/`

第一次 build 出來的 wheel **還含著已刪除的 `reaction_manager.py`** ——
`packages/ark-bot-agent/build/lib/` 是 setuptools 的快取，
**它不會清掉已從原始碼刪除的檔案**。

症狀：磁碟已刪、git 已 commit、測試全綠，**但產物裡還在**。
是「裝前驗內容」那一步抓到的。

> 💡 與 sdist 空殼、wheel 版號分岔同一形狀：**build 成功不等於產物正確。**

## [0.6.0] — 2026-08-18 · ✅ Release

### 🔴 Security — 設了白名單卻對所有人開放

`handlers._ALLOWED_USERS` **只從環境變數 `ADMIN_CHAT_IDS` 讀**，
`bot.yaml` 的 `access` 段**從來沒有任何人讀**。而 `.env` 沒設時它是空集合
→「空＝不限制」：

```
access.admin_chat_ids     = [937896656]
handlers._ALLOWED_USERS   = set()
_is_authorized(123456789) → True        ← 陌生人通過
```

註解寫著「bot.yaml 的 `access.admin_chat_ids`；env 可覆寫」，
**但實作只有 env 那一半**。

> 🔴 **「宣告了但沒接上」最危險的形狀**：不是功能沒生效，而是
> **安全設定沒生效而且看起來生效了**。判斷這類缺陷的嚴重度，要看
> 「沒接上之後，讀設定的人會做出什麼錯誤決定」。

### Added

- **R7 `AccessControl`** —— 靜態（`bot.yaml` + env **疊加**）+ 動態
  （`state/access.json`）兩層
  - 舊格式 `admin_chat_ids: [int]` 載入時正規化成 `{id: level}`，
    執行期只有一種形狀（兩種並存＝每個讀取點判斷兩次）
  - **不回寫 `bot.yaml`** —— 那會動到人寫的註解與排版
  - 空白名單維持「不限制」語意，但 `is_admin()` 回 `False`
    （「不限制使用」不等於「都是管理員」）
  - **只能移除動態加入的** —— 移除靜態項目會讓下次重啟又出現，
    那種假成功比拒絕更糟
  - 壞掉的 state 檔不讓 Bot 起不來，**但一定要 log**
    （靜默忽略等於白名單被悄悄清空）
- R8 verify hook、UI helpers（`bot/keyboard.py` / `bot/pagination.py`）

### 明確不取（連理由一起記，避免日後重問）

| 項目 | 為什麼 |
|---|---|
| `conversation/router.py` 的 LLM 意圖分流 | **與核心前提衝突** —— 套件設計是「模式由使用者選，不靠 AI 猜」 |
| `tool_tracker.py` | 靠 `feed(text)` 從輸出流解析進度，而套件用 `communicate()` 等完整輸出 —— **沒有 stream 就沒得 feed** |
| `gateway/` OpenAI 相容層 | 另一個產品面 |
| `task_store` / `event_log` | 團隊套件的職責 |

### 🔴 事故：14 個新檔案從 0.5.0 到 0.6.0 從來沒進版控

`git commit -- <pathspec>` **只提交已追蹤的修改** ——
未追蹤的新檔案被**靜默排除**，而且 commit 回報成功。

漏掉的是 `result.py` / `errors.py` / `model_router.py` / `access.py` /
`keyboard.py` / `pagination.py` + 7 個測試檔 —— 意思是**任何人 clone
都拿到 import 就炸的套件**。

為什麼沒被發現：wheel 從工作區 build（產物正常）、測試跑工作區（全綠）、
`git status` 有 `??` 但每次只看「有沒有夾帶別人的」。

> 🔴 **pathspec 防的是夾帶，不防漏帶。**
> 正解：`git add <明確檔案>` 再 `git commit -- <同一批>`，
> 並核對 `git show --stat` 的檔案數與預期相符。
> **「commit 成功」不等於「該進去的都進去了」。**

## [0.5.2] — 2026-08-18 · 未發布

### Added

- **R6 多模型分工** —— `llm.models` 以**用途**為 key
  （`chat` / `plan` / `verify` / `synthesis`），未指定則 fallback 到 `llm.model`
  （不可移除，既有部署還在用）

  > key 用「用途」而非節點名：節點名與工作流拓樸綁死，而且無法判斷
  > 未知 key 是打錯還是新節點 —— 用途名可以 warning。

- `agent_cli_chat(fresh=True)` —— 跳過常駐進程走 one-shot 路徑（不帶 `--resume`）

### ⚠️ R5 的前提在本套件不成立，量測後改了做法

ninja 的「21k → 686 tokens（−97%）」來自 Claude CLI 帶完整工具 schema
的規劃呼叫。本套件實測：

| 項目 | 字元數 |
|---|---|
| plan prompt | 654 ← spec 原本要精簡這個 |
| `describe_agents` | 203 |
| **leader steering** | **3901 ← 真正的量都在這** |

精簡成員描述最多省 ~100 字元，在 4555 總量裡是噪音。

> 🔴 **照 spec 字面做會改錯地方，而且看起來有做事。**
> 別的專案的數字不會自動適用 —— 抄結論前先量自己的。

真正的成本源是 `leader` 的 `skip_resume: false` → 每次規劃都
`kiro-cli chat --resume`，**重播整段累積歷史**（`~/.kiro` 已 68MB）。
而規劃階段的 prompt 是自足的，歷史純粹是成本。

> 💡 **改成 `fresh=True` 是零代價的省** —— `AgentProcess._execute()`
> 本來就每則訊息 spawn 新 process，所謂「常駐」是佇列與設定，不是長駐 REPL。

## [0.5.1] — 2026-08-18 · 未發布

### Added — `AgentPool` 四個缺口（都是「原本沒有」而不是「原本有別的值」）

| 缺口 | 原本 | 風險 |
|---|---|---|
| queue 上限 | `asyncio.Queue()` **無上限** | 無背壓，只會無聲堆積 |
| idle 回收 | 無 | 8 個常駐 kiro-cli 永不釋放 |
| health loop | 無 | 進程死了沒人知道 |
| crash 計數 | 無 | 崩潰迴圈偵測不到 |

> 不是新設計 —— `ark_team_agent` 的 lazy spawn + `idle_timeout_minutes`
> 在 slot 上驗證過（13 instances、閒置 30 分回收）。
> **兩個套件不共用程式碼，但該共用經驗。**

三個關鍵實作決策：

1. **queue 溢出丟最舊，且被丟的那則必須收到結果。**
   最新的是使用者剛送的，丟它使用者立刻感覺到沒反應；最舊的本來就快逾時了。
   🔴 但被丟的 future 不能懸掛 —— 那會變成永不完成的 await，**比丟掉更糟**。
2. **crash 用時間窗不用累計。** 累計值會讓長時間運行的健康服務
   慢慢累積到誤觸上限。過期紀錄就地清除，避免列表無限長。
3. **health loop 出錯不停迴圈。** 停掉等於回收與偵測都沒了，而且無聲。

- **per-agent timeout（R4）** —— 實測 data 37s / report 50s / **ai-dev 123s**，
  全體共用一個值必然有一邊不對。解析順序：呼叫端 → per-agent → 全域。
  **未設定時 120s，與 0.4.3 完全同值**（有測試釘住，升版不得改變既有行為）。

## [0.5.0] — 2026-08-18 · 未發布 · 🔴 破壞性

### Changed — `agent_cli_chat()` 從 `str | None` 改成 `CliResult`

補的是「失敗看不出來」。呼叫端分不出四種情況，而**失敗是用回傳值表示的，
`try/except` 抓不到**：

| 實際發生 | 舊回傳 |
|---|---|
| agent 回了空字串 | `None` |
| 子進程崩潰 | `None` |
| 逾時 | `None` |
| CLI 根本沒安裝 | `None` |

前三種該重試，第四種重試一百次也不會好（那是設定問題）。

### Added

- `agent/result.py` — `CliResult`（`status` / `output` / `error` /
  `retryable` / `elapsed_ms` / `backend` / `model`）。
  `partial` 用於「逾時但已有部分輸出」—— **部分結果比沒結果好**
- `agent/errors.py` — `classify()` / `is_retryable()`。
  `not_found` 與 `permission` **不可重試**
- `backend.timeout` / `backend.retries` / `AgentDef.timeout`

### 三個刻意的決策

1. **不叫 `AgentResult`** —— `llm/agent_loop.py` 已有同名類別，
   撞名會讓 import 拿到哪個變成靠順序決定
2. **不讓它當字串用** —— 不實作 `__str__` 回 output，漏改的呼叫點要立刻
   `TypeError` 而不是靜默送出物件字串。（`__repr__` 含欄位值是刻意保留的，log 需要）
3. **`retries` 預設 0** —— 0.4.3 沒有重試，開新行為要由設定明示，
   不能因為升版就默默多花時間

### 💡 錯誤分類的順序有意義

例外先判、`returncode` **最後** —— **逾時被 kill 的子進程也有非 0 returncode**，
先看 returncode 會把逾時誤分類成 crash，而兩者處置不同
（逾時調 timeout、crash 查 agent 本身）。有測試釘住。

8 個呼叫點全部更新，並用 `ast` 掃描守門（`test_all_call_sites_handle_cliresult`）。

## [0.4.3] — 2026-08-18 · ✅ Release

### 🔴 Fixed — `modes.chat.tools` 從裝飾變成真的白名單

這份設定**沒有任何人讀**（`agent_loop` 拿 `reg.all_schemas()`，七個工具全開），
而設定只列四個，其中 `wiki_search` 這個名字**在系統裡根本不存在**
（實際叫 `search_wiki`）。

比「設定沒生效」嚴重的是，多出來的三個**有寫入副作用**：

```
save_to_wiki   → filepath.write_text() 直接寫共用知識庫
save_memory    → 寫記憶
execute_skill  → 執行 skill（含 execute_code）
```

> 🔴 **讀設定的人會以為快答模式只能查不能寫 —— 這比沒有這個設定更糟。**
> 「設定存在但沒接上」的一般形狀是功能沒生效；這一個的形狀是**假的保證**。

修法：`agent_loop(allowed_tools=...)`（`None` = 全部，保留給 agent 模式的
fallback）；快答的三個呼叫點傳 `modes.chat.tools`；預設白名單訂正成
`search_wiki` 並**只留唯讀 + 派工**，三個寫入工具改 opt-in
（專案規則是「wiki 只由 ingest 產出、禁止手寫」，預設開與那條衝突）。

`_pick_schemas()` 三種可見性：未知名稱 warning 並列出可用名稱、
排除了哪些用 info 講、全部被過濾掉時 ERROR 並退回全部
（**安靜地變成無工具模式比報錯更糟**）。

### ⚠️ 實測：白名單不是安全邊界

叫娜娜寫知識庫 → 她**改用 `dispatch_to_agent` 請管家寫，管家走 kiro-cli
有全部權限，寫入成功**（頁數 54→55，已清殘留並重建索引）。

> 💡 **限制了工具，沒限制「叫別人用工具」。** 要真的擋住寫入，
> 得限制被派工那一端。白名單解決的是「設定說謊」，不是「模型不能寫」——
> **兩件事別混。** 這條寫進 CHANGELOG，不假裝已經安全。

## [0.4.2] — 2026-08-18 · 未發布（Release notes 併入 v0.4.3）

M5.6 實測（31 條結構 + 三條真實路由）挖出四個「宣告了但沒接上」。
**四個都不拋例外、不寫 log。**

### 🔴 ① M4 報告管線在 TG 上從來沒執行過

`deliver_report()` 只被 `ModeRouter._maybe_report()` 呼叫，而
**`ModeRouter.route()` 沒有任何呼叫點** —— TG 走的是 `handlers.py` 自己那套
Path 1/2/3。整個 M4 里程碑的產出在正式路徑上等於不存在
（`output/reports/` 的檔案全是手動產物）。

發現方法：`grep "\.route("` → **0 個呼叫點**。

修法：handlers 兩條收尾各呼叫 `_attach_report()`，而它**借用 router 的
`_maybe_report`** —— 不自己重寫判斷，否則門檻／標題規則／降級訊息會變兩份。
TG 送檔器由 handlers 注入（`DocumentSender` 協議，套件不綁 telegram）。

> 🔴 **同一功能有兩套實作時，「新的那套有測試」不代表跑的是新的那套。**
> `ModeRouter` 有 213 個測試全綠，而它從來沒被呼叫過。

### 🔴 ② 清洗與報告的順序反了 —— 畫面乾淨、存檔是髒的

輸出過濾原本只掛在 TG 層，而報告管線在 router 裡 → MD/HTML **存進工具雜訊**
（`I will run the following command: … (using tool: shell)`），
**但畫面上看起來乾淨**，因為 TG 那層之後又清了一次。

新增 `clean_cli_output()`，在 `run_agent` / `run_team` 建 `Reply` 時就清；
報告也移到 3000 字截斷**之前**（截斷是 TG 訊息限制，不該切掉存檔內容）。

> 💡 **有兩個出口時（畫面／存檔），只驗畫面會漏掉另一個。**

### 🔴 ③ BM25 索引路徑的第三個實例

`server/main.py` 的 index-status **自己組路徑**（`get_knowledge_dir()/".index"`，
少一層 `shared`）→ **rebuild 明明回 `ok`、status 永遠回 `not_built`**。
改用 `indexer.INDEX_DIR`。**「模組不自己組路徑」要貫徹到端點層。**

順帶：BM25 索引**本來就沒建過**（`.index/` 不存在），四層搜尋一直只有 layer0。
機制修好了但沒人啟用 —— 已建（54 頁）。

### 🟠 ④ `modes.chat.max_iterations` 三處硬編 `5`

**剛好與設定同值，所以「沒接上」看不出來。** 三處改讀設定。

### 相依

- `google-genai` 下限提到 **`>=2.0`** —— 1.2.0 的 `Part` 沒有
  `thought_signature` 屬性 → `getattr` 永遠回 None → 送不回去 →
  第二輪 `400 INVALID_ARGUMENT`。**快答模式只要用工具就掛，而症狀看起來
  像 API 或金鑰問題，不像相依版本。**

### 測試

新增 `test_report_wiring.py` —— 驗**可達性與副作用**不驗函式存在。

> 🔴 這批缺陷的共同形狀：**`import` 掃得到、單元測試會過、模組載入得了，
> 就是沒有人呼叫。** 掃字面值與檢查符號存在都擋不住，只有行為測試釘得住。

## [0.4.1] — 2026-08-18 · ✅ Release

### Removed
- `/recall` 指令已移除（import + BOT_COMMANDS + handler 三處清除）

## [0.4.0] — 2026-08-17

### Changed
- 輸出過濾重構為三層管線（`output_filter.py`）
  - Layer 1: `clean_output()` — 移除 ANSI/XML/spinner/工具UI/shell/docker/pytest 噪音（80+ 規則）
  - Layer 2: `extract_final_reply()` — `> ` 精確提取 + [DONE] + reply() + fallback + 3500 字硬截斷
  - Layer 3: `_format_telegram()` — 既有 Markdown → TG HTML 轉換
- 移除舊單層 `_clean_output()` 與 `_TOOL_LINE_PREFIXES`

### Added
- `src/ark_bot_agent/bot/output_filter.py` — 獨立可測試的過濾模組
- `tests/ark_bot_agent/test_output_filter.py` — 25 個單元測試

## [0.3.1] — 2026-08-17

### Fixed
- 設定層路徑解析修正

## [0.3.0] — 2026-08-15

### Added
- 設定層、三模式路由、報告管線初版
- MD-first 報告流程（MD → HTML → TG）

## [0.2.0] — 2026-08-17

### Added
- 初始發布：路徑層、三模式框架、Wiki/Memory 系統
