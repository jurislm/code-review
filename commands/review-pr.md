---
description: Comprehensive PR review using specialized agents (GitHub & Bitbucket)
argument-hint: [pr-number | pr-url] [--focus security|performance|types|tests]
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
   gh pr view [number] --json body --jq '.body' | \
     grep -oP '(?i)(?:Fixes|Closes|Resolves|Related to)\s+#\K\d+' | sort -u | \
     xargs -I{} gh issue view {} --json number,title,body 2>/dev/null
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
     grep -r "SymbolName" --include="*.{ts,tsx,js,jsx,py,go,rs}" -l . | head -10
     ```
     Read **3–5 most relevant call sites**. Flag any caller that makes assumptions about the changed behavior (return type, side effects, throw contract). Skip private helpers and test-only symbols.

3. Run specialized review agents in parallel:
   - `code-reviewer`
   - `comment-analyzer`
   - `pr-test-analyzer`
   - `silent-failure-hunter`
   - `type-design-analyzer`
   - `code-simplifier`

4. Verify and aggregate results:

   **Step 4a — Deduplicate**: Group findings by file + approximate line. Keep only one instance per issue.

   **Step 4b — Contradiction filter**: If two agents flag the same code for opposite reasons, or one flags while another explicitly approves, require ≥ 2 agents in agreement before including.

   **Step 4c — Confidence filter**: Drop findings framed as "might", "possibly", or "consider" unless CRITICAL severity. Only report findings with ≥ 80% confidence.

   **Step 4d — False positive guard**:
   - Magic numbers in test fixtures or well-named constants → skip
   - Fire-and-forget patterns in tests or logging → skip
   - Style suggestions not agreed by ≥ 2 agents → downgrade to LOW or omit

   **Step 4e — Rank**: CRITICAL → HIGH → MEDIUM → LOW

5. Post results back to PR:
   - **GitHub**: `gh pr review [number] --approve|--request-changes|--comment --body "..."`
   - **Bitbucket**: `curl -X POST .../approve` or `.../request-changes`, plus `.../comments` for summary

6. Report findings grouped by severity

## Confidence Rule

Only report issues with confidence >= 80:

- Critical: bugs, security, data loss
- Important: missing tests, quality problems, style violations
- Advisory: suggestions only when explicitly requested
