Address every review/issue comment on a GitHub PR by drafting solutions, implementing them as individual commits, and replying to each comment with the implementing commit URL.

PR URL: `$ARGUMENTS`

## Step 1: Parse PR + fetch metadata

Extract `owner`, `repo`, `pr_number` from the URL. Then:

```bash
gh pr view <pr_number> --repo <owner>/<repo> --json number,title,headRefName,baseRefName,state,url,author,mergedAt
```

If PR closed/merged, ask the user before continuing.

## Step 2: Checkout PR branch

```bash
gh pr checkout <pr_number> --repo <owner>/<repo>
```

Verify working tree clean (`git status --porcelain`). If dirty, stop and ask.

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
- Comments by the PR author themselves (unless user says otherwise)
- Bot comments (`user.type == "Bot"`)
- Outdated review comments where `position` is null AND purely informational (use judgment)
- Replies the PR author already addressed in-thread

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
<concrete plan: files, approach>

**Commit:** _pending_

---

### 2. ...
```

After writing, show the list of comments + proposed fixes. Ask the user to confirm or adjust the plan before implementing.

## Step 5: Implement each fix, one commit per comment

Sequentially:

1. Apply the code change.
2. Verify it compiles (`go install ./...` for Go projects; skip for non-code).
3. Stage only the files for this fix: `git add <files>` — **never** `git add -A` (would stage `feedback.md`).
4. Commit with message: `fix: <short summary> (PR #<pr_number> review)`.
5. Capture the commit hash: `git rev-parse HEAD`.
6. Update the matching section in `feedback.md`, replacing `_pending_` with the hash.

If a fix can't be done (ambiguous, declined, needs info), mark commit as `_skipped — <reason>_` and ask the user before moving on.

Never use `--no-verify`. If a pre-commit hook fails, fix root cause and create a new commit.

Bundle multiple comments into one commit only when they describe the same coherent change (e.g. a rename split across two reply messages). Note the bundling in feedback.md.

## Step 6: Pause for push approval

Show:
- Commits created: `git log --oneline <baseRefName>..HEAD`
- Any skipped comments

Ask: "Push these commits to `origin/<headRefName>`?" Do not push until explicit yes.

## Step 7: Push

```bash
git push origin <headRefName>
```

## Step 8: Reply to every comment

For each addressed comment, build the commit URL: `https://github.com/<owner>/<repo>/commit/<full_hash>`.

Reply body: `Implemented in this commit [<short_hash>](<commit_url>)`.

**Inline review comments** — reply in-thread:
```bash
gh api -X POST "repos/<owner>/<repo>/pulls/<pr_number>/comments/<comment_id>/replies" \
  -f body="Implemented in this commit [<short_hash>](<commit_url>)"
```

**Issue comments** — no threading, post a new PR comment that references the author:
```bash
gh pr comment <pr_number> --repo <owner>/<repo> \
  --body "@<original_author> re [your comment](<original_html_url>): Implemented in this commit [<short_hash>](<commit_url>)"
```

For comments that were **declined** (not implemented), post a reply explaining the reasoning instead of the implementation link. Do **not** auto-reply on no-op comments ("Smart!", "LGTM", etc.).

## Step 9: Cleanup

- Confirm `feedback.md` is still untracked (`git status` should show it as `??`).
- Open the PR: `open <pr_url>`.
- Report: commits created, comments replied to, anything skipped.

## Rules

- One commit per comment unless multiple comments describe a single coherent change — then bundle and note it.
- `feedback.md` stays untracked the entire run.
- Always ask before pushing.
- Don't `git add -A` / `git add .` — explicit file paths only.
- Match commit message convention from recent `git log` if it differs from the default above.
