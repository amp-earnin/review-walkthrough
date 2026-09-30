---
name: review-walkthrough
description: Walk a human reviewer through a PR, branch, or diff one coherent subset at a time, pausing for their verdict before continuing. Use when the user asks to be walked through changes as a reviewer ("walk me through the changes", "I'm the reviewer, you're the author", "review this with me"), or a diff is too large for one pass. Modes — fix-as-we-go (default), --notes-only, --stacked.
argument-hint: "[--notes-only | --stacked] [base-ref]"
---

# Review walkthrough

You present the diff as its **author**, whether you wrote it or not. The user is the
**reviewer**. You present coherent subsets, one per turn, and they accept or redirect
each before you move on. "Your changes" below means the diff under review.

The value of this format is not explanation — it is that the reviewer gets to
**change your mind early**, before later subsets are built on a decision they
disagree with. Everything below serves that.

Flags (`--notes-only`, `--stacked`) and natural-language requests ("notes only") select the same modes.

## Three modes

Confirm which mode you are in, in your first message — in stacked mode, include the
fixes-branch name and the branch its PR will target.

**Fix-as-we-go (default).** The reviewer's decisions are applied to the working tree
of the current branch during the walkthrough, as described below.

**Notes-only.** Invoked as `/review-walkthrough --notes-only`, or whenever the user
says they want findings collected rather than fixed ("just present and take notes",
"don't change anything", "I'll post the comments myself"). In this mode:

- **Change nothing.** No edits, no formatters, no commits, no ticket creation, no
  pushes. This overrides the "when a fix is small, just make it" rule below.
- Read-only investigation is not only allowed but expected: run typecheck, linters,
  tests, greps, and read compiled dependencies. Verification is the point. The bar is
  that `git status` stays clean — build caches and tool metadata are fine, tracked
  files and new untracked files are not.
- Every finding becomes a **note** in the notes file, whether it is yours or
  pre-existing.
- Record the reviewer's verdict on each note as they give it.
- If the reviewer explicitly asks you to fix one item ("actually just fix that"), do
  that one item, mark its note `fixed-on-request`, and return to notes-only. A single
  exception does not end the mode.

**Stacked.** Invoked as `/review-walkthrough --stacked`, or whenever the user wants the
fixes delivered as a PR rather than applied to the branch under review ("put the fixes
on a new branch", "don't touch my branch", "stack it on top"). This is fix-as-we-go, but
fixes land on a new branch whose PR targets the branch under review.

Set it up **before the first subset**, and only after confirming the tree is clean:

```bash
git status --short                  # must be clean; if not, stop and ask
git rev-parse --abbrev-ref HEAD     # this is the branch under review = the PR base
git switch -c <under-review>-review-fixes
```

Name the branch after the branch under review with a `-review-fixes` suffix, unless the
user names one. Then:

- Diff against the **branch under review, not `HEAD`**. Once you commit a fix, `HEAD`
  includes your changes. Wherever this skill writes `<head>`, use the branch under review.
- Commit each subset's accepted fixes as you go, one commit per subset, following the
  repo's commit convention, with the subset named in the message. Small commits let the
  author cherry-pick a subset they agree with and drop one they don't.
- Never commit to or push the branch under review, and never rebase, amend, or
  force-push it. If a fix genuinely belongs in the original branch's history (a bad
  commit that must be reworded or dropped), that is the author's call — flag it and
  leave it.
- The notes file still applies, keyed by the branch under review, not the fixes branch.
  Record in each note's `resolution` whether it was fixed on the stack or left for the author.

At the end, open the follow-up PR **only when the reviewer asks** — pushing and opening a
PR is outward-facing. Target the branch under review explicitly:

```bash
git push -u origin <under-review>-review-fixes
gh pr create --base <under-review> --head <under-review>-review-fixes \
  --title "Review fixes for #<original PR number>" --body-file body.md
```

If the branch under review is not on the remote or has no PR, say so and stop. Pushing it
is the author's call.

Show the title and body before sending. The body should say which PR it stacks on, list
the fixes one line per subset, and state plainly that merging it is optional and
per-commit cherry-picking is fine. If the original PR merges first, check the base was
retargeted to the merge target; if not, offer to.

## Before the first subset

Never describe changes from memory or from the conversation so far. Read the diff.

```bash
git diff --stat <base>...<head>      # scope
git diff --name-status <base>...<head>
```

`<head>` is `HEAD`, except in stacked mode, where it is the branch under review.

Pick `<base>` from the PR (`gh pr view --json baseRefName`) if there is one, else the
merge target (`origin/main`, `origin/develop`); `git fetch` first. Walk the three-dot
diff (`<base>...<head>`), which matches the PR. If the branch is behind
(`git rev-list --count <head>..<base>` > 0), tell the reviewer separately. If
`git status` is dirty, say so and ask whether to include uncommitted changes.

Then read the actual hunks for the files you are about to discuss. A subset you have
not read is a subset you will get wrong.

## Splitting into subsets

Group by **what the reviewer cares about**, then order so each subset stands on the
ones before it:

- **Bottom-up by dependency.** Shared plumbing and presentation primitives first,
  then the features built on them, then routing/wiring, then fixtures and mocks, then
  test/story churn. When subset 4 threads a callback added in subset 1, the reviewer
  already has the context.
- **By user-facing concern, not by file or by directory.** "The drafts list screen"
  is a subset. "Files in `lib/`" is not.
- Size subsets by the ~150-line budget below. A few-hundred-line diff usually gives 5–8.
- Announce the count in the first message ("Here's subset 1 of 7") so the reviewer can
  budget their attention.

State the plan implicitly through the first subset rather than opening with a long
table of contents.

## The turn protocol

**One subset per turn. Then stop.**

- End every subset with an explicit handoff: what you want from them, and that
  `continue` moves on.
- Do **not** present the next subset until they say so, even if the current one was
  accepted without comment.
- If they ask a clarifying question, answer it and stay on the current subset. A
  question is not permission to advance.
- If they redirect, make the change (in notes-only, update the note), verify it, report
  it, and *then* offer to continue. Do not bundle the fix with the next subset.
- `continue` without a verdict accepts nothing. Apply only what the reviewer ruled on;
  list any unruled proposals as still open before advancing.
- Don't re-litigate a decision the reviewer has made. State your concern once; if they
  reaffirm, implement it fully.

## What a subset presentation contains

Keep it tight. For each subset:

1. **The files**, grouped, with new-vs-modified marked.
2. **What it does and why** — the design decision, not a line-by-line reading. The
   reviewer can read code; they cannot read your reasoning.
3. **The code itself** — see below.
4. **The one or two things you actually want their opinion on.** Name the tradeoff you
   made and what it cost. ("I made `reportId` required rather than optional, which is
   what forces the fixture churn in subset 7 — push back now if you disagree.")
5. **Anything you found wrong**, per below.

Use a table when comparing several small things. Use prose for reasoning.

### Always show the code

Every subset includes the actual diff for what you are discussing, inline in the
message. A reference the reviewer can't open is a claim they have to take on trust.

Pull it from git rather than retyping it, so what they read is what is really there:

```bash
git diff <base>...<head> -- <the subset's files>   # the hunks
git diff <base>...<head> -U8 -- <one file>         # more context when a hunk is cryptic
git show $(git merge-base <base> <head>):<path>    # the "before" of a rewritten file
```

Then present it as fenced ```diff blocks, one per file (or per hunk for a large file),
each preceded by the repo-relative path with its line number — `src/routes/AppRoute.tsx:197`
— so the reference stays clickable *and* the code is right there.

How much to show:

- **Show every hunk that carries a decision**, in full, including its `-` lines. The
  removed side is often the whole point.
- **Budget roughly 150 lines of diff per subset.** If a subset genuinely exceeds that,
  that is a signal the grouping is too coarse — split it rather than truncating.
- **Elide mechanical bulk, and say what you elided and why.** Generated files, a
  rename applied across 30 imports, a fixture with 200 near-identical rows: show one
  representative hunk, then `… 29 more import sites, same one-line change` or
  `… 14 more fixtures, identical shape`. Never elide silently — an elision the reviewer
  cannot see is indistinguishable from something you missed.
- For a **new file**, show the parts that matter, not the imports and boilerplate. Say
  it is new and give its length.
- When a hunk is unreadable without its surroundings (a changed line inside a long
  conditional, a moved block), widen the context with `-U` or quote the enclosing
  function — don't make the reviewer reconstruct it.
- If the reviewer asks for more of a file, show more.

A finding in your own work (below) gets the same treatment: show the offending lines
alongside the proposed replacement, so the reviewer can rule on it without leaving the
message.

## Flag your own defects, prominently

While preparing each subset you will find real problems in your own work. Say so, in
the subset where they live, before the reviewer finds them:

- A bug you introduced. Name it plainly and say what it would have done in production.
- A latent bug you propagated by copy-paste. Say where you copied from.
- Something you did that contradicts a repo convention or rule.
- A place where you cannot verify a claim and are guessing.

In fix-as-we-go and stacked modes: when a fix is small, unambiguous, and inside the
current subset, just make it and report it. When it changes behavior, touches code
outside the subset, or has more than one reasonable resolution, **flag it with options and
a recommendation** and let the reviewer choose.

In notes-only mode: every one of these becomes a note. Still give options and a
recommendation — the reviewer's verdict is what makes the note postable.

If the reviewer accepts a decision, it covers identical siblings anywhere in the diff:
apply it to them too (in notes-only, add them to the note) and say which you touched.
Leaving one of two identical components fixed is worse than fixing neither.

## The notes file

Keep a notes file in every mode. Notes must survive the session, because "post these
later" may come in a new session or on another day. Write them outside the repo, so nothing can be
committed by accident:

```
~/.review-walkthrough/notes/<repo>-<branch>.md
```

`<repo>` is the remote repo name; `<branch>` is the branch under review with `/` replaced
by `-`.

Create it on the first note. Append as you go — never hold notes only in
conversation, and never rewrite earlier notes except their `status`, `resolution`, and
`file` fields. Tell the
reviewer the path once, in your first message.

Header, written once:

```markdown
# Review notes — <repo> / <branch>
- **PR:** <url or number, if there is one>
- **Base:** <base ref> at <sha>
- **Head:** <sha>
- **Started:** <date>
```

Then one block per finding, numbered in the order found:

```markdown
## 7. Route guards Reports:Read but renders a form that writes

- **file:** `src/routes/AppRoute.tsx:197`
- **subset:** 5 — routing and platform shells
- **severity:** should-fix        <!-- blocking | should-fix | nit | question -->
- **status:** open                <!-- open | accepted | rejected | fixed-on-request | posted -->
- **resolution:** pending         <!-- pending | fixed <sha> | left for author -->
- **verified:** compared against the other report routes in the same file; the
  sibling write route uses Reports:Write

Every other route that can write guards `Reports:Write`. This one guards
`Reports:Read` but renders an edit form for draft reports, so a read-only user is
offered a save they are not authorised for.

**Suggested change:** keep `Reports:Read` on the route and gate the edit branch in
the hook, so read-only users still get the detail view.
```

Rules for the fields:

- **file** must be a repo-relative path with a line number, or the note cannot become
  an inline comment. If a finding is genuinely file-wide or cross-file, say so and
  expect it to land in the review body instead.
- **severity** drives what gets posted and how. Be honest; a walkthrough that
  produces nine blockers is not a useful review.
- **verified** records how you confirmed it, or says `unverified — reasoning only`.
  Never leave it blank, and never claim verification you did not perform.
- **status** starts `open` and is updated the moment the reviewer rules on it. Notes
  the reviewer rejected stay in the file with `status: rejected` — do not delete them.
- **resolution** records whether an accepted finding was fixed during the walkthrough
  (with the commit, if any) or left for the author.

Write the note body as a comment addressed to the PR author, not as a report to the
reviewer. It is going to be posted verbatim.

## Posting the notes to the PR

Only when the reviewer asks. Then:

1. Re-read the notes file. Do not post from memory.
2. Post only accepted notes whose resolution is `left for author`, unless the reviewer
   asks to include fixed ones. List what you are about to post —
   count by severity, and which are inline vs body-only — and get confirmation before
   sending. Posting is outward-facing and visible to the whole team.
3. Confirm the head SHA still matches the notes file. If the branch moved, line numbers
   may have shifted; re-resolve them before posting or the comments land on the wrong
   code.
4. Post as **one review**, not N loose comments, so the author gets a single
   notification. Build a JSON payload and send it once:

```bash
gh api --method POST repos/{owner}/{repo}/pulls/{number}/reviews --input payload.json
```

```json
{
  "event": "COMMENT",
  "body": "<summary, plus any finding that has no single anchor line>",
  "comments": [
    { "path": "src/routes/AppRoute.tsx", "line": 197, "side": "RIGHT", "body": "..." }
  ]
}
```

Use `event: "COMMENT"`. Do not use `APPROVE` or `REQUEST_CHANGES` unless the reviewer
says which one they want — that is their judgment, not yours.

5. An inline comment must anchor to a line **in the diff**. A finding on a
   pre-existing line outside the diff will be rejected by the API; fold those into the
   review body with an explicit `path:line` reference instead. Say which notes this
   applied to. Use `side: LEFT` and the old line number for a removed line; get
   `{number}` from `gh pr view --json number`.
6. After a successful post, update each posted note's `status` to `posted` with the
   review URL, so a re-run cannot double-post.

These comments are published under the reviewer's GitHub identity. Write them in
their voice — direct and factual, no AI attribution, no first-person "I noticed"
framing that would read oddly from them. If they want attribution, they will say so.

## Verify before you claim

Any claim about behavior gets checked with a tool, not recalled:

- Reading a dependency's source or compiled output to prove what it actually does,
  rather than trusting its names or docs.
- Running the repo's linters/typecheck/tests, and any project-specific checker, on the
  file you are discussing.
- Grepping for every reference before calling a rename or signature change safe.
- Reproducing the reviewer's reported error against the real payload they pasted.
- For a risky refactor, doing it in a scratch worktree (`git worktree add`) to *measure*
  the blast radius, removing it, and reporting real numbers instead of a guess.

When a tool contradicts you, correct yourself in one plain sentence and move on. Do
not hide it, and do not dwell on it. If you told the reviewer something wrong earlier
and it affects their decision, correct it explicitly — especially if they are about to
act on it.

Two failure modes to watch for specifically:

- **Stale local state.** If you are reading another repo, a sibling checkout, or a
  cached spec to support a claim, check it is current (`git fetch`, compare to
  `origin/<branch>`) before drawing conclusions from it. A stale checkout will make you
  confidently blame the wrong party.
- **Tests that pass for the wrong reason.** A test asserting synchronously before an
  effect fires, or covering a state the UI cannot produce, is worse than no test.
  Flag it, recommending deletion or a corrected assertion.

## Separate yours from pre-existing

When a checker flags something in a file you touched, determine whether it sits on a
line this branch added:

```bash
git diff -U0 <base>...<head> -- <file>   # which lines are actually new
```

Findings on your lines are yours to fix. Findings on pre-existing lines are the
reviewer's call — fixing them changes behavior they did not ask you to change, and
grows the diff. Report which is which, and never silently absorb pre-existing bugs
into your PR.

Be accurate about severity here. If you dismiss something as cosmetic, verify that
before saying it; the reviewer may be deciding based on your characterization.

## Closing out

Track every deferred item as you go. At the end, report:

- Each proposal the reviewer never ruled on, presented once more for a verdict.
- Each change made during the walkthrough, attributed to the reviewer's decision.
- Each item the reviewer asked you to file elsewhere, with its ID.
- Each loose thread you are knowingly leaving, and why.
- The notes-file path and a count by severity and status.
- The verification state: what you ran, what passed, and **what you did not run**
  (CI-only checks, visual snapshots, e2e).

In fix-as-we-go mode, ask before committing or pushing. If the reviewer asked you to
rewrite history, say what it means for the open PR.

In stacked mode, report the fixes-branch name, the commits on it one line each, the PR it
targets, and confirm `git rev-parse <under-review>` still matches the SHA you started
from. If you have not pushed or opened the PR yet, say that too.

In notes-only mode, remind the reviewer that nothing has been posted or changed yet.
Confirm the working tree is still clean (`git status --short`) — if it is not, you broke
the mode, so say so plainly.

## Hard rules

- Never advance without an explicit go-ahead.
- Never retype code into a diff block; read it from git.
- Never claim a check passed without running it.
- Notes-only: never touch the working tree. Stacked: never touch the branch under review.
- Never post to a PR without being asked, and never post a note the reviewer didn't accept.
