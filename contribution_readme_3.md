# Contribution 3: Built-in customize-opencode skill uses wrong MCP env key

**Contribution Number:** 3  
**Student:** Joshua Liu  
**Issue:** [anomalyco/opencode#35860](https://github.com/anomalyco/opencode/issues/35860)  
**Fork:** [github.com/NumerousJLs/opencode](https://github.com/NumerousJLs/opencode)  
**Branch:** `fix/skill-mcp-env-key` (off `dev`)  
**Pull Request:** [anomalyco/opencode#35867](https://github.com/anomalyco/opencode/pull/35867)  
**Status:** Phase IV Complete — PR submitted, awaiting review

---

## Why I Chose This Issue

opencode has a built-in skill called `customize-opencode` that users invoke with `/customize-opencode` in the TUI. It serves as the canonical reference for configuring opencode — users read it and copy its examples directly into their `opencode.json`. If an example in this skill is wrong, users have no obvious way to find out, because the config loader strips unknown keys silently instead of throwing an error.

This issue was filed the same day I picked it up. The reporter spent 30 minutes debugging why their MCP environment variables were never reaching their subprocess — they had no error, no warning, just silent misbehavior. The root cause is that the skill shows `"env"` as the key for MCP subprocess environment variables, but the actual config schema and the runtime both use `"environment"`. The fix is two character changes in one file, but the impact on users following the docs is real.

---

## Understanding the Issue

### Problem Description

opencode lets users configure local MCP (Model Context Protocol) servers in their `opencode.json`. A local MCP server is an external process that opencode spawns and communicates with. You can pass environment variables to that subprocess by setting them in the config.

The built-in `customize-opencode` skill (the file users see when they run `/customize-opencode`) shows this example:

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

The skill shows `"env"` in two places. Users who copy either example have their env vars silently dropped. Their MCP server runs without the configured variables, which can cause degraded behavior (wrong browser, missing credentials, missing feature flags) with no indication of why.

### Affected Components

`packages/core/src/plugin/skill/customize-opencode.md` — the built-in skill file. Two places:
- Line 115 (in the top-level config reference block): `"env": {}`
- Line 374 (in the dedicated MCP section): `"env": { "BROWSER": "chromium" }`

---

## Reproduction Process

### Environment Setup

No special setup needed. The bug is a wrong key name in a markdown file that serves as both a skill prompt and a config example.

### Steps to Reproduce

1. Open opencode and run `/customize-opencode` in the TUI.
2. Read the MCP section. It shows `"env"` as the field for subprocess env vars.
3. Copy the example into your `opencode.json`, adding real env vars under `"env"`.
4. Restart opencode. Your MCP server starts but the env vars are absent.
5. Add a `console.log(process.env)` in the MCP server to confirm the vars were never set.

There is no error. The config loads successfully because the schema parser strips unknown keys without warning.

### Reproduction Evidence

- **Schema confirmed**: `packages/core/src/config/mcp.ts:12` — field is `environment`.
- **Runtime confirmed**: `packages/opencode/src/mcp/index.ts:342` — runtime reads `mcp.environment`.
- **Skill confirmed**: the file on `upstream/dev` at the time of this fix has `"env"` at both locations (lines 115 and 374).
- **No error path**: `Schema.parseJson` strips unknown keys — there is no runtime warning when `"env"` is discarded.

---

## Solution Approach

### Analysis

The mismatch is one-directional: the runtime and schema are correct, the skill example is wrong. There is no ambiguity — `"env"` was a wrong key name, not a renamed field. The fix is to update the two example blocks in the skill file to use `"environment"`.

The same file was touched by two existing open PRs:
- PR #33655 ("docs(skill): update customize-opencode permissions") — updates permission guidance, does not touch the MCP section or the env key.
- PR #35233 ("feat(core): run subagent commands in background") — unrelated feature, does not touch the env key.

Neither covers the env/environment fix, so there is no competing work.

### Implementation Plan

**Understand:** The skill file is the primary user-facing config reference. Two example blocks use `"env"` instead of the correct `"environment"` key, causing env vars to be silently dropped when users follow the example.

**Match:** The existing schema (`packages/core/src/config/mcp.ts`) and runtime (`packages/opencode/src/mcp/index.ts`) are already correct. The fix is to make the documentation match the implementation, not the other way around.

**Plan:**
1. In `packages/core/src/plugin/skill/customize-opencode.md`, change `"env": {}` (line 115) to `"environment": {}`.
2. Change `"env": { "BROWSER": "chromium" }` (line 374) to `"environment": { "BROWSER": "chromium" }`.

**Implement:** Two character-level changes, one file, clean branch off `dev`.

**Review:**
- PR title: `fix(skill): correct MCP local server env key to environment`
- Closes #35860
- `bun test test/plugin/skill.test.ts` passes

**Evaluate:** After the fix, a user copying the example into their config will have their vars reach the subprocess. The field name now matches both the schema and runtime.

---

## Testing Strategy

### Unit Tests

`packages/core/test/plugin/skill.test.ts` — the existing test loads and parses each built-in skill and verifies it is a non-empty string. Ran with the fix applied:

```
✓ 1 pass, 0 fail, 1 expect() calls
```

This test would not catch a wrong key name in an example block (it only checks that the skill file loads), but it does verify that no parsing error was introduced.

### Manual Testing

The change is in a markdown file — there is no runtime path to exercise via the TUI for this specific fix. Correctness is verified by:
1. Comparing the fixed key name against `packages/core/src/config/mcp.ts:12` (schema) — matches.
2. Comparing against `packages/opencode/src/mcp/index.ts:342` (runtime) — matches.
3. Confirming via `grep -n '"environment"' packages/core/src/plugin/skill/customize-opencode.md` that both occurrences now use the correct key.

A user can functionally verify the fix by adding env vars under `"environment"` in a local MCP server config and confirming they appear in `process.env` inside the subprocess.

---

## Implementation Notes

### What I built

One targeted edit in `packages/core/src/plugin/skill/customize-opencode.md`:

**Before (line 115):**
```json
"env": {}
```

**After (line 115):**
```json
"environment": {}
```

**Before (line 374):**
```json
"env": { "BROWSER": "chromium" }
```

**After (line 374):**
```json
"environment": { "BROWSER": "chromium" }
```

Net change: 1 file, 2 lines changed (2 insertions, 2 deletions).

### Verification

- `bun test test/plugin/skill.test.ts` — 1 pass, 0 fail
- `git diff --stat` — 1 file changed, 2 insertions(+), 2 deletions(-)
- All four oss-verify gates pass before pushing (see below)

### oss-verify gates (run before pushing)

**Gate 1 — issue timeline:** No cross-references from any open PR on issue #35860.

**Gate 2 — keyword search:** No open PRs matching "env environment mcp skill", "customize-opencode env", or "McpLocal environment" targeting this fix.

**Gate 3 — file-touching PRs:** Two open PRs touch `customize-opencode.md` (#33655 permissions update, #35233 subagent commands). Diff-inspected both — neither changes the `"env"` key. No conflict.

**Gate 4 — code on dev:** Confirmed `"env"` at lines 115 and 374 on `upstream/dev` at the time of the fix.

### Code Changes

- **Files modified:** `packages/core/src/plugin/skill/customize-opencode.md`
- **Files added:** none
- **Commit:** `fix(skill): correct MCP local server env key to environment in customize-opencode`

---

## Pull Request

**PR Link:** [anomalyco/opencode#35867](https://github.com/anomalyco/opencode/pull/35867)

**Summary:** The built-in `customize-opencode` skill used `"env"` as the key for MCP local server environment variables in two example blocks. The config schema and runtime both use `"environment"`. The config loader strips unknown keys silently, so users following the example had their env vars dropped with no error. This PR corrects both occurrences to `"environment"`. One file changed, 2 lines.

**Maintainer Feedback:** *(pending)*

**Status:** Submitted — awaiting maintainer review.

---

## Learnings & Reflections

### Technical Skills Gained

This contribution required tracing a "no error, wrong behavior" class of bug — harder to find than crashes because there is no signal pointing to the problem. The key technique was reading the issue reporter's debugging story (30 minutes, no error, BM25-only results), identifying what "silently dropped" means in the schema layer, then confirming the specific mechanism: Effect schema parsing strips unknown fields by default rather than rejecting them.

I also learned to cross-reference a built-in skill/documentation file against both the schema definition (`config/mcp.ts`) and the runtime consumer (`mcp/index.ts`). Documentation that drifts from implementation without a test to pin it is a recurring class of bug in projects where the docs are part of the source but aren't covered by the type system.

### Challenges Overcome

Finding a fixable issue in a codebase under heavy development was the main challenge. The opencode project's own `opencode-agent` bot was actively claiming most open issues, and the majority of unclaimed bugs were V2-specific (targeting a version not yet on the default branch). This required looking at issues filed within the last 24 hours before the bot processed them, and cross-checking labels carefully.

Once the issue was identified, the technical fix was straightforward — the challenge was confirming it thoroughly enough to be confident before committing, without running the full MCP subprocess stack.

### What I'd Do Differently Next Time

I would look for these "documentation drifts from implementation" issues proactively, rather than waiting for a user report. A quick grep for `"env"` in all skill markdown files against the schema's actual field names would have caught this before it was filed. That kind of cross-reference check (does every example in the docs match the schema?) is fast and finds real bugs.

---

## Resources Used

- [anomalyco/opencode#35860](https://github.com/anomalyco/opencode/issues/35860) — the issue
- `packages/core/src/config/mcp.ts` — schema definition confirming `environment` field name
- `packages/opencode/src/mcp/index.ts` — runtime confirming `mcp.environment` is what's read
- `packages/core/src/plugin/skill/customize-opencode.md` — the file changed
