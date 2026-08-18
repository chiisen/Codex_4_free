# AI Agent 指南：GitHub Projects v2 工作流 / AI Agent Guide: GitHub Projects v2 Workflow

> 本文件提供給 AI Agent 讀取的標準作業程序 (SOP)，用於把任務從專案文件導入並持續維護 GitHub Projects v2 看板。
> This document is a SOP for AI agents to import tasks from project docs into and maintain a GitHub Projects v2 board.

---

## 1. 適用情境 / When to Use

- 把 `TODO.md` / `docs/OPEN_QUESTIONS.md` / 其他規劃文件的待辦事項**建立為 GitHub Issues**
- 把多個 issues **加入 Projects v2 看板** 統一追蹤
- 為 issues/projects **設定自訂欄位** (Priority, Status, Sprint 等)
- 後續**持續更新**狀態、優先級、新增項目

---

## 2. 必要前置 / Prerequisites

### 2.1 確認 gh CLI 與授權
```bash
gh --version        # 需要 2.40+ 才有完整 Projects v2 支援
gh auth status      # 確認已登入
```

### 2.2 確認 Token Scopes
Projects v2 操作需要 `project` 與 `read:project` scope。缺少時提示使用者執行：
```bash
gh auth refresh -s project,read:project
```
**不要自行執行** `gh auth refresh` —— 需要使用者互動確認。

### 2.3 確認目標 owner 與 repo
```bash
gh repo view        # 確認目前 repo
gh project list --owner <owner>   # 確認現有 projects
```

---

## 3. SOP：從文件建立 Project + Issues

### 3.1 解析來源文件
讀取指定章節（例如 `TODO.md` 的「未來優化方向」），輸出結構化資料：

```yaml
categories:
  - name: <類別名稱>
    priority: P1 | P2 | P3
    estimate_hours: <int>
    subtasks:
      - <子任務描述>
      - <子任務描述>
```

### 3.2 建立 Issues（每大類 1 個 parent）
```bash
gh issue create \
  --repo <owner>/<repo> \
  --title "[Enhancement] <類別名稱>" \
  --label "enhancement" \
  --body "$(cat <<'EOF'
## 目標
<一段話>

## 子任務
- [ ] <subtask 1>
- [ ] <subtask 2>
...

## 預估工時
<X 小時>

## 參考
- 來源: <檔案 §章節>
- 優先級: 🔴/🟡/🟢 <P1/P2/P3>
EOF
)"
```

**粒度選擇**：除非使用者明確要求，**每大類 1 個 parent issue**，子任務用 checklist 內嵌。

### 3.3 建立 Projects v2 Board
```bash
gh project create \
  --owner <owner> \
  --title "<repo> Roadmap" \
  --format json
```

回傳 JSON 需記住：
- `number` (看板編號，用於後續指令)
- `id` (node_id，如 `PVT_xxx`，用於 GraphQL 與 item-edit)
- `url` (使用者瀏覽器連結)

### 3.4 把 Issues 加入 Project
```bash
gh project item-add <PROJECT_NUMBER> \
  --owner <owner> \
  --url https://github.com/<owner>/<repo>/issues/<N>
```

### 3.5 建立自訂欄位（推薦 Priority）
```bash
gh project field-create <PROJECT_NUMBER> \
  --owner <owner> \
  --name "Priority" \
  --data-type "SINGLE_SELECT" \
  --single-select-options "P1,P2,P3"
```

### 3.6 設定每個 Item 的欄位值

需要先查詢 item 與 field 的 ID：
```bash
# 列出 items（含 ID）
gh project item-list <PROJECT_NUMBER> --owner <owner> --format json

# 列出 fields（含 ID 與 option ID）
gh project field-list <PROJECT_NUMBER> --owner <owner> --format json
```

然後批次設定：
```bash
gh project item-edit \
  --id <PVTI_xxx> \
  --project-id <PVT_xxx> \
  --field-id <PVTSSF_xxx> \
  --single-select-option-id <option_id>
```

---

## 4. SOP：後續維護作業

### 4.1 新增 Issue 並加入 Project
```bash
gh issue create --repo <owner>/<repo> --title "..." --label "..." --body "..."
gh project item-add <PROJECT_NUMBER> --owner <owner> --url <issue_url>
```

### 4.2 切換 Status
預設 Status 欄位 option ID：
- Todo: `f75ad846`
- In Progress: `47fc9ee4`
- Done: `98236657`

```bash
gh project item-edit \
  --id <PVTI_xxx> \
  --project-id <PVT_xxx> \
  --field-id <PVTSSF_lAHOANeT-s4BgVxYzhaiTM0> \
  --single-select-option-id <47fc9ee4|98236657>
```

### 4.3 改 Priority
```bash
gh project item-edit \
  --id <PVTI_xxx> \
  --project-id <PVT_xxx> \
  --field-id <PVTSSF_xxx_Priority> \
  --single-select-option-id <P1|P2|P3 的 option_id>
```

### 4.4 檢視整體狀態
```bash
gh project item-list <PROJECT_NUMBER> --owner <owner> \
  --format json --jq '.items[] | "[\(.content.number)] \(.title) → \(.status)"'
```

### 4.5 封存已完成項目
```bash
gh project item-archive --id <PVTI_xxx> --project-id <PVT_xxx>
```

---

## 5. 常見陷阱 / Common Pitfalls

### ⚠️ User-level vs Org-level Project

| Owner 類型 | 出現位置 | 適合情境 |
|-----------|----------|----------|
| User (`@chiisen`) | `github.com/<user>?tab=projects` | 個人/學習專案 |
| Org (`@my-org`) | `github.com/orgs/<org>/projects` | 團隊協作 |
| **Repo-level** | **不存在** —— GitHub 不支援 | N/A |

**User-level project 不會出現在 repo 的 Projects tab**，即使執行 `gh project link` 也無效（這是 GitHub 設計限制，非 bug）。

### ⚠️ 私有 Project 對匿名訪問 404

`public: false` 的 project 對未登入或無權限的使用者會回 404，但對 owner 正常顯示。WebFetch 工具無法用 token 認證，所以拿到 404 是**預期**的；以 GraphQL/REST API 驗證為準。

### ⚠️ 平行指令

`gh project item-edit` 對同一個 item 平行執行可能 race。批次改欄位時**循序執行**或加入短暫 sleep。

### ⚠️ 不要破壞性操作

除非使用者明確要求，**不要**：
- `gh project delete`（刪除整個看板）
- `gh project item-delete`（從看板移除，issue 本身仍在）
- `gh project close`（看板關閉後變唯讀）

### ⚠️ Token Scope 不可繞過

若 `gh auth status` 顯示缺少 `project` scope，**必須**停下來請使用者執行 refresh，**不要**嘗試用其他方式 hack。

---

## 6. 完整範例 / Complete Example

完整案例請參考本專案的實際建立記錄：
- Issues: `chiisen/cc-cli-go#1` ~ `#6`
- Project: https://github.com/users/chiisen/projects/5

執行於：2026-08-14

---

## 7. 快速檢核清單 / Quick Checklist

每次協助完成 GitHub Projects 工作後，確認：

- [ ] Issues 已建立且 label 正確
- [ ] Project URL 已提供給使用者
- [ ] 所有 issues 已加入 project
- [ ] 自訂欄位已建立並指派
- [ ] Status 全部為 Todo（或依使用者要求）
- [ ] CHANGELOG.md 已更新（依 CLAUDE.md 規範）
- [ ] README 文件索引已同步（如新增主要文件）
- [ ] 回報限制（user-level 不會出現在 repo tab）

---

*最後更新：2026-08-14 — 初版建立*
