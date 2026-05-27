---
description: Comprehensive PR review using specialized agents (GitHub & Bitbucket)
argument-hint: [pr-number | pr-url] [--focus comments|tests|errors|types|code|simplify]
---

Run a comprehensive multi-perspective review of a pull request.

## Usage

`/review-pr [PR-number-or-URL] [--focus=comments|tests|errors|types|code|simplify]`

If no PR is specified, review the current branch's PR. If no focus is specified, run the full review stack.

## Steps

1. Identify the PR and detect platform:
   - URL contains `bitbucket.org` → use Bitbucket REST API (`curl -u "$BB_USERNAME:$BB_APP_PASSWORD"`)
   - URL contains `github.com` or no URL → use `gh pr view`
   - Number only → check `git remote get-url origin` to determine platform

   **GitHub:**
   ```bash
   gh pr view [number] --json number,title,body,author,baseRefName,headRefName,changedFiles
   gh pr diff [number]
   # Fetch linked issues for PR intent context (Fixes/Closes/Resolves/Related to #N)
   # Extracts all #N on each matching line (handles "Fixes #1, #2" correctly)
   gh pr view [number] --json body --jq '.body' | \
     perl -ne 'while (/(?:Fixes|Closes|Resolves|Related to)[^#]*#(\d+)/gi) { print "$1\n" }' | sort -u | \
     while read -r num; do gh issue view "$num" --json number,title,body 2>/dev/null; done
   ```

   **Bitbucket:**
   ```bash
   curl -s -u "$BB_USERNAME:$BB_APP_PASSWORD" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{slug}/pullrequests/{id}"
   curl -s -u "$BB_USERNAME:$BB_APP_PASSWORD" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{slug}/pullrequests/{id}/diff"
   ```

2. Find project guidance and trace call context:
   - Read `CLAUDE.md`, lint config, TypeScript config, repo conventions
   - For each modified exported function, class, or method identified in the diff, trace its callers:
     ```bash
     # -n outputs file:line:match, making it easy to pick the 3–5 most relevant call sites
     grep -r "SymbolName" --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" \
       --include="*.py" --include="*.go" --include="*.rs" -n . | head -20
     ```
     Read **3–5 most relevant call sites**. Flag any caller that makes assumptions about the changed behavior (return type, side effects, throw contract). Skip private helpers and test-only symbols.
   - Run fast static analysis and capture output as context for the parallel agents:
     ```bash
     # Append || true so pipefail environments don't abort context collection
     # Node.js/TS
     npx tsc --noEmit 2>&1 | head -60 || true
     npm run lint -- --format=compact 2>&1 | head -60 || true
     # Rust: cargo clippy 2>&1 | head -60 || true
     # Go:   go vet ./... 2>&1 | head -60 || true
     # Python: ruff check . 2>&1 | head -60 || true
     ```
     Pass linter output alongside the diff when launching each agent in Step 3.

2.5. **Code Impact Map** — Run `code-graph-analyzer` **sequentially** (wait for result before proceeding to Step 3). This pre-computation step provides cross-file dependency context that all parallel reviewers need.

   Pass to the agent:
   - List of LOGIC and SECURITY files from the diff (exclude DOCS/CONFIG/TEST)
   - Cache key: `pr-<number>-<first8charsOfHeadSha>` (e.g. `pr-123-abc12345`)

   The agent will:
   1. Check `.claude/code-graph/pr-<number>-<sha8>-impact-map.md` — return cached map if it exists
   2. Otherwise run L2 import dependency tracing + L3 co-change git analysis and write to cache
   3. Return the structured impact map markdown

   **Store the returned markdown as `IMPACT_MAP`.**

   When launching each parallel agent in Step 3, prepend the following to their prompt:
   ```text
   [IMPACT MAP — use as additional context when reviewing cross-file risk; do not repeat verbatim in your findings]
   <IMPACT_MAP content>
   [END IMPACT MAP]
   ```

   If `code-graph-analyzer` returns an error or times out, proceed to Step 3 without impact map context — do not block the review.

3. Run specialized review agents in parallel:
   - `code-reviewer`
   - `security-reviewer` — OWASP Top 10 analysis
   - `comment-analyzer`
   - `pr-test-analyzer`
   - `silent-failure-hunter`
   - `type-design-analyzer`
   - `code-simplifier`
   - `pr-walkthrough-writer` — generates structured walkthrough and Mermaid sequence diagram

3.5. **Verification pass** — Launch `verification-reviewer` with all HIGH and CRITICAL findings from Step 3. Wait for its output before proceeding. Only carry forward findings that survive verification (CONFIRMED status). CRITICAL findings from any agent are never dropped by this pass — they may be demoted to HIGH but not removed.

4. Verify and aggregate results:

   **Step 4a — Deduplicate**: Group findings by file + approximate line. Keep only one instance per issue.

   **Step 4b — Contradiction filter**: If two agents flag the same code for opposite reasons, or one flags while another explicitly approves, require ≥ 2 agents in agreement before including. **Exception**: CRITICAL-severity findings from any agent are always included regardless of agent agreement count.

   **Step 4c — Confidence filter**: Drop findings framed as "might", "possibly", or "consider" unless CRITICAL severity. Only report findings with ≥ 80% confidence.

   **Step 4d — False positive guard**:
   - Magic numbers in test fixtures or well-named constants → skip
   - Fire-and-forget patterns in tests or logging → skip
   - Style suggestions not agreed by ≥ 2 agents → downgrade to LOW or omit

   **Step 4e — Rank**: CRITICAL → HIGH → MEDIUM → LOW

5. Post dedicated walkthrough comment — this is the **first** thing developers see, posted before any findings:

   **5a — Build walkthrough content:**

   If `pr-walkthrough-writer` ran in Step 3, use its structured output directly. Otherwise generate inline using the rubric below.

   Effort score rubric (1–5):
   - **1** — Docs, config, or formatting only; no logic change
   - **2** — Small bug fix or minor feature; 1–3 logic files
   - **3** — Medium feature or refactor; 4–10 files, some logic complexity
   - **4** — Complex refactor, core logic rewrite, or >10 files changed
   - **5** — Architectural change, schema migration, auth system, or cross-cutting concern

   Build a one-row-per-file table: file path, Added/Modified/Deleted, one-sentence summary of what changed and why.

   **5b — Sequence diagram** (only when ≤10 files changed AND at least one non-test logic file is present):

   Use the `pr-walkthrough-writer` diagram if generated. If generating inline, trace the main data flow through the diff (entry → processing → data layer) and render as Mermaid. If no clear multi-layer flow is discernible, **omit entirely** — do not force a diagram.

   **5c — Post as first standalone comment:**

   **GitHub:**
   ```bash
   WALKTHROUGH_BODY="## 🔍 PR Walkthrough

   **Review Effort**: <N>/5 — <label> (<brief reason>)

   | File | Change | Summary |
   |------|--------|---------|
   | \`<file>\` | Added/Modified/Deleted | <one-sentence summary> |

   <mermaid_block_if_generated>"

   gh pr review <NUMBER> --comment --body "$WALKTHROUGH_BODY"
   ```

   **Bitbucket:**
   ```bash
   curl -s -X POST -u "$BB_USERNAME:$BB_APP_PASSWORD" \
     -H "Content-Type: application/json" \
     -d "{\"content\": {\"raw\": \"## PR Walkthrough\n\n<walkthrough_content>\"}}" \
     "https://api.bitbucket.org/2.0/repositories/$WORKSPACE/$REPO_SLUG/pullrequests/$PR_ID/comments"
   ```

6. Post findings with severity-based delivery (three steps):

   **6a — Inline comments for CRITICAL/HIGH** (post individually, max 10; excess joins the summary table in 6b):

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

   **GitHub:**
   ```bash
   HEAD_SHA=$(gh pr view <NUMBER> --json headRefOid --jq .headRefOid)
   gh api "repos/{owner}/{repo}/pulls/<NUMBER>/comments" \
     -f body="$COMMENT_BODY" \
     -f path="<file>" \
     -F line=<line_number> \
     -f side="RIGHT" \
     -f commit_id="$HEAD_SHA"
   ```

   **Bitbucket:**
   ```bash
   # Added/modified line → use "to"; removed line → use "from"
   curl -s -X POST -u "$BB_USERNAME:$BB_APP_PASSWORD" \
     -H "Content-Type: application/json" \
     -d "{\"content\":{\"raw\":\"$COMMENT_BODY\"},\"inline\":{\"path\":\"<file>\",\"to\":<line>}}" \
     "https://api.bitbucket.org/2.0/repositories/$WORKSPACE/$REPO_SLUG/pullrequests/$PR_ID/comments"
   ```

   **6b — Main review body** (MEDIUM findings as table + APPROVE/REQUEST CHANGES decision):

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
   Include: MEDIUM findings + any HIGH findings beyond the 10-inline limit.

   **GitHub:**
   ```bash
   # APPROVE — zero CRITICAL/HIGH, all validation passes
   gh pr review <NUMBER> --approve --body "$REVIEW_BODY"

   # REQUEST CHANGES — any HIGH or validation failure
   gh pr review <NUMBER> --request-changes --body "$REVIEW_BODY"

   # COMMENT — draft PR
   gh pr review <NUMBER> --comment --body "$REVIEW_BODY"
   ```

   **Bitbucket:**
   ```bash
   curl -X POST -u "$BB_USERNAME:$BB_APP_PASSWORD" \
     .../approve  # or .../request-changes
   curl -X POST -u "$BB_USERNAME:$BB_APP_PASSWORD" \
     -H "Content-Type: application/json" \
     -d "{\"content\":{\"raw\":\"$REVIEW_BODY\"}}" \
     .../comments
   ```

   **6c — Collapsible LOW/NITPICK** (if any; posted as a separate final comment):

   **GitHub:**
   ```bash
   gh pr review <NUMBER> --comment --body "<details>
   <summary>🔵 Low Priority / Style Suggestions (${LOW_COUNT})</summary>

   ${LOW_NITPICK_LIST}

   </details>"
   ```

   **Bitbucket** (no HTML folding — plain format):
   ```bash
   curl -X POST -u "$BB_USERNAME:$BB_APP_PASSWORD" \
     -H "Content-Type: application/json" \
     -d "{\"content\":{\"raw\":\"**Low Priority (${LOW_COUNT}):**\n\n${LOW_LIST}\"}}" \
     .../comments
   ```

7. Report findings grouped by severity

## Confidence Rule

Only report issues with confidence >= 80:

- Critical: bugs, security, data loss
- Important: missing tests, quality problems, style violations
- Advisory: suggestions only when explicitly requested
