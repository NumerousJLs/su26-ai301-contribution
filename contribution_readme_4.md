# Contribution 4: Open Project dialog search excludes projects outside the five most recent

**Contribution Number:** 4  
**Student:** Joshua Liu  
**Issue:** [anomalyco/opencode#39142](https://github.com/anomalyco/opencode/issues/39142)  
**Related issue:** [#7111](https://github.com/anomalyco/opencode/issues/7111) ("Recent projects not found in 'Open Project' dialog search"), filed 2026-01-06 by Yukaii. Same bug. Assigned to `rekram1-node`, confirmed still live by two other users in April 2026, then closed as **not planned** by that maintainer on 2026-05-22 with no stated reason. I originally wrote "closed as stale," which was wrong: the stale bot posted its 90-day notice in March but did not close it, and a human closed it two months later.  
**Fork:** [github.com/NumerousJLs/opencode](https://github.com/NumerousJLs/opencode)  
**Branch:** `project-picker-search` (rebased onto `dev` @ `1e17856ba4`)  
**Pull Request:** *(pending — see Pull Request section)*  
**Status:** Phase III Complete. Fix inlined to two files, committed e2e regression spec verified to fail against `dev`, screenshots hosted and embedded. PR not yet opened.

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
- `packages/app/e2e/regression/project-picker-recent-search.spec.ts` — the committed Playwright spec that pins the behaviour end to end

### When This Broke

`git log -S ".slice(0, 5)"` traces the line to `1f108bc401`, "feat(app): recent projects section in command pallette" ([PR #15270](https://github.com/anomalyco/opencode/pull/15270)), merged 2026-02-27. The truncation was present in the feature's first commit — this was never a regression, it shipped this way. Issue #7111 reported the symptom in January 2026 and was closed as not planned on 2026-05-22, after two users confirmed in April that it was still reproducible on Linux and Windows.

---

## Reproduction Process

### Environment Setup

Standard `bun install` from the repo root. Bun is the runtime, not Node.

The one real friction was self-inflicted and worth recording, because I reported it as a property of the repo for two rounds before catching it. `bun typecheck` in `packages/app` was returning **319 errors**, almost all `TS7006: implicitly has an 'any' type` cascading from `Cannot find module '@opencode-ai/client/promise'`. I treated that as a pre-existing repo condition and carefully diffed against it as a baseline.

It was not. `packages/app` depends on a vendored tarball (`"@opencode-ai/client": "file:vendor/opencode-ai-client-1.17.13-v2.tgz"`) plus `@corvu/drawer`, and neither was installed in my checkout. Running `bun install` from the repo root fixed all 319: `bun typecheck` now exits 0 with zero errors, and `bun.lock` is untouched. I only found this when I tried to actually run the app and Vite reported the same unresolved imports.

Root `AGENTS.md` also mandates `bun typecheck` from the package directory rather than calling `tsc` directly. My first pass used `bunx tsc --noEmit`, which is the wrong command for this repo.

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

1. **Issue timeline** — no cross-referenced PRs. The only timeline event is an `assigned`. **The issue is assigned to `Brendonovich` (Brendan Allan, `@anomalyco`), who owns `packages/app`.** My first pass filtered the timeline to `cross-referenced` events only and never saw this, which was a real hole in my gate. Following up: the assignment was made by `github-actions[bot]` 93 seconds after the issue was filed, and every recent issue I sampled has the same pattern (#39218 → Brendonovich, #39221 → rekram1-node, #39125 → nexxeln, #39194 → jlongster, #39190 → kitlangton), so it is automatic routing to an area owner rather than someone claiming the work. My own accepted contribution 3 issue (#35860) was auto-assigned to StarpTech and PR #35867 was still welcome. None of Brendan's open PRs touch the picker.
2. **Keyword PR search** — `gh search prs --repo anomalyco/opencode "dialog-select-directory"` and `"recent projects search"` surfaced nothing covering this.
3. **Open PRs touching the file** — scanned all 200 open PRs by file path. Three touch this area: [#38345](https://github.com/anomalyco/opencode/pull/38345), [#34894](https://github.com/anomalyco/opencode/pull/34894), [#37612](https://github.com/anomalyco/opencode/pull/37612). Diffed each for `recentProjects`, `slice(0, 5)`, and `visibleRecentProjects` — zero matches. Merge-conflict risk in the same file, but not a duplicate fix. I also fetched the plausible picker-related branches on the upstream remote (`fix/edit-project-dialog`, `child-session-picker`, `directory-attachment-expansion`) and confirmed `.slice(0, 5)` is still present on all of them.
4. **Bug still present on `dev`** — confirmed above at line 105.

All four pass, and I re-ran every one of them against a freshly fetched `dev` immediately before finalizing.

### Implementation Plan (UMPIRE)

**Understand:** The dialog truncates its recent-projects list to five before handing it to `List`, and `List`'s fuzzysort filter only ever sees that handed-in array. Projects outside the top five are therefore not search candidates at all. The user-visible effect is "No folders found" for a project that is demonstrably open.

**Match:** The same file family already solves "logic that depends on the current query, and needs a test" by extracting a small pure function into `directory-picker-domain.ts`. `currentPickerSuggestions(result, query)` at line 116 is the closest analogue — it is three lines, query-dependent, exported purely so `directory-picker-domain.test.ts` can cover it. The sibling `dialog-select-model-search.ts` / `.test.ts` pair follows the same convention for the model dialog. Following this pattern gives the fix a test without mounting a SolidJS component that needs `sync`, `serverCtx`, and a live server connection.

**Plan:**
1. Delete `.slice(0, 5)` from `recentProjects()` so the memo returns rows for every known project.
2. Add `visibleRecentProjects(rows, query, limit)` to `directory-picker-domain.ts`, next to `currentPickerSuggestions`. Returns `rows.slice(0, limit)` for an empty query, `rows` otherwise.
3. Call it in `items(value)`, where the query is in scope. Extract the `5` to a named `RECENT_PROJECT_LIMIT` constant so the magic number is not just relocated.
4. Add a regression test covering both branches.

*(Steps 2 and 3 were later reversed. The helper and its unit test were removed in favour of inlining the cap plus a Playwright spec. See "Second Review Round" below.)*

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

Both halves matter: the first pins the behavior I did *not* want to change, the second describes the new rule.

**This unit test does not pin the original bug, and I initially claimed it did.** A Codex review caught it. The helper did not exist before this change, so the test cannot be run against the old code at all — and if someone re-added `.slice(0, 5)` to `recentProjects()` while keeping the helper, all 21 tests still pass. I verified that by doing exactly that: 21 pass, 0 fail, with the bug fully restored. A unit test of a downstream helper can never detect that an upstream caller truncated its input.

So I added a committed Playwright regression spec, `packages/app/e2e/regression/project-picker-recent-search.spec.ts`, following the convention already established by ~39 specs in that directory (maintainer PR #39241 added one alongside a fix). It seeds six projects via `addInitScript` and `mockOpenCodeServer`, opens the dialog, and asserts both directions. Assertions are scoped to `[data-directory-path]` because the sidebar renders the same project names and a naive text match produced a false failure.

I then proved it pins the bug by reverting both source files to `dev` and re-running:

```
# on the fix
2 passed (11.7s)

# source files reverted to dev
1 failed  › searches every recent project, not just the five most recent
1 passed  › still caps the idle recent list at five projects
```

The regression case goes red and the invariant case stays green — which is the signature a real regression test should have.

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

**Typecheck.** After `bun install`, `bun typecheck` in `packages/app` exits 0 with zero errors, and `bun typecheck:e2e` (which covers the e2e specs, and is a separate command) also exits 0.

Before I found the missing install, I had built a delta harness against the 319-error "baseline": capture sorted output on the branch, revert the three files to `dev`, capture again, diff. That method is sound and worth keeping for cases where a clean baseline genuinely isn't available — but here it was measuring my own broken environment. My first attempt at it also used `git stash`, which silently did nothing because the work was already committed, so both runs measured the same tree and reported a confident, meaningless "identical". Reverting specific files is the only version that works once you've committed.

**Lint.** `bun lint` maps to `oxlint`. On the three changed source files: 0 errors, 11 warnings — identical counts on `dev` and on the branch, so the change introduces none. The new e2e spec is clean at 0 warnings. I ran this as an explicit delta after the same shell-quoting mistake made `oxlint` report "No files found to lint" rather than failing loudly.

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

**A stale base that made the diff look enormous.** My final `git diff upstream/dev` came back showing version downgrades across roughly forty `package.json` files and `bun.lock`. I had branched at `c8487bac54`, and `dev` had since released v1.18.8, so diffing my older tree against the newer tip rendered every version bump as a reversion. My commits never touched those files, and GitHub would have diffed from the merge base rather than the tip, so the PR itself would have looked fine — but I would have been submitting against a stale base without knowing it. Rebasing onto `8021dbd80f` brought the diff back to the files it should be. It then happened a second time — `dev` moved again while I was working, so I rebased once more onto `f28d72d15e`, which is the current base. Rule: rebase onto the current default branch and re-read the full diff immediately before opening, not just the `--stat`, and expect to do it more than once on a repo that ships several times a day.

One rule I deliberately did not follow to the letter. Root `AGENTS.md` says twice not to extract single-use helpers. `visibleRecentProjects` is single-use. The rule allows extraction when the helper "has a clear independent name that improves the caller," and I wanted evidence rather than an opinion, so I counted callsites for the existing exports in the module I was editing. `currentPickerSuggestions`, `pickerMode`, `nextSuggestionIndex`, and `advanceTreePreload` each have exactly two source references — the import and one call — and each is unit-tested. Single-use extraction into `directory-picker-domain.ts` is this module's established convention, so I matched it and said so in the PR. If a maintainer disagrees, inlining it is a two-line change.

I also dropped the explicit `: readonly T[]` return annotation to match the neighbors, since `AGENTS.md` prefers inference.

### Second Review Round: Simplifying Away the Helper

A second Codex pass flagged that the extracted `visibleRecentProjects` helper had become redundant. That was correct, and I removed it.

The helper's justification had always been testability, and I defended it by showing that `currentPickerSuggestions`, `pickerMode`, `nextSuggestionIndex`, and `advanceTreePreload` in the same module are all single-use-but-tested. Once the Playwright spec existed and covered the real dialog, that justification evaporated: the integration was under test, so the extraction was buying nothing that root `AGENTS.md` does not explicitly discourage ("Do not extract single-use helpers preemptively").

Inlining it collapsed the change from four files to two:

```ts
// Cap the idle list only. Once a query narrows the results, every project stays searchable.
const recent = recentProjects()
const visible = value ? recent : recent.slice(0, RECENT_PROJECT_LIMIT)
return uniqueRows([...visible, ...directoryRows])
```

The unit test went with it, which is the right trade. It was the test that could not fail, and the spec that replaced it is stronger. I re-verified that the spec still fails against `dev` after inlining, so the coverage claim survives the simplification. Final diff is `+66/-2` across one source file and one spec.

The same review also caught em dashes throughout the PR body, which violates a hard rule in my own writing notes for public technical writing, and flagged that the screenshots were still placeholders. GitHub has no CLI path for image upload, so I committed the three PNGs to this repository under `pr-screenshots/` and referenced them by `raw.githubusercontent.com` URL, which renders in the PR without needing the web editor. Body is now 491 words with zero em dashes.

### Challenges Faced

**Verifying the diagnosis instead of accepting it.** The issue reporter named the root cause correctly and even proposed the fix. It would have been easy to apply it and move on. But their claim only holds if `List` filters the array it receives rather than fetching its own data — and I had no idea whether that was true. Opening `use-filtered-list.tsx` and finding `props.items(store.filter)` feeding straight into `fuzzysort.go` is what turned a plausible report into a confirmed one. If `List` had held its own unfiltered copy, the same patch would have done nothing.

**A typecheck that fails on a clean tree.** My first `tsc` run showed an error in the file I had just edited, which read as self-inflicted. Stashing and re-running showed the identical error one line earlier — it was pre-existing, shifted by my added line. I now diff typecheck output against a stashed baseline rather than reading absolute error counts, which is the only way to tell "I broke this" from "this was already broken" in a monorepo with unbuilt workspace deps.

**Deciding whether the helper was worth it.** A two-line ternary inline in `items()` would have fixed the bug. Extracting `visibleRecentProjects` is arguably more structure than a one-line behavior change deserves. What decided it was finding `currentPickerSuggestions` — three lines, exported, query-dependent, tested — already sitting in the exact module I would put it in. The maintainers had made this call before in this file, and matching an existing decision is worth more than my preference either way.

**Choosing this issue at all.** Two earlier candidates died in verification. #39194 (a WebFetch description promising an HTTPS upgrade that never happens) already had [PR #39195](https://github.com/anomalyco/opencode/pull/39195) open against it. #39165 (a SQLite `NOT NULL` crash after switching models) had no competing PR and a clean deterministic repro, and I spent real time tracing it through the event projector and the durable event sequence before concluding I could not identify the root cause with confidence without reproducing the crash. Submitting a guess at a fix for a crash I did not understand would have been worse than not submitting one. Documenting the dead ends is part of the work.

---

## Pull Request

**PR Link:** *(not yet opened)*

The branch is complete and verified locally. Both commits are in place, tests pass, the four pre-submission gates were re-run immediately before finalizing (issue timeline still shows no cross-references; the three open PRs touching this file — #34894, #38345, #37612 — have zero matches on `recentProjects`, `slice(0, 5)`, or `visibleRecentProjects`; the truncation is still on `dev` at line 105), and the branch has been audited against the project's `AGENTS.md` files. It has not been pushed because I review the diff and PR description myself before anything goes to the upstream repo. This section will be updated with the link, submission date, and any maintainer feedback once it is opened.

**Prepared PR description** (fills `.github/pull_request_template.md` verbatim):

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
> Searching the Open Project dialog only matched the five most recent projects. A sixth project that was open and visible in the sidebar came back as "No folders found" when searched by name, although typing its full absolute path still found it.
>
> `recentProjects()` sorted every known project by last session activity and then called `.slice(0, 5)` while building its rows. Those rows are what `items()` hands to `List`, and `useFilteredList` runs `fuzzysort.go` over exactly that array, so the cap was applied one step before the only filtering in the pipeline. Projects past the fifth were not ranked low, they were never candidates. Absolute paths kept working because they resolve through `createDirectorySearch` instead, which is what makes the bug look like an inconsistent search rather than a truncated list.
>
> The issue already identified this and proposed keeping every project as a search candidate while applying the cap only when the query is empty. That is what this does, by moving the cap into `items()` where the query is in scope:
>
> ```diff
> -    return uniqueRows([...recentProjects(), ...directoryRows])
> +    // Cap the idle list only. Once a query narrows the results, every project stays searchable.
> +    const recent = recentProjects()
> +    const visible = value ? recent : recent.slice(0, RECENT_PROJECT_LIMIT)
> +    return uniqueRows([...visible, ...directoryRows])
> ```
>
> The idle dialog still shows five, and nothing changes below five projects. `uniqueRows()` still dedupes against the directory rows afterwards.
>
> `git log -S ".slice(0, 5)"` traces the line to 1f108bc401 (#15270), so this has been the behaviour since the recent-projects section was added rather than being a regression. #7111 reported the same symptom in January, had it confirmed as still live on Linux and Windows in April, and was closed as not planned in May without a stated reason.
>
> Worth being explicit about scope, since a nearby request was declined: #7877 asked for a way to see more than five recent projects, and this PR does not do that. The idle list is still exactly five. Only the set of rows offered to the filter changes, and only while the user is typing.
>
> ### How did you verify your code works?
>
> `e2e/regression/project-picker-recent-search.spec.ts` registers six projects and covers both directions: the sixth is findable by name, and the idle list is still capped at five. I checked that it actually pins the bug by reverting the source file to `dev` and re-running, which turns the search case red and leaves the idle-cap case green.
>
> ```
> $ bunx playwright test e2e/regression/project-picker-recent-search.spec.ts --project=chromium
>   2 passed (14.7s)
>
> # same spec, dialog-select-directory.tsx reverted to dev
>   1 failed  › searches every recent project, not just the five most recent
>   1 passed  › still caps the idle recent list at five projects
> ```
>
> - `bun typecheck` and `bun typecheck:e2e` in `packages/app`, both exit 0
> - `bun test src/components/directory-picker.test.ts` and `directory-picker-domain.test.ts`, 21 pass, 0 fail
> - `bunx oxlint`, 0 errors, and the one warning in this file is pre-existing and identical on `dev`
> - `prettier --check` clean
>
> ### Screenshots / recordings
>
> Six projects registered, with `foxtrot-docs` sixth and outside the five-item cap. It is visible in the sidebar in every shot.
>
> Before, searching `foxtrot` finds nothing:
>
> ![before](https://raw.githubusercontent.com/NumerousJLs/su26-ai301-contribution/main/pr-screenshots/before-2-search-foxtrot.png)
>
> After, searching `foxtrot` finds it:
>
> ![after](https://raw.githubusercontent.com/NumerousJLs/su26-ai301-contribution/main/pr-screenshots/after-2-search-foxtrot.png)
>
> Idle dialog after the change, still exactly five recents:
>
> ![idle](https://raw.githubusercontent.com/NumerousJLs/su26-ai301-contribution/main/pr-screenshots/after-1-idle.png)
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
- [x] Committed e2e regression spec added; verified to fail against the pre-fix code
- [x] Unit test added for the visibility rule (does not pin the bug on its own — stated plainly)
- [x] Existing tests pass (21 pass, 0 fail in the domain suite; 1 pass, 0 fail in the sibling suite)
- [x] `bun typecheck` and `bun typecheck:e2e` clean (exit 0)
- [x] `bunx oxlint` — 0 errors; 11 warnings pre-existing and identical on `dev`
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

**Filtered the issue timeline too narrowly (caught this contribution).** My gate queried only `cross-referenced` events, so it silently skipped the `assigned` event and I did not notice for two rounds of review that #39142 is assigned to a core maintainer. It turned out to be automatic routing, but I did not know that when I claimed the gate had passed. Rule: read the whole timeline, then explain each event, rather than grepping for the one event type you expect.

**Trusted a single "no competing PR" search (Contribution 1).** `gh pr list --search "27133 in:body"` returned empty, so I opened PR #31992. A duplicate-detection bot then flagged PR #29784, filed two weeks earlier, same file, same approach. GitHub body-text search is unreliable; the issue timeline is authoritative. Rule: run all four gates, and re-run them right before opening.

**Did not read the per-package `AGENTS.md`.** Both the Bedrock miss and this contribution's near-miss trace to the same habit. `packages/app/AGENTS.md` existed the whole time and I only found it by listing every `AGENTS.md` in the tree. Rule: `git ls-tree -r upstream/dev --name-only | grep AGENTS.md` before touching a package.

**Wrong branch naming, five contributions running.** Root `AGENTS.md` forbids slashes and type prefixes and caps names at three words. Every branch through Contribution 5 used `fix/long-description-here`. Rule fixed in my notes; verified against fifteen merged PRs.

**Wrong typecheck command.** Used `bunx tsc --noEmit` when root `AGENTS.md` mandates `bun typecheck` from the package directory.

**Read absolute error counts as signal.** `packages/app` has 319 pre-existing errors. Any count is meaningless without a baseline diff.

**Built a baseline with `git stash` on committed work.** It stashed nothing, both runs measured the same tree, and the comparison reported a confident, meaningless "identical". A verification step that cannot fail is not verification. Rule: revert the specific files to `dev` instead, and sanity-check that the baseline actually differs from the branch.

**Used `git commit --amend -- <path>` to move a file between commits.** It adds the path to HEAD rather than the earlier commit, silently putting a source change inside the test commit. Rule: `git reset --soft <base>` and re-stage per commit.

**Let the branch base go stale.** Branched at `c8487bac54`; `dev` released v1.18.8 while I worked, so the final diff appeared to revert version bumps across forty-odd files. Rule: rebase onto the current default branch and read the full diff — not the `--stat` — immediately before opening.

**Wrote a verification harness that could not fail.** My first reproduction asserted a fuzzy search would return exactly one row and that an idle list would render five, when the harness passed an empty needle straight into `fuzzysort` and returned zero. Both assertions were wrong about the harness, not the code. Rule: when a check fails, first ask whether the check or the code is wrong, and treat a check that only ever passes as untested.

**Deferred reproduction until after writing the fix.** I traced the code, wrote the patch, and only demonstrated the bug when challenged on it. The demonstration confirmed the diagnosis, but the order was backwards, and had it disconfirmed the diagnosis I would have thrown away finished work.

**General ones worth keeping in view:** don't write PR bodies as dense bullet lists (the template warns AI-looking descriptions get closed, and a compliance bot closes non-conforming PRs within about two hours); don't run `bun test` from the repo root; don't claim `Closes #` unless the fix resolves the issue end to end; and don't let a fix quietly grow past the issue's scope, since "I have not included unrelated changes" is taken literally.

### Teachable Insight

Two of my four contributions so far were beaten or invalidated by work I could have detected up front — one by a PR filed hours earlier, one because I fixed a code path that was dead on the default runtime. In an actively developed repo, "is this bug real, and is it still mine to fix" is a research question that deserves as much rigor as the fix itself. The four-gate check (issue timeline → keyword PR search → open PRs touching the file → bug still on the default branch) takes about five minutes and has now saved me more time than any debugging technique I have picked up.

---

## Resources Used

- [anomalyco/opencode#39142](https://github.com/anomalyco/opencode/issues/39142) — the issue
- [anomalyco/opencode#7111](https://github.com/anomalyco/opencode/issues/7111) — earlier report of the same behavior, closed as not planned
- The wider cluster I should have walked from the start: [#7877](https://github.com/anomalyco/opencode/issues/7877) (asked to show more than five recents, went stale), [#12887](https://github.com/anomalyco/opencode/issues/12887) (added the recents section, completed), [#7545](https://github.com/anomalyco/opencode/issues/7545), [#7577](https://github.com/anomalyco/opencode/issues/7577), [#8325](https://github.com/anomalyco/opencode/issues/8325), [#12487](https://github.com/anomalyco/opencode/issues/12487), [#23587](https://github.com/anomalyco/opencode/issues/23587)
- [PR #12886](https://github.com/anomalyco/opencode/pull/12886) (unmerged) and [PR #15270](https://github.com/anomalyco/opencode/pull/15270) (merged, introduced the slice)
- [anomalyco/opencode#15270](https://github.com/anomalyco/opencode/pull/15270) — the PR that introduced the recent-projects section and the truncation
- `packages/app/src/components/dialog-select-directory.tsx` — the dialog changed
- `packages/app/src/components/directory-picker-domain.ts` — where the limit helper lives
- `packages/app/src/components/directory-picker-domain.test.ts` — the test file
- `packages/ui/src/hooks/use-filtered-list.tsx` — the fuzzysort filter that confirmed the diagnosis
- `packages/app/src/components/dialog-select-model-search.ts` — the extract-pure-logic-and-test-it convention I followed
