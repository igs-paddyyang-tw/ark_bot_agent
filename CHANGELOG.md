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
- 設定層路徑解析修正

## [0.3.0] — 2026-08-15

### Added
- 設定層、三模式路由、報告管線初版
- MD-first 報告流程（MD → HTML → TG）

## [0.2.0] — 2026-08-10

### Added
- 初始發布：路徑層、三模式框架、Wiki/Memory 系統
