# cs - Claude Sessions Manager

Track, label, and monitor your parallel Claude Code sessions.

## Features

### 1. Session Dashboard (`cs`)

List all active Claude Code sessions on the machine:

```bash
cs        # your sessions
cs -a     # all users (shared server)
```

```
PID      TTY      TIME    PROJECT                          TASK
-------  -------  ------ -------------------------------  --------------------
123456   pts/1    2h40m   workspace/my-project             fix auth module
234567   pts/2    23h5m   workspace/backend          [yolo] database migration
345678   pts/70   1d6h    workspace/frontend                add dark mode
456789   pts/26   1d7h    workspace/backend        [orphan] refactor infra
```

- **Green** = label in `session-labels.json` (manual or auto)
- **Dim** = auto-detected from history at query time
- **Red `[orphan]`** = parent terminal/IDE has died, process still running
- `[wt]` = `--worktree` mode, `[yolo]` = `--dangerously-skip-permissions`

Manual labeling:

```bash
cs label 52 "refactor MoE layer"   # by pts number
cs label 528378 "fix attention bug" # by PID
cs unlabel 52                       # remove
cs clean                            # remove labels for dead sessions
```

### 2. Rich Statusline

A custom Claude Code statusline showing everything at a glance:

```
🏷️ fix auth module  📁 workspace/my-project  🌿 feat/auth  🤖 Opus 4.6  📟 v2.1.70  🎨 concise
🧠 Ctx: 56% [=====-----]  ⚡ Session: 40% used, resets in 2h 31m [====------]  📊 Weekly: 6% used, resets in 6d 15h [----------]
```

**Line 1** — Session label, working directory, git branch, model, Claude Code version, output style

**Line 2** — Context window remaining, session (5h) usage limit, weekly (7d) usage limit

The statusline auto-adapts to terminal width so Claude Code doesn't clip it. Two modes:

| Width  | Layout                                                                                     |
|--------|--------------------------------------------------------------------------------------------|
| ≥ 140  | **full** — everything inline with progress bars (`Session: 24% used, resets in 1h 12m [==--------]`) |
| < 140  | **compact** — short labels (`S:` / `W:`), no bars, Weekly on its own row; line 1 drops `📟 version`, `🎨 style`, and the ` (1M context)` suffix |

Terminal width is detected in this order: `CS_STATUSLINE_WIDTH` override → `$COLUMNS` → reading the controlling pts device of an ancestor process → `100` fallback.

### 3. Usage Limit Monitoring

Usage limits displayed in the statusline, read straight from the data Claude Code
already pipes to the statusline on stdin (`rate_limits`) — no API call, no OAuth
token, no background hook:

- **Session (5h)** — current 5-hour window utilization
- **Weekly (7d)** — 7-day rolling utilization
- Color-coded: mint (normal) → peach (>=70%) → red (>=90%) → bold red (limit hit)

`rate_limits` is provided by Claude Code **only for Claude.ai subscribers (Pro/Max)**,
and only after the first API response in a session; each window can be independently
absent. When it isn't present (e.g. API-key auth, or very early in a session), the
Session/Weekly segments are simply hidden — nothing to configure, nothing to fail.

> **Earlier versions** ran a `ratelimit-probe.sh` PostToolUse hook that made a
> background Haiku API call to fetch rate-limit headers. That's gone — Claude Code
> now surfaces the same numbers natively, so the probe, its OAuth token handling,
> and the `ratelimit-cache.json` file were all removed. Re-running the installer
> (or updating the plugin) cleans up the old hook and files automatically.

### 4. Smart Auto-labeling

A PreToolUse hook (`cs-hook`) automatically labels each session on first tool use:

- Short messages (<=30 chars) → used directly as the label
- Long messages → **summarized by Haiku** into a concise label (~30 chars, same language)
- Labels appear in both `cs` output and the statusline
- Only runs once per session, cost is negligible

Examples:

| First message | Auto-label |
|---------------|------------|
| handle issue 1311, plan carefully, ask me if unclear | plan issue 1311 |
| why is usage limit not showing for other users on this machine | debug usage limit display |
| fix bug in auth | fix bug in auth |

### 5. Install & Upgrade

**Recommended — Claude Code plugin** (zero-config hooks):

```
/plugin marketplace add xiayuqing0622/claude-sessions
/plugin install claude-sessions@claude-sessions
/claude-sessions:setup
```

Then **exit and restart Claude Code** — the auto-labeling hook only loads on startup.

The plugin auto-registers `cs-hook`. The `/claude-sessions:setup` slash command runs once to configure the statusline and symlink `cs` into `~/bin`.

**Upgrading** — Claude Code caches plugin files by version, so pulling a new commit isn't enough:

```
/plugin                                          # open UI
→ Marketplaces → claude-sessions → Update marketplace
→ Installed    → claude-sessions → Update now
```

Then exit Claude Code (`exit` / Ctrl+D) and re-run `claude`. Verify in `/plugin` that the version bumped and the Errors tab has no entries for claude-sessions.

## Troubleshooting

**Statusline shows `Ctx` but not `Session` / `Weekly` usage.** `rate_limits` is only in the statusline input for **Claude.ai subscribers (Pro/Max)**, and only **after the first API response** in a session. Common cases:

- You authenticate with an **API key** (console billing) rather than a Pro/Max subscription — API-key usage has no 5h/7d windows, so these segments never appear. This is expected.
- You just started the session and Claude hasn't made an API call yet — the segments appear once it does.

Quick check — see what Claude Code is actually handing the statusline:

```bash
echo '' | your-statusline-cmd    # or inspect: the input JSON has a top-level "rate_limits" object only for subscribers
```

**`Ctx` shows `…`.** `context_window` is `null` before the first API response of a session; it fills in as soon as Claude makes a call.

---

**Alternative — clone + script** (no plugin):

```bash
git clone https://github.com/xiayuqing0622/claude-sessions.git
cd claude-sessions
./cs install
```

Does the same thing as the plugin path, but registers hooks directly in `~/.claude/settings.json`. Upgrading: `git pull` (symlinks stay live).

Custom bin directory: `./cs install /usr/local/bin`

## Requirements

- Python 3.6+ — for the `cs` dashboard and auto-labeling
- Linux (the `cs` dashboard uses the `/proc` filesystem)
- `jq` — **optional**; the statusline parses its JSON with a pure-bash fallback when `jq` isn't on `PATH`

The statusline itself has no other dependencies — usage limits come straight from Claude Code's statusline input, so there's no API call, OAuth token, or `curl` involved.

## Custom Claude config dir

All scripts honor Claude Code's `CLAUDE_CONFIG_DIR` env var. If you've moved your Claude config out of `~/.claude` (e.g. `export CLAUDE_CONFIG_DIR=/data/.claude`), `cs`, the statusline, and both hooks read/write inside that dir instead. Project-level `.claude` dirs keep their conventional name.
