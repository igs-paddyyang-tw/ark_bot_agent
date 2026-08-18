# Changelog

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

## [0.3.0] — 2026-08-15

### Added
- 設定層、三模式路由、報告管線初版
- MD-first 報告流程（MD → HTML → TG）

## [0.2.0] — 2026-08-10

### Added
- 初始發布：路徑層、三模式框架、Wiki/Memory 系統
