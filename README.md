# The Planner/Executor Workflow for Vibe Coders

> Stop letting your AI coding agent freestyle into production. This is a structured
> two-role workflow that keeps Claude Chat as your brain and Claude Code as your hands —
> so you ship fast without losing control.

---

## The Core Idea

Most people vibe code by chatting directly with Claude Code and just... hoping it doesn't
break anything. This workflow splits the job into two roles:

| Role | Tool | Job |
|------|------|-----|
| **Planner** | Claude Chat | Think, spec, review, gate |
| **Executor** | Claude Code | Audit, implement, test, report |

Claude Chat writes the prompts. Claude Code executes them. You approve every step.
Nothing gets committed until you say so.

---

## Why This Works

- **Audit before every feature** — CC reads the actual code before touching it. No assumptions.
- **Plan before every implementation** — you see the full commit breakdown before a single line is written.
- **Gate before every commit** — all tests must pass. No exceptions.
- **Gate before every push** — you verify the exact commits going out. No surprises.
- **You hold the push button** — CC never pushes autonomously. Ever.

---

## Step 1 — Audit

Before writing a single line, Claude Chat gives you this prompt to paste into CC:

```
Audit before we begin. Report only — no edits.

1. Read [relevant files] and report their export shape — are they object-literal exports
   (export const x = { async fn(){} }) or standalone export async functions?
2. Confirm [table/schema/column names] exist as assumed — don't guess.
3. Confirm route insertion point — show surrounding lines to verify no /:id shadowing.
4. grep for [variable/function names] the implementation will reference.

STOP and wait for approval before any edits.
```

CC reports what it finds. You review. If anything is wrong (wrong column name, wrong export
shape, wrong variable name), you fix the spec before implementation starts — not after.

---

## Step 2 — Plan

```
Based on the audit, propose a commit-by-commit implementation plan.
For each commit: which files change, what the change does, commit message.
Do NOT implement yet. STOP for approval.
```

You see the full plan. You approve it or adjust it. CC doesn't write a single line of code
until you give the go-ahead.

---

## Step 3 — Implement (one commit at a time)

For each commit, Claude Chat writes a focused implementation prompt ending with the gate:

```
Implement: [specific change for this commit only]

[Exact instructions — file names, function names, what to add/change]

Then run all gates and report results + full diff. Do NOT commit.

[YOUR GATE COMMANDS HERE — see Five-Green Gate below]
```

CC implements, runs the gates, reports the diff. You review. If everything is green and
the diff looks right, you issue the commit prompt:

```
Commit now:
git add [exact files only]
git commit -m "feat(scope): description (#issue)"
Report commit hash.
```

Path-scoped `git add` only. Never `git add .`.

---

## Step 4 — Push Gate

After all commits, before pushing:

```
Push gate:
git log origin/main..HEAD --oneline
```

You see exactly what's about to go out. Expected: only the commits from this session,
no strays. You push manually. Then post-push verify:

```
git log origin/main..HEAD --oneline
```

Expected: empty. Confirms origin is fully caught up.

---

## Step 5 — Doc Commit

Code commits and doc commits are always separate. At the end of every session, CC
updates your backlog and session log:

```
Doc commit — no code changes.
1. BACKLOG.md — mark item #NNN as shipped, add next pending phase
2. SCHEDULE.md — append today's session entry
git add BACKLOG.md SCHEDULE.md
git commit -m "docs: add #NNN to backlog and schedule"
```

You push the doc commit manually too.

---

## The Five-Green Gate

Every commit must pass all gates. Adapt these to your stack — the principle is what matters:
**type-check + unit tests + build, all exit 0, before every single commit.**

```bash
# Type check (server)
npx tsc --noEmit 2>&1; echo "=== SERVER TSC EXIT: $? ==="

# Type check (client)
cd client && npx tsc --noEmit 2>&1; echo "=== CLIENT TSC EXIT: $? ==="

# Unit tests (canary — your most critical regression tests)
npx jest --testPathPattern="<your critical tests>" --no-coverage 2>&1; echo "=== JEST EXIT: $? ==="

# Build
npm run build 2>&1; echo "=== BUILD EXIT: $? ==="

# Component/unit tests (client)
cd client && npx vitest run 2>&1; echo "=== VITEST EXIT: $? ==="
```

**If any gate is red:**
- CC stops. Does not commit.
- CC pastes full stderr verbatim — no summarizing.
- You read the raw error. You issue a fix prompt.
- All five gates re-run from scratch after the fix.

One red gate = no commit. No exceptions.

---

## Git Rules

- `git add` is always path-scoped to the exact intended files
- Code commits and doc commits are always separate
- Migrations / schema changes get flagged before commit — they touch prod data
- **You hold the push. CC never pushes autonomously.**

---

## Backlog Number Discipline

Before starting any feature, verify the next available issue number:

```
grep "^\- \*\*#" BACKLOG.md | tail -20
```

Highest existing number + 1 = your next number. Confirm before writing any prompt that
references it. Never guess.

---

## Prompt-Writing Rules (for Claude Chat)

When Claude Chat writes prompts for CC:

1. **Audit first** — every prompt starts read-only with a STOP
2. **Name exact files** — never "find the relevant file"
3. **State the expected export shape** — object-literal vs standalone function; CC will match the existing pattern
4. **Gate commands are mandatory** — include them at the end of every implementation prompt
5. **End with "Do NOT commit"** — CC reports, you gate, you issue the commit prompt separately
6. **Guard async state updates** — when a fetch can resolve after user input changes:
```ts
setState(s => {
  if (userHasEdited) return s;  // discard stale fetch result
  return { ...s, field: result };
});
```

---

## Session Recap (Start of Every Session)

Before writing any prompt, Claude Chat recaps the current state:

- What shipped last session (feature, version, commits)
- Any open high-priority items or pending phases
- Today's stated goal — restated back to you for confirmation

This takes 30 seconds. It prevents starting from stale assumptions and catches scope drift before it starts.

---

## Scope Creep Guard

Any change to the agreed session scope must be flagged and confirmed before proceeding.

**Triggers:**
- "While we're here, let's also..."
- CC discovers something related mid-implementation
- A bug surfaces that wasn't in the original scope
- A phase boundary shifts

**How to handle:**
> ⚠️ Scope change — original goal was [X]. You're now suggesting [Y].
> Recommend: defer to backlog unless [Y] directly unblocks [X] and is small.
> Confirm to expand, or I'll log it for next session.

Never silently absorb scope creep into the current session's prompts.

---

## Test Coverage Check

After every new feature commit, Claude Chat asks:

> Does this change need a new test?
> - Is there new business logic, a new query, or a new calculation not yet covered?
> - Does the existing canary catch regressions here?

If yes → propose a test commit before the doc commit.
If no → state why (e.g. "pure UI, no new logic") and move on.

Ask every session, every feature commit. No exceptions.

---

## Rollback Plan (Migration Sessions)

Whenever a session touches schema or migrations, Claude Chat asks before the push gate:

> ⚠️ Migration detected — this deploy will run schema changes against prod immediately.
> 1. Is there a fresh prod backup?
> 2. What is the undo SQL if the deploy fails?

Do not proceed to push until both are confirmed.

---

## CC Report Format (Standard)

Every CC report after implementation must follow this structure:

```
=== GATE RESULTS ===
Server TSC:    EXIT 0 ✅ / EXIT 1 ❌
Client TSC:    EXIT 0 ✅ / EXIT 1 ❌
Test canary:   EXIT 0 ✅ / EXIT 1 ❌
Client build:  EXIT 0 ✅ / EXIT 1 ❌
Component tests: EXIT 0 ✅ / EXIT 1 ❌

=== DIFF ===
[full path-by-path diff]

=== DEVIATIONS FROM SPEC ===
[anything CC changed vs the prompt, with reason — or "None"]

=== STANDING BY ===
```

If CC skips any section, ask for it before reviewing. You need the full picture to gate correctly.

---

## Assumption Surfacing

Before writing any implementation prompt, Claude Chat lists the assumptions baked into it — and flags which are verified vs unverified.

```
Assumptions for this prompt:
✅ [verified] — confirmed from audit output
⚠️ [unverified] — assumed from memory or inference
```

Any unverified assumption that is material to the implementation gets added to the audit step first. Never build on an unverified assumption.

---

## Complexity Warning

Before agreeing to implement a feature in one session, Claude Chat estimates scope and flags it if it's large.

**Triggers:**
- More than 3 files changed
- Any schema / migration change
- Touches both server AND client non-trivially
- More than 3 estimated commits

**Format:**
> ⚠️ Complexity check — estimated [N] files, [N] commits, migration: yes/no.
> Recommend: [do in one session / split into P1+P2]. Confirm scope before I write any prompt.

Never start writing prompts for a large session without explicit scope confirmation.

---

## Reversibility Check

Before any schema change, data migration, or architectural decision, Claude Chat asks:

> Is this reversible? If the deploy fails, what's the undo?
> If there's no clean undo — can we make it reversible? (additive migration, soft-delete, feature flag)

Do not proceed to push gate for any irreversible change without a confirmed undo plan.

---

## Naming Consistency Check

Before finalising any new function, endpoint, or variable name, CC greps for existing naming patterns — Claude Chat never relies on memory alone.

```
grep -n "async get" src/repositories/yourRepo.ts | head -20
grep -rn "router.get" src/routes/index.ts
```

Claude Chat checks: does the new name follow the existing convention? Is it in the right layer? If not, flag and suggest a conforming alternative before writing the implementation prompt.

---

## Post-Implementation Sanity Check

After all gates are green, before committing, Claude Chat writes one more prompt:

```
Gates are green. Before committing, run a logic sanity check:
[Call the endpoint / run the query / render the component with specific inputs]
Report the actual output. Do NOT commit until confirmed correct.
```

The sanity check directly tests the core logic path — not just "does it compile." If the output is wrong, issue a fix prompt before committing, even if gates were green.

---

## SQL Injection Prevention (Critical)

Every server-side query must use parameterised queries. No exceptions.

```ts
// ❌ NEVER — string interpolation
const sql = `SELECT * FROM users WHERE name = '${input}'`;

// ✅ ALWAYS — parameterised
const sql = `SELECT * FROM users WHERE name = $1`;
await query(sql, [input]);

// ❌ NEVER — IN-list concatenation
WHERE id IN (${ids.join(',')})

// ✅ ALWAYS — ANY with typed array
WHERE id = ANY($1::int[])
// params: [ids]

// ❌ NEVER — dynamic value in optional filter
if (filter) sql += ` AND col = '${filter}'`;

// ✅ ALWAYS — push to params array
if (filter) { sql += ` AND col = $${params.length + 1}`; params.push(filter); }
```

**Claude Chat reviews every diff for SQL strings.** Any `${...}` inside a SQL template literal is a blocker — flag before issuing the commit prompt.

CC audit step for any query-related change:
```
grep -n "\${" src/repositories/*.ts
```
Any hit must be reviewed before committing.

---

## Performance Budget

Any new query must be assessed for complexity before the commit prompt is issued.

**Claude Chat checks in every diff review:**
- Correlated subqueries? (each row triggers a separate query — avoid on hot paths)
- N+1 pattern? (querying inside a loop)
- More than 3 JOINs on a frequently-called endpoint?
- Missing index on a filtered or joined column?

**Format:**
> ⚠️ Performance check — this query has [N] correlated subqueries / N+1 / [N] JOINs.
> Recommend: [rewrite / add index / batch fetch]
> Accept this tradeoff or rewrite before committing?

Never silently ship a known slow query.

---

## Error Handling Consistency

Every new server-side function must follow the existing error handling pattern. Claude Chat checks the diff before issuing any commit prompt.

**Common patterns to enforce:**
- Controllers: `try/catch` with error passed to `next(e)` — never swallow silently
- Repo methods: throw on unexpected DB errors; return `null` only when "not found" is a valid expected state
- No silent `catch {}` blocks

If any new function deviates, flag as a blocker before committing.

---

## Secret / Credential Guard (Mandatory Diff Check)

Every diff review must include a credential scan. No exceptions.

**CC runs before every commit:**
```bash
grep -rn "password\|secret\|api_key\|DATABASE_URL\|token\|bearer" \
  $(git diff --name-only HEAD) | grep -v ".env.example"
```

Also check:
- Hardcoded URLs or environment-specific values in committed files
- `console.log` statements that print sensitive data
- Any `.dump` / `.sql` / `.env` file staged

Any hit = full stop. Do not commit until resolved.

---

## Accessibility / Mobile Consideration

Any new UI component or layout change must be assessed before committing.

**Claude Chat asks after every client diff:**
- Is this mobile-friendly? Responsive breakpoints?
- Touch target size ≥ 44px?
- Any `onClick` on a non-interactive element without a keyboard equivalent?
- If this appears on a mobile view — has the mobile layout been considered?

If desktop-only by design, state why. If it should be mobile-accessible but isn't, flag before committing.

---

## Dead Code / Cleanup Rule

At the end of every session, before the doc commit, CC runs:

```bash
grep -rn "console\.log\|TODO\|FIXME\|debugger" \
  $(git diff origin/main..HEAD --name-only)
```

Also check: unused imports, commented-out code, temporary scaffold added during implementation.

All findings cleaned up in the same session. Nothing carries over.

---

## Dependency Introduction Rule

Any new `npm install` or first-time import from an unused library must be justified before Claude Chat writes the implementation prompt.

**Four questions before approving:**
1. Is it necessary? Does a native or existing alternative work?
2. Is it actively maintained? (check last published date)
3. Any known vulnerabilities? (`npm audit` after install)
4. Bundle size impact for client deps — is it tree-shakeable?

If a native alternative is sufficient, recommend it instead. Never add a dependency without answering all four.

---

## TypeScript `any` Guard

Every diff review checks for newly introduced `any` types.

**CC runs before every commit:**
```bash
git diff origin/main..HEAD -- '*.ts' '*.tsx' | grep "^+" | grep ": any\|as any\|<any>"
```

Any hit is a blocker unless:
1. It genuinely comes from an untyped third-party source
2. It is explicitly commented explaining why

Otherwise: replace with `unknown` + type guard, or a specific interface. Claude Chat blocks the commit and asks CC to type it properly.

---

## Session Start — Paste Last Session Summary

At the very start of every session, Claude Chat asks:

> Please paste the last session summary so I can orient before we begin.

Claude Chat reads it, confirms last shipped state, notes any carried-over items, flags any conflicts with current memory, then proceeds with the session recap.

If no summary is available, Claude Chat proceeds with memory only and states its assumptions explicitly.

---

## Session End — Summary

At the end of every session, Claude Chat produces a structured summary for the developer to save and paste at the next session start.

**Format:**
```
Session Summary
Date: YYYY-MM-DD

Commits:
| Hash    | Version  | Description |
|---------|----------|-------------|
| xxxxxxx | v1.0.NNN | feat(...): ... |

Goal: [what was built]

Key changes:
- [server changes]
- [client changes]
- [doc/config changes]

Decisions made:
- [architectural or product decisions locked this session]

Deferred / next session:
- [anything explicitly deferred]

Gate: all five green ✓
```

---

## Session End Checklist

Before generating the session summary, Claude Chat runs through this checklist:

```
□ Working tree clean?
□ All code commits pushed?
□ Doc commit pushed?
□ Deploy live?
□ Post-deploy doc commit pushed?
□ Any open TODO / FIXME from this session?
□ Test coverage gaps noted?
□ Next session priorities in backlog?
```

Each item reported as ✅ or ⚠️. Any ⚠️ must be resolved or logged in the backlog before the session is closed.

---

## Regression Test Policy

- Never delete a test to make a gate green — fix the code or update the test with justification
- Never skip a test without a backlog entry explaining why
- Test updates ship in the same commit as the feature — never separately after the fact
- If a feature change requires updating an existing test, flag it in the plan step

If CC reports a gate failure by deleting or skipping a test, Claude Chat blocks the commit and issues a fix prompt targeting the root cause.

---

## Breaking Change Detection

Any change to an existing API contract or DB schema is a breaking change — flag before writing the implementation prompt.

**API:** endpoint rename/removal, required param added/removed, response shape changed, HTTP method changed.
**DB:** column rename/removal, type change, constraint added/removed, table rename/removal.

**Format:**
> ⚠️ Breaking change — this affects [existing consumers / existing data].
> What depends on the old [endpoint/column/shape]?
> Migration strategy or backward-compatible intermediate step needed?

Do not proceed to implementation until impact is assessed and strategy confirmed.

---

## Environment Parity Rule

Any feature depending on an env var, external service, or environment-specific data must document its fallback before implementation.

**Claude Chat asks:**
- Is the env var set locally AND in prod? What happens if it's missing?
- If an external API fails — does the app degrade gracefully or break?
- Is there local seed data to test prod-dependent behaviour?

A feature that silently breaks when an env var is missing is a bug. Define the fallback before writing code.

---

## Commit Atomicity Rule

Every commit must be self-contained — reverting it must not break any other commit.

- Server endpoint and client consumer: **separate commits, server first**
- Migration and app code that depends on it: **separate commits, migration first**
- Tests and feature code: **same commit, never separated**
- Never leave the app in a broken intermediate state between commits

Claude Chat enforces this sequencing in the plan step. If CC proposes a plan that violates atomicity, correct it before approving.

---

## Context Window Management

For long sessions (>10 commits), Claude Chat proactively recaps known state to prevent stale context from corrupting later prompts.

**Trigger:** every 10 commits, or when referencing audit output from early in the session.

**Format:**
> 📋 Context recap — [N] commits in
> Last confirmed: v[NNN] · [hash] · [description]
> Files modified this session: [list]
> Open unverified assumptions: [list or "none"]
> Consistent with your understanding?

Resolve any conflict before writing the next prompt.

---

## Hotfix Protocol

For urgent prod issues that cannot wait for a full planned session:

**Lightweight flow:**
1. **Triage** — CC reads the error, identifies root cause. Report only, no edits.
2. **Minimal fix** — smallest possible change. No opportunistic cleanup.
3. **Five-green gate** — mandatory even for hotfixes. No exceptions.
4. **Push immediately** — no pre-push doc commit required.
5. **Deploy** — confirm live.
6. **Doc commit after** — backlog + schedule updated after deploy, not before.

> 🚨 Hotfix mode — urgent fix, lightweight flow.
> Root cause: [X] · Fix: [minimal change]
> Skipping: doc-first, scope discussion
> Five-green gate: still mandatory.

The gate is never skipped. Only the pre-push doc commit is deferred.

---

## The Mental Model (final)

```
You (ideas + last session summary)
  → Claude Chat
      (session recap · planner · assumption checker · complexity guard
       · scope guard · second opinion · decision support · breaking change
       · env parity · atomicity · SQL safety · credential scan · any guard
       · perf budget · error handling · dead code · deps · context recap)
    → Claude Code
        (executor · self-stop on conflict · standard report · sanity check
         · never autonomous)
      → Gate (all five green + logic verified + security checked)
        → You (push · deploy · doc update · session summary)
```

Vibe coding is fast. This workflow keeps it from becoming a liability.

