# ark_bot_agent

> **個體模式** AI Agent Bot 框架 —— 一個 Telegram Bot、三種對話模式、
> Wiki/Memory 子系統、MD-first 報告管線。
> 消費端只需要一份設定檔和 **8 行 `start.py`**。

[![Python](https://img.shields.io/badge/Python-≥3.11-blue?logo=python)](https://www.python.org/)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

---

## 🚨 這裡沒有 `src/`

| 你要做什麼 | 去哪裡 |
|-----------|--------|
| **改程式碼** | `paddy-team-agent/src/ark_bot_agent/`（唯一開發源） |
| 看發版紀錄 | 本 repo 的 [`CHANGELOG.md`](CHANGELOG.md) |
| 下載 wheel | 本 repo 的 Releases |

比照 `ark_team_agent` 的做法（開發源與發布通道分離）。
**在這裡改東西不會進入任何 wheel。**

---

## 定位：兩個套件，兩種模式

| 套件 | 模式 | 一句話 | 適用 |
|------|------|--------|------|
| [`ark_team_agent`](https://github.com/igs-paddyyang-tw/ark_team_agent) | **團隊** | 多 agent 常駐、群組 topic 路由、決策矩陣、排程 | 一個專案養一支長期團隊 |
| **`ark_bot_agent`（本套件）** | **個體** | 單一 Bot + 三種對話模式 + 選擇性派工 | 一個人要一個什麼都能問的入口 |

**兩者互不依賴、不合併。** 選錯會很痛：團隊模式的複雜度（topic 路由、
決策矩陣、成員生命週期）對「我只想有個 bot 能問」是純負擔；
反之個體模式沒有常駐團隊與任務板。

---

## 核心設計：模式由使用者選，不靠 AI 猜

同一句話在不同模式下**該有不同的處理成本**。讓人決定比讓模型判斷準確，
而且省掉一次意圖分類的 LLM 呼叫。

TG 面板只有三顆按鈕：

```
　💬 快答 │ ✅ 👑 管家 │ 　⚔️ 團隊
```

| 模式 | 引擎 | 費用 | 用在什麼時候 |
|------|------|------|------------|
| 💬 **快答** | Gemini ReAct loop | API 計費 | 閒聊、查知識庫、上網、不確定找誰 |
| 👑 **管家** | 單一 agent on kiro-cli | **零 API 費用** | 明確知道找誰，要對方的 skill 與 memory |
| ⚔️ **團隊** | leader + Python 三階段工作流 | 零 API 費用 | 需要拆解、跨職能 |

`@代號 訊息` 的直送**不受模式影響** —— 永遠直接找那個人。

### 💬 快答 —— Gemini ReAct

七個工具的 ReAct 迴圈，由 `modes.chat.tools` 白名單決定給哪些：

| 工具 | 預設 | 說明 |
|------|:----:|------|
| `search_wiki` | ✅ | 四層搜尋管線（見下） |
| `recall_memory` | ✅ | SQLite FTS5 記憶檢索 |
| `web_search` | ✅ | 外部搜尋 |
| `dispatch_to_agent` | ✅ | 把這題轉給某個職人 |
| `save_to_wiki` | ⬚ | **寫**共用知識庫 |
| `save_memory` | ⬚ | **寫**記憶 |
| `execute_skill` | ⬚ | 執行 skill（含 `execute_code`） |

三個寫入工具**預設關**，要開得在 `bot.yaml` 明寫。

> ⚠️ **白名單是範圍控制，不是安全邊界。** 拿掉 `save_to_wiki` 之後，
> 模型仍可能改用 `dispatch_to_agent` 請一個有全部權限的 agent 去寫。
> 要真的擋住寫入，得限制**被派工那一端**。

### 👑 管家 —— kiro-cli 就是大腦

不經 LLM provider。套件只負責 spawn 與 IO，agent 自己讀
`.kiro/steering/`（SOUL / TEAM / MEMORY）、自己的 skills、wiki 與 memory。

**刻意不在 prompt 注入「你現在在管家模式」** —— 這個模式沒有替代行為，
注入只是干擾。（其他兩個模式必須注入，因為 agent 讀不到設定檔，
不明寫它不知道自己該拆解或該收斂。）

#### 執行結果是型別化的

`agent_cli_chat()` 回 **`CliResult`** 而不是 `str | None`：

```python
res = await agent_cli_chat("查一下服務狀態", agent_id="admin")
res.status       # success | partial | error
res.retryable    # not_found / permission → False（那是設定問題，重試無用）
res.elapsed_ms   # 實際量測
res.reply_text() # 可直接顯示；error 時回 ""
```

> 🔴 舊版回 `str | None` —— 呼叫端分不出「空回覆／崩潰／逾時／CLI 沒安裝」，
> 而**失敗是用回傳值表示的，`try/except` 抓不到**。
> `partial` 用於「逾時但已讀到部分輸出」—— 部分結果比沒結果好。

### ⚔️ 團隊 —— 三階段真派工

```
① 規劃   leader 判斷 self / delegate，輸出 JSON
② 執行   Python 讀計畫，平行呼叫成員（Semaphore 上限 3）
③ 收斂   成員回覆交回 leader 整合成一份答覆
```

ReAct 迴圈在 **Python 層**，不需要 function calling、不需要自建 MCP server，
每一步可觀測可攔截。

**執行階段用 `asyncio.wait(FIRST_COMPLETED)` 逐一收，不用 `gather`** ——
實測三人派工：`data` 37s / `report` 50s / `ai-dev` **123s**。
用 `gather` 會讓前兩人陪最慢的等 73 秒，而使用者看不到任何進展。

到 `stage_deadline` 就用**已完成的**去收斂，逾期者 cancel 並在答覆末尾明講。
進度每完成一個就更新：`⚔️ 碼哥 ✓　數據 ✓　⏳ 還有 1 人`。

> 💡 **部分結果比沒結果好**，也比讓人等到硬超時好。

收斂之後可選一個 **verify 階段**（`modes.team.verify`，**預設關**）——
把答覆交回去檢查有沒有明確錯誤或重大遺漏。有意見**附在後面，不改寫答覆本身**
（驗證者不是作者），而 verify 失敗**不影響已完成的協作**。

> 這裡刻意做成 hook 而非第四階段 —— 不用它的人不該吃到階段管理的複雜度。
> 用哪個模型由 `llm.models.verify` 決定，**不寫死廠商**。

降級策略（任何一步失敗都不讓使用者看到錯誤）：

| 情況 | 行為 |
|------|------|
| leader 沒吐 JSON | 把原文當答案回（通常已經有用） |
| 派給非成員 | 過濾 + log，退回規劃說明 |
| 成員全部無回應 | 回規劃 + 一句「成員都沒回應」 |
| **只有一人回覆** | **省掉收斂那次呼叫**，直接組裝 |
| 收斂無回應 | 自己組裝，不讓已完成的工作白費 |

---

## 子系統

121 個 `.py`、11 個子套件。

| 子套件 | 檔數 | 做什麼 |
|--------|:----:|--------|
| `skills/` | 33 | Skill 註冊表 + 12 個內建 skill |
| `llm/` | 19 | ReAct loop、provider 抽象、tool registry、context builder、壓縮 |
| `memory/` | 12 | daily log、recall（FTS5）、consolidate、chat trace、推薦 |
| `coordinator/` | 11 | A2A 跨 agent 協調（TaskGraph + SharedMemory，選配） |
| `bot/` | 8 | TG handlers、三模式 router、team_flow、輸出過濾、進度堆疊 |
| `wiki/` | 8 | 四層搜尋、indexer、engine、lint |
| `agent/` | 6 | kiro-cli 進程管理、常駐佇列、session |
| `report/` | 4 | MD-first 報告管線（writer / renderer / sender） |
| `access.py` | 1 | 權限等級 + 執行期增刪 + 持久化（跨層，故放根層） |
| `tools/` | 4 | 維運工具 |
| `conversation/` | 2 | 對話狀態 |
| `server/` | 2 | FastAPI + 6 個 Web UI 頁面 |

### Wiki —— 四層搜尋管線

| 層 | 做什麼 | 缺依賴時 |
|----|--------|---------|
| **L0** 精確 | metadata 查找 + 子字串掃描 | 保底層，永遠可用 |
| **L1** BM25 | 持久化索引（`bm25s` + `jieba`） | graceful fallback 到 L0 |
| **L2** 混合 | 語意向量 + 圖譜擴散 + RRF 融合 | 同上 |
| **L3** Rerank | LLM 重排 | 選配 |

查詢順序：**私有 → 共用 → 外部搜尋，不可跳層。**

> ⚠️ **索引要建**。`POST /api/v1/wiki/rebuild-index`。
> 沒建的話搜尋一直只有 L0，而且**不會報錯**。

### Memory

- `daily/` 每日對話 log（自動寫入）
- `recall` SQLite **FTS5** 全文檢索
- `consolidate` 週期性壓縮（> 14 天歸檔）
- `chat_trace` 每則訊息的路由決策軌跡（`/api/chat/traces`）

### Agent 執行池

`kiro-cli` 的呼叫由一個池管理，每個 agent 一條佇列。

| 機制 | 說明 |
|------|------|
| **佇列上限** | `backend.max_queue`（預設 32）。滿了**丟最舊的**，而**被丟的那則會收到 `queue overflow` 結果** —— 懸掛的 future 是永不完成的 await，比丟掉更糟 |
| **idle 回收** | `backend.idle_timeout`（預設 1800s）。`persistent: true` 的 agent 不受此限。**回收 ≠ 永久消失**，下次呼叫叫得回來 |
| **health loop** | 週期檢查存活；死亡記 crash 並寫 log。迴圈本身出錯**不會停掉**（停掉等於回收與偵測都沒了，而且無聲） |
| **crash 計數** | **時間窗**（`backend.crash_window`）不是累計 —— 累計值會讓長時間運行的健康服務慢慢累積到誤觸上限 |
| **per-agent 逾時** | `AgentDef.timeout`。實測 `data` 37s / `report` 50s / **`ai-dev` 123s**，全體共用一個值必然有一邊不對 |
| **重試** | 只對 `retryable` 的錯誤，次數由 `backend.retries` 決定（**預設 0**）。重試一定留 log —— 靜默重試會讓「慢」看起來像「當」 |

`pool_status()` 回 `total` / `alive` / `persistent` / `crashes` / `idle`。

> 💡 「常駐」指的是**佇列與設定**，不是長駐 REPL ——
> `_execute()` 每則訊息都 spawn 一個新 process。
> 所以「不帶對話歷史的一次性呼叫」（`fresh=True`）**不會多付冷啟成本**。

### 存取控制

兩層：**靜態**（`bot.yaml` 的 `access` + 環境變數，**疊加**不是取代）
+ **動態**（`state/access.json`，執行期增刪）。

```yaml
access:
  admin_chat_ids: [937896656]      # 舊格式，仍支援（載入時正規化）
  users:                            # 新格式
    111222333: user
```

- **不回寫 `bot.yaml`** —— 那會動到人寫的註解與排版
- 舊格式**載入時正規化**成 `{id: level}`，執行期只有一種形狀
- 空白名單 = 不限制（開發模式），但 **`is_admin()` 回 `False`** ——
  「不限制使用」不等於「都是管理員」
- 只能移除動態加入的 —— 移除靜態項目會讓下次重啟又出現，**那種假成功比拒絕更糟**

> 🔴 **0.6.0 之前這整段設定沒有任何人讀** —— `handlers` 只從環境變數取白名單，
> 而 `.env` 沒設時它是空集合 → 「空＝不限制」→
> **設了白名單卻對所有人開放**。註解寫著會讀 `bot.yaml`，實作只有 env 那一半。
>
> 這是「宣告了但沒接上」最危險的形狀：**不是功能沒生效，而是安全設定沒生效
> 而且看起來生效了。**

### 報告管線 —— MD first

三種模式的分析產出**一律**走同一條：

```
MD（給 AI 讀、可 diff、進版控）→ HTML（給人看、自帶 CSS）→ TG 附件
```

- **MD 是 source of truth** —— 只有它寫入失敗才拋例外；
  HTML 與送檔失敗只反映在 `ReportResult`，並由 `user_note()` **把降級講出來**
- HTML 自帶 CSS + `prefers-color-scheme`，不依賴外部資源
- 送檔用 `DocumentSender` **Protocol** 注入 —— 套件不綁 `telegram`
- **短回覆不走管線**（`report.min_chars`，預設 150）——
  一句結論產一份 HTML 只是噪音
- 標題取使用者訊息前 30 字，**刻意不叫 LLM 取** —— 那是一次額外往返與費用

---

## 安裝

```bash
# 最小
pip install ark_bot_agent-0.4.3-py3-none-any.whl

# 含 Wiki 搜尋優化 + 內建 skill 與報告管線的依賴（建議）
pip install "ark-bot-agent[all] @ file:///path/to/ark_bot_agent-0.4.3-py3-none-any.whl"
```

| extra | 內容 |
|-------|------|
| `search` | `bm25s` · `jieba` · `PyStemmer`（缺了 wiki 搜尋降到 L0） |
| `skills` | `openai` · `beautifulsoup4` · `jinja2` · `feedparser` · `apscheduler` · `markdown` |
| `all` | 以上全部 |

> ⚠️ **`google-genai` 下限是 `>=2.0`，不可降。** 1.x 的 `Part` 沒有
> `thought_signature` 屬性，而 Gemini 要求 function call 回送時必須帶 ——
> 結果是 💬 快答模式**只要命中任何工具就回 400**，而症狀看起來像 API 或金鑰問題。

---

## 消費端要準備什麼

### `start.py`

```python
from ark_bot_agent import run_bot

run_bot(skills=["skills"])      # skills= 是選配：注入消費端自己的 skill 套件
```

### 設定檔 —— 各自單一職責，不合併

| 檔案 | 職責 | 必要性 |
|------|------|:------:|
| **`agents.yaml`** | **有誰**（agent 定義、代號、工作目錄、可否派工、`group_members`） | 🔴 **哨兵** |
| `bot.yaml` | **怎麼跑**（server / modes / llm / backend / report / features / access） | 選配 |
| `scheduler.yaml` | **何時**（排程） | 選配 |
| `.env` | **只放機密**（`TELEGRAM_BOT_TOKEN` / `GEMINI_API_KEY`） | 看 Tier |

**只有哨兵是必要的** —— 缺 `bot.yaml` 全走預設，不該讓服務起不來。
未知欄位**忽略但寫 log**，舊 yaml 留著已移除的欄位時服務仍要能起來。

`bot.yaml` 範例（節錄）：

```yaml
server:
  port: 8088                # ← port 的唯一來源，顯示與實際綁定都從這裡展開

modes:
  default: agent            # 新 session 的預設模式
  default_agent: admin      # 👑 管家的預設對象
  team_leader: leader
  chat:
    max_iterations: 5
    tools: [search_wiki, recall_memory, web_search, dispatch_to_agent]
  team:
    stage_deadline: 100     # 執行階段整體上限（秒）
    member_timeout: 150     # 單一成員硬超時
    max_concurrency: 3
    max_assignments: 3
    member_reply_cap: 2500  # 單一成員回覆進收斂 prompt 的字數上限

llm:
  model: gemini-3.5-flash   # 未指定用途時的 fallback（不可移除）
  models:                   # 依**用途**分工，key 打錯會 warning
    plan: cheap-model       #   規劃：輸出短 JSON，用便宜快的
    verify: strong-model    #   驗證：要挑得出問題，用強的

backend:
  timeout: 120              # 全域逾時（per-agent 可用 AgentDef.timeout 覆蓋）
  retries: 0                # 只對 retryable 的錯誤重試
  max_queue: 32
  idle_timeout: 1800        # 0 = 不回收
  crash_window: 600

report:
  enabled: true
  md_dir: output/reports
  min_chars: 150            # 短於此直接回文字，不走管線

access:
  admin_chat_ids: [937896656]
```

### 目錄

```
your-project/
├── start.py            8 行
├── agents.yaml         ← 哨兵
├── bot.yaml
├── .env
├── agents/<id>-agent/  各 agent 的 .kiro/steering/（SOUL 等）+ memory + knowledge
├── knowledge/shared/   共用知識庫（wiki / raw / .index）
├── templates/          選配 —— 同名檔會覆蓋套件內建的 Web UI
└── output/reports/     報告產出
```

---

## Tier 分級 —— 缺憑證不該讓服務起不來

| Tier | 內容 | 需要 |
|:----:|------|------|
| 0 | Prompts + Skills + Wiki | 永遠可用 |
| 1 | Telegram Bot | `TELEGRAM_BOT_TOKEN` |
| 2 | Gemini AI + RAG | `GEMINI_API_KEY` |
| 3 | Agent 分身常駐 | kiro-cli / claude / agy |

少一個 Tier 就少一組功能，其餘照跑 —— **部署可以分段驗收**（先跑 Tier 0/3，token 之後補）。

啟動橫幅把每一項算出來，不寫死：

```
Tier 0: ✅ Prompts + Skills + Wiki
Tier 1: ✅ Telegram Bot
Tier 2: ✅ Gemini AI + RAG
Tier 3: ✅ Agent 分身常駐（backend: kiro）

⚙️  設定: agents.yaml, bot.yaml　⬚ 未提供（走預設）: scheduler.yaml
📦 Skills: IDE 60 | 內建 12 | 消費端 3
📚 Wiki:   shared 21 | raw 7 | agents 33
🔍 Lint:   ✅ 0 issues
🧠 SOUL:   ✅ 已載入
🧠 Agents: 9 定義 | 可派工 8
📝 Report: ✅ output/reports
```

---

## 路徑解析：哨兵搜尋，不數 `.parent`

套件化最大的技術障礙不是搬檔案，是路徑。平移時實測有
**24 處 `parents[N]`**（N 值 2/3 混用）與 **37 處硬編相對路徑**。

```python
get_home()   # ARK_BOT_HOME → 往上找 agents.yaml → cwd
```

14 個衍生出口全部集中在 `paths.py`（`get_knowledge_dir` / `get_agents_dir` /
`get_steering` / `get_templates_dir` …），**其他模組不自己組路徑**。

> 🔴 `Path(__file__).resolve().parents[N]` 定位專案根**必錯且不報錯** ——
> 套件安裝後 `__file__` 在 site-packages，層數必然改變。
> 而且 `N` 值不一致本身就是警訊：它證明那寫法依賴檔案層數。
>
> 實際踩過三次同型 bug（BM25 索引路徑少一層 `shared`），
> 症狀都是「搜尋安靜地變差」或「status 永遠回 not_built」，
> 不拋例外、不寫 log。**「模組不自己組路徑」要貫徹到端點層。**

檢查目前解析結果：

```bash
python -m ark_bot_agent paths
```

兩個刻意的設計選擇：

- `get_steering()` **不自動建目錄** —— 缺人格檔要讓呼叫端知道，
  自動建空目錄會讓「讀不到人格」看起來像「檔案是空的」
- `get_templates_dir()` **消費端優先** —— 要客製 Web UI 只需放同名檔案，不必 fork

---

## 擴充點

```python
from ark_bot_agent import create_bot
from ark_bot_agent.bot.router import ModeRouter

class MyRouter(ModeRouter):
    async def run_chat(self, ctx):
        ...                      # 只覆寫要改的那一個模式

bot = create_bot(
    router=MyRouter(),           # 自訂路由策略
    skills=["myapp.skills"],     # 注入自己的 skill 套件
    tools=[my_react_tool],       # 注入 ReAct 工具
    on_startup=[warm_cache],     # 生命週期 hook
)
bot.run()
```

| 擴充點 | 怎麼做 |
|--------|--------|
| 換路由策略 | 繼承 `ModeRouter`，覆寫 `route()` 或任一 `run_*()` |
| 加 skill | `create_bot(skills=[...])` → 進**共用** registry |
| 加 ReAct 工具 | `create_bot(tools=[...])` |
| 客製 Web UI | 消費端 `templates/` 放同名檔案 |
| 換訊息來源 | `MessageContext` 不吃 `telegram.Update` —— Web UI / CLI / 測試同一條路由 |
| 換送檔方式 | `DocumentSender` Protocol |
| 多模型分工 | `llm.models` 依用途指定（`plan` / `verify` / `chat` / `synthesis`） |
| 權限模型 | `AccessControl` 兩層（靜態設定 + 執行期增刪） |

---

## Web UI 與 API

| 頁面 | 路徑 |
|------|------|
| Chat / Dashboard / Wiki / Graph / Builder / API Docs | `/` `/admin` `/wiki` `/graph` `/builder` `/api-docs` |

**健康檢查是 `/health`**（`ark_team_agent` 是 `/api/health`，兩者不同）。

主要 API：`/api/v1/chat` · `/api/v1/skills` · `/api/v1/wiki/{query,pages,lint,rebuild-index,index-status}`
· `/api/v1/memory/{recall,daily,consolidate}` · `/api/chat/traces`

---

## 設計原則（都是踩出來的）

1. **同一事實只寫一處** —— port、版號、路徑、成員清單。
   寫兩處必然分岔，差別只在多久（`__version__` 與 pyproject 分岔過、
   啟動訊息印 8000 而實際綁 8088）。
2. **設定要嘛真的接上，要嘛刪掉** —— 「宣告了但沒接上」比沒有這個設定更糟，
   因為讀設定的人會在錯誤假設上做決定。最壞的形狀是**假的保證**
   （工具白名單看起來限制了寫入，實際七個全開）。
3. **靜默失效要有 log** —— 這個套件的缺陷有固定形狀：
   **不拋例外、不寫 log，只讓行為安靜地錯**。
   import 不到可以回 0，但一定要留一行。
4. **失敗要降級不要中斷** —— 報告失敗不影響對話回覆、
   一個 skill 壞掉不影響服務啟動、成員逾時就用已完成的收斂。
   **但降級必須講出來**（`ReportResult.user_note()`）。
5. **清單類邏輯只能有一個來源** —— agent 查詢一律走 `config.py`，
   註冊表是全域事實（`get_registry()`），不該一個呼叫點一份。
6. **驗可達性不驗符號存在** —— 有一整個里程碑的功能寫好了、測試全綠、
   模組載入得了，**就是沒有人呼叫**。這種只有行為測試釘得住。

---

## 版權

Copyright (c) 2026 paddyyang — [MIT License](LICENSE)
