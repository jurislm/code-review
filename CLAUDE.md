# Code Review Plugin — 專案指引

## 專案概覽

Claude Code plugin，提供完整 code review 生態系統。

- **Repo**: https://github.com/jurislm/code-review
- **Landing page**: https://jurislm.github.io/code-review/（GitHub Pages，從 `docs/` 提供）
- **Plugin marketplace**: `jurislm/code-review`（在 Claude Code 中 `/plugin` 搜尋安裝）
- **License**: MIT

## 目錄結構

```
code-review/
├── .claude-plugin/
│   ├── plugin.json          # Plugin manifest（name, version, author, keywords）
│   └── marketplace.json     # Marketplace 發布設定
├── agents/                  # 24 個 reviewer agents（.md 格式）
├── commands/                # 9 個 slash commands（.md 格式）
├── skills/                  # 3 個 auto-trigger skills
│   ├── security-review/
│   ├── security-scan/
│   └── flutter-dart-code-review/
└── docs/
    └── index.html           # GitHub Pages landing page
```

## Agents（24 個）

### 通用主審
- `code-reviewer` — 主審，含 false positive 過濾
- `security-reviewer` — OWASP Top 10，遇 CRITICAL 警報

### `/review-pr` 協作（5 個）
- `comment-analyzer` · `pr-test-analyzer` · `silent-failure-hunter` · `type-design-analyzer` · `code-simplifier`

### 語言 / 框架專項（17 個）
- **藍色**：typescript · python · go · rust · cpp · csharp · java · kotlin · swift · fsharp
- **黃色**：django · fastapi · flutter · database · network-config
- **紫色**：healthcare（Opus）· mle（Sonnet）

## Commands（9 個）

| 檔案 | Slash command |
|------|--------------|
| `code-review.md` | `/code-review` — 本地 diff 或 PR review |
| `review-pr.md` | `/review-pr` — 六 agent 並行，支援 `--focus` |
| `python-review.md` | `/python-review` |
| `go-review.md` | `/go-review` |
| `rust-review.md` | `/rust-review` |
| `cpp-review.md` | `/cpp-review` |
| `kotlin-review.md` | `/kotlin-review` |
| `fastapi-review.md` | `/fastapi-review` |
| `flutter-review.md` | `/flutter-review` |

## Skills（3 個，自動觸發）

| 目錄 | 觸發時機 |
|------|---------|
| `security-review/` | 實作 auth、處理 user input、建立 API endpoint |
| `security-scan/` | 掃描 `.claude/` 設定安全漏洞 |
| `flutter-dart-code-review/` | Review Flutter / Dart 程式碼 |

## 修改指引

### 新增 Agent

1. 在 `agents/` 建立 `<name>.md`，frontmatter 至少包含 `name`、`description`、`color`
2. 若需在 `/review-pr` 並行流程中使用，更新 `commands/review-pr.md` 的 agent 列表
3. 更新 `README.md` 的 Agent 表格
4. 更新 `docs/index.html` 的 agent tag 列表

### 新增 Command

1. 在 `commands/` 建立 `<name>.md`，frontmatter 包含 `description`、`argument-hint`
2. 更新 `README.md` 的 Slash Commands 表格
3. 更新 `docs/index.html` 的 stats 數字

### 更新 Landing Page

Landing page 是純 HTML，直接編輯 `docs/index.html`。push 到 `main` 後 GitHub Pages 自動重新部署（約 1 分鐘）。

### 更新 Plugin Manifest

`plugin.json` 和 `marketplace.json` 需同步維護 version、keywords、description。

## 設計原則（勿破壞）

1. **高信心原則** — 只報告 >80% 確信的問題
2. **讀全文** — PR review 讀每個檔案全文，不只 diff 行
3. **零 finding 合法** — clean code → APPROVE
4. **HIGH/CRITICAL 三要素** — 精確行號 + 具體失敗場景 + 現有 guard 為何不夠
5. **False positive 過濾** — 排除 magic number、fire-and-forget、test fixture 等常見誤判

## GitHub Repo 設定

- **Description**: "24-agent code review ecosystem for Claude Code — language-specific reviewers, 6-agent parallel PR review, and security scanning"
- **Homepage**: https://jurislm.github.io/code-review/
- **Topics**: code-review · claude-code · claude-code-plugin · security · pr-review · typescript · python · golang · rust · multi-agent
- **GitHub Pages**: `main` 分支 `/docs` 資料夾
