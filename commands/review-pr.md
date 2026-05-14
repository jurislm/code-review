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
   ```

   **Bitbucket:**
   ```bash
   curl -s -u "$BB_USERNAME:$BB_APP_PASSWORD" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{slug}/pullrequests/{id}"
   curl -s -u "$BB_USERNAME:$BB_APP_PASSWORD" \
     "https://api.bitbucket.org/2.0/repositories/{workspace}/{slug}/pullrequests/{id}/diff"
   ```

2. Find project guidance:
   - look for `CLAUDE.md`, lint config, TypeScript config, repo conventions

3. Run specialized review agents in parallel:
   - `code-reviewer`
   - `comment-analyzer`
   - `pr-test-analyzer`
   - `silent-failure-hunter`
   - `type-design-analyzer`
   - `code-simplifier`

4. Aggregate results:
   - dedupe overlapping findings
   - rank by severity

5. Post results back to PR:
   - **GitHub**: `gh pr review [number] --approve|--request-changes|--comment --body "..."`
   - **Bitbucket**: `curl -X POST .../approve` or `.../request-changes`, plus `.../comments` for summary

6. Report findings grouped by severity

## Confidence Rule

Only report issues with confidence >= 80:

- Critical: bugs, security, data loss
- Important: missing tests, quality problems, style violations
- Advisory: suggestions only when explicitly requested
