[简体中文](README.md) | **English**

# dsh-approval-gate

**Auto-approval gate for DeepSeek Harness — minimal human intervention: safe operations auto-approve, risky ones go to a human (fail-safe).**

> ## ⚠️ This is a fork
>
> Upstream: [moon09300731/dsh-approval-gate](https://github.com/moon09300731/dsh-approval-gate)
>
> Upstream `0.5.2` is **completely inert on DSH `0.1.5-rc.2`**: the approval listener throws on its very
> first step and the exception is silently swallowed, so Flash judging, auto-approval and learning never
> run — the gate behaves as if it were not installed.
>
> This fork applies a **one-line fix** for that issue and keeps everything else identical to upstream,
> so it stays easy to sync.
>
> ```sh
> dsh plugin --profile web add github:VanemKrAu/dsh-approval-gate
> ```

A Flash model pre-judges every sandbox escalation: routine operations auto-approve, hard-risk operations (deletion / credentials / remote / system / bulk) always require human confirmation; learned rules only ever cover operations you confirmed, with an in-app human review UI.

## 🔧 What this fork changes

### The change

One line, inside the `approval/request` listener in `src/index.mjs`:

```diff
- preset = permissionPresets.current(session.events)
+ preset = permissionPresets.current(session)
```

### Why

DSH's `PermissionPresetService.current(session)` expects a **Session object**: it resolves the session's
permission projection via `permissionState(session)` → `sessionProjections.stateOf(session, 'permissions')`
→ `cellFor(registration, session)`, which reads `session.header` and calls `session.snapshotEvents()`.

The `Session` class has **no `events` property** (only a `snapshotEvents()` method), so `session.events`
is always `undefined` and the call throws:

```
[dsh-approval-gate] permissionPresets.current failed
TypeError: Cannot read properties of undefined (reading 'header')
```

The listener's own `try/catch` swallows the exception and falls through to `return next()`, handing the
approval request straight to the downstream human prompt — the gate never engages.

### Symptoms before the fix

- Every sandbox escalation prompted a human; **nothing was ever auto-approved**
- The "Approval" view stayed empty
- `~/.dsh/auto-approve/` contained only `allowlist.json` — `audit.log` / `events.jsonl` / `learning.json` were **never created**
- The "learned rules" panel stayed empty and the "learning n/N" progress never appeared

### Verification

Environment: DSH `0.1.5-rc.2` + this plugin `0.5.2`, profile `web`, session pinned to `auto-approve`.

| Item | Before | After |
| --- | --- | --- |
| `audit.log` | missing | one line per verdict |
| Flash judging | never ran | `ALLOW … (flash-safe)` auto-approved |
| Human prompts | 97 across 6 auto-approve sessions, all escalated | safe operations pass silently |
| Learning counter | always empty | `learning.json` accumulates (`neutral` category) |
| Risky operations | escalate | still escalate (fail-safe intact) |

### Relation to upstream

The same issue has been reported upstream and has fix PRs open — **none merged** as of this fork:

- Issues [#3](https://github.com/moon09300731/dsh-approval-gate/issues/3) (original report), [#5](https://github.com/moon09300731/dsh-approval-gate/issues/5), [#14](https://github.com/moon09300731/dsh-approval-gate/issues/14) (duplicate)
- PRs [#1](https://github.com/moon09300731/dsh-approval-gate/pull/1), [#7](https://github.com/moon09300731/dsh-approval-gate/pull/7), [#10](https://github.com/moon09300731/dsh-approval-gate/pull/10), [#11](https://github.com/moon09300731/dsh-approval-gate/pull/11)

If upstream lands a fix, this fork can be synced directly — the diff is one line.

## ✨ Features

- ⚡ **Flash risk pre-judgment**: every sandbox escalation is judged by a Flash model (`SAFE` / `RISKY:<category>`); recoverable operations auto-approve
- 🛡️ **Hard risks are always human**: deletion, credentials, remote/production, system paths, and bulk irreversible operations go directly to human — no counting, no learning
- 🎯 **Confirmation-based learning**: once the same tool | mode | category has been confirmed by a human **N times (default 3), the N+1th occurrence auto-approves**; persisted rules carry an **operation fingerprint**, so only operations you confirmed are auto-approved
- 🧠 **Semantic similarity verification**: operations with different wording but the same intent are judged by Flash against your confirmed samples — no keyword dependency
- 🔧 **Hot-reloadable config**: `allowlist.json` edits take effect immediately, no restart
- ✅ **Human review UI**: a green notice appears above the composer on auto-approval; the "Approval" view (right of Trajectory) shows the current session's full auto-approval timeline
- 📄 **File diff & revert** (v0.5.0+): click a file in an approval record to view a **unified diff** — changed lines with ±5 context lines, multiple changes grouped into hunks separated by gray "N unmodified lines" bars, green additions / red deletions / gray context, dual line numbers; one-click **Revert** sends a command for the AI to restore the file from snapshot
- 🗂️ **Session-scoped snapshots** (v0.5.0+): snapshots belong to the event's session; the approval view shows only the current session's snapshot stats; clearing supports "this session only" vs "clear all" to avoid wiping other sessions' unviewed diffs

## 📸 Interface Overview

### ① Approval View

![Approval View](docs/screenshots/approval-view.png)

The "Approval" tab (right of Trace) lists the current session's auto-allowed and manually-approved actions in reverse-chronological order: each record shows the tool (`bash` / `pwsh` / `edit`), a verdict tag ("Auto-allowed · Flash safe", "Approved" etc.), timestamp and description. The top bar shows this session's **diff snapshot usage** (`2.9 KB · 3 items`) with two cleanup options: **"This session only"** (removes only the current session's snapshots, never touching other sessions' unviewed diffs) and **"Clear all"** (double-confirmed, clears every session).

### ② File Diff

![Diff Dialog](docs/screenshots/diff-panel.png)

Click a file in an approval record to open the diff dialog: a **unified diff** with green additions (`+`), red deletions (`-`) and gray context lines; dual **old/new line numbers** on the left; multiple changes grouped into **hunks** with gray "`6 unmodified lines`" separators folding unchanged regions. The header shows `+2 / -2 changed · 20 unchanged`. The **Revert** button at the bottom sends an undo command to the conversation so the AI restores the file from the pre-approval snapshot.

### ③ Settings · Auto-approval

![Settings Auto-approval](docs/screenshots/settings-auto-approve.png)

The "Auto-approval" section in Settings provides full configuration: **preset initialization** (one-click write of the `auto-approve` preset into `cordis.patch.yml`), **pipeline overview** (DENY → allowlist → denyRules → Flash → learning), **deny-keyword blacklist** (built-in entries + custom add), and hot-reload notes (changes take effect immediately, no restart).

## 🚀 Quick Start

Install this fork:

```sh
dsh plugin --profile web add github:VanemKrAu/dsh-approval-gate
```

1. **Add the permission preset**: append the `auto-approve` preset to `~/.dsh/profiles/web/cordis.patch.yml` ([see guide](docs/GUIDE.en.md#%E2%9A%A0%EF%B8%8F-manual-permission-preset-required-after-install))
2. **Restart** DSH (CLI: `dsh web`; desktop app: quit and reopen)
3. **Select the preset**: choose "Auto" in the session's permission dropdown

## 📖 Docs

- [Full Guide (pipeline / configuration / security / review UI)](docs/GUIDE.en.md) · [中文指南](docs/GUIDE.md)

## 📄 License

MIT, same as upstream. Original copyright belongs to [moon09300731](https://github.com/moon09300731); this fork only adds the one-line fix described above.
