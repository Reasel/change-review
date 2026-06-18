---
name: change-review
description: Use when implementation is complete and the user wants to review changes made by the agent, annotate the diff, and have the agent act on their feedback before committing.
---

# Change Review

Hand the user a readable diff with rationale, let them annotate freely, then address their feedback.

## Steps

### 1. Generate REVIEW.md

Run `git diff HEAD` (or `git diff --cached` if changes are staged). Write `REVIEW.md` in the repo root:

```markdown
# Diff Review

> Agent: <one-line summary of what was done and why>

---

## `path/to/file.ext`

**What changed:** <plain-English summary>
**Why:** <rationale — requirement, bug, constraint>

\```diff
<diff hunk(s) for this file>
\```

---

<!-- repeat per file -->
```

Keep rationale factual — reference the actual requirement or bug, not generic justification.

### 2. Backup

```bash
cp REVIEW.md REVIEW.md.backup
```

### 3. Open in editor

```bash
${VISUAL:-${EDITOR:-}} REVIEW.md
```

If `$VISUAL`/`$EDITOR` is unset or GUI (returns immediately), tell the user:

> REVIEW.md is ready. Open it in your editor, annotate freely anywhere in the file, then say **"Review complete"** when done.

If terminal editor (vim/nano/etc), the shell blocks until closed — just wait, then skip to step 4 automatically.

### 4. Wait for "Review complete"

Do nothing until the user says it. Do not re-read the file early.

### 5. Extract feedback

```bash
diff REVIEW.md.backup REVIEW.md
```

Lines added by the user (prefixed `>` in diff output) are their feedback. Read every added line in context of the surrounding diff hunk.

### 6. Address feedback

For each piece of user feedback:
- Quote the comment
- State what you will change
- Make the change
- Do NOT re-open REVIEW.md automatically

After all changes: ask user if another round is needed or if they want to commit.

### 7. Cleanup

Once the user confirms no further rounds are needed:

```bash
rm -f REVIEW.md REVIEW.md.backup
```

Do not delete earlier — user may want another round or to re-diff manually.

## Notes

- Never commit REVIEW.md or REVIEW.md.backup — add both to `.gitignore` if not already there
- If no git diff exists (nothing changed), say so and stop
- `git diff HEAD` covers both staged and unstaged; use `git diff HEAD~1` if changes are already committed
