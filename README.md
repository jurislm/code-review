# Code Review Plugin for Claude Code

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
![Agents](https://img.shields.io/badge/Agents-27-blue)
![Command](https://img.shields.io/badge/Command-1_unified-green)
![Skills](https://img.shields.io/badge/Skills-3-purple)
![Platform](https://img.shields.io/badge/Platform-Claude_Code-orange)
![GitHub](https://img.shields.io/badge/PR-GitHub-181717?logo=github)
![Bitbucket](https://img.shields.io/badge/PR-Bitbucket-0052CC?logo=bitbucket)

完整 code review 生態系統，以 Claude Code plugin 形式發布。提供 27 個 agent（含語言/框架專項 reviewer + Code Graph Analyzer）、單一 `/code-review` command（自動偵測語言/框架並 dispatch 對應 agent）、以及 3 個安全 review skill。

---

## ✨ 亮點

| | |
|---|---|
| 🤖 **27 個 Agent** | TypeScript · Python · Go · Rust · Java · Kotlin · Swift · C++ · C# · F# · Django · FastAPI · Flutter · DB · Healthcare · ML + Code Graph Analyzer + Verification + PR Walkthrough |
| 🧭 **單一 `/code-review`，自動 dispatch** | 一個 command 搞定全部。依變更檔案副檔名自動偵測語言/框架（`.py`→python-reviewer、`.go`→go-reviewer、Django/FastAPI/Flutter 以內容輔助），自動加對應專項 agent — 不用再記每個語言的 command |
| ⚡ **Code Graph + 八 Agent 並行** | PR 模式先執行 `code-graph-analyzer`（import deps + co-change risk），結果快取至 `.claude/code-graph/`；再並行啟動 8 通用 agent + 自動 dispatch 的語言 agent；HIGH/CRITICAL finding 由 `verification-reviewer` 二次確認 |
| 🔗 **雙平台 PR Review** | 自動偵測 GitHub（`gh` CLI）或 Bitbucket Cloud（REST API v2.0） |
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

# 審查 GitHub PR
/code-review 123
/code-review https://github.com/owner/repo/pull/123

# 審查 Bitbucket PR
/code-review https://bitbucket.org/workspace/repo/pull-requests/123

# 只跑特定面向（PR 模式）
/code-review 123 --focus tests
```

---

## 架構總覽

```
code-review plugin
├── Command（單一統一入口）
│   └── /code-review          本地 diff 或 GitHub/Bitbucket PR；自動偵測語言/框架並 dispatch
│       ├── 本地模式          完整並行 stack + 自動語言 dispatch，輸出終端機不發佈（blank / --from=<commit>）
│       ├── PR 模式           code-graph 前置 + 八通用 agent + 自動 dispatch 語言 agent 並行 + verification
│       ├── --focus           comments | tests | errors | types | code | simplify
│       └── --profile         chill（只 CRITICAL/HIGH）| assertive（預設，全等級）
│
├── Agents（27 個 agent）
│   ├── 通用主審
│   │   ├── code-reviewer          主審，含 false positive 過濾
│   │   ├── security-reviewer      OWASP Top 10，遇 CRITICAL 警報
│   │   └── verification-reviewer  HIGH/CRITICAL 二次確認（PR 模式 Phase 3.5）
│   ├── 前置分析
│   │   └── code-graph-analyzer    import deps + co-change risk，快取於 .claude/code-graph/
│   ├── PR 模式並行協作
│   │   ├── comment-analyzer  行內 comment 品質
│   │   ├── pr-test-analyzer  測試覆蓋
│   │   ├── silent-failure-hunter  swallowed error 偵測
│   │   ├── type-design-analyzer  型別設計
│   │   ├── code-simplifier   過度複雜實作
│   │   └── pr-walkthrough-writer  Walkthrough + Mermaid diagram
│   └── 語言 / 框架專項（17 個，由 /code-review 依副檔名自動 dispatch）
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

本地模式跑與 PR 模式**相同的完整 pipeline**（code-graph 前置 → 八通用 agent + 自動 dispatch 語言 agent 並行 → verification），只是結果輸出到終端機、不發佈。傳 `--from=<commit>` 可只審查自該 commit 以來的變更。

### 2. PR Review

支援 **GitHub** 和 **Bitbucket Cloud**，自動依 URL 或 git remote 偵測平台。

| 平台 | 前置條件 | 指令範例 |
|------|---------|---------|
| **GitHub** | 安裝 [`gh` CLI](https://cli.github.com/) | `/code-review 123` |
| **Bitbucket Cloud** | 設定 `BB_USERNAME` + `BB_APP_PASSWORD`（見下方） | `/code-review https://bitbucket.org/ws/repo/pull-requests/123` |

決策邏輯：

| 結果 | 條件 |
|------|------|
| **BLOCK** | 任一 CRITICAL finding |
| **REQUEST CHANGES** | 任一 HIGH finding 或 validation 失敗 |
| **APPROVE with comments** | 僅 MEDIUM / LOW finding |
| **APPROVE** | 零 finding |

#### Bitbucket 設定

在 `~/.zshenv` 加入（**必須是 App Password，非普通密碼**）：

```bash
export BB_USERNAME="your-bitbucket-username"
export BB_APP_PASSWORD="your-app-password"
```

App Password 建立：Bitbucket → Settings → Personal settings → App passwords

需勾選以下權限：
- **Repositories**: Read
- **Pull requests**: Read + Write（comment / approve / request-changes 必須）

### 3. 多 Agent 並行 PR Review

傳入 PR 號/URL 即進入 PR 模式：自動啟動 8 個通用 agent + 依變更檔案自動 dispatch 的語言/框架 agent，confidence < 80% 的 finding 自動過濾。可用 `--focus` 只跑特定面向：

```bash
/code-review 123
/code-review 123 --focus comments
/code-review 123 --focus tests
/code-review 123 --focus errors
/code-review 123 --focus types
/code-review 123 --focus code
/code-review 123 --focus simplify
```

### 4. 語言 / 框架自動 Dispatch

**不需要記每個語言的 command**。`/code-review` 會依變更檔案副檔名自動加上對應的專項 reviewer：

| 變更檔案 | 自動加入的 agent |
|---------|-----------------|
| `.ts` / `.tsx` / `.js` / `.vue` / `.svelte` | `typescript-reviewer` |
| `.py` | `python-reviewer`（含 `from django` → +`django-reviewer`、`from fastapi` → +`fastapi-reviewer`） |
| `.go` · `.rs` · `.kt` · `.swift` · `.java` · `.cs` · `.cpp`/`.c`/`.h` · `.fs` | 對應語言 reviewer |
| `.dart`（+ `pubspec.yaml`） | `flutter-reviewer` |
| `.sql` / migration 目錄 | `database-reviewer` |

偵測到才加，零匹配不浪費。`network-config` / `healthcare` / `mle` 為內容導向，依上下文相關性加入。

### 5. 直接呼叫 Agent

在對話中描述任務，Claude 會自動選用對應 agent：

```
@code-reviewer 請審查這段 auth middleware
@security-reviewer 這個 API endpoint 有安全問題嗎？
@typescript-reviewer 這個 generic type 設計合理嗎？
@database-reviewer 這個 migration 安全嗎？
```

---

## Slash Command

單一統一入口，依參數自動切換模式：

| 用法 | 行為 |
|------|------|
| `/code-review` | 本地 uncommitted 變更（完整並行 stack + 自動語言 dispatch，輸出到終端機不發佈） |
| `/code-review --from=<commit>` | 增量本地 review（只看自 `<commit>` 以來的變更） |
| `/code-review <PR 號 \| URL>` | PR 模式：八通用 agent + 自動 dispatch 語言 agent 並行 + verification（GitHub / Bitbucket 自動偵測） |
| `/code-review <PR> --focus <面向>` | 只跑 `comments` / `tests` / `errors` / `types` / `code` / `simplify` |
| `/code-review … --profile=chill` | 只輸出 CRITICAL / HIGH（預設 `assertive` 顯示全等級） |

---

## Agents

### 通用主審

| Agent | Color | 說明 |
|-------|-------|------|
| `code-reviewer` | 🟢 green | 主審，含嚴格 false positive 過濾，React / Node.js 專項規則 |
| `security-reviewer` | 🔴 red | OWASP Top 10 掃描，遇 CRITICAL 發緊急警報 |
| `verification-reviewer` | 🟠 orange | HIGH/CRITICAL finding 二次確認（三道關卡）；由 `/code-review` PR 模式 Phase 3.5 自動調用 |

### 前置分析

| Agent | Color | 說明 |
|-------|-------|------|
| `code-graph-analyzer` | 🔵 cyan | L2 import dependency + L3 co-change risk；並行 agents 前執行；快取於 `.claude/code-graph/` |

### `/code-review` PR 模式並行協作 Agents

| Agent | Color | 說明 |
|-------|-------|------|
| `comment-analyzer` | 🔵 cyan | 行內 comment 品質審查 |
| `pr-test-analyzer` | 🔵 cyan | 測試覆蓋分析 |
| `silent-failure-hunter` | 🔵 cyan | 偵測 swallowed error、ignored promise |
| `type-design-analyzer` | 🔵 cyan | 型別設計審查 |
| `code-simplifier` | 🔵 cyan | 找過度複雜的實作 |
| `pr-walkthrough-writer` | 🔵 blue | 生成結構化 PR walkthrough 與 Mermaid sequence diagram |

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
| `mle-reviewer` | 🟣 magenta | ML pipeline（Sonnet 模型） |

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
6. **Verification gate** — `/code-review` PR 模式 Phase 3.5 由 `verification-reviewer` 對所有 HIGH/CRITICAL finding 執行三道關卡二次確認；CONFIRMED finding 及 UNCERTAIN（降為 MEDIUM）的 finding 均進入最終報告；若本 PR diff 已修復問題，以「FIXED IN THIS PR」verdict 移除（不受 CRITICAL 保護限制）；CRITICAL finding 不可被完全移除，最多降為 HIGH
7. **NITPICK 分層** — 純風格偏好歸類為 NITPICK；`--profile=chill` 時略過 MEDIUM/LOW/NITPICK，`--profile=assertive`（預設）時顯示所有等級

---

## 目錄結構

```
code-review/
├── .claude-plugin/
│   ├── plugin.json          # Plugin manifest
│   └── marketplace.json     # Marketplace 發布設定
├── agents/                  # 27 個 agent
├── commands/                # 1 個統一 slash command（code-review.md）
└── skills/                  # 3 個 review skills
    ├── security-review/
    ├── security-scan/
    └── flutter-dart-code-review/
```

---

## License

MIT © [Terry Chen](https://github.com/jurislm)
