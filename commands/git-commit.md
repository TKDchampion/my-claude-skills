Analyze the current git changes and propose a commit message for confirmation.

## Step 1: Gather Context

Run in parallel:
- `git status --short`
- `git diff --cached` (staged)
- `git diff` (unstaged)
- `git log --oneline -5` (style reference)

If nothing to commit → report "Nothing to commit" and stop.

## Step 2: Stage Files (if needed)

If there are unstaged changes but nothing staged, ask:

```
No staged files found. Stage all changes? (y/n)
Or specify files to stage:
```

Wait for response before proceeding. If yes → `git add -A`. Warn if diff includes `*.env`, `*secret*`, `*credential*`, `*key*` files.

## Step 3: Generate Conventional Commit Message

Analyze the diff and generate a message following this format:

```
<type>(<scope>): <subject>

[optional body]
```

**Types:** `feat` / `fix` / `refactor` / `chore` / `docs` / `style` / `test` / `perf` / `ci` / `revert`

**Rules:**
- Subject: imperative mood, lowercase, no period, ≤72 chars (`add login` not `Added login.`)
- Scope: optional module/file name
- Body: explain *why*, not *what* (the diff shows what), wrap at 72 chars
- Breaking change: `feat!:` + `BREAKING CHANGE:` footer

## Step 4: Present for Confirmation

Show the proposed commit message clearly:

```
Proposed commit:

  <type>(<scope>): <subject>

  <body if needed>

Commit with this message? (y/n/e)
  y = commit now
  n = cancel
  e = edit message
```

**Wait for user response. Do NOT commit until confirmed.**

## Step 5: Execute

- `y` → run `git commit` with the message using HEREDOC format → report commit hash
- `n` → "Cancelled. No changes committed."
- `e` → show message for editing, then re-confirm

**Never skip confirmation. Never use `--no-verify`.**
