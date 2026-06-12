# Contribution 1: OpenCode crashes on startup when .agents/ contains Cursor-format agent files

**Contribution Number:** 1  
**Student:** Joshua Liu  
**Issue:** [sst/opencode#31481](https://github.com/sst/opencode/issues/31481)  
**Related:** [sst/opencode#27133](https://github.com/sst/opencode/issues/27133)  
**Fork:** [github.com/NumerousJLs/opencode](https://github.com/NumerousJLs/opencode)  
**Branch:** `fix/agent-load-graceful-skip` (off `dev`)  
**Pull Request:** [anomalyco/opencode#31992](https://github.com/anomalyco/opencode/pull/31992)  
**Status:** Phase IV Complete — PR submitted; deferred to earlier duplicate PR #29784

---

## Why I Chose This Issue

opencode is one of the most active open source AI coding agent projects right now (172k+ stars, multiple external contributor PRs merged same-day). I wanted a bug fix with a clear root cause rather than a vague feature request, and this one stood out because the fix is fully traceable in the source. The error message in the crash log literally tells you where to look: `SchemaError: Expected object | undefined, got ["execute/runNotebookCell"] at ["tools"]`. The bug is that `packages/opencode/src/config/agent.ts`'s `load()` function calls `ConfigParse.schema()` which throws on any schema mismatch, crashing the whole startup. The fix already exists in the same file — `loadMode()` uses a graceful `Exit.isSuccess` pattern to skip invalid files instead of throwing. The issue also has a related open issue (#27133) that frames the broader inconsistency and was filed months earlier, so fixing both together makes the PR more useful.

I read the CONTRIBUTING guide, studied 10+ recent merged PRs to understand the PR format and style expectations, and confirmed the root cause by reading the source before claiming.

---

## Understanding the Issue

### Problem Description

opencode is an AI coding agent with support for custom `.agents/` directories. Cursor, another AI coding tool, also uses `.agents/` with a different frontmatter format where `tools` is a YAML array (`tools: [execute/runNotebookCell]`). When opencode tries to load an agent file using Cursor's format, it throws a `ConfigInvalidError` that crashes the entire startup — all 4-5 API requests fail, nothing loads.

The same file in `.opencode/mode/` would be silently skipped instead of crashing, because `loadMode()` was written with a graceful failure path. `load()` for agents was not.

### Expected Behavior

An invalid or incompatible agent file should be skipped with a warning, and the rest of the agents should load normally. Same behavior as `loadMode()`.

### Current Behavior

Any schema-invalid agent file in `.agents/` crashes the entire startup:

```
SchemaError: Expected object | undefined, got ["execute/runNotebookCell"]
  at ["tools"]
ConfigInvalidError: ConfigInvalidError
...
Error: 4 of 5 requests failed: Unexpected server error.
```

### Affected Components

`packages/opencode/src/config/agent.ts` — the `load()` function calls `ConfigParse.schema(ConfigAgentV1.Info, config, item)` which throws on schema mismatch. Compare to `loadMode()` in the same file which uses `Schema.decodeUnknownExit()` + `Exit.isSuccess()` to skip invalid files gracefully.

The schema itself (`packages/core/src/v1/config/agent.ts`) defines `tools` as `Schema.optional(Schema.Record(Schema.String, Schema.Boolean))` — a key-value object. Cursor-format files pass a YAML array, which fails the schema check.

---

## Reproduction Process

### Environment Setup

Requirements: Bun 1.3+. Clone the repo, run `bun install` from the root, then `bun dev <dir>` to run against a directory.

### Steps to Reproduce

1. Create a directory with `.agents/agents/example.agent.md`:
```markdown
---
name: "My Agent"
description: "Test agent"
tools: [execute/runNotebookCell]
---
You are a helpful assistant.
```
2. Run `opencode` (or `bun dev`) in that directory.
3. Startup crashes with `4 of 5 requests failed: Unexpected server error`.

### How I Actually Reproduced It

The reporter ran the full app, but to isolate the bug I called the affected
function directly. `load()` scans `{agent,agents}/**/*.md` relative to each
config dir, so I built a folder with an `agents/` subdir containing one valid
agent and one Cursor-format agent (`tools` as a YAML array), then called
`ConfigAgent.load(dir)` on the current `dev` branch.

**Before the fix (the bug):**
- `load()` **threw `ConfigInvalidError`** — so *none* of the agents loaded, not
  even the valid one. This is the startup crash (`4 of 5 requests failed`).
- `loadMode()` on a folder with a valid + invalid mode returned only the valid
  mode and **silently dropped** the invalid one — no error, no warning. This is
  the agent-vs-mode inconsistency in #27133.

### Reproduction Evidence

- **Root cause confirmed in source:** `packages/opencode/src/config/agent.ts`,
  `result[config.name] = ConfigParse.schema(ConfigAgentV1.Info, config, item)`
  throws when `tools` is an array instead of an object.
- **Fix pattern already in same file:** `loadMode()` (just below `load()`) uses
  `Schema.decodeUnknownExit` + `Exit.isSuccess(parsed)` and continues on failure.
- **Confirmed still present on latest `dev`** at the time of work.

---

## Solution Approach

### Analysis

`load()` and `loadMode()` in `packages/opencode/src/config/agent.ts` both parse markdown frontmatter files into `ConfigAgentV1.Info`. They were written with different error handling:

- `loadMode()`: Uses `Schema.decodeUnknownExit()` and checks `Exit.isSuccess()` — skips invalid files, keeps loading.
- `load()`: Uses `ConfigParse.schema()` — throws `InvalidError` on any schema mismatch, propagating up through `Config.loadInstanceState` and crashing startup.

The fix is to make `load()` consistent with `loadMode()`.

### Implementation Plan

**Understand:** A schema-invalid `.agents/` file crashes startup because `load()` throws rather than skips.

**Match:** `loadMode()` in the same file already uses the correct pattern. `ConfigCommand.load()` in `config/command.ts` is another reference for the `Exit.isSuccess` pattern with `Cause` for error details.

**Plan (as executed):**
1. In `load()`, replaced `ConfigParse.schema(...)` with `Schema.decodeUnknownExit(ConfigAgentV1.Info)(config, ...)` and an `Exit.isSuccess` check — keep the agent on success, skip on failure.
2. On the skip path, emit `console.warn` naming the file and the schema error. (Chose `console.warn` because these are plain `async` functions called via `Effect.promise`, not Effect contexts; `console.warn` is already used elsewhere in the package and keeps the change minimal. Routing through the Effect logger would mean restructuring return types and both call sites — out of scope for a bug fix.)
3. Added the `Cause` import from `effect` to format the error in the warning; removed the now-unused `ConfigParse` import.
4. Applied the same warning to `loadMode()`'s existing skip path so a bad mode file is visible too (resolves #27133).
5. Added a test that pins the regression: one valid + one invalid file → valid loads, no crash.

**Decision left out of scope:** actually *converting* Cursor's array format into opencode's permission model. That's a feature, not a bug fix, and it carries a semantic-mismatch risk (Cursor's tool names aren't opencode tools), so per opencode's "features need design discussion first" rule it would belong in a separate feature-request issue. This PR only stops the crash and makes skips visible.

**Implement:** Branched off `dev` (opencode's default branch is `dev`, not `main`).

**Review:** 
- PR title follows `fix(opencode): <summary>` convention.
- PR body uses the template: Closes #31481, Closes #27133.
- Keep description short and in own words — no AI-generated walls of text.
- Run `bun typecheck` and `bun test packages/opencode/test/config/` before opening PR.

**Evaluate:** Create a test directory with one valid and one broken agent file. Startup should succeed and load the valid agent. The broken agent's name should appear in a warning.

---

## Testing Strategy

### Unit Tests (added in `packages/opencode/test/config/agent.test.ts`)

- [x] `load()` does not throw when an agent file is invalid
- [x] `load()` keeps valid agents and skips the invalid Cursor-format one
- [x] `loadMode()` keeps valid modes and skips the invalid one

The test creates a temp dir with `agents/valid.md`, `agents/cursor.md` (array
`tools`), `modes/valid.md`, and `modes/bad.md`, then asserts the valid entries
load and the invalid ones are dropped without throwing.

**Fail-before / pass-after confirmed:** with the source fix temporarily reverted,
the new test fails (2 of 3 cases) because `load()` throws; with the fix in place
it passes (3 of 3). This proves the test pins the actual regression.

### Manual Testing

Called `ConfigAgent.load()` and `ConfigAgent.loadMode()` directly against a temp
folder with valid + invalid files. Before: `load()` threw and returned nothing.
After: the valid agent loads and the invalid one is skipped with a warning that
names the file and the schema error (`Expected object ... got [array] at tools`).

---

## Implementation Notes

### What I built

Two targeted edits in `packages/opencode/src/config/agent.ts` (the only source
file changed):

- **`load()`** — switched from the throwing `ConfigParse.schema()` to
  `Schema.decodeUnknownExit` + `Exit.isSuccess`. On success the agent is added;
  on failure the file is skipped with a `console.warn`. Removed the now-unused
  `ConfigParse` import; added `Cause` for the warning message.
- **`loadMode()`** — added the same `console.warn` on its existing skip path so a
  bad mode file is no longer silently dropped (#27133).

Net change: one source file (~12 lines) plus one new test file. I left
`ConfigParse.schema()` itself unchanged, since throwing is the correct behavior
for a single intentional config file like `opencode.json`.

### Verification (all clean)

- `bun typecheck` — pass
- `oxlint` on the changed + new files — 0 warnings, 0 errors
- `git diff --check` — clean
- `bun test test/config test/agent` — 233 pass, 0 fail (was 230 + 3 new)

### Challenges / notes

- The biggest correctness check was confirming `load()` *throws* (not "returns
  empty"). An early repro used a directory layout that didn't match the glob
  `load()` scans, so it found no files and looked like it returned `{}`. Fixing
  the layout to `agents/*.md` showed the real behavior: it throws.
- Logging mechanism was the one judgment call. Picked `console.warn` for minimal
  blast radius (these aren't Effect contexts); flagged it as the place a
  maintainer might prefer a different approach.

### Code Changes

- **Files modified:** `packages/opencode/src/config/agent.ts`
- **Files added:** `packages/opencode/test/config/agent.test.ts`
- **Commit:** `fix(opencode): skip invalid agent config files instead of crashing startup`

---

## Pull Request

**PR Link:** [anomalyco/opencode#31992](https://github.com/anomalyco/opencode/pull/31992)

**Summary:** Makes `load()` and `loadMode()` in `packages/opencode/src/config/agent.ts` skip an invalid agent or mode config file with a warning instead of throwing (which crashed startup) or dropping it silently. Valid agents and modes still load. Closes #31481 and #27133. The PR is two files: the fix and a regression test.

**Verification:** `bun typecheck`, `oxlint`, and `git diff --check` clean; `bun test test/config test/agent` passes 233. The repo's `check-compliance`, `check-standards`, and contributor-label CI checks all passed on open.

**Maintainer Feedback:** The repository's duplicate-detection bot flagged an earlier pull request, [#29784](https://github.com/anomalyco/opencode/pull/29784), which I confirmed is a genuine duplicate. It predates mine by two weeks, changes the same file, takes the same approach to #27133, and reports invalid files through a structured `Session.Event.Error` rather than a console warning, which is a more complete answer to the issue. The right thing in open source when you find an earlier active PR doing the same work is to defer to it, so I am closing mine in favor of #29784 and noting there that it also resolves #31481.

**Status:** Submitted, then deferred to the earlier duplicate PR #29784.

---

## Learnings & Reflections

### Technical skills gained

I learned how opencode is built, which was new ground: a Bun and TypeScript monorepo on the Effect library, where errors and dependencies are explicit values rather than thrown exceptions. The whole bug came down to that model. One loader validated with a helper that throws, its sibling validated with `Schema.decodeUnknownExit` and checked the result as an `Exit` value, and the fix was to make the throwing one behave like the safe one. I also learned to read a project's culture from its data instead of guessing, by measuring review latency, how often outside contributors get merged, and whether PRs carry inline comments, which told me opencode wants short prose descriptions and almost no annotation.

### The biggest lesson

Two things I will not repeat. First, I based my branch on a dev commit that the maintainers later force-pushed away, and without a rebase the pull request would have carried thirty-seven unrelated files. The check that catches this is the full three-dot diff against current upstream, `git diff upstream/dev...HEAD`, not just checking whether my own file conflicts. Second, and more important, I opened a pull request that duplicated an existing active one because I trusted a single search that came back empty. The reliable way to find existing PRs is the issue timeline, which lists every PR that references the issue, not a body-text search. I should run that check for every issue I plan to close, and run it again right before opening.

### What I would do differently

Spend more of the up-front time confirming the issue is genuinely unclaimed, using the timeline and a keyword search rather than one query, before writing any code. The fix itself was small and correct. The cost came from skipping that verification, which is cheap compared to the work it would have saved.

### On the outcome

The pull request did not merge, and that is a normal open source result. The milestone was a clean, review-ready submission, which I made, and the more valuable part was learning how a real project's review process, culture, and contributor etiquette actually work. Deferring gracefully to an earlier contributor is part of that, and it was worth doing right.
