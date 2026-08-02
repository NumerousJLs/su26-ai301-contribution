# Contribution 4: Search all known projects in the Open Project dialog

**Contribution Number:** 4

**Student:** Joshua Liu

**Issue:** [anomalyco/opencode#39142](https://github.com/anomalyco/opencode/issues/39142)

**Fork:** [github.com/NumerousJLs/opencode](https://github.com/NumerousJLs/opencode)

**Branch:** `project-picker-search`

**Base reviewed:** `upstream/dev` at `32f278b48f`

**Pull Request:** Pending user review

**Status:** The fix and regression test are committed locally. Nothing from this review has been pushed.

## Summary

The Open Project dialog displays at most five recent projects when its search box is empty. Before this fix, the same five-item limit was applied before the dialog's fuzzy filter received its candidates. A known project outside that slice therefore could not be found by name.

The fix keeps every known project in the search corpus and applies the five-item cap only when the query is empty. A Playwright test registers six projects and proves both requirements:

- With no query, the dialog still displays five recent projects.
- With a query, the sixth project can be found by name.

## Why I Chose This Issue

The report included a concrete symptom, the suspected line, and the expected scope. That made it possible to validate the diagnosis against the implementation instead of guessing.

The issue is also a useful example of a data-flow bug. The fuzzy matcher works correctly, but its input has already been truncated. The problem is not the search algorithm. The problem is where the display limit is applied.

## How the Dialog Works

`DialogSelectDirectory` builds two groups of rows:

1. Known projects from the application's stored project list.
2. Directory results returned by the server for the current input.

The combined rows are passed to the shared `List` component. `useFilteredList` calls the dialog's asynchronous `items(value)` function and fuzzy-filters only the rows returned by that function.

Before the fix, `recentProjects()` contained:

```ts
return projects
  .map((project, index) => ({ project, at: byProject.get(project.worktree) ?? 0, index }))
  .sort((a, b) => b.at - a.at || a.index - b.index)
  .slice(0, 5)
  .map(({ project }) => {
    // Build the row.
  })
```

Because `.slice(0, 5)` ran there, projects six and later never reached `items(value)` or the fuzzy filter. Searching harder could not recover data that had already been removed.

The full-path behavior is separate. Directory lookup can return a path even when that project is absent from the recent-project rows. This explains why the reporter could find the project by absolute path but not by project name.

## Fix

The patch removes the early slice and applies the limit where the query is available:

```ts
const items = async (value: string) => {
  const results = await directories(value)
  const directoryRows = results.map((absolute) => toRow(absolute, home(), "folders"))
  // Cap the idle list only. Once a query narrows the results, every project stays searchable.
  const recent = recentProjects()
  const visible = value ? recent : recent.slice(0, RECENT_PROJECT_LIMIT)
  return uniqueRows([...visible, ...directoryRows])
}
```

This preserves the intended idle display:

| State | Recent-project candidates |
| --- | --- |
| Empty query | First five projects |
| Non-empty query | Every known project |

Directory results, sorting, grouping, selection, and deduplication are unchanged.

## Accurate History

The related issues and pull requests do not all describe the same defect. Their chronology matters.

| Date | Item | What the primary source establishes |
| --- | --- | --- |
| 2026-01-06 | [#7111](https://github.com/anomalyco/opencode/issues/7111) | The dialog had no recent-project data source. The reporter asked for recent projects to be included in Open Project search. This predates the code changed here. |
| 2026-01-11 | [#7877](https://github.com/anomalyco/opencode/issues/7877) | Requested a way to see more than five projects on the home page. This is a visibility request, not the dialog-search defect. |
| 2026-02-10 | [#12887](https://github.com/anomalyco/opencode/issues/12887) | Requested the top five most recently accessed projects in the Open Project modal. |
| 2026-02-27 | [PR #15270](https://github.com/anomalyco/opencode/pull/15270) | Added the recent-project section. An initial commit built rows for all recent projects. Follow-up commit `b296f0c9` added the five-item slice after review called for the top-five behavior from #12887. |
| 2026-02-27 | Merge `1f108bc401` | Shipped the recent-project feature with the early slice. |
| 2026-07-27 | [#39142](https://github.com/anomalyco/opencode/issues/39142) | Identified that the slice runs before filtering and proposed moving the limit to the empty-query path. |

### Why the Cap Was Added, and Why This Patch Honours It

The single most useful fact in the investigation, and the one I lacked entirely until the second review round, is the wording of the review that produced the slice. On [PR #15270](https://github.com/anomalyco/opencode/pull/15270) the reviewer wrote:

> The issue (#12887) specifies displaying "the top 5 most recently accessed projects", but the current implementation shows all projects without any limit. Consider adding a `.slice(0, 5)` after sorting to limit the number of recent projects **displayed**, as specified in the requirements.

The author replied "limited to 5" and added commit `b296f0c9` ("feat: limit to 5"), sixth of seven commits in that PR.

The cap was therefore always a **display** rule. The defect is that it was implemented at a point in the pipeline that also determines the search corpus. That reframes this patch: it is not relaxing a deliberate limit, it is restoring the limit to the scope the reviewer asked for. That argument is now the second paragraph of the PR body, and it is far stronger than anything I had written before, which was a speculative scope caveat built on a misreading of #7877.

Method note: this is only visible through `gh api repos/OWNER/REPO/pulls/N/commits` and `/pulls/N/comments`. Because opencode squash-merges, `b296f0c9` does not exist in `dev` history at all, and `git log -S` on `dev` correctly but misleadingly attributes the line to the squash commit `1f108bc401`.

### Relationship to #7111

#7111 is a predecessor, not the same code bug. When it was filed, `recentProjects()` and its `.slice(0, 5)` did not exist. PR #15270 partially addressed the earlier request by adding recent projects to the dialog, but the shipped top-five implementation left projects outside the slice unsearchable.

#7111 was closed with GitHub's `not planned` state on 2026-05-22. There is no maintainer comment explaining that closure. It was part of a mass cleanup: **441 issues were closed repo-wide on 2026-05-22**, including #7877. The state alone is therefore no evidence that maintainers rejected anything. My earlier attempt to test this returned "0 other issues closed that day" because the query filtered on `since=` (which matches *updated*, not *closed*) and was unpaginated, so a broken check produced a confident wrong answer that I then wrote into the PR.

### Relationship to #7877

#7877 asks to display more projects on the home page. This patch intentionally does not do that. The visible Open Project list remains capped at five, so the patch does not reverse or override #7877's closure.

### Automated Cross-References

GitHub Actions added similarity links among several project-picker issues. Those links are discovery aids, not proof that the issues share a root cause. The other linked reports cover CLI project listing, filesystem traversal, server folder loading, sidebar search, and other adjacent behavior. Each needs its own code-path and timeline check.

## Assignment and Ownership

The issue timeline shows `github-actions[bot]` assigning #39142 to `Brendonovich` shortly after creation. That establishes automated routing, not that the assignee claimed or began the work. A competing-work check still needs to examine the full issue timeline, open pull requests, file overlap, and the current default branch.

The final check found no open pull request that implemented this same change.

## Environment Fidelity

The report names Windows 11 as the host and WSL2 as the opencode server environment. Code running inside WSL generally receives POSIX paths such as `/home/user/project`, not native `C:/...` paths.

The regression test therefore uses POSIX project paths. A native Windows-path fixture might test another supported topology, but it would not reproduce this report more accurately merely because the host operating system is Windows.

## Regression Test

The committed Playwright spec is:

`packages/app/e2e/regression/project-picker-recent-search.spec.ts`

It seeds six known projects before the app loads, opens the real project dialog, and scopes row assertions to `[data-directory-path]` so sidebar text cannot produce a false positive.

The search test targets `foxtrot-docs`, the sixth project:

```ts
test("searches every recent project, not just the five most recent", async ({ page }) => {
  const search = await openProjectDialog(page)
  await expect(row(page, OUTSIDE_CAP)).toHaveCount(0)

  await search.fill("foxtrot")

  await expect(row(page, OUTSIDE_CAP)).toHaveCount(1)
})
```

The second test separately preserves the original display requirement:

```ts
test("still caps the idle recent list at five projects", async ({ page }) => {
  await openProjectDialog(page)

  await expect(row(page, NAMES[4])).toHaveCount(1)
  await expect(row(page, OUTSIDE_CAP)).toHaveCount(0)
})
```

This is stronger than the removed helper unit test. The helper test only proved what a new helper did when given all rows. It could not detect an upstream caller truncating those rows first. The e2e test exercises the actual data path that contained the bug.

## Verification

All commands were rerun after rebasing onto `upstream/dev` at `32f278b48f`.

```text
bun test src/components/directory-picker-domain.test.ts
20 pass, 0 fail

bun test src/components/directory-picker.test.ts
1 pass, 0 fail

bun typecheck
exit 0

bun typecheck:e2e
exit 0

bunx playwright test e2e/regression/project-picker-recent-search.spec.ts --project=chromium
2 passed

prettier --check on both changed files
clean

targeted oxlint on both changed files
0 warnings, 0 errors

git diff upstream/dev...HEAD --check
clean
```

### Red/Green Proof

The same Playwright spec was run after temporarily restoring the old early `.slice(0, 5)`:

```text
1 failed: searches every recent project, not just the five most recent
1 passed: still caps the idle recent list at five projects
```

The source was then restored. This result proves that the test fails for the reported defect while preserving the intended idle behavior.

### Verification Environment Note

After fetching the latest `dev`, `solid-sonner` was present in the lockfile but not in the existing local installation. `bun typecheck` initially failed in untouched toast code. Running `bun install` with Bun available on `PATH` installed the new dependency without changing tracked files, after which both typechecks passed.

The current upstream `.oxlintrc.json` contains repeated `options` objects and is rejected by Oxlint 1.60.0 before linting starts. The two changed files were therefore checked with the same non-type-aware lint categories and rule exceptions in a temporary configuration. The repository configuration itself was not modified because it is unrelated to this contribution.

## Commits

The branch contains two commits after the final rebase:

```text
3e934aa63a fix(app): search every known project in the open project dialog
4440d07202 test(app): cover recent project search in the open project dialog
```

Net diff:

```text
2 files changed, 66 insertions, 2 deletions
```

## Pull Request Draft

```markdown
### Issue for this PR

Closes #39142

### Type of change

- [x] Bug fix
- [ ] New feature
- [ ] Refactor / code improvement
- [ ] Documentation

### What does this PR do?

Searching the Open Project dialog now checks every known project when you type, while the idle Recent projects list remains capped at five.

Before this change, `recentProjects()` applied `.slice(0, 5)` before its rows reached `List`. A project outside that slice could not match the fuzzy filter because it was never included in the filter's input. The directory search still handled full paths separately, which is why entering the same project's absolute path worked.

This implements the reporter's proposed fix by moving the cap to `items()`, where the query is available. An empty query still returns the first five recent projects. A non-empty query keeps every known project available to the existing fuzzy filter. Directory results, grouping, and deduplication are unchanged.

### How did you verify your code works?

`e2e/regression/project-picker-recent-search.spec.ts` registers six projects and covers both directions: the sixth is findable by name, and the idle list is still capped at five. Re-running the same spec against the pre-fix source makes the search case fail while the idle-cap case still passes.

- `bun typecheck` and `bun typecheck:e2e`
- 20 existing directory-picker domain tests
- 1 existing directory-picker routing test
- 2 Playwright regression tests
- Prettier on both changed files

### Checklist

- [x] I have tested my changes locally
- [x] I have not included unrelated changes in this PR
```

## Screenshots

Before the fix, searching for the sixth project returns no dialog row:

![Before: sixth project missing from search](https://raw.githubusercontent.com/NumerousJLs/su26-ai301-contribution/main/pr-screenshots/before-2-search-foxtrot.png)

After the fix, searching for the same name returns the project:

![After: sixth project appears in search](https://raw.githubusercontent.com/NumerousJLs/su26-ai301-contribution/main/pr-screenshots/after-2-search-foxtrot.png)

With no query, the dialog still shows five recents:

![After: idle list remains capped at five](https://raw.githubusercontent.com/NumerousJLs/su26-ai301-contribution/main/pr-screenshots/after-1-idle.png)

## Review Corrections

The earlier write-up made several claims that the primary sources do not support:

- It called #7111 the same bug even though #7111 predates the recent-project implementation and the slice.
- It treated #7877's `not planned` state as a reasoned maintainer rejection, despite the absence of an explanatory comment and evidence of batch issue closure.
- It treated automated similarity cross-references as one causal issue cluster.
- It said the slice existed in PR #15270's first commit, but a later commit introduced it.
- It described a native `C:/...` test as WSL2-specific even though the reported server runs inside WSL.
- It retained removed helper files, obsolete test counts, and pre-rebase commit IDs after the implementation changed.

The corrective rule is simple: every historical or causal statement must be tied to the issue body, full timeline, review, commit diff, or current runtime path that proves it. GitHub metadata and bot-generated links can suggest where to investigate, but they cannot supply the conclusion.

The complete reusable process is documented in [CONTRIBUTION_WORKFLOW.md](CONTRIBUTION_WORKFLOW.md).

## Resources

- [Issue #39142](https://github.com/anomalyco/opencode/issues/39142)
- [Predecessor issue #7111](https://github.com/anomalyco/opencode/issues/7111)
- [Distinct visibility request #7877](https://github.com/anomalyco/opencode/issues/7877)
- [Top-five feature request #12887](https://github.com/anomalyco/opencode/issues/12887)
- [Merged implementation PR #15270](https://github.com/anomalyco/opencode/pull/15270)
- `packages/app/src/components/dialog-select-directory.tsx`
- `packages/ui/src/hooks/use-filtered-list.tsx`
- `packages/app/e2e/regression/project-picker-recent-search.spec.ts`
