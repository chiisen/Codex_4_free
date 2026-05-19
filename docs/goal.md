
# 參考 https://www.aivi.fyi//llms/codex-goal 整理成一份 .md 檔案，務必簡單明瞭



# Codex `/goal` 命令完全指南

## 一、什麼是 `/goal`？

`/goal` 是 OpenAI 在 **Codex CLI 0.128.0**（2026 年 4 月 30 日發布）中新增的命令，它不是普通的提示詞模板，而是一套完整的**目標生命週期管理機制**。[[1](https://www.aivi.fyi//llms/codex-goal)]

### 核心組成

1. **持久化層**：將目標作為獨立狀態儲存，支援狀態機（`active` / `paused` / `achieved` / `unmet` / `budget_limited`）
2. **App-server RPC**：提供 `thread/goal/{get, set, clear}` 三個介面
3. **模型工具**：`get_goal`、`create_goal`、`update_goal` 三個工具
4. **運行時延續 + TUI**：每輪空閒時自動注入延續提示詞，讓模型決定下一步[[1](https://www.aivi.fyi//llms/codex-goal)]

### 關鍵特性

- **無人值守執行**：給定目標後，Codex 會自動一輪接一輪推進
- **跨會話恢復**：目標狀態獨立於對話歷史，可跨會話持續
- **內建 Ralph Loop**：OpenAI 總裁 Greg Brockman 稱之為 "built in Ralph loop++"[[1](https://www.aivi.fyi//llms/codex-goal)]

---

## 二、解決的核心問題

### 1. 目標持久化
- `/compact` 壓縮對話不會破壞 goal 狀態
- 關閉終端後可透過 `codex resume <id>` 繼續
- 支援多天跨度的長任務[[1](https://www.aivi.fyi//llms/codex-goal)]

⚠️ **已知問題**：若 `/compact` 發生在模型調用執行中間，延續提示詞可能不會重新注入（Issue #19910）[[1](https://www.aivi.fyi//llms/codex-goal)]

### 2. 內建完成審計
每輪空閒後自動注入 `continuation.md` 提示詞，要求：
- 將目標重述為具體交付物或成功標準
- 建立**提示詞到產物**的清單
- 檢查真實證據（檔案、命令輸出、測試結果等）
- **不接受代理信號**（測試通過、清單填滿等不能單獨作為完成依據）
- **將不確定視作未達成**[[1](https://www.aivi.fyi//llms/codex-goal)]

### 3. Token 預算軟停止
- 達到 token 上限時不會粗暴中斷
- 注入 `budget_limit.md` 讓模型收尾
- 總結進度、列出剩餘工作、給出建議後停止[[1](https://www.aivi.fyi//llms/codex-goal)]

### 4. 完整生命週期控制

| 命令 | 作用 |
|------|------|
| `/goal <objective>` | 建立或替換目標 |
| `/goal` | 顯示目標摘要 |
| `/goal pause` | 暫停延續 |
| `/goal resume` | 恢復暫停的目標 |
| `/goal clear` | 清空目標 |

[[1](https://www.aivi.fyi//llms/codex-goal)]

---

## 三、適用場景

### ✅ 適合使用

- **批量任務**：批量修 bug、重命名、生成測試、補文檔
- **覆蓋式任務**：QA 完整流程、型別嚴格化、文檔同步
- **明確工程任務**：模組遷移、功能移植、按規格實作
- **長程探索**：程式碼考古、架構梳理
- **規格驅動開發**：配合 OpenSpec 等工具[[1](https://www.aivi.fyi//llms/codex-goal)]

### ❌ 不適合使用

- **單輪小任務**：直接用普通 prompt 更快
- **探索性任務**：無法定義「完成長什麼樣」
- **需要人工決策**：產品決策、商業取捨、UX 偏好
- **破壞性操作**：刪除資料庫、不可逆遷移
- **快速原型階段**：幾分鐘就能完成的原型
- **⚠️ Plan 模式下**：在 `/plan` 模式下 `/goal` 不會自動延續（Issue #20656）[[1](https://www.aivi.fyi//llms/codex-goal)]

---

## 四、啟用方式

### 方法 1：修改設定檔

編輯 `~/.codex/config.toml`：

```toml
[features]
goals = true
collaboration_modes = true  # 可選，用於 /plan 配合
```

儲存後**重啟 Codex**。[[1](https://www.aivi.fyi//llms/codex-goal)]

### 方法 2：讓 Codex 自動修改

在 Codex 中輸入：

```
請幫我開啟 Codex 0.128 新增的 /goal 命令。
配置文件位置：~/.codex/config.toml
需要在 [features] 段下加上 goals = true。
```

### 驗證

輸入 `/` 檢查是否出現 `/goal` 命令，或直接輸入 `/goal` 查看狀態。[[1](https://www.aivi.fyi//llms/codex-goal)]

---

## 五、提示詞撰寫心法

### 五段式黃金模板

```
/goal <一句話描述目標>

Scope: <作用範圍 — 改哪些檔案/子系統，其他不要碰>

Constraints:
- <硬性約束 1 — 例：不修改資料庫 schema>
- <硬性約束 2 — 例：保持現有公開 API 不變>
- <硬性約束 3>

Done when:
1. <可驗證的產物 1 — 引用具體檔案名或命令>
2. <可驗證的產物 2>
3. <可驗證的產物 3>

Stop if:
- <機械可識別的停止條件 1 — 例：需要新依賴>
- <機械可識別的停止條件 2>

Use a token budget of <N> tokens for this goal.
```

[[1](https://www.aivi.fyi//llms/codex-goal)]

### 各段要點

- **Objective**：避開虛詞（全部、所有、徹底、improve、optimize）
- **Scope**：畫出明確邊界，防止擴散
- **Constraints**：可機械識別的硬性規則
- **Done when**：引用具體檔案路徑或命令
- **Stop if**：防止鑽牛角尖或越界
- **Token budget**：必給，提供軟停止機制[[1](https://www.aivi.fyi//llms/codex-goal)]

### 實例

```
/goal 把 src/data/words.json 裡的詞庫擴展到 1000 個唯一詞條。

Scope: 只改 src/data/words.json，其他檔案不動。

Constraints:
- 詞條 schema 保持不變（id / word / phonetic / meaning / example）
- 不允許重複詞條（以 word 欄位為準去重）
- 只能用真實的、常見的英語單字

Done when:
1. words.json 包含恰好 1000 個唯一詞條
2. 所有詞條 schema 校驗通過（用 tools/validate.js 跑一遍）
3. 在終端輸出最終詞條數和檔案大小

Stop if:
- 需要修改 words.json 以外的任何檔案
- 需要新增 npm 依賴
- 出現 schema 校驗失敗超過 3 次

Use a token budget of 80000 tokens.
```

[[1](https://www.aivi.fyi//llms/codex-goal)]

---

## 六、三種典型工作流

### A. 直接使用 `/goal`（中等任務）

1. 輸入完整的五段式 `/goal` 提示詞
2. 回車執行
3. 隨時用 `/goal`（無參數）查看進度
4. 用 `/goal pause`、`/goal resume`、`/goal clear` 控制[[1](https://www.aivi.fyi//llms/codex-goal)]

**適用**：70% 的任務，邊界清楚、能寫出五段式模板

### B. `/plan` + `/goal`（複雜任務）

1. 進入 Plan 模式（`/plan` 或 `Shift+Tab`）
2. 輸入模糊需求
3. 與 Codex 互動討論，生成完整計畫
4. **選擇「保持在 Plan 模式」**
5. 用 `Shift+Tab` 退出 Plan 模式
6. 輸入 `/goal 執行上面的開發計畫` + Done when / Stop if / token budget[[1](https://www.aivi.fyi//llms/codex-goal)]

⚠️ **關鍵**：必須先退出 Plan 模式再下 `/goal`，否則會落入 Issue #20656 的坑

### C. OpenSpec + `/goal`（規格驅動開發）

配合 OpenSpec 等工具，將規格文件直接交給 `/goal` 執行。[[1](https://www.aivi.fyi//llms/codex-goal)]

---

## 參考資料

[[1](https://www.aivi.fyi//llms/codex-goal)] https://www.aivi.fyi/llms/codex-goal



1. [🚀开发者必看！Codex /goal命令你真用对了吗？goal命令高级技巧保姆级教程，Plan模式+Spec-Driven+自研Skill，三大高级技巧组合让开发效率倍增！真正内置Ralph Loop - AI超元域的博客](https://www.aivi.fyi//llms/codex-goal)

