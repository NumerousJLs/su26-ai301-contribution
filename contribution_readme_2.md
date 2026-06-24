# Contribution 2: Agent mode resets to default when navigating back to a session

**Contribution Number:** 2  
**Student:** Joshua Liu  
**Issue:** [anomalyco/opencode#31862](https://github.com/anomalyco/opencode/issues/31862)  
**Fork:** [github.com/NumerousJLs/opencode](https://github.com/NumerousJLs/opencode)  
**Branch:** `fix/restore-agent-on-session-nav` (off `dev`)  
**Pull Request:** [anomalyco/opencode#TBD](https://github.com/anomalyco/opencode/pulls) *(pending push)*  
**Status:** Phase IV In Progress — PR ready to push, pending manual test confirmation

---

## Why I Chose This Issue

While investigating the TUI session route for my first contribution, I noticed the navigation `createEffect` in `packages/tui/src/routes/session/index.tsx` was already reading `result.data.workspaceID` and `result.data.directory` to restore TUI context on session switch — but not `result.data.agent`, which the server also provides. That asymmetry is a clean bug: the data is already there, the server tracks it correctly, and the TUI just ignores it.

The issue itself was filed recently and has no cross-references or competing PRs. opencode's custom agent system is a core feature — users can define agents like `researcher`, `reviewer`, or `oracle` in their config and switch between them. When you navigate away from a session using a non-default agent and come back, the TUI silently resets to `build` mode. The fix is three guarded lines in one file, consistent with a pattern that already exists in the same block of code.

---

## Understanding the Issue

### Problem Description

opencode lets users define multiple agents — named configurations with their own system prompts and tool permissions. The two built-in ones are `build` (the default, can write code) and `plan` (read-only planning mode). Users can also define custom ones like `researcher` or `oracle` in their config.

When a user runs a prompt in a non-default agent, opencode fires an `AgentSwitched` event, which the backend projects into the `session.agent` column of the SQLite database. The `session.get` API endpoint returns this field. So the server always knows what agent a session last ran in.

The TUI has a global reactive store called `local.agent` that holds which agent is currently active. When you switch sessions, the navigation `createEffect` fetches the session from the server and restores workspace and editor directory from the response — but it never reads `result.data.agent`. So `local.agent` stays on whatever it was when you navigated away, defaulting to `build` if this is a fresh load.

### Expected Behavior

Navigate to a session that last ran with a custom agent or plan mode. The agent indicator in the TUI prompt bar should show that agent — the same one the server has stored.

### Current Behavior

The agent indicator always shows `build` (the default) when you navigate to a session, regardless of what agent that session was actually using. You have to manually switch back every time.

### Affected Components

`packages/tui/src/routes/session/index.tsx` — the navigation `createEffect` (around line 279) fetches session data and restores TUI context, but skips the `agent` field. The relevant line on upstream/dev:

```ts
if (route.sessionID === sessionID && scroll) scroll.scrollBy(100_000)
```

The guard and scroll were combined in one condition. The `agent` field from `result.data` was never read.

---

## Reproduction Process

### Environment Setup

Requirements: Bun 1.3+, repo cloned with upstream remote pointing to `anomalyco/opencode`. Run `bun install` from the repo root, then `bun run dev` to start the TUI from source.

### Steps to Reproduce

1. Run `bun run dev` from the repo root.
2. Open or create a session.
3. Press `Shift+Tab` to switch to a non-default agent (e.g. `plan`).
4. Send at least one message in that agent — this is required to trigger `AgentSwitched` and write `session.agent` to the DB. Just switching the selector without prompting doesn't persist.
5. Navigate to a different session (press `Esc` to reach the session list, select another session).
6. Navigate back to the original session.
7. **Observed:** the agent indicator shows `build` instead of `plan`.

### Reproduction Evidence

- **Root cause confirmed in source:** `packages/tui/src/routes/session/index.tsx` — the navigation `createEffect` reads `result.data.workspaceID` and `result.data.directory` but not `result.data.agent`.
- **Server-side confirmed:** `packages/core/src/session/projector.ts` — `AgentSwitched` events write `session.agent` to the DB. `packages/opencode/src/session/session.ts` `fromRow()` includes `agent: row.agent ?? undefined` in the response. The data is there.
- **Confirmed present on latest dev:** upstream/dev at v1.17.8 (the most recent release) still has the unmodified navigation `createEffect` with no agent restore logic. Verified with `git show upstream/dev:packages/tui/src/routes/session/index.tsx | grep "result.data.agent"` — no output.
- **No competing PRs:** ran four-gate oss-verify check (issue timeline, four keyword search angles, file-touching PR diff inspection, code on dev) — all passed clean.

---

## Solution Approach

### Analysis

The navigation `createEffect` in `routes/session/index.tsx` is the designated place for "restore TUI context from server when switching sessions." It already does this for two fields:

```ts
// Restores workspace
if (result.data.workspaceID !== previousWorkspace) {
  project.workspace.set(result.data.workspaceID)
}

// Restores editor directory
editor.reconnect(result.data.directory)
```

`result.data.agent` is a third field from the same API response. Adding the restore here is consistent with the existing pattern. The only additional complexity comes from two edge cases that need guards:

1. **CLI `--agent` flag:** If the user started opencode with `--agent plan`, that should take precedence over whatever the session stored. `prompt/index.tsx:321` already has this guard: `if (!args.agent) local.agent.set(msg.agent)`. Our restore needs the same check.

2. **Stale agent list on failed workspace bootstrap:** When navigating to a session in a different workspace and `sync.bootstrap()` fails (workspace no longer exists), `sync.data.agent` remains stale from the previous workspace. `local.agent.set()` validates the name against that stale list, would fail to find it, and emit a misleading "Agent not found" toast. Guard: only restore if bootstrap succeeded.

### Proposed Solution

After `sync.session.sync(sessionID)`, inside the stale-navigation guard, call `local.agent.set(result.data.agent)` when the three conditions hold: the server has a stored agent, the user didn't pass `--agent` on the CLI, and workspace bootstrap succeeded.

### Implementation Plan

**Understand:** The TUI ignores `result.data.agent` on session navigation, causing `local.agent` to always reset to the default.

**Match:** The restore pattern already exists twice in the same block (`workspaceID` → `project.workspace.set`, `directory` → `editor.reconnect`). The `--agent` guard already exists in `prompt/index.tsx:321`. The `bootstrapOk` flag is new but follows naturally from the existing `try { await sync.bootstrap(...) } catch {}` block.

**Plan:**
1. Import `useArgs` from `../../context/args` — this context is provided by `ArgsProvider` which wraps the whole app above the session route, so it's valid here.
2. Call `const args = useArgs()` alongside the other context hooks at the top of `Session()`.
3. Add `let bootstrapOk = true` before the workspace-switch block, and set `bootstrapOk = false` in the catch.
4. After `sync.session.sync(sessionID)`, inside `if (route.sessionID === sessionID)`, add the restore line with all three guards.

**Implement:** Branched off upstream/dev, rebased after upstream advanced to v1.17.8.

**Review:**
- PR title follows `fix(tui): <summary>` convention (matching `fix(tui): handle move directory errors` and similar recent merges).
- PR body uses the project template: closes issue, explains root cause, explains why the fix works, honest about what was verified.
- No AI-generated walls of text — the warning in the template is literal.
- `bun run typecheck` passes before pushing.

**Evaluate:** Run opencode from the fixed branch, switch to a non-default agent, send a message, navigate away, navigate back — agent indicator should show the correct agent.

---

## Testing Strategy

### Unit Tests

No new unit tests added. The fix is in a SolidJS reactive `createEffect` inside a component — not a pure function that can be called in isolation. The agent restoration depends on `local.agent` (a SolidJS store), `sync.data.agent` (a reactive memo), and `route.sessionID` (a reactive route context), none of which are easily testable outside the full TUI runtime. opencode's existing test suite doesn't test TUI component behavior, only core/config logic. Adding a test here would require a SolidJS test harness that the project doesn't have.

### Manual Testing

- [ ] Run `bun run dev` on the fixed branch
- [ ] Switch to non-default agent, send a message, navigate away and back — agent indicator correctly restored
- [ ] Verify `--agent plan` on CLI overrides session-stored agent (the `!args.agent` guard)
- [ ] `bun run typecheck` passes

*(Manual test pending — will check the checkbox in the PR once confirmed)*

---

## Implementation Notes

### What I built

Three targeted edits in `packages/tui/src/routes/session/index.tsx` — the only source file changed:

**Import added:**
```ts
import { useArgs } from "../../context/args"
```

**Context hook call added** (alongside existing hooks at the top of `Session()`):
```ts
const args = useArgs()
```

**Navigation createEffect — before:**
```ts
if (result.data.workspaceID !== previousWorkspace) {
  project.workspace.set(result.data.workspaceID)
  try {
    await sync.bootstrap({ fatal: false })
  } catch {}
}
editor.reconnect(result.data.directory)
await sync.session.sync(sessionID)
if (route.sessionID === sessionID && scroll) scroll.scrollBy(100_000)
```

**Navigation createEffect — after:**
```ts
let bootstrapOk = true
if (result.data.workspaceID !== previousWorkspace) {
  project.workspace.set(result.data.workspaceID)
  try {
    await sync.bootstrap({ fatal: false })
  } catch {
    bootstrapOk = false
  }
}
editor.reconnect(result.data.directory)
await sync.session.sync(sessionID)
if (route.sessionID === sessionID) {
  if (result.data.agent && !args.agent && bootstrapOk) local.agent.set(result.data.agent)
  if (scroll) scroll.scrollBy(100_000)
}
```

Net change: 1 file, 10 lines (8 added, 2 modified).

### Verification

- `bun run typecheck` — pass
- `oxlint packages/tui/src/routes/session/index.tsx` — 14 warnings (same as baseline before the change, 0 new)
- All four oss-verify gates re-run the day of push — clean

### Challenges / notes

**The `bootstrapOk` flag:** The existing `catch {}` was empty and I almost left it that way, assuming the stale-list risk was minor. But `local.agent.set()` validates the name against `sync.data.agent`, and if bootstrap failed, that list is from the wrong workspace. Calling `set()` on it produces a misleading "Agent not found" toast and does nothing — worse than silently skipping. The flag costs nothing in the common case (same-workspace navigation, `bootstrapOk` stays `true`) and prevents a confusing user-visible error on a failure path the codebase already treats as non-fatal.

**PR framing:** Originally framed around plan mode specifically, but Aiden (the maintainer) commented in Discord that plan mode is likely being removed in v2. Reframed the PR description around any non-default agent — custom agents are the broader use case and are clearly not going away. The fix itself was always generic; only the description changed.

**Code review findings:** Ran the project's `code-review` skill on the diff. Two confirmed issues surfaced and were fixed before pushing: the `--agent` guard (no equivalent in the original fix, but `prompt/index.tsx` has it), and the `bootstrapOk` flag. One candidate (a race with the `prompt/index.tsx` agent restore effect) was refuted — that effect fires at most once per session ID due to a `syncedSessionID` closure guard, so it doesn't create a repeating override conflict.

### Code Changes

- **Files modified:** `packages/tui/src/routes/session/index.tsx`
- **Files added:** none
- **Commit:** `fix(tui): restore agent mode when navigating to a session`

---

## Pull Request

**PR Link:** *(pending push)*

**Summary:** Reads `result.data.agent` in the navigation `createEffect` after `sync.session.sync()` and calls `local.agent.set()` to restore the TUI to the session's actual agent. Three guards: server has a stored agent, user didn't pass `--agent` on the CLI, workspace bootstrap succeeded. One file changed, 10 lines.

**Maintainer Feedback:** *(pending)*

**Status:** Ready to push — pending manual test confirmation and one `#dev` Discord post after the PR opens.

---

## Learnings & Reflections

### Technical Skills Gained

I learned SolidJS's reactive model deeply enough to reason about effect timing and closure behavior. The key insight is that `createEffect` in SolidJS runs its callback synchronously the first time during component setup, but the async IIFE inside it suspends at the first `await` — so by the time the code after `await sync.session.sync()` runs, all the context hooks (including `local`, declared after the effect registration in source order) are already initialized. Understanding that made the fix feel safe rather than uncertain.

I also learned the difference between session-level state (`result.data.agent` — set by `AgentSwitched` events) and message-level state (`lastUserMessage().agent` — the agent field on the last user message object). These usually agree, but can diverge if the user switched agents after their last message. `result.data.agent` is more authoritative for this use case.

### Challenges Overcome

Finding this issue required understanding that the fix was a *missing* line, not a *wrong* line. There's no crash, no error message, just silent wrong state. The way to see it is to trace what the navigation `createEffect` restores and notice what's absent. That kind of gap-finding — reading code to see what should be there but isn't — is a different skill from reading code to find what's wrong.

The code review step surfaced two real issues I hadn't caught: the `--agent` guard and the stale-list risk. Both required understanding the codebase beyond just the three lines I was adding. The `--agent` guard came from reading how a parallel mechanism in `prompt/index.tsx` handles the same operation; the `bootstrapOk` flag came from tracing what `local.agent.set()` does when the name isn't in the list.

### What I'd Do Differently Next Time

I ran the four-gate oss-verify check correctly this time — issue timeline with `--paginate`, multiple keyword angles including a file-path search angle, file-touching PR diff inspection, and a live code check on the default branch. That's the lesson from contribution 1 applied directly.

The one thing I'd add for next time: test the fix manually *before* the code review, not after. I ran the review on untested code and then had to apply fixes, meaning the review was on an earlier version. Testing first would have caught the `--agent` guard issue through a concrete test case (`opencode --agent plan` + navigating sessions) rather than static analysis.

---

## Resources Used

- [anomalyco/opencode#31862](https://github.com/anomalyco/opencode/issues/31862) — the issue
- [SolidJS documentation — createEffect](https://www.solidjs.com/docs/latest#createeffect) — understanding reactive effect timing
- `packages/tui/src/context/local.tsx` — `local.agent` store implementation
- `packages/core/src/session/projector.ts` — how `AgentSwitched` events write `session.agent`
- `packages/opencode/src/session/prompt.ts` — when `AgentSwitched` is published
- `packages/tui/src/component/prompt/index.tsx:321` — reference for the `!args.agent` guard pattern
