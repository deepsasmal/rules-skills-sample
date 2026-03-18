---
name: push-to-github
description: Push code to GitHub using git CLI — initialize a repo, configure a remote, create branches, stage and commit changes, and open a pull request. Use when the user wants to push to GitHub, set up a remote, create a branch, commit changes, or start a pull request workflow.
---

# Push to GitHub

## Workflow overview

Follow these steps in order based on where the user is in the process:

1. [Initialize repo](#1-initialize-repo) — only if not already a git repo
2. [Connect remote](#2-connect-remote) — only if no remote is set
3. [Authenticate](#3-authenticate) — only if not yet authenticated
4. [Stage and commit](#4-stage-and-commit)
5. [Push branch](#5-push-branch)
6. [Open a pull request](#6-open-a-pull-request)

---

## 1. Initialize repo

```bash
git init
git add .
git commit -m "initial commit"
```

---

## 2. Connect remote

```bash
git remote add origin https://github.com/<user>/<repo>.git
git branch -M main
```

Verify with:

```bash
git remote -v
```

---

## 3. Authenticate

**HTTPS (recommended):** Use a [Personal Access Token (PAT)](https://github.com/settings/tokens) as the password when prompted, or store credentials with:

```bash
git config --global credential.helper store
```

**SSH:** Generate a key and add the public key to GitHub → Settings → SSH keys:

```bash
ssh-keygen -t ed25519 -C "your@email.com"
# Then add ~/.ssh/id_ed25519.pub to GitHub
```

Switch remote to SSH after setup:

```bash
git remote set-url origin git@github.com:<user>/<repo>.git
```

---

## 4. Stage and commit

```bash
git add .                        # stage all changes
git add <file>                   # stage a specific file
git status                       # review what is staged
git commit -m "your message"
```

Use snake_case in branch names and commit messages where variable-like identifiers appear.

---

## 5. Push branch

**Push main:**

```bash
git push -u origin main
```

**Create and push a new branch:**

```bash
git checkout -b feature/branch_name
git push -u origin feature/branch_name
```

**Force push** (use only when you know what you're doing):

```bash
git push --force-with-lease origin <branch>
```

---

## 6. Open a pull request

After pushing, GitHub prints a URL to open a PR directly. Copy and open it in your browser:

```
https://github.com/<user>/<repo>/compare/<branch>?expand=1
```

Or install the [GitHub CLI](https://cli.github.com/) and run:

```bash
gh pr create --title "Your title" --body "Description" --base main
```

---

## Quick reference

| Task | Command |
|---|---|
| Check status | `git status` |
| View log | `git log --oneline -10` |
| List branches | `git branch -a` |
| Switch branch | `git checkout <branch>` |
| Pull latest | `git pull origin main` |
| Undo last commit (keep changes) | `git reset --soft HEAD~1` |
