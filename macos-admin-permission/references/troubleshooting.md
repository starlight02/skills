# Troubleshooting Reference

Failure modes and recovery steps for `osascript ... with administrator privileges` workflows on macOS. When an admin command fails, walk these entries in order — most failures fit one of them.

## `sudo: a terminal is required to read the password`

You used `sudo` inside the command instead of `with administrator privileges`. `sudo` needs an interactive TTY and refuses to read the password from a GUI or agent context.

Fix: drop `sudo`, wrap the command in `do shell script ... with administrator privileges`. The macOS authorization dialog is the privilege prompt.

```bash
# wrong
/usr/bin/osascript -e 'do shell script "sudo /usr/sbin/systemsetup -setremotelogin on"'

# right
/usr/bin/osascript -e 'do shell script "/usr/sbin/systemsetup -setremotelogin on" with administrator privileges'
```

## `Expected end of line but found identifier`

AppleScript could not parse the script. Almost always a quoting issue — a stray quote, an unescaped double-quote inside a double-quoted string, or a multi-word identifier where AppleScript expected one.

Fix: switch to a heredoc form and use `quoted form of` for paths and dynamic values. See `references/quoting.md` for the full rules.

```bash
/usr/bin/osascript <<'APPLESCRIPT'
set targetPath to "/Library/Application Support/MyTool"
do shell script ("/bin/mkdir -p " & quoted form of targetPath) with administrator privileges
APPLESCRIPT
```

The heredoc removes the bash-side quoting layer entirely, so AppleScript sees its source verbatim.

## `command not found`

`do shell script` runs with a minimal `PATH` (typically `/usr/bin:/bin:/usr/sbin:/sbin`). Tools installed via Homebrew, mise, asdf, or fnm are **not** on that `PATH`, and neither are anything you added in `~/.zshrc` or `~/.bash_profile` — those files are not sourced.

Fix: use absolute command paths.

```
/usr/bin/grep         /bin/echo            /bin/mkdir
/bin/chmod            /usr/sbin/chown      /bin/cp
/usr/sbin/systemsetup /bin/launchctl       /usr/bin/awk
/usr/bin/sed          /usr/bin/tee         /usr/bin/python3
```

If you need a tool not under those directories, resolve its absolute path first (e.g. `which jq`) and use that.

## `Operation not permitted`

This usually means macOS privacy / TCC, **not** admin rights. Admin authorization grants standard POSIX root capabilities; TCC is a separate consent layer that gates access to Documents, Desktop, Downloads, removable volumes, Photos, Calendar, Contacts, Full Disk Access, and Accessibility.

Fix: do **not** retry with admin rights — the issue is privacy permission. Tell the user to grant the relevant TCC permission to the calling app (typically Terminal, iTerm, or whichever app is driving `osascript`) in **System Settings → Privacy & Security → Full Disk Access / Files and Folders / Accessibility**. After they grant the permission, retry.

If the calling app is part of an agentic harness, mention that the harness's host application is the one that needs the permission.

## `User canceled` (AppleScript error `-128`)

The user dismissed the macOS authorization dialog. Stop and ask before retrying — repeated retries without an explicit user request are unhelpful and feel coercive.

```applescript
try
  do shell script "..." with administrator privileges
on error errMsg number errNum
  if errNum is -128 then
    -- User cancelled. Stop. Ask the user whether to retry.
  end if
end try
```

## "Nothing happened" — the command ran but the change is missing

Three common causes:

1. **`~` resolved to `/var/root`**. Once elevated, `~` and `$HOME` belong to root, not the user. Resolve user-side paths to absolute form before elevating. See `references/quoting.md` Rule 4.
2. **Shell features without `/bin/sh -c`**. `&&`, `||`, `>`, `>>`, and `|` are not parsed by `do shell script` directly. Wrap the whole script in `/bin/sh -c "..."` to get a real shell.
3. **Wrong target file or destination**. Verify with `ls -l` or `cat` afterwards. A misspelled path produces no error but writes the wrong file.

## Authorization Lasts About Five Minutes

After a successful `with administrator privileges` call, the same user does not get re-prompted for ~5 minutes. This is convenient for sequences of related commands. It also means: if you batch several privileged steps, run them inside one AppleScript via `/bin/sh -c` with `set -e` so the authorization scope is well-defined. Avoid leaving privileged shells lying around between steps.

## Multiple Concurrent Prompts

If two scripts trigger admin authorization at the same time, macOS shows two dialogs. The user can answer them in either order, but background agents should serialize requests to avoid flooding the screen. If you control the calling code, take a lock or run a queue.

## When in Doubt, Probe First

Before running the privileged action:

```bash
# Read-only inspection — no auth needed
/bin/ls -ld /Library/Application\ Support/MyTool
/usr/bin/grep -nF "MYLINE" /etc/paths
/usr/sbin/systemsetup -getremotelogin
/bin/launchctl print system/com.openssh.sshd
```

If a non-privileged read already shows the desired state, you do not need to ask for elevation at all. That is the most respectful outcome: the user keeps working without seeing a password dialog.
