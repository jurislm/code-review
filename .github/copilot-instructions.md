# Copilot Instructions

請使用繁體中文回覆所有問題與建議。

## 專案概覽

`jurislm/code-review` 是 Claude Code plugin，提供完整 code review 生態系統：
- 24 個 reviewer agents（語言 / 框架專項）
- 9 個 slash commands
- 3 個 auto-trigger skills

所有元件均以 Markdown 格式撰寫，無 TypeScript 或其他程式語言原始碼。

## Git Workflow

- `main` 分支觸發 GitHub Pages 自動部署（`docs/`）及 Release Please
- 所有開發在 `.worktrees/develop` 進行，透過 PR develop → main 合併
- 嚴禁直接 push 到 `main`

## 目錄結構

| 目錄 | 用途 |
|------|------|
| `agents/` | 24 個 reviewer agents（`.md`） |
| `commands/` | 9 個 slash commands（`.md`） |
| `skills/` | 3 個 auto-trigger skills（各含 `SKILL.md`） |
| `docs/` | GitHub Pages landing page（`index.html`） |
| `.claude-plugin/` | Plugin manifest（`plugin.json`、`marketplace.json`） |

## 版本管理

版本號統一由 Release Please 自動管理，修改後需同步更新：
- `.claude-plugin/plugin.json` → `$.version`
- `.claude-plugin/marketplace.json` → `$.metadata.version` 和 `$.plugins[0].version`
- `.release-please-manifest.json`

## Agent Frontmatter 格式

```markdown
---
name: agent-name
description: 說明此 agent 的用途
color: blue
---
```

必填欄位：`name`、`description`、`color`。

## Code Review 重點

- Agent `.md` 檔案：確認 frontmatter 格式正確、description 清楚說明用途
- Command `.md` 檔案：確認 `argument-hint` 存在、流程說明完整
- JSON 設定檔：確認 `version` 各欄位一致（`plugin.json` 與 `marketplace.json`）
- `docs/index.html`：確認統計數字（agents/commands/skills 數量）與實際一致

## 忽略範圍

- `.worktrees/` — git worktree build 產物
- `docs/` — 靜態網頁，不審查 HTML/CSS 細節
