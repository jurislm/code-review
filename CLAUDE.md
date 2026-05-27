# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 專案概覽

Claude Code plugin，提供完整 code review 生態系統。

- **Repo**: https://github.com/jurislm/code-review
- **Landing page**: https://jurislm.github.io/code-review/（GitHub Pages，從 `docs/` 提供）
- **Plugin marketplace**: `jurislm/code-review`（在 Claude Code 中 `/plugin` 搜尋安裝）
- **版本**: v1.2.0
- **License**: MIT

## 常用操作速查

| 想做什麼 | 參考章節 |
|---------|---------|
| 新增 agent / command / skill | [修改指引](#修改指引) |
| 本地測試 plugin 變更 | [本地驗證](#本地驗證) |
| 建立 PR | [分支工作流](#分支工作流) |
| commit type 選哪個 | [Commit Type 指引](#commit-type-指引release-please) |

## 無 Build 步驟

這是純 Markdown 內容 repo，無 `package.json`、無測試、無 lint 指令。「工作內容」= agent / command / skill `.md` 檔案 + `docs/index.html`。

## 目錄結構

```
agents/          # 26 個 agent（frontmatter: name/description/tools/model/color）
commands/        # 9 個 slash command（/code-review、/review-pr、語言專項）
skills/          # 3 個自動觸發 skill（security-review/security-scan/flutter-dart-code-review）
docs/            # Landing page（GitHub Pages，純 HTML）
.claude-plugin/  # Plugin manifest（plugin.json + marketplace.json，version 由 Release Please 自動同步）
```

## 分支工作流

日常開發在 `.worktrees/develop`，完成後 PR `develop → main`；`feat`/`fix` commits 累積後，Release Please 在 main 自動建立 release PR，合入該 PR = 新版本發布。

建立 PR 後必須設定 labels 和 assignee（GitHub MCP 不支援，一律用 `gh api`）：

```bash
gh api repos/jurislm/code-review/issues/<num>/labels -f "labels[]=enhancement"
gh api repos/jurislm/code-review/issues/<num> -X PATCH -f "assignees[]=terry90918"
```

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

## 版本同步

`plugin.json` 和 `marketplace.json` 的版本由 `release-please-config.json` 的 `extra-files` **自動同步**——Release Please 合入 release PR 時更新 main 分支。

⚠️ **develop 分支需手動跟上**：每次 Release Please release PR 合入 main 後，執行 `git merge origin/main` 將版本 bump 同步回 develop。
觸發時機：`git log origin/main --oneline -3` 若出現 `chore(main): release v...` commit，即代表需要同步。
驗證指令：`grep '"version"' .claude-plugin/plugin.json .claude-plugin/marketplace.json`

## Agent Frontmatter 建議欄位

最低必填：`name`、`description`、`color`。`tools` 和 `model` 建議明確指定以確保一致行為。

```yaml
---
name: <kebab-case>           # 必填
description: <一句話描述，用於 PROACTIVELY 觸發條件>  # 必填
tools: [Read, Grep, Glob]   # 建議；按需加 Bash, Write, Edit
model: sonnet               # 建議；預設 sonnet；特殊：healthcare-reviewer 用 opus
color: green                # 必填；green/blue/yellow/magenta/red/orange/purple/cyan/gray
---
```

Skill frontmatter 只需 `name` + `description`（無 tools / model / color）。

## 修改指引

### 新增 Agent

1. 在 `agents/` 建立 `<name>.md`，frontmatter 包含必填欄位
2. 若需在 `/review-pr` 並行流程中使用，更新 `commands/review-pr.md` 的 agent 列表
3. 更新 `README.md` 的 Agent 表格
4. 更新 `docs/index.html` 的 agent tag 列表（stats 數字 + agent badge）
5. 更新本 CLAUDE.md 的 Agents 清單（數字 + 語言專項列表）

### 新增 Command

1. 在 `commands/` 建立 `<name>.md`，frontmatter 含 `description`、`argument-hint`
2. 更新 `README.md` 的 Slash Commands 表格
3. 更新 `docs/index.html` 的 stats 數字
4. 更新本 CLAUDE.md 的 Commands 清單表格

### 新增 Skill

1. 在 `skills/` 建立 `<name>/` 子目錄，內含 `SKILL.md`（frontmatter 只需 `name` + `description`，無 tools/model/color）
2. 若需附加參考文件，放在同一子目錄（如 `skills/security-review/cloud-infrastructure-security.md`）
3. 更新 `README.md` 的 Skills 表格
4. 更新 `docs/index.html` 的 stats 數字
5. 更新本 CLAUDE.md 的 Skills 清單（觸發時機）

### 更新 Landing Page

Landing page 是純 HTML，直接編輯 `docs/index.html`。push 到 `main` 後 GitHub Pages 自動重新部署（約 1 分鐘）。

### 更新 Plugin Manifest

手動同步 `plugin.json` 和 `marketplace.json` 的 `keywords`、`description`（`version` 由 Release Please 自動同步，勿手動修改）。

## 本地驗證

修改 agent / command / skill 後，commit 前先驗：

```bash
# 快速確認 agent frontmatter 格式（name/description/color 必填）
grep -l "^---" agents/*.md | xargs -I{} head -10 {}

# 驗證各類別實際數量（與 CLAUDE.md 計數一致）
echo "agents: $(ls agents/*.md | wc -l | tr -d ' '), commands: $(ls commands/*.md | wc -l | tr -d ' '), skills: $(ls -d skills/*/ | wc -l | tr -d ' ')"
```

修改後在 Claude Code 中執行（非 shell 指令）：`/reload-plugins`

commit checklist：
- [ ] 更新 `README.md` 對應表格
- [ ] 更新 `docs/index.html` stats 數字
- [ ] 更新本 `CLAUDE.md` 的計數（Agents 清單 / Commands 清單）
- [ ] commit type 用 `feat:`（非 `docs:`）才會觸發 Release Please

## 設計原則（勿破壞）

1. **高信心原則** — 只報告 >80% 確信的問題
2. **讀全文** — PR review 讀每個檔案全文，不只 diff 行
3. **零 finding 合法** — clean code → APPROVE
4. **HIGH/CRITICAL 三要素** — 精確行號 + 具體失敗場景 + 現有 guard 為何不夠
5. **False positive 過濾** — 排除 magic number、fire-and-forget、test fixture 等常見誤判
6. **Verification gate** — `/review-pr` Step 3.5 由 `verification-reviewer` 對 HIGH/CRITICAL 執行三道關卡二次確認；CRITICAL 不可被移除（最多降為 HIGH）
7. **NITPICK 分層** — 純風格偏好歸類為 NITPICK；`--profile=chill` 時略過，`--profile=assertive`（預設）時顯示

## Agents 清單（26 個）

### 通用主審
- `code-reviewer`（green）— 主審，含 false positive 過濾
- `security-reviewer`（red）— OWASP Top 10，遇 CRITICAL 警報
- `verification-reviewer`（orange）— 第二道驗證，在輸出前過濾 HIGH/CRITICAL false positive

### `/review-pr` 協作（6 個）
- `comment-analyzer` · `pr-test-analyzer` · `silent-failure-hunter` · `type-design-analyzer` · `code-simplifier` · `pr-walkthrough-writer`

### 語言 / 框架專項（17 個）
`cpp` · `csharp` · `database` · `django` · `fastapi` · `flutter` · `fsharp` · `go` · `healthcare`（opus）· `java` · `kotlin` · `mle` · `network-config` · `python` · `rust` · `swift` · `typescript`

## Commands 清單

> 注意：只有 7 個語言設有專屬 command。其餘（csharp/database/django/fsharp/healthcare/java/mle/network-config/swift/typescript）直接以 agent 形式呼叫，無對應 `/xxx-review` command。

| 檔案 | Slash command | 備註 |
|------|--------------|------|
| `code-review.md` | `/code-review` | 本地 diff 或 PR review（GitHub / Bitbucket） |
| `review-pr.md` | `/review-pr` | 七 agent 並行（含 `pr-walkthrough-writer`）；`--focus=comments\|tests\|errors\|types\|code\|simplify` |
| `cpp-review.md` | `/cpp-review` | C++ 專項 |
| `fastapi-review.md` | `/fastapi-review` | FastAPI 專項 |
| `flutter-review.md` | `/flutter-review` | Flutter/Dart 專項 |
| `go-review.md` | `/go-review` | Go 專項 |
| `kotlin-review.md` | `/kotlin-review` | Kotlin 專項 |
| `python-review.md` | `/python-review` | Python 專項 |
| `rust-review.md` | `/rust-review` | Rust 專項 |

## Skills（3 個，自動觸發）

| 目錄 | 觸發時機 |
|------|---------|
| `security-review/` | 實作 auth、處理 user input、建立 API endpoint |
| `security-scan/` | 掃描 `.claude/` 設定安全漏洞 |
| `flutter-dart-code-review/` | Review Flutter / Dart 程式碼 |

