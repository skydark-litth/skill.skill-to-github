---
name: skill-to-github
description: "Sync a WorkBuddy user-level skill directory to a GitHub repository using git over SSH. Use when the user asks to publish, update, back up, version-control, or reconcile a local skill with a GitHub repo. The user must explicitly name both the local skill and the GitHub project; if the repo does not exist, guide the user to create it on the GitHub website. Runs a leak/privacy audit before every upload and reports any concern to the user for a decision. Ensures README.md exists and is current — generating it when missing, or updating it from the old README plus the new skill's changes when the repo README's version is older than the skill's, always on user approval. Relies on git's built-in integrity checks rather than re-downloading files, cleans up the local clone after a verified push, and self-updates when it hits a problem it does not yet cover."
agent_created: true
version: 2.5.2
---

# skill-to-github

## Overview

Mirror a WorkBuddy user-level skill folder (typically `C:\Users\USER\.workbuddy\skills\<skill-name>\`) to a GitHub repository. The skill enforces explicit targeting (which local skill, which GitHub project), ensures the target repo exists (guiding the user to create it on the website when it is missing), audits for leaks/privacy issues before every upload and reports any concern to the user for a decision, keeps `README.md` current — generating it when missing, or updating it from the old README plus the new skill's changes when the repo README's version is older than the skill's, always on user approval — verifies with git's own integrity mechanisms, and records any newly discovered fixes into itself.

## When to Use

- User asks to "把 <skill> 同步/更新/发布到 github", "把 skill 上传到 GitHub", or "存个备份到仓库".
- User wants to version-control a user-level skill, or reconcile an existing repo with the local copy after edits.

## Step 0 — Confirm both targets explicitly (mandatory)

Never infer the skill or the repo. Ask until the user explicitly provides:

1. **Local skill — full path.** Do not sync "the skill" by guesswork. Verify the path exists and read its `SKILL.md`; confirm the `name:` (and `version:` if present) matches what the user intended before doing anything else.
2. **GitHub project — exact `owner/repo`.** Do not default to a similarly named repo. Confirm whether it should be public or private.
3. **Git/SSH readiness.** A working `git` that pushes over SSH is required — verify with `ssh -T git@github.com`. One-time install/config is in `references/setup.md`.

Only proceed after both identifiers are stated and the local skill is confirmed. This prevents reading the wrong skill or overwriting the wrong project.

## Step 1 — Resolve the GitHub repo (create it if missing)

Probe whether the target already exists:

```
git ls-remote git@github.com:<owner>/<repo>.git
```

Read that probe's **own** exit status, not a piped command's: `git ls-remote ... | head` reports `head`'s status, which is always 0, and an **existing-but-empty** repo also prints nothing and exits 0. So the only reliable signals are: **exit 0 + no output = exists but empty**; **non-zero + message = missing or no access**. When a probe comes back ambiguous, re-run it capturing the status directly:

```
out=$(git ls-remote git@github.com:<owner>/<repo>.git 2>&1); code=$?
```

- **Exists** → clone it to a fresh working directory under the current workspace:
  ```
  git clone git@github.com:<owner>/<repo>.git <workdir>
  ```
- **Does not exist** → guide the user to create it on the GitHub website, then wait for confirmation before continuing:
  1. Give the direct link: https://github.com/new
  2. Tell them exactly what to enter: **Repository name** = `<repo>`; choose the owner matching `<owner>`; set Public/Private as previously confirmed; leave "Initialize this repository" **unchecked** (so it stays an empty repo with no conflicting initial commit).
  3. Click **Create repository**, then tell the agent it is done.
  4. Re-run `git ls-remote` to confirm it now exists, then clone as above.

> An SSH key alone cannot create a remote repo — it only reads/writes repos that already exist — so creation is done by the user on the website. This keeps the flow free of any extra token.

## Step 2 — Ensure README.md exists and is current

Before each upload/update, check `README.md`. The repo's current README is available in the clone after Step 1; the local skill may or may not have one. Compare versions when the skill declares one.

1. **Local skill has no `README.md`** → ask the user whether to auto-generate it.
2. **Version comparison (when the skill has `version:` in `SKILL.md`)** → read the repo's README (from the clone) and the version it states (typically a `当前版本` / `Version:` line); compare with the local skill's `version:`:
   - **Repo README version < local skill version** → ask the user: "远程 README 的版本（vX）比技能版本（vY）旧，是否调用大模型，按照旧版 README 与新版技能的变化来更新 README？"
     - On approval: read the repo's **old README** and the local **new `SKILL.md`** (plus supporting files); have the model update the README by applying what changed between the two skill versions — keep the old README's structure and tone where still valid, refresh the version number, features, and workflow to match the new skill. Write the result into the **local skill directory** so it becomes part of the skill and is uploaded, then continue.
     - If the user declines: keep the repo's old README as-is (do not overwrite it) and proceed.
3. **Present but no version signal, or otherwise possibly stale** → spot-check that the described requirements and features match the current `SKILL.md`; when in doubt, treat it as not-current.
4. If missing or not-current, **ask the user**: "README.md 缺失/可能已过期，是否调用大模型总结该技能的要求与功能，自动生成 / 更新？"
   - On approval, read the full `SKILL.md` (and supporting files) and have the model summarize the skill's purpose, requirements, features, workflow, and usage into a clear `README.md`. Write it into the **local skill directory** so it becomes part of the skill, then continue.
   - If the user declines, leave the README as-is and proceed.

## Step 3 — Check for leaks or privacy issues (before every upload/update)

Run a **leak / privacy audit** on the local skill before overlaying anything. Check at minimum:

1. **File inventory.** List all files in the skill. Confirm nothing unintended is present: no key files (e.g. `id_ed25519`, `*.pem`), no credential/config files, no personal archives or logs. If anything unintended appears, stop and report to the user.
2. **Secret patterns.** Scan file contents for credential markers — `BEGIN (RSA|OPENSSH|EC|PRIVATE) KEY`, `ghp_`/`gho_` (GitHub tokens), cloud keys, `password=`/`api_key=`/`Bearer` — and confirm any hits are **documentation examples only**, never real values.
3. **Absolute personal paths.** Flag hardcoded absolute paths that identify the user's machine / account (e.g. `C:\Users\USER\...`, `.ssh`, `.venv`). In a user-level setup skill these are often intentional example paths — keep them only if they are clearly illustrative.
4. **Git artifacts.** Ensure there is no `.git` directory inside the skill (a nested repo would leak history).

If any check raises a concern — a real secret, an unexpected file, or a privacy leak — **do not proceed**. Report the finding(s) to the user with what was found and the risk, and wait for their explicit decision before continuing. Only proceed (or proceed with the user's chosen remediation) after they decide.

## Step 4 — Overlay the local skill into the clone

```
cp -r "<skill-dir>/." "<workdir>/"
```

Copy the directory *contents* (trailing `/.`) so repo-only files the local skill does not have are preserved, and the repo's `.git` is untouched.

> ⚠️ Exception: if the source directory **contains its own `.git`** (e.g. it is a clone of an upstream repo, or you are publishing an arbitrary project folder rather than a skill folder), **do NOT** use `cp -r src/. dst/` — that copies the source `.git` over the clone and corrupts/replaces the target repo's `.git`. Instead copy the intended files explicitly (excluding `.git`), or use `tar`/`rsync --exclude=.git`.

## Step 5 — Verify with git's built-in checks (no re-download)

Do not re-download files for byte comparison. git already guarantees object integrity end to end (every blob is content-addressed by SHA-1; push transfers the same objects). Use:

```
cd "<workdir>"
git status --short
git diff --stat
git fsck --full
```

- `git status` / `git diff` show exactly what changed; byte-identical files show no diff. Confirm the changed set matches expectation.
- `git fsck --full` validates local object/database integrity (expect no errors).
- Because the working files are the same source files and git hashes content, a clean `fsck` plus the expected diff set is sufficient verification. Skip the external curl-download + sha256 step.

## Step 6 — Commit and push; confirm via the push result

```
git add -A
git commit -m "Sync <skill-name> (mirror local)"
git push origin main
git ls-remote origin
```

- Treat the **push output itself** as confirmation: it must exit 0 and show the ref advancing (e.g. `a0ac6e9..b767b10 main -> main`).
- **A target that is already identical has nothing to push.** Check the changed set before committing (`diff -rq --exclude=.git <clone> <skill-dir>`, or Step 5's `git status`); if it is empty, report "already in sync" and skip `commit`/`push` — an empty `git commit` fails, and manufacturing an empty commit serves no purpose.
- Optionally `git ls-remote origin` and confirm `refs/heads/main` equals the commit just pushed. GitHub confirms the ref update on push, so this is authoritative — no extra local clone is required.

## Step 7 — Clean up the local clone after a verified push

After Step 6 confirms a successful push (`git push` exits 0 and the ref advances), the local clone working directory `<workdir>` has served its purpose. To save disk space, **remove it automatically — no need to ask the user**:

```bash
# Git Bash
rm -rf "<workdir>"
# PowerShell
Remove-Item -Recurse -Force "<workdir>"
```

Rules:
- Delete **only** the exact `<workdir>` created in Step 1. Never touch the original local skill directory (`<skill-dir>`) or any sibling/other path.
- Delete **only after a verified successful push**. If the push failed or was aborted, **keep** `<workdir>` — it still holds unpushed changes and is the natural retry point.
- In the final summary, state whether `<workdir>` was removed or retained (and why).

## Step 8 — Self-update when the skill lacked an answer

If this run hits a problem that the skill (this file or `references/setup.md`) does **not** already cover:

1. Solve the concrete problem first and confirm the sync succeeds.
2. Then update the skill so the next run does not rediscover it: add the symptom → cause → handling to the relevant section (workflow step, `Key Pitfalls`, or `setup.md`). Keep each addition minimal and directly tied to the problem.
3. Bump `version:` when the change alters behavior or workflow.
4. Do not add speculative fixes for problems that did not occur — only record what was actually encountered and verified.

## Key Pitfalls

- **Explicit targets first.** Never sync without the user naming the exact local skill and `owner/repo`; verify the local `SKILL.md` name before overlaying.
- **`core.autocrlf=false` is mandatory.** Without it, Git for Windows rewrites line endings and corrupts Python scripts.
  - To *measure* line endings, count bytes (`python -c` / `od -c`); **do not trust `grep -c $'\r'` in Git Bash** — the escape can degrade to an empty pattern, which matches every line and reports the line count instead of 0, producing a false "CRLF everywhere" alarm.
- **Creating a repo is done on the GitHub website**, not with the SSH key: send the user to https://github.com/new with the exact name/owner/visibility, then confirm via `git ls-remote` and clone.
- **README is checked before every push**: generate if missing; when the repo README's version is older than the local skill's, ask the user before letting the model update it based on the old README + the new skill's changes. Never auto-write without the user's decision.
- **Verify via git (`status`/`diff`/`fsck`) and the push result**, not by re-downloading.
- **A rendered / injected copy of this skill file may not match the disk.** When the skill is loaded, backticked sequences in the file can be command-substituted, so the injected copy can show shell error text (e.g. "command not found") where the file on disk actually holds the intended characters. Always read the file itself before editing — never "fix" text that only looks broken in the injected copy.
- **Audit before every upload/update**: file inventory, secret patterns, absolute personal paths, no nested `.git`. Any concern → stop, report to the user, wait for a decision.
- **Preserve repo-only files:** use `cp -r src/. dst/` (trailing `/.`), never `cp -r src dst` — **except** when the source has its own nested `.git`, in which case copy the intended files explicitly and exclude `.git` (a blanket copy would clobber the clone's `.git`).
- **Non-skill project folders:** the flow also works for arbitrary local directories; Step 2's version comparison does not apply (project READMEs typically have no `version:`), and when the target repo is **public**, additionally flag license/redistribution risk if the project derives from upstream code that has no LICENSE.
  - **License-risk mitigation (leave the choice to the user):** when the user still wants a public repo, offer excluding the upstream-derived files (a common carrier is an upstream-shipped `autounattend.xml`) — the project often re-fetches it at runtime, so the repo stays functional. Write that `.gitignore` into the **local source directory**, not only the clone, so Step 4's overlay carries it into the repo and it survives the next sync. Verify via `git status --short --ignored` that the excluded file shows as `!!` and is absent from `git diff --cached`.
- **Empty target repo:** `git clone` warns "You appear to have cloned an empty repository" and yields no commits; the clone's HEAD already tracks `main`, so `git add -A` → `git commit` → `git push -u origin main` works directly (no need to pre-create the branch).
- **`ssh-keygen` needs a Windows-native path** (`-f C:/Users/USER/.ssh/id_ed25519`); an msys `/c/...` path fails.
- **`winget install Git.Git` requires `--scope user`** to avoid an admin/UAC prompt that hangs non-interactive shells.

## Verification Checklist

- [ ] User explicitly named the local skill (path verified, `SKILL.md` name confirmed) and the exact `owner/repo`.
- [ ] Target repo exists (cloned), or was created by the user on https://github.com/new and confirmed via `git ls-remote`, then cloned.
- [ ] **Leak/privacy audit passed** (file inventory clean, no real secrets, absolute paths are only illustrative examples, no nested `.git`); any concern was reported to the user and resolved by their decision.
- [ ] README.md exists and is current: generated if missing (on user approval), or updated from the old README + new skill changes when the repo README's version is older than the skill's (on user approval).
- [ ] `git status`/`git diff` show only the expected changed files; `git fsck --full` reports no errors.
- [ ] `git push` exits 0 and advances `main`; optionally `git ls-remote` confirms the remote hash.
- [ ] Local clone `<workdir>` was removed after a verified push (or retained with reason if push failed).
- [ ] Any newly encountered, previously undocumented problem was solved and recorded into the skill (version bumped if behavior changed).
