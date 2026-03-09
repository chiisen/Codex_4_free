# Project Rules

## 範圍
- 這份 `AGENTS.md` 遵循 Codex 官方指引，讓 Codex 依照從專案根目錄向下合併的規則載入本專案內的 Instructions (Codex 處理 `AGENTS.override.md`、`AGENTS.md`，並自上而下合併)。

## Markdown 放置
- 所有新增與變更的 `.md` 檔案必須建立於 `docs/` 目錄下，避免直接留存於專案根目錄或其他資料夾。
- 當某份 `.md` 在其他地方被發現，應主動將它移入 `docs/`，並更新所有引用路徑。

## 特殊情況
- 若某個工具/套件強制要求 `.md` 在特定位置，先告知團隊並在這份規則中新增備註；建立例外前請先確認是否可重配置。
