---
name: macos-admin-permission
description: >-
  Run approved macOS admin operations through the native authorization dialog using
  AppleScript's `do shell script ... with administrator privileges`, instead of relying
  on `sudo` (which requires a TTY and fails in GUI / agent contexts). Use this skill
  whenever a task on macOS needs administrator privileges and should show the standard
  GUI password prompt — editing protected files such as `/etc/paths`, `/etc/hosts`,
  `/Library/LaunchDaemons/*.plist`, files owned by `root`; changing ownership or
  permissions outside the user's home directory; creating, moving, chmod/chown in
  `/Library`, `/usr/local`, `/opt`, or other protected directories; running system-level
  `launchctl`, `systemsetup`, `pmset`, `tmutil` commands; installing or removing system
  services; or recovering from `sudo: a terminal is required`, `Permission denied`, or
  `Operation not permitted` failures where admin rights are the missing piece. Also use
  whenever the user says "ask for the password", "request admin", "弹权限框", "use the
  macOS permission prompt", or mentions `osascript` / AppleScript for elevation. Always
  consult this skill before running any privileged macOS command — picking the wrong
  pattern (e.g. `sudo` inside an agent) wastes a turn and produces an unrecoverable
  TTY error.
---

# macOS Admin Permission Prompt

Use AppleScript's `do shell script ... with administrator privileges` whenever a legitimate macOS operation needs admin rights and the user should see the native password dialog. This is the right pattern for GUI apps, background agents, and CLI tools running outside a real terminal — `sudo` in those contexts almost always fails with `sudo: a terminal is required`.

The macOS authorization dialog is itself the privilege mechanism. There is no need to chain in `sudo`.

## Core Pattern

For a single fixed command:

```bash
/usr/bin/osascript -e 'do shell script "/path/to/command args" with administrator privileges'
```

For paths with spaces, dynamic values, redirection, or multiple steps — prefer a heredoc with `quoted form of`:

```bash
/usr/bin/osascript <<'APPLESCRIPT'
set targetPath to "/Library/Application Support/MyTool"
do shell script ("/bin/mkdir -p " & quoted form of targetPath) with administrator privileges
APPLESCRIPT
```

The key phrase is `with administrator privileges`. Do **not** put `sudo` inside the command — the authorization dialog already runs the wrapped command as root.

## References — Read These When the Task Calls for Them

- `references/quoting.md` — the eight quoting rules, multi-line scripts, `quoted form of`, absolute command paths, `~`/`$HOME` resolution
- `references/examples.md` — worked, copy-paste-ready templates: `/etc/paths`, SSH, `launchctl`, protected directories, LaunchDaemon install, `/etc/hosts`, multi-step scripts
- `references/troubleshooting.md` — recovery for `sudo: a terminal is required`, parser errors, `command not found`, `Operation not permitted` (TCC vs admin), user cancellation, "nothing happened"

## When to Use This Skill

Trigger this skill for:

- Editing protected files: `/etc/paths`, `/etc/hosts`, `/Library/LaunchDaemons/*.plist`, files owned by `root`.
- Creating, moving, `chmod`-ing, or `chown`-ing files inside protected directories like `/Library`, `/usr/local`, `/opt`.
- Running system-level `launchctl`, `systemsetup`, `pmset`, `tmutil`, or anything that mutates `system/...` launchd domains.
- Installing or removing LaunchDaemons.
- Recovering from `sudo: a terminal is required`, `Permission denied`, or `Operation not permitted` where the missing piece is admin rights.

Do **not** use this pattern for unauthorized access, credential harvesting, or destructive commands the user has not explicitly approved.

## Standard Workflow

1. **Probe first.** Run a read-only non-privileged command (`ls -l`, `cat`, `grep`, `systemsetup -get…`) to confirm the target path and current state. If the desired state already holds, you do not need to ask for elevation at all.
2. **Tell the user briefly** what admin action you are about to request and what it changes.
3. **Run the privileged command** through `/usr/bin/osascript ... with administrator privileges`.
4. **Check the exit code and stderr.** Treat AppleScript error `-128` as "user cancelled" — stop and ask before retrying.
5. **Verify the result** with a non-privileged command.
6. **Report what changed**, including the verification output.

## Quoting Essentials

Quoting failures are the dominant source of bugs in this pattern. The full rules and rationale are in `references/quoting.md`. The minimum you need to keep in mind:

- For anything non-trivial, use a heredoc, not inline `-e`.
- Wrap user-controlled values and paths with `quoted form of`.
- Use **absolute** command paths — `do shell script` has a minimal `PATH` and does not source shell rc files.
- Do not rely on `~`; as root it resolves to `/var/root`. Resolve the user-side home first, then use the absolute path.
- For shell redirection (`>`, `>>`), pipes, `&&`, or `||`, wrap the whole script in `/bin/sh -c "..."` and pass it as one argument.
- `with administrator privileges` belongs **outside** the parenthesized command expression.

Idempotency template — safe to re-run:

```bash
/usr/bin/osascript <<'APPLESCRIPT'
set lineToAdd to "/opt/homebrew/bin"
set scriptText to "/usr/bin/grep -qxF " & quoted form of lineToAdd & " /etc/paths || /bin/echo " & quoted form of lineToAdd & " >> /etc/paths"
do shell script ("/bin/sh -c " & quoted form of scriptText) with administrator privileges
APPLESCRIPT
```

More worked patterns — `systemsetup`, `launchctl kickstart`, LaunchDaemon installation, `/etc/hosts` edits, multi-step scripts with `set -e` — are in `references/examples.md`.

## Common Failure Modes

The four pitfalls that account for almost every failure (full diagnosis in `references/troubleshooting.md`):

| Symptom | Cause | Fix |
|---|---|---|
| `sudo: a terminal is required` | Used `sudo` inside the command | Drop `sudo`, use `with administrator privileges` |
| `Expected end of line but found identifier` | AppleScript quoting broken | Switch to a heredoc, use `quoted form of` |
| `command not found` | Minimal `PATH` inside `do shell script` | Use absolute command paths |
| `Operation not permitted` (after admin succeeds) | macOS privacy / TCC, not admin | Grant the calling app Full Disk Access / Files & Folders in System Settings |

`Operation not permitted` is the one that catches people most often — it looks like a permissions error but admin rights are **not** the fix. macOS has a separate consent layer (TCC) for Documents, Desktop, Downloads, removable volumes, Photos, Calendar, Contacts, Accessibility, and Full Disk Access. The calling application (Terminal, iTerm, the host of whatever agent is running `osascript`) needs the TCC permission. Re-prompting for admin will not help.

## Safety Rules

- Only request admin authorization after the user has approved the specific action. The skill is for **carrying out** approved operations, not for inventing privileged work.
- Never ask the user to type their password into chat or anywhere outside the macOS GUI dialog. macOS handles the password itself.
- Never store, log, or echo passwords, tokens, keys, cookies, or other secrets.
- Avoid destructive commands (`rm -rf` on system paths, overwriting system files, unloading critical services) unless the user explicitly requested them and the target has been verified by a non-privileged probe.
- Prefer idempotent commands — `grep -qxF ... || echo ... >>` instead of unconditional append, `mkdir -p` instead of `mkdir`, `launchctl kickstart -k` instead of `bootout` + `bootstrap`.
- Always run a verification command afterwards. Report what changed based on what the verification actually shows, not what you intended.

## Quick Checklist

Before invoking `osascript`:

- Is the target path exact and verified by a non-privileged probe?
- Does the command avoid `sudo`?
- Are user-provided values wrapped with `quoted form of`?
- Are absolute command paths used where useful?
- Is there a non-privileged verification command ready?

After `osascript` returns:

- Check stdout, stderr, and exit code (`-128` = user cancelled).
- Run the verification command.
- Summarize the observed state in the report.
