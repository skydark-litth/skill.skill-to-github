# One-Time Environment Setup (skill-to-github)

Covers installing Git for Windows without admin rights, configuring it, generating an SSH key, and linking it to GitHub. Run once per machine. The sync workflow in SKILL.md assumes this is already done; verify with `ssh -T git@github.com`.

## 1. Install Git for Windows (user scope, no admin)

```
winget install --id Git.Git -e --source winget --scope user --silent --accept-package-agreements --accept-source-agreements
```

- `--scope user` installs to `%LOCALAPPDATA%\Programs\Git` and does NOT trigger a UAC prompt. A plain `winget install Git.Git` (machine scope) requests admin and hangs in non-interactive shells.

## 2. Add Git to the user PATH (persist)

Add these two directories to the **user** (not system) PATH:

- `C:\Users\USER\AppData\Local\Programs\Git\cmd`
- `C:\Users\USER\AppData\Local\Programs\Git\usr\bin`  (provides `ssh`, `ssh-keygen`, `scp`)

Do this via PowerShell (no admin needed):

```powershell
$adds = @("C:\Users\USER\AppData\Local\Programs\Git\cmd", "C:\Users\USER\AppData\Local\Programs\Git\usr\bin")
$p = [Environment]::GetEnvironmentVariable("Path","User") -split ';' | Where-Object { $_ }
foreach ($a in $adds) { if ($p -notcontains $a) { $p += $a } }
[Environment]::SetEnvironmentVariable("Path", ($p -join ';'), "User")
```

## 3. git global configuration

```
git config --global init.defaultBranch main
git config --global core.autocrlf false        # CRITICAL: protects Python script line endings
git config --global pull.rebase false
git config --global core.editor "notepad"
```

Git Credential Manager ships with Git for Windows and is the default helper — no extra setup for HTTPS if ever needed.

## 4. Generate the SSH keypair

```
mkdir C:\Users\USER\.ssh
ssh-keygen -t ed25519 -C "USER@workbuddy" -f C:/Users/USER/.ssh/id_ed25519 -N ""
```

- Use the **Windows-native path** `C:/Users/USER/.ssh/id_ed25519` (not `/c/Users/...`); msys path translation breaks `ssh-keygen`.
- After generation, tighten permissions from Git Bash: `chmod 600 C:/Users/USER/.ssh/id_ed25519`.
- Print and share only the public key: `cat C:\Users\USER\.ssh\id_ed25519.pub`

## 5. Add the public key to GitHub

- Go to https://github.com/settings/ssh/new
- Paste the contents of `id_ed25519.pub`, give it a recognizable title (e.g. `WorkBuddy-PC`), click Add SSH key.
- Verify: `ssh -T git@github.com` → `Hi <user>! You've successfully authenticated...`

## 6. Set the git commit identity

```
git config --global user.name  "<GitHub username>"
git config --global user.email "<id>+<username>@users.noreply.github.com"
```

- If the user enabled GitHub "Keep my email private", use the `noreply` address shown under GitHub → Settings → Emails → Email privacy, so the real email never appears in public commits.
- If using a real email, it must match the GitHub account or commits will not link to it.

## Security notes

- The private key (`id_ed25519`) stays only on this machine. Never copy, upload, screenshot, or commit it. Only the `.pub` file is shareable.
- Prefer SSH (no tokens needed). If using HTTPS + PAT, let Git Credential Manager cache it; never hardcode tokens in commands or scripts. Prefer a fine-grained PAT scoped to the single repo when possible.
- Do not place `.ssh` or any token file under version control.
