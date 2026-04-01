---
name: review-github-pr
description: Review a PR and submit suggestions as comments, along with an optional approval/request for changes.
compatibility: Requires git, gh, jq, and internet access.
model: opus
argument-hint: "[PR URL]"
---

**Read-only:** Do not modify any files. Your job is to review, report, and submit feedback on GitHub. Do not to make local changes.

**Before proceeding, verify prerequisites:**
- Run `gh auth status` to confirm GitHub CLI is authenticated. If not, stop and tell the user to run `gh auth login`.
- Confirm you have a PR or diff to review. If unclear, ask the user what to review before proceeding.

## Instructions

Review a GitHub pull request, generate prioritized inline comments, and submit them as a batch review.

Use when: reviewing, evaluating, critiquing, auditing, or assessing a GitHub PR. Triggers on requests like "review this PR", "review PR #123", or when given a PR URL to review.

**All GitHub interactions must use the `gh` CLI tool.** Do not use `curl`, direct API URLs, or web fetching. Always use `gh pr`, `gh api`, or other `gh` subcommands.

### 1. Fetch the PR diff

Use `gh` CLI to get the full diff and PR metadata:

```bash
gh pr view <PR> --json number,title,body,baseRefName,headRefName
gh pr diff <PR>
```

If the user provides a URL, extract the repo and PR number from it. For private repos, always use `gh` CLI rather than fetching URLs directly.

### 2. Analyze the entire diff

- Read the **complete diff**, not just individual chunks. Understand the full scope of changes.
- Consider how changes across multiple files relate to each other.
- Check for correctness, security, performance, readability, and maintainability.
- **Do not include fluff, praise, or "LGTM"-style commentary.** Only include actionable findings: critical issues (P0/P1) and nits (P2/P3). Every comment must point to a specific problem or concrete suggestion. Comments that merely note something is "a good change" or compliment the author are not actionable — omit them entirely.

### 3. Generate prioritized comments

Organize all findings into priority levels:

- **P0 — Blocking:** Bugs, security vulnerabilities, data loss risks, broken functionality. These must be fixed before merge.
- **P1 — Important:** Logic errors, missing edge cases, poor error handling, test gaps. Strongly recommended to fix.
- **P2 — Suggestion:** Code style improvements, better naming, minor refactors, documentation gaps.
- **P3 — Nit:** Trivial style preferences, optional improvements, minor observations.

Before finalizing, review all findings — including P3 nits — and deliberately decide whether each is worth including. Do not silently drop nits; either include them or note that they were considered and omitted. Try to frame feedback as questions.

**Present the full list of comments to the user, grouped and sorted by priority, before submitting.** Each comment should include:
- An index of the feedback to be used in step 4 below.
- Priority level (P0/P1/P2/P3).
- File path and line number.
- The comment text.

### 4. Ask for review disposition

Ask the user if they want to submit all feedback, or just some, and how they want to submit the review:
- **Approve** — approve the PR with the comments.
- **Request changes** — request changes (appropriate when there are P0 or P1 items).
- **Comment** — submit comments without approving or requesting changes.

The user may skip submitting certain feedback by index (provided in part 3).

### 5. Submit as a single batch review

Construct a JSON payload and submit all comments as a single review using `gh api` with `--input`.

**Always write the JSON to a temp file** rather than piping via `echo` — comment bodies often contain single quotes, backticks, and other characters that break shell quoting:

```bash
cat <<'PAYLOAD' > /tmp/pr_review.json
<json_payload>
PAYLOAD
gh api repos/{owner}/{repo}/pulls/{number}/reviews --method POST --input /tmp/pr_review.json
```

JSON payload format:

```json
{
  "event": "APPROVE|REQUEST_CHANGES|COMMENT",
  "body": "",
  "comments": [
    {
      "path": "relative/file/path.ext",
      "line": 42,
      "side": "RIGHT",
      "body": "**[P0]** Comment text here"
    }
  ]
}
```

Every comment body **must** start with the priority as a bold prefix: `**[P0]**`, `**[P1]**`, `**[P2]**`, or `**[P3]**`.

For the review body/summary, leave it **empty** (`"body": ""`). Do not include summaries, priority counts (e.g. "1 P1, 3 P2"), fluff, filler, or praise. All feedback belongs in inline comments only.

**Important — use `line` and `side`, NOT `position`:**
- `line`: The **actual file line number** in the new version of the file on the PR branch. Do NOT try to calculate this from diff hunk headers — hunk offsets are error-prone. Instead, fetch the real file from the PR branch to confirm each line number before submitting:
  ```bash
  gh api "repos/{owner}/{repo}/contents/{path}?ref={head_branch}" --jq '.content' | base64 -d | cat -n | sed -n '{start},{end}p'
  ```
- **Only fetch files whose paths appear in the diff.** Do not guess or infer file paths that are not explicitly listed in the `gh pr diff` output — they may not exist, leading to 404 errors.
- `side`: Always `"RIGHT"` for comments on new/changed lines.
- Do **not** use the deprecated `position` field — it refers to a line's offset within the diff hunk and is error-prone across multi-hunk files.

Return the review URL to the user when complete.
