# review-walkthrough

A skill for Claude Code and Cursor. The AI is the author. You are the reviewer. It walks you through a PR, branch, or diff one subset at a time, and waits for your verdict on each before it moves on.

## Modes

| Mode | How to start | What it does |
|---|---|---|
| Fix-as-we-go (default) | `/review-walkthrough` | Applies your decisions to the current branch. |
| Notes-only | `/review-walkthrough --notes-only` | Changes nothing. Saves findings to a notes file you can post as PR comments later. |
| Stacked | `/review-walkthrough --stacked` | Puts fixes on a new `<branch>-review-fixes` branch and opens a follow-up PR against the branch under review. |

You can also just ask: "walk me through this PR, notes only".

In Claude Code the command is namespaced: `/review-walkthrough:review-walkthrough`.

## Install in Claude Code

```text
/plugin marketplace add amp-earnin/review-walkthrough
/plugin install review-walkthrough@review-walkthrough
```

Or from a shell:

```bash
claude plugin marketplace add amp-earnin/review-walkthrough
claude plugin install review-walkthrough@review-walkthrough
```

If you had an old copy in `~/.claude/skills/review-walkthrough/`, delete it so the skill does not load twice.

## Install in Cursor

**Team.** A Cursor admin adds this repo once:
Dashboard → Plugins & MCPs → Add Marketplace → Import from Repo → `https://github.com/amp-earnin/review-walkthrough`.
Then each person installs it from **Customize**.

**Personal.** Clone into the local plugins folder, then run **Developer: Reload Window**:

```bash
git clone https://github.com/amp-earnin/review-walkthrough.git ~/.cursor/plugins/local/review-walkthrough
```

On a team plan, this works only if your admin has turned on **Allow Local Plugin Imports** (Dashboard → Settings → Security & Identity → Marketplace and Plugins).

## Update

- Claude Code: `/plugin marketplace update review-walkthrough`
- Cursor (personal): `git -C ~/.cursor/plugins/local/review-walkthrough pull`, then Reload Window.

## Notes file

Every mode saves notes outside your repo:

```text
~/.review-walkthrough/notes/<repo>-<branch>.md
```

Nothing is posted to GitHub unless you ask.

## Layout

```text
.claude-plugin/plugin.json        Claude manifest
.claude-plugin/marketplace.json   Claude marketplace
.cursor-plugin/plugin.json        Cursor manifest
.cursor-plugin/marketplace.json   Cursor marketplace
skills/review-walkthrough/SKILL.md  the skill (shared by both tools)
```

## Contributing

Issues and pull requests are welcome. To release a change:

1. Edit `skills/review-walkthrough/SKILL.md`.
2. Bump `version` in all four JSON files.
3. Add a line to `CHANGELOG.md`.
4. Run `claude plugin validate .`
5. Merge to `main`.

## License

[MIT](LICENSE)
