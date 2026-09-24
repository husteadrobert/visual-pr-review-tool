# pr-review-tool

Step-by-step runtime walkthrough of a pull request, for human reviewers.

Give it a PR number. Get one HTML file that walks through what the code does at runtime, one step at a
time: the diff for that step, the unchanged code it calls into, what the system state looks like after
it, what happens on each error, and the questions a human should ask there.

## Usage

Set these once, for example in your shell profile:

```
export PRT_REPO=~/Repos/your-app        # local checkout of the repo you review (--repo PATH overrides it)
export PRT_GH_REPO=your-org/your-app    # the same repo on GitHub, as owner/name
```

```
bin/generate 1234                  # full run: gh + worktree + claude + html  -> out/walkthrough-1234.html
bin/generate 1234 --dry-run        # print the prompt Claude gets
bin/generate 1234 --from-trace .cache/1234/trace_raw.json   # re-render the page without calling Claude
bin/generate 1234 --offline        # reuse cached PR metadata and diff (no gh calls)
bin/generate 1234 --model claude-fable-5-1 --effort xhigh   # defaults: claude-opus-5, effort high (env PRT_MODEL / PRT_EFFORT)
bin/generate 1234 --lang ja        # everything in Japanese, code identifiers untouched -> out/walkthrough-1234.ja.html
```

Open `out/walkthrough-<pr>.html` in a browser. Arrow keys move between steps, `b` follows the first
error branch, `esc` returns to the main path, `f` toggles focus mode.

## What the page shows

- **Overview first.** The whole flow as a diagram, the purpose and approach in plain English, and one
  easy sentence per step. Every node and every list item is clickable.
- **Three tiers of step.** `changed` means the step's code is in the diff (decided from the hunks).
  `affected` means unchanged code whose behaviour is different after this PR (decided by Claude).
  `context` is the rest of the path. Focus mode, on by default, shows changed and affected steps in full
  and collapses runs of context steps into one gray row. Next and Prev skip collapsed steps.
- **Per step:** the diff hunks in a green frame, the unchanged code the step runs through in a dashed
  gray frame, every error branch including inherited ones, a collapsible spec section with the actual
  spec excerpt and spec diff, and one to three human questions.
- **State panel:** rows written, jobs enqueued, external calls, files, cumulative through the current
  step. Click a counter for the full rundown grouped by table, with the step that caused each change.
- **Error branches:** a rescue with its own path becomes a branch you can follow step by step, with the
  main path grayed out behind it.

## Cost and time so far

| PR | Lines | Model / effort | Steps (main + branch) | Turns | Time | Cost |
| --- | --- | --- | --- | --- | --- | --- |
| A | +294 | Fable 5.1 / max | 9 + 4, 2 branches | 48 | 10 min | $4.43 |
| B | +994 | Fable 5.1 / max | 10 + 10, 5 branches | 70 | 14 min | $6.89 |
| C | +4,229 | Opus 5 / high | 12 + 4, 3 branches | 34 | 8 min | $4.72 |
| D | +150 | Opus 5 / high | 7 + 0 | 33 | 4 min | $1.76 |
| E | +106 | Opus 5 / high | 5 + 2, 1 branch | 27 | 4 min | $1.21 |
| D (ja) | +150 | Opus 5 / high | 8 + 2, 1 branch | 31 | 4 min | $1.62 |
| F (ja) | +122 | Opus 5 / high | 8 + 2, 1 branch | 22 | 4 min | $1.32 |

Every run so far: zero unverified file:line references. Switching from Fable at max effort to Opus 5 at
high cut thinking from 58% of output tokens to about 22% and handled a 14x larger PR for the same money.

## How it works

1. `gh pr view` / `gh pr diff` fetch metadata and the diff. The diff is split into hunks `H1..Hn`.
2. A detached git worktree of the `PRT_REPO` checkout is created at the PR head SHA under `.worktrees/<pr>`.
   Your main working tree is never touched.
3. `claude -p` runs inside that worktree with `prompts/trace.md`. It reads the changed files, follows
   calls into unchanged code and base classes, reads the PR's specs, and emits a JSON trace.
   It can only read; it has no Write or Edit tools.
4. The generator attaches the real diff text per hunk, reads the referenced unchanged code, verifies
   every `file:line` exists at that SHA, adds the per-layer question bank, and inlines the JSON into
   `lib/template.html`.
5. The page (Vue 3, diff2html, Mermaid, all vendored in `vendor/`, CDN fallback) renders the JSON.

State shown on the page is **predicted from code and specs, not observed**. Every state line carries a
`source` tag (`spec`, `code`, `inferred`) and a `file:line`. Anything the generator could not find at
the PR SHA is marked "not found".

## Layout

```
prompts/trace.md     the prompt and JSON schema Claude follows
bin/generate         Ruby, stdlib only (system ruby is fine)
bin/vendor           re-download the browser libraries
lib/template.html    the Vue page
vendor/              vue, diff2html, mermaid (git-ignored; run bin/vendor)
.cache/<pr>/         pr.json, diff.patch, prompt.md, claude_stdout.json, trace_raw.json
.worktrees/<pr>/     detached checkout at the PR head (git-ignored)
out/                 walkthrough-<pr>.html and trace-<pr>.json
```

## Cleanup

```
git -C "$PRT_REPO" worktree remove .worktrees/<pr>   # or: git worktree prune
```
