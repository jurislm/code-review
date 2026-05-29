# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 專案概覽

Claude Code plugin，提供完整 code review 生態系統。

- **Repo**: https://github.com/jurislm/code-review
- **Landing page**: https://jurislm.github.io/code-review/（GitHub Pages，從 `docs/` 提供）
- **Plugin marketplace**: `jurislm/code-review`（在 Claude Code 中 `/plugin` 搜尋安裝）
- **版本**: v1.3.0
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
agents/          # 27 個 agent（frontmatter: name/description/tools/model/color）
commands/        # 1 個 slash command（/code-review，自動偵測語言/框架並 dispatch 對應 agent）
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

⚠️ 錯用 `docs:` 更新 agent 內容 → `docs:` 雖會進 CHANGELOG，但不觸發版本 bump（不建立 release PR）。

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
description: <triggering 格式，見下>  # 必填
tools: [Read, Grep, Glob]   # 建議；按需加 Bash, Write, Edit
model: sonnet               # 建議；預設 sonnet；特殊：healthcare-reviewer 用 opus
color: blue                 # 必填；官方 validator 認可：blue/cyan/green/yellow/magenta/red（避免 orange/purple/cyan 以外的色）
---
```

**`description` 採官方 triggering 格式**（flat prose，單行，提升自動 dispatch 命中率）：
`Use this agent when <conditions>. Typical triggers include <2-4 個 noun-phrase 場景>. See "When to invoke" in the agent body for worked scenarios.`

並在 frontmatter 後緊接一個 `## When to invoke` 區塊（2-4 條第三人稱 prose bullet，描述情境 + agent 應做什麼，**不要**引用對話逐字稿）。可用官方 validator 驗證：
```bash
bash ~/.claude/plugins/cache/claude-plugins-official/plugin-dev/*/skills/agent-development/scripts/validate-agent.sh agents/<name>.md
```
（`<example>` blocks 警告為舊慣例，現行格式改用 `## When to invoke` body 區塊，可忽略該警告。）

Skill frontmatter 只需 `name` + `description`（無 tools / model / color）。

## 修改指引

### 新增 Agent

1. 在 `agents/` 建立 `<name>.md`，frontmatter 包含必填欄位
2. 若是語言/框架專項，更新 `commands/code-review.md` 的「Language / Framework Auto-Dispatch」對照表（副檔名 → agent，或框架內容特徵 → agent），讓 `/code-review` 能自動 dispatch
3. 若是 PR 模式並行協作 agent，更新 `commands/code-review.md` Phase 3 的「General agents」列表
4. 更新 `README.md` 的 Agent 表格
5. 更新 `docs/index.html` 的 agent tag 列表（stats 數字 + agent badge）
6. 更新本 CLAUDE.md 的 Agents 清單（數字 + 語言專項列表）

### 修改 Command

> 本 repo 只有單一 command `/code-review`。所有 review 功能（本地 / GitHub PR / Bitbucket PR / 多 agent 並行 / verification / 語言自動 dispatch）都在 `commands/code-review.md` 內。新增能力時直接擴充該檔，不另建 command。

1. 編輯 `commands/code-review.md`，frontmatter 維持 `description`、`argument-hint`
2. 若改動使用者可見行為（新 flag、新模式），更新 `README.md` 與 `docs/index.html`

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
- [ ] 實質內容變更用 `feat:` 或 `fix:`（非 `docs:`）— 只有這兩者才觸發版本 bump

## 設計原則（勿破壞）

1. **高信心原則** — 只報告 >80% 確信的問題
2. **讀全文** — PR review 讀每個檔案全文，不只 diff 行
3. **零 finding 合法** — clean code → APPROVE
4. **HIGH/CRITICAL 三要素** — 精確行號 + 具體失敗場景 + 現有 guard 為何不夠
5. **False positive 過濾** — 排除 magic number、fire-and-forget、test fixture 等常見誤判
6. **Verification gate** — `/code-review` PR 模式 Phase 3.5 由 `verification-reviewer` 對 HIGH/CRITICAL 執行三道關卡二次確認；CONFIRMED 及 UNCERTAIN（降為 MEDIUM）的 finding 均進入最終報告；若本 PR diff 已修復問題，以「FIXED IN THIS PR」verdict 移除（不受 CRITICAL 保護限制）；CRITICAL 不可被移除（最多降為 HIGH）
7. **NITPICK 分層** — 純風格偏好歸類為 NITPICK；`--profile=chill` 時略過，`--profile=assertive`（預設）時顯示

## Agents 清單（27 個）

### 通用主審
- `code-reviewer`（green）— 主審，含 false positive 過濾
- `security-reviewer`（red）— OWASP Top 10，遇 CRITICAL 警報
- `verification-reviewer`（yellow）— 第二道驗證，在輸出前過濾 HIGH/CRITICAL false positive

### 前置分析
- `code-graph-analyzer`（cyan）— L2 import dependency + L3 co-change 風險圖；在並行 agents 前執行，結果快取於 `.claude/code-graph/`

### `/code-review` PR 模式並行協作（6 個）

> `/code-review` PR 模式並行 8 通用 agent = `code-reviewer` + `security-reviewer`（通用主審）+ 下列 6 個。語言/框架專項 agent 由副檔名自動 dispatch 額外加入。

- `comment-analyzer` · `pr-test-analyzer` · `silent-failure-hunter` · `type-design-analyzer` · `code-simplifier` · `pr-walkthrough-writer`

### 語言 / 框架專項（17 個，由 `/code-review` 自動 dispatch）

> `/code-review` 依變更檔案副檔名自動加對應 agent（如 `.py`→`python-reviewer`、`.go`→`go-reviewer`）；框架（Django/FastAPI/Flutter）再以內容特徵輔助判斷。dispatch 對照表見 `commands/code-review.md`「Language / Framework Auto-Dispatch」。

`cpp` · `csharp` · `database` · `django` · `fastapi` · `flutter` · `fsharp` · `go` · `healthcare`（opus）· `java` · `kotlin` · `mle` · `network-config` · `python` · `rust` · `swift` · `typescript`

## Commands 清單（1 個）

> 統一為單一 `/code-review`：依參數自動切換本地 / GitHub PR / Bitbucket PR 模式，依變更檔案自動 dispatch 語言/框架專項 agent。語言專項不再各自有 command（舊 `/python-review`、`/go-review`、`/review-pr` 等已移除，能力併入 `/code-review`）。

| 檔案 | Slash command | 備註 |
|------|--------------|------|
| `code-review.md` | `/code-review` | 全功能單一入口。**本地模式**（blank / `--from=<commit>`）與 **PR 模式**（PR 號/URL，GitHub / Bitbucket）跑**相同完整 pipeline**：`code-graph-analyzer` 前置 + 八通用 agent + 自動 dispatch 語言 agent 並行 + `verification-reviewer` Phase 3.5。差別僅在本地輸出到終端機、PR 模式發佈到平台。支援 `--focus=comments\|tests\|errors\|types\|code\|simplify`、`--profile=chill\|assertive` |

## Skills（3 個，自動觸發）

| 目錄 | 觸發時機 |
|------|---------|
| `security-review/` | 實作 auth、處理 user input、建立 API endpoint |
| `security-scan/` | 掃描 `.claude/` 設定安全漏洞 |
| `flutter-dart-code-review/` | Review Flutter / Dart 程式碼 |

