# change-review

An agent skill for human-in-the-loop diff review. The agent writes a readable summary of every change it made and why, you annotate freely in your editor, and the agent addresses your feedback before committing.

## How it works

1. Agent generates `REVIEW.md` — the git diff, file by file, with plain-English explanations
2. Agent backs it up as `REVIEW.md.backup`
3. Agent opens it in your `$VISUAL`/`$EDITOR` (or tells you to open it yourself)
4. You add comments, questions, or corrections anywhere in the file
5. You say **"Review complete"**
6. Agent diffs the two files to find exactly what you wrote, then acts on it
7. Agent cleans up both files when done

## Install

Copy the `change-review/` folder into your agent's skills directory:

**Claude Code:**
```bash
cp -r change-review ~/.claude/skills/
```

**Cross-runtime (`~/.agents/skills/`):**
```bash
cp -r change-review ~/.agents/skills/
```

## Usage

After an agent finishes implementing something, trigger the skill:

```
/change-review
```

or just ask naturally:

```
Do a change review before we commit.
```

## Example workflow

```
You:    Refactor the auth module to use JWT instead of sessions.

Agent:  [makes changes]
        Which editor should I open REVIEW.md in?

You:    VS Code

Agent:  [runs: code REVIEW.md]

# You open REVIEW.md and see:
#
#   ## `src/auth/middleware.js`
#   **What changed:** Replaced session lookup with JWT verify
#   **Why:** Migrating to stateless auth per the refactor request
#
#   ```diff
#   - const session = await getSession(req.cookies.sid)
#   + const payload = jwt.verify(req.headers.authorization, SECRET)
#   ```
#
# You add a comment inline:
#   > SECRET should come from env, not be hardcoded — where is it set?

You:    Review complete.

Agent:  Found your note on SECRET. Moving it to process.env.JWT_SECRET
        and adding a startup check. [makes change]
        Another round or ready to commit?

You:    Commit it.

Agent:  [commits] [cleans up REVIEW.md and REVIEW.md.backup]
```

## Compatibility

Follows the [Agent Skills specification](https://agentskills.io/specification). Works with Claude Code, Codex, Copilot CLI, Gemini CLI, and any runtime that supports the `SKILL.md` format.
