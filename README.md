# Code Review Plugin for Claude Code

完整 code review 生態系統，以 Claude Code plugin 形式發布。提供 24 個語言/框架專項 reviewer agent、9 個 slash command、以及 3 個安全 review skill。

## 安裝

### Plugin 安裝（推薦）

在 Claude Code 中執行：

```
/plugin
```

搜尋 `jurislm/code-review`，點擊安裝，再執行 `/reload-plugins` 套用。

安裝後可直接使用所有 slash commands 與 agents，無需額外設定。

### 手動安裝

```bash
# Clone repo
git clone https://github.com/jurislm/code-review.git

# 複製 commands 到全域
cp commands/*.md ~/.claude/commands/

# 複製 agents 到全域
cp agents/*.md ~/.claude/agents/

# 複製 skills 到全域
cp -r skills/* ~/.claude/skills/
```

---

## 使用教學

### 1. 本地變更 Review

對目前 uncommitted 的變更進行 review：

```
/code-review
```

流程：`git diff` 收集變更 → 逐檔審查 → 輸出報告

### 2. PR Review

傳入 PR 號碼或 URL：

```
/code-review 123
/code-review https://github.com/owner/repo/pull/123
```

流程（8 phases）：Fetch → Context → Review → Validate → Decide → Report → Publish → Output

決策邏輯：

| 結果 | 條件 |
|------|------|
| **BLOCK** | 任一 CRITICAL finding |
| **REQUEST CHANGES** | 任一 HIGH finding 或 validation 失敗 |
| **APPROVE with comments** | 僅 MEDIUM / LOW finding |
| **APPROVE** | 零 finding |

### 3. 多 Agent 並行 PR Review

同時啟動 6 個專項 agent，confidence < 80% 的 finding 自動過濾：

```
/review-pr 123
/review-pr 123 --focus security
/review-pr 123 --focus performance
/review-pr 123 --focus types
/review-pr 123 --focus tests
```

### 4. 語言專項 Review

針對特定語言，直接叫用對應 command：

```
/python-review
/go-review
/rust-review
/cpp-review
/kotlin-review
/fastapi-review
/flutter-review
```

所有語言 command 可傳入目錄或檔案路徑作為參數，不傳則預設審查 staged changes。

### 5. 直接呼叫 Agent

在對話中 @ 指定 agent，或描述任務讓 Claude 自動選用：

```
@code-reviewer 請審查這段 auth middleware
@security-reviewer 這個 API endpoint 有安全問題嗎？
@typescript-reviewer 這個 generic type 設計合理嗎？
@database-reviewer 這個 migration 安全嗎？
```

---

## Slash Commands

| 指令 | 說明 |
|------|------|
| `/code-review` | 本地 review 或 PR review（傳 PR 號/URL） |
| `/review-pr` | 六 agent 並行 PR review，支援 `--focus` |
| `/python-review` | Python 專項 review |
| `/go-review` | Go 專項 review |
| `/rust-review` | Rust 專項 review |
| `/cpp-review` | C++ 專項 review |
| `/kotlin-review` | Kotlin 專項 review |
| `/fastapi-review` | FastAPI 專項 review |
| `/flutter-review` | Flutter / Dart 專項 review |

---

## Agents

### 通用主審

| Agent | Color | 說明 |
|-------|-------|------|
| `code-reviewer` | 🟢 green | 主審，含嚴格 false positive 過濾，React / Node.js 專項規則 |
| `security-reviewer` | 🔴 red | OWASP Top 10 掃描，遇 CRITICAL 發緊急警報 |

### `/review-pr` 協作 Agents

| Agent | Color | 說明 |
|-------|-------|------|
| `comment-analyzer` | 🔵 cyan | 行內 comment 品質審查 |
| `pr-test-analyzer` | 🔵 cyan | 測試覆蓋分析 |
| `silent-failure-hunter` | 🔵 cyan | 偵測 swallowed error、ignored promise |
| `type-design-analyzer` | 🔵 cyan | 型別設計審查 |
| `code-simplifier` | 🔵 cyan | 找過度複雜的實作 |

### 語言 / 框架專項

| Agent | Color | 適用 |
|-------|-------|------|
| `typescript-reviewer` | 🔵 blue | TypeScript / JavaScript |
| `python-reviewer` | 🔵 blue | Python |
| `go-reviewer` | 🔵 blue | Go |
| `rust-reviewer` | 🔵 blue | Rust |
| `cpp-reviewer` | 🔵 blue | C++ |
| `csharp-reviewer` | 🔵 blue | C# |
| `java-reviewer` | 🔵 blue | Java |
| `kotlin-reviewer` | 🔵 blue | Kotlin |
| `swift-reviewer` | 🔵 blue | Swift |
| `fsharp-reviewer` | 🔵 blue | F# |
| `django-reviewer` | 🟡 yellow | Django |
| `fastapi-reviewer` | 🟡 yellow | FastAPI |
| `flutter-reviewer` | 🟡 yellow | Flutter / Dart |
| `database-reviewer` | 🟡 yellow | DB query、migration |
| `network-config-reviewer` | 🟡 yellow | 網路設備設定 |
| `healthcare-reviewer` | 🟣 magenta | PHI / HIPAA 合規（Opus 模型） |
| `mle-reviewer` | 🟣 magenta | ML pipeline（Opus 模型） |

---

## Skills

Skills 會根據任務上下文自動啟動，無需手動呼叫。

| Skill | 自動觸發時機 |
|-------|-------------|
| `security-review` | 實作 auth、處理 user input、建立 API endpoint、存取 secrets |
| `security-scan` | 掃描 Claude Code 設定（`.claude/`）的安全漏洞 |
| `flutter-dart-code-review` | Review Flutter / Dart 程式碼 |

---

## 設計原則

1. **高信心原則** — 只報告 >80% 確信的問題，不製造雜訊
2. **讀全文** — PR review 讀每個檔案全文，不只 diff 行
3. **零 finding 合法** — clean code → APPROVE，不強行挑毛病
4. **HIGH/CRITICAL 三要素** — 精確行號 + 具體失敗場景 + 現有 guard 為何不夠
5. **False positive 過濾** — 明確排除 LLM reviewer 常見誤判（magic number、fire-and-forget、test fixture 等）

---

## 目錄結構

```
code-review/
├── .claude-plugin/
│   ├── plugin.json          # Plugin manifest
│   └── marketplace.json     # Marketplace 發布設定
├── agents/                  # 24 個 reviewer agents
├── commands/                # 9 個 slash commands
└── skills/                  # 3 個 review skills
    ├── security-review/
    ├── security-scan/
    └── flutter-dart-code-review/
```
