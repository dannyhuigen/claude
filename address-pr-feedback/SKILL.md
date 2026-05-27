---
name: address-pr-feedback
description: Read all comments on a GitHub PR, draft fixes in a feedback.md, implement each as its own commit, then reply to each comment with the implementing commit URL. Use when user says "address PR feedback", "implement PR comments", or provides a PR URL with a request to action its review comments.
allowed-tools: Bash(gh:*), Bash(git:*), Bash(go:*), Bash(jq:*), Bash(open:*), Read, Edit, Write
---

Address every review/issue comment on a GitHub PR by drafting solutions, implementing them as individual commits, and replying to each comment with the implementing commit URL.

`$ARGUMENTS` = PR URL (e.g. `https://github.com/breeze-social/api/pull/6167`).

## Step 1: Parse PR + fetch metadata

Extract `owner`, `repo`, `pr_number` from URL. Then:

```bash
gh pr view <pr_number> --repo <owner>/<repo> --json number,title,headRefName,baseRefName,state,url,author
```

If PR closed/merged, ask user before continuing.

## Step 2: Checkout PR branch

```bash
gh pr checkout <pr_number> --repo <owner>/<repo>
```

Verify working tree clean (`git status --porcelain`). If dirty, stop and ask user.

## Step 3: Fetch all comments

Two sources — fetch both:

**Inline review comments** (threaded, on diff):
```bash
gh api "repos/<owner>/<repo>/pulls/<pr_number>/comments" --paginate
```

**Issue comments** (general PR discussion):
```bash
gh api "repos/<owner>/<repo>/issues/<pr_number>/comments" --paginate
```

Filter out:
- Comments authored by the PR author themselves (unless user says otherwise)
- Bot comments (`user.type == "Bot"`)
- Outdated review comments where `position` is null AND the comment is purely informational (use judgment)
- Replies to comments already addressed by PR author in-thread

For each remaining comment capture: `id`, `user.login`, `body`, `path` + `line` (inline only), `html_url`, source type (`review` vs `issue`).

## Step 4: Draft feedback.md

Write `feedback.md` at repo root. **Do not stage or commit this file.** Format:

```markdown
# PR #<pr_number> — <title>

<url>

## Comments

### 1. <short summary> — @<author>
- Source: review-inline | issue-general
- Location: `<path>:<line>` (if inline)
- Link: <html_url>
- Comment ID: <id>

**Original:**
> <quoted body>

**Proposed fix:**
<your plan, concrete: files to change, approach>

**Commit:** _pending_

---

### 2. ...
```

After writing, show user the list of comments + proposed fixes. Ask user to confirm or adjust the plan before implementing.

## Step 5: Implement each fix, one commit per comment

Sequentially, per comment:

1. Apply the code change.
2. Verify it compiles: `go install ./...` (skip if non-Go change).
3. Stage only the files for this fix: `git add <files>` — **never** `git add -A` (would stage `feedback.md`).
4. Commit with message: `fix: <short summary> (PR #<pr_number> review)`.
5. Capture the commit hash: `git rev-parse HEAD`.
6. Update the corresponding section in `feedback.md`, replacing `_pending_` with the hash.

If a fix can't be implemented (ambiguous, needs more info, disagree), mark commit as `_skipped — <reason>_` in feedback.md and ask user before moving on.

Never use `--no-verify`. If a pre-commit hook fails, fix the root cause and create a new commit.

## Step 6: Pause for push approval

Show the user:
- Summary of commits created (`git log --oneline <baseRefName>..HEAD`)
- Any skipped comments

Ask: "Push these commits to `origin/<headRefName>`?" Do not push until explicit yes.

## Step 7: Push

```bash
git push origin <headRefName>
```

## Step 8: Reply to every comment

For each implemented comment, build the commit URL: `https://github.com/<owner>/<repo>/commit/<full_hash>`.

Reply body (exact):
```
Implemented in this commit <full_hash>
```
With the hash linked to the commit URL — use Markdown: `Implemented in this commit [<short_hash>](<commit_url>)`.

**Inline review comments** — reply in-thread:
```bash
gh api -X POST "repos/<owner>/<repo>/pulls/<pr_number>/comments/<comment_id>/replies" \
  -f body="Implemented in this commit [<short_hash>](<commit_url>)"
```

**Issue comments** — no threading, post new PR comment referencing the original author:
```bash
gh pr comment <pr_number> --repo <owner>/<repo> \
  --body "@<original_author> re [your comment](<original_html_url>): Implemented in this commit [<short_hash>](<commit_url>)"
```

For skipped comments, do **not** auto-reply — leave to the user.

## Step 9: Cleanup

- Confirm `feedback.md` is **not** staged (`git status` should still show it untracked).
- Open the PR: `open <pr_url>`.
- Report: commits created, comments replied to, anything skipped.

## Rules

- One commit per comment. Never bundle.
- `feedback.md` stays untracked the entire run.
- Always ask before pushing (see [[feedback_ask_before_push]]).
- Don't `git add -A` / `git add .` — explicit file paths only.
- Use commit message convention from recent `git log` if it differs from the default above.
