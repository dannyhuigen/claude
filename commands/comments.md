Clean up comments in the files changed against `main`.

## Scope

- Operate only on files in `git diff main...HEAD` (the current branch's changes).
- Only edit comment text. Do not change code, logic, blank lines, or alignment.
- Never touch Go directives — they are not comments: `//go:build`, `//go:generate`, `//nolint`, `//goland:`, etc. (no space after `//`).

## Rules

### Doc comments (directly above an exported func/type/const/var)
- Keep to a single sentence: capitalized, ending in a period.
- Must start with the identifier name (godoc convention): `// FetchUser returns…`, not `// This function fetches…`.
- State only *what* it does. Drop *why* it exists, what motivated it, or what bug/ticket it addresses.
- Do not delete doc comments on exported identifiers — godoc needs them; shorten them instead.

### Unexported funcs/helpers
- Do not add comments to self-explanatory ones.
- If one has a comment, apply the same "what, not why, one line" rule, or remove it if redundant.

### File-header comments (top of file)
- Allowed, but only to describe *what the file is for*.
- Remove anything explaining the use case or problem it was created to solve — especially context that won't be meaningful once the branch is merged (e.g. "added to fix X" where X no longer exists).

### All other comments (inline, block, in-body)
- Remove unless genuinely necessary to understand non-obvious code (a workaround, a tricky invariant, why an unusual approach was taken). When in doubt, remove.
- Delete commented-out code entirely — do not keep it "for reference".

### Test files (`_test.go`)
- Comments explaining *why* a test case exists are the legitimate "why" — keep those.

### TODO / FIXME
- Do not silently keep or delete. Flag each one to me; this repo bans TODO comments for unpicked work (create a ClickUp task instead).

## After editing

- Run `go install ./...` to confirm nothing broke.
- Show a per-file summary of what was removed or shortened before considering it done.
