# Code Review Ecosystem

完整 code review agent 生態系統。

## 目錄結構

```
code-review/
├── commands/     # Slash commands（/code-review、/review-pr 等）
├── agents/       # 各語言/框架/功能專用 reviewer agent
└── skills/       # 背景知識 skill（security-review、security-scan）
```

## 使用方式

### 安裝到 Claude Code

```bash
# 複製 commands 到專案 .claude/
cp commands/*.md /Users/terrychen/Documents/Github/jurislm/<project>/.claude/commands/

# 複製 agents 到全域
cp agents/*.md ~/.claude/agents/

# 複製 skills 到全域
cp -r skills/* ~/.claude/skills/
```

---

## Slash Commands

| 指令 | 說明 |
|------|------|
| `/code-review` | 本地未 commit 變更 review，或 PR review（傳 PR 號/URL） |
| `/review-pr` | 六 agent 並行 PR review，可用 `--focus` 聚焦特定面向 |
| `/python-review` | Python 專項 review |
| `/go-review` | Go 專項 review |
| `/rust-review` | Rust 專項 review |
| `/cpp-review` | C++ 專項 review |
| `/kotlin-review` | Kotlin 專項 review |
| `/fastapi-review` | FastAPI 專項 review |
| `/flutter-review` | Flutter 專項 review |

### `/code-review` 核心流程

**Local 模式**（無參數）：3 phases — Gather → Review → Report

**PR 模式**（傳 PR 號）：8 phases — Fetch → Context → Review → Validate → Decide → Report → Publish → Output

決策邏輯：
- CRITICAL → **BLOCK**
- HIGH 或 validation 失敗 → **REQUEST CHANGES**
- 僅 MEDIUM/LOW → **APPROVE with comments**
- 零 finding → **APPROVE**

### `/review-pr` 並行 agents

同時啟動六個 agent，confidence < 80 的 finding 自動過濾：

| Agent | 負責 |
|-------|------|
| `code-reviewer` | 品質主審 |
| `comment-analyzer` | 行內 comment 品質 |
| `pr-test-analyzer` | 測試覆蓋 |
| `silent-failure-hunter` | 靜默失敗路徑 |
| `type-design-analyzer` | 型別設計 |
| `code-simplifier` | 可簡化的地方 |

---

## Agents

### 通用主審

| Agent | 說明 |
|-------|------|
| `code-reviewer` | 主審，含 React/Node.js 專項規則，有嚴格 false positive 過濾清單 |
| `security-reviewer` | OWASP Top 10 專項掃描，遇 CRITICAL 發緊急警報 |

### `/review-pr` 協作 agents

| Agent | 說明 |
|-------|------|
| `comment-analyzer` | 審查行內 comment 品質 |
| `pr-test-analyzer` | 分析測試覆蓋狀況 |
| `silent-failure-hunter` | 找 swallowed error、ignored promise |
| `type-design-analyzer` | 型別設計審查 |
| `code-simplifier` | 找過度複雜的實作 |

### 語言 / 框架專項

| Agent | 適用 |
|-------|------|
| `typescript-reviewer` | TypeScript / JavaScript |
| `python-reviewer` | Python |
| `go-reviewer` | Go |
| `rust-reviewer` | Rust |
| `cpp-reviewer` | C++ |
| `csharp-reviewer` | C# |
| `java-reviewer` | Java |
| `kotlin-reviewer` | Kotlin |
| `flutter-reviewer` | Flutter / Dart |
| `fastapi-reviewer` | FastAPI |
| `django-reviewer` | Django |
| `swift-reviewer` | Swift |
| `fsharp-reviewer` | F# |
| `database-reviewer` | DB query、migration |
| `mle-reviewer` | ML pipeline |
| `network-config-reviewer` | 網路設備設定 |
| `healthcare-reviewer` | PHI / HIPAA 合規 |

---

## Skills

| Skill | 說明 |
|-------|------|
| `security-review` | 安全 review checklist，含 cloud infrastructure security |
| `security-scan` | 安全掃描工作流 |
| `flutter-dart-code-review` | Flutter / Dart 專項 review 知識 |

---

## 設計原則

1. **高信心原則**：只報告 >80% 確信的問題
2. **讀全文**：PR review 讀每個檔案全文，不只 diff 行
3. **零 finding 合法**：clean code → APPROVE，不製造雜訊
4. **HIGH/CRITICAL 三要素**：精確行號 + 具體失敗場景 + 現有 guard 為何不夠
5. **False positive 過濾**：明確排除 LLM reviewer 常見誤判（magic number、fire-and-forget、test fixture 等）
