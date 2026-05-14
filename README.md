# Code Review Plugin for Claude Code

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
![Agents](https://img.shields.io/badge/Agents-24-blue)
![Commands](https://img.shields.io/badge/Commands-9-green)
![Skills](https://img.shields.io/badge/Skills-3-purple)
![Platform](https://img.shields.io/badge/Platform-Claude_Code-orange)

完整 code review 生態系統，以 Claude Code plugin 形式發布。提供 24 個語言/框架專項 reviewer agent、9 個 slash command、以及 3 個安全 review skill。

---

## ✨ 亮點

| | |
|---|---|
| 🤖 **24 個 Reviewer Agent** | TypeScript · Python · Go · Rust · Java · Kotlin · Swift · C++ · C# · F# · Django · FastAPI · Flutter · DB · Healthcare · ML |
| ⚡ **六 Agent 並行 PR Review** | `/review-pr` 同時啟動 6 個專項 agent，confidence < 80% 自動過濾 |
| 🔒 **多層安全掃描** | OWASP Top 10 · PHI/HIPAA · Claude Code 設定掃描 |
| 🎯 **高信心原則** | 只報告 >80% 確信的問題，零 finding = APPROVE，不製造雜訊 |

---

## 🚀 30 秒快速開始

在 Claude Code 中執行：

```
/plugin
```

搜尋 `jurislm/code-review`，點擊安裝，再執行 `/reload-plugins` 套用。

**立即使用：**

```bash
# 審查目前 uncommitted 變更
/code-review

# 審查 PR（傳號碼或 URL）
/code-review 123

# 六 agent 並行 PR review
/review-pr 123
```

---

## 架構總覽

```
code-review plugin
├── Commands（9 個 slash commands）
│   ├── /code-review          本地 diff 或 PR review
│   ├── /review-pr            六 agent 並行 + --focus 過濾
│   └── /python-review ... /flutter-review  語言專項
│
├── Agents（24 個 reviewer agents）
│   ├── 通用主審
│   │   ├── code-reviewer     主審，含 false positive 過濾
│   │   └── security-reviewer OWASP Top 10，遇 CRITICAL 警報
│   ├── PR 協作（搭配 /review-pr）
│   │   ├── comment-analyzer  行內 comment 品質
│   │   ├── pr-test-analyzer  測試覆蓋
│   │   ├── silent-failure-hunter  swallowed error 偵測
│   │   ├── type-design-analyzer  型別設計
│   │   └── code-simplifier   過度複雜實作
│   └── 語言 / 框架專項（17 個）
│       TypeScript · Python · Go · Rust · C++ · C# · Java
│       Kotlin · Swift · F# · Django · FastAPI · Flutter
│       Database · Network · Healthcare · MLE
│
└── Skills（3 個，自動觸發）
    ├── security-review       實作 auth / 處理 user input 時觸發
    ├── security-scan         掃描 .claude/ 設定安全漏洞
    └── flutter-dart-code-review  Review Flutter / Dart 時觸發
```

---

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
git clone https://github.com/jurislm/code-review.git

cp commands/*.md ~/.claude/commands/
cp agents/*.md ~/.claude/agents/
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

針對特定語言叫用對應 command（不傳參數則預設審查 staged changes）：

```
/python-review
/go-review
/rust-review
/cpp-review
/kotlin-review
/fastapi-review
/flutter-review
```

### 5. 直接呼叫 Agent

在對話中描述任務，Claude 會自動選用對應 agent：

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

---

## License

MIT © [Terry Chen](https://github.com/jurislm)
