# Contribution 1: OpenCode crashes on startup when .agents/ contains Cursor-format agent files

**Contribution Number:** 1  
**Student:** Joshua Liu  
**Issue:** [sst/opencode#31481](https://github.com/sst/opencode/issues/31481)  
**Related:** [sst/opencode#27133](https://github.com/sst/opencode/issues/27133)  
**Fork:** [github.com/NumerousJLs/opencode](https://github.com/NumerousJLs/opencode)  
**Status:** Phase I Complete

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

### Reproduction Evidence

- **Root cause confirmed in source:** `packages/opencode/src/config/agent.ts` line `result[config.name] = ConfigParse.schema(ConfigAgentV1.Info, config, item)` throws when `tools` is an array instead of an object.
- **Fix pattern already in same file:** `loadMode()` (lines below `load()`) uses `Exit.isSuccess(parsed)` and simply continues on failure.

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

**Plan:**
1. In `load()`, replace `ConfigParse.schema(ConfigAgentV1.Info, config, item)` with `Schema.decodeUnknownExit(ConfigAgentV1.Info)(config, ...)`.
2. Wrap in `Exit.isSuccess` check — add the agent to the result on success, skip (with a logged warning) on failure.
3. Add `Cause` import from `effect` to format the error message in the warning.
4. Apply the same improvement to `loadMode()` — currently it silently skips, which is also unhelpful. Add a warning there too for consistency (addresses #27133's point that both should surface errors).
5. Write a test that verifies a directory with one valid and one invalid agent file loads the valid agent without crashing.

**Implement:** Branch off `dev` (not `main` — opencode's default branch is `dev`).

**Review:** 
- PR title follows `fix(opencode): <summary>` convention.
- PR body uses the template: Closes #31481, Closes #27133.
- Keep description short and in own words — no AI-generated walls of text.
- Run `bun typecheck` and `bun test packages/opencode/test/config/` before opening PR.

**Evaluate:** Create a test directory with one valid and one broken agent file. Startup should succeed and load the valid agent. The broken agent's name should appear in a warning.

---

## Testing Strategy

### Unit Tests

- [ ] Test: directory with 1 valid + 1 invalid agent file loads valid agent without crashing
- [ ] Test: directory with 1 valid + 1 invalid mode file loads valid mode and logs a warning
- [ ] Test: all-valid agent directory still loads all agents (regression)

### Manual Testing

Run `bun dev` against a test directory containing:
- A valid agent with object-format `tools`
- An invalid agent with array-format `tools` (Cursor-style)

Expected: startup succeeds, valid agent loads, warning shown for invalid agent.

---

## Implementation Notes

*To be filled in during Phase III.*

---

## Pull Request

**PR Link:** [To be added]

**Maintainer Feedback:**
*To be filled in during Phase IV.*

**Status:** Not yet submitted

---

## Learnings & Reflections

*To be filled in at the end of the program.*
