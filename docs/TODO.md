# Codex 體驗 To-do

- [x] 安裝並開啟 Codex app（macOS；若有 Windows 也確認 Windows 版可用）
- [x] 在 Codex app 開兩個 agent 並行任務，確認可切換/不掉上下文
- [x] 驗證 worktrees 與 diff 檢視（套用、丟棄、或轉成 PR）流程
- [x] 驗證 sandbox 權限提示與允許規則（例如需要提升權限的命令）
- [x] 切換到 GPT-5.3-Codex 並做一段實作，記錄速度與品質感受
- 說明：`GPT-5.3-Codex-Spark` 屬於分批開放的研究預覽，是否顯示取決於帳號/工作區資格；目前先不列為待辦項目。
- 說明：`Codex Security` 為研究預覽且分批開放；若目前看不到介面，通常是帳號/工作區尚未開通，先不列為待辦項目。

## 體驗紀錄

- 日期：
- 環境（OS / 方案）：
- 結果摘要：
- 問題與待追蹤：

### 2026-03-09（GPT-5.3-Codex 實作驗證完成）

- 日期：2026-03-09
- 環境（OS / 方案）：macOS / GPT-5.3-Codex
- 結果摘要：已完成一段實作驗證，功能交付與操作流程正常。
- 問題與待追蹤：後續可持續累積不同任務型態下的速度與品質觀察資料。

### 2026-03-09（sandbox 驗證完成）

- 日期：2026-03-09
- 環境（OS / 方案）：macOS / Codex workspace sandbox
- 結果摘要：sandbox 驗證成功，已確認允許範圍內操作可正常執行，受限操作會觸發阻擋或提權流程。
- 補充說明：
  - 讀取型指令（如 `pwd`、`ls`）可直接執行。
  - 工作區可寫入路徑可正常建立/修改檔案。
  - 超出允許範圍或受限操作時，行為符合預期（阻擋或要求權限提升）。

### 2026-03-09（worktrees 與 diff 驗證完成）

- 日期：2026-03-09
- 環境（OS / 方案）：macOS / Git worktree + diff 檢視
- 結果摘要：worktree 建立、切換、差異檢視與變更回復流程皆可正常運作。
- 驗證流程說明：
  1. 建立測試分支與 worktree：`git worktree add ../Codex_4_free-wt-test -b codex/wt-diff-test`
  2. 在新 worktree 做小幅檔案修改（例如 `README.md` 一行）。
  3. 以 `git worktree list` 確認主工作區與測試 worktree 都存在。
  4. 在測試 worktree 以 `git diff` 與 `git status` 檢視差異與修改狀態。
  5. 驗證套用/丟棄流程：`git add <file>`（套用到 stage）與 `git restore <file>`（丟棄變更）。
  6. 清理測試環境：`git worktree remove ../Codex_4_free-wt-test`，必要時 `git branch -D codex/wt-diff-test`。

### 2026-03-09（雙 agent 並行驗證完成）

- 日期：2026-03-09
- 環境（OS / 方案）：macOS / Codex app（雙 agent）
- 結果摘要：已完成多個並行任務驗證，可穩定切換 agent 且上下文維持正常，未出現任務互相污染。
- 補充說明：後續可持續以不同任務型態（讀取、修改、驗證）觀察長時段並行穩定性。
