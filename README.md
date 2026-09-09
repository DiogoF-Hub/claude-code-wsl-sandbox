# Sandboxed Claude Code on Windows (WSL 2 + ai-jail + Bitwarden SSH signing)

A reproducible setup for running an AI coding agent on Windows inside an OS-level
sandbox, while keeping SSH commit signing backed by Bitwarden.

The agent can only read and write the project directory. It cannot reach `~/.ssh`,
the Windows filesystem, or any Windows executable. Commits are still signed by the
key held in Bitwarden, which never touches disk.

## Why I built this

This is my personal setup, documented so I can rebuild it and so others can copy the
parts they want.

The motivation is **not** fear of malware. I do not assume Claude Code is compromised or
hostile. The problem is a much more ordinary one: an agent that wanders. Left
unconstrained it will run `reg.exe` queries, poke at Windows config, read dotfiles, or
start "helpfully" fixing things nowhere near the code I asked about. Sometimes it goes
far beyond the request, which is scope creep rather than sabotage.

So the goal is containment of *reach*, not defence against an attacker. When I ask for
help with a project, the agent should only be able to act on that project. Everything
else on the machine is simply not there. As a bonus, the same boundary happens to limit
the damage if a repo ever does contain a prompt injection.

Worth being clear about the limit: the sandbox constrains **where** the agent can act,
not **how much** it does. For bounding behaviour, see
[Security notes](#scope-creep-is-not-a-security-control).

> Paths below assume the username `diogo` on both Windows and WSL. Adjust as needed.

## What is in this repo

- `README.md`, this guide
- [`.ai-jail`](.ai-jail), my per-project sandbox config, and
  [`.ai-jail-global`](.ai-jail-global), which belongs at `~/.ai-jail`. See
  [Part 8](#part-8-the-ai-jail-config)
- [`CLAUDE.md`](CLAUDE.md) and [`jail.md`](jail.md), the standing session instructions,
  see [Part 9](#part-9-session-instructions)

They are meant to be copied and adapted, not used verbatim.

---

## Table of contents

1. [Why WSL](#why-wsl)
2. [How it fits together](#how-it-fits-together)
3. [Part 1: WSL base setup](#part-1-wsl-base-setup)
4. [Part 2: VS Code](#part-2-vs-code)
5. [Part 3: Claude Code](#part-3-claude-code)
6. [Part 4: mise, ai-jail and Node](#part-4-mise-ai-jail-and-node)
7. [Part 5: Bitwarden SSH agent bridge](#part-5-bitwarden-ssh-agent-bridge)
8. [Part 6: Git signing in WSL](#part-6-git-signing-in-wsl)
9. [Part 7: Git signing on Windows](#part-7-git-signing-on-windows)
10. [Part 8: The .ai-jail config](#part-8-the-ai-jail-config)
11. [Part 9: Session instructions](#part-9-session-instructions)
12. [Maintenance](#maintenance)
13. [Troubleshooting](#troubleshooting)
14. [Security notes](#security-notes)

---

## Why WSL

[ai-jail](https://github.com/akitaonrails/ai-jail) has no native Windows support and
will not get it. Its Linux backend is `bubblewrap` (namespaces + Landlock LSM +
seccomp); its macOS backend is `sandbox-exec`. Windows has no userspace equivalent.
AppContainers are a different API, need admin to configure, and do not map onto what
bwrap does.

WSL 2 runs a real Linux kernel, so bwrap works normally. Claude Code must therefore be
installed **inside the WSL distro**, not on Windows, because ai-jail sandboxes the Linux
binary.

A useful side effect: inside the jail there is no `/mnt`, so `reg.exe`,
`powershell.exe` and `cmd.exe` are unreachable. The agent cannot touch Windows at all.

## How it fits together

```
Windows                              │  WSL 2 (Ubuntu)
─────────────────────────────────────┼──────────────────────────────────────
Bitwarden Desktop                    │
  └── \\.\pipe\openssh-ssh-agent  ◄──┼── npiperelay.exe ◄── socat
                                     │        ▲                 ▲
VS Code (Remote-WSL)  ───────────────┼────────┼─────────────────┤
                                     │        │      ~/.ssh/agent.sock.$$
                                     │        │                 ▲
                                     │   git commit -S ─────────┘
                                     │
                                     │   ai-jail ──► bwrap ──► claude
                                     │                  (project dir only)
```

Commits are made **outside** the jail. The agent writes code; you commit.

## Prerequisites

- Windows 11 with WSL 2 and an Ubuntu distro
- Bitwarden Desktop with the SSH agent enabled. See the
  [official Bitwarden SSH agent guide](https://bitwarden.com/help/ssh-agent/) and
  [Part 7](#part-7-git-signing-on-windows) below for the full Windows walkthrough
- An SSH key stored in Bitwarden

---

## Part 1: WSL base setup

### 1.1 `.wslconfig` (Windows side)

`C:\Users\diogo\.wslconfig`:

```ini
[wsl2]
# 1. Shut the VM down as soon as all WSL instances have exited
vmIdleTimeout=0
```

This trades a few seconds of start-up delay for the VM not sitting idle in the
background. It only takes effect once **no process** is left running in the distro.
See [Part 5.4](#54-why-the-relay-must-die-with-the-shell) for why the SSH relay matters
here.

### 1.2 Fix `/etc/resolv.conf` (required for ai-jail)

By default WSL makes `/etc/resolv.conf` a **symlink** to `/mnt/wsl/resolv.conf`.
bwrap cannot bind over that symlink and ai-jail fails to start with:

```
bwrap: Can't create file at /etc/resolv.conf: No such file or directory
```

The fix is to make it a regular file with the same contents. First, note your current
nameserver and search domain:

```bash
cat /etc/resolv.conf
```

Then:

```bash
sudo tee /etc/wsl.conf > /dev/null << 'EOF'
[network]
generateResolvConf = false
EOF
```

Shut down from PowerShell so `wsl.conf` takes effect **before** writing the file:

```powershell
wsl --shutdown
```

Reopen WSL, then:

```bash
# 1. Remove the symlink
sudo rm -f /etc/resolv.conf

# 2. Write a regular file with the same values noted above
printf 'nameserver 10.255.255.254\nsearch <your-tailnet>.ts.net\n' | sudo tee /etc/resolv.conf

# 3. Confirm it is a regular file (expect -rw-r--r--, no arrow)
ls -l /etc/resolv.conf
ping -c1 github.com
```

Notes:

- `10.255.255.254` is WSL's DNS proxy into the Windows resolver. It is a fixed address
  and does not drift across reboots, so you keep whatever DNS Windows is using,
  including Tailscale MagicDNS and any Pi-hole filtering.
- The `search` line only matters for bare single-label hostnames. Drop it if you do not
  use MagicDNS.
- Trade-off: WSL no longer updates this file automatically. If you join a VPN that
  pushes its own search domains, add them by hand.
- Revert with `sudo rm /etc/wsl.conf /etc/resolv.conf` then `wsl --shutdown`.

> Order matters. Writing the file before `wsl --shutdown` lets WSL recreate the symlink
> on the next boot.

---

## Part 2: VS Code

Install the **WSL** extension in Windows VS Code, then launch from inside the distro:

```bash
cd ~/Projects/my-app
code .
```

The bottom-left corner should read `WSL: Ubuntu`. VS Code Server runs *outside* the
jail, so it sees and edits the project normally. You watch every agent edit live.

**Keep repositories in `~/Projects`, not `/mnt/c`.** On `/mnt/c` the DrvFS mount has
poor inotify support, so VS Code often fails to auto-refresh when the agent writes
files, which defeats the purpose. It also has no real Unix permission bits, so git
reports phantom mode changes (`git config core.fileMode false` if you hit it).

---

## Part 3: Claude Code

Use the native installer. It needs no Node.js and avoids the common WSL failure where
`npm install -g` picks up the *Windows* npm and errors on a platform mismatch.

```bash
curl -fsSL https://claude.ai/install.sh | bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc
claude --version
```

Run `claude` once to log in. This is a **separate login** from any Windows-side install,
because WSL has its own `~/.claude`. Use `claude doctor` to diagnose install problems.

---

## Part 4: mise, ai-jail and Node

[mise](https://mise.jdx.dev/) manages tool versions per project and installs standalone
binaries from GitHub releases. ai-jail has built-in mise integration, so tools it
manages are exposed correctly inside the jail.

```bash
curl https://mise.run | sh
echo "eval \"\$(/home/diogo/.local/bin/mise activate bash)\"" >> ~/.bashrc
exec bash

sudo apt update && sudo apt install -y bubblewrap socat
mise use -g github:akitaonrails/ai-jail
mise use -g node@lts    # example only, use whatever runtime the project needs

ai-jail --version
```

If you previously installed ai-jail by hand, remove it so there is only one copy:

```bash
rm -f ~/.local/bin/ai-jail
exec bash
which -a ai-jail     # expect a single path under ~/.local/share/mise/installs/
```

> `mise upgrade` refuses releases younger than 24 hours (`minimum_release_age`) as a
> supply-chain guard. A new release is picked up on the next run once it has aged in.
> `mise upgrade --bump` bypasses it, but leave the default alone.

---

## Part 5: Bitwarden SSH agent bridge

Bitwarden's SSH agent listens on a Windows **named pipe** (`\\.\pipe\openssh-ssh-agent`).
Linux tools speak to a **Unix domain socket**. `npiperelay.exe` bridges the two:
it opens the named pipe and relays it over stdin/stdout, while `socat` creates the Unix
socket on the Linux side and spawns the relay per connection.

The private key never enters WSL. Only the sign request and the signature cross.

Enable the agent in Bitwarden Desktop first. Full walkthrough in
[Part 7](#part-7-git-signing-on-windows), or see
[bitwarden.com/help/ssh-agent](https://bitwarden.com/help/ssh-agent/).

### 5.1 Install npiperelay (Windows)

Use the actively maintained [albertony fork](https://github.com/albertony/npiperelay).
The original (`jstarks`) has not been updated since 2020.

```powershell
winget install albertony.npiperelay
where.exe npiperelay
```

Open a **new** PowerShell window before running `where.exe`, because the PATH change
does not reach already-running terminals.

### 5.2 Symlink it into WSL

Symlinking rather than copying means `winget upgrade` maintains the binary and the
bridge follows automatically:

```bash
sudo ln -s "/mnt/c/Users/diogo/AppData/Local/Microsoft/WinGet/Packages/albertony.npiperelay_Microsoft.Winget.Source_8wekyb3d8bbwe/npiperelay.exe" \
    /usr/local/bin/npiperelay.exe

/usr/local/bin/npiperelay.exe    # expect the usage text
```

Adjust the path to whatever `where.exe npiperelay` reported.

### 5.3 The `.bashrc` block

Append to `~/.bashrc`:

```bash
# Bitwarden SSH agent bridge (per-shell; dies with this shell)
# 1. Clear sockets whose owning shell no longer exists
for s in "$HOME"/.ssh/agent.sock.*; do
    [ -e "$s" ] || continue
    pid="${s##*.}"
    kill -0 "$pid" 2>/dev/null || { pkill -f "UNIX-LISTEN:$s," 2>/dev/null; rm -f "$s"; }
done

# 2. Start the relay in its own session, so a stray Ctrl+C never reaches it
export SSH_AUTH_SOCK="$HOME/.ssh/agent.sock.$$"
setsid socat UNIX-LISTEN:"$SSH_AUTH_SOCK",fork,unlink-early \
    EXEC:"npiperelay.exe -ei -s //./pipe/openssh-ssh-agent",nofork >/dev/null 2>&1 &

# 3. Tear it down when this shell exits, matched by socket path rather than PID
trap 'pkill -f "UNIX-LISTEN:$SSH_AUTH_SOCK," 2>/dev/null; rm -f "$SSH_AUTH_SOCK"' EXIT
```

Then verify:

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
exec bash
ssh-add -l                    # expect your key fingerprint
ls -la ~/.ssh/agent.sock.*    # expect exactly one, numbered for this shell
```

Design notes:

- **`.$$` suffix**: each shell gets its own socket, so two terminals never fight over
  one path. It also makes the socket self-identifying, which is what the sweep relies on.
- **`trap ... EXIT`**: kills the relay when the shell exits. No interactive guard is
  needed, because Ubuntu's stock `.bashrc` already returns early for non-interactive
  shells. It matches on the socket path rather than a stored PID, because `setsid` forks
  and `$!` no longer points at socat.
- **`setsid`**: puts the relay in its own session, so a stray Ctrl+C at the prompt cannot
  reach it. Without this, one mistyped command kills your agent for that shell and
  nothing tells you until a signature fails. Note that `trap '' INT` before `exec socat`
  does *not* work: socat installs its own SIGINT handler at startup rather than checking
  whether the signal is already ignored, so the inherited disposition is overwritten. The
  signal has to not be delivered at all.
- **The sweep exists because the trap is not guaranteed.** `EXIT` fires on a clean exit
  or a SIGHUP, but not on SIGKILL. A terminal that is force-killed rather than closed
  leaves its relay running with no parent, and one orphaned relay is enough to keep the
  whole distro alive and defeat `vmIdleTimeout=0`. Since each socket is named after its
  shell's PID, `kill -0` on that PID says whether the owner still exists, so opening any
  new shell clears what the last killed one left behind.
- **`unlink-early`**: removes a stale socket file if one is still sitting at the path.

One known limit: the sweep matches sockets to shells by PID, so if a dead shell's PID has
been reused by an unrelated process, that orphan survives one extra round. It clears on a
later pass, and `wsl --shutdown` is always a hard reset.

Check for strays at any time with:

```bash
ps -eo pid,etime,cmd --sort=-etime | grep [s]ocat
```

Anything older than your current session is an orphan.

### 5.4 Why the relay must die with the shell

`setsid` alone, with no teardown, leaves a relay running after every terminal closes,
which keeps the distro alive and silently defeats `vmIdleTimeout=0`. That is why the
block above pairs it with the EXIT trap: detached enough to survive Ctrl+C, still torn
down when the shell exits, so the VM shuts down once the last one closes.

Confirm with:

```powershell
wsl --list --running    # expect "There are no running distributions"
```

Known trade-off: VS Code captures `SSH_AUTH_SOCK` once when it resolves the shell
environment, so the **git panel** may end up with a dead socket path. The **integrated
terminal** always works, since each terminal spawns its own live relay. Commit from the
terminal.

---

## Part 6: Git signing in WSL

### 6.1 Register the key on GitHub twice

GitHub stores authentication keys and signing keys as **separate resources**. The same
public key must be added twice, once as an *Authentication Key* and once as a *Signing
Key*. This is the intended setup, not a duplicate.

Symptom of a missing auth key: `git@github.com: Permission denied (publickey)` even
though signing works. Symptom of a missing signing key: commits land **Unverified**.

Verify with:

```bash
ssh -T git@github.com    # expect "Hi <user>! You've successfully authenticated"
```

> Never delete an existing Signing entry to re-add it as Authentication, because that
> breaks verification on all existing signed commits. Add, do not replace.

### 6.2 Git config in WSL

```bash
# 1. Copy the public key from Windows so both sides match
cp /mnt/c/Users/diogo/.ssh/bitwarden_signing.pub ~/.ssh/
chmod 644 ~/.ssh/bitwarden_signing.pub

# 2. Identity, must match the Windows config or GitHub splits your contributions
git config --global user.name "Diogo"
git config --global user.email "diogo@carvalhofer.lu"

# 3. Sign with SSH rather than GPG
git config --global gpg.format ssh
git config --global user.signingkey "$HOME/.ssh/bitwarden_signing.pub"
git config --global commit.gpgsign true

# 4. Use SSH for your own repos, even when the URL is HTTPS
git config --global url."git@github.com:<user>/".insteadOf "https://github.com/<user>/"
```

Step 4 is worth setting. GitHub removed password authentication for HTTPS in 2021, so
cloning an HTTPS URL prompts for a personal access token, which is a second credential
to manage. The rewrite means you can paste one of your own GitHub URLs straight from the
browser and it silently goes over SSH through the Bitwarden key. It applies to submodules
and to any tool that shells out to git.

**Scope it to your account rather than all of github.com.** The unscoped form,
`url."git@github.com:".insteadOf "https://github.com/"`, forces *every* GitHub URL
through your key, including public repos that any tool might clone in the background.
Those clones then fail with `git@github.com: Permission denied (publickey)` whenever the
subprocess does not inherit your agent socket, which is a confusing error for something
that would have worked anonymously over HTTPS.

Add a line per account or organisation you push to:

```bash
git config --global url."git@github.com:<org>/".insteadOf "https://github.com/<org>/"
```

Trailing slashes matter: without them the prefix would also match names that merely start
with the same characters. Review what you have with:

```bash
git config --global --get-regexp 'url\..*\.insteadof'
```

For a repo already cloned over HTTPS, point it at SSH directly:

```bash
git remote set-url origin git@github.com:<user>/<repo>.git
git remote -v
```

Two settings that look contradictory but are not:

- `gpg.format` selects the **backend** (`openpgp`, `ssh`, `x509`)
- `commit.gpgsign` selects **whether to sign at all**. The name is historical, from
  when GPG was the only option.

Do **not** copy these two lines from a Windows `.gitconfig`:

| Windows line | Why it breaks in WSL |
| --- | --- |
| `user.signingkey=C:\Users\...` | Windows path, will not resolve |
| `gpg.ssh.program=C:/Windows/System32/OpenSSH/ssh-keygen.exe` | Git passes it Linux temp paths the Windows binary cannot read, so signing fails outright |

The same reason rules out the commonly suggested `alias ssh=ssh.exe` trick: it fixes
push and pull, but never signing.

### 6.3 Local verification (optional)

Without this, `git log --show-signature` reports
`gpg.ssh.allowedSignersFile needs to be configured` and shows `No signature`, even
though the commit **is** signed and GitHub shows Verified.

```bash
echo "diogo@carvalhofer.lu $(cat ~/.ssh/bitwarden_signing.pub)" > ~/.ssh/allowed_signers
git config --global gpg.ssh.allowedSignersFile ~/.ssh/allowed_signers
git log --show-signature -1    # expect: Good "git" signature
```

Optionally import GitHub's web-flow key so commits created in the GitHub UI verify too:

```bash
curl -sL https://github.com/web-flow.gpg | gpg --import
```

---

## Part 7: Git signing on Windows

Standalone and independent of everything above. This is how to get the **Verified**
badge for commits made from a normal Windows VS Code window or PowerShell prompt. It is
not required for the WSL workflow, but it is the natural companion to it, and steps 1
and 2 are prerequisites for the WSL bridge in [Part 5](#part-5-bitwarden-ssh-agent-bridge)
as well.

> Signing is separate from pushing. Pushing already works without any of this. Signing
> just proves the commit author is really you, which is what earns the Verified badge.

### Step 1: Disable the Windows OpenSSH Authentication Agent

Bitwarden's SSH agent uses the same Windows named pipe as the built-in OpenSSH agent, so
the built-in one has to be turned off first.

1. Press `Win + R`, type `services.msc`, press Enter.
2. Find **OpenSSH Authentication Agent**.
3. Double-click it.
4. Set **Startup type** to **Disabled**.
5. Click **Stop**, then **OK**.

> This affects all SSH usage on the machine, not just Git. Any SSH connections (homelab,
> servers) now go through Bitwarden's agent too.

### Step 2: Enable Bitwarden as SSH agent

1. Open Bitwarden Desktop (version 2025.1.2 or newer).
2. Go to **Settings** then **Security**.
3. Enable **Use Bitwarden as SSH agent**.
4. Set **Ask for authorization** to **Always**.

Reference: [bitwarden.com/help/ssh-agent](https://bitwarden.com/help/ssh-agent/)

### Step 3: Generate the signing key in Bitwarden

Skip if the key already exists.

1. Click the **+** button, then **SSH Key**.
2. Name it something like `GitHub Signing Key - Windows`.
3. Click **Generate**, then choose **Ed25519**.
4. Save the item.
5. Copy the **public key** (starts with `ssh-ed25519 AAAA...`).

### Step 4: Register the public key on GitHub

1. GitHub, then **Settings**, **SSH and GPG keys**, **New SSH key**.
2. Title: `Bitwarden Signing - Windows`.
3. **Key type: Signing Key.** This is the important part. The default is
   "Authentication Key", which will NOT verify commits.
4. Paste the public key, then **Add SSH key**.

If you also push over SSH, repeat this with **Key type: Authentication Key**. See
[6.1](#61-register-the-key-on-github-twice).

### Step 5: Save the public key as a file

```powershell
# 1. Create the .ssh folder if it does not exist
New-Item -Path "$env:USERPROFILE\.ssh" -ItemType Directory -Force

# 2. Save the public key (replace the placeholder with the value copied from Bitwarden)
Set-Content -Path "$env:USERPROFILE\.ssh\bitwarden_signing.pub" -Value "ssh-ed25519 AAAA... PASTE_YOUR_PUBLIC_KEY_HERE"
```

### Step 6: Configure Git

```powershell
# 1. Use SSH (not GPG) for signing
git config --global gpg.format ssh

# 2. Point Git at the public key file
git config --global user.signingkey "$env:USERPROFILE\.ssh\bitwarden_signing.pub"

# 3. Sign every commit by default
git config --global commit.gpgsign true

# 4. Use Windows OpenSSH ssh-keygen so it can talk to Bitwarden's named pipe
git config --global gpg.ssh.program "C:/Windows/System32/OpenSSH/ssh-keygen.exe"
```

The last line in that block is the one that matters most. Git for Windows bundles its
own MSYS2 `ssh-keygen`, which expects a Unix socket and cannot reach a Windows named
pipe, so signing fails silently or errors out. Pointing `gpg.ssh.program` at the
*Windows* OpenSSH binary makes it talk to Bitwarden directly.

Confirm the Git email matches a verified email on GitHub, otherwise commits sign but
show as **Unverified**:

```powershell
# 1. Check the configured email
git config --global user.email

# 2. If it is wrong, set it to a verified GitHub email
git config --global user.email "diogo@carvalhofer.lu"
```

For SSH remotes (skip if you push over HTTPS), the same named-pipe issue applies to the
transport:

```powershell
git config --global core.sshCommand "C:/Windows/System32/OpenSSH/ssh.exe"
```

### Step 7: Test in VS Code

VS Code uses the global Git config, so nothing extra to configure inside it.

1. Open a repo in VS Code.
2. Make a small change, stage it, commit via the Source Control panel.
3. Bitwarden pops up an authorization prompt, click **Authorize**.
4. Push the commit.
5. On GitHub, refresh the commit and confirm the **Verified** badge.

### Windows troubleshooting

| Symptom | Likely cause |
| --- | --- |
| Commit shows **Unverified** | Git email does not match a verified GitHub email, or the key on GitHub is set as **Authentication** instead of **Signing** |
| `gpg failed to sign the data` | Bitwarden Desktop not running, vault locked, or wrong `gpg.ssh.program` path |
| No Bitwarden authorization prompt | SSH agent not enabled in Bitwarden, or the Windows OpenSSH agent was not properly disabled |
| Commit signs but no prompt ever appears | "Ask for authorization" is set to **Never** instead of **Always** |
| Prompt appears when opening a VS Code window | VS Code Git auto-fetch, which is an *authentication* request rather than signing. Read the dialog text to tell them apart. Disable per-project with `"git.autofetch": false` in `.vscode/settings.json` |

### Good to know

- The key lives in the Bitwarden vault (cloud-synced), so it is available on every
  machine logged into the same account. One key, registered once on GitHub.
- Revoking the key on GitHub revokes it for **all** machines using it. Past commits keep
  their Verified status regardless.
- To isolate machines instead, generate a separate key in Bitwarden per machine and add
  each as its own Signing Key on GitHub.
- `gpg.ssh.program` and `core.sshCommand` are **Windows-only**. Never copy them into the
  WSL config, see the table in [6.2](#62-git-config-in-wsl).

---

## Part 8: The .ai-jail config

Written against **ai-jail 1.20.x**. Earlier versions had permissive defaults and a
project config that could grant capabilities, so a guide written for those is now wrong
in both directions. Check with `ai-jail --version`.

### Two files, and only one of them is trusted

| File | Authority |
| --- | --- |
| `./.ai-jail` (project) | **Untrusted and monotonic.** It may tighten the sandbox but can never enable a capability. Setting `network = true` here does nothing. |
| `~/.ai-jail` (global) | **Trusted.** A base table plus optional `[commands.<name>]` tables keyed by the first word of the command. |
| CLI flags | Highest authority. |

This is the single most important thing to internalise. A capability line in a project
file is silently ignored, so the file looks like it is doing something it is not. If a
setting is not taking effect, that asymmetry is the first thing to check.

To let specific checkouts ship their own capability opt-ins, list their parent directory
under `trust_project_config` in the global config. Keep that list narrow: everything at
or beneath a listed directory is trusted, including repositories cloned there later.

### The defaults are secure now

Off unless you ask: network, GPU, display, X11, host `/dev/shm`, terminal passthrough,
linked worktree metadata, Docker, SSH, Tailscale, the systemd user bus, and the status
bar's update check.

Two that catch people out:

- **Private home is on by default.** `$HOME` is a fresh tmpfs. Nothing under it is
  mounted unless you map it, which means no `~/.cache` and no mise toolchain.
- **Agent credential state is not mounted.** `~/.claude` and `~/.claude.json` need
  `--agent-state`, or Claude Code asks for a fresh login every launch and has no session
  history.

The environment is a minimal allowlist too, not your shell environment. Extend it with
`--env NAME`, or `env_pass` in the global config.

### My global config

Kept in this repo as [`.ai-jail-global`](.ai-jail-global), because a dotfile named
`.ai-jail` at the repo root would be the project config. Copy it to `~/.ai-jail`:

```bash
cp .ai-jail-global ~/.ai-jail
```

```toml
# ai-jail sandbox configuration in home folder: ~/.ai-jail
# https://github.com/akitaonrails/ai-jail
# Edit freely. Regenerate with: ai-jail --clean --init

no_status_bar = true

[commands.claude]
network = true
agent_state = true
terminal_passthrough = true
rw_maps = ["~/.cache"]
ro_maps = ["~/.config/mise", "~/.local/share/mise"]
```

| Line | Why |
| --- | --- |
| `network = true` | Claude Code cannot reach the API without it. Note what you are granting: unrestricted egress, so anything readable in the sandbox can leave. |
| `agent_state = true` | Mounts `~/.claude` and `~/.claude.json`, so the login and session history persist. The trade is real: that directory holds a live OAuth token and supports hooks that run shell commands, and everything in the sandbox can reach it. |
| `terminal_passthrough = true` | Output is filtered through a VT parser by default, which breaks full-screen TUI rendering. Raw forwarding fixes it and exposes the terminal's clipboard, query and parser surface. |
| `rw_maps = ["~/.cache"]` | Restores caches that private home would otherwise discard, Playwright browsers among them. |
| `ro_maps = [mise config, mise installs]` | mise needs both: the config says which versions are active, the installs hold the binaries. With neither present, ai-jail skips activation entirely. |

Both mise paths are read-only on purpose. The agent needs to run those tools, not
install new ones.

### My project template

```toml
# ai-jail sandbox configuration
# https://github.com/akitaonrails/ai-jail
# Edit freely. Regenerate with: ai-jail --clean --init

command = ["claude"]
hide_dotdirs = [
    ".azure",
    ".vscode-server",
]
mask = [
    ".env",
    ".env.local",
]
no_gpu = true
no_display = true
```

Everything here tightens, so it is honoured from an untrusted project file. It gets
adapted per project with extra masks for whatever secrets that repo actually holds.

Note what is **not** here. Capabilities like `network` and `terminal_passthrough` are
ignored in a project file, so writing them here would only mislead whoever reads it next.
They belong in `~/.ai-jail`.

`no_gpu` and `no_display` are redundant against current defaults, but harmless, and they
keep the file honest if a future version flips a default back.

| Option | Why |
| --- | --- |
| `mask` | Replaces matching project files with empty placeholders. The project directory is writable by default, so in-repo secrets need masking explicitly. `deny_paths` makes them inaccessible instead of empty. |
| `hide_dotdirs` | Largely moot under private home, since nothing from host `$HOME` is mounted anyway. Kept as belt and braces. |
| `no_display` | Was load-bearing on WSL before 1.20, when display passthrough mounted all of `XDG_RUNTIME_DIR` and dragged in VS Code's IPC socket. That is fixed upstream: only the validated Wayland socket is mounted now. Claude Code is a TUI, so this costs nothing either way. |

### Masks only cover files that already exist

A literal path missing at launch, or a glob matching nothing, is skipped with a warning,
and a file the agent creates later in the session is **not** covered. If you want the
rule enforced, create the file first:

```bash
touch .env .env.local
```

An empty file is enough. Globs sidestep the problem for variants you have not thought of,
so `mask = [".env", ".env.*", "*.pem"]` is worth considering over naming each file, as
long as you quote the pattern so your shell does not expand it first.

### Auto-save gotcha

`--save-config` is **on by default**: any flag passed on the command line is silently
written into the project `.ai-jail` and applies to every later run. This includes
`claude --resume <id>`, which gets recorded as part of `command` and pins that session
forever.

It interacts badly with the trust model. Pass `--network` once, and auto-save writes
`network = true` into the project file, where it is then ignored. The session works, the
next one does not, and the config file appears to say otherwise.

- Put ai-jail flags **before** the command, and avoid passing Claude's own flags after it
- Use `--no-save-config` for anything experimental
- Use `--init` to write a config deliberately
- When ai-jail behaves oddly, `cat .ai-jail` first, then `cat ~/.ai-jail`

### The one flag that cannot be saved

`--exec` is direct execution: no PTY proxy, no status bar. It is a runtime mode rather
than sandbox policy, so `--init` silently drops it and there is no `.ai-jail` key for it.
Every persistable option has a paired form (`--gpu / --no-gpu`, `--mise / --no-mise`);
`--exec` has no `--no-exec` twin, which is the tell.

If you want it every time, alias it:

```bash
echo "alias ai-jail='ai-jail --exec'" >> ~/.bash_aliases
source ~/.bashrc
```

Bash does not recurse on an alias that invokes its own name, so this is safe. `.ai-jail`
still supplies everything else, and `\ai-jail` bypasses the alias when you want the
status bar back.

`--exec` does not weaken the sandbox. It changes how output reaches your terminal, not
what the sandboxed process can reach: the PTY proxy sits outside the bwrap boundary, and
`--landlock-exec --landlock` is passed either way. What you lose is the status line that
shows the jail is active, so use `hostname` instead, which returns `ai-sandbox` inside.

For the same reason, neither `--exec` nor `--terminal-passthrough` appears in
`--dry-run` output. Both are outer-wrapper concerns that the landlock helper never sees,
so their absence there is not evidence they are off.

### How to create the files

Write them by hand. `--init` works, and it formats keys correctly, but it silently drops
what it cannot persist and it writes to the project file, which is the untrusted one.
Hand-written is more predictable for a config this small.

Verify what actually took, which is the part that matters:

```bash
ai-jail --dry-run claude | grep -E 'unshare-net|landlock-exec'
```

No `--unshare-net` means the network is on. The trailing arguments after
`/tmp/.ai-jail-landlock` are the resolved policy: `--agent-state`, `--network`,
`--private-home` and the rest appear there explicitly.

### Running it

```bash
cd ~/Projects/my-app
ai-jail --dry-run claude    # inspect the mount plan
ai-jail claude              # run for real
```

To resume a previous session, either use `/resume` from inside Claude Code, or pass the
flag with auto-save disabled so the session ID is not written into `.ai-jail`:

```bash
ai-jail --no-save-config claude --resume 11f2b4c6-625a-4650-b0c2-96cf276f91f4
```

Workflow: the agent edits inside the jail, you commit from a normal terminal.

### Why commits happen outside the jail

Two things are missing inside it. `user.signingkey` points at
`~/.ssh/bitwarden_signing.pub`, and `~/.ssh` is never mounted, so that path does not
exist. `SSH_AUTH_SOCK` is not forwarded either, so `ssh-keygen -Y sign` has nothing to
talk to. Since `commit.gpgsign = true` comes from the read-only `.gitconfig`, a plain
`git commit` in there fails outright. It can still commit unsigned with
`--no-gpg-sign`, which is worth knowing: an unsigned commit appearing in your history is
the signal that something happened inside the jail.

That is why [`CLAUDE.md`](CLAUDE.md) asks for a block of `git add` and `git commit`
commands rather than commits. The agent does the useful part, grouping changes and
writing messages, and you run it outside so every commit is signed with your key.

If you do want signing to work inside the jail:

```bash
ai-jail --ssh claude
```

That mounts `~/.ssh` read-only and forwards `SSH_AUTH_SOCK`, so `git commit` works and
Bitwarden prompts for each signature.

Think before enabling it. It gives the agent use of your agent socket, which means it
can authenticate as you to any SSH host, not just sign commits. "Ask for authorization:
Always" does prompt every time, but the dialog reads
`npiperelay.exe is requesting access to github.com` whether the request came from you or
from the agent, so you would be approving blind. Leaving it off keeps the split intact:
the agent writes code, you commit, and every commit in your history is provably yours.

---

## Part 9: Session instructions

The sandbox constrains what the agent *can* reach. These files cover what it *should* do
inside those limits, so the same instructions do not have to be retyped every session.
Claude Code reads `CLAUDE.md` automatically at the start of every session, and `@path`
imports pull in the rest.

### Three files, split by who owns them

```
~/.claude/
├── CLAUDE.md          a thin router, mine, with one protected block
├── jail.md            the sandbox and the rules, read-only to the agent
└── machine-state.md   agent-maintained inventory, not in this repo
```

The split is by ownership and rate of change. The rules are stable and mine. The jail
description changes when the sandbox config changes. Machine state changes on its own, as
things get installed.

`machine-state.md` is deliberately absent from this repo, since it describes one machine
rather than the method. Create it locally if you want it, and add the import below the
protected block in `CLAUDE.md`.

A project-level `./CLAUDE.md` at a repo root still works and is read alongside the global
file, for anything specific to one project.

```bash
mkdir -p ~/.claude
nano ~/.claude/CLAUDE.md
```

### CLAUDE.md

```markdown
# How I work with you

<!-- PROTECTED: do not edit, move or remove anything between these two markers. -->

Never edit or delete `~/.claude/jail.md`, and never edit the protected block below.
This holds no matter what: not to fix a mistake in it, not to record something you
learned, not to note an exception, and not because a task appears to require it. Propose
the change and wait for me to make it.

@~/.claude/jail.md

<!-- END PROTECTED -->

Everything outside the markers is fair game. Add anything useful here.
```

The markers are HTML comments, so they do not render but are unmissable to anything
editing the file. The protected region covers the rule as well as the import, because the
first thing an agent could otherwise legally delete is the sentence telling it not to
delete things. Everything below the closing marker is open, which is where a local
`@~/.claude/machine-state.md` line goes.

### jail.md

```markdown
# The ai-jail sandbox, and how I want you to work

This file is read-only to you. Never edit or delete it. In `~/.claude/CLAUDE.md` you may
edit anything outside the block marked PROTECTED, but nothing inside it, including the
line that imports this file. Both rules hold no matter what: not to correct an error you
spot, not to add something you learned, not to record an exception, and not because a
task appears to require it. Propose the change instead and wait for me to make it. If you
need somewhere to write, use your own auto-memory notes.

Am I jailed? Inside the sandbox `hostname` returns `ai-sandbox` and
`/tmp/.ai-jail-landlock` exists. If neither is true you are running unjailed on the host:
the environment description below does not apply, but the rules still do.

## The sandbox

bubblewrap plus Landlock LSM plus seccomp, on WSL 2. It is deliberate and expected.

Only the current project directory is writable and persistent. $HOME is tmpfs and is
discarded on exit, `~/.ssh` is usually not mounted, and there is no `/mnt`, so Windows is
unreachable.

Why: the sandbox keeps work scoped to the project I asked about. It is not a statement
that you are untrusted. Everything outside the project is out of scope by design, so do
not try to escape it.

## Scope

- Files inside the working directory are yours to change as needed. Anything else outside
  the project, including my system, is off limits: mention it and wait rather than
  touching it.
- In `~/.claude` you may write your own auto-memory notes without asking. Record what is
  true rather than how you worked around something missing. Anything else there is mine
  unless I have pointed you at it explicitly.
- If a new notes or instruction file would help, propose it first: what it is for, what
  would go in it, and where it would be imported. Do not create the file, and do not add
  an import line for it, until I have agreed.

## Setting things up

- Setting up whatever a task needs is allowed while it stays ephemeral and
  self-contained: installing dependencies, fetching tools or runtimes, unpacking files
  into the scratchpad and pointing environment variables at them. $HOME and /tmp are
  tmpfs, so all of it disappears when the jail exits and costs nothing permanent.
- Say what you set up and why in the same turn, especially when it was large or took
  several steps. Often it can be made permanent instead of repeated every session:
  anything installed on the WSL host is visible inside the jail because `/usr` is mounted
  read-only from it, and `~/.cache` is writable and survives between sessions.
- Stop and ask when the fix needs something the jail deliberately withholds: anything
  requiring sudo or root, anything system-level, anything outside the project directory
  and `~/.claude`. sudo is inert here by design, so finding that you need it is the
  signal to stop rather than a problem to route around.
- If something genuinely needs a system package, name it and say what it unblocks. I
  install those manually on the host, and only when they are strictly necessary, so
  expect me to decline anything that only makes sense for a single task.

## Processes

- You may start servers, watchers and other long-running processes when they help you
  check your work. Bind anything that listens to 127.0.0.1 only, never 0.0.0.0, since
  this machine is on a tailnet.
- Stop anything you started once you are done with it. Nothing you launched should still
  be running when you hand back to me. This applies only to your own processes.
- I may have my own processes running in other shells, including servers on common
  ports. Do not try to kill them, and pick a port that is free. If a change of yours
  means one of mine needs restarting, tell me and I will do it manually.

## Commits

- Before ending any turn in which you changed files, output the commit block. Do not
  wait for me to ask and do not defer it to a later turn.
- Do not commit yourself by default. I want to review changes before they enter history,
  and I run the commits so they are signed with my key. Signing cannot work inside the
  jail anyway, since `~/.ssh` is not mounted.
- When changes are ready, output one bash block of `git add` and `git commit` commands
  with the messages written, then stop. Do not run it.
- Judge the number of commits per case. Split into several when the changes are
  genuinely separate concerns, and use a single commit when that is all the work
  amounts to. Do not split for the sake of splitting.
- The exception: if I explicitly ask you to commit during a session, go ahead. I will
  approve each signature in Bitwarden.
```

### Why it is worded this way

- **The jail check comes first.** The global file loads in *every* session, including
  unjailed ones, so a file that opens by asserting "$HOME is tmpfs" is simply wrong half
  the time. Inside the sandbox `hostname` returns `ai-sandbox` and
  `/tmp/.ai-jail-landlock` exists, both straight from ai-jail's own bwrap invocation. The
  check gates the environment description only, so the rules still apply everywhere.
- **Explaining that the sandbox is intentional matters more than it looks.** An agent that
  hits a missing path with no context reads it as a broken environment and starts working
  around it. Told the restriction is deliberate, it reports the problem instead, which is
  what you want.
- **The setup rules are split into three deliberately.** A blanket "do not work around the
  sandbox" collapses two different situations. Downloading a browser into tmpfs to take a
  screenshot is ordinary dev work and costs nothing permanent, so it is allowed. Needing
  sudo or a system-level change is the sandbox saying no, so that stops. The middle rule
  exists because the same workaround repeating every session is usually a sign it should
  be solved properly instead: `/usr` is bind-mounted read-only from the host, so
  `sudo apt install` outside the jail makes libraries permanently visible inside it, and
  `~/.cache` is bind-mounted read-write rather than tmpfs, so downloads landing there
  survive between sessions.
- **The system-package rule invites a request rather than promising a yes.** Host installs
  are permanent and accumulate, so they are worth it for a recurring dependency and not
  for a one-off.
- **The instruction files are protected explicitly.** In practice an agent will edit the
  file containing its own rules for entirely defensible reasons: it learned something
  durable and machine-wide, and that file was the only globally loaded place to put it.
  Naming that specific justification as insufficient matters more than a general "do not
  edit", which reads as having reasonable exceptions. The same applies to creating new
  instruction files: proposing one is fine, wiring it into `CLAUDE.md` without asking is
  not, because that changes what loads in every future session.
- **The commit block has an explicit trigger.** "When changes are ready" turned out to be
  vague enough to skip, so the rule names the moment: before ending any turn in which
  files changed.
- **The rules avoid naming specific tools.** They are written in terms of what an action
  costs, whether it is ephemeral or permanent, inside the project or outside it, rather
  than listing package managers or frameworks. A rule phrased around one tool gets read as
  not applying to the next one.
- **Running processes is allowed on purpose.** Checking your own work by starting a dev
  server is useful, and `--die-with-parent` plus the unshared PID namespace means anything
  it starts dies with the jail. No orphans on the host. The cleanup rule is about the
  session rather than safety: a forgotten server holds its port for the rest of the
  session, and the next thing that needs that port fails for no obvious reason.

### Auto memory

Claude Code also writes notes for itself, unprompted, into
`~/.claude/projects/<repo>/memory/`. Those are project-scoped, so a lesson about the
machine rather than the project will not travel between repos, which is what
`machine-state.md` is for. Review them with `/memory` now and then: a wrong note is worse
than no note, and only the first 200 lines of `MEMORY.md` load at startup.
`CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` turns the feature off entirely.

A useful header for `machine-state.md`, since it is the one file the agent maintains:
record what is installed and where rather than how to work around something missing,
verify an entry before relying on it, date each entry, and delete anything that no longer
holds. That last part matters more than it sounds, because host state can quietly stop
being true after a distro rebuild while the file keeps loading in every session.

### One limit worth stating

`CLAUDE.md` is context, not enforcement. It is guidance the agent follows, not something
the kernel blocks. `chattr +i ~/.claude/CLAUDE.md ~/.claude/jail.md` makes edits fail at
the filesystem level if you want it actually enforced, at the cost of `chattr -i` whenever
you edit them yourself.

---

## Maintenance

| What | Command |
| --- | --- |
| Everything on the WSL side | `all-update` (alias below) |
| npiperelay, everything Windows | `winget upgrade --all` |

Aliases worth having, both in `~/.bash_aliases`:

```bash
echo "alias all-update='sudo apt update && sudo apt full-upgrade -y && sudo apt autoremove -y && mise self-update && mise upgrade && claude update'" >> ~/.bash_aliases
source ~/.bashrc
```

Ubuntu's stock `.bashrc` sources `~/.bash_aliases` automatically, so nothing else is
needed. Run `all-update` from any shell, outside the jail. The `ai-jail` alias is covered
in [Part 8](#the-one-flag-that-cannot-be-saved).

`claude update` goes last on purpose. With `&&` chaining a failed step skips everything
after it, so a hiccup in the Claude Code updater cannot block your system upgrades.

### Claude Code cannot update itself inside the jail

Native installations auto-update in the background, and `claude update` forces it
immediately. Both write to the binary at `~/.local/bin/claude`, and the dry run shows
that path mounted read-only:

```
--ro-bind /home/diogo/.local /home/diogo/.local
```

So the update fails inside the jail, silently or with a warning. This is deliberate
rather than a gap: read-only `~/.local` is the same rule that keeps the ai-jail binary
itself out of the agent's reach and blocks PATH shadowing.

Run `claude` unjailed now and then, or let `all-update` handle it. `claude doctor`
reports the result of the most recent update attempt, which is the quickest way to check
a version that looks stale.

After a npiperelay upgrade, restart the bridge in each open shell:

```bash
pkill -f 'UNIX-LISTEN:.*agent.sock' && rm -f ~/.ssh/agent.sock.* && exec bash
```

Or simply `wsl --shutdown` from PowerShell. The symlink target survives upgrades,
because the winget package path is stable.

---

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| `bwrap: Can't create file at /etc/resolv.conf` | `/etc/resolv.conf` is a symlink | [Part 1.2](#12-fix-etcresolvconf-required-for-ai-jail) |
| `bwrap: setting up uid map: Permission denied` | Ubuntu 24.04+ AppArmor blocks unprivileged user namespaces | `sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0` (persist via `/etc/sysctl.d/`, needs systemd enabled in `wsl.conf`) |
| `ssh-add -l` gives `Could not open a connection` | Relay not running, or Bitwarden locked / agent disabled | `exec bash`, then unlock Bitwarden Desktop |
| `ssh-add -l` gives `agent has no identities` | Bridge works, no key in the agent | Add an SSH key item in Bitwarden |
| `git@github.com: Permission denied (publickey)` | Key registered as Signing only | [Part 6.1](#61-register-the-key-on-github-twice) |
| `Username for 'https://github.com':` prompt on clone | Remote is an HTTPS URL, and GitHub dropped password auth in 2021 | Clone the `git@github.com:` URL, or set the `insteadOf` rewrite in [Part 6.2](#62-git-config-in-wsl) |
| `Permission denied (publickey)` when a tool clones a public repo | An unscoped `insteadOf` rewrote its HTTPS URL to SSH, and the subprocess has no agent socket | Scope the rewrite to your own account, see [Part 6.2](#62-git-config-in-wsl) |
| Commit shows **Unverified** on GitHub | Key not registered as a Signing key, or email mismatch | [Part 6.1](#61-register-the-key-on-github-twice) |
| `gpg.ssh.allowedSignersFile needs to be configured` | Local verification not set up, signing itself is fine | [Part 6.3](#63-local-verification-optional) |
| A capability set in the project `.ai-jail` has no effect | Project config is untrusted and can only tighten | Move it to `~/.ai-jail`, see [Part 8](#two-files-and-only-one-of-them-is-trusted) |
| Claude Code cannot reach the API | Network is off by default since 1.20 | `network = true` under `[commands.claude]` in `~/.ai-jail` |
| Claude Code renders inline instead of full screen | Output is filtered through a VT parser by default | `terminal_passthrough = true`, or the `--exec` alias in [Part 8](#the-one-flag-that-cannot-be-saved) |
| mise tools missing inside the jail | Private home means neither mise config nor installs are mounted | `ro_maps` for both paths, see [Part 8](#my-global-config) |
| Claude Code asks to log in every run | Agent state is not mounted by default since 1.20 | `agent_state = true` under `[commands.claude]` in `~/.ai-jail` |
| Distro stays running after closing all terminals | A detached process: an orphaned relay from a force-killed shell, a dev server, VS Code Server, or Docker Desktop integration | `ps -eo pid,etime,cmd --sort=-etime \| head` to find what is old, then `wsl --shutdown`. The sweep in [Part 5.3](#53-the-bashrc-block) clears orphaned relays on the next shell |
| Bitwarden prompts on VS Code window focus | VS Code Git auto-fetch, not signing | `"git.autofetch": false` in `.vscode/settings.json` |

---

## Security notes

### What the jail does protect

Verified empirically from inside a running jail:

- **Project-only persistent writes.** Private home is the default, so `$HOME` is a fresh
  tmpfs discarded on exit and nothing under host `$HOME` is mounted unless you map it
- **`~/.ssh` needs `--ssh`**, off by default, so keys and the agent socket are out of
  reach
- **No `/mnt`**, so the Windows filesystem and all Windows executables are unreachable
- **PID namespace unshared**, so host processes are invisible and cannot be signalled
- **Empty capability bounding set**, `NoNewPrivs=1`, every mount `nosuid`, so setuid
  binaries are inert
- **Around 40 syscalls blocked** by seccomp: the whole mount family, `ptrace`, `bpf`,
  `unshare`/`setns`, `io_uring`, keyring, `kexec`, module loading
- **Landlock LSM** enforced at VFS level, on top of the mount layout
- **`~/.local` read-only**, so the ai-jail binary itself is out of reach and PATH
  shadowing is blocked

### What it does not protect

- **Network egress is unfiltered when you enable it.** bwrap either unshares the network
  namespace or does not; there is no middle ground. `--network` is all or nothing, and
  Claude Code needs it, so anything readable in the sandbox can leave. `--allow-tcp-port`
  is still accepted for compatibility but now fails closed, because UDP cannot be
  constrained through it. `--lockdown` blocks network and makes the project read-only,
  which suits review work rather than coding.
- **`--agent-state` hands over a live credential.** `~/.claude` holds
  `.credentials.json`, an OAuth token, and supports hooks that run shell commands, which
  is a persistence path into future *unjailed* sessions. It is opt-in now, but Claude
  Code is impractical without it. `chattr +i ~/.claude/settings.json` closes the hook
  path specifically, at the cost of `chattr -i` whenever you change a setting.
- **Every map you add is a hole you chose.** `rw_maps = ["~/.cache"]` is convenient and
  low-value, but it is host state the agent can write. Audit the list occasionally rather
  than letting it grow.
- **The agent can open listening ports** once network is on. Check from the host:
  `ss -lptn | grep -v 127.0.0.1`
- **No default credential denylist.** Every `mask` is operator-supplied, and masks only
  cover files that exist at launch. The threat model is *contain the host blast radius*,
  not *keep secrets from the agent*.
- **Kernel escapes are out of scope**, per the author. For genuinely untrusted code, use
  a disposable VM.

### What 1.20 fixed

Three of these were findings from an earlier version of this guide, so they are worth
recording as closed rather than deleted:

- **Display no longer mounts all of `XDG_RUNTIME_DIR`.** Only the validated Wayland
  socket is bound, and X11 moved behind a separate `--x11`. On WSL the old behaviour
  pulled in `/mnt/wslg/runtime-dir` with VS Code Server's `vscode-ipc-*.sock`, which was
  a real escape path: anything able to write that socket could drive the host editor and
  spawn processes outside the jail.
- **Host `/dev/shm` is off by default**, behind `--host-shm`. It used to be bind-mounted
  at mode 1777, a bidirectional channel outside Landlock's file rules by construction.
- **Agent credentials are no longer mounted by default.** `~/.claude` used to be
  read-write with no way to hide it.

Also new: the environment is a minimal allowlist rather than your shell's, and the status
bar's version check no longer makes an outbound request unless you ask for it.

### Scope creep is not a security control

The sandbox constrains *where* the agent can act, not *how much* it does. For bounding
behaviour, use Claude Code's own permission system (`/permissions`, written to
`.claude/settings.local.json`) and tighter prompts.

---

## Known upstream issues

Worth reporting to [akitaonrails/ai-jail](https://github.com/akitaonrails/ai-jail/issues)
if they still reproduce on your version:

1. **`/etc/resolv.conf` symlink breaks startup on WSL.** ai-jail already binds
   `/mnt/wsl/resolv.conf`, so it has some WSL awareness, but it does not handle
   `/etc/resolv.conf` being a symlink to it. Covered in
   [Part 1.2](#12-fix-etcresolvconf-required-for-ai-jail).
2. **TUI apps cannot use the alternate screen.** `--dev /dev` gives the sandbox a fresh
   devpts instance, so the inherited pty has no node inside it: `isatty()` still passes
   because the fd is a real pty, but `ttyname()` fails because there is nothing to name.
   Applications that resolve their terminal by name fall back to inline rendering, which
   leaves your shell scrollback interleaved with the session. Repro is two lines, `tty`
   inside the jail versus outside. Not WSL-specific. `--rw-map /dev/pts` restores it at
   the cost of letting the jail write into your other terminal sessions; `--exec` avoids
   the PTY proxy instead and costs nothing.
3. **`--init` silently drops flags it cannot persist**, `--exec` among them. It already
   warns when a project config grants something your flags did not, so the mechanism for
   saying something exists.

**Fixed upstream**, both reported here against earlier versions: display passthrough
mounting all of `XDG_RUNTIME_DIR` and exposing VS Code IPC sockets on WSL, and host
`/dev/shm` being bind-mounted by default. See
[What 1.20 fixed](#what-120-fixed).

---

## License

[CC0 1.0 Universal](https://github.com/DiogoF-Hub/claude-code-wsl-sandbox/blob/main/LICENSE). Public domain, no attribution required, though it is always appreciated.