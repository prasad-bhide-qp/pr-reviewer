# PR Review Skill — Team Setup Guide

A Claude Code skill that reviews mobile PRs (React Native, Flutter, Android, iOS) and posts
inline comments directly on GitHub. You set it up once and it works across all your repos.

---

## Prerequisites

Before installing, make sure you have:

- **Claude Code** installed — [claude.ai/code](https://claude.ai/code)
- **GitHub CLI (`gh`)** installed — [cli.github.com](https://cli.github.com)

---

## Step 1 — Authenticate GitHub CLI

Open your terminal and run:

```bash
gh auth login --web
```

Follow the browser prompt to log in. Verify it worked:

```bash
gh auth status
```

You should see `Logged in to github.com`.

> Already done? Skip to Step 2.

---

## Step 2 — Install the skill

Run this once in your terminal:

```bash
git clone https://github.com/prasad-bhide-qp/pr-reviewer ~/.claude/commands/pr-review
```

That's it. The skill is now available in Claude Code across all your repos.

---

## Step 3 — Enable auto-updates (recommended)

Add this line to your `~/.zshrc` or `~/.bashrc` so the skill updates silently every time
you open a terminal:

```bash
(git -C ~/.claude/commands/pr-review pull --quiet 2>/dev/null &)
```

Apply it immediately:

```bash
source ~/.zshrc   # or source ~/.bashrc
```

---

## Using the skill

Open any project in Claude Code and simply describe what you want in plain English:

```
review this PR https://github.com/org/repo/pull/123
```

```
review PR #42
```

```
do a mobile code review on PR https://github.com/org/repo/pull/99
```

Claude will automatically:

1. Verify your GitHub authentication
2. Detect the mobile platform (React Native, Flutter, Android, iOS)
3. Analyse the diff against mobile-specific rules
4. Show you a summary of all findings
5. Ask you once whether to post the comments — reply `post` to confirm

> All comments are posted as a single GitHub review. You will never be asked per-file or
> per-comment.

---

## What gets reviewed

The skill checks for issues across four mobile platforms:

| Platform | Example checks |
|---|---|
| **React Native** | Infinite re-render loops, missing `keyExtractor`, unsafe state mutations, hardcoded pixel values |
| **Flutter** | `BuildContext` used after `await`, missing `const` constructors, `dispose` not calling `super` |
| **Android** | Network on main thread, cursor leaks, `ViewModel` holding Activity reference, unregistered receivers |
| **iOS** | Retain cycles in closures, `weak` delegate missing, sensitive data in `UserDefaults`, background task leaks |
| **All platforms** | Hardcoded secrets, insecure storage, missing accessibility labels, force-unwrap on nullables, missing lifecycle cleanup |

Findings are sorted by severity: 🔴 Critical · 🟡 Important · 🟢 Suggestion.

---

## Updating manually

If you skipped auto-updates, pull the latest skill anytime:

```bash
git -C ~/.claude/commands/pr-review pull
```

---

## Uninstalling

```bash
rm -rf ~/.claude/commands/pr-review
```

---

## Troubleshooting

**Skill not triggering?**
Restart Claude Code after installation so it picks up the new command.

**`gh auth status` shows not logged in?**
Re-run `gh auth login --web` and follow the browser prompt.

**`gh: command not found`?**
Install GitHub CLI from [cli.github.com](https://cli.github.com), then re-run Step 1.

**Getting an error after posting?**
The most common error is `"Pull request review thread line must be part of the diff"` —
this means a comment was anchored to a line not in the diff. Claude will tell you which
entry to drop; confirm and it will repost.

---

## Questions or issues

Reach out to [prasad.bhide@questionpro.com](mailto:prasad.bhide@questionpro.com) or open
an issue at [github.com/prasad-bhide-qp/pr-reviewer](https://github.com/prasad-bhide-qp/pr-reviewer).
