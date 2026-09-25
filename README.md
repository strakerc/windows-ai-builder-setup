# Windows PowerShell + Claude Code Setup

**Straker's Windows setup.** Written up after doing it the hard way — the order
and the warnings here exist because each one cost time.

A reference for setting up Claude Code, Oh My Posh, Vim, less, wrangler,
OpenCode, ShareX and directory-restoring prompts on Windows — and for avoiding
the trap that makes this take an hour instead of ten minutes.

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

### ShareX (scrolling screenshots)

Install manually from the Microsoft Store — search "ShareX" and click Get.
(A `winget install ShareX.ShareX` package also exists if you'd rather script
it, but the Store install is what this was verified against, and it's a
one-off GUI install either way — nothing here needs to be repeatable.)

The built-in Snipping Tool only captures what's on screen. For anything taller
than one viewport — a long chat thread, a scrollable settings panel, a web page
that doesn't fit — that means either multiple screenshots or a cropped one that
leaves out the part that mattered. Pasting a partial screenshot into Claude
Code means Claude only sees what you happened to capture, not the whole thing,
which is easy to not notice until it answers based on a part you cut off.

ShareX's **scrolling capture** stitches the full scrollable region into one
image. It's a GUI capture tool, not a CLI — there's nothing to add to PATH or
verify with a version flag. After installing, open it once from the Start menu
so it finishes first-run setup, then use `Capture` → `Scrolling capture` (or
its hotkey, configurable in ShareX's settings), drag a region over the
scrollable area, and let it auto-scroll and stitch before saving.

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

## Part 8: Reaching the PC while you're away (optional)

Two separate problems hide behind "I want to use Claude Code from my phone":
keeping a **session** reachable, and keeping the **machine** reachable. The
first is a desktop-app behaviour that isn't documented anywhere obvious and
cost an afternoon to work out. The second is Wake-on-LAN, and is for desktops
only.

### How remote sessions actually work

The claude.ai/code web view and the mobile app don't talk to your PC. They
talk to a **session process** — one `claude.exe` per open session, spawned by
the desktop app. If that process isn't running, the session shows
*"Can't reach your computer. It may be asleep or offline"* even though the PC
is on and other sessions answer fine. The banner is about the session, not
the machine.

What kills session processes:

- **A reboot.** Every session, including ones that were mid-task. A driver
  update that asks to restart is enough.
- **Quitting the desktop app.**

What doesn't: sleep and wake, the screen locking, or switching between
sessions in the sidebar. A session you opened earlier keeps its process while
you work in another.

See which sessions are actually alive, from any PowerShell tab:

```powershell
Get-CimInstance Win32_Process |
  Where-Object { $_.CommandLine -match 'claude-code.*--resume=' } |
  ForEach-Object { '{0}  {1}' -f $_.CreationDate.ToString('HH:mm:ss'),
    [regex]::Match($_.CommandLine, '--model (\S+)').Groups[1].Value }
```

One line per live session, with its start time and model. If a session you
want isn't there, it isn't reachable.

**Before you leave the house:**

1. Open every session you might want, in the desktop app, on the PC. Open
   means it has a process. Open a spare one in the repo you're most likely to
   need — see below for why.
2. Don't reboot afterwards. Finish driver and Windows updates first.
3. Stop the sleep timer. The High Performance plan still sleeps after 15
   minutes on AC by default, which is long enough to lose the whole trip:

   ```powershell
   powercfg /change standby-timeout-ac 0
   ```

   Put it back when you're home (`powercfg /change standby-timeout-ac 30`).

#### Remote Control is a second, separate switch

A live process is necessary but **not sufficient**. The session also needs
**Remote Control** turned on, which is what links it to your claude.ai account
so it appears in the Code section of the phone app. A session can be running
perfectly and still be invisible there because that switch is off.

It's a toggle in the desktop app's toolbar. There is no `claude rc` command.
You can also just ask a session to turn it on, for itself or another session.

So the two failure modes look different and have different fixes:

| Symptom | Cause | Fix |
|---|---|---|
| Session missing from the app entirely | Remote Control off | Toggle it on |
| Session listed but "Can't reach your computer" | No live process | Open it on the PC, or have a live session message it |

**Keep one lifeline session open with Remote Control on.** From it you can
start and expose all the others remotely. With none, bootstrapping from away
needs Remote Desktop or a trip to the machine.

**A live session can wake a dead one.** Any running session can send a
message to any other session in the sidebar through its session-management
tool, and the target's process starts to handle the message. From then on the
target is reachable from your phone too. Ask the live session something like
*"send a message to the Lineup backtest session to bring it back online — no
work needed, reply in one line"*. It costs one short turn of the target's
model and leaves a "From *<session>*" note in that conversation, which is a
fair price. This is how sessions killed by a reboot were recovered from a
phone on 13 Sep 2026, and it's why the spare session in point 1 matters: as
long as one session in the list is alive, the rest can be brought back.

### Remote desktop

The session route above gives you a terminal, which covers most of what you'd
actually do. If you want the screen as well, everything here needs a UAC click
on the PC, so set it up before you travel, not from the road.

**Windows Remote Desktop** is the best quality option and Windows 11 Pro
already has the host. Turn it on from an elevated prompt:

```powershell
Set-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server' `
  -Name fDenyTSConnections -Value 0
Set-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' `
  -Name UserAuthentication -Value 1          # require Network Level Authentication
Enable-NetFirewallRule -DisplayGroup 'Remote Desktop'
Get-NetFirewallRule -DisplayGroup 'Remote Desktop' |
  Where-Object DisplayName -notmatch 'Shadow' |
  Set-NetFirewallRule -Profile 'Private,Domain'   # deliberately not Public
```

**Check the network profile, or none of that takes effect.** Windows often
classifies a home network as *Public*, and the rules above only apply to
*Private*. This is silent - Remote Desktop listens on 3389 and simply never
answers:

```powershell
Get-NetConnectionProfile | Select-Object InterfaceAlias, NetworkCategory
Set-NetConnectionProfile -InterfaceAlias Ethernet -NetworkCategory Private
```

The Android/iOS client is Microsoft's **Windows App** (formerly Microsoft
Remote Desktop). Connect to the PC's reserved IP. You'll get a certificate
warning: the RDP certificate is self-signed by the PC and no public authority
issues certificates for private addresses, so it is expected. Verify it rather
than clicking through blind - the app shows a **SHA-256** thumbprint, so
compare against that, not the SHA-1 one Windows shows by default:

```powershell
$c = Get-ChildItem 'Cert:\LocalMachine\Remote Desktop' | Select-Object -First 1
($([Security.Cryptography.SHA256]::Create().ComputeHash($c.RawData)) |
  ForEach-Object { '{0:X2}' -f $_ }) -join ':'
```

#### The Microsoft account password trap

This is the part that will cost you an evening, so read it before you start.

Remote Desktop cannot use a Windows Hello PIN. Hello is device-bound by
design, and RDP creates a *new* session that demands a password up front. So
you need the account's password, and on a Microsoft account that is where it
gets strange:

- **Windows keeps its own cached copy of the password**, made when it was last
  set *on that machine*. Check with `(Get-LocalUser -Name <you>).PasswordLastSet`.
- **Sign-in never contacts Microsoft.** Both the lock screen and RDP compare
  against that cached copy only. Verified by the complete absence of
  Microsoft-Account identity events across repeated sign-ins.
- So if you changed your Microsoft password online and have signed in with a
  PIN ever since, **the machine still wants the old one** - the current one is
  correct everywhere except here. Rebooting does not fix it, and neither does
  resetting the password online.
- `Ctrl+Alt+Del` has **no "Change a password"** option for a Microsoft account.
  That's normal, not a policy problem.
- *Settings > Accounts > Your info* offers no Verify prompt either, because
  Windows doesn't know the two have diverged.

The practical answer is to use the old password for RDP and record it as the
Windows sign-in password, separate from the Microsoft account one. If you
want them unified, the only reliable route is *Sign in with a local account
instead* on that same Settings page, which gives you one password you control
at the cost of Microsoft account sync.

**If the lock screen won't even offer a password box**, the passwordless
experience is switched on. Turn it off (elevated), then reboot:

```powershell
Set-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\PasswordLess\Device' `
  -Name DevicePasswordLessBuildVersion -Value 0 -Type DWord
```

Set it back to `2` to restore Hello-only sign-in.

**To diagnose a rejected login**, read the substatus rather than guessing. It
distinguishes a wrong password from a missing account outright, and needs an
elevated prompt:

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625; StartTime=(Get-Date).AddMinutes(-30)} |
  ForEach-Object {
    [regex]::Match($_.Message,'Sub Status:\s*(\S+)').Groups[1].Value
  }
```

`0xc000006a` means wrong password with the account found - which also proves a
password hash exists. `0xc000006e` usually means the account has no usable
password at all.

#### Chrome Remote Desktop, which sidesteps all of that

If the password turns into a dead end, **Chrome Remote Desktop shares the
session that is already running** rather than creating a new one. You see the
real screen, so if Windows is locked you type your **PIN** through it exactly
as if you were sitting at the desk. No Windows password anywhere in the flow.

It's lower quality than RDP and Google brokers the connection, but for
reaching a home PC from a phone neither matters much. Setup is a Chrome
extension, a host install and a PIN you choose.

**RemoteApp** - the *Apps* tab in the Windows App, which publishes individual
programs instead of a whole desktop - needs Windows Server with the Remote
Desktop Services role. No edition of Windows 11 has it, so that tab will
always read "No connected apps".

TeamViewer and AnyDesk add nothing over these for one person and one PC, and
both nag about commercial use.

**From outside the house**, never expose RDP directly to the internet. Put
Tailscale in front of it; the same install is the foundation for waking the PC
remotely (see the end of the next section).

### Wake-on-LAN (desktops only)

**Is this a desktop you'll want to wake remotely for Claude Code access from
afar?** If not, or it's a laptop, skip to the next section. Laptops on Wi-Fi
don't wake reliably and shouldn't be left asleep on a shelf anyway.

> **Status, 23 Sep 2026: working.** On the reference machine (Gigabyte X870E
> AORUS PRO, Realtek RTL8125 2.5GbE, six-node eero mesh, Windows 11 Pro, S3
> sleep) the cause turned out to be the router, not the PC. **eero silently
> drops the subnet broadcast to a wired client but forwards unicast fine.**
> Addressing the magic packet to the PC's own reserved IP instead of
> `x.x.x.255` made it wake first try. Every Windows- and BIOS-side setting
> had been correct for eleven days while the packet was never arriving at
> all. Step 7 is the one that matters; step 8 is how to prove it yourself.

Wake-on-LAN is a "magic packet" broadcast on the local network that the
network card listens for while the PC sleeps. Everything below is about
making sure the card is listening and the packet can reach it.

**1. Confirm the sleep state.** Classic S3 standby is what works. Modern
Standby (*S0 Low Power Idle*) is common on laptops and some newer desktops,
and wake from it is hit and miss.

```powershell
powercfg /a
```

You want `Standby (S3)` under *available* and `S0 Low Power Idle` under *not
available*.

**2. Use the wired port** and note its MAC. Wi-Fi adapters drop their link
during sleep on most consumer hardware; the magic packet goes to the wired
MAC.

```powershell
Get-NetAdapter | Select-Object Name, Status, MacAddress, LinkSpeed
```

**3. Adapter settings.** Device Manager → the Ethernet adapter → *Power
Management*: tick *Allow this device to wake the computer* and *Only allow a
magic packet*. Then on *Advanced*, check the driver's own switches:

```powershell
Get-NetAdapterAdvancedProperty -Name Ethernet |
  Where-Object DisplayName -match 'Wake|Shutdown|Energy|Green' |
  Select-Object DisplayName, DisplayValue
```

*Wake on Magic Packet* and *Shutdown Wake-On-Lan* should be Enabled.
*Energy-Efficient Ethernet* and *Green Ethernet* are the usual cause of
intermittent failures because they let the link drop during sleep — turn them
off from an elevated prompt if wake is unreliable:

```powershell
Set-NetAdapterAdvancedProperty -Name Ethernet -DisplayName 'Energy-Efficient Ethernet' -DisplayValue Disabled
Set-NetAdapterAdvancedProperty -Name Ethernet -DisplayName 'Green Ethernet' -DisplayValue Disabled
```

Also confirm **ARP Offload** is enabled on the same *Advanced* tab. It lets
the card answer address queries by itself while the PC sleeps, so the router
never forgets which port the machine is on. This is what makes the unicast
approach in step 7 survive a long sleep:

```powershell
Get-NetAdapterAdvancedProperty -Name Ethernet |
  Where-Object DisplayName -match 'ARP Offload' |
  Select-Object DisplayName, DisplayValue
```

Verify Windows has armed the adapter — it should be in this list:

```powershell
powercfg /devicequery wake_armed
```

If `Get-NetAdapterPowerManagement` throws *"A device attached to the system
is not functioning"* on a Realtek adapter, ignore it. It does that on every
driver version tried; `powercfg` and the advanced properties are the checks
that count.

**4. BIOS.** The one thing Windows can't verify. On Gigabyte boards it's
*Settings → Platform Power*: **ErP** must be *Disabled* (it cuts standby
power to the network card) and **Wake on LAN** *Enabled*. Other vendors call
it *Power On By PCI-E*, *Resume by LAN*, or *PME Event Wake Up*.

**5. Fast Startup.** Leave it on unless wake from a full shutdown fails, in
which case (elevated):

```powershell
Set-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Power' -Name HiberbootEnabled -Value 0
```

**6. DHCP reservation** for the wired MAC, so remote-desktop clients have a
fixed address to come back to after the wake. On eero: *Settings → Advanced
Networking → Reservations & port forwarding → Add a reservation*, pick the
wired entry (the app shows a generic Wi-Fi icon for wired devices too — check
the MAC). Other routers: *DHCP → Address reservation* or *Static lease*.

**7. The phone app — and the one thing that actually mattered.** *WolOn*
(Android, Darkside Dev) is well maintained. Enter the wired MAC and port 9.
For the address field, **use the PC's own reserved IP (e.g.
`192.168.1.50`), not the broadcast address.**

A magic packet is conventionally broadcast to `x.x.x.255` so that every
device on the segment hears it, and every guide tells you to do that. On a
mesh router it can fail silently. An eero forwards unicast to a wired client
perfectly but drops the directed broadcast, so the packet leaves the phone
and never arrives — no error anywhere, on either end. If your WoL app has a
field labelled *Broadcast Address*, put the host address in it anyway.

Unicast works against a sleeping machine only because the card answers ARP
on its own (**ARP Offload**, step 3). Without that the router eventually
forgets where the sleeping host lives and the packet is dropped instead.

Still **wait 20–30 seconds after the screen goes dark before sending**, and
send two or three times.

**8. When it doesn't work**, split the problem in half. First, what actually
woke the PC last time. A USB device means the network card never fired:

```powershell
powercfg /lastwake
```

Note that `powercfg /lastwake` and the *Power-Troubleshooter* event log are
the only trustworthy record. The Kernel-Power "resumed from sleep" event is
stamped with the clock from *before* the machine slept, because the system
clock stops during S3 and has not resynced when that event is written — so a
genuine hour-long sleep can look like one second. Don't diagnose from it.

Then, with the PC **awake**, listen for the packet directly. This proves
whether it reaches the machine at all, and unlike a `pktmon` capture it
verifies itself first. Elevated prompt:

```powershell
$mac  = [byte[]](0xAA,0xBB,0xCC,0xDD,0xEE,0xFF)      # your wired MAC
$myIp = (Get-NetIPAddress -InterfaceAlias Ethernet -AddressFamily IPv4).IPAddress
$pkt  = [byte[]](,0xFF*6) + ($mac*16)

New-NetFirewallRule -DisplayName TEMP-WOL -Direction Inbound -Protocol UDP `
  -LocalPort 9 -Action Allow -Profile Any | Out-Null
$udp = New-Object Net.Sockets.UdpClient
$udp.Client.Bind([Net.IPEndPoint]::new([Net.IPAddress]::Any, 9))
$udp.Client.ReceiveTimeout = 500

# Controls: loopback and self-unicast DO come back. A broadcast does NOT
# return to its own sender, so it is useless as a self-test.
$s = New-Object Net.Sockets.UdpClient
$s.Send($pkt,$pkt.Length,'127.0.0.1',9) | Out-Null
$s.Send($pkt,$pkt.Length,$myIp,9)       | Out-Null
$s.Close()

Write-Host 'Tap the WoL app now (45 s)...'
$end = (Get-Date).AddSeconds(45)
while ((Get-Date) -lt $end) {
  $ep = [Net.IPEndPoint]::new([Net.IPAddress]::Any,0)
  try { $d = $udp.Receive([ref]$ep); "  from $($ep.Address)  $($d.Length) bytes" } catch {}
}
$udp.Close(); Remove-NetFirewallRule -DisplayName TEMP-WOL
```

Read it like this:

- **Controls didn't come back:** the listener is broken; the run says nothing
  about the phone. Fix that before concluding anything.
- **Controls came back, phone didn't:** the packet is not reaching the PC.
  Switch the app from the broadcast address to the host address (step 7).
  If that still fails, check the phone is on the main network rather than
  Guest, and that no VPN is active on it.
- **Phone's packets arrived but the PC won't wake from sleep:** now, and only
  now, the BIOS (step 4) and the power-saving Ethernet options (step 3) are
  worth suspecting.

A `pktmon` capture is the obvious tool here and it silently captured nothing
on the reference machine, costing days. If you use it, always send yourself
a control packet first.

**From outside the house**, the unicast trick in step 7 changes everything,
and the answer turns out to be much simpler than the usual advice.

The two reasons waking over the internet is "impossible" are that a broadcast
cannot cross it and that routers refuse to forward a port to a broadcast
address. **Neither applies once you send to the host address.** A plain port
forward then works, and no relay, VPN or extra hardware is needed. This was
proven end to end on the reference setup: phone on cellular, packet through
the forward, machine awake, `powercfg /lastwake` naming the network card.

**1. Check for carrier-grade NAT first**, because it is the one thing that
kills this outright. Compare what the internet sees with your router's WAN
address:

```powershell
Invoke-RestMethod https://api.ipify.org     # what the internet sees
```

Then find the WAN address in your router (on eero: *Settings > Advanced
networking > Internet > WAN IP address*). **If they match, you're fine.** If
the router shows something in `100.64.x.x`-`100.127.x.x`, you are behind
CGNAT, inbound forwarding is impossible, and you need the relay described at
the end of this section.

Do not try to infer this from a traceroute. ISPs use the CGNAT range for their
own internal transit links, so a `100.x` hop appears on plenty of connections
that are not behind CGNAT at all. Comparing the two addresses is the only
reliable test.

**2. Forward UDP port 9** to the PC's reserved address. On eero: the device's
page, *Reservation & port forwarding*, *Open a port*. Protocol **UDP**,
external and internal port both **9**.

**3. Add a second entry in the WoL app** rather than editing one back and
forth. Same MAC - that never changes, it is the payload that identifies the
card - but the address is your home's public one instead of the private one.
Leave the status-check field blank, since nothing is forwarded that would
answer it.

| Field | Home entry | Away entry |
|---|---|---|
| MAC | the wired MAC | *same* |
| Address | the PC's reserved IP | your public IP |
| Port | 9 | 9 |
| Status check | the reserved IP | blank |

**4. Test with Wi-Fi off**, on cellular, which is the only way to prove the
packet really left the house. Verify it arrived using the listener in step 8 -
a packet from your carrier's address is real proof, whereas one showing your
*own* public address as the source means the router hairpinned it and you were
still on Wi-Fi. (That hairpin behaviour is handy: the Away entry also works
from home. The Home entry is still preferable there, being one hop instead of
a round trip.)

**5. Set up dynamic DNS**, or this breaks silently the day your ISP hands you
a new address. Many routers have it built in, eero included (*Settings >
Advanced networking > Dynamic DNS*) - but on eero that needs a paid eero Plus
subscription.

The free alternative is DuckDNS plus a scheduled task on the PC. Sign in at
duckdns.org with Google or GitHub, add a subdomain, and copy the token shown
at the top of the page. Keep the credential in its own file, outside any repo
and readable only by you:

```
C:\Users\<you>\duckdns\config.txt          # domain=<subdomain>  token=<token>
C:\Users\<you>\duckdns\update-duckdns.ps1
```

The updater is one call wrapped in logging. Passing an **empty** `ip=` makes
DuckDNS use the source address it observes, which is correct behind any NAT
and avoids depending on a lookup service:

```powershell
Invoke-RestMethod "https://www.duckdns.org/update?domains=$domain&token=$token&ip="
```

Register it to run every 15 minutes with `StartWhenAvailable`, so a resume
from sleep fires the missed run promptly instead of waiting out the interval:

```powershell
$trigger = New-ScheduledTaskTrigger -Once -At (Get-Date).AddMinutes(1) `
  -RepetitionInterval (New-TimeSpan -Minutes 15) `
  -RepetitionDuration (New-TimeSpan -Days 3650)   # TimeSpan::MaxValue is rejected
$settings = New-ScheduledTaskSettingsSet -StartWhenAvailable
```

**Never put `pwsh.exe` in a scheduled task by name.** If PowerShell 7 came
from the Microsoft Store it lives under `WindowsApps` behind an execution
alias, and Task Scheduler cannot follow that alias. The task fails with
`2147942402` (`0x80070002`, "cannot find the file"), which says nothing about
the real cause, and it fails *silently* - there is no notification, and a
DDNS record that stops updating looks exactly like a working one until your
address changes weeks later.

Give it a real path. Either PowerShell 7's own, when it was installed by MSI
rather than the Store, or Windows PowerShell, which is always at a fixed
location:

```powershell
$exe = Get-ChildItem 'C:\Program Files\PowerShell' -Directory -ErrorAction SilentlyContinue |
       ForEach-Object { Join-Path $_.FullName 'pwsh.exe' } |
       Where-Object { Test-Path $_ } | Select-Object -First 1
if (-not $exe) { $exe = "$env:SystemRoot\System32\WindowsPowerShell\v1.0\powershell.exe" }

New-ScheduledTaskAction -Execute $exe -Argument "-NoProfile -NonInteractive ..."
```

**Always verify a scheduled task actually ran**, rather than trusting that
registering it was enough:

```powershell
Get-ScheduledTaskInfo -TaskName '<name>' |
  Select-Object LastRunTime, LastTaskResult, NextRunTime   # LastTaskResult 0 = success
```

Then point the Away entry at `<subdomain>.duckdns.org` instead of the raw
address. **Check the spelling.** A `.com` for `.org` fails exactly like a
broken setup, with no error from the app and nothing in any log.

**The limitation of updating from the PC:** it only runs while the machine is
awake, so if your ISP changes your address during a long sleep the hostname
goes stale precisely when you need it. Residential addresses usually change at
modem restarts, so this is unlikely rather than impossible. A scheduled wake
every few hours to refresh the record closes the gap if it ever bites, and a
router that does DDNS itself avoids it entirely.

**On security:** a magic packet can only power the machine on. It carries no
payload that reaches anything running on it, and nothing is listening on that
port while the PC is awake. The worst a stranger who guessed your address
could do is turn your computer on.

**If you are behind CGNAT**, forwarding is off the table and something at home
has to send the packet for you: a Raspberry Pi, NAS, or an old phone that
stays plugged in, reached over Tailscale; or Home Assistant, which has a
Wake-on-LAN switch built in. Tailscale on the desktop itself does not help -
the desktop is the thing that is asleep.

### If sites crawl on the wired connection (only if measured)

**Skip this unless you have measured the problem on your own machine.** It
is a workaround for one driver or card misbehaving, not a general tuning
step. Most wired connections are fine, and applying this blind just means
running two connections for no reason.

> **Found 24 Sep 2026 on the reference machine** (Realtek RTL8125, drivers
> 11.29.50 and 11.31.50, eero mesh). After switching from Wi-Fi to Ethernet
> for Wake-on-LAN, signed-in Facebook, Messenger and claude.ai became very
> slow in Chrome *and* Firefox. Speed tests, ping and DNS all looked normal
> the whole time.

**The symptom.** Speed tests read normal, and logged-out pages load fine,
but signed-in pages and chat apps crawl: sign-in spinners hang and feeds
take seconds to fill in. Speed tests and cached static files use TCP. Live
signed-in data mostly arrives over **HTTP/3 (QUIC), which runs on UDP**, so a
connection that mishandles UDP passes every speed test and still feels
broken.

**How to confirm it.** Load a slow signed-in page and read its resource
timings in the DevTools console:

```javascript
performance.getEntriesByType('resource')
  .filter(e => e.encodedBodySize > 30000 && e.transferSize > 0)
  .map(e => ({ file: new URL(e.name).pathname.split('/').pop(),
               proto: e.nextHopProtocol,
               KB: Math.round(e.encodedBodySize / 1024),
               downloadMs: Math.round(e.responseEnd - e.responseStart) }))
```

You have this problem only if **both** of these hold:

1. `h3` responses of 100-500 KB take several seconds to download (about
   0.2-0.5 Mbps), while `h2` ones are fast. Setting
   `chrome://flags/#enable-quic` to *Disabled* makes the same page fast.
   Put the flag back afterwards.
2. With the Ethernet cable **unplugged**, on Wi-Fi to the same router, the
   same `h3` responses are fast. This is the test that tells a bad wired
   path apart from a router or ISP that interferes with UDP. If Wi-Fi is
   slow too, this section is not your fix.

On the reference machine: a 473 KB response took 6.5-7.1 s over Ethernet,
0.9-1.1 s over Wi-Fi and 0.8 s with QUIC disabled.

**What did not fix it there.** Each one was tested and reverted, so there is
no need to repeat them. They may still help on a different card:

- Windows UDP Receive Offload: `netsh int udp set global uro=disabled`, even
  after restarting the adapter
- Windows UDP Send Offload: `netsh int udp set global uso=disabled`
- The card's *UDP Checksum Offload (IPv4)* set to *Tx Enabled*
- Updating the Realtek driver from Gigabyte's 11.29.50 to Realtek's 11.031.50
  (NetAdapterCx)
- A different port on the router
- MTU, network profile and QoS policy: identical on both connections

The untried next step was Realtek's NDIS driver (10.80.50), which is a
different driver design rather than a newer version.

**If you update the Realtek driver**, the install resets the card's advanced
settings. It turned *Energy-Efficient Ethernet* and *Green Ethernet* back on
and set *WOL & Shutdown Link Speed* to *10 Mbps First*. Put those back
(Disabled, Disabled, Not Speed Down), then re-check the step 3 settings.

**The workaround: keep the cable, prefer Wi-Fi.** The cable stays plugged in
for Wake-on-LAN, and everyday traffic goes over Wi-Fi. A sleeping PC listens
only on the wired card, and the magic packet in step 7 is addressed to the
wired card's reserved IP, so waking is unaffected. Remote Desktop to
`<wired-IP>` also keeps working. In an **administrator** PowerShell:

```powershell
Set-NetIPInterface -InterfaceAlias "Wi-Fi" -InterfaceMetric 10
Set-NetIPInterface -InterfaceAlias "Ethernet" -InterfaceMetric 50
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\WcmSvc\GroupPolicy" /v fMinimizeConnections /t REG_DWORD /d 0 /f
```

**The metrics alone do nothing.** Windows's default policy, *minimize the
number of simultaneous connections*, keeps Wi-Fi associated whenever a cable
is present but routes nothing over it. The routing table shows Wi-Fi
preferred while every connection still leaves over the cable. The `reg add`
line turns that policy off. After running all three, toggle Wi-Fi off and on
and fully restart the browser, because existing connections stay on the
cable until they close.

**Verify the traffic really moved.** New connections should leave from the
Wi-Fi address, not the wired one:

```powershell
curl.exe -s -o NUL -w "%{local_ip}`n" https://example.org/
```

**Then re-test Wake-on-LAN** (step 8). Check `powercfg -lastwake`, not the
Power-Troubleshooter summary. On this board the summary named only the USB4
host router, while `-lastwake` listed the Realtek card as well.

**To undo it all:**

```powershell
Set-NetIPInterface -InterfaceAlias "Wi-Fi","Ethernet" -AutomaticMetric Enabled
reg delete "HKLM\SOFTWARE\Policies\Microsoft\Windows\WcmSvc\GroupPolicy" /v fMinimizeConnections /f
```

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
| Phone/web shows "Can't reach your computer" for one session while the PC is on | That session has no live process — a reboot killed it, or it hasn't been opened since | Open it on the PC, or ask a live session to send it a message (Part 8) |
| PC went to sleep during a trip despite being "left on" | High Performance plan still sleeps after 15 min on AC | `powercfg /change standby-timeout-ac 0` |
| Wake-on-LAN: packet never reaches the PC | Mesh router (eero) drops the subnet broadcast to wired clients | Address the packet to the PC's reserved IP, not `x.x.x.255` (Part 8, step 7) |
| Sleep looks like it lasts 1 second in Event Viewer | Kernel-Power resume event is stamped with the pre-sleep clock | Use `powercfg /lastwake` and Power-Troubleshooter instead |
| Wake-on-LAN: packet arrives but the PC stays asleep | BIOS ErP on / Wake on LAN off, or Energy-Efficient Ethernet dropped the link | BIOS *Platform Power*; disable EEE and Green Ethernet |
| Scheduled task fails with `2147942402` and no other clue | `pwsh.exe` given by name, but it is a Store build behind a `WindowsApps` alias Task Scheduler cannot follow | Use a real path to the executable (Part 8) |
| Speed tests fine but signed-in sites crawl, only on Ethernet | HTTP/3 (UDP) mishandled on the wired path; confirm by measuring, since most machines don't have this | Measure first, then keep the cable and prefer Wi-Fi (Part 8, *If sites crawl on the wired connection*) |
| Wi-Fi given a lower metric but traffic still uses the cable | Windows's *minimize simultaneous connections* policy stops routing over Wi-Fi while a cable is present | Set `fMinimizeConnections` to 0, then toggle Wi-Fi (Part 8) |

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

If you set up Part 8 on a desktop, `powercfg /devicequery wake_armed` should
list the Ethernet adapter and `powercfg /a` should show `Standby (S3)` as
available. Also optional.

ShareX has no CLI check — open it from the Start menu and confirm the tray
icon appears.
