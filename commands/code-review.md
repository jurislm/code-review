---
description: Code review — local uncommitted changes or GitHub/Bitbucket PR (pass PR number/URL for PR mode)
argument-hint: [pr-number | pr-url | --from=<commit> | --profile=chill|assertive | blank for local review]
---

# Code Review

> PR review mode adapted from PRPs-agentic-eng by Wirasm. Part of the PRP workflow series.

**Input**: $ARGUMENTS

---

## Mode Selection

If `$ARGUMENTS` is blank:
→ Use **Local Review Mode**.

If `$ARGUMENTS` contains `--from=<commit>`:
→ Use **Incremental Local Review Mode** — same as Local Review Mode but only reviews files changed since `<commit>` (e.g. `--from=main`, `--from=HEAD~3`, `--from=abc1234`).

If `$ARGUMENTS` contains `--profile=chill`:
→ **Chill profile** — output only CRITICAL and HIGH findings. Skip MEDIUM, LOW, and NITPICK entirely. Best for fast-moving feature branches.

If `$ARGUMENTS` contains `--profile=assertive` or no `--profile` flag:
→ **Assertive profile** (default) — output all severity levels including NITPICK. Best for PR reviews targeting main/release branches.

If `$ARGUMENTS` contains a PR number, PR URL, or `--pr`:

1. Detect platform:
   - URL contains `bitbucket.org` → **Bitbucket PR Review Mode**
   - URL contains `github.com` → **GitHub PR Review Mode**
   - Number only → run `git remote get-url origin`:
     - Contains `bitbucket.org` → **Bitbucket PR Review Mode**
     - Otherwise → **GitHub PR Review Mode**

---

## Local Review Mode

Comprehensive security and quality review of uncommitted changes.

### Phase 1 — GATHER

```bash
# Full local review (default — no --from flag)
git diff --name-only HEAD

# Incremental review (--from=<commit> specified)
git diff --name-only <commit>
```

Use the incremental form when `--from=<commit>` is specified. In subsequent phases, use `git diff <commit>` (not `git diff <commit>..HEAD`) wherever a diff is needed — this includes any uncommitted working tree changes relative to `<commit>`, matching the Local Review Mode behaviour.

If no changed files, stop: "Nothing to review."

### Phase 2 — REVIEW

Read each changed file in full. Check for:

**Security Issues (CRITICAL):**
- Hardcoded credentials, API keys, tokens
- SQL injection vulnerabilities
- XSS vulnerabilities
- Missing input validation
- Insecure dependencies
- Path traversal risks

**Code Quality (HIGH):**
- Functions > 50 lines
- Files > 800 lines
- Nesting depth > 4 levels
- Missing error handling
- console.log statements
- TODO/FIXME comments
- Missing JSDoc for public APIs

**Best Practices (MEDIUM):**
- Mutation patterns (use immutable instead)
- Emoji usage in code/comments
- Missing tests for new code
- Accessibility issues (a11y)

### Phase 3 — REPORT

Generate report with:
- Severity: CRITICAL, HIGH, MEDIUM, LOW
- File location and line numbers
- Issue description
- Suggested fix

Block commit if CRITICAL or HIGH issues found.
Never approve code with security vulnerabilities.

---

## GitHub PR Review Mode

Comprehensive GitHub PR review — fetches diff, reads full files, runs validation, posts review.

### Phase 1 — FETCH

Parse input to determine PR:

| Input | Action |
|---|---|
| Number (e.g. `42`) | Use as PR number |
| URL (`github.com/.../pull/42`) | Extract PR number |
| Branch name | Find PR via `gh pr list --head <branch>` |

```bash
gh pr view <NUMBER> --json number,title,body,author,baseRefName,headRefName,changedFiles,additions,deletions
gh pr diff <NUMBER>
```

If PR not found, stop with error. Store PR metadata for later phases.

### Phase 1.5 — CLASSIFY

**Incremental review detection**: Check if this PR was previously reviewed; skip if no new commits:

```bash
LAST_COMMIT_FILE=".claude/reviews/pr-<NUMBER>-last-commit.txt"
CURRENT_HEAD=$(gh pr view <NUMBER> --json headRefOid --jq '.headRefOid')
if [ -f "$LAST_COMMIT_FILE" ]; then
  LAST_COMMIT=$(cat "$LAST_COMMIT_FILE")
  if [ "$LAST_COMMIT" = "$CURRENT_HEAD" ]; then
    echo "No new commits since last review. Nothing to do."
    exit 0
  fi
  echo "New commits detected since last review ($LAST_COMMIT → $CURRENT_HEAD). Running full review."
fi
# LAST_COMMIT_FILE is written at the end of Phase 8 so the next run can skip if unchanged
```

> Note: This is a **skip-if-unchanged** gate, not a diff-scoping mechanism. When new commits exist, the full PR diff is always reviewed. This ensures no regressions are missed from rebases or force-pushes.

**File classification**: Route between Fast Path and Slow Path:

```bash
CHANGED_FILES=$(gh pr diff <NUMBER> --name-only)
FAST_ONLY=true
LOGIC_FILES=""
SECURITY_FILES=""
while IFS= read -r file; do
  case "$file" in
    *.md|*.txt|*.rst|*.adoc|CHANGELOG*|LICENSE*|README*)
      echo "DOCS: $file" ;;
    *.json|*.yaml|*.yml|*.toml|*package-lock*|*.lock|*.sum)
      echo "CONFIG: $file" ;;
    *auth*|*crypto*|*token*|*password*|*session*|*jwt*|*oauth*|*secret*)
      echo "SECURITY: $file"; SECURITY_FILES="${SECURITY_FILES:+$SECURITY_FILES }$file"; FAST_ONLY=false ;;
    *test*|*spec*|*__tests__*|*.test.*|*.spec.*)
      echo "TEST: $file" ;;
    *.ts|*.tsx|*.js|*.jsx|*.py|*.go|*.rs|*.kt|*.swift|*.java|*.cs|*.rb|*.php|\
*.cpp|*.cc|*.c|*.h|*.hpp|*.dart|*.scala|*.ex|*.exs|*.lua|*.vue|*.svelte)
      echo "LOGIC: $file"; LOGIC_FILES="${LOGIC_FILES:+$LOGIC_FILES }$file"; FAST_ONLY=false ;;
    *)
      echo "OTHER: $file" ;;
  esac
done <<< "$CHANGED_FILES"
```

`$LOGIC_FILES` and `$SECURITY_FILES` are now available as space-separated lists for Phase 2.5.

**Fast Path** (only DOCS + CONFIG + OTHER — zero LOGIC/SECURITY files):
- Skip Phases 2, 3, 4, and 5 entirely
- Run **secret scan only**: execute the static analysis / secret scan commands from Phase 2 Step 6 (linters + credential patterns)
- If zero secrets or credentials found: APPROVE with brief note "Docs/config changes — no logic review needed", then proceed directly to Phase 6
- If a secret/credential is detected: exit Fast Path and run the full Slow Path from Phase 2 onwards

**Slow Path** (any LOGIC or SECURITY file present): proceed to Phase 2 normally.

**SECURITY files**: in Phase 3, pass `$SECURITY_FILES` explicitly to `security-reviewer` for prioritized analysis.

### Phase 2 — CONTEXT

Build review context:

1. **Project rules** — Read `CLAUDE.md`, `.claude/docs/`, and any contributing guidelines. Also load path-based review rules if present:
   ```bash
   [ -f ".claude/review-paths.yaml" ] && cat ".claude/review-paths.yaml"
   ```
   This is an **optional, user-created** file (not bundled with the plugin). If present, find the most specific matching `pattern` for each changed file and include its `rules` in the review prompt. Files matching a `skip` list have those check categories suppressed. Example schema to create in your project:
   ```yaml
   paths:
     - pattern: "src/api/**"
       focus: security
       rules: ["All public endpoints must require authentication", "Rate limiting required on POST"]
     - pattern: "src/db/**"
       focus: performance
       rules: ["No queries without LIMIT", "No string concatenation in SQL"]
     - pattern: "**/*.test.ts"
       skip: [magic_numbers, function_length, console_log]
   ```
2. **Planning artifacts** — Check `.claude/prds/`, `.claude/plans/`, `.claude/reviews/`, and legacy `.claude/PRPs/{prds,plans,reports,reviews}/` for context related to this PR
3. **PR intent** — Parse PR description for goals, linked issues, test plans. Explicitly fetch any linked issues:
   ```bash
   # Extracts all #N on each matching line (handles "Fixes #1, #2" correctly)
   gh pr view <NUMBER> --json body --jq '.body' | \
     perl -ne 'while (/(?:Fixes|Closes|Resolves|Related to)[^#]*#(\d+)/gi) { print "$1\n" }' | sort -u | \
     while read -r num; do gh issue view "$num" --json number,title,body 2>/dev/null; done
   ```
   Include issue title and description as review context — understanding PR intent reduces false positives.
4. **Changed files** — List all modified files and categorize by type (source, test, config, docs)
5. **Caller tracing** — For each modified exported function, class, or method in the diff, find and read its call sites:
   ```bash
   # -n outputs file:line:match, making it easy to pick the 3–5 most relevant callers
   grep -r "SymbolName" --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" \
     --include="*.py" --include="*.go" --include="*.rs" -n . | head -20
   ```
   Read **3–5 most relevant callers**. Document any that make behavioral assumptions about the changed code (return type, thrown errors, side effects). Skip private/internal symbols used only within the same file.
6. **Static analysis — run before review, capture as context** — Execute fast linters and type checker now. Do not fail on findings; treat output as signals for Phase 3:

   **Node.js / TypeScript:**
   ```bash
   # Append || true so pipefail environments don't abort context collection
   npx tsc --noEmit 2>&1 | head -60 || true
   npm run lint -- --format=compact 2>&1 | head -60 || true
   ```
   **Rust:** `cargo clippy 2>&1 | head -60 || true`
   **Go:** `go vet ./... 2>&1 | head -60 || true`
   **Python:** `ruff check . 2>&1 | head -60 || true`

   Store this output. Any `file:line` flagged here → review that location with elevated priority in Phase 3.

### Phase 2.5 — CODE GRAPH

Run `code-graph-analyzer` **sequentially** (wait for result before starting Phase 3). This provides cross-file dependency context unavailable from the diff alone.

Pass to the agent:
- `$LOGIC_FILES` and `$SECURITY_FILES` space-separated lists from Phase 1.5 (excludes DOCS/CONFIG/TEST)
- Cache key: `pr-<number>-<first8charsOfHeadSha>` for PR mode; `local-<sha8>` for local diff mode (use `git rev-parse --short=8 HEAD`)

The agent checks `.claude/code-graph/<cache_key>-impact-map.md` first — returns cached map if the SHA matches. Otherwise computes L2 import dependencies and L3 co-change risk, then writes to `.claude/code-graph/`.

**Store the returned markdown as `IMPACT_MAP`.** Pass it as context to Phase 3 review, annotating findings that involve files mentioned in the "High-risk dependents" or "Missing co-changes" sections with elevated priority.

If `code-graph-analyzer` returns an error or times out, proceed to Phase 3 without impact map context — do not block the review.

7. **CI check status** — Read the current CI check results to surface known failures as review context:
   ```bash
   gh pr checks <NUMBER> 2>/dev/null | head -30
   ```
   If any checks are failing, note them at the top of Phase 3 with `⚠️ CI failing: <check_name>` and treat the related code paths as elevated-priority review areas. Do not block the review — just elevate priority.

### Phase 3 — REVIEW

**Cross-reference static analysis signals first** — Check the linter/typecheck output captured in Phase 2 Step 6. For any `file:line` already flagged:
- Treat as elevated-confidence finding (linter + code review = double signal)
- Include the linter rule in the finding description
- Skip re-flagging if the linter message is already precise and actionable

Read each changed file **in full** (not just the diff hunks — you need surrounding context).

For PR reviews, fetch the full file contents at the PR head revision:
```bash
gh pr diff <NUMBER> --name-only | while IFS= read -r file; do
  gh api "repos/{owner}/{repo}/contents/$file?ref=<head-branch>" --jq '.content' | base64 -d
done
```

Apply the review checklist across 7 categories:

| Category | What to Check |
|---|---|
| **Correctness** | Logic errors, off-by-ones, null handling, edge cases, race conditions |
| **Type Safety** | Type mismatches, unsafe casts, `any` usage, missing generics |
| **Pattern Compliance** | Matches project conventions (naming, file structure, error handling, imports) |
| **Security** | Injection, auth gaps, secret exposure, SSRF, path traversal, XSS |
| **Performance** | N+1 queries, missing indexes, unbounded loops, memory leaks, large payloads |
| **Completeness** | Missing tests, missing error handling, incomplete migrations, missing docs |
| **Maintainability** | Dead code, magic numbers, deep nesting, unclear naming, missing types |

Assign severity to each finding:

| Severity | Meaning | Action |
|---|---|---|
| **CRITICAL** | Security vulnerability or data loss risk | Must fix before merge |
| **HIGH** | Bug or logic error likely to cause issues | Should fix before merge |
| **MEDIUM** | Code quality issue or missing best practice | Fix recommended |
| **LOW** | Style nit or minor suggestion | Optional |

### Phase 4 — VALIDATE

Run available validation commands:

Detect the project type from config files (`package.json`, `Cargo.toml`, `go.mod`, `pyproject.toml`, etc.), then run the appropriate commands:

**Node.js / TypeScript** (has `package.json`):
```bash
npm test        # Tests
npm run build   # Build
```

**Rust** (has `Cargo.toml`):
```bash
cargo test   # Tests
cargo build  # Build
```

**Go** (has `go.mod`):
```bash
go test ./...   # Tests
go build ./...  # Build
```

**Python** (has `pyproject.toml` / `setup.py`):
```bash
pytest  # Tests
```

> lint + typecheck already ran in Phase 2 Step 6 — record those results here; only re-run test + build.

Run only the commands that apply to the detected project type. Record pass/fail for each.

### Phase 5 — DECIDE

Form recommendation based on findings:

| Condition | Decision |
|---|---|
| Zero CRITICAL/HIGH issues, validation passes | **APPROVE** |
| Only MEDIUM/LOW issues, validation passes | **APPROVE** with comments |
| Any HIGH issues or validation failures | **REQUEST CHANGES** |
| Any CRITICAL issues | **BLOCK** — must fix before merge |

Special cases:
- Draft PR (GitHub only) → Always use **COMMENT** (not approve/block). Note: Bitbucket Cloud has no draft state — states are OPEN/MERGED/DECLINED/SUPERSEDED only.
- Only docs/config changes → Lighter review, focus on correctness
- Explicit `--approve` or `--request-changes` flag → Override decision (but still report all findings)

**Count findings by severity** and store for Phase 6.5:
```bash
CRITICAL_COUNT=<number of CRITICAL findings>
HIGH_COUNT=<number of HIGH findings>
MEDIUM_COUNT=<number of MEDIUM findings>
LOW_COUNT=<number of LOW findings>
```

### Phase 6 — REPORT

Create review artifact at `.claude/reviews/pr-<NUMBER>-review.md`:

```markdown
# PR Review: #<NUMBER> — <TITLE>

**Reviewed**: <date>
**Author**: <author>
**Branch**: <head> → <base>
**Decision**: APPROVE | REQUEST CHANGES | BLOCK

## Walkthrough

| File | Change | Summary |
|------|--------|---------|
| `<file>` | Added / Modified / Deleted | <one-sentence description of what changed and why> |

## Summary
<1-2 sentence overall assessment>

## Findings

### CRITICAL
<findings or "None">

### HIGH
<findings or "None">

### MEDIUM
<findings or "None">

### LOW
<findings or "None">

## Validation Results

| Check | Result |
|---|---|
| Type check | Pass / Fail / Skipped |
| Lint | Pass / Fail / Skipped |
| Tests | Pass / Fail / Skipped |
| Build | Pass / Fail / Skipped |

## Files Reviewed
<list of files with change type: Added/Modified/Deleted>
```

### Phase 6.5 — UPDATE PR DESCRIPTION

Auto-append a structured review summary to the PR description. Idempotent — strips any previously generated block before appending:

```bash
CURRENT_BODY=$(gh pr view <NUMBER> --json body --jq '.body')

STRIPPED=$(echo "$CURRENT_BODY" | python3 -c "
import sys, re
body = sys.stdin.read()
body = re.sub(r'<!-- cr-summary:start -->.*?<!-- cr-summary:end -->', '', body, flags=re.DOTALL)
body = re.sub(r'<!-- cr-summary:start -->', '', body)
print(body.strip())
")

SUMMARY_BLOCK="<!-- cr-summary:start -->

---
## 📋 Code Review Summary

| Category | Files |
|---|---|
| Logic / Feature | <logic_files> |
| Tests | <test_files> |
| Config / Docs | <config_doc_files> |

**Reviewed**: $(date +%Y-%m-%d) · **Findings**: ${CRITICAL_COUNT} critical · ${HIGH_COUNT} high · ${MEDIUM_COUNT} medium
<!-- cr-summary:end -->"

printf '%s\n\n%s' "$STRIPPED" "$SUMMARY_BLOCK" \
  | gh pr edit <NUMBER> --body-file -
```

> Skip this phase for Bitbucket (REST API does not support PR description updates via the same CLI workflow).

### Phase 7 — PUBLISH

Post the review to GitHub using severity-based delivery (three steps):

**Step 7a — Inline comments for CRITICAL/HIGH** (post individually, max 10; excess joins the summary table in Step 7b):

For each CRITICAL/HIGH finding with a specific `file:line`, build the comment:
```text
**[{SEVERITY}] {issue_title}**

{concrete_failure_scenario}

**Why existing guards don't catch it:** {guard_gap}
```
If the fix is a single-line replacement, append a committable suggestion block so the author can apply it with one click:
````markdown
```suggestion
{fixed_line_content}
```
````

```bash
HEAD_SHA=$(gh pr view <NUMBER> --json headRefOid --jq .headRefOid)
gh api "repos/{owner}/{repo}/pulls/<NUMBER>/comments" \
  -f body="$COMMENT_BODY" \
  -f path="<file>" \
  -F line=<line_number> \
  -f side="RIGHT" \
  -f commit_id="$HEAD_SHA"
```

**Step 7b — Main review body** (MEDIUM findings as table + overall decision):

> **Profile gate**: if `--profile=chill` was specified, skip the MEDIUM section entirely — include only the severity summary counts and the overall decision. Proceed directly to the `gh pr review` command without listing MEDIUM findings.

Build `$REVIEW_BODY`:
```markdown
## 📊 Review Summary

| Severity | Count |
|---|---|
| 🔴 CRITICAL | {n} |
| 🟠 HIGH | {n} |
| 🟡 MEDIUM | {n} |
| 🔵 LOW | {n} |

## ⚠️ Issues Requiring Attention

| File:Line | Issue | Suggested Fix |
|---|---|---|
| `file:line` | description | fix |
```
Include MEDIUM findings (assertive profile only). Also include HIGH findings beyond the 10-inline limit.

```bash
# APPROVE — zero CRITICAL/HIGH issues, validation passes
gh pr review <NUMBER> --approve --body "$REVIEW_BODY"

# REQUEST CHANGES — any HIGH issues or validation failures
gh pr review <NUMBER> --request-changes --body "$REVIEW_BODY"

# COMMENT — draft PR or informational
gh pr review <NUMBER> --comment --body "$REVIEW_BODY"
```

**Step 7c — Collapsible LOW/NITPICK** (if any; posted as a separate final comment):

> **Profile gate**: skip Step 7c entirely when `--profile=chill` — LOW and NITPICK findings are suppressed.

```bash
gh pr review <NUMBER> --comment --body "<details>
<summary>🔵 Low Priority / Style Suggestions (${LOW_COUNT})</summary>

${LOW_NITPICK_LIST}

</details>"
```

### Phase 8 — OUTPUT

Report to user:

```
PR #<NUMBER>: <TITLE>
Decision: <APPROVE|REQUEST_CHANGES|BLOCK>

Issues: <critical_count> critical, <high_count> high, <medium_count> medium, <low_count> low
Validation: <pass_count>/<total_count> checks passed

Artifacts:
  Review: .claude/reviews/pr-<NUMBER>-review.md
  GitHub: <PR URL>
```

Save current HEAD commit for incremental review tracking (next review will only process new commits):

**Only write this file if Phase 7 completed successfully** (the `gh pr review` command in Step 7b returned exit code 0). If Phase 7 failed for any reason (network error, expired token, API error), skip writing this file so the next run retries from scratch rather than silently skipping the PR.

```bash
# Only run if Phase 7 succeeded
mkdir -p .claude/reviews
gh pr view <NUMBER> --json headRefOid --jq '.headRefOid' \
  > ".claude/reviews/pr-<NUMBER>-last-commit.txt"
```

---

## Bitbucket PR Review Mode

Comprehensive Bitbucket Cloud PR review via REST API v2.0.

**Prerequisites**:
- `BB_USERNAME` — Bitbucket 帳號名稱
- `BB_APP_PASSWORD` — Bitbucket App Password（**非普通密碼**）

App Password 需要以下權限：
- Repositories: Read
- Pull requests: Read + Write（comment / approve / request-changes 必須）

建立路徑：Bitbucket → Settings → Personal settings → App passwords

### Phase 1 — FETCH

Extract workspace, repo slug, and PR ID from input:

| Input | Action |
|---|---|
| Number (e.g. `42`) | Parse workspace/slug from `git remote get-url origin` |
| URL (`bitbucket.org/{ws}/{slug}/pull-requests/42`) | Extract ws, slug, ID directly |

Parse remote URL（同時支援 HTTPS 和 SSH 格式）：
```bash
REMOTE=$(git remote get-url origin)

# HTTPS: https://bitbucket.org/workspace/repo.git
# SSH:   git@bitbucket.org:workspace/repo.git

# 統一提取 workspace/repo_slug
PATH_PART=$(echo "$REMOTE" \
  | sed 's|https://bitbucket.org/||' \
  | sed 's|git@bitbucket.org:||' \
  | sed 's|\.git$||')

WORKSPACE=$(echo "$PATH_PART" | cut -d/ -f1)
REPO_SLUG=$(echo "$PATH_PART" | cut -d/ -f2)
```

Fetch PR metadata:
```bash
curl -s -u "$BB_USERNAME:$BB_APP_PASSWORD" \
  "https://api.bitbucket.org/2.0/repositories/$WORKSPACE/$REPO_SLUG/pullrequests/$PR_ID" \
  | jq '{
      title,
      description,
      author: .author.display_name,
      source_branch: .source.branch.name,
      dest_branch: .destination.branch.name,
      head_commit: .source.commit.hash
    }'
```

Fetch diff:
```bash
curl -s -u "$BB_USERNAME:$BB_APP_PASSWORD" \
  "https://api.bitbucket.org/2.0/repositories/$WORKSPACE/$REPO_SLUG/pullrequests/$PR_ID/diff"
```

If credentials fail (HTTP 401):
- 確認使用的是 App Password（非 Bitbucket 帳號密碼）
- 確認 App Password 有 Repositories: Read 和 Pull requests: Read+Write 權限
- 測試：`curl -u "$BB_USERNAME:$BB_APP_PASSWORD" "https://api.bitbucket.org/2.0/user"`

### Phase 2 — CONTEXT

Same as GitHub PR Review Mode Phase 2 — read `CLAUDE.md`, planning artifacts, PR description.

### Phase 3 — REVIEW

Get changed files（分頁：若 response 含 `next` 欄位，繼續 fetch 直到 `next` 為 null）：
```bash
curl -s -u "$BB_USERNAME:$BB_APP_PASSWORD" \
  "https://api.bitbucket.org/2.0/repositories/{workspace}/{slug}/pullrequests/{id}/diffstat" \
  | jq '.values[] | {path: (.new.path // .old.path), status}'
# status 值：added | modified | removed | renamed
# .new 在 removed 時為 null，需 fallback 到 .old.path
```

Read each changed file in full at the PR head commit:
```bash
curl -s -u "$BB_USERNAME:$BB_APP_PASSWORD" \
  "https://api.bitbucket.org/2.0/repositories/{workspace}/{slug}/src/{head_commit}/{filepath}"
```

Apply the same 7-category review checklist as GitHub PR Review Mode Phase 3.

### Phase 4 — VALIDATE

Same as GitHub PR Review Mode Phase 4 — detect project type and run local validation commands.

### Phase 5 — DECIDE

Same decision logic as GitHub PR Review Mode Phase 5.

### Phase 6 — REPORT

Create review artifact at `.claude/reviews/pr-<ID>-review.md` using the same format as GitHub PR Review Mode Phase 6.

### Phase 7 — PUBLISH

Post review result to Bitbucket using severity-based delivery:

**Step 7a — Inline comments for CRITICAL/HIGH** (max 10; excess joins Step 7b):

```bash
# For each CRITICAL/HIGH finding — build comment body first
COMMENT_BODY="**[{SEVERITY}] {issue_title}**\n\n{failure_scenario}\n\n**Why existing guards don't catch it:** {guard_gap}"

# Added/modified line → use "to"
curl -s -X POST -u "$BB_USERNAME:$BB_APP_PASSWORD" \
  -H "Content-Type: application/json" \
  -d "{\"content\":{\"raw\":\"$COMMENT_BODY\"},\"inline\":{\"path\":\"$FILEPATH\",\"to\":$LINE_NUMBER}}" \
  "https://api.bitbucket.org/2.0/repositories/$WORKSPACE/$REPO_SLUG/pullrequests/$PR_ID/comments"

# Removed line → use "from"
curl -s -X POST -u "$BB_USERNAME:$BB_APP_PASSWORD" \
  -H "Content-Type: application/json" \
  -d "{\"content\":{\"raw\":\"$COMMENT_BODY\"},\"inline\":{\"path\":\"$FILEPATH\",\"from\":$LINE_NUMBER}}" \
  "https://api.bitbucket.org/2.0/repositories/$WORKSPACE/$REPO_SLUG/pullrequests/$PR_ID/comments"
```

**Step 7b — Main review comment** (MEDIUM table + approve/request-changes decision):

Build `$REVIEW_SUMMARY` as a Markdown table of MEDIUM findings plus HIGH overflow, then post:

```bash
# Post summary comment (all cases)
curl -s -X POST -u "$BB_USERNAME:$BB_APP_PASSWORD" \
  -H "Content-Type: application/json" \
  -d "{\"content\": {\"raw\": \"$REVIEW_SUMMARY\"}}" \
  "https://api.bitbucket.org/2.0/repositories/$WORKSPACE/$REPO_SLUG/pullrequests/$PR_ID/comments"

# If APPROVE（回傳 200 + JSON: {approved: true, user, role, participated_on}）
curl -s -X POST -u "$BB_USERNAME:$BB_APP_PASSWORD" \
  "https://api.bitbucket.org/2.0/repositories/$WORKSPACE/$REPO_SLUG/pullrequests/$PR_ID/approve"

# If REQUEST CHANGES or BLOCK（回傳 204 No Content，無 JSON body）
curl -s -o /dev/null -w "%{http_code}" -X POST -u "$BB_USERNAME:$BB_APP_PASSWORD" \
  "https://api.bitbucket.org/2.0/repositories/$WORKSPACE/$REPO_SLUG/pullrequests/$PR_ID/request-changes"
```

**Step 7c — Low priority findings** (if any; Bitbucket has no HTML folding — use plain format):

```bash
LOW_BODY="**Low Priority / Style Suggestions (${LOW_COUNT}):**\n\n${LOW_NITPICK_LIST}"
curl -s -X POST -u "$BB_USERNAME:$BB_APP_PASSWORD" \
  -H "Content-Type: application/json" \
  -d "{\"content\": {\"raw\": \"$LOW_BODY\"}}" \
  "https://api.bitbucket.org/2.0/repositories/$WORKSPACE/$REPO_SLUG/pullrequests/$PR_ID/comments"
```

### Phase 8 — OUTPUT

Report to user:

```
PR #<ID>: <TITLE>
Platform: Bitbucket Cloud
Decision: <APPROVE|REQUEST_CHANGES|BLOCK>

Issues: <critical_count> critical, <high_count> high, <medium_count> medium, <low_count> low
Validation: <pass_count>/<total_count> checks passed

Artifacts:
  Review: .claude/reviews/pr-<ID>-review.md
  Bitbucket: https://bitbucket.org/{workspace}/{slug}/pull-requests/{id}
```

---

## Edge Cases

- **No `gh` CLI (GitHub mode)**: Fall back to local-only review, skip publish. Warn user.
- **Missing BB credentials (Bitbucket mode)**: Stop with instructions to set `BB_USERNAME` and `BB_APP_PASSWORD`.
- **Diverged branches**: Suggest `git fetch origin && git rebase origin/<base>` before review.
- **Large PRs (>50 files)**: Warn about review scope. Focus on source changes first, then tests, then config/docs.
