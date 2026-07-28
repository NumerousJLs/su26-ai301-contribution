# Contribution 4: Open Project dialog search excludes projects outside the five most recent

**Contribution Number:** 4  
**Student:** Joshua Liu  
**Issue:** [anomalyco/opencode#39142](https://github.com/anomalyco/opencode/issues/39142)  
**Related issue:** [#7111](https://github.com/anomalyco/opencode/issues/7111) ("Recent projects not found in 'Open Project' dialog search") — same bug, reported earlier and closed as stale on 2026-05-22 with a request to reopen if still relevant. It was.  
**Fork:** [github.com/NumerousJLs/opencode](https://github.com/NumerousJLs/opencode)  
**Branch:** `project-picker-search` (rebased onto `dev` @ `8021dbd80f`, v1.18.8)  
**Pull Request:** *(pending — see Pull Request section)*  
**Status:** Phase III Complete — implementation and tests done, PR not yet opened

---

## Why I Chose This Issue

I wanted a bug where the user-visible symptom and the actual defect look like completely different problems. This one qualifies: the dialog reports "No folders found" for a project that is open, indexed, and visible in the sidebar at that exact moment. Nothing about that message points at the real cause, which is an array being truncated one step before the search runs.

It also had the property I most wanted to practice after Contribution 3 — a fix that is verifiable by reading the code path end to end rather than by guessing. The chain is short enough to hold in your head (dialog builds rows, list filters rows, filter only sees what it was handed) but long enough that you have to actually open the list component to know where the filtering happens. I had assumed the dialog did its own filtering; it does not. Checking that assumption is what confirmed the diagnosis.

The scope was honest, too. It is one array operation in the wrong place. That is small enough to finish properly, with a real regression test, rather than shipping something half-verified.

What "fixed" looks like in concrete terms: with more than five projects open, typing part of the name of the sixth returns that project as a selectable row. The dialog with an empty search box still shows exactly five recents, unchanged.

---

## Understanding the Issue

### Problem Description

opencode Desktop's Open Project dialog (Ctrl+O, or the `project.open` command) shows a "Recent projects" section listing the five most recently active projects, followed by a directory browser. Typing in the search box is supposed to filter across everything the dialog knows about.

It does not. Only those five projects are ever searchable by name. A sixth project — already open, listed in the sidebar, with sessions in it — returns "No folders found" when you search for it by name. Typing its full absolute path does find it, because absolute paths are resolved by the directory browser rather than by the recent-projects list.

That split is what makes the bug confusing to report. From the user's side it looks like the search is inconsistent: the same project is findable one way and invisible the other.

### Root Cause

Three pieces have to be read together.

`recentProjects()` in `packages/app/src/components/dialog-select-directory.tsx` sorted every known project by last session activity and then truncated to five while building its rows (line 105 on `dev`):

```ts
return projects
  .map((project, index) => ({ project, at: byProject.get(project.worktree) ?? 0, index }))
  .sort((a, b) => b.at - a.at || a.index - b.index)
  .slice(0, 5)                                    // ← truncation happens here
  .map(({ project }) => { /* build the Row */ })
```

Those rows are returned from `items(value)` (line 116), which is what the `List` component is given:

```ts
const items = async (value: string) => {
  const results = await directories(value)
  const directoryRows = results.map((absolute) => toRow(absolute, home(), "folders"))
  return uniqueRows([...recentProjects(), ...directoryRows])
}
```

And `List` runs its fuzzy filter over exactly that returned array — nothing else. From `packages/ui/src/hooks/use-filtered-list.tsx`:

```ts
items: typeof props.items === "function" ? props.items(store.filter) : props.items,
// ...
: fuzzysort.go(needle, filterable, { keys: props.filterKeys! }).map((x) => x.obj)
```

So the truncation happens *before* the only filtering step in the pipeline. Projects six and beyond are not ranked low by the search — they are never candidates for it. No amount of typing can surface them.

The `.slice(0, 5)` is not wrong in intent. The idle dialog genuinely should show five recents rather than every project ever opened. It is in the wrong place: it encodes a display limit inside a function whose output is also used as the search corpus.

### Expected vs. Actual Behavior

| | Empty search box | Typing "sixth-project" |
|---|---|---|
| **Expected** | 5 recent projects | The sixth project appears |
| **Actual** | 5 recent projects ✓ | "No folders found" ✗ |
| **Actual (full path)** | — | Found, via directory browser |

### Affected Components

- `packages/app/src/components/dialog-select-directory.tsx` — `recentProjects()` (the truncation) and `items()` (where the query is available)
- `packages/ui/src/hooks/use-filtered-list.tsx` — the fuzzysort filter that only sees what `items()` returns. Not modified; needed to confirm the diagnosis.
- `packages/app/src/components/directory-picker-domain.ts` — the pure-logic sibling module where the limit helper now lives

### When This Broke

`git log -S ".slice(0, 5)"` traces the line to `1f108bc401`, "feat(app): recent projects section in command pallette" ([PR #15270](https://github.com/anomalyco/opencode/pull/15270)), merged 2026-02-27. The truncation was present in the feature's first commit — this was never a regression, it shipped this way. Issue #7111 reported the symptom and was closed as stale on 2026-05-22 without the cause being found.

---

## Reproduction Process

### Environment Setup

Standard `bun install` from the repo root. Bun is the runtime, not Node.

The one real friction is typechecking. `packages/app` reports **319 errors on a clean checkout of `dev`**, almost all `TS7006: implicitly has an 'any' type` cascading from `Cannot find module '@opencode-ai/client/promise'`. That subpath isn't in the client package's `exports` map, so it doesn't resolve in a plain `bun install` checkout. Absolute error counts are therefore meaningless here; only the delta against a baseline means anything. Method is in Testing Strategy below.

Root `AGENTS.md` also mandates `bun typecheck` from the package directory rather than calling `tsc` directly. My first pass used `bunx tsc --noEmit`, which is the wrong command for this repo even though it produces similar output.

### Steps to Reproduce

1. Open opencode Desktop and open more than five projects, so the sidebar lists at least six.
2. Confirm one project is visible in the sidebar but *not* in the five-item "Recent projects" section of the Open Project dialog. Sorting is by most recent session activity, so the easiest way is to open an older project last-but-one.
3. Press Ctrl+O (or run the `project.open` command) to open the dialog.
4. Type part of that older project's name.
5. Observe "No folders found".
6. Clear the box and type the project's full absolute path instead. It is found and opens normally.

Step 6 is the one that separates this from "the project isn't registered." It is registered. The search just can't see it.

### Reproduction Evidence

- **Truncation confirmed on `dev`:** `git show upstream/dev:packages/app/src/components/dialog-select-directory.tsx | grep -n "slice(0, 5)"` → line 105, inside `recentProjects()`.
- **Filter scope confirmed:** `packages/ui/src/hooks/use-filtered-list.tsx:29` calls `props.items(store.filter)` and line 45 runs `fuzzysort.go` over that result only. There is no second data source.
- **Not a duplicate of the v2 dialog:** `dialog-select-directory-v2.tsx` also contains `.slice(0, 5)`, but it truncates *directory search results*, not the recent-projects list. Different concern, not affected.
- **Independent report:** #7111 describes the same behavior from the user side, closed stale before anyone traced it.

---

## Solution Approach

### Analysis

The limit itself is correct behavior; its position in the pipeline is the defect. So the fix is to move it rather than remove it: build rows for every project, and apply the cap only at the moment the query is known to be empty.

`items(value)` already receives the query string, so no new plumbing is needed. The whole change is deciding how many rows to spread there instead of in the memo.

I checked whether anyone else was already on this. Four gates:

1. **Issue timeline** — `gh api repos/anomalyco/opencode/issues/39142/timeline` returned no cross-references. No linked PR.
2. **Keyword PR search** — `gh search prs --repo anomalyco/opencode "dialog-select-directory"` and `"recent projects search"` surfaced nothing covering this.
3. **Open PRs touching the file** — two exist: [#34894](https://github.com/anomalyco/opencode/pull/34894) ("reslove issue of not showing file and dirs list") and [#38345](https://github.com/anomalyco/opencode/pull/38345) ("Improve keyboard navigation and add accept button in open project dialog"). I diffed both against `slice(0, 5)` and `recentProjects` — neither touches this logic. Merge-conflict risk in the same file, but not a duplicate fix.
4. **Bug still present on `dev`** — confirmed above at line 105.

All four pass.

### Implementation Plan (UMPIRE)

**Understand:** The dialog truncates its recent-projects list to five before handing it to `List`, and `List`'s fuzzysort filter only ever sees that handed-in array. Projects outside the top five are therefore not search candidates at all. The user-visible effect is "No folders found" for a project that is demonstrably open.

**Match:** The same file family already solves "logic that depends on the current query, and needs a test" by extracting a small pure function into `directory-picker-domain.ts`. `currentPickerSuggestions(result, query)` at line 116 is the closest analogue — it is three lines, query-dependent, exported purely so `directory-picker-domain.test.ts` can cover it. The sibling `dialog-select-model-search.ts` / `.test.ts` pair follows the same convention for the model dialog. Following this pattern gives the fix a test without mounting a SolidJS component that needs `sync`, `serverCtx`, and a live server connection.

**Plan:**
1. Delete `.slice(0, 5)` from `recentProjects()` so the memo returns rows for every known project.
2. Add `visibleRecentProjects(rows, query, limit)` to `directory-picker-domain.ts`, next to `currentPickerSuggestions`. Returns `rows.slice(0, limit)` for an empty query, `rows` otherwise.
3. Call it in `items(value)`, where the query is in scope. Extract the `5` to a named `RECENT_PROJECT_LIMIT` constant so the magic number is not just relocated.
4. Add a regression test to `directory-picker-domain.test.ts` covering both branches.

**Review:** Two commits (fix, then test), conventional-commit subjects, `git add` on named paths only. Prettier check on all three files. Typecheck compared against a stashed baseline so pre-existing errors aren't misread as new ones.

**Evaluate:** Empty query must still yield exactly five rows (no visual change to the idle dialog). Non-empty query must yield all rows. The test asserts both, and the second assertion fails against the old code, which returned five regardless.

### Edge Cases Considered

- **Fewer than five projects:** `slice(0, 5)` on a shorter array returns the whole array. Unchanged.
- **Duplicate rows:** `uniqueRows()` still runs after the spread, deduplicating by absolute path against the directory rows. Unchanged.
- **Group ordering:** rows keep `group: "recent"`, and `sortGroupsBy` still sorts recents above folders. Unchanged.
- **Large project counts:** the memo now builds rows for every project instead of five. The per-project work is a `toRow()` string build; the surrounding loop that computes last-activity timestamps already iterated every project and every session before my change, so the added cost is small relative to what was already there.
- **Whitespace-only query:** `items` receives the raw value and `!query` treats `""` as empty but `" "` as a query. This matches how `useFilteredList` decides whether to filter at all (`if (!needle) return x`), so the two stay consistent.

---

## Testing Strategy

### Unit Tests

Added to `packages/app/src/components/directory-picker-domain.test.ts` (commit `46204ee472`):

```ts
test("keeps every recent project searchable but caps the idle list", () => {
  const projects = ["one", "two", "three", "four", "five", "six", "seven"]
  expect(visibleRecentProjects(projects, "", 5)).toEqual(["one", "two", "three", "four", "five"])
  expect(visibleRecentProjects(projects, "sev", 5)).toEqual(projects)
})
```

Both halves matter. The first pins the behavior I did *not* want to change — the idle dialog still shows five. The second is the regression: against the old code it returns five items and fails, because the old code sliced unconditionally.

The test uses plain strings rather than `Row` objects on purpose. `visibleRecentProjects` is generic over the element type and does not inspect elements, so constructing real rows would add setup that tests nothing.

Naming and placement follow the file's existing convention: flat `test("...")` calls with sentence-style descriptions, placed next to the `currentPickerSuggestions` test it mirrors.

### Test Output

```
$ cd packages/app && bun test src/components/directory-picker-domain.test.ts
bun test v1.3.14 (0d9b296a)

 21 pass
 0 fail
 75 expect() calls
Ran 21 tests across 1 file. [357.00ms]
```

20 of these existed before; mine is the 21st. The sibling suite also still passes:

```
$ bun test src/components/directory-picker.test.ts
 1 pass
 0 fail
 3 expect() calls
```

### Manual Verification

- `prettier --check` on all three changed files → "All matched files use Prettier code style!"
- Read `useFilteredList` line by line to confirm `items()`'s return value is the sole filter input — this is the step that actually proved the diagnosis rather than assuming it.

**Typecheck, done as a delta.** Because `packages/app` has 319 pre-existing errors, I compared full sorted output between my branch and the same three files reverted to `dev`:

```bash
bun typecheck 2>&1 | sort > /tmp/tc-branch.txt
git checkout upstream/dev -- <the three files>
bun typecheck 2>&1 | sort > /tmp/tc-dev.txt
git checkout HEAD -- <the three files>
diff /tmp/tc-dev.txt /tmp/tc-branch.txt
```

319 errors on both sides. Every line in the diff is the same error text and column at a shifted line number, caused by the lines I added. Nothing introduced.

My first attempt at this baseline used `git stash`, which silently did nothing — my changes were already committed, so both runs measured the same tree and reported a meaningless "identical". Reverting the specific files is the only version of this that actually works once you've committed.

**What I did not verify.** I did not exercise this in the running Desktop app. `packages/app/AGENTS.md` documents how (`bun run --conditions=browser ./src/index.ts serve --port 4096` from `packages/opencode`, `bun dev -- --port 4444` from `packages/app`), but reproducing requires six or more real projects with session history, and creating those would write into the actual local opencode state. I verified by reading the path end to end and unit-testing the extracted limit instead. That is a real gap and I've stated it in the PR rather than implying coverage I don't have.

---

## Implementation Notes

### What I built

**Commit 1 — `04c9c58d26`:** `fix(app): search every known project in the open project dialog`

`packages/app/src/components/dialog-select-directory.tsx`:

```diff
       .sort((a, b) => b.at - a.at || a.index - b.index)
-      .slice(0, 5)
       .map(({ project }) => {
```

```diff
   const items = async (value: string) => {
     const results = await directories(value)
     const directoryRows = results.map((absolute) => toRow(absolute, home(), "folders"))
-    return uniqueRows([...recentProjects(), ...directoryRows])
+    const recent = visibleRecentProjects(recentProjects(), value, RECENT_PROJECT_LIMIT)
+    return uniqueRows([...recent, ...directoryRows])
   }
```

Plus `const RECENT_PROJECT_LIMIT = 5` and the added import.

`packages/app/src/components/directory-picker-domain.ts`:

```diff
+export function visibleRecentProjects<T>(rows: readonly T[], query: string, limit: number): readonly T[] {
+  if (!query) return rows.slice(0, limit)
+  return rows
+}
```

**Commit 2 — `46204ee472`:** `test(app): pin regression for recent project search candidates`

**Net change:** 3 files, 22 insertions(+), 3 deletions(-).

```
packages/app/src/components/dialog-select-directory.tsx     | 13 ++++++++++---
packages/app/src/components/directory-picker-domain.test.ts |  7 +++++++
packages/app/src/components/directory-picker-domain.ts      |  5 +++++
```

### Conventions Audit

Before finalizing, I re-read the project's own rules and audited this branch against them rather than assuming my habits from earlier contributions still applied. Four things were wrong.

**Branch name.** Root `AGENTS.md` says: "Use a short branch name of at most three words, separated by hyphens. Do not use slashes or type prefixes such as `feat/` or `fix/`." My branch was `fix/project-picker-search-all-projects` — a slash, a `fix/` prefix, and five words. I checked it against fifteen recently merged PRs and every one matched the documented form (`sdk-session-recovery`, `mcp-debug-port`, `windows-tree-dev`, `search-timeout`). Renamed to `project-picker-search`. Contributions 1 through 5 all used the `fix/...` form, so this was a standing error in my process, not a one-off.

**Typecheck command.** Root `AGENTS.md`: "Always run `bun typecheck` from package directories, never `tsc` directly." I had been running `bunx tsc --noEmit -p tsconfig.json`.

**A per-package `AGENTS.md` I hadn't read.** `packages/app/AGENTS.md` exists and I only found it by listing every `AGENTS.md` in the tree. This is precisely the mistake that sank Contribution 4's Bedrock fix — writing code before reading the local architecture notes. It sets priorities as stability, then simplicity, then performance; requires `createStore` over multiple `createSignal`; and documents the local dev commands I cite in the "what I did not verify" note.

**Commit contents.** A late style fix landed in the wrong commit because `git commit --amend -- <path>` adds a path to HEAD rather than to the earlier commit I meant. I rebuilt both commits with `git reset --soft` and re-staged per commit, so commit 1 is the two source files and commit 2 is only the test.

**A stale base that made the diff look enormous.** My final `git diff upstream/dev` came back showing version downgrades across roughly forty `package.json` files and `bun.lock`. I had branched at `c8487bac54`, and `dev` had since released v1.18.8, so diffing my older tree against the newer tip rendered every version bump as a reversion. My commits never touched those files, and GitHub would have diffed from the merge base rather than the tip, so the PR itself would have looked fine — but I would have been submitting against a stale base without knowing it. Rebasing onto `8021dbd80f` brought the diff back to the three files it should be. Rule: rebase onto the current default branch and re-read the full diff immediately before opening, not just the `--stat`.

One rule I deliberately did not follow to the letter. Root `AGENTS.md` says twice not to extract single-use helpers. `visibleRecentProjects` is single-use. The rule allows extraction when the helper "has a clear independent name that improves the caller," and I wanted evidence rather than an opinion, so I counted callsites for the existing exports in the module I was editing. `currentPickerSuggestions`, `pickerMode`, `nextSuggestionIndex`, and `advanceTreePreload` each have exactly two source references — the import and one call — and each is unit-tested. Single-use extraction into `directory-picker-domain.ts` is this module's established convention, so I matched it and said so in the PR. If a maintainer disagrees, inlining it is a two-line change.

I also dropped the explicit `: readonly T[]` return annotation to match the neighbors, since `AGENTS.md` prefers inference.

### Challenges Faced

**Verifying the diagnosis instead of accepting it.** The issue reporter named the root cause correctly and even proposed the fix. It would have been easy to apply it and move on. But their claim only holds if `List` filters the array it receives rather than fetching its own data — and I had no idea whether that was true. Opening `use-filtered-list.tsx` and finding `props.items(store.filter)` feeding straight into `fuzzysort.go` is what turned a plausible report into a confirmed one. If `List` had held its own unfiltered copy, the same patch would have done nothing.

**A typecheck that fails on a clean tree.** My first `tsc` run showed an error in the file I had just edited, which read as self-inflicted. Stashing and re-running showed the identical error one line earlier — it was pre-existing, shifted by my added line. I now diff typecheck output against a stashed baseline rather than reading absolute error counts, which is the only way to tell "I broke this" from "this was already broken" in a monorepo with unbuilt workspace deps.

**Deciding whether the helper was worth it.** A two-line ternary inline in `items()` would have fixed the bug. Extracting `visibleRecentProjects` is arguably more structure than a one-line behavior change deserves. What decided it was finding `currentPickerSuggestions` — three lines, exported, query-dependent, tested — already sitting in the exact module I would put it in. The maintainers had made this call before in this file, and matching an existing decision is worth more than my preference either way.

**Choosing this issue at all.** Two earlier candidates died in verification. #39194 (a WebFetch description promising an HTTPS upgrade that never happens) already had [PR #39195](https://github.com/anomalyco/opencode/pull/39195) open against it. #39165 (a SQLite `NOT NULL` crash after switching models) had no competing PR and a clean deterministic repro, and I spent real time tracing it through the event projector and the durable event sequence before concluding I could not identify the root cause with confidence without reproducing the crash. Submitting a guess at a fix for a crash I did not understand would have been worse than not submitting one. Documenting the dead ends is part of the work.

---

## Pull Request

**PR Link:** *(not yet opened)*

The branch is complete and verified locally. Both commits are in place, tests pass, the four pre-submission gates were re-run immediately before finalizing (issue timeline still shows no cross-references; the three open PRs touching this file — #34894, #38345, #37612 — have zero matches on `recentProjects`, `slice(0, 5)`, or `visibleRecentProjects`; the truncation is still on `dev` at line 105), and the branch has been audited against the project's `AGENTS.md` files. It has not been pushed because I review the diff and PR description myself before anything goes to the upstream repo. This section will be updated with the link, submission date, and any maintainer feedback once it is opened.

**Prepared PR description** (fills `.github/pull_request_template.md` verbatim — the repo auto-rejects PRs that don't follow it, and a compliance bot closes non-conforming PRs within about two hours):

> ### Issue for this PR
>
> Closes #39142
>
> ### Type of change
>
> - [x] Bug fix
> - [ ] New feature
> - [ ] Refactor / code improvement
> - [ ] Documentation
>
> ### What does this PR do?
>
> Typing a project's name into the Open Project dialog now finds it even when it isn't one of the five most recent.
>
> Before this change, `recentProjects()` in `dialog-select-directory.tsx` sorted every known project by last session activity and then called `.slice(0, 5)` while building its rows. Those rows are what `items()` returns, and `items()` is what `List` is given. `useFilteredList` calls `props.items(store.filter)` and runs `fuzzysort.go` over exactly that result, so the truncation was happening one step before the only filtering in the pipeline. Projects outside the top five weren't ranked low by the search, they were never candidates for it, which is why a project that was open and visible in the sidebar still came back as "No folders found". Typing the full absolute path worked, because that path is resolved by the directory browser instead of the recent-projects list, and that split is what makes the bug look like an inconsistent search rather than a truncated list. The `.slice(0, 5)` itself isn't wrong, the idle dialog should show five recents rather than every project ever opened; it was just sitting inside a function whose output doubles as the search corpus.
>
> This PR moves the cap to the point where the query is known. `recentProjects()` now returns rows for every project, and `items()` applies the limit through `visibleRecentProjects()` only when the search box is empty, so the idle dialog renders exactly as it did before and typing searches the whole set. The limit helper sits in `directory-picker-domain.ts` alongside `currentPickerSuggestions`, `pickerMode`, and `nextSuggestionIndex`, which are the existing single-use, query-dependent helpers in this module that are extracted so `directory-picker-domain.test.ts` can cover them. `.slice()` on a shorter array still returns the whole array, so nothing changes for users with fewer than five projects, and `uniqueRows()` still dedupes against the directory rows afterwards.
>
> The truncation traces to 1f108bc401 ("feat(app): recent projects section in command pallette", #15270), so this has been the behavior since the section was introduced rather than being a recent regression. #7111 reported the same symptom and was closed as stale.
>
> ### How did you verify your code works?
>
> Added a regression test to `packages/app/src/components/directory-picker-domain.test.ts` covering both branches. The empty-query case pins the behavior that shouldn't change, and the non-empty case fails against the old code, which returned five rows regardless of the query.
>
> ```
> $ cd packages/app && bun test src/components/directory-picker-domain.test.ts
> bun test v1.3.14 (0d9b296a)
>
>  21 pass
>  0 fail
>  75 expect() calls
> Ran 21 tests across 1 file. [290.00ms]
> ```
>
> The sibling suite still passes (`bun test src/components/directory-picker.test.ts` — 1 pass, 0 fail).
>
> `bun typecheck` from `packages/app` reports 319 errors both before and after this change. They come from `@opencode-ai/client/promise` not resolving in my checkout and the implicit-`any` cascade behind it. I diffed the full sorted output against the same command run with these three files reverted to `dev`: every difference is a line-number shift with identical error text and column, so this change introduces none of them. `prettier --check` is clean on all three files.
>
> What I did not do is exercise this in the running Desktop app, which would need six or more real projects with session history. I verified it by reading the path instead — `recentProjects()` to `items()` to `props.items(store.filter)` to `fuzzysort.go` in `use-filtered-list.tsx` — and by unit-testing the extracted limit. The step that actually confirmed the diagnosis was checking that `List` filters the array it is handed rather than holding its own copy; if it did the latter, this patch would do nothing.
>
> ### Screenshots / recordings
>
> No screenshot. This change is deliberately invisible in the state a screenshot would capture: with an empty search box the dialog still renders the same five recent projects. The difference only appears while typing, with more than five projects open, where a previously unfindable project becomes selectable.
>
> ### Checklist
>
> - [x] I have tested my changes locally
> - [x] I have not included unrelated changes in this PR

**Before/After:**

Before — the search corpus is truncated before filtering:

```ts
.slice(0, 5)                                  // only 5 rows ever exist
.map(({ project }) => /* Row */)
// ...
return uniqueRows([...recentProjects(), ...directoryRows])
// List fuzzy-filters these 5 rows. Project #6 is not a candidate.
```

After — the full corpus reaches the filter; the cap applies only when idle:

```ts
.map(({ project }) => /* Row */)              // rows for every project
// ...
const recent = visibleRecentProjects(recentProjects(), value, RECENT_PROJECT_LIMIT)
return uniqueRows([...recent, ...directoryRows])
// Empty query -> 5 rows, as before. Non-empty query -> all rows searchable.
```

**Acceptance criteria:**
- [x] Projects outside the five most recent are searchable by name
- [x] Idle dialog still shows exactly five recent projects — no visual change
- [x] Regression test added; fails against the pre-fix behavior
- [x] Existing tests pass (21 pass, 0 fail in the domain suite; 1 pass, 0 fail in the sibling suite)
- [x] No new typecheck errors (319 before and after; verified by diffing full sorted output against a `dev` baseline)
- [x] Prettier clean on all changed files
- [x] Diff scoped to the issue — no unrelated changes
- [x] Branch name follows `AGENTS.md` (three words, hyphens, no slash or type prefix)
- [x] Commit messages and PR title are conventional-commit form
- [x] PR body fills every section of the repo template
- [x] `Closes #39142` in the PR description
- [x] Four pre-submission gates re-run immediately before finalizing
- [ ] PR opened against `anomalyco/opencode:dev`
- [ ] Maintainer review

**Maintainer Feedback:** *(pending — will log feedback, my response, dates, and commit refs when received)*

---

## Learnings & Reflections

### Technical Skills Gained

The transferable idea here is that filtering has a corpus, and bugs hide in the gap between what you think the corpus is and what it actually is. `recentProjects()` served two roles that nobody had separated: "what to display when idle" and "what is searchable." Those happened to be the same array, so one `.slice()` silently governed both. The fix is not really about slicing — it is about noticing that a single value was answering two different questions and only one of them wanted a limit.

I also got better at a specific verification habit: when a bug report hands you the root cause, find the one assumption it rests on and check that instead of checking the fix. Here the assumption was "the list filters what it's given." That took one file to confirm and would have invalidated the entire patch if false. Checking the assumption is cheaper than checking the conclusion.

The typecheck-baseline habit generalizes past this repo. In any monorepo with unbuilt workspace deps, absolute error counts are noise. The signal is the delta against a stash.

### What I'd Do Differently Next Time

I would set a hard time budget on root-causing before committing to an issue. I spent a long stretch on #39165 — reading the event projector, the durable sequence allocation, the message updater — and came away without a confident root cause. That was not wasted in the sense that I learned the event-sourcing layer, but it was time I could not spend on a fix I could actually finish. A reasonable rule: if I cannot state the root cause in one sentence after an hour of tracing, the issue needs a reproduction I can run, and I should either build that or move on.

The other change is to check for competing PRs *before* reading any code. I found #39195 on #39194 only after I had already understood the bug. The gate checks are cheap and take a couple of minutes; running them first would have saved that entirely.

### Mistakes Register

Running list of every mistake made across these contributions, kept so they stop repeating. The first four cost a contribution each; the rest were caught during this one's audit.

**Fixed dead code (Contribution 4).** Sanitized Bedrock document names in `packages/llm/src/protocols/utils/bedrock-media.ts`, which sits behind `OPENCODE_EXPERIMENTAL_NATIVE_LLM` and never runs for Bedrock. Every Bedrock request goes through the AI SDK path in `ProviderTransform.message()`. PR #37535 fixed the live path and the work was wasted. Rule: trace the runtime path from entry point to wire and confirm the function is reachable for the failing case before writing anything.

**Trusted a single "no competing PR" search (Contribution 1).** `gh pr list --search "27133 in:body"` returned empty, so I opened PR #31992. A duplicate-detection bot then flagged PR #29784, filed two weeks earlier, same file, same approach. GitHub body-text search is unreliable; the issue timeline is authoritative. Rule: run all four gates, and re-run them right before opening.

**Did not read the per-package `AGENTS.md`.** Both the Bedrock miss and this contribution's near-miss trace to the same habit. `packages/app/AGENTS.md` existed the whole time and I only found it by listing every `AGENTS.md` in the tree. Rule: `git ls-tree -r upstream/dev --name-only | grep AGENTS.md` before touching a package.

**Wrong branch naming, five contributions running.** Root `AGENTS.md` forbids slashes and type prefixes and caps names at three words. Every branch through Contribution 5 used `fix/long-description-here`. Rule fixed in my notes; verified against fifteen merged PRs.

**Wrong typecheck command.** Used `bunx tsc --noEmit` when root `AGENTS.md` mandates `bun typecheck` from the package directory.

**Read absolute error counts as signal.** `packages/app` has 319 pre-existing errors. Any count is meaningless without a baseline diff.

**Built a baseline with `git stash` on committed work.** It stashed nothing, both runs measured the same tree, and the comparison reported a confident, meaningless "identical". A verification step that cannot fail is not verification. Rule: revert the specific files to `dev` instead, and sanity-check that the baseline actually differs from the branch.

**Used `git commit --amend -- <path>` to move a file between commits.** It adds the path to HEAD rather than the earlier commit, silently putting a source change inside the test commit. Rule: `git reset --soft <base>` and re-stage per commit.

**Let the branch base go stale.** Branched at `c8487bac54`; `dev` released v1.18.8 while I worked, so the final diff appeared to revert version bumps across forty-odd files. Rule: rebase onto the current default branch and read the full diff — not the `--stat` — immediately before opening.

**General ones worth keeping in view:** don't write PR bodies as dense bullet lists (the template warns AI-looking descriptions get closed, and a compliance bot closes non-conforming PRs within about two hours); don't run `bun test` from the repo root; don't claim `Closes #` unless the fix resolves the issue end to end; and don't let a fix quietly grow past the issue's scope, since "I have not included unrelated changes" is taken literally.

### Teachable Insight

Two of my four contributions so far were beaten or invalidated by work I could have detected up front — one by a PR filed hours earlier, one because I fixed a code path that was dead on the default runtime. In an actively developed repo, "is this bug real, and is it still mine to fix" is a research question that deserves as much rigor as the fix itself. The four-gate check (issue timeline → keyword PR search → open PRs touching the file → bug still on the default branch) takes about five minutes and has now saved me more time than any debugging technique I have picked up.

---

## Resources Used

- [anomalyco/opencode#39142](https://github.com/anomalyco/opencode/issues/39142) — the issue
- [anomalyco/opencode#7111](https://github.com/anomalyco/opencode/issues/7111) — earlier report of the same behavior, closed as stale
- [anomalyco/opencode#15270](https://github.com/anomalyco/opencode/pull/15270) — the PR that introduced the recent-projects section and the truncation
- `packages/app/src/components/dialog-select-directory.tsx` — the dialog changed
- `packages/app/src/components/directory-picker-domain.ts` — where the limit helper lives
- `packages/app/src/components/directory-picker-domain.test.ts` — the test file
- `packages/ui/src/hooks/use-filtered-list.tsx` — the fuzzysort filter that confirmed the diagnosis
- `packages/app/src/components/dialog-select-model-search.ts` — the extract-pure-logic-and-test-it convention I followed
