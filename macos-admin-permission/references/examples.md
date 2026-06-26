# Worked Examples — `osascript ... with administrator privileges`

Copy-paste-ready templates for the most common privileged macOS operations. Every example follows the quoting rules in `references/quoting.md`: absolute command paths, `quoted form of` for dynamic values, `/bin/sh -c` for shell features, and `~`/`$HOME` resolved before elevation.

## Table of Contents

- [Append a line to `/etc/paths` (idempotent)](#append-a-line-to-etcpaths-idempotent)
- [Toggle SSH / Remote Login via `systemsetup`](#toggle-ssh--remote-login-via-systemsetup)
- [Restart a system service with `launchctl kickstart`](#restart-a-system-service-with-launchctl-kickstart)
- [Create a protected directory under `/Library`](#create-a-protected-directory-under-library)
- [Install a LaunchDaemon (copy plist + load)](#install-a-launchdaemon-copy-plist--load)
- [Edit `/etc/hosts` (idempotent block replace)](#edit-etchosts-idempotent-block-replace)
- [Multi-step script with `set -e`](#multi-step-script-with-set-e)
- [Copy a file from the user's home into a protected location](#copy-a-file-from-the-users-home-into-a-protected-location)

---

## Append a line to `/etc/paths` (idempotent)

Adds `/opt/homebrew/bin` only if it is not already present. Safe to re-run.

```bash
/usr/bin/osascript <<'APPLESCRIPT'
set lineToAdd to "/opt/homebrew/bin"
set scriptText to "/usr/bin/grep -qxF " & quoted form of lineToAdd & " /etc/paths || /bin/echo " & quoted form of lineToAdd & " >> /etc/paths"
do shell script ("/bin/sh -c " & quoted form of scriptText) with administrator privileges
APPLESCRIPT
```

Verify afterwards:

```bash
/usr/bin/grep -nF "/opt/homebrew/bin" /etc/paths
```

## Toggle SSH / Remote Login via `systemsetup`

`systemsetup` lives under `/usr/sbin` and needs admin for any `-set…` flag.

```bash
# Enable Remote Login (SSH)
/usr/bin/osascript -e 'do shell script "/usr/sbin/systemsetup -setremotelogin on" with administrator privileges'

# Disable Remote Login (SSH)
/usr/bin/osascript -e 'do shell script "/usr/sbin/systemsetup -setremotelogin off" with administrator privileges'
```

Verify:

```bash
/usr/sbin/systemsetup -getremotelogin
```

## Restart a system service with `launchctl kickstart`

`launchctl kickstart -k` restarts a service in the `system/` domain idempotently — prefer it over `bootout` + `bootstrap`.

```bash
/usr/bin/osascript -e 'do shell script "/bin/launchctl kickstart -k system/com.openssh.sshd" with administrator privileges'
```

To target a specific service, replace `system/com.openssh.sshd` with the correct `domain/label`. Verify with a read-only print first:

```bash
/bin/launchctl print system/com.openssh.sshd
```

## Create a protected directory under `/Library`

`/Library/Application Support/…` is owned by root. Use `mkdir -p` so the command is idempotent.

```bash
/usr/bin/osascript <<'APPLESCRIPT'
set targetPath to "/Library/Application Support/MyTool"
do shell script ("/bin/mkdir -p " & quoted form of targetPath) with administrator privileges
APPLESCRIPT
```

Optionally set ownership/permissions in the same step (see the multi-step script below). Verify:

```bash
/bin/ls -ld "/Library/Application Support/MyTool"
```

## Install a LaunchDaemon (copy plist + load)

Copies a plist from the user's home into `/Library/LaunchDaemons`, fixes ownership/permissions, then bootstraps it. The user-side path is resolved **before** elevation so `~` does not collapse to `/var/root`.

```bash
/usr/bin/osascript <<'APPLESCRIPT'
-- Resolve user-side path BEFORE elevating (root's ~ is /var/root)
set userHome to do shell script "printf %s \"$HOME\""
set sourcePlist to userHome & "/Downloads/com.example.mytool.plist"
set destPlist to "/Library/LaunchDaemons/com.example.mytool.plist"

set scriptText to "/bin/cp " & quoted form of sourcePlist & " " & quoted form of destPlist & " && /usr/sbin/chown root:wheel " & quoted form of destPlist & " && /bin/chmod 644 " & quoted form of destPlist & " && /bin/launchctl bootstrap system " & quoted form of destPlist

do shell script ("/bin/sh -c " & quoted form of scriptText) with administrator privileges
APPLESCRIPT
```

To later reload after editing the plist, use `kickstart -k` or `bootout` + `bootstrap`:

```bash
/usr/bin/osascript -e 'do shell script "/bin/launchctl bootout system /Library/LaunchDaemons/com.example.mytool.plist" with administrator privileges'
/usr/bin/osascript -e 'do shell script "/bin/launchctl bootstrap system /Library/LaunchDaemons/com.example.mytool.plist" with administrator privileges'
```

Verify:

```bash
/bin/launchctl print system/com.example.mytool
```

## Edit `/etc/hosts` (idempotent block replace)

Replaces (or appends) a managed block in `/etc/hosts` delimited by sentinel comments. Idempotent — re-running yields the same file content.

```bash
/usr/bin/osascript <<'APPLESCRIPT'
set block to "# BEGIN managed-by-mytool" & linefeed & "127.0.0.1    mytool.local" & linefeed & "# END managed-by-mytool"
set scriptText to "/usr/bin/awk -v block=" & quoted form of block & " 'BEGIN{p=1} /^# BEGIN managed-by-mytool/{p=0} /^# END managed-by-mytool/{p=0; next} p' /etc/hosts > /tmp/hosts.new && /bin/echo " & quoted form of block & " >> /tmp/hosts.new && /bin/mv /tmp/hosts.new /etc/hosts"

do shell script ("/bin/sh -c " & quoted form of scriptText) with administrator privileges
APPLESCRIPT
```

For a simpler append-only edit, mirror the `/etc/paths` pattern with `grep -qxF ... || echo ... >> /etc/hosts`. Verify:

```bash
/usr/bin/grep -nF "mytool.local" /etc/hosts
```

## Multi-step script with `set -e`

Bundle several related privileged steps into one authorization prompt. `set -e` aborts on the first failure so the authorization scope stays well-defined. Concatenate lines with `& linefeed &`.

```bash
/usr/bin/osascript <<'APPLESCRIPT'
set targetPath to "/Library/Application Support/MyTool"
set scriptText to "set -e" & linefeed & ¬
  "/bin/mkdir -p " & quoted form of targetPath & linefeed & ¬
  "/usr/sbin/chown root:admin " & quoted form of targetPath & linefeed & ¬
  "/bin/chmod 775 " & quoted form of targetPath & linefeed & ¬
  "/bin/launchctl kickstart -k system/com.example.mytool"

do shell script ("/bin/sh -c " & quoted form of scriptText) with administrator privileges
APPLESCRIPT
```

Because `with administrator privileges` authorization lasts about five minutes, batching related steps into one call also avoids re-prompting the user mid-sequence.

## Copy a file from the user's home into a protected location

Demonstrates Rule 4 from `references/quoting.md`: resolve `$HOME` as the user first, then elevate with a fully absolute path.

```bash
/usr/bin/osascript <<'APPLESCRIPT'
set userHome to do shell script "printf %s \"$HOME\""
set sourceFile to userHome & "/Downloads/example.plist"
set destFile to "/Library/LaunchDaemons/example.plist"

do shell script ("/bin/cp " & quoted form of sourceFile & " " & quoted form of destFile) with administrator privileges
APPLESCRIPT
```

Verify:

```bash
/bin/ls -l /Library/LaunchDaemons/example.plist
```
