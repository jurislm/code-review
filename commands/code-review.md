---
description: Code review — local uncommitted changes or GitHub/Bitbucket PR (pass PR number/URL for PR mode)
argument-hint: [pr-number | pr-url | blank for local review]
---

# Code Review

> PR review mode adapted from PRPs-agentic-eng by Wirasm. Part of the PRP workflow series.

**Input**: $ARGUMENTS

---

## Mode Selection

If `$ARGUMENTS` is blank:
→ Use **Local Review Mode**.

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
git diff --name-only HEAD
```

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

### Phase 2 — CONTEXT

Build review context:

1. **Project rules** — Read `CLAUDE.md`, `.claude/docs/`, and any contributing guidelines
2. **Planning artifacts** — Check `.claude/prds/`, `.claude/plans/`, `.claude/reviews/`, and legacy `.claude/PRPs/{prds,plans,reports,reviews}/` for context related to this PR
3. **PR intent** — Parse PR description for goals, linked issues, test plans
4. **Changed files** — List all modified files and categorize by type (source, test, config, docs)

### Phase 3 — REVIEW

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
npm run typecheck 2>/dev/null || npx tsc --noEmit 2>/dev/null  # Type check
npm run lint                                                    # Lint
npm test                                                        # Tests
npm run build                                                   # Build
```

**Rust** (has `Cargo.toml`):
```bash
cargo clippy -- -D warnings  # Lint
cargo test                   # Tests
cargo build                  # Build
```

**Go** (has `go.mod`):
```bash
go vet ./...    # Lint
go test ./...   # Tests
go build ./...  # Build
```

**Python** (has `pyproject.toml` / `setup.py`):
```bash
pytest  # Tests
```

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
- Draft PR → Always use **COMMENT** (not approve/block)
- Only docs/config changes → Lighter review, focus on correctness
- Explicit `--approve` or `--request-changes` flag → Override decision (but still report all findings)

### Phase 6 — REPORT

Create review artifact at `.claude/reviews/pr-<NUMBER>-review.md`:

```markdown
# PR Review: #<NUMBER> — <TITLE>

**Reviewed**: <date>
**Author**: <author>
**Branch**: <head> → <base>
**Decision**: APPROVE | REQUEST CHANGES | BLOCK

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

### Phase 7 — PUBLISH

Post the review to GitHub:

```bash
# If APPROVE
gh pr review <NUMBER> --approve --body "<summary of review>"

# If REQUEST CHANGES
gh pr review <NUMBER> --request-changes --body "<summary with required fixes>"

# If COMMENT only (draft PR or informational)
gh pr review <NUMBER> --comment --body "<summary>"
```

For inline comments on specific lines:
```bash
gh api "repos/{owner}/{repo}/pulls/<NUMBER>/comments" \
  -f body="<comment>" \
  -f path="<file>" \
  -F line=<line-number> \
  -f side="RIGHT" \
  -f commit_id="$(gh pr view <NUMBER> --json headRefOid --jq .headRefOid)"
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

---

## Bitbucket PR Review Mode

Comprehensive Bitbucket Cloud PR review via REST API v2.0.

**Prerequisites**: Set `BB_USERNAME` and `BB_APP_PASSWORD` environment variables.
Verify with: `echo $BB_USERNAME`

### Phase 1 — FETCH

Extract workspace, repo slug, and PR ID from input:

| Input | Action |
|---|---|
| Number (e.g. `42`) | Parse workspace/slug from `git remote get-url origin` |
| URL (`bitbucket.org/{ws}/{slug}/pull-requests/42`) | Extract ws, slug, ID directly |

Parse remote URL (supports both HTTPS and SSH formats):
```bash
REMOTE=$(git remote get-url origin)
# HTTPS: https://bitbucket.org/workspace/repo.git
# SSH:   git@bitbucket.org:workspace/repo.git
```

Fetch PR metadata:
```bash
curl -s -u "$BB_USERNAME:$BB_APP_PASSWORD" \
  "https://api.bitbucket.org/2.0/repositories/{workspace}/{slug}/pullrequests/{id}" \
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
  "https://api.bitbucket.org/2.0/repositories/{workspace}/{slug}/pullrequests/{id}/diff"
```

If credentials fail (HTTP 401), stop: "Set BB_USERNAME and BB_APP_PASSWORD environment variables."

### Phase 2 — CONTEXT

Same as GitHub PR Review Mode Phase 2 — read `CLAUDE.md`, planning artifacts, PR description.

### Phase 3 — REVIEW

Get changed files:
```bash
curl -s -u "$BB_USERNAME:$BB_APP_PASSWORD" \
  "https://api.bitbucket.org/2.0/repositories/{workspace}/{slug}/pullrequests/{id}/diffstat" \
  | jq '.values[] | {path: .new.path, status}'
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

Post review result to Bitbucket:

```bash
# Post overall review comment (all cases)
curl -s -X POST -u "$BB_USERNAME:$BB_APP_PASSWORD" \
  -H "Content-Type: application/json" \
  -d "{\"content\": {\"raw\": \"$REVIEW_SUMMARY\"}}" \
  "https://api.bitbucket.org/2.0/repositories/{workspace}/{slug}/pullrequests/{id}/comments"

# If APPROVE
curl -s -X POST -u "$BB_USERNAME:$BB_APP_PASSWORD" \
  "https://api.bitbucket.org/2.0/repositories/{workspace}/{slug}/pullrequests/{id}/approve"

# If REQUEST CHANGES or BLOCK
curl -s -X POST -u "$BB_USERNAME:$BB_APP_PASSWORD" \
  "https://api.bitbucket.org/2.0/repositories/{workspace}/{slug}/pullrequests/{id}/request-changes"
```

For inline comments on specific lines:
```bash
curl -s -X POST -u "$BB_USERNAME:$BB_APP_PASSWORD" \
  -H "Content-Type: application/json" \
  -d "{
    \"content\": {\"raw\": \"$COMMENT\"},
    \"inline\": {
      \"path\": \"$FILEPATH\",
      \"to\": $LINE_NUMBER
    }
  }" \
  "https://api.bitbucket.org/2.0/repositories/{workspace}/{slug}/pullrequests/{id}/comments"
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
