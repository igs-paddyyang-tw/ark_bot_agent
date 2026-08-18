# ark_bot_agent

> **個體模式** AI Agent Bot 框架 —— Telegram + 三種對話模式 + Wiki/Memory。
> 本 repo 是**發布通道**，原始碼不在這裡。

[![Python](https://img.shields.io/badge/Python-≥3.11-blue?logo=python)](https://www.python.org/)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

---

## 🚨 這裡沒有 `src/`

| 你要做什麼 | 去哪裡 |
|-----------|--------|
| **改程式碼** | `paddy-team-agent/src/ark_bot_agent/`（唯一開發源） |
| 看發版紀錄 | 本 repo 的 `CHANGELOG.md` |
| 下載 wheel | 本 repo 的 Releases |

比照 `ark_team_agent` 的做法（開發源在 `nana-team-agent/src/`，發布通道獨立）。
**在這裡改東西不會進入任何 wheel。**

---

## 定位

| 套件 | 模式 | 一句話 |
|------|------|--------|
| [`ark_team_agent`](https://github.com/igs-paddyyang-tw/ark_team_agent) | **團隊** | 多 agent 常駐、群組 topic 路由、決策矩陣 |
| **`ark_bot_agent`（本套件）** | **個體** | 單一 Bot + 三種對話模式 + 選擇性派工 |

**兩者互不依賴，也不合併** —— 定位不同。

## 三種對話模式

模式由使用者在 TG 選（`/agents` 面板），**不靠 AI 猜** ——
同一句話在不同模式下該有不同的處理成本，讓人決定比讓模型判斷準確。

| 模式 | 引擎 | 能力來源 |
|------|------|---------|
| 💬 快答 | Gemini ReAct loop | 外部 web + 內部 wiki，需要時才派單一 agent |
| 👑 管家 | `admin-agent` on kiro-cli | 全靠 agent 本身 + 提詞 + skill + wiki + memory |
| ⚔️ 團隊 | `leader-agent` + Python 工作流 | 意圖分析 → 建工作流 → 彙整回報 |

三種模式的分析產出**一律** MD（給 AI）→ HTML（給人）→ TG。

## 安裝

```bash
pip install https://github.com/igs-paddyyang-tw/ark_bot_agent/releases/download/v0.4.1/ark_bot_agent-0.4.1-py3-none-any.whl

# 含 Wiki 搜尋優化與內建 skill 依賴
pip install "ark-bot-agent[all] @ https://github.com/igs-paddyyang-tw/ark_bot_agent/releases/download/v0.4.1/ark_bot_agent-0.4.1-py3-none-any.whl"
```

## 用法

```python
# start.py —— 零 code
from ark_bot_agent import run_bot
run_bot()
```

消費端只需要：

```
agents.yaml     # agent 定義 + bot 設定（同時是專案根的哨兵檔）
.env            # 只放機密（TG token / API key）
agents/*/       # 各 agent 的提詞（.kiro/steering/）
knowledge/      # 知識庫
```

要注入自訂邏輯時用 `create_bot(router=..., skills=..., tools=...)`。

## 狀態

**v0.4.1** —— 輸出過濾三層管線重構 + 移除 /recall 指令。三模式路由、報告管線穩定運行。

`run_bot()` / `create_bot()` 仍是 stub —— M5（消費端化）才接上啟動流程。
現在已可用：`get_config()` · `ModeRouter` · `deliver_report()` · `paths.*`。

## 版權

Copyright (c) 2026 paddyyang — MIT License
