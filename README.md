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

**What you see in your editor (REVIEW.md):**

```
## src/ratelimit/limiter.ts

**What changed:** Replaced fixed-window counter with sliding window
**Why:** Fixed window allows burst at window boundary ok fine but this only matters at scale we dont have

  removed: const key = userId + ":" + Math.floor(Date.now() / windowMs)
  removed: const count = await redis.incr(key)
  removed: await redis.expire(key, windowMs / 1000)
  added:   const now = Date.now()
  added:   const windowStart = now - windowMs
  added:   await redis.zremrangebyscore(key, 0, windowStart)
  added:   const count = await redis.zcard(key)
  added:   await redis.zadd(key, now, now + "-" + Math.random()) Math.random() in a key?? use crypto.randomUUID(). collisions under load
  added:   // TTL not set
this will leak redis memory forever. add expire back at 2x window.

## src/ratelimit/middleware.ts

**What changed:** Wired new limiter into express middleware
**Why:** Apply rate limit before route handlers

  removed: const allowed = await checkLimit(req.user.id) good riddance
  added:   const allowed = await checkLimit(req.user?.id ?? req.ip)

whole thing needs try/catch. if redis is down this 500s every request. fail open + log.
```

**What the agent sees (`diff REVIEW.md.backup REVIEW.md`):**

```diff
 ## src/ratelimit/limiter.ts
 
 **What changed:** Replaced fixed-window counter with sliding window
-**Why:** Fixed window allows burst at window boundary
+**Why:** Fixed window allows burst at window boundary ok fine but this only matters at scale we dont have
 
   removed: const key = userId + ":" + Math.floor(Date.now() / windowMs)
   removed: const count = await redis.incr(key)
   removed: await redis.expire(key, windowMs / 1000)
   added:   const now = Date.now()
   added:   const windowStart = now - windowMs
   added:   await redis.zremrangebyscore(key, 0, windowStart)
   added:   const count = await redis.zcard(key)
-  added:   await redis.zadd(key, now, now + "-" + Math.random())
+  added:   await redis.zadd(key, now, now + "-" + Math.random()) Math.random() in a key?? use crypto.randomUUID(). collisions under load
   added:   // TTL not set
+this will leak redis memory forever. add expire back at 2x window.
 
 ## src/ratelimit/middleware.ts
 
 **What changed:** Wired new limiter into express middleware
 **Why:** Apply rate limit before route handlers
 
-  removed: const allowed = await checkLimit(req.user.id)
+  removed: const allowed = await checkLimit(req.user.id) good riddance
   added:   const allowed = await checkLimit(req.user?.id ?? req.ip)
+
+whole thing needs try/catch. if redis is down this 500s every request. fail open + log.
```

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
