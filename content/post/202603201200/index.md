---
title: "Speck-Kit：用 AI 驅動的規格化開發工作流"
date: 2026-03-20T22:07:07+08:00
categories: ["軟體開發", "AI"]
tags: ["Speck-Kit", "AI", "規格化", "工作流"]
draft: false
---

Speck-Kit 是一套規格化的專案管理框架，專為 AI 輔助軟體開發設計。它透過結構化的規格模板、分階段的開發流程，以及與 GitHub Copilot 深度整合的自訂 Agent，讓團隊在動手寫程式之前就能完成完整的需求釐清與技術規劃。本文以我的圖書管理系統（Library）專案為例，介紹 Speck-Kit 的核心概念與實際應用。

---

## 1. 什麼是 Speck-Kit？

Speck-Kit（目前版本 v0.3.2）是一個開發流程的「元框架」，它不產出程式碼，而是定義**如何產出程式碼**。核心理念是：

> 在實作之前，先把需求寫清楚。

它提供了以下能力：

- **結構化規格模板**：以 User Story、Acceptance Criteria、Gherkin 測試情境定義功能需求
- **分階段開發流程**：Clarify → Specify → Plan → Implement → Tasks
- **Copilot 自訂 Agent**：每個階段對應一個專屬 Agent，共 9 個
- **PowerShell 自動化腳本**：環境檢查、功能建立、計畫初始化
- **專案 Constitution**：定義不可違反的品質、測試、效能標準

---

## 2. 安裝 Spec-Kit

- 官方 GitHub：https://github.com/github/spec-kit

### 前置需求

- **作業系統**：Linux / macOS / Windows
- **Python 3.11+**：https://www.python.org/downloads/
- **Git**：https://git-scm.com/downloads
- **uv**（Python 套件管理工具）：https://docs.astral.sh/uv/
- **支援的 AI 編碼助手**（如 GitHub Copilot、Claude Code、Cursor 等）

### 安裝 Specify CLI

#### 方式一：永久安裝（推薦）

安裝一次，隨處可用：

```bash
uv tool install specify-cli --from git+https://github.com/github/spec-kit.git
```

升級到最新版：

```bash
uv tool install specify-cli --force --from git+https://github.com/github/spec-kit.git
```

#### 方式二：一次性使用

不安裝，直接透過 `uvx` 執行：

```bash
uvx --from git+https://github.com/github/spec-kit.git specify init <PROJECT_NAME>
```

### 初始化專案

在現有專案中初始化（以 GitHub Copilot + PowerShell 為例）：

```bash
# 在現有目錄初始化
specify init --here --ai copilot --script ps

# 或建立新專案
specify init my-project --ai copilot --script ps

# 檢查環境是否就緒
specify check
```

`--ai` 常用選項：`copilot`、`claude`、`cursor-agent`、`gemini`、`windsurf`、`codex` 等，完整清單請參考官方文件。

---

## 3. 初始化後的專案結構

初始化完成後會產生 `.specify/` 資料夾，其中 `init-options.json` 記錄核心設定：

```json
{
  "ai": "copilot",
  "speckit_version": "0.3.2",
  "script": "ps",
  "ai_skills": false,
  "here": true
}
```

- `ai`：使用 GitHub Copilot 作為 AI 後端
- `script`：使用 PowerShell 作為自動化腳本語言
- `here`：規格檔案放在專案根目錄下

專案結構關鍵資料夾如下：

```
.specify/
├── init-options.json        # 核心設定
├── memory/
│   └── constitution.md      # 專案憲法
├── scripts/powershell/      # 自動化腳本
│   ├── check-prerequisites.ps1
│   ├── common.ps1
│   ├── create-new-feature.ps1
│   └── setup-plan.ps1
└── templates/               # 規格模板
    ├── spec-template.md
    ├── plan-template.md
    ├── tasks-template.md
    └── checklist-template.md
```

---

## 4. 專案 Constitution（憲法）

Constitution 是 Speck-Kit 最強大的機制之一。它定義了專案的不可妥協標準，所有功能在規劃階段就必須通過 "Constitution Check"。

以我的 Library 專案為例，定義了四大原則：

### I. 程式品質 (Code Quality)

- 嚴格分層架構：Controller → Service → Repository，Controller 不得直接存取 DbContext
- TypeScript 啟用 strict mode，禁止 `any`；C# 啟用 nullable reference types
- 前端 ESLint 零錯誤，後端編譯零警告

### II. 測試標準 (Testing Standards)

- 後端使用 xUnit / NUnit，覆蓋率目標 ≥ 80%
- 前端使用 Vitest + Vue Test Utils
- 測試命名格式：`MethodName_Scenario_ExpectedResult`
- Bug 修復必須附帶重現測試案例

### III. 使用者體驗一致性 (UX Consistency)

- 唯一 UI 元件庫：Element Plus
- 樣式技術：SCSS + Tailwind CSS
- 所有操作須有 Loading 指示與錯誤處理

### IV. 效能需求 (Performance)

- API p95 ≤ 200ms（CRUD）、≤ 500ms（複雜查詢）
- LCP ≤ 2.5s，初始 Bundle ≤ 300KB（gzip）
- 禁止 N+1 資料庫查詢

---

## 5. 開發工作流：從需求到交付

Speck-Kit 將開發切分為明確的五個階段，每個階段都有對應的 Copilot Agent：

```
使用者需求
    ↓
① /speckit.clarify   → 提出最多 5 個釐清問題，消除模糊
    ↓
② /speckit.specify   → 撰寫完整的功能規格（User Story + Acceptance Criteria）
    ↓
③ /speckit.plan      → 建立技術實作計畫（含 Constitution Check）
    ↓
⑤ /speckit.tasks     → 拆分工作項目並追蹤進度
    ↓
④ /speckit.implement → 依照計畫逐步實作

```

### 其他輔助 Agent

| Agent | 用途 |
|---|---|
| `speckit.analyze` | 分析現有程式碼或架構 |
| `speckit.constitution` | 管理與更新專案憲法 |
| `speckit.checklist` | 產生與追蹤檢核清單 |
| `speckit.taskstoissues` | 將 Tasks 轉為 GitHub Issues |

---

## 6. 實際案例：使用者註冊功能

以 Library 專案的 `001-user-registration` 為例，展示 Speck-Kit 如何運作。

### 功能規格目錄結構

```
specs/001-user-registration/
├── spec.md          # 功能規格（User Story + Acceptance Criteria）
├── plan.md          # 技術實作計畫
├── tasks.md         # 工作項目拆分
├── checklists/      # 進度追蹤
├── contracts/       # API 合約
├── data-model.md    # 資料模型定義
└── research.md      # 技術研究筆記
```

### 規格範例（spec.md 摘錄）

**User Story 1 - 使用者自助註冊帳號 (P1)**

> 一位新使用者進入圖書管理系統的登入頁面，點擊「立即註冊」連結，系統導向註冊頁面。使用者填入帳號名稱、電子郵件地址與密碼（需輸入兩次確認），提交表單後系統建立帳號並告知使用者需至信箱完成驗證。

**Acceptance Scenarios（Gherkin 格式）**：

1. **Given** 使用者在註冊頁面，**When** 填入有效帳號、電子郵件與密碼並提交，**Then** 系統建立帳號並顯示「註冊成功，請至信箱完成驗證」
2. **Given** 使用者在註冊頁面，**When** 填入已被使用的電子郵件，**Then** 顯示「此電子郵件已被使用」
3. **Given** 使用者在註冊頁面，**When** 兩次密碼不一致，**Then** 前端即時顯示「密碼不一致」，表單不送出

---

## 7. 技術堆疊整合

在 Library 專案中，Speck-Kit 搭配以下技術堆疊運作：

| 層級 | 技術 |
|---|---|
| 後端框架 | ASP.NET Core 8.0 (C#) |
| 認證機制 | ASP.NET Core Identity + JWT |
| 資料庫 | SQL Server + Entity Framework Core 9 |
| 前端框架 | Vue 3 + TypeScript 5.8 (strict mode) |
| 建置工具 | Vite |
| UI 元件庫 | Element Plus |
| 樣式 | SCSS + Tailwind CSS 4 |
| 狀態管理 | Pinia |
| 路由 | Vue Router 4 |

專案分層架構：

```
Library.Server/      → API 層（Controllers, Services）
Zheng.Bll/           → 商業邏輯層（Services, Repositories）
Zheng.Infra.Data/    → 資料存取層（EF Core Models）
Zheng.Mdl/           → 領域模型（Enums, Shared Types）
library.client/      → 前端 Vue 3 應用
```

---

## 8. 自動化腳本

Speck-Kit 提供 PowerShell 腳本來自動化常見流程：

- **`check-prerequisites.ps1`**：驗證環境是否正確設定，定位功能目錄與規格檔案
- **`create-new-feature.ps1`**：建立新功能的目錄結構，從模板產生 spec.md、plan.md 等檔案
- **`setup-plan.ps1`**：初始化功能的技術規劃
- **`common.ps1`**：共用輔助函式（偵測 repo 根目錄、分支名稱等）

---

## 9. 使用心得與建議

### 優點

- **減少返工**：在寫程式前就把需求釐清，避免「寫了才發現不對」
- **AI 協作更有效**：給 Copilot 明確的規格和約束，產出品質更高
- **團隊一致性**：Constitution 確保所有人遵循相同標準
- **可追溯性**：每個功能都有完整的規格 → 計畫 → 實作紀錄

### 適用場景

- 中大型全端專案，需要多人協作
- 使用 GitHub Copilot 進行 AI 輔助開發
- 重視程式品質與測試覆蓋的團隊

---

## 參考

- Spec-Kit 官方 GitHub：https://github.com/github/spec-kit
- Spec-Kit 版本：v0.3.2
- 實作專案：https://github.com/hezhengmin/Library/tree/feats/001-user-registration



