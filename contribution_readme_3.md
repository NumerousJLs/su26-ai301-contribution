# Contribution 3: Built-in customize-opencode skill uses wrong MCP env key

**Contribution Number:** 3  
**Student:** Joshua Liu  
**Issue:** [anomalyco/opencode#35860](https://github.com/anomalyco/opencode/issues/35860)  
**Related issues:** [#30892](https://github.com/anomalyco/opencode/issues/30892), [#26332](https://github.com/anomalyco/opencode/issues/26332) (same symptom from the user side, filed before the root cause in the skill was known)  
**Fork:** [github.com/NumerousJLs/opencode](https://github.com/NumerousJLs/opencode)  
**Branch:** `fix/skill-mcp-env-key` (off `dev`)  
**Pull Request:** [anomalyco/opencode#35867](https://github.com/anomalyco/opencode/pull/35867)  
**Status:** Phase IV Complete — PR submitted, awaiting review

---

## Why I Chose This Issue

opencode has a built-in skill called `customize-opencode` that users invoke with `/customize-opencode` in the TUI. It serves as the canonical reference for configuring opencode — users read it and copy its examples directly into their `opencode.json`. If an example in this skill is wrong, users have no obvious way to find out, because the config loader strips unknown keys silently instead of throwing an error.

This issue was filed the same day I picked it up. The reporter spent 30 minutes debugging why their MCP environment variables were never reaching their subprocess — they had no error, no warning, just silent misbehavior. Two earlier issues (#30892 and #26332) reported the same symptom from the user side without identifying the documentation as the source. The root cause is that the skill shows `"env"` as the key for MCP subprocess environment variables, but the actual config schema and the runtime both use `"environment"`. The fix is two lines in one file, but the impact on users following the docs is real.

What "fixed" looks like in concrete terms: a user who copies the MCP local server example from the skill into their `opencode.json` should see their configured env vars present in `process.env` inside the subprocess. Before the fix, those vars were absent with no error.

---

## Understanding the Issue

### Problem Description

opencode lets users configure local MCP (Model Context Protocol) servers in their `opencode.json`. A local MCP server is an external process that opencode spawns and communicates with. You can pass environment variables to that subprocess by setting them in the config.

The built-in `customize-opencode` skill (the file users see when they run `/customize-opencode`) showed this example:

```json
"mcp": {
  "playwright": {
    "type": "local",
    "command": ["npx", "-y", "@playwright/mcp"],
    "enabled": true,
    "env": { "BROWSER": "chromium" }
  }
}
```

The field shown is `"env"`. But the config schema defines it as `"environment"`:

```ts
// packages/core/src/config/mcp.ts:12
environment: Schema.Record(Schema.String, Schema.String).pipe(Schema.optional),
```

And the runtime in `packages/opencode/src/mcp/index.ts:342` reads it as:

```ts
env: {
  ...process.env,
  ...mcp.environment,   // reads .environment, not .env
}
```

Because the config loader uses Effect schema parsing and strips unknown keys, `"env"` is silently discarded. The subprocess starts with no user-defined env vars, and there is no error or warning to signal that anything was dropped.

### Expected Behavior

Users who copy the MCP example from the skill should get working environment variable injection. The example should use the field name the runtime reads: `"environment"`.

### Current Behavior

The skill showed `"env"` in two places. Users who copy either example have their env vars silently dropped. Their MCP server runs without the configured variables, which can cause degraded behavior (wrong browser, missing credentials, missing feature flags) with no indication of why.

### Affected Components

`packages/core/src/plugin/skill/customize-opencode.md` — the built-in skill file. Two places:
- Line 115 (in the top-level config reference block): `"env": {}`
- Line 374 (in the dedicated MCP section): `"env": { "BROWSER": "chromium" }`

---

## Reproduction Process

### Environment Setup

No special setup beyond the standard `bun install` from the repo root. The bug is a wrong key name in a markdown file — no build step or server is needed to confirm the root cause by reading the schema and runtime.

For full verification (checking vars actually reach the subprocess), you'd need a local MCP server process that logs `process.env`. The key confirmation is code-level: comparing the skill example against `packages/core/src/config/mcp.ts` and `packages/opencode/src/mcp/index.ts`.

### Steps to Reproduce

1. Open opencode and run `/customize-opencode` in the TUI.
2. Read the MCP section. It shows `"env"` as the field for subprocess env vars.
3. Copy the example into your `opencode.json`, adding real env vars under `"env"`.
4. Restart opencode. Your MCP server starts but the env vars are absent.
5. Add a `console.log(process.env)` in the MCP server to confirm the vars were never set.

There is no error. The config loads successfully because the Effect schema parser strips unknown keys without warning.

### Reproduction Evidence

- **Schema confirmed**: `packages/core/src/config/mcp.ts:12` — field name is `environment`.
- **Runtime confirmed**: `packages/opencode/src/mcp/index.ts:342` — runtime reads `mcp.environment`, not `mcp.env`.
- **Skill confirmed on dev**: `git show upstream/dev:packages/core/src/plugin/skill/customize-opencode.md | grep '"env"'` returned both incorrect lines (115 and 374) before the fix.
- **No error path**: Effect `Schema.parseJson` strips unknown keys by design — there is no runtime warning when `"env"` is discarded.
- **History**: `git log --follow -p -- packages/core/src/plugin/skill/customize-opencode.md` shows the MCP example section was added when the skill was first introduced; the `"env"` key was wrong from the beginning — it was never `"environment"` in the skill file. The schema has always used `"environment"`.

---

## Solution Approach

### Analysis

The mismatch is one-directional: the runtime and schema are correct, the skill example is wrong. There is no ambiguity — `"env"` was a wrong key name from the start, not a renamed field. The fix is to update the two example blocks in the skill file to match the schema.

The same file was touched by two existing open PRs:
- [PR #33655](https://github.com/anomalyco/opencode/pull/33655) ("docs(skill): update customize-opencode permissions") — updates permission guidance only; diff-inspected, does not change the `"env"` key.
- [PR #35233](https://github.com/anomalyco/opencode/pull/35233) ("feat(core): run subagent commands in background") — unrelated feature; diff-inspected, does not touch the env key.

Neither covers this fix.

### Implementation Plan

**Understand:** The skill file is the primary user-facing config reference. Two example blocks use `"env"` instead of `"environment"`, causing env vars to be silently dropped when users follow the example. The mechanism: Effect schema parsing strips unknown fields by default without error.

**Match:** The existing schema (`packages/core/src/config/mcp.ts:12`) and runtime (`packages/opencode/src/mcp/index.ts:342`) are already correct. The pattern is: documentation should match implementation. A directly analogous case in the same repo is `packages/core/src/config/formatter.ts:8`, which also defines `environment` — that field appears correctly in the skill's formatter examples.

**Plan:**
1. In `packages/core/src/plugin/skill/customize-opencode.md`, change `"env": {}` (line 115) to `"environment": {}`.
2. Change `"env": { "BROWSER": "chromium" }` (line 374) to `"environment": { "BROWSER": "chromium" }`.
3. Add a regression test to `packages/core/test/plugin/skill.test.ts` that pins this: assert the skill content does not match `/"env":\s*\{/` and does contain `"environment"`.

**Implement:** Two character-level changes in the skill file, plus a regression test. Clean branch off `dev`.

**Review:**
- PR title follows `fix(skill): <summary>` convention, matching recent merged skill fixes.
- PR body: closes #35860, explains why before what, notes the silent-drop mechanism.
- `bun test test/plugin/skill.test.ts` — 2 pass, 0 fail before pushing.
- `bun run typecheck` (all 30 packages) — 30 successful before pushing.

**Evaluate:** After the fix: `grep '"environment"' packages/core/src/plugin/skill/customize-opencode.md` returns both corrected lines. The regression test fails if either line is reverted to `"env"`.

---

## Testing Strategy

### Unit Tests

Added a new regression test in `packages/core/test/plugin/skill.test.ts` (commit `26f1c86a0`):

```ts
test("customize-opencode skill uses correct 'environment' key for MCP local server env vars", () => {
  const content = readFileSync(
    join(import.meta.dir, "../../src/plugin/skill/customize-opencode.md"),
    "utf-8",
  )
  expect(content).not.toMatch(/"env":\s*\{/)
  expect(content).toContain('"environment"')
})
```

This test directly pins the regression: if either `"env": {}` or `"env": { ... }` reappears in the skill file, the test fails. It follows the project's existing test pattern of reading from `import.meta.dir`-relative paths (the same helper pattern used in `test/config/` tests).

**Fail-before / pass-after confirmed:** Reverting the skill file change causes this test to fail (`"env": {` matches the negative assertion). With the fix in place, both assertions pass.

### Existing Tests

`bun test test/plugin/skill.test.ts` — 2 pass, 0 fail (was 1 pass before the new test was added).

```
bun test v1.3.14 (0d9b296a)
 2 pass
 0 fail
 3 expect() calls
Ran 2 tests across 1 file.
```

### Manual Testing

The fix is in a markdown file — there is no TUI interaction path to exercise for this specific change. Verification:
1. `grep -n '"environment"' packages/core/src/plugin/skill/customize-opencode.md` → returns lines 115 and 374, both correct.
2. Compared against `packages/core/src/config/mcp.ts:12` (schema field name) — matches.
3. Compared against `packages/opencode/src/mcp/index.ts:342` (runtime consumer) — matches.
4. `bun run typecheck` (all 30 packages) — 30 successful, 0 errors.

---

## Implementation Notes

### What I built

**Commit 1 — `2b69ab1d2` (2026-07-07):** `fix(skill): correct MCP local server env key to environment in customize-opencode`

One targeted edit in `packages/core/src/plugin/skill/customize-opencode.md`:

```diff
-      "env": {}
+      "environment": {}
```

```diff
-      "env": { "BROWSER": "chromium" }
+      "environment": { "BROWSER": "chromium" }
```

**Commit 2 — `26f1c86a0` (2026-07-15):** `test(skill): pin regression for MCP env key in customize-opencode skill`

Added regression test in `packages/core/test/plugin/skill.test.ts` asserting the skill content uses `"environment"` and not `"env": {`.

**Net change:** 2 files, 4 lines changed (3 insertions(+), 1 deletion(-)).

### Verification

- `bun test test/plugin/skill.test.ts` — 2 pass, 0 fail
- `bun run typecheck` (turbo, 30 packages) — 30 successful, 0 errors
- All four oss-verify gates pass (run before pushing — see Solution Approach)
- `git diff --stat upstream/dev...fix/skill-mcp-env-key`:
  ```
  packages/core/src/plugin/skill/customize-opencode.md | 4 ++--
  packages/core/test/plugin/skill.test.ts              | 14 +++++++++++++-
  2 files changed, 16 insertions(+), 2 deletions(-)
  ```

### Challenges Overcome

**Finding a fixable issue in an actively-bot-claimed repo:** The opencode project runs its own `opencode-agent` bot that monitors and claims open issues. Most unclaimed issues in the bug queue were V2-specific (targeting a not-yet-released version). I had to look at issues filed within the last 24 hours before the bot processed them, cross-checking labels to filter V2-only bugs.

**Confirming the fix without running the subprocess:** The full reproduction requires a running MCP server process that logs `process.env`. I couldn't easily set that up, so I verified through the code path instead — tracing from the schema definition, through the config loader's parse behavior, to the runtime's `mcp.environment` read. The regression test provides ongoing verification without requiring the subprocess.

### Code Changes

- **Files modified:** `packages/core/src/plugin/skill/customize-opencode.md`, `packages/core/test/plugin/skill.test.ts`
- **Commits:**
  - `2b69ab1d2` — fix: two env→environment corrections in skill file
  - `26f1c86a0` — test: regression test pinning the correct field name

---

## Pull Request

**PR Link:** [anomalyco/opencode#35867](https://github.com/anomalyco/opencode/pull/35867)

**Summary:** The built-in `customize-opencode` skill used `"env"` as the key for MCP local server environment variables in two example blocks. The config schema (`McpLocalConfig`) and the runtime (`mcp/index.ts:342`) both use `"environment"`. The Effect schema parser strips unknown keys silently, so users following the example had their env vars dropped with no error — exactly the silent debugging session the reporter described. This PR corrects both occurrences to `"environment"` and adds a regression test that fails if either reverts. Two files changed.

**Before/After:**

Before (what the skill showed — what users copied):
```json
"mcp": {
  "playwright": {
    "type": "local",
    "command": ["npx", "-y", "@playwright/mcp"],
    "enabled": true,
    "env": { "BROWSER": "chromium" }   ← silently dropped by schema parser
  }
}
```

After (what the skill now shows):
```json
"mcp": {
  "playwright": {
    "type": "local",
    "command": ["npx", "-y", "@playwright/mcp"],
    "enabled": true,
    "environment": { "BROWSER": "chromium" }   ← reaches the subprocess
  }
}
```

**Acceptance criteria:**
- [x] Two occurrences of `"env"` in MCP local server examples corrected to `"environment"`
- [x] New regression test added — fails if `"env": {` reappears in the skill
- [x] All tests pass (`bun test test/plugin/skill.test.ts` — 2 pass, 0 fail)
- [x] Typecheck passes across all 30 packages
- [x] Diff scoped to the issue — no unrelated changes
- [x] `Closes #35860` in PR description

**Maintainer Feedback:** *(pending — will update with dates and commit refs when received)*

**Status:** Submitted — awaiting maintainer review.

---

## Learnings & Reflections

### Technical Skills Gained

This contribution required tracing a "no error, wrong behavior" class of bug — harder to find than crashes because there is no signal pointing to the problem. The key technique was reading the issue reporter's debugging story (30 minutes, no error, degraded MCP behavior) and identifying what "silently dropped" means in the schema layer: Effect's `Schema.parseJson` strips unknown fields by design rather than rejecting them. Understanding this let me confirm the fix without needing to reproduce the full subprocess behavior.

I also learned to cross-reference a built-in skill/documentation file against both the schema definition and the runtime consumer. Documentation that drifts from implementation without a type-level or test-level pin is a recurring class of bug in monorepos where the docs live in the source but aren't covered by the type system. The regression test I added provides exactly that pin for future changes.

### Challenges Overcome

Finding a fixable issue under heavy development pressure was the main challenge. The opencode project's own `opencode-agent` bot was actively claiming most open bugs, and most unclaimed issues were V2-specific — targeting a version not yet on the default branch. I had to filter issues filed within the last 24 hours and cross-check labels carefully. The lesson: in an actively-bot-managed repo, issue freshness is as important as issue quality.

### What I'd Do Differently Next Time

I would look for "documentation drifts from implementation" issues proactively by running a targeted grep (`grep '"env"' packages/core/src/plugin/skill/*.md`) against the schema's actual field names, rather than waiting for a user report. That kind of cross-reference check is fast and catches a real class of bugs before they affect users. A greppable test that asserts all skill examples match the schema would catch these automatically — something worth proposing to the maintainers as a follow-up.

---

## Resources Used

- [anomalyco/opencode#35860](https://github.com/anomalyco/opencode/issues/35860) — the issue
- [anomalyco/opencode#30892](https://github.com/anomalyco/opencode/issues/30892) — earlier report of same symptom
- [anomalyco/opencode#26332](https://github.com/anomalyco/opencode/issues/26332) — earlier report of same symptom
- `packages/core/src/config/mcp.ts` — schema definition confirming `environment` field name
- `packages/opencode/src/mcp/index.ts` — runtime confirming `mcp.environment` is what's read
- `packages/core/src/plugin/skill/customize-opencode.md` — the file changed
- `packages/core/test/plugin/skill.test.ts` — the test file updated
