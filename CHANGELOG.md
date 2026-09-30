# Changelog

## 1.1.0 — 2026-09-30

- First public release under the MIT license.
- Marketplace renamed from `earnin-review-tools` to `review-walkthrough`. If you installed the old one, add the new marketplace and reinstall.

## 1.0.1 — 2026-09-28

- Fixed conflicting instructions in the skill: stacked diffs use the branch under review, PR diffs use the three-dot diff and the PR's base, and "continue" and notes-only redirects are defined.
- Every mode now saves notes to `~/.review-walkthrough/notes/`, with a complete notes schema.
- Trimmed the skill description and replaced the "Do not" list with short hard rules.

## 1.0.0 — 2026-09-28

- First release as a Claude Code and Cursor plugin.
- Notes file moved from `~/.claude/review-notes/` to `~/.review-walkthrough/notes/`, so it works in both tools.
