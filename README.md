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

### Simple

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

### Messy (real-world)

The agent extracts feedback from *whatever you wrote* — inline comments, live code edits, half-sentences, rage-typing. It diffs against the backup and reads everything you touched.

$\textcolor{red}{\text{red = agent removed}}$ · $\textcolor{green}{\text{green = agent added}}$ · $\textcolor{blue}{\text{blue = your edits}}$

---

**`REVIEW.md` after 5 minutes in your editor:**

$\textcolor{blue}{\text{wait why did we need this again? the fixed bucket was shipping fine}}$

**`src/ratelimit/limiter.ts`**<br>
**What changed:** Replaced fixed-window counter with sliding window<br>
$\textcolor{blue}{\text{**Why:** Fixed window allows burst at window boundary (double the limit in 2x time) ok fine valid but only matters at scale we dont have yet}}$

$\textcolor{red}{\text{removed: const key = userId + ":" + Math.floor(Date.now() / windowMs)}}$<br>
$\textcolor{red}{\text{removed: const count = await redis.incr(key)}}$<br>
$\textcolor{red}{\text{removed: await redis.expire(key, windowMs / 1000)}}$<br>
$\textcolor{green}{\text{added:   const now = Date.now()}}$<br>
$\textcolor{green}{\text{added:   const windowStart = now - windowMs}}$<br>
$\textcolor{green}{\text{added:   await redis.zremrangebyscore(key, 0, windowStart)}}$<br>
$\textcolor{green}{\text{added:   const count = await redis.zcard(key)}}$<br>
$\textcolor{blue}{\text{added:   await redis.zadd(key, now, now + "-" + Math.random())  Math.random() IN A KEY?? use crypto.randomUUID(). collisions under load}}$<br>
$\textcolor{blue}{\text{added:   // TTL not set  THIS WILL LEAK MEMORY. add redis.expire(key, Math.ceil(windowMs / 1000) * 2) back}}$

---

**`src/ratelimit/middleware.ts`**<br>
**What changed:** Wired new limiter into express middleware<br>
**Why:** Apply rate limit before route handlers

$\textcolor{red}{\text{removed: import checkLimit from "./fixed"}}$<br>
$\textcolor{green}{\text{added:   import checkLimit from "./sliding"}}$<br>
$\textcolor{blue}{\text{removed: const allowed = await checkLimit(req.user.id)  good riddance}}$<br>
$\textcolor{green}{\text{added:   const allowed = await checkLimit(req.user?.id ?? req.ip)}}$<br>
$\textcolor{blue}{\text{whole thing needs try/catch. if redis is down this 500s every request. fail open + log.}}$

---

**`src/ratelimit/config.ts`**<br>
**What changed:** Added windowMs and maxRequests constants<br>
**Why:** Centralise config rather than magic numbers

$\textcolor{blue}{\text{added:   export const windowMs = 60000  env var. RATE LIMIT WINDOW MS. default this value}}$<br>
$\textcolor{blue}{\text{added:   export const maxRequests = 100  same. RATE LIMIT MAX REQUESTS}}$

---

---

**Agent response after "Review complete":**

```
Agent:  Got it. Four things to fix:
        1. Replace Math.random() in zadd key with crypto.randomUUID()
        2. Add redis.expire() back on the sorted set at 2× window duration
        3. Wrap middleware in try/catch, fail open on Redis errors
        4. Move WINDOW_MS and MAX_REQUESTS to env vars with current values as defaults
        Making those now... [makes changes]
        Done. Another round or ready to commit?
```

## Compatibility

Follows the [Agent Skills specification](https://agentskills.io/specification). Works with Claude Code, Codex, Copilot CLI, Gemini CLI, and any runtime that supports the `SKILL.md` format.
