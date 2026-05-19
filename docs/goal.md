# Codex `/goal` 命令完全指南

## 出處

本文整理自 AI超元域部落格（2026-05-05 更新）：

- [Codex /goal 命令進階教程（Plan + Spec-Driven + 自研 Skill）](https://www.aivi.fyi/llms/codex-goal)

以下「工具與命令速查」將原文散落各章的資源集中於此，正文不再重複標註出處。

---

## 工具與命令速查

### Codex 內建（需先啟用 `goals = true`）

**設定檔** `~/.codex/config.toml`：

```toml
[features]
goals = true
collaboration_modes = true  # 可選，配合 /plan
```

**TUI 斜線命令**

- `/goal <objective>` — 建立或替換目標
- `/goal` — 顯示目標摘要（狀態、耗時、token 用量）
- `/goal pause` / `/goal resume` / `/goal clear` — 暫停、恢復、清空
- `/plan` — 先規劃再執行（須退出 Plan 模式後才能讓 `/goal` 自動延續）
- `/compact` — 壓縮對話；長任務中避免手動觸發（見下方踩坑）

**CLI**

- `codex resume <id>` — 跨會話恢復 thread 與 goal 狀態

**模型工具**（由 Codex 內部調用，使用者不直接執行）

- `get_goal`、`create_goal`、`update_goal`

**App-server RPC**

- `thread/goal/get`、`thread/goal/set`、`thread/goal/clear`

**運行時提示模板**（自動注入，路徑見 Codex 原始碼 `codex-rs/core/templates/goals/`）

- `continuation.md` — 每輪空閒後的完成審計與延續
- `budget_limit.md` — Token 預算用盡時的軟停止收尾

**目標狀態**

- `active` / `paused` / `achieved` / `unmet` / `budget_limited`（UI 有時顯示 `pursuing`、`complete`）

---

### 搭配工具：OpenSpec（規格驅動開發）

開源 SDD 工具，與 `/goal` 組成「先寫規格、再無人值守實作」工作流。

- 官網／倉庫：[Fission-AI OpenSpec](https://github.com/Fission-AI/OpenSpec)（MIT）
- 需求：Node.js 20.19.0+

```bash
npm install -g @fission-ai/openspec@latest
cd your-project && openspec init
```

- 斜線命令：`/opsx:propose <需求描述>` — 生成 `openspec/changes/<name>/`（`proposal.md`、`specs/`、`design.md`、`tasks.md`）
- 之後用五段式 `/goal` 指向該變更目錄執行；建議在 goal 開頭加 **First action**：先讀規格檔並回報摘要，確認後再動手

---

### 搭配工具：goal-prompt-builder（自研 Skill）

用來**自動生成可審計的五段式 `/goal` 提示詞**的 Claude Skill（非 OpenAI 官方）。

- 倉庫：<https://github.com/win4r/goal-prompt-builder>（MIT）
- 預設安裝位置：`~/.claude/skills/`
- 內建 7 種場景模板：refactor、SDD feature、batch、archaeology、UI audit、gatekeeper、custom
- 觸發語：例如「help me write a /goal for …」「我要用 /goal 來…」

**安裝（擇一）**

```bash
# 一行下載
curl -L -o /tmp/goal-prompt-builder.skill \
  https://github.com/win4r/goal-prompt-builder/raw/main/goal-prompt-builder.skill
mkdir -p ~/.claude/skills
unzip -o /tmp/goal-prompt-builder.skill -d ~/.claude/skills/
rm /tmp/goal-prompt-builder.skill
```

```bash
# 或克隆後軟連結
git clone https://github.com/win4r/goal-prompt-builder.git
ln -s "$(pwd)/goal-prompt-builder/goal-prompt-builder" ~/.claude/skills/goal-prompt-builder
```

亦可直接在 Codex 中描述安裝需求，由 Codex 代為下載。安裝後需**重啟客戶端**；相容性以各客戶端 Skills 文件為準（README 列有 Claude Code、Claude Desktop 等）。

---

### 三種工作流對照

| 工作流 | 組合 | 適用 |
|--------|------|------|
| A | `/goal` 直接用 | 邊界清楚的中等任務（約 70% 情境） |
| B | `/plan` → 退出 Plan → `/goal` | 需求模糊、需先討論計畫的複雜任務 |
| C | OpenSpec `/opsx:propose` → `/goal` | 規格驅動、有明確 tasks／驗收清單 |

---

### 常見踩坑（原文第八章摘要）

1. **Plan 模式下 `/goal` 不延續** — Issue #20656；須 `Shift+Tab` 退出 Plan 再下 `/goal`
2. **中途 `/compact` 丟失延續上下文** — Issue #19910；長任務勿手動 compact
3. **第一則訊息就是 `/goal`** — Issue #20792；`codex resume` 列表可能看不到；先發一句普通訊息再 `/goal`
4. **目標含「全部／徹底／improve」等虛詞** — 無法建立審計清單，易早稱完成
5. **未設 token budget** — 無軟停止，易 token 跑飛
6. **破壞性操作** — 刪庫、不可逆遷移等不宜用 `/goal`；若必用須寫進 Constraints／Stop if

---

## 一、什麼是 `/goal`？

`/goal` 是 OpenAI 在 **Codex CLI 0.128.0**（2026 年 4 月 30 日發布）中新增的命令，它不是普通的提示詞模板，而是一套完整的**目標生命週期管理機制**。

### 核心組成

1. **持久化層**：將目標作為獨立狀態儲存，支援狀態機（`active` / `paused` / `achieved` / `unmet` / `budget_limited`）
2. **App-server RPC**：提供 `thread/goal/{get, set, clear}` 三個介面
3. **模型工具**：`get_goal`、`create_goal`、`update_goal` 三個工具
4. **運行時延續 + TUI**：每輪空閒時自動注入延續提示詞，讓模型決定下一步

### 關鍵特性

- **無人值守執行**：給定目標後，Codex 會自動一輪接一輪推進
- **跨會話恢復**：目標狀態獨立於對話歷史，可跨會話持續
- **內建 Ralph Loop**：OpenAI 總裁 Greg Brockman 稱之為 "built in Ralph loop++"

---

## 二、解決的核心問題

### 1. 目標持久化
- `/compact` 壓縮對話不會破壞 goal 狀態
- 關閉終端後可透過 `codex resume <id>` 繼續
- 支援多天跨度的長任務

⚠️ **已知問題**：若 `/compact` 發生在模型調用執行中間，延續提示詞可能不會重新注入（Issue #19910）

### 2. 內建完成審計
每輪空閒後自動注入 `continuation.md` 提示詞，要求：
- 將目標重述為具體交付物或成功標準
- 建立**提示詞到產物**的清單
- 檢查真實證據（檔案、命令輸出、測試結果等）
- **不接受代理信號**（測試通過、清單填滿等不能單獨作為完成依據）
- **將不確定視作未達成**

### 3. Token 預算軟停止
- 達到 token 上限時不會粗暴中斷
- 注入 `budget_limit.md` 讓模型收尾
- 總結進度、列出剩餘工作、給出建議後停止

### 4. 完整生命週期控制

| 命令 | 作用 |
|------|------|
| `/goal <objective>` | 建立或替換目標 |
| `/goal` | 顯示目標摘要 |
| `/goal pause` | 暫停延續 |
| `/goal resume` | 恢復暫停的目標 |
| `/goal clear` | 清空目標 |


---

## 三、適用場景

### ✅ 適合使用

- **批量任務**：批量修 bug、重命名、生成測試、補文檔
- **覆蓋式任務**：QA 完整流程、型別嚴格化、文檔同步
- **明確工程任務**：模組遷移、功能移植、按規格實作
- **長程探索**：程式碼考古、架構梳理
- **規格驅動開發**：配合 OpenSpec 等工具

### ❌ 不適合使用

- **單輪小任務**：直接用普通 prompt 更快
- **探索性任務**：無法定義「完成長什麼樣」
- **需要人工決策**：產品決策、商業取捨、UX 偏好
- **破壞性操作**：刪除資料庫、不可逆遷移
- **快速原型階段**：幾分鐘就能完成的原型
- **⚠️ Plan 模式下**：在 `/plan` 模式下 `/goal` 不會自動延續（Issue #20656）

---

## 四、啟用方式

### 方法 1：修改設定檔

編輯 `~/.codex/config.toml`：

```toml
[features]
goals = true
collaboration_modes = true  # 可選，用於 /plan 配合
```

儲存後**重啟 Codex**。

### 方法 2：讓 Codex 自動修改

在 Codex 中輸入：

```
請幫我開啟 Codex 0.128 新增的 /goal 命令。
配置文件位置：~/.codex/config.toml
需要在 [features] 段下加上 goals = true。
```

### 驗證

輸入 `/` 檢查是否出現 `/goal` 命令，或直接輸入 `/goal` 查看狀態。

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


### 各段要點

- **Objective**：避開虛詞（全部、所有、徹底、improve、optimize）
- **Scope**：畫出明確邊界，防止擴散
- **Constraints**：可機械識別的硬性規則
- **Done when**：引用具體檔案路徑或命令
- **Stop if**：防止鑽牛角尖或越界
- **Token budget**：必給，提供軟停止機制

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


---

## 六、三種典型工作流

### A. 直接使用 `/goal`（中等任務）

1. 輸入完整的五段式 `/goal` 提示詞
2. 回車執行
3. 隨時用 `/goal`（無參數）查看進度
4. 用 `/goal pause`、`/goal resume`、`/goal clear` 控制

**適用**：70% 的任務，邊界清楚、能寫出五段式模板

### B. `/plan` + `/goal`（複雜任務）

1. 進入 Plan 模式（`/plan` 或 `Shift+Tab`）
2. 輸入模糊需求
3. 與 Codex 互動討論，生成完整計畫
4. **選擇「保持在 Plan 模式」**
5. 用 `Shift+Tab` 退出 Plan 模式
6. 輸入 `/goal 執行上面的開發計畫` + Done when / Stop if / token budget

⚠️ **關鍵**：必須先退出 Plan 模式再下 `/goal`，否則會落入 Issue #20656 的坑

### C. OpenSpec + `/goal`（規格驅動開發）

1. 安裝並 `openspec init`（見文首「OpenSpec」）
2. 在 Codex 輸入 `/opsx:propose <需求>` 生成規格目錄
3. 以五段式 `/goal` 指向 `openspec/changes/<name>/`，並在開頭要求先讀 `proposal.md`、`specs/`、`design.md`、`tasks.md` 後回報再實作

詳細安裝與範例提示詞見文首速查；原文完整 OpenSpec 範例見[出處連結](https://www.aivi.fyi/llms/codex-goal)。

---

## 延伸閱讀

- [goal-prompt-builder](https://github.com/win4r/goal-prompt-builder) — 五段式 `/goal` 提示詞生成 Skill
- [OpenSpec](https://github.com/Fission-AI/OpenSpec) — 規格驅動開發工具

