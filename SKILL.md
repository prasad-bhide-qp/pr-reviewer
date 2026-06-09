---
name: mobile-pr-review
description: >
  Use when the user asks to review a GitHub PR for a mobile app (React Native, Flutter,
  Android, iOS). Accepts a PR URL or number, auto-detects the mobile tech stack, applies
  platform-specific best-practice rules, and posts all inline file:line review comments
  as a single GitHub review via `gh api`. Triggers on: "review mobile PR", "review PR
  <url|number>", "do a mobile code review", "review this RN/Flutter/Android/iOS PR".
caveman: optional
---

# mobile-pr-review

## Overview

A strict, mobile-first PR review skill. Extends the base `pr-reviewer` workflow with
platform-specific lint rules for React Native, Flutter, Android (Kotlin/Java), and iOS
(Swift/Objective-C). All findings are collected into one review payload and posted in a
single `gh api` call — the user is asked once, not per-file.

Token-saving: at startup, check whether the `/caveman` skill is available in the current
session's skill list. If it is, invoke `Skill("caveman")` before any review work to
activate compressed-output mode. If `/caveman` is absent or fails, skip silently and
proceed normally.

## When to Use

Triggers: "review PR <url|number>", "mobile PR review", "review this Flutter/RN/Android/iOS PR",
"do an inline code review for mobile", "strict mobile review on PR #N".

**Don't use for:**
- Web-only or backend-only PRs → use `/review` or `/code-review`.
- Approve/Request-Changes with no comments → `gh pr review --approve`.
- A single top-level comment → `gh pr comment`.

---

## Workflow

### Step 0 — Caveman check (token optimisation)

```
if "caveman" in available_skills:
    invoke Skill("caveman")   # activates compressed output; ignore any error
```

### Step 1 — Parse inputs

- **PR identifier:** full URL → split `owner/repo` + number; bare `#N` or plain number →
  infer repo via `gh repo view --json nameWithOwner -q .nameWithOwner`.
- **Reviewer (optional):** user may pass `using <skill-or-agent>`. Resolve at Step 4.

### Step 2 — Auth check (halt on failure)

```bash
gh auth status
```

If not authenticated:

> "GitHub CLI is not authenticated. Run one of the following, then re-invoke this skill:
>
> **Option A — browser (recommended in Claude Code Mac app):**
> ```
> gh auth login --web
> ```
> **Option B — token (headless / CI):**
> ```
> gh auth login --with-token < token.txt
> ```
> After auth, run `/mobile-pr-review <PR-URL>` again."

Do NOT continue until `gh auth status` passes.

### Step 3 — Fetch PR meta + diff

```bash
gh pr view <pr> --repo <owner>/<repo> --json title,headRefName,state,files
gh pr diff <pr> --repo <owner>/<repo>
```

- If `state` is not `OPEN`, warn the user and ask whether to continue before proceeding.
- Parse the diff to build a valid-anchor set: `(file, new-side-line)` for every `+` or
  context (` `) line. Only these lines can receive inline comments via the GitHub API.

### Step 4 — Detect mobile tech stack

Examine changed file paths and extensions from the diff header:

| Indicator | Platform |
|---|---|
| `*.tsx`, `*.jsx`, `metro.config.*`, `react-native.config.*`, `android/` + `ios/` both present, `package.json` has `react-native` dep | **React Native** |
| `*.dart`, `pubspec.yaml`, `lib/`, `android/` + `ios/` both present | **Flutter** |
| `*.kt`, `*.java`, `AndroidManifest.xml`, `build.gradle`, `res/` | **Android** |
| `*.swift`, `*.m`, `*.h`, `*.xib`, `*.storyboard`, `Podfile`, `*.xcodeproj` | **iOS** |

When multiple signals are detected (e.g., a monorepo with both RN and native modules),
report all matched platforms and apply rules for each.

If no mobile signals are detected, ask: *"No mobile platform signals found. Continue with
generic review, or is this the wrong PR?"*

### Step 5 — Run the review

Apply **General Mobile Rules** (all platforms) plus the **platform-specific rules** for
every detected platform.

**Produce findings as a JSON array (internal, not shown to user until Step 7):**
```json
[
  {
    "file": "path/from/repo/root",
    "line": 42,
    "severity": "critical|important|suggestion",
    "platform": "RN|Flutter|Android|iOS|General",
    "rule": "short rule ID or name",
    "issue": "what is wrong",
    "fix": "how to fix it"
  }
]
```

If an optional reviewer skill/agent was passed by the user:
1. **Skill match:** check available skills list. If found → `Skill(<name>)` with the
   instruction: *"Review the attached PR diff for mobile issues. Return findings ONLY as a
   JSON array `[{file, line, severity, platform, rule, issue, fix}]`. No prose."*
2. **Agent match:** `Agent(subagent_type=<name>)` with the same instruction.
3. **Neither:** halt and ask which skill/agent to use.

Merge returned findings with your own; deduplicate by `(file, line, rule)`.

---

## General Mobile Rules (all platforms)

### Performance
- **PERF-01 Synchronous I/O on main thread** — any blocking read/write/network call on the
  UI thread. Consequence: UI freeze/ANR/jank.
- **PERF-02 Excessive rebuilds / re-renders** — state updates that trigger full-tree
  redraws when only a leaf changed.
- **PERF-03 Unthrottled scroll listeners** — high-frequency callbacks (scroll, accelerometer)
  without debounce/throttle.
- **PERF-04 Large in-memory image** — loading full-resolution images without downsampling
  or caching.

### Memory
- **MEM-01 Retained context/activity reference** — passing `Context` or `Activity` into a
  long-lived object (singleton, coroutine, closure) without a weak reference.
- **MEM-02 Missing lifecycle cleanup** — listeners, timers, subscriptions created in
  `onCreate`/`componentDidMount` but not removed in the matching teardown.
- **MEM-03 Bitmap not recycled** — bitmap objects allocated but `.recycle()` never called
  (Android).

### Security
- **SEC-01 Hardcoded credential/API key** — any string literal that looks like a secret
  key, token, password, or private URL embedded in source.
- **SEC-02 Insecure storage** — sensitive data written to `SharedPreferences`, `NSUserDefaults`,
  `AsyncStorage`, or `localStorage` without encryption.
- **SEC-03 Cleartext network** — HTTP URLs in production network calls or manifests without
  `usesCleartextTraffic=false` / ATS exception rationale.
- **SEC-04 Debug flag in production path** — `BuildConfig.DEBUG`, `kDebugMode`, `#if DEBUG`
  gates missing around logging or dev tooling that ships in release builds.
- **SEC-05 Exported component without permission** — Android `Activity`/`Service`/
  `BroadcastReceiver` with `exported=true` and no `android:permission` set.

### Accessibility
- **A11Y-01 Missing content description** — interactive or informational image/icon without
  `contentDescription` (Android), `accessibilityLabel` (RN/iOS), or `Semantics` label (Flutter).
- **A11Y-02 Touch target too small** — interactive element smaller than 44×44 pt/dp.
- **A11Y-03 Colour as sole differentiator** — state (error, success) communicated only via
  colour with no text or icon fallback.

### Crash-safety
- **CRASH-01 Force-unwrap on nullable** — `!` (Swift), `!!` (Kotlin), or `.value!` (Dart)
  on a value that can legitimately be null/nil at runtime.
- **CRASH-02 Unhandled async error** — `async`/`await`, `Promise`, `Future`, or coroutine
  with no `catch`/`onError` path.
- **CRASH-03 Division without zero guard** — arithmetic division where the denominator
  could be zero.

### Code quality
- **QUAL-01 God component/class** — single class/component exceeding ~300 lines with
  multiple responsibilities.
- **QUAL-02 Magic number** — bare numeric literal used for layout, timing, or business
  logic with no named constant.
- **QUAL-03 TODO/FIXME in diff** — new `TODO` or `FIXME` comments introduced without an
  issue tracker reference.

---

## Platform-Specific Rules

### React Native

- **RN-01 setState in useEffect without dependency array** — triggers infinite re-render loop.
- **RN-02 Anonymous inline function as prop** — creates a new reference every render,
  defeating `React.memo`; use `useCallback`.
- **RN-03 FlatList without keyExtractor** — missing stable keys cause incorrect item
  recycling.
- **RN-04 FlatList without `getItemLayout` for fixed-height rows** — missed scroll-position
  optimisation.
- **RN-05 Direct state mutation** — `state.array.push(...)` instead of returning a new
  array; breaks change detection.
- **RN-06 Native module call on JS thread** — heavy computation (crypto, image processing)
  not offloaded to a worker or native module.
- **RN-07 Absolute imports that escape project root** — `../../../../../../utils` style
  paths; use path aliases.
- **RN-08 Platform.OS check instead of Platform.select** — multiple `if (Platform.OS ===)`
  chains that could be a single `Platform.select({})` call.
- **RN-09 Hardcoded px dimensions without `PixelRatio`** — pixel values that won't scale on
  high-density screens.
- **RN-10 Async storage read in render** — `AsyncStorage.getItem` called directly in
  component body without `useEffect`/state.

### Flutter

- **FL-01 `setState` wrapping non-UI work** — expensive computation inside `setState()`;
  compute first, then call `setState` with the result.
- **FL-02 `BuildContext` used after `await`** — using `context` after an `await` without
  checking `mounted`; can throw if widget is disposed.
- **FL-03 Rebuilding subtrees that don't need it** — passing a new object literal directly
  to a widget parameter instead of a `const` or `final`.
- **FL-04 `const` constructor missing** — widget with all-final fields that could be `const`
  but isn't; wastes rebuild cycles.
- **FL-05 `StreamBuilder` without error case** — `builder` handles `data` but not
  `snapshot.hasError`.
- **FL-06 `dispose()` not calling `super.dispose()`** — breaks controller/listener cleanup
  chain.
- **FL-07 Direct `Isolate` instead of `compute`** — verbose low-level isolate API when
  `compute()` would suffice.
- **FL-08 Large widget tree not split into sub-widgets** — monolithic `build()` method
  over ~80 lines; hard to test and re-use.
- **FL-09 `initState` calling async without guard** — `async` method called in `initState`
  without storing the `Future` and checking `mounted`.
- **FL-10 Missing `RepaintBoundary` around expensive animated subtree** — animation triggers
  full parent repaint.

### Android (Kotlin / Java)

- **AND-01 Network on main thread** — `StrictMode` violation; any blocking network call
  outside a coroutine/`Executor`/`AsyncTask`.
- **AND-02 Cursor not closed** — `Cursor` obtained but not closed in `finally` or
  `use {}`.
- **AND-03 Fragment transaction after `onSaveInstanceState`** — causes
  `IllegalStateException` on activity restore.
- **AND-04 `apply()` vs `commit()` on SharedPreferences inside a transaction** — using
  `apply()` when synchronous persistence is required (e.g., before process death).
- **AND-05 Missing `@Nullable`/`@NonNull` annotation** — public API method parameters
  or return values with no nullability contract.
- **AND-06 `lateinit var` accessed before init** — `lateinit` property used in a path
  that can run before the property is initialised.
- **AND-07 `ViewModel` holding `Activity` reference** — `ViewModel` lives longer than
  `Activity`; holding a direct reference leaks it.
- **AND-08 Hardcoded string in layout XML** — user-visible string not referencing
  `@string/` resource; breaks localisation.
- **AND-09 `BroadcastReceiver` not unregistered** — dynamic receiver registered in
  `onResume` / `onCreate` but not unregistered in the matching lifecycle method.
- **AND-10 ProGuard/R8 rule keeping everything** — `-keep class * { *; }` or equivalent
  blanket rule that negates obfuscation.

### iOS (Swift / Objective-C)

- **IOS-01 Strong self in closure causing retain cycle** — `[self doX]` or `self.x`
  inside an escaping closure without `[weak self]` / `[unowned self]`.
- **IOS-02 Force-unwrap optional** — `variable!` where the variable can be `nil` at
  runtime.
- **IOS-03 `viewDidLoad` doing layout work that should be in `viewDidLayoutSubviews`** —
  frame-dependent calculations run before Auto Layout has resolved.
- **IOS-04 Main-queue dispatch for non-UI work** — `DispatchQueue.main.async` used for
  background work, starving the run loop.
- **IOS-05 Delegate property not `weak`** — delegate stored as `strong` causing a
  retain cycle between parent and child view controllers.
- **IOS-06 `UserDefaults` storing sensitive data** — credentials, tokens, or PII written
  to `UserDefaults` instead of Keychain.
- **IOS-07 Missing `NSLocalizedString`** — user-visible string literal not wrapped in
  `NSLocalizedString()`/`String(localized:)`.
- **IOS-08 Unbalanced `beginUpdates`/`endUpdates`** — table/collection view batch update
  without a matching end call, risking assertion failure.
- **IOS-09 `@objc` exposure of Swift type without `@objcMembers` discipline** — blanket
  `@objcMembers` on a large class exposes everything to ObjC runtime; prefer selective
  `@objc`.
- **IOS-10 Background task not ended** — `UIApplication.beginBackgroundTask` called but
  `endBackgroundTask` not guaranteed to be called in all paths.

---

## Finding Severity Levels

| Severity | Emoji | When to use |
|---|---|---|
| Critical | 🔴 | Crash risk, data loss, security vulnerability, privacy violation |
| Important | 🟡 | Performance regression, memory leak, accessibility failure, bad UX |
| Suggestion | 🟢 | Style, readability, minor optimisation, missing `const`/lint fixable |

---

## Comment Format

Use this template for each inline finding:

```
**[<emoji> <Severity>] <Title — consequence-led, max ~80 chars>**

**Platform:** <RN | Flutter | Android | iOS | General>
**Rule:** <rule-id>

**Description:**
- <file:line>: <what is wrong>

**Solution:** <one sentence framing the fix>

```<lang>
// After
<corrected code snippet>
```

**Reference:** <RFC, OWASP item, Android/Apple doc, or omit if none>
```

**Rendering rules (mandatory):**
- Title names the *consequence* for the user/system, not the technical cause.
- Each Description bullet starts with `file:line:`.
- Solution is exactly one sentence; code options go as labelled comments inside the block.
- Reference: canonical citation only (doc URL, RFC, OWASP ID). Omit if nothing fits.
- Drop any section that doesn't apply rather than leaving a blank placeholder.

---

## Step 6 — Sort and separate findings

Split findings into:
- **Inline-eligible:** `(file, line)` present in the valid-anchor set (Step 3).
- **Fallback:** anchor not in diff → goes to top-level review body as `### file:line` sections.

Sort all findings: Critical first, then Important, then Suggestion. Within each severity,
sort by file path then line number.

---

## Step 7 — Generate `review.json`

Write to `/tmp/mobile-review-<pr>.json` (never inside the project repo):

```json
{
  "event": "COMMENT",
  "body": "## Mobile PR Review\n\n**Stack detected:** <platforms>\n**Findings:** <N critical, M important, K suggestions>\n\n<fallback findings rendered here>",
  "comments": [
    {
      "path": "path/from/repo/root",
      "line": 42,
      "side": "RIGHT",
      "body": "<formatted finding>"
    }
  ]
}
```

---

## Step 8 — Preview and confirm (MANDATORY GATE — do not post before user replies)

Show the user:

1. Repo and PR number.
2. Detected platform(s).
3. Finding summary: `N critical 🔴 · M important 🟡 · K suggestions 🟢`.
4. Inline comment count + list of `file:line` anchors.
5. One fully-rendered example comment (the highest-severity finding).
6. Any findings demoted to top-level body (list file:line only).
7. The exact command that will run:
   ```
   gh api repos/<owner>/<repo>/pulls/<pr>/reviews --method POST --input /tmp/mobile-review-<pr>.json
   ```

Ask **once**:
> *"Post <N> comments to <repo>#<pr>? Reply `post` to confirm, `edit` to revise, or `cancel` to abort."*

**Do NOT call any write command before the user replies `post`.**

If the user replies `edit`: ask which finding(s) to remove or amend, update `review.json`,
and show a fresh preview. Do not re-post automatically.

---

## Step 9 — Post (after `post` confirmation)

```bash
gh api repos/<owner>/<repo>/pulls/<pr>/reviews \
  --method POST \
  --input /tmp/mobile-review-<pr>.json
```

Parse the response JSON for `html_url` and print it.

On error `"Pull request review thread line must be part of the diff"`:
- Surface the error verbatim.
- Identify the offending `(file, line)` entry.
- Ask the user: drop the entry and repost, or cancel. Do not auto-retry.

---

## Quick Reference

| Step | Command |
|---|---|
| Verify auth | `gh auth status` |
| Auth (browser) | `gh auth login --web` |
| PR metadata | `gh pr view <pr> --repo <owner>/<repo> --json title,state,files` |
| Diff | `gh pr diff <pr> --repo <owner>/<repo>` |
| Post review | `gh api repos/<owner>/<repo>/pulls/<pr>/reviews --method POST --input /tmp/mobile-review-<pr>.json` |

---

## Security Boundary (non-negotiable)

This skill is **read-only on the codebase** and **write-only to PR review comments**.

**ALLOWED:**
- `gh auth status`, `gh pr view`, `gh pr diff`, `gh pr list`, `gh repo view`
- `gh api repos/.../pulls/<pr>/reviews --method POST` (post new review)
- `gh api repos/.../pulls/<pr>/reviews/<id> --method PUT` (fix our own review body only)
- `git log`, `git show`, `git status`, `git fetch`, `git rev-parse`, `git diff` (read-only)
- File writes to `/tmp/` only

**FORBIDDEN — refuse and explain:**
- `git push`, `git commit`, `git rebase`, `git reset --hard`, `git checkout <branch>`
- `gh pr merge`, `gh pr close`, `gh pr edit`, `gh pr review --approve`
- `gh api ... --method DELETE`
- Any `Edit`/`Write` outside `/tmp/`
- Any shell pipe that downloads and executes (`curl ... | sh`)

---

## Common Mistakes

| Mistake | Result | Fix |
|---|---|---|
| Posting before user replies `post` | Unwanted live review | Always wait for explicit confirmation |
| Anchoring to a `-` (deleted) line | API rejects the comment | Use valid-anchor set from Step 3 only |
| Path relative to cwd, not repo root | Anchor mismatch | Use repo-root-relative paths |
| Auto-retrying on bad anchors | Duplicate/spam reviews | Halt and ask the user |
| Forgetting `--method POST` | GET, no review created | Always pass `--method POST --input` |
| Asking per-file or per-comment | Interrupts the flow | Confirm once, post all at once |

---

## Examples

```
User: review PR https://github.com/myorg/myapp/pull/42
You:  → caveman check → auth check → detect Flutter → diff + anchor set
       → apply Flutter + General rules → 3🔴 5🟡 2🟢
       → "Post 10 comments to myorg/myapp#42? Reply `post`…"
User: post
You:  → gh api ... --input /tmp/mobile-review-42.json → prints html_url
```

```
User: review PR 12 using amber-code-review
You:  → resolve "amber-code-review" → Skill match → dispatch → ingest JSON
       → merge with own findings → detect RN + Android
       → preview → confirm → post
```

```
User: review this iOS PR #7
You:  → auth check → detect iOS only → apply iOS + General rules
       → sort by severity → format → preview once → confirm → post
```
