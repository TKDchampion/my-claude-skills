Review current git changes with lightweight tests, security checks, and optimization suggestions before finalizing.

## Step 1: Gather Diff Context

Run in parallel:
- `git diff --cached` (staged changes)
- `git diff` (unstaged changes)
- `git status --short`

If no changes found → report "Nothing to review." and stop.

Identify affected files and their languages/frameworks.

## Step 2: Lightweight Unit Test Check

Detect test runner from project files (`pytest.ini`, `pyproject.toml`, `package.json`, etc.).

Run the minimal test suite scoped to changed files only:
- **Python**: `pytest --tb=short -q` (if pytest available)
- **Node/TS**: `npm test -- --passWithNoTests` or `vitest run`
- **Go**: `go test ./...`
- If no test runner found → skip and note "No test runner detected."

Report:
```
Tests: ✓ X passed / ✗ X failed / ⚠ skipped
```

If tests fail → show failure output and continue (do not stop review).

## Step 3: Basic Security Check

Quickly scan the diff for obvious risks only:

- Hardcoded secrets / API keys / passwords in code
- Raw user input passed directly into queries or shell commands
- New endpoints missing authentication

Report findings as plain text. If none found → "✓ No obvious issues."

## Step 4: Code Quality Review

Evaluate changed code against best practices:

- **Complexity**: functions > 30 lines or cyclomatic complexity concerns
- **Duplication**: repeated logic that could be extracted
- **Naming**: unclear variable/function names
- **Error handling**: missing try/catch or exception handling at boundaries
- **Type safety**: missing type hints / type mismatches
- **Dead code**: unused imports, variables, commented-out blocks
- **Performance**: N+1 queries, unnecessary loops, missing indexes implied by query patterns

## Step 5: Present Full Report

Display a structured report:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  CODE REVIEW REPORT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📁 Files Changed: <list>

🧪 Tests
  <result or skipped reason>

🔒 Security
  <finding or ✓ No obvious issues>

📋 Code Quality
  • <issue or suggestion>
  ...
  (or ✓ Looks good)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  OVERALL ASSESSMENT: <PASS / PASS WITH SUGGESTIONS / NEEDS WORK>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Top Recommendations:
  1. <most important action>
  2. <second action>
  3. <third action>
```

## Step 6: Ask for Next Action

After presenting the report, ask:

```
What would you like to do?

  p = proceed (approve changes as-is)
  f = fix issues (I will apply the suggested fixes)
  e = edit manually (you will fix, then re-run /git-review)
  c = cancel (do nothing)
```

**Wait for response before taking any action.**

- `p` → "Review complete. Changes approved. You can now run /git-commit."
- `f` → Apply all Critical/High fixes automatically, then re-run the report
- `e` → "OK. Fix manually and run /git-review again when ready."
- `c` → "Cancelled. No changes made."

**Never auto-fix without confirmation. Never commit from this command.**
