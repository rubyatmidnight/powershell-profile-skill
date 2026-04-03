---
name: powershell-profile
description: >
  Use this skill when the user wants to create, improve, refactor, or debug a
  PowerShell $PROFILE file -- especially for Windows Terminal users who want
  shell shortcuts, aliases, SSH/SCP helpers, or app launchers. Trigger on any
  mention of $PROFILE, PowerShell profile, PS1 profile, shell aliases, Windows
  Terminal shortcuts, or requests to "add a shortcut/alias in PowerShell". Also
  trigger when the user pastes a profile and asks to clean it up, add verbosity,
  centralize config, or fix parse errors.
---

# PowerShell $PROFILE Skill

Guide for writing clean, maintainable PowerShell `$PROFILE` files -- useful for
anyone who wants quick shell shortcuts, SSH/SCP helpers, and app launchers in
Windows Terminal.

---

## Terminal Software

Depending on the user's usual method of accessing the terminal, some guidance may change.
This guide is based on Powershell, but more specifically, Powershell through the official 
[Windows Terminal](https://github.com/microsoft/terminal) application, which can be used
portable, too, and profiles persistently saved across installs. Some users may not
know about this much better, updated, and feature-rich version of the basic command terminal
or Powershell terminal in Windows; it can do both.  

## Design Philosophy

A good profile functions like a keyboard shortcut menu for the terminal. The
guiding question for every function you write is: **what is the minimum I have
to type before I am doing the thing I actually came here to do?**

This produces a few concrete principles:

**The terminal is a launchpad, not a destination.** Opening a new window should
get you to your working context in under five seconds. If you find yourself
typing the same `cd` path every morning, that path belongs in a function.

**We're in the terminal all the time, and you built applications that stay inside it.**
So optimize those flows and make it even faster than clicking away. 

**Collapse multi-step tasks into one word.** Anything that requires looking up
an IP, recalling a key path, or constructing a long command from parts is a
candidate for a function. The function remembers the boilerplate so you do not
have to. `fetchdb` beats `scp -r -i ~/.ssh/myKey alice@1.2.3.4:~/app/db .`
every time.

**Name functions for what you are doing, not what they are.** You will type
these from muscle memory, so they should feel like verbs you reach for
instinctively. `sshprod` not `connectToProductionServer`. `fetchlogs` not
`scpLogsFromServer`. Short, unambiguous, action-oriented.

**The shortcuts list is a menu, not documentation.** It exists so you can open
a terminal, type `shortcuts`, and immediately remember which one-word command
does the thing you want. If a function is too obscure to appear on the list, it
probably needs a better name.

**Automate the lookup, not just the typing.** The best functions eliminate a
cognitive step, not just keystrokes. A function that still requires you to
remember a variable or check a config before calling it has not done its job.

When reviewing or refactoring a profile, ask of each function: could a new
terminal session get from zero to working faster? If yes, the function needs
work. If no, it is earning its place.

## Secrets and Sensitive Values

The config block deliberately centralises values like IPs and key paths so
they are easy to update -- but that also means they are easy to read. A few
things worth keeping in mind:

**Tier 1 -- convenience values** (IPs, key filenames, usernames): low-to-medium
sensitivity. Fine in the profile as long as the machine is not shared and the
file is not committed to a public repo. If an IP is directly exposed to the
public internet with no firewall, treat it as higher sensitivity.

**Tier 2 -- actual secrets** (passwords, API tokens, private keys): never in
the profile. These belong in Windows Credential Manager or as user environment
variables, then referenced in the profile as `$env:MY_SECRET` rather than
hardcoded values.

**Version control**: if you keep your profile in git, use a private repo or
maintain a sanitised template version for sharing. A public repo with real IPs
and key names is a meaningful fingerprint even if nothing is immediately
exploitable.

**Shell history**: `Set-ProfileVar` calls are logged to PowerShell's history
file. If you use it to set a sensitive value, consider clearing the relevant
entry from `(Get-PSReadlineOption).HistorySavePath` afterward.


---

## Profile Anatomy

A well-structured profile follows this layout:

```
1. Config block        - IPs, paths, keys (single place to update)
2. Internal helpers    - reusable _verb functions
3. Shortcuts reference - a function that prints available commands
4. SSH functions
5. SCP / data transfer
6. App / repo launchers
7. System utilities
8. Profile editing helpers
9. Deprecated (clearly marked)
10. .SYNOPSIS comment block
```

---

## Config Block Pattern

Put all environment-specific values at the top in a config block. Never
hardcode IPs, key paths, or directories inside individual functions -- if
anything changes, you want to update it in exactly one place.

```powershell
# ============================================================
#  CONFIG -- edit these, not the functions below
# ============================================================
$SERVER_IP   = "1.2.3.4"
$SERVER_KEY  = "$HOME\.ssh\myKey"
$SERVER_USER = "alice"

$APP_DIR  = "C:\repos\myapp"
$LOGS_DIR = "C:\logs"
```

### Updating config values from the terminal

Rather than opening the profile file every time a value changes, you can add
a `Set-ProfileVar` function that does an in-place text replacement in the
profile file itself:

```powershell
function Set-ProfileVar {
    param(
        [Parameter(Mandatory)][string]$Name,
        [Parameter(Mandatory)][string]$Value
    )
    $content = Get-Content $PROFILE -Raw
    # Matches lines of the form: $NAME = "anything"
    $pattern     = '(\$' + [regex]::Escape($Name) + '\s*=\s*)"[^"]*"'
    $replacement = '$1"' + $Value + '"'
    if ($content -notmatch $pattern) {
        Write-Warning "Variable `$$Name not found in profile."
        return
    }
    $content -replace $pattern, $replacement | Set-Content $PROFILE -Encoding UTF8BOM
    Write-Host "Updated `$$Name = `"$Value`" -- reload with: . `$PROFILE"
}
```

Usage:
```powershell
Set-ProfileVar SERVER_IP "5.6.7.8"
Set-ProfileVar SERVER_KEY "$HOME\.ssh\newKey"
```

This only matches variables in the `$VAR = "value"` form used in the config
block, so it won't accidentally overwrite the same variable name used inside
a function body. After running it, reload with `. $PROFILE` for the new value
to take effect in the current session.

---

## Internal Helper Pattern

Extract repeated patterns into `_verb` prefixed private helpers. The
underscore prefix is a convention signalling "not a user-facing command".

```powershell
# Navigate to a directory and print a message
function _nav {
    param([string]$Path, [string]$Msg)
    Set-Location $Path
    if ($Msg) { Write-Host $Msg }
}

# Reusable SCP wrapper -- avoids repeating key/user/host on every transfer function
function _scp {
    param(
        [string]$Key,
        [string]$User,
        [string]$RemoteHost,
        [string]$Remote,
        [string]$Local,
        [switch]$Upload
    )
    if ($Upload) {
        scp -r -i $Key "$Local" "${User}@${RemoteHost}:${Remote}"
    } else {
        scp -r -i $Key "${User}@${RemoteHost}:${Remote}" "$Local"
    }
}
```

---

## Alias Patterns

Use `Set-Alias` to point multiple names at the same function rather than
duplicating the function body. Sometimes, in a rush, we may think of a similar
word or command when it's actually something near, but not exactly that, 
such as 'text' 'editor' 'word' 'sublime' 'notepad' 'wordpad' etc. We may
want the same text editor to appear regardless of what we write. Tailor to the
user's expectations and concerns.

```powershell
Set-Alias shortcut  shortcuts
Set-Alias hotkeys   shortcuts
```

For executable files:
```powershell
Set-Alias myterm "C:\path\to\terminal.exe"
```

---

## Shortcuts Display Function

A `shortcuts` function gives you a quick reference of what your profile
provides. Keep it grouped and aligned -- it only needs to be readable at a
glance, not exhaustive.

```powershell
function shortcuts {
    Write-Host "+-------- SSH --------------------------+"
    Write-Host "| sshprod   : Connect to prod server   |"
    Write-Host "+-------- Data -------------------------+"
    Write-Host "| fetchdb   : Pull latest DB backup    |"
    Write-Host "| fetchlogs : Pull archived logs       |"
    Write-Host "+-------- Apps -------------------------+"
    Write-Host "| myapp     : Launch my app            |"
    Write-Host "+---------------------------------------+"
}
Set-Alias shortcut shortcuts
Set-Alias hotkeys  shortcuts
```

The format is flexible. What matters is that related commands are visually
grouped, descriptions fit on one line, and you can scan the whole thing in
under five seconds.

---

## Verbosity in Functions

Each function should print enough that you know what it is doing and where
things ended up, without becoming noise. One line at the start, one line at
the end if a destination is involved. That way, if something like a file is
downloaded, it will say exactly where it was saved, rather than just saying
it was downloaded without saying where it was saved.

```powershell
function fetchdb {
    $dest = "$DATA_DIR\backup_$(Get-Date -Format 'MMdd').db"
    Write-Host "Fetching database to $dest ..."
    _scp -Key $SERVER_KEY -User $SERVER_USER -Host $SERVER_IP `
         -Remote "~/app/data/app.db" -Local $dest
    Write-Host "Done."
}
```

For functions that just navigate or launch something, a single line is enough:

```powershell
function myapp {
    _nav $APP_DIR "Opening app repo."
}
```

Use `Write-Host` rather than `echo`. `echo` is an alias for `Write-Output`,
which writes to the PowerShell pipeline rather than directly to the console,
and can cause unexpected behaviour when its output gets captured by something
else in the session.

---

## SSH Function Template

```powershell
function sshprod {
    Write-Host "Connecting to $SERVER_USER@$SERVER_IP ..."
    try { ssh -i $SERVER_KEY "${SERVER_USER}@${SERVER_IP}" }
    catch [System.SystemException] {}
}
```

The `try/catch [System.SystemException]` suppresses the exit-code error
PowerShell prints when an SSH session closes normally.

---

## SCP Function Template

```powershell
function fetchdb {
    $currdate = Get-Date -Format "MMdd"
    $dest = "$DATA_DIR\backup$currdate.db"
    Write-Host "Fetching db to $dest ..."
    _scp -Key $SERVER_KEY -User $SERVER_USER -Host $SERVER_IP `
         -Remote "~/app/data/app.db" -Local $dest
    Write-Host "Done."
}
```

For log pulls where the destination directory needs to be created first:

```powershell
function fetchlogs {
    $timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
    $targetDir = "$LOGS_DIR\$timestamp"
    try {
        New-Item -ItemType Directory -Path $targetDir -ErrorAction Stop | Out-Null
    } catch {
        $targetDir = "$LOGS_DIR\tmp"
        New-Item -ItemType Directory -Path $targetDir -Force | Out-Null
        Write-Warning "Could not create timestamped folder. Using: $targetDir"
    }
    Write-Host "Fetching logs to $targetDir ..."
    _scp -Key $SERVER_KEY -User $SERVER_USER -Host $SERVER_IP `
         -Remote "~/app/logs/*" -Local $targetDir
    Write-Host "Done."
}
```

`New-Item` throws if the directory already exists without `-Force`, so the
try/catch prevents a crash if the function is run twice in the same second.

---

## Profile Editing Helpers

```powershell
function profile {
    Write-Host "Opening profile..."
    Start-Process "C:\path\to\editor.exe" $PROFILE
}

function profiledebug {
    Write-Host "Opening profile in Notepad..."
    notepad $PROFILE
}
```

Having both is useful: your preferred editor for normal edits, Notepad as a
fallback that always works regardless of what else is installed.

---

## Deprecated Function Template

Do not silently delete old functions -- mark them visibly and leave breadcrumbs:

```powershell
function old-command {
    Write-Warning "DEPRECATED: old-command -- use new-command instead."
    Write-Host "Manual equivalent if needed: some-command --flag value"
}
```

---

## ⚠️ Known Pitfalls

### 1. `-` operator inside double-quoted strings

PowerShell parses `-` as an operator before resolving string context. Any
`-word` token inside a double-quoted string will cause a parse error.

**Broken:**
```powershell
Write-Host "Copying $src -> $dest"   # -> causes parser error
Write-Host "Count: $n -gt 0"        # -gt causes parser error
```

**Safe alternatives:**
```powershell
Write-Host "Copying $src to $dest"
Write-Host "Copying $src => $dest"
```

This affects `->`, `-->`, and any `-flag` style literal inside an interpolated
string. Parameters passed to functions (e.g. `_scp -Key ...`) are fine because
they are not inside a string.

### 2. Non-ASCII characters and encoding

PowerShell 5.x (the Windows default) expects profile files saved as
**UTF-8 with BOM**. If your editor saves UTF-8 without BOM, non-ASCII
characters like em-dashes, curly quotes, or smart apostrophes will be
corrupted at load time. This typically manifests as:

- Garbled characters (e.g. `a-Euro-`) in `(Get-Command foo).Definition` output
- Misleading "missing `}`" parse errors far from the actual problem line
- Functions swallowing subsequent function definitions into their body

**Fix:** Either save the file with UTF-8 BOM (most editors support this under
File > Save with Encoding), or avoid non-ASCII punctuation entirely. Sticking
to plain ASCII is the most portable option and won't be corrupted by editors,
git, or file transfers.

### 3. Cascade parse failures

When PowerShell hits a parse error mid-function, it often fails to find the
closing `}` for that function. All subsequent function definitions then get
swallowed into the broken function's body. The reported error line is usually
a symptom, not the root cause.

**Diagnostic:**
```powershell
(Get-Command suspectFunction).Definition
```

If the output includes the bodies of other functions, search upward from the
reported error line for the real cause.

### 4. `START` vs `Start-Process`

`START` is a cmd.exe builtin that PowerShell inherits. It works inconsistently
with paths containing spaces and does not support PowerShell error handling.
Prefer the native cmdlet:

```powershell
Start-Process "C:\Program Files\My App\app.exe"
```

### 5. `cd` vs `Set-Location`

`cd` is an alias for `Set-Location`. Both work, but `Set-Location` is the
idiomatic form in profile scripts and will not be affected if someone redefines
the `cd` alias elsewhere.

### 6. Reserved automatic variable names in parameters

PowerShell reserves certain variable names as automatic variables —
`$Host`, `$Error`, `$Input`, `$Output`, `$Matches`, `$Args`, and others.
Using any of these as a `param()` name will throw: 
`Cannot overwrite variable Host because it is read-only or constant.`

If a parameter name feels generic and single-word, check it against
PowerShell's automatic variables list first. Prefer specific names:
`$RemoteHost` instead of `$Host`, `$ErrMsg` instead of `$Error`.

---

## Debugging Checklist

When the profile fails to load or a function misbehaves:

1. Run `(Get-Command funcName).Definition` -- if the output contains other
   functions' bodies, there is a parse error somewhere above that function
2. Search the file for `->`, `-->`, or any `-word` literals inside strings
3. Search for non-ASCII punctuation (em-dashes, curly quotes, smart apostrophes)
4. Check the file encoding and re-save as UTF-8 with BOM if unsure
5. Reload the profile in the current session with `. $PROFILE`
