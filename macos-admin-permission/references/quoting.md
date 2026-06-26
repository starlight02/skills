# Quoting Reference — `do shell script` and AppleScript

Quoting is the dominant source of failures with `osascript ... with administrator privileges`. Two languages are layered on top of each other (shell ↔ AppleScript), and `do shell script` runs commands with a minimal environment. This document is the source of truth for the quoting rules so you do not need to rediscover them under time pressure.

## Choose the Right Container First

| Situation | Container |
|---|---|
| Single fixed command, no variables, no special characters | Inline `-e` |
| Paths with spaces, user-provided values, anything dynamic | Heredoc with `quoted form of` |
| Shell redirection (`>`, `>>`), pipes, `&&`, `\|\|`, `set -e` | Heredoc that wraps in `/bin/sh -c` |

When in doubt, pick the heredoc form. Inline `-e` strings tempt brittle hand-escaping.

## Rule 1: `quoted form of` Before Concatenation

Always wrap user-controlled or path-shaped values with `quoted form of` before concatenating them into the script string:

```applescript
set userPath to "/Users/anya/Downloads/file with space.plist"
do shell script ("/bin/cp " & quoted form of userPath & " /Library/LaunchDaemons/") with administrator privileges
```

`quoted form of` produces a shell-safe single-quoted form (and escapes embedded single quotes), so the resulting shell command is correct regardless of what `userPath` contains.

## Rule 2: Use Absolute Paths for System Commands

`do shell script` runs with a minimal `PATH`. Commands like `grep`, `mkdir`, `chmod`, `chown`, `cp`, `systemsetup`, `launchctl` should be invoked by their full path:

```
/usr/bin/grep
/bin/mkdir
/bin/chmod
/usr/sbin/chown
/bin/cp
/usr/sbin/systemsetup
/bin/launchctl
```

This avoids "command not found" failures that depend on the user's `PATH`.

## Rule 3: Wrap Compound Commands in `/bin/sh -c`

`do shell script` runs each command directly — there is no shell parsing of `&&`, `||`, `>`, `>>`, or `|`. To use shell features, pass the whole script to a shell explicitly:

```applescript
set scriptText to "/usr/bin/grep -qxF " & quoted form of line & " /etc/paths || /bin/echo " & quoted form of line & " >> /etc/paths"
do shell script ("/bin/sh -c " & quoted form of scriptText) with administrator privileges
```

The pattern is:

1. Build the shell script as a single AppleScript string.
2. `quoted form of` that string.
3. Pass it as the argument to `/bin/sh -c`.

Wrap the entire `/bin/sh -c ...` expression in parentheses before the `with administrator privileges` clause when the right-hand side is a concatenation. The parentheses keep AppleScript from misparsing the call.

## Rule 4: Resolve `~` and `$HOME` **Before** Elevating

Once `do shell script ... with administrator privileges` runs, the effective user is root. `~` and `$HOME` resolve to `/var/root`, which is almost never what the user meant. Resolve the user-side path **first**:

```applescript
set userHome to do shell script "printf %s \"$HOME\""
set sourcePath to userHome & "/Downloads/example.plist"
do shell script ("/bin/cp " & quoted form of sourcePath & " /Library/LaunchDaemons/") with administrator privileges
```

The first `do shell script` runs as the user (no `with administrator privileges`), so `$HOME` is correct. The second runs as root with a fully resolved absolute path.

## Rule 5: Escape Embedded Quotes Carefully

When you must embed a double quote inside a single-quoted shell string inside an AppleScript double-quoted string, use AppleScript's character escape:

```applescript
do shell script "printf %s \"$HOME\""
```

The `\"` is the AppleScript escape for a double-quote character. Inside `do shell script`, that becomes `printf %s "$HOME"` which is what `/bin/sh` actually sees.

For more than one level of nesting, switch to a heredoc form and use `quoted form of` instead of hand-escaping.

## Rule 6: Multi-line Scripts Use `& linefeed & ¬`

To pass several lines to `/bin/sh -c`, concatenate with `& linefeed & ¬`:

```applescript
set scriptText to "set -e" & linefeed & ¬
  "/bin/mkdir -p /Library/Application\\ Support/MyTool" & linefeed & ¬
  "/bin/chmod 755 /Library/Application\\ Support/MyTool"
do shell script ("/bin/sh -c " & quoted form of scriptText) with administrator privileges
```

`linefeed` is the AppleScript newline character. `¬` is the AppleScript line-continuation marker (option-L on a US keyboard).

## Rule 7: `with administrator privileges` Is the Privilege Mechanism

Do **not** prepend `sudo` inside the command. The AppleScript authorization dialog is what asks the user for their password and elevates the wrapped command. Adding `sudo` inside the script does nothing useful and often produces `sudo: a terminal is required`.

```applescript
-- correct
do shell script "/usr/sbin/systemsetup -setremotelogin on" with administrator privileges

-- wrong
do shell script "sudo /usr/sbin/systemsetup -setremotelogin on"
```

## Rule 8: Idempotent by Default

Prefer commands that are safe to re-run:

- Check before append: `grep -qxF "$LINE" $FILE || echo "$LINE" >> $FILE`
- `mkdir -p` instead of `mkdir`
- `chmod` / `chown` are idempotent on their own
- For service control, use `launchctl kickstart -k` to restart instead of `bootout` + `bootstrap`

Idempotent commands let you retry safely after partial failures or after a user cancels and re-approves.

## Quick Checklist

Before invoking `osascript`:

- Is the target path exact and verified by a non-privileged probe?
- Does the command avoid `sudo`?
- Are user-provided values wrapped with `quoted form of`?
- Are absolute command paths used where useful?
- Is there a non-privileged verification command ready?

After `osascript` returns:

- Check stdout, stderr, and the AppleScript exit code.
- If exit code is `-128`, the user cancelled — stop and ask.
- Otherwise, run the verification command and summarize the observed result.
