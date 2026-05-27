# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 專案概覽

Claude Code plugin，提供完整 code review 生態系統。

- **Repo**: https://github.com/jurislm/code-review
- **Landing page**: https://jurislm.github.io/code-review/（GitHub Pages，從 `docs/` 提供）
- **Plugin marketplace**: `jurislm/code-review`（在 Claude Code 中 `/plugin` 搜尋安裝）
- **License**: MIT

## 無 Build 步驟

這是純 Markdown 內容 repo，無 `package.json`、無測試、無 lint 指令。「工作內容」= agent / command / skill `.md` 檔案 + `docs/index.html`。

## 分支工作流

日常開發在 `.worktrees/develop`，完成後 PR `develop → main`（push to main = plugin marketplace 自動發布）。

## 環境變數

| 變數 | 用途 | 必要性 |
|------|------|-------|
| `BB_USERNAME` | Bitbucket PR review 認證 | Bitbucket 用戶才需要 |
| `BB_APP_PASSWORD` | Bitbucket App Password | Bitbucket 用戶才需要 |

GitHub 用戶無需額外設定（使用 `gh` CLI）。

## Commit Type 指引（Release Please）

本 repo 的「產品」是 skill / agent / command 內容，不是程式碼：

| 類型 | 適用場景 | 版本影響 |
|------|---------|---------|
| `feat:` | 新增 agent、command、skill；更新內容（新規則、新範例） | minor bump |
| `fix:` | 修正 agent 錯誤資訊、broken behavior | patch bump |
| `docs:` | 純格式或標點，無實質內容改動 | 無 |
| `chore:` | manifest 更新、README 格式 | 無 |

⚠️ 錯用 `docs:` 更新 agent 內容 → Release Please 不會發布新版本。

## 版本同步（必做）

`plugin.json` 和 `marketplace.json` 的 `version` 欄位**必須手動保持一致**。  
目前 `plugin.json` v1.2.0 / `marketplace.json` v1.0.0（drift 中）。每次版本 bump 兩個檔案都要更新。

## Agent Frontmatter 必填欄位

```yaml
---
name: <kebab-case>
description: <一句話描述，用於 PROACTIVELY 觸發條件>
tools: [Read, Grep, Glob]   # 按需加 Bash, Write, Edit
model: sonnet               # 預設 sonnet；特殊：healthcare-reviewer 用 opus
color: green                # green/blue/yellow/magenta/red/orange/purple/cyan/gray
---
```

Skill frontmatter 只需 `name` + `description`（無 tools / model / color）。

## 修改指引

### 新增 Agent

1. 在 `agents/` 建立 `<name>.md`，frontmatter 包含必填欄位
2. 若需在 `/review-pr` 並行流程中使用，更新 `commands/review-pr.md` 的 agent 列表
3. 更新 `README.md` 的 Agent 表格
4. 更新 `docs/index.html` 的 agent tag 列表（stats 數字 + agent badge）

### 新增 Command

1. 在 `commands/` 建立 `<name>.md`，frontmatter 含 `description`、`argument-hint`
2. 更新 `README.md` 的 Slash Commands 表格
3. 更新 `docs/index.html` 的 stats 數字

### 更新 Landing Page

Landing page 是純 HTML，直接編輯 `docs/index.html`。push 到 `main` 後 GitHub Pages 自動重新部署（約 1 分鐘）。

### 更新 Plugin Manifest

同步更新 `plugin.json` 和 `marketplace.json` 的 version、keywords、description。

## 設計原則（勿破壞）

1. **高信心原則** — 只報告 >80% 確信的問題
2. **讀全文** — PR review 讀每個檔案全文，不只 diff 行
3. **零 finding 合法** — clean code → APPROVE
4. **HIGH/CRITICAL 三要素** — 精確行號 + 具體失敗場景 + 現有 guard 為何不夠
5. **False positive 過濾** — 排除 magic number、fire-and-forget、test fixture 等常見誤判

## Agents 清單（24 個）

### 通用主審
- `code-reviewer`（green）— 主審，含 false positive 過濾
- `security-reviewer`（red）— OWASP Top 10，遇 CRITICAL 警報

### `/review-pr` 協作（5 個）
- `comment-analyzer` · `pr-test-analyzer` · `silent-failure-hunter` · `type-design-analyzer` · `code-simplifier`

### 語言 / 框架專項（17 個）
詳見 `agents/`；特殊：`healthcare-reviewer` 使用 `opus`（其餘皆 `sonnet`）。

## Commands 清單（9 個）

| 檔案 | Slash command | 備註 |
|------|--------------|------|
| `code-review.md` | `/code-review` | 本地 diff 或 PR review（GitHub / Bitbucket） |
| `review-pr.md` | `/review-pr` | 六 agent 並行；`--focus=comments\|tests\|errors\|types\|code\|simplify` |

語言專項 commands 命名規則：`/<language>-review`（詳見 `commands/`）。

## Skills（3 個，自動觸發）

| 目錄 | 觸發時機 |
|------|---------|
| `security-review/` | 實作 auth、處理 user input、建立 API endpoint |
| `security-scan/` | 掃描 `.claude/` 設定安全漏洞 |
| `flutter-dart-code-review/` | Review Flutter / Dart 程式碼 |

