# Codex_4_free

Codex 4 free

OpenAI 正式推出 Codex 桌面版應用程式(macOS)，免費用戶也能用，並限時加倍用量。

目前 Codex 除了免費版可以體驗外，也已經支援 Windows 平台且有提供沙盒機制，讓我再次訂閱並嘗試使用它並紀錄使用當下的心得。

## Codex `/goal` 命令指南

本 repo 整理 Codex CLI **0.128.0** 起新增的 [`/goal`](docs/goal.md) 命令用法，可作為長任務、無人值守開發的參考筆記。

[`docs/goal.md`](docs/goal.md) 涵蓋：

- **是什麼**：目標生命週期管理（狀態機、跨會話恢復、內建延續迴圈），而非單次提示詞
- **解決什麼**：目標持久化、`/compact` 不破壞 goal、完成審計、Token 預算軟停止
- **怎麼用**：`config.toml` 啟用 `goals = true`、五段式提示詞模板、三種工作流（直接 `/goal`、`/plan` + `/goal`、OpenSpec + `/goal`）
- **何時用／不用**：適合批量修 bug、規格驅動實作；不適合單輪小改、需人工決策或仍在 Plan 模式時

## 近期官網更新（2026-01-09～2026-03-09）

- 2026-02-02：推出 Codex macOS 桌面 app（支援多代理並行、長任務、隔離 worktrees、diff 檢視與套用），並有免費用戶限時加倍用量的方案說明。
- 2026-03-04：Codex app Windows 版上線。
- 2026-02-05：推出 GPT-5.3-Codex（比 GPT-5.2-Codex 更強、約快 25%）。
- 2026-02-12：推出 GPT-5.3-Codex-Spark 研究預覽，主打低延遲即時編碼體驗（Cerebras）。
- 2026-03-06：推出 Codex Security 研究預覽，協助自動化應用安全檢測與修補流程。

## 參考來源

- https://openai.com/codex
- https://openai.com/index/gpt-5-3-codex/
- https://openai.com/index/gpt-5-3-codex-spark/
- https://openai.com/index/codex-security/

[用 Codex 做的網頁版的 To Do List](index.html)
