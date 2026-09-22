---
name: skill-to-github
description: "Sync a WorkBuddy user-level skill directory to a GitHub repository using git over SSH. Use when the user asks to publish, update, back up, version-control, or reconcile a local skill with a GitHub repo. The user must explicitly name both the local skill and the GitHub project; if the repo does not exist, guide the user to create it on the GitHub website. Ensures README.md exists and is current (AI-generated on request), relies on git's built-in integrity checks rather than re-downloading files, and self-updates when it hits a problem it does not yet cover."
agent_created: true
version: 2.0.0
---

# skill-to-github

## Overview

Mirror a WorkBuddy user-level skill folder (typically `C:\Users\USER\.workbuddy\skills\<skill-name>\`) to a GitHub repository. The skill enforces explicit targeting (which local skill, which GitHub project), ensures the target repo exists (guiding the user to create it on the website when it is missing), keeps `README.md` current, verifies with git's own integrity mechanisms, and records any newly discovered fixes into itself.

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

Before each upload/update, check `README.md` (in the local skill; note what the repo currently has after cloning):

1. **Missing** (no `README.md` in the local skill) → ask the user whether to auto-generate it.
2. **Present but possibly stale** → judge whether it still reflects the current skill: if the skill has a `version:`, the README should reference it; spot-check that the described requirements and features match the current `SKILL.md`. When in doubt, treat it as not-current.
3. If missing or not-current, **ask the user**: "README.md 缺失/可能已过期，是否调用大模型总结该技能的要求与功能，自动生成 / 更新？"
   - On approval, read the full `SKILL.md` (and supporting files) and have the model summarize the skill's purpose, requirements, features, workflow, and usage into a clear `README.md`. Write it into the **local skill directory** so it becomes part of the skill, then continue.
   - If the user declines, leave the README as-is and proceed.

## Step 3 — Overlay the local skill into the clone

```
cp -r "<skill-dir>/." "<workdir>/"
```

Copy the directory *contents* (trailing `/.`) so repo-only files the local skill does not have are preserved, and the repo's `.git` is untouched.

## Step 4 — Verify with git's built-in checks (no re-download)

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

## Step 5 — Commit and push; confirm via the push result

```
git add -A
git commit -m "Sync <skill-name> (mirror local)"
git push origin main
git ls-remote origin
```

- Treat the **push output itself** as confirmation: it must exit 0 and show the ref advancing (e.g. `a0ac6e9..b767b10 main -> main`).
- Optionally `git ls-remote origin` and confirm `refs/heads/main` equals the commit just pushed. GitHub confirms the ref update on push, so this is authoritative — no extra local clone is required.

## Step 6 — Self-update when the skill lacked an answer

If this run hits a problem that the skill (this file or `references/setup.md`) does **not** already cover:

1. Solve the concrete problem first and confirm the sync succeeds.
2. Then update the skill so the next run does not rediscover it: add the symptom → cause → handling to the relevant section (workflow step, `Key Pitfalls`, or `setup.md`). Keep each addition minimal and directly tied to the problem.
3. Bump `version:` when the change alters behavior or workflow.
4. Do not add speculative fixes for problems that did not occur — only record what was actually encountered and verified.

## Key Pitfalls

- **Explicit targets first.** Never sync without the user naming the exact local skill and `owner/repo`; verify the local `SKILL.md` name before overlaying.
- **`core.autocrlf=false` is mandatory.** Without it, Git for Windows rewrites line endings and corrupts Python scripts.
- **Creating a repo is done on the GitHub website**, not with the SSH key: send the user to https://github.com/new with the exact name/owner/visibility, then confirm via `git ls-remote` and clone.
- **README is checked before every push**, and only auto-generated after the user agrees.
- **Verify via git (`status`/`diff`/`fsck`) and the push result**, not by re-downloading.
- **Preserve repo-only files:** always `cp -r src/. dst/`, never `cp -r src dst`.
- **`ssh-keygen` needs a Windows-native path** (`-f C:/Users/USER/.ssh/id_ed25519`); an msys `/c/...` path fails.
- **`winget install Git.Git` requires `--scope user`** to avoid an admin/UAC prompt that hangs non-interactive shells.

## Verification Checklist

- [ ] User explicitly named the local skill (path verified, `SKILL.md` name confirmed) and the exact `owner/repo`.
- [ ] Target repo exists (cloned), or was created by the user on https://github.com/new and confirmed via `git ls-remote`, then cloned.
- [ ] README.md exists and is current — or the user was asked and an AI-generated README was added on approval.
- [ ] `git status`/`git diff` show only the expected changed files; `git fsck --full` reports no errors.
- [ ] `git push` exits 0 and advances `main`; optionally `git ls-remote` confirms the remote hash.
- [ ] Any newly encountered, previously undocumented problem was solved and recorded into the skill (version bumped if behavior changed).
