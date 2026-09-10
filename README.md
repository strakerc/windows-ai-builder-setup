# Windows PowerShell + Claude Code Setup

**Straker's Windows setup.** Written up after doing it the hard way — the order
and the warnings here exist because each one cost time.

A reference for setting up Claude Code, Oh My Posh, Vim, less, wrangler,
OpenCode and directory-restoring prompts on Windows — and for avoiding the
trap that makes this take an hour instead of ten minutes.

### Scope and assumptions

- **Windows 10/11 with Windows Terminal.** Not WSL — if your projects live in
  WSL, install Claude Code inside the WSL distro instead and most of this
  doesn't apply.
- **Paths use `$env:USERPROFILE` and `$PROFILE`** rather than literal usernames,
  so commands are copy-pasteable as written.
- **If your Documents folder is redirected to OneDrive**, profiles land under
  `OneDrive\Documents\...` instead of `Documents\...`. This works fine but syncs
  across machines — worth knowing before you put anything machine-specific in a
  profile. `$PROFILE` resolves correctly either way.
- Written against Claude Code v2.1.x. Version-specific notes are flagged inline.

---

## Prerequisites

Install these first. Skip any you already have.

### Windows Terminal

Ships with Windows 11. On Windows 10, install from the Microsoft Store or:

```powershell
winget install Microsoft.WindowsTerminal
```

### PowerShell 7

```powershell
winget install Microsoft.PowerShell
```

This installs *alongside* Windows PowerShell 5.1 rather than replacing it — see
the next section, which is the single most important thing in this document.

### Git for Windows

```powershell
winget install Git.Git
```

Accept the default "Git from the command line and also from 3rd-party software"
PATH option. Claude Code uses Git Bash for its Bash tool; without Git it falls
back to the PowerShell tool instead.

Verify with `git --version` in a **new** terminal.

### GitHub CLI

```powershell
winget install GitHub.cli
```

Unlike Vim in Part 4, you don't have to touch PATH for this one. The MSI adds
`C:\Program Files\GitHub CLI\` to the **machine** PATH itself. Verify before you
edit anything:

```powershell
([Environment]::GetEnvironmentVariable('PATH','Machine') -split ';') -match 'GitHub'
```

If that prints the path, PATH is fine and `gh` failing means something else —
see [Why a PATH change doesn't take effect](#why-a-path-change-doesnt-take-effect)
below. Adding a second copy at user scope fixes nothing and leaves you with a
duplicate.

Then, in that **new** terminal:

```powershell
gh auth login
```

Answer **GitHub.com** → **HTTPS** → **Y** to authenticate Git with your GitHub
credentials → **Login with a web browser**. Copy the one-time code it prints,
press Enter, paste the code in the browser tab that opens.

The third answer is the one worth getting right — saying yes registers `gh` as
Git's credential helper, so `git push` stops prompting and you never store a
personal access token in a config file.

`gh auth login` is interactive. It can't be run through Claude Code, the VS Code
extension, or any non-interactive shell — the prompts hit EOF and it fails or
hangs. Run it yourself in a real terminal tab, once.

Verify with `gh auth status`. It reports the account, the protocol, and where
the token is kept — on Windows that's `keyring`, meaning Windows Credential
Manager rather than a file on disk. (Wrangler in Part 6 does the opposite and
writes a token to a config file, which is why that one has a path worth
knowing.)

Claude Code uses `gh` for anything GitHub-side — PRs, issues, releases — so
without this step it can commit locally but can't open a PR.

### Node.js

```powershell
winget install OpenJS.NodeJS.LTS
```

Installs to `C:\Program Files\nodejs` and puts `node`, `npm` and `npx` on the
machine PATH. Verify with `node --version` in a **new** terminal.

The Mac guide uses nvm to juggle versions; the Windows equivalent is
`winget install CoreyButler.NVMforWindows`. Only worth it if your projects pin
different Node versions — a single LTS install covers Claude Code and wrangler
fine.

One thing to know now, because it surfaces later as a baffling error: `npm` and
`npx` resolve to **`npm.ps1` and `npx.ps1`**, PowerShell shims. If you skip
Part 2 Step 2, they're blocked by the execution policy and every `npx` call dies
with *"running scripts is disabled on this system"* — while `node` itself keeps
working, which makes it look like an npm problem.

### Oh My Posh

```powershell
winget install JanDeDobbeleer.OhMyPosh -s winget
```

Use winget, not `Install-Module`. The PowerShell Gallery module installs to a
5.1-scoped module path and won't be available in PowerShell 7. The winget
install puts a binary on PATH that both editions can reach.

Reopen your terminal, then verify with `oh-my-posh --version`.

---

## Read this first: there are two PowerShells

Windows ships with **Windows PowerShell 5.1** (`powershell.exe`). Microsoft
froze it years ago for backward compatibility and it is still installed on
every Windows machine.

**PowerShell 7** (`pwsh.exe`) is a separate program you install yourself. It is
the modern, actively developed version.

They are installed side by side and **share almost nothing**:

| Thing | Windows PowerShell 5.1 | PowerShell 7 |
|---|---|---|
| Executable | `powershell.exe` | `pwsh.exe` |
| Profile folder | `Documents\WindowsPowerShell\` | `Documents\PowerShell\` |
| Execution policy | Set separately | Set separately |
| Installed modules | Separate path | Separate path |

This separation is deliberate — upgrading one can't break scripts that depend on
the other. But it means **every setup step below has to be done twice**, once in
each shell, or you get a working setup in one and a broken one in the other.

Windows Terminal lists both as profiles. Confusingly, the profile named
"PowerShell" is version 7, and the one named "Windows PowerShell" is 5.1.

### How to tell which one you're in

```powershell
$PSVersionTable.PSVersion
```

`5.1.x` → Windows PowerShell. `7.x` → PowerShell 7.

Also: `$PROFILE` always resolves to the profile path for whichever shell you're
currently running. It is a variable PowerShell fills in, not a setting you
change. Print it any time you're unsure which file you're about to edit:

```powershell
$PROFILE
```

### The recommendation

**Set up both.** It's the same three commands and the same paste each time. The
alternative is discovering months later that your terminal behaves differently
depending on which tab you opened.

---

## Part 1: Install Claude Code

From PowerShell:

```powershell
irm https://claude.ai/install.ps1 | iex
```

Or from CMD:

```batch
curl -fsSL https://claude.ai/install.cmd -o install.cmd && install.cmd && del install.cmd
```

WinGet also works: `winget install Anthropic.ClaudeCode`

### Add it to PATH

The installer usually tells you this is needed. It puts `claude.exe` in
`%USERPROFILE%\.local\bin`, which isn't on PATH by default.

```powershell
$p = [Environment]::GetEnvironmentVariable('PATH','User')
[Environment]::SetEnvironmentVariable('PATH', "$p;$env:USERPROFILE\.local\bin", 'User')
```

Or via GUI: System Properties → Environment Variables → User `Path` → Edit →
New → paste the path.

**Close and reopen your terminal.** The running session won't pick up a PATH
change.

Verify:

```powershell
claude --version
```

### Git Bash detection

Claude Code uses Git Bash for the Bash tool (installed in Prerequisites above).
If it can't find it, set the path in `~/.claude\settings.json`:

```json
{
  "env": {
    "CLAUDE_CODE_GIT_BASH_PATH": "C:\\Program Files\\Git\\bin\\bash.exe"
  }
}
```

Note the escaped backslashes. Only do this if you actually see the
"requires git-bash" error — don't set it preemptively.

**Known issue:** the desktop app and VS Code extension sometimes fail to detect
Git Bash even with this set correctly, while the CLI works fine. If that
happens, it's a known bug, not your configuration.

---

## Part 2: Create your profile files

Do this **once per shell**. Open a tab of each and run the same commands.

### Step 1 — Create the file

```powershell
New-Item -ItemType File -Path $PROFILE -Force
```

`-Force` is doing real work here: it creates the missing parent folder, not just
the file. Without it, `notepad $PROFILE` fails with **"The system cannot find
the path specified"** because Notepad can create files but not folders.

### Step 2 — Allow scripts to run

```powershell
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
```

Answer `Y`. Without this you get
**"cannot be loaded because running scripts is disabled on this system"** on
every new tab.

`RemoteSigned` runs local scripts you wrote, but requires signatures on anything
downloaded. No admin rights needed at `CurrentUser` scope.

**This is per-edition.** Setting it in 5.1 does nothing for 7.

### Step 3 — Open it

```powershell
notepad $PROFILE
```

### Step 4 — Paste this

Identical content works in both shells:

```powershell
# Like Oh My ZSH but for powershell
oh-my-posh init pwsh | Invoke-Expression

# Keep oh-my-posh's prompt, and also tell Windows Terminal our directory
$Global:__OriginalPrompt = $function:Prompt

function prompt {
    $esc = [char]27
    Write-Host -NoNewline ($esc + ']9;9;"' + (Convert-Path $PWD) + '"' + $esc + '\')
    & $Global:__OriginalPrompt
}
```

Save and close.

### Step 5 — Reload without opening a new tab

```powershell
. $PROFILE
```

---

## Why the prompt function looks like that

Two non-obvious things are going on.

### It wraps Oh My Posh instead of replacing it

`oh-my-posh init` defines a `prompt` function. If you define your own `prompt`
afterward, **yours silently replaces it** and Oh My Posh stops rendering — no
error, you just get a plain prompt and assume the install failed.

Most guides online show a standalone `prompt` function that does exactly this.
The fix is to stash Oh My Posh's prompt in `$Global:__OriginalPrompt` and call
it at the end of yours.

Alternative: skip the custom function entirely and add `"pwd": "osc99"` to your
Oh My Posh theme JSON, which makes Oh My Posh emit the sequence itself. Cleaner,
but requires forking a theme file, so the wrapper is easier if you're using a
stock theme.

### It uses `[char]27`, not `` `e ``

The `` `e `` escape for the ESC character **was added in PowerShell 6**. In
Windows PowerShell 5.1 it emits a literal `e`, so your prompt prints garbage
like:

```
e]9;9;"C:\Users\you"e\PS C:\Users\you>
```

`[char]27` works in both editions. Most guides assume PowerShell 7 and use
`` `e ``.

### What the escape sequence does

`OSC 9;9` is how a shell tells Windows Terminal its current directory. Terminal
can't see it otherwise, which is why new tabs, split panes, and restored
sessions all default to your home folder. This one line makes all three
inherit the directory you were actually in.

---

## Part 3: Windows Terminal settings

### Restore your session

Settings → **Startup** → **When Terminal starts** → *Open windows from a
previous session*.

This restores window layout, tabs, and splits. Combined with the OSC 9;9 prompt
above, it also restores the directory each tab was in.

### Nerd Font (for Oh My Posh icons)

Oh My Posh themes use glyphs that standard fonts don't carry. Without a Nerd
Font you get diamonds or boxes where icons should be.

```powershell
oh-my-posh font install CascadiaCode
```

Then check the output — **it prints the exact family names it registered**, and
they are not what you'd guess. Cascadia Code installs as:

- `CaskaydiaCove NF` ← use this one
- `CaskaydiaCove NFM` (mono glyphs)
- `CaskaydiaCove NFP` (proportional)

Nerd Fonts must rename patched fonts due to reserved-name licensing, so
"Cascadia Code" becomes "CaskaydiaCove". Typing "CaskaydiaCove Nerd Font" will
not match anything.

Set it in Settings → **Profiles → Defaults** → Appearance → Font face. Use
**Defaults**, not an individual profile, so both shells get it.

**Then fully quit Terminal** — every window, check the taskbar for minimized
ones — and reopen. Terminal enumerates fonts at launch, and restored tabs keep
running with the old font until the app itself restarts.

Test glyphs in a fresh tab:

```powershell
"`u{f09b} `u{e0b0} `u{f015}"
```

GitHub logo, solid triangle, house. Boxes mean the font isn't applying.

### Color scheme

Settings → **Profiles → Defaults** → Appearance → **Color scheme**. Windows
Terminal ships with Solarized Dark, Solarized Light, One Half Dark, Tango, and
others built in — no download or config file needed.

Set it on **Defaults** so every shell inherits it, then Save.

This changes the background and the 16 ANSI colors. Oh My Posh segment colors
are separate — they're hardcoded hex values in the theme JSON and won't follow
the terminal scheme. In practice the stock themes usually look fine against a
new background, so **change the scheme first and only touch the theme if
something actually clashes.**

If you do need to recolor the prompt, fork a theme rather than editing the
bundled one:

```powershell
cp "$env:POSH_THEMES_PATH\jandedobbeleer.omp.json" "$env:USERPROFILE\mytheme.omp.json"
```

Then point the init line at it in **both** profiles:

```powershell
oh-my-posh init pwsh --config "$env:USERPROFILE\mytheme.omp.json" | Invoke-Expression
```

---

## Part 4: Vim and less

Both of these are already on your machine in some form, and both are invisible
from PowerShell for the same reason: **Git for Windows deliberately keeps its
Unix tools off the Windows PATH.** Git Bash tabs see them, PowerShell tabs
don't, which makes it look like your install is broken when it isn't.

### Vim

```powershell
winget install vim.vim
```

The installer does **not** add Vim to PATH. A user-scope install lands in
`%LOCALAPPDATA%\Programs\Vim`; an admin install lands in
`C:\Program Files\Vim\vim92`. Check which you got before pasting:

```powershell
$env:LOCALAPPDATA + '\Programs\Vim\vim.exe' | Get-Item -ErrorAction SilentlyContinue
```

Then add that folder, same pattern as Claude Code in Part 1:

```powershell
$p = [Environment]::GetEnvironmentVariable('PATH','User')
[Environment]::SetEnvironmentVariable('PATH', "$p;$env:LOCALAPPDATA\Programs\Vim", 'User')
```

**Close and reopen your terminal**, then `vim --version`.

Git Bash ships its own separate Vim at `/usr/bin/vim`, which is why `vim` can
work in one tab and be "not recognized" in the next. Two different builds — but
they read the same `~/.vimrc`, so you only configure once.

### Vim configuration

Same as the Mac setup: don't hand-roll a vimrc, use
[amix/vimrc](https://github.com/amix/vimrc). From a **Git Bash** tab (the
installer is a shell script):

```bash
git clone --depth=1 https://github.com/amix/vimrc.git ~/.vim_runtime
sh ~/.vim_runtime/install_awesome_vimrc.sh
```

All that script does is write a `~/.vimrc` that sources the runtime — if you'd
rather not run a downloaded script, open it and paste the `.vimrc` it echoes.

Then your own config on top:

```bash
vim ~/.vim_runtime/my_configs.vim
```

```vim
" Start NERDTree on startup with cursor in the file, not the file explorer
autocmd VimEnter * NERDTree | wincmd p

" Start NERDTree on the left
let g:NERDTreeWinPos = "left"

" Exit Vim if NERDTree is the only window remaining in the only tab.
autocmd BufEnter * if tabpagenr('$') == 1 && winnr('$') == 1 && exists('b:NERDTree') && b:NERDTree.isTabTree() | call feedkeys(":quit\<CR>:\<BS>") | endif
```

**The Windows-only trap:** Vim checks `$HOME\_vimrc` *before* `$HOME\.vimrc`.
The amix installer writes `.vimrc`. If a `_vimrc` exists in your home folder —
some Windows installers create one — it wins silently and every edit you make
appears to do nothing. Check with `ls $env:USERPROFILE\_vimrc` and delete it.

Vim resolves `~` from `%HOME%` if set, otherwise `%HOMEDRIVE%%HOMEPATH%`
(`C:\Users\you`). PowerShell doesn't set `HOME` and that's fine — the fallback
lands in the same place. Don't set `HOME` elsewhere to tidy up, or Vim stops
finding your config.

### less

Git for Windows already includes less, at
`C:\Program Files\Git\usr\bin\less.exe`. Resist adding that folder to PATH: it
carries the entire MSYS toolset, so `find`, `sort`, `tee` and friends start
shadowing the Windows commands of the same name and scripts break in
hard-to-trace ways.

Shim just the one tool instead. `%USERPROFILE%\.local\bin` is already on your
PATH from Part 1, so drop a `less.cmd` there:

```powershell
notepad $env:USERPROFILE\.local\bin\less.cmd
```

```batch
@echo off
rem less for PowerShell / cmd, borrowed from Git for Windows.
rem TERM and LESS are set locally so nothing leaks into the parent shell.
setlocal
set "TERM=xterm-256color"
if "%LESS%"=="" set "LESS=-R"
"%ProgramFiles%\Git\usr\bin\less.exe" %*
```

Reopen your terminal and check with `less --version`.

Two lines earn their keep. Without `TERM`, MSYS less greets you with *"WARNING:
terminal is not fully functional"* on every invocation. Without `LESS=-R`, ANSI
colors arrive as `ESC[31m` litter. `setlocal` scopes both to the shim so your
shell environment stays clean.

Note this is only for *your* pagers — `Get-Content big.log | less`, `less
file.txt`. Git has always paged fine, because it uses its own bundled copy
through an internal PATH rather than yours.

---

## Part 5: Claude Code and project folders

Claude Code has no separate "create a project" step — **the working directory is
the project**. Open a terminal in your repo and run `claude`, or in the desktop
app's Code tab pick the folder.

Run `/init` in a new session. Claude scans the repo and writes a `CLAUDE.md` at
the root with structure, commands, and conventions. Review it and commit it — it
loads at the start of every future session in that folder.

**No length limit.** Length is a judgement call, not a rule — a long `CLAUDE.md`
costs context every session, but a hard-won correction that isn't written down
costs more than the tokens do. Write what the next session actually needs.

If you do want a second opinion on trimming one, `/doctor` proposes cuts for a
checked-in `CLAUDE.md` on v2.1.206+. Treat it as a suggestion.

### Cowork projects

Cowork projects and Claude Code sessions are separate features — Claude Code
can't open a Cowork project directly, and project instructions and project
memory don't carry over. What connects them is the **folder**: point both at the
same directory and use `CLAUDE.md` as shared context.

Don't run both against the same folder simultaneously; neither knows about the
other's edits. Use the **worktree** toggle in the app to isolate a session if
you need parallel work.

---

## Part 6: Cloudflare Workers with wrangler

Wrangler is Cloudflare's CLI for Workers. It's what moves a Worker out of the
dashboard and into a repo — deploy, tail live logs, manage secrets and bindings
from the terminal — which is also what makes a Worker something Claude Code can
actually work on.

### Install per project, not globally

```powershell
npm install --save-dev wrangler
npx wrangler --version
```

A bare `npx wrangler` with nothing installed locally fetches whatever version is
newest that day. Wrangler ships breaking changes between majors, so the version
that deploys should be the one pinned in `package.json` and the lockfile — then
`npx wrangler` resolves to the local copy instead of guessing. If you want to be
certain no network lookup happens at all, call `.\node_modules\.bin\wrangler`
directly.

### Log in

```powershell
npx wrangler login
```

A browser opens, you authorize, and wrangler catches the redirect on a local
callback server.

**The Windows-specific part:** that local server trips a Windows Defender
Firewall prompt for Node.js on first run. Dismiss it and you get the worst
failure mode available — the browser page says success while the terminal waits
forever. Allow it; private networks is enough. If you already dismissed it, add
`node.exe` under Windows Security → Firewall & network protection → Allow an app
through firewall, then run the command again.

Like `gh auth login`, this is interactive. It can't be driven from Claude Code
or any non-interactive shell — run it yourself in a real terminal tab.

Verify with:

```powershell
npx wrangler whoami
```

which prints the account email and account ID it's authenticated as. Check the
email — it's easy to authorize with a personal account while your Workers live
under a different one.

### Where the credentials actually live

Not `~/.wrangler`, which is everyone's first guess. On Windows:

```
%APPDATA%\xdg.config\.wrangler\config\default.toml
```

That file holds the OAuth token, its refresh token and expiry, and the granted
scopes; logs land beside it in `...\.wrangler\logs\`. When auth misbehaves,
`npx wrangler logout` followed by `login` is the clean reset — deleting
`default.toml` by hand does the same thing.

It's user-scoped, so one login covers every project on the machine and both
PowerShell editions. Unlike the profile work in Part 2, you don't do this twice.

### Non-interactive and CI

For anything that can't open a browser — CI, a scheduled job — skip `login` and
use a scoped API token from the Cloudflare dashboard:

```powershell
$env:CLOUDFLARE_API_TOKEN = '...'
```

with `CLOUDFLARE_ACCOUNT_ID` alongside it if your token can see more than one
account. Keep tokens out of the repo: `.dev.vars`, `.env` and friends belong in
`.gitignore` before the first commit, not after.

---

## Part 7: OpenCode (optional second agent)

[OpenCode](https://opencode.ai/) is a separate, open-source terminal coding
agent (from the team now at GitHub org `anomalyco`, still published under the
`SST` name in some package managers) that talks to whichever model provider
you point it at — Anthropic, OpenAI, GitHub Copilot, local models, and more.
It's independent of Claude Code: different binary, different config, own
session/auth state. Worth having alongside Claude Code if you want a second
opinion from a different model, or a fallback when one provider is down.

Upstream's own docs recommend WSL for "the best experience." This guide stays
in scope with the rest of the document — native Windows, no WSL — and that
works fine for normal use; treat the WSL note as upstream's preference, not a
requirement.

### Install

Pick one. npm is the natural choice here since Node is already installed from
the Prerequisites section above.

```powershell
npm install -g opencode-ai
```

Or via winget, matching the pattern from Part 1:

```powershell
winget install SST.opencode
```

(`SST.OpenCodeDesktop` is a separate package for the GUI desktop app, if you
want that instead of or alongside the terminal CLI.)

Chocolatey and Scoop both have packages too (`choco install opencode` /
`scoop install opencode`) if you already use one of those. The curl one-liner
advertised on the homepage (`curl -fsSL https://opencode.ai/install | bash`)
needs a real `bash` — run it from a **Git Bash** tab, not PowerShell.

**Close and reopen your terminal** after a winget or choco/scoop install, for
the same reason `gh` and `claude` need it — see
[Why a PATH change doesn't take effect](#why-a-path-change-doesnt-take-effect).
The npm install doesn't have this problem: it lands in npm's existing global
bin, which is already on PATH from the Node.js step.

Verify:

```powershell
opencode --version
```

### Log in

```powershell
opencode auth login
```

or run `/connect` from inside the OpenCode TUI. Either way it opens a browser
for OAuth (or prompts you to paste an API key, depending on the provider) and
writes credentials to `~/.local/share/opencode/auth.json` — on Windows that
resolves under `%USERPROFILE%\.local\share\opencode\auth.json`, the same
`~`-under-`HOME` resolution Vim uses in Part 4.

Like `gh auth login` and `wrangler login`, this is interactive and opens a
browser — run it in a real terminal tab, not from inside Claude Code or
another non-interactive shell.

**If you plan to authenticate with an existing Claude Pro/Max subscription**
rather than a separate Anthropic API key: OpenCode's own docs note that
Anthropic's terms prohibit using a Claude subscription this way through
third-party tools. Read that against your own Anthropic account terms before
choosing it — an API key (pay-per-token, no such restriction) is the
uncomplicated alternative and is what `ANTHROPIC_API_KEY` and most other
providers' `*_API_KEY` env vars give you directly, without the `/connect`
flow, if you'd rather skip OAuth entirely.

### Project config

`opencode init` (or the equivalent first-run prompt) writes an `AGENTS.md` at
the project root — OpenCode's analogue of Claude Code's `CLAUDE.md`. Review
and commit it the same way. Per-project provider settings, model
allow/blocklists, and API-key env-var references go in an `opencode.json`
file alongside it.

---

## Why a PATH change doesn't take effect

This costs more time than any other item in this document, because the symptom
lies: a tool is installed, its folder *is* on PATH, and the shell still says
"not recognized."

**A process gets its environment as a copy, at launch.** Editing PATH — by
installer, by `SetEnvironmentVariable`, by the GUI — writes the registry and
broadcasts a change notice. Explorer picks that up, so anything you launch from
the Start menu afterwards is current. Already-running processes are not
updated, and neither is anything they go on to spawn.

That last clause is the part that bites, because the chain is longer than it
looks:

```
explorer.exe  →  Windows Terminal  →  pwsh          (your tab)
explorer.exe  →  Claude Code app   →  pwsh / bash   (its Bash and PowerShell tools)
```

A new **tab** is a child of the Windows Terminal process that's already running,
so it inherits Terminal's stale copy. Same for Claude Code: every shell its
tools run is a child of the app process, and inherits whatever the app started
with. So "open a new tab" is not enough, and neither is starting a new Claude
Code session inside a running app.

**The fix is to restart the process that's holding the stale copy** — fully
quit Windows Terminal (all windows, including minimized ones) and fully quit the
Claude Code app, then reopen. No PATH edit is required, and adding a second copy
of the entry at user scope will not help, because a newly written entry is
exactly as invisible to an already-running process as the original was.

To see it directly, compare what the registry says against what your shell got:

```powershell
$proc = $env:PATH -split ';'
[Environment]::GetEnvironmentVariable('PATH','Machine') -split ';' |
  Where-Object { $_ -and $proc -notcontains $_ }
[Environment]::GetEnvironmentVariable('PATH','User') -split ';' |
  Where-Object { $_ -and $proc -notcontains $_ }
```

Anything printed is on PATH but invisible to this shell — which tells you the
install is fine and the process is stale. An empty result means the entry really
is missing and needs adding.

To unblock yourself inside a session without restarting anything, call the tool
by full path, or prepend it for that shell only:

```powershell
$env:PATH += ";C:\Program Files\GitHub CLI\"
```

That lasts until the shell closes and changes nothing permanent.

---

## Troubleshooting quick reference

| Error / symptom | Cause | Fix |
|---|---|---|
| "The system cannot find the path specified" from `notepad $PROFILE` | Profile folder doesn't exist | `New-Item -ItemType File -Path $PROFILE -Force` |
| "running scripts is disabled on this system" | Execution policy is Restricted **in this edition** | `Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned` |
| Prompt prints literal `e]9;9;"C:\..."e\` | `` `e `` escape used in PowerShell 5.1 | Use `[char]27` |
| Theme vanished, plain `PS C:\>` prompt | Custom `prompt` overrode Oh My Posh, **or** you're in the shell whose profile is empty | Wrap the original prompt; check `$PSVersionTable` |
| Directory not restored on reopen | OSC 9;9 not emitted, or Terminal startup setting not set | Check both |
| Font shows "Missing fonts" | Family name doesn't match what's installed | Use the exact name from the installer output |
| Font set but glyphs still boxes | Terminal didn't restart, or set on wrong profile | Fully quit Terminal; set on Defaults |
| `claude` not recognized | `.local\bin` not on PATH | Add it, then reopen terminal |
| `oh-my-posh` not recognized in PowerShell 7 only | Installed via `Install-Module` (5.1-scoped path) | Reinstall with winget |
| `gh auth login` hangs or exits immediately | Run from Claude Code or another non-interactive shell | Run it yourself in a real terminal tab |
| `git push` keeps asking for a password | Declined the credential-helper prompt during `gh auth login` | Re-run `gh auth login` and answer **Y** |
| `vim` works in a Git Bash tab, "not recognized" in PowerShell | Git Bash has its own Vim; the winget one is not on PATH | Add `%LOCALAPPDATA%\Programs\Vim` to User PATH |
| Vimrc edits have no effect | A `_vimrc` in your home folder shadows `.vimrc` on Windows | Delete `$env:USERPROFILE\_vimrc` |
| `less` not recognized | Git's `usr\bin` is off PATH by design | Use the `less.cmd` shim in `.local\bin` |
| less says "terminal is not fully functional" | `TERM` unset outside Git Bash | The shim sets it — check you are calling the shim, not `less.exe` directly |
| NERDTree: `3 Invalid file(s): NTUSER.DAT, ntuser.dat.LOG1, ntuser.dat.LOG2` | Vim was opened in `C:\Users\you`, where NERDTree cannot stat the locked registry hive | Cosmetic. Open Vim in a project folder instead |
| `npx` says "running scripts is disabled on this system" while `node` works | npm resolves to `npx.ps1`, blocked by the execution policy | Part 2 Step 2: set `RemoteSigned` |
| `npx wrangler login` never returns, though the browser said success | Firewall prompt for Node dismissed, so the local callback cannot land | Allow `node.exe` through Windows Defender Firewall, re-run |
| `wrangler` runs in one repo, not found in another | It is a devDependency, not a global install | `npm install --save-dev wrangler` in that repo |
| `gh` not recognized right after `winget install` | Every process in the chain predates the machine-PATH update | Fully restart Terminal **and** the Claude Code app — don't add a duplicate PATH entry |
| Setup works in one tab but not another | The two-PowerShell trap | Do it in both |
| `opencode` not recognized right after `winget install` | Same stale-PATH issue as `gh` | Fully restart Terminal, then re-verify |
| `opencode auth login` hangs or does nothing | Run from Claude Code or another non-interactive shell | Run it yourself in a real terminal tab |

---

## Sanity checklist

Run in **each** shell:

```powershell
$PSVersionTable.PSVersion          # which edition am I in
$PROFILE                            # which file does this edition use
Test-Path $PROFILE                  # does it exist
Get-ExecutionPolicy -Scope CurrentUser
claude --version
oh-my-posh --version
node --version                      # npm and npx come with it
gh auth status                      # gh installed and logged in
(vim --version)[0]                  # on PATH in this shell
(less --version)[0]                 # shim resolves
```

If all ten look right in both 5.1 and 7, you're done.

If you also set up Part 7, `opencode --version` is the equivalent check —
left out of the count above since OpenCode is optional.
