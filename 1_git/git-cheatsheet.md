# 🌿 Git Comprehensive Cheatsheet

> A complete reference for Git — distributed version control — covering theory, internals, CLI usage, workflows, and best practices.

---

## 📚 Table of Contents

1. [Core Concepts & Theory](#1-core-concepts--theory)
   - [What is Git?](#what-is-git)
   - [The Three States](#the-three-states)
   - [The Three Trees](#the-three-trees)
   - [How Git Stores Data](#how-git-stores-data)
   - [DAG — Commit History](#dag--commit-history)
   - [Content-Addressable Storage](#content-addressable-storage)
   - [Local vs Remote](#local-vs-remote)
   - [Bare vs Non-Bare](#bare-vs-non-bare)
2. [Configuration](#2-configuration)
   - [git config Levels](#git-config-levels)
   - [Essential Config](#essential-config)
   - [Useful Config](#useful-config)
   - [.gitconfig Example](#gitconfig-example)
   - [.gitignore](#gitignore)
   - [.gitattributes](#gitattributes)
3. [Starting a Repository](#3-starting-a-repository)
   - [git init](#git-init)
   - [git clone](#git-clone)
   - [Clone Protocols](#clone-protocols)
4. [Staging & Committing](#4-staging--committing)
   - [Status, Add, Commit](#status-add-commit)
   - [Interactive Staging](#interactive-staging)
   - [Amending Commits](#amending-commits)
   - [Conventional Commits](#conventional-commits)
   - [Moving & Removing Files](#moving--removing-files)
   - [git stash](#git-stash)
5. [Branching & Merging](#5-branching--merging)
   - [What is a Branch?](#what-is-a-branch)
   - [git branch](#git-branch)
   - [git switch & git checkout](#git-switch--git-checkout)
   - [Detached HEAD](#detached-head)
   - [git merge](#git-merge)
   - [Merge Strategies](#merge-strategies)
   - [Merge Conflicts](#merge-conflicts)
6. [Rebasing](#6-rebasing)
   - [What Rebase Does](#what-rebase-does)
   - [Rebase vs Merge](#rebase-vs-merge)
   - [Interactive Rebase](#interactive-rebase)
   - [git rebase --onto](#git-rebase---onto)
   - [Resolving Rebase Conflicts](#resolving-rebase-conflicts)
   - [git pull --rebase](#git-pull---rebase)
7. [Remote Repositories](#7-remote-repositories)
   - [git remote](#git-remote)
   - [fetch vs pull](#fetch-vs-pull)
   - [git push](#git-push)
   - [Remote Tracking Branches](#remote-tracking-branches)
   - [Force Push Safely](#force-push-safely)
   - [Multiple Remotes](#multiple-remotes)
8. [History & Inspection](#8-history--inspection)
   - [git log](#git-log)
   - [git show & git diff](#git-show--git-diff)
   - [git blame & Line History](#git-blame--line-history)
   - [git bisect](#git-bisect)
   - [git shortlog & git describe](#git-shortlog--git-describe)
   - [Commit References](#commit-references)
9. [Undoing & Rewriting History](#9-undoing--rewriting-history)
   - [git restore](#git-restore)
   - [git reset](#git-reset)
   - [git revert](#git-revert)
   - [git clean](#git-clean)
   - [git reflog](#git-reflog)
   - [Undo Strategy Guide](#undo-strategy-guide)
   - [fixup & autosquash](#fixup--autosquash)
10. [Tagging](#10-tagging)
    - [Lightweight vs Annotated](#lightweight-vs-annotated)
    - [git tag Commands](#git-tag-commands)
    - [Signing Tags](#signing-tags)
    - [SemVer Conventions](#semver-conventions)
11. [Advanced Features](#11-advanced-features)
    - [git cherry-pick](#git-cherry-pick)
    - [git worktree](#git-worktree)
    - [git submodule](#git-submodule)
    - [git sparse-checkout](#git-sparse-checkout)
    - [git archive](#git-archive)
    - [git bundle](#git-bundle)
    - [git filter-repo](#git-filter-repo)
    - [git notes](#git-notes)
12. [Internals & Plumbing](#12-internals--plumbing)
    - [.git/ Directory](#git-directory)
    - [Object Types](#object-types)
    - [Plumbing Commands](#plumbing-commands)
    - [Packfiles & GC](#packfiles--gc)
    - [Git Hooks](#git-hooks)
    - [Special Refs](#special-refs)
13. [Workflows & Collaboration](#13-workflows--collaboration)
    - [Gitflow](#gitflow)
    - [GitHub Flow](#github-flow)
    - [GitLab Flow](#gitlab-flow)
    - [Trunk-Based Development](#trunk-based-development)
    - [Fork & Pull Request](#fork--pull-request)
    - [Branch Naming Conventions](#branch-naming-conventions)
    - [Protecting Branches](#protecting-branches)
14. [Performance & Maintenance](#14-performance--maintenance)
    - [gc, prune, fsck](#gc-prune-fsck)
    - [Shallow & Partial Clones](#shallow--partial-clones)
    - [git maintenance](#git-maintenance)
    - [Git LFS](#git-lfs)
15. [Quick Reference Tables](#15-quick-reference-tables)

---

## 1. Core Concepts & Theory

### What is Git?

Git is a **distributed version control system (DVCS)** created by Linus Torvalds in 2005. Unlike centralised systems (SVN, CVS), every developer has a **complete copy** of the repository — history and all — on their local machine.

**Why use Git?**
- Every clone is a full backup of the entire project history
- Branching and merging are fast and cheap (just pointer manipulation)
- Work offline — commit, branch, and diff without network access
- Cryptographic integrity — every object is checksummed; history cannot be silently corrupted
- Industry standard — tooling, CI/CD, and hosting ecosystems built around it

---

### The Three States

Every file in a Git repo exists in one of three states:

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   Working          Staging Area         Repository      │
│   Directory          (Index)            (.git/)         │
│                                                         │
│  [modified]  ──git add──▶  [staged]  ──git commit──▶  [committed] │
│                                                         │
│  ◀──────────────── git restore ──────────────────────  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

| State | Description |
|-------|-------------|
| **Modified** | File has changed but not yet staged — Git knows about it but won't include it in the next commit |
| **Staged** | File is marked to go into the next commit snapshot — lives in the Index |
| **Committed** | Data is safely stored in the local repository as a snapshot |

---

### The Three Trees

Git manages three "trees" (file collections) simultaneously:

```
┌──────────────┐    git add     ┌──────────────┐   git commit  ┌──────────────┐
│              │ ─────────────▶ │              │ ────────────▶ │              │
│   Working    │                │    Index     │               │     HEAD     │
│    Tree      │ ◀───────────── │  (Staging)   │ ◀──────────── │  (Last       │
│  (disk files)│  git restore   │              │  git reset    │   Commit)    │
│              │                │              │   --soft      │              │
└──────────────┘                └──────────────┘               └──────────────┘
```

| Tree | Also Called | What It Contains |
|------|------------|-----------------|
| **HEAD** | Repository | Pointer to the last commit of the current branch |
| **Index** | Staging Area | Proposed next commit snapshot |
| **Working Tree** | Working Directory | Actual files you see and edit on disk |

---

### How Git Stores Data

> Git thinks of its data as a **series of snapshots**, not a set of diffs.

Every time you commit, Git stores a snapshot of ALL tracked files. Unchanged files are stored as references to the identical previous blob — not duplicated.

**Four object types stored in `.git/objects/`:**

```
BLOB   — raw file content (no filename, no metadata)
  │
TREE   — directory listing: maps filenames → blobs/subtrees
  │
COMMIT — snapshot: points to a root TREE + parent commit(s) + metadata
  │
TAG    — named pointer to a specific commit (annotated tag)
```

```
Commit C2
│  tree:    a1b2c3   ← root tree snapshot
│  parent:  f0e1d2   ← previous commit
│  author:  Alice
│  message: "Add login"
│
└──▶ Tree a1b2c3
      ├── blob 9d8e7f  README.md
      ├── blob 4c5b6a  main.py
      └── tree 1a2b3c  src/
            ├── blob 7f8e9d  auth.py
            └── blob 2c3d4e  utils.py
```

---

### DAG — Commit History

Git history is a **Directed Acyclic Graph (DAG)** — commits point to their parent(s), forming a chain that never loops.

```
A ──▶ B ──▶ C ──▶ D          ← main
              └──▶ E ──▶ F   ← feature
                         │
                         ▼
                    G (merge commit — two parents: D and F)
```

- **Normal commit**: one parent
- **Merge commit**: two (or more) parents
- **Root commit**: no parent (first commit in repo)
- Branches are just **named pointers** to a commit in this graph

---

### Content-Addressable Storage

Git identifies every object by the **SHA-1 hash** (or SHA-256 in newer repos) of its contents:

```bash
# SHA-1 is a 40-character hex string derived from the object's content
echo "hello" | git hash-object --stdin
# ce013625030ba8dba906f756967f9e9ca394464a

# Two identical files → identical hash → stored ONCE (deduplication is automatic)
# Any bit-flip in content → completely different hash (integrity guaranteed)
```

- The hash IS the name — Git can always verify an object hasn't been corrupted
- You only need the first 7–8 characters to uniquely reference an object in most repos
- Git 2.29+ supports SHA-256 repositories (`git init --object-format=sha256`)

> 💡 **Tip:** Git's SHA-1 is used for addressing, not security. It's being phased out in favour of SHA-256, but SHA-1 collision attacks on Git history remain extremely difficult in practice.

---

### Local vs Remote

```
Local Repository                Remote Repository
(your machine)                  (GitHub, GitLab, etc.)

.git/                           bare repo on server
├── objects/  ◀──git fetch──   ├── objects/
├── refs/                       ├── refs/
│   ├── heads/  (local)         │   └── heads/
│   └── remotes/origin/ ──────▶ └── (no working tree)
└── HEAD
```

| | Local | Remote |
|--|-------|--------|
| **Has working tree** | ✅ | ❌ (bare) |
| **Commit to directly** | ✅ | Via push |
| **Speed** | Instant (disk) | Network latency |
| **Offline work** | ✅ | ❌ |

---

### Bare vs Non-Bare

```bash
# Non-bare (normal) — has a working tree, used for development
git init myproject          # creates myproject/.git/

# Bare — no working tree, used as a shared/server repo
git init --bare myproject.git   # all git internals at the root level
```

- **Bare repos** are what GitHub/GitLab serve — you push/pull to them but never work in them directly
- Convention: bare repos end in `.git` (e.g. `project.git`)
- You cannot `git checkout` in a bare repo

---

## 2. Configuration

### git config Levels

```bash
# Three levels — lower overrides higher:
#
#   system  →  global  →  local  →  worktree
#  (weakest)                       (strongest)

# SYSTEM: applies to all users on the machine
git config --system core.editor vim
# File: /etc/gitconfig  (or  $(git --exec-path)/../etc/gitconfig)

# GLOBAL: applies to all repos for the current OS user
git config --global user.name "Alice Smith"
# File: ~/.gitconfig  (or  ~/.config/git/config)

# LOCAL: applies only to the current repository (default level)
git config --local user.email "alice@work.com"
# File: .git/config  (inside the repo)

# WORKTREE: applies to a specific git worktree (rarely used)
git config --worktree core.sparseCheckout true
# File: .git/config.worktree

# Show effective config and where each value comes from
git config --list --show-origin --show-scope
```

---

### Essential Config

```bash
# --- Identity (required for commits) ---
git config --global user.name "Alice Smith"
git config --global user.email "alice@example.com"

# --- Default branch name for new repos ---
git config --global init.defaultBranch main

# --- Editor for commit messages, interactive rebase, etc. ---
git config --global core.editor "code --wait"   # VS Code
git config --global core.editor "vim"
git config --global core.editor "nano"

# --- Default pull behaviour (rebase instead of merge) ---
git config --global pull.rebase true            # recommended
# git config --global pull.rebase false         # merge (old default)
# git config --global pull.ff only              # fail if not fast-forward

# --- Coloured output ---
git config --global color.ui auto

# --- Show branch info in status ---
git config --global status.showUntrackedFiles all
```

---

### Useful Config

```bash
# --- Line endings (CRITICAL for cross-platform teams) ---
# Windows: convert LF→CRLF on checkout, CRLF→LF on commit
git config --global core.autocrlf true
# macOS/Linux: convert CRLF→LF on commit, never on checkout
git config --global core.autocrlf input
# Disable conversion (use .gitattributes instead — recommended)
git config --global core.autocrlf false

# --- Diff tool ---
git config --global diff.tool vscode
git config --global difftool.vscode.cmd 'code --wait --diff $LOCAL $REMOTE'

# --- Merge tool ---
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'

# --- Credential helper (cache password for 1 hour) ---
git config --global credential.helper 'cache --timeout=3600'
# macOS keychain
git config --global credential.helper osxkeychain
# Windows credential manager
git config --global credential.helper manager

# --- Push behaviour ---
git config --global push.default current          # push current branch to same-name remote
git config --global push.autoSetupRemote true     # auto --set-upstream on first push (Git 2.37+)

# --- Reuse recorded resolutions (auto-replay conflict resolutions) ---
git config --global rerere.enabled true

# --- Pager for long output ---
git config --global core.pager "less -FRX"

# --- Safer force push by default ---
git config --global alias.pushf "push --force-with-lease"

# --- Prune remote tracking branches on fetch ---
git config --global fetch.prune true
```

---

### .gitconfig Example

```ini
# ~/.gitconfig — annotated full example

[user]
    name  = Alice Smith
    email = alice@example.com
    signingkey = ABC123DEF456      # GPG key ID for signed commits/tags

[core]
    editor     = code --wait       # VS Code as default editor
    autocrlf   = input             # normalise line endings on commit
    pager      = less -FRX         # pager for log/diff output
    excludesfile = ~/.gitignore_global  # global ignore patterns

[init]
    defaultBranch = main           # new repos start on "main"

[pull]
    rebase = true                  # pull = fetch + rebase (not merge)

[push]
    default         = current      # push to branch of same name
    autoSetupRemote = true         # auto-set upstream on first push

[fetch]
    prune = true                   # delete stale remote-tracking refs

[rebase]
    autosquash = true              # auto-apply fixup! commits in rebase -i
    autostash  = true              # stash before rebase, pop after

[merge]
    conflictstyle = zdiff3         # show base context in conflict markers (Git 2.35+)
    tool = vscode

[diff]
    algorithm = histogram          # better diff algorithm (fewer spurious hunks)
    tool = vscode
    colorMoved = default           # colour moved lines differently from add/del

[rerere]
    enabled = true                 # remember & replay conflict resolutions

[credential]
    helper = osxkeychain           # macOS keychain credential storage

[color]
    ui = auto                      # colour output when writing to terminal

[alias]
    # --- Shortcuts ---
    st   = status -sb              # compact status with branch info
    co   = checkout
    sw   = switch
    br   = branch
    ci   = commit
    cp   = cherry-pick

    # --- Log visualisation ---
    lg   = log --oneline --graph --decorate --all
    ll   = log --oneline --decorate -20       # last 20 commits

    # --- Undo helpers ---
    undo = reset HEAD~1 --mixed    # undo last commit, keep changes staged
    unstage = restore --staged     # remove from staging area

    # --- Safe force push ---
    pushf = push --force-with-lease

    # --- List aliases ---
    aliases = config --get-regexp alias

[gpg]
    program = gpg                  # GPG binary location

[commit]
    gpgsign = true                 # sign ALL commits by default
```

---

### .gitignore

```bash
# .gitignore — patterns of files Git should never track

# --- Syntax ---
# Blank lines and lines starting with # are comments
# A leading / anchors to the root of the repo
# A trailing / matches only directories
# A leading ! negates a pattern (un-ignore something)
# * matches anything except /
# ** matches across directories

# --- Common patterns ---
# Build artefacts
dist/
build/
*.o
*.pyc
__pycache__/
*.class

# Dependency directories
node_modules/
vendor/
.venv/

# IDE / editor files
.idea/
.vscode/
*.swp
*.swo
.DS_Store        # macOS metadata
Thumbs.db        # Windows thumbnail cache

# Secrets & environment
.env
.env.local
*.key
*.pem
secrets.yaml

# Logs
*.log
logs/

# Test coverage
coverage/
.coverage
htmlcov/

# Negation: ignore all *.log EXCEPT important.log
*.log
!important.log

# Anchor: only ignore /TODO at root, not src/TODO
/TODO

# Directory: ignore any directory named "build" anywhere
build/
```

```bash
# Global ignore — applies to ALL repos on your machine
# Set the path in git config:
git config --global core.excludesfile ~/.gitignore_global
# Then add patterns to ~/.gitignore_global
echo ".DS_Store" >> ~/.gitignore_global
echo "*.swp"     >> ~/.gitignore_global

# Per-repo local ignore (NOT committed — for personal patterns)
# Edit: .git/info/exclude
echo "my-notes.txt" >> .git/info/exclude

# Check why a file is being ignored
git check-ignore -v path/to/file

# Show all ignored files
git status --ignored

# Force-add an ignored file (bypass .gitignore)
git add -f ignored-file.txt
```

> 💡 **Tip:** Use [gitignore.io](https://gitignore.io) or GitHub's gitignore templates to generate language/framework-specific `.gitignore` files instantly.

---

### .gitattributes

```ini
# .gitattributes — per-path Git attribute rules (committed to the repo)
# Controls line endings, diff drivers, merge strategies, and more.

# --- Line ending normalisation (recommended over core.autocrlf) ---
# Normalise ALL text files: store as LF, checkout as native
* text=auto

# Force specific files to always use LF (shell scripts, Python, YAML)
*.sh    text eol=lf
*.py    text eol=lf
*.yaml  text eol=lf
*.yml   text eol=lf

# Force Windows line endings for .bat/.cmd
*.bat   text eol=crlf
*.cmd   text eol=crlf

# Mark as binary (no line-ending conversion, no diff)
*.png   binary
*.jpg   binary
*.pdf   binary
*.zip   binary
*.exe   binary

# --- Custom diff drivers ---
# Use word-level diff for Markdown
*.md    diff=markdown

# Use exif tool for images (show metadata in git diff)
*.png   diff=exif

# Configure the exif diff driver in .git/config:
# [diff "exif"]
#     textconv = exiftool

# --- Merge strategies ---
# Always keep "ours" version for auto-generated files (never conflict)
package-lock.json   merge=ours
yarn.lock           merge=ours
Gemfile.lock        merge=ours

# --- Export ignore: exclude from git archive ---
.gitignore          export-ignore
.gitattributes      export-ignore
tests/              export-ignore
docs/               export-ignore

# --- linguist (GitHub language detection) ---
vendor/**           linguist-vendored
docs/**             linguist-documentation
*.min.js            linguist-generated
```

> 💡 **Tip:** Commit `.gitattributes` early in a project. Changing line-ending rules later requires re-normalising all files with `git add --renormalize .` which creates a noisy commit.

---

## 3. Starting a Repository

### git init

```bash
# Initialise a new local repository in the current directory
git init
# Creates: .git/ directory with all Git internals

# Initialise in a new named directory
git init my-project
cd my-project

# Initialise with a specific default branch name
git init --initial-branch=main
# or set globally: git config --global init.defaultBranch main

# Create a bare repository (server/shared use — no working tree)
git init --bare myproject.git

# Re-initialise an existing repo (safe — doesn't overwrite data)
# Useful after changing config or templates
git init   # run inside an existing repo
```

---

### git clone

```bash
# Basic clone (clones default branch + full history)
git clone https://github.com/user/repo.git

# Clone into a specific directory name
git clone https://github.com/user/repo.git my-local-name

# --- Shallow clone (truncated history — faster for CI) ---
# Only fetch the last N commits (no full history)
git clone --depth 1 https://github.com/user/repo.git
# Fetch history since a specific date
git clone --shallow-since="2023-01-01" https://github.com/user/repo.git

# --- Single-branch clone (only one branch — saves bandwidth) ---
git clone --single-branch --branch main https://github.com/user/repo.git
git clone --single-branch --branch develop https://github.com/user/repo.git

# --- Sparse checkout (download repo but only checkout certain paths) ---
git clone --filter=blob:none --sparse https://github.com/user/repo.git
cd repo
git sparse-checkout set src/ docs/   # only materialise these directories

# --- Partial clone (omit large blobs from initial download) ---
# Download commits/trees but fetch blobs lazily on demand
git clone --filter=blob:none https://github.com/user/repo.git

# Clone a bare repo (for server setup or mirroring)
git clone --bare https://github.com/user/repo.git
git clone --mirror https://github.com/user/repo.git   # mirror: includes all refs
```

---

### Clone Protocols

```bash
# --- HTTPS (most common, works everywhere) ---
git clone https://github.com/user/repo.git
# Auth: username + password / token
# Pros: works through firewalls, no key setup
# Cons: must supply credentials (use credential helper)

# --- SSH (preferred for developers) ---
git clone git@github.com:user/repo.git
# Auth: SSH key pair
# Pros: no password prompts after key setup, more secure
# Cons: requires SSH key configured with the hosting service

# Set up SSH key:
ssh-keygen -t ed25519 -C "alice@example.com"   # generate key
cat ~/.ssh/id_ed25519.pub                        # copy this to GitHub/GitLab

# --- Git protocol (fast but unauthenticated — read-only public repos) ---
git clone git://github.com/user/repo.git
# Pros: fastest, no auth overhead
# Cons: no auth, no encryption — rarely used now

# --- Local file path ---
git clone /path/to/repo                          # hard-links where possible (fast)
git clone file:///path/to/repo                   # forces full copy (safer)

# Switch remote URL from HTTPS to SSH after cloning
git remote set-url origin git@github.com:user/repo.git
```

> 💡 **Tip:** Use SSH for repos you actively develop in. Use HTTPS for read-only or one-off clones. Store SSH keys in `~/.ssh/` and add them to your SSH agent with `ssh-add ~/.ssh/id_ed25519`.

---

## 4. Staging & Committing

### Status, Add, Commit

```bash
# --- git status ---
git status                  # full status output
git status -s               # short format (compact)
git status -sb              # short + branch info

# Short format legend:
# M  = modified in index (staged)
#  M = modified in working tree (unstaged)
# A  = new file added to index
# ?? = untracked file
# !! = ignored file

# --- git add ---
git add file.txt            # stage a specific file
git add src/                # stage all changes in a directory
git add .                   # stage all changes in current directory (recursive)
git add -A                  # stage ALL changes: new + modified + deleted (whole repo)
git add -u                  # stage modified + deleted only (no untracked)

# Stage only specific lines/hunks interactively (see next section)
git add -p                  # patch mode
git add -i                  # full interactive mode

# --- git commit ---
git commit -m "feat: add user authentication"   # inline message
git commit                                       # opens $EDITOR for message
git commit -a -m "fix: correct typo"            # stage tracked files + commit in one step
                                                 # (skips staging for already-tracked files)

# Commit with a multi-line message (body after blank line)
git commit -m "feat: add login endpoint

Implements JWT-based login with bcrypt password hashing.
Closes #42"

# Verbose commit: shows diff in editor when writing message
git commit -v
```

---

### Interactive Staging

`git add -p` lets you stage **individual hunks** (chunks) of a file — essential for making clean, focused commits when a file has multiple unrelated changes.

```bash
git add -p              # interactively stage hunks one at a time
git add -p file.txt     # same, but only for a specific file

# Hunk options:
# y  — yes, stage this hunk
# n  — no, skip this hunk
# s  — split hunk into smaller hunks (if possible)
# e  — manually edit the hunk in $EDITOR
# d  — skip this file entirely
# q  — quit (stop processing)
# ?  — show help
# /  — search for a hunk by regex

# Also works for other commands:
git checkout -p         # interactively discard hunks from working tree
git reset -p            # interactively unstage hunks
git stash -p            # interactively stash hunks
```

> 💡 **Tip:** `git add -p` is the secret to a clean Git history. Instead of one giant "WIP" commit, you can make 3 small, logical commits even when all the changes sit in the same file.

---

### Amending Commits

```bash
# Change the last commit message (before pushing)
git commit --amend -m "fix: correct the commit message"

# Add more changes to the last commit (forgot to include a file)
git add forgotten-file.txt
git commit --amend --no-edit          # --no-edit: keep existing message

# Amend last commit's author
git commit --amend --author="Bob <bob@example.com>" --no-edit

# Amend timestamp to now
git commit --amend --reset-author --no-edit
```

> ⚠️ **Warning:** `--amend` rewrites the commit (creates a new SHA). Never amend commits that have already been pushed to a shared branch — it will break history for everyone who has pulled that commit.

---

### Conventional Commits

A standardised commit message format that enables automated changelogs and semantic versioning.

```
<type>(<scope>): <short summary>
│       │              │
│       │              └── Imperative mood. No capital. No period at end.
│       └── Optional: affected component/module (e.g. auth, api, ui)
└── Required: feat | fix | docs | style | refactor | test | chore | perf | ci | build | revert

BLANK LINE (optional body below)
More detailed explanation. Wrap at 72 chars.
Explain WHY, not what (the diff shows what).

BLANK LINE (optional footers below)
Refs: #123
Co-authored-by: Bob <bob@example.com>
BREAKING CHANGE: <description>   ← triggers major version bump
```

```bash
# Examples:
git commit -m "feat(auth): add OAuth2 login with Google"
git commit -m "fix(api): return 404 when user not found"
git commit -m "docs: add API endpoint documentation"
git commit -m "refactor(db): extract query builder to separate module"
git commit -m "perf(search): add index on users.email column"
git commit -m "chore: update dependencies to latest versions"
git commit -m "test(auth): add unit tests for JWT validation"
git commit -m "ci: add GitHub Actions workflow for PR checks"

# Breaking change:
git commit -m "feat(api)!: rename /users endpoint to /accounts

BREAKING CHANGE: The /users endpoint has been renamed to /accounts.
Update all API clients accordingly."
```

> 💡 **Tip:** Tools like `commitizen`, `commitlint`, and `semantic-release` can automate validation and changelog generation from Conventional Commits. Add `commitlint` as a `commit-msg` hook.

---

### Moving & Removing Files

```bash
# --- git mv (rename/move a file) ---
git mv old-name.txt new-name.txt      # rename: stages the old deletion + new addition
git mv src/utils.py lib/utils.py      # move to different directory

# Under the hood, git mv is equivalent to:
mv old-name.txt new-name.txt
git rm old-name.txt
git add new-name.txt

# --- git rm (remove a file) ---
git rm file.txt                       # delete from disk AND stage the deletion
git rm --cached file.txt              # remove from staging/tracking ONLY (keep on disk)
                                      # useful for files you forgot to .gitignore

git rm -r build/                      # recursively remove a directory
git rm --force file.txt               # force removal even with local modifications

# Remove a file that was accidentally committed (then add to .gitignore):
git rm --cached secret.env
echo "secret.env" >> .gitignore
git commit -m "chore: stop tracking secret.env"
```

---

### git stash

Stash temporarily shelves changes so you can switch context without committing.

```bash
# --- Basic stash ---
git stash                              # stash tracked modified files
git stash push -m "WIP: login form"   # stash with a descriptive name
git stash -u                           # also stash untracked files
git stash -a                           # also stash ignored files (--all)

# --- Viewing stashes ---
git stash list                         # list all stashes
# Output: stash@{0}: WIP on main: abc123 Fix typo
#         stash@{1}: WIP on feature: def456 Add button

git stash show                         # show summary of latest stash
git stash show stash@{1}               # show specific stash
git stash show -p                      # show full diff of latest stash

# --- Applying stashes ---
git stash pop                          # apply latest stash + remove from stash list
git stash apply                        # apply latest stash + KEEP in stash list
git stash apply stash@{2}             # apply a specific stash

# --- Cleaning up stashes ---
git stash drop                         # delete the latest stash
git stash drop stash@{1}              # delete a specific stash
git stash clear                        # delete ALL stashes (irreversible!)

# --- Stash only specific files / hunks ---
git stash push -p                      # interactively choose hunks to stash
git stash push -- file1.txt file2.txt # stash specific files only

# --- Create a branch from a stash (useful if stash conflicts on apply) ---
git stash branch new-feature-branch stash@{0}
# Creates branch at the commit where stash was made, then applies stash
```

> 💡 **Tip:** Don't let stashes pile up. They're easy to forget and can conflict badly when applied later. Prefer short-lived feature branches over long-lived stashes.

---

## 5. Branching & Merging

### What is a Branch?

A branch in Git is simply a **lightweight movable pointer to a commit**. Creating a branch doesn't copy files — it just creates a 41-byte file in `.git/refs/heads/`.

```
main   ──▶  A ──▶ B ──▶ C
                         │
                       HEAD

# After: git switch -c feature
main   ──▶  A ──▶ B ──▶ C
                         │
feature ─────────────────┘
                         │
                       HEAD

# After committing on feature:
main   ──▶  A ──▶ B ──▶ C
feature ──▶ A ──▶ B ──▶ C ──▶ D ──▶ E
                                       │
                                     HEAD
```

**HEAD** is a special pointer that tracks which branch (or commit) you are currently on.

---

### git branch

```bash
# --- List branches ---
git branch                       # local branches (* marks current)
git branch -r                    # remote tracking branches
git branch -a                    # all branches (local + remote)
git branch -v                    # local with last commit info
git branch -vv                   # also shows upstream tracking branch
git branch --merged              # branches merged into current (safe to delete)
git branch --no-merged           # branches NOT yet merged (be careful deleting)

# --- Create branches ---
git branch feature/login         # create branch at current HEAD
git branch hotfix/bug-42 main    # create branch at a specific branch tip
git branch release/v1.0 v1.0    # create branch at a tag

# --- Delete branches ---
git branch -d feature/login      # delete (safe — refuses if unmerged)
git branch -D feature/login      # force delete (even if unmerged)
git branch -d -r origin/old-branch  # delete a remote-tracking ref locally

# --- Rename / copy ---
git branch -m old-name new-name  # rename current or named branch
git branch -c feature/login feature/login-backup  # copy a branch

# --- Set upstream tracking ---
git branch --set-upstream-to=origin/main main   # track remote branch
git branch -u origin/develop feature            # short form
git branch --unset-upstream                     # remove upstream tracking
```

---

### git switch & git checkout

```bash
# git switch (modern — Git 2.23+, preferred for branch operations)
git switch main                    # switch to main branch
git switch -c feature/new          # create AND switch to new branch
git switch -c hotfix origin/main   # create branch tracking a remote
git switch -                       # switch back to previous branch (like cd -)

# git checkout (older, multi-purpose command)
git checkout main                  # switch to main branch
git checkout -b feature/new        # create AND switch (-b = --branch)
git checkout -B feature/new        # create or reset branch to current HEAD
git checkout -                     # switch back to previous branch

# Switch to a specific commit (enters detached HEAD — see below)
git switch --detach abc1234
git checkout abc1234

# Create a branch from a specific starting point
git switch -c feature/from-v2 v2.0.0    # from a tag
git switch -c patch origin/release/1.x  # from a remote branch
```

---

### Detached HEAD

**Detached HEAD** occurs when HEAD points directly to a commit instead of a branch pointer.

```
Normal:     HEAD ──▶ main ──▶ C3
Detached:   HEAD ──────────▶ C2   (HEAD points to commit, not a branch)
```

```bash
# This puts you in detached HEAD:
git checkout abc1234       # check out a specific commit
git checkout v1.0.0        # check out a tag
git checkout origin/main   # check out a remote tracking branch

# You can explore, build, run tests — but commits made here are "dangling":
# Once you switch branches, these commits become unreachable (garbage collected eventually)

# --- How to recover from detached HEAD ---

# Option 1: Create a branch to keep your work
git switch -c my-new-branch   # save detached commits as a new branch

# Option 2: Don't need the commits — just go back to a branch
git switch main               # or: git checkout main

# Option 3: You made commits in detached HEAD and switched away — use reflog
git reflog                    # find the detached HEAD commits
git switch -c rescue-branch abc1234  # recreate branch at that commit
```

> ⚠️ **Warning:** Commits made in detached HEAD state are not lost immediately — but they'll be garbage collected by `git gc` after ~30 days if no branch or tag points to them. Save them with `git switch -c` immediately.

---

### git merge

```bash
# Merge a branch into the current branch
git switch main
git merge feature/login

# --- Fast-Forward Merge ---
# Possible when main hasn't diverged from feature's base
# Git simply moves main forward — no merge commit created
#
# Before:  main ──▶ A ──▶ B
#                          └──▶ C ──▶ D  ◀── feature
#
# After:   main ──────────────────────▶ D
#
git merge --ff-only feature/login   # only merge if FF is possible, else abort

# --- 3-Way Merge ---
# When main has new commits since feature branched off
# Git creates a new "merge commit" with two parents
#
# Before:  main ──▶ A ──▶ B ──▶ E
#                    └──▶ C ──▶ D  ◀── feature
#
# After:   main ──▶ A ──▶ B ──▶ E ──▶ M  (M = merge commit, parents: E and D)
#                    └──▶ C ──▶ D ─────┘

# Force a merge commit even when FF is possible (preserves branch topology)
git merge --no-ff feature/login -m "Merge feature/login into main"

# Squash all feature commits into one unstaged change (then commit manually)
git merge --squash feature/login
git commit -m "feat: add login (squashed from feature/login)"

# Abort an in-progress merge (after conflicts)
git merge --abort
```

---

### Merge Strategies

```bash
# Specify a merge strategy with -s
git merge -s recursive feature      # default for 2-branch merges (Git < 2.34)
git merge -s ort feature            # default in Git 2.34+ (faster recursive)
git merge -s ours feature           # always keep OUR version for ALL conflicts
git merge -s theirs feature         # always keep THEIR version (not built-in, use -X)
git merge -s octopus branch1 branch2 branch3  # merge 3+ branches at once
git merge -s subtree feature        # merge with subtree remapping

# Strategy options (passed with -X)
git merge -X ours feature           # on conflict, prefer our side
git merge -X theirs feature         # on conflict, prefer their side
git merge -X ignore-space-change    # ignore whitespace changes in conflicts
git merge -X patience feature       # use "patience" diff algorithm (better for refactored code)
```

---

### Merge Conflicts

```bash
# Anatomy of a conflict marker:
#
# <<<<<<< HEAD                   ← start of OUR version (current branch)
# const greeting = "Hello";
# =======                        ← separator
# const greeting = "Hi there";
# >>>>>>> feature/greeting       ← start of THEIR version (branch being merged)

# With zdiff3 conflict style (shows common ancestor too):
# <<<<<<< HEAD
# const greeting = "Hello";
# ||||||| merged common ancestors    ← original (base) version
# const greeting = "Howdy";
# =======
# const greeting = "Hi there";
# >>>>>>> feature/greeting

# --- Conflict resolution workflow ---

# 1. See which files have conflicts
git status                         # shows "both modified: file.txt"
git diff --diff-filter=U           # show only conflicted files

# 2. Resolve manually (edit files, remove conflict markers) OR use a tool
git mergetool                      # opens configured GUI merge tool

# 3. Stage resolved files
git add resolved-file.txt

# 4. Complete the merge
git commit                         # Git pre-fills merge commit message

# 5. Abort entirely if needed
git merge --abort                  # return to pre-merge state

# --- Useful helpers during conflict resolution ---
git checkout --ours   file.txt     # take our version entirely
git checkout --theirs file.txt     # take their version entirely
git diff --staged                  # review what you've resolved so far
```

> 💡 **Tip:** Configure `merge.conflictstyle = zdiff3` globally. Seeing the common ancestor ("base") alongside both sides makes it far easier to understand what each side changed and why there's a conflict.

---

## 6. Rebasing

### What Rebase Does

Rebase **replays** commits from one branch onto a new base commit, producing new commits with different SHAs (same changes, new identities).

```
Before rebase:
  main:    A ──▶ B ──▶ C
  feature: A ──▶ B ──▶ D ──▶ E

git switch feature
git rebase main

After rebase:
  main:    A ──▶ B ──▶ C
  feature: A ──▶ B ──▶ C ──▶ D' ──▶ E'
                              │
                (D' and E' are NEW commits — same changes as D,E but rebased onto C)
```

---

### Rebase vs Merge

```
Merge: preserves full history, adds a merge commit
─────────────────────────────────────────────────
main: A──B──C──────────M   (M = merge commit)
              \       /
feature:       D──E──F

Rebase: rewrites history for a linear timeline
──────────────────────────────────────────────
main: A──B──C──D'──E'──F'   (clean linear history)
(feature branch is deleted after merging — was FF merged into main)
```

**The Golden Rule of Rebase:**

> ⚠️ **Warning:** **Never rebase commits that have been pushed to a shared/public branch.** Rebase rewrites commit SHAs. If others have based work on those commits, their history diverges and everyone gets a bad time. Rebase is safe on local branches or personal feature branches that only you are working on.

```bash
# Safe: rebase your local feature branch onto updated main
git switch feature/my-work
git rebase main               # replay feature commits on top of latest main

# Dangerous: rebasing a shared branch (DON'T DO THIS)
git switch main
git rebase origin/develop     # if others have pulled main — DANGER
```

---

### Interactive Rebase

`git rebase -i` is one of Git's most powerful features — it lets you rewrite, reorder, squash, and edit commits before sharing them.

```bash
# Open interactive rebase for last N commits
git rebase -i HEAD~3          # edit last 3 commits
git rebase -i HEAD~5          # edit last 5 commits
git rebase -i abc1234         # rebase all commits after abc1234

# This opens $EDITOR with a list of commits (oldest first):
#
# pick a1b2c3 feat: add user model
# pick d4e5f6 fix: correct email validation
# pick 7890ab docs: add API documentation
#
# Commands:
# pick   (p) — use commit as-is
# reword (r) — use commit but edit the message
# edit   (e) — use commit but pause to amend (add/remove files)
# squash (s) — meld into previous commit (combine messages)
# fixup  (f) — meld into previous commit (discard this message)
# drop   (d) — remove this commit entirely
# exec   (x) — run a shell command after this commit
# break      — pause here (continue with git rebase --continue)
# label  (l) — label current HEAD with a name
# reset  (t) — reset HEAD to a label

# --- Example operations ---

# Squash last 3 commits into one:
# pick a1b2c3 First commit
# squash d4e5f6 Second commit   ← change "pick" to "squash"
# squash 7890ab Third commit    ← change "pick" to "squash"

# Reorder commits (just reorder the lines)
# pick 7890ab docs: add API documentation   ← moved up
# pick a1b2c3 feat: add user model
# pick d4e5f6 fix: correct email validation

# Drop a commit (delete the line or use "drop")
# drop d4e5f6 fix: correct email validation  ← this commit disappears

# After making changes in the editor, save and close
# Git replays commits according to your instructions

# Continue after stopping for edit/conflict
git rebase --continue

# Abort and return to original state
git rebase --abort

# Skip a commit that causes conflicts (loses that commit's changes)
git rebase --skip
```

---

### git rebase --onto

`--onto` is for **advanced branch surgery** — rebasing a range of commits onto an arbitrary new base.

```bash
# Syntax: git rebase --onto <newbase> <upstream> [<branch>]

# Scenario: feature-2 was branched from feature-1, not main
# You want to rebase feature-2 directly onto main
#
# Before:
#   main:      A──B──C
#   feature-1: A──B──C──D──E
#   feature-2: A──B──C──D──E──F──G
#
git rebase --onto main feature-1 feature-2
#
# After:
#   main:      A──B──C
#   feature-1: A──B──C──D──E
#   feature-2: A──B──C──F'──G'   ← F and G rebased directly onto main

# Remove a range of commits from a branch
# Remove commits D and E from main (rebase everything after E onto C)
git rebase --onto C E main
# Before: A──B──C──D──E──F──G  (main)
# After:  A──B──C──F'──G'
```

---

### Resolving Rebase Conflicts

```bash
# Rebase pauses on each conflicting commit (unlike merge which pauses once)
# Conflict markers are the same as merge conflicts

# 1. Edit conflicting files to resolve
# 2. Stage the resolved files
git add resolved-file.txt

# 3. Continue to the next commit
git rebase --continue

# 4. Repeat for each conflicting commit

# At any point, abort the whole rebase
git rebase --abort    # returns to pre-rebase state

# Skip a problematic commit (CAUTION: loses that commit's changes)
git rebase --skip
```

> 💡 **Tip:** Enable `rerere` (`git config --global rerere.enabled true`). Git will **r**emember and **re**use your conflict **re**solutions — so if the same conflict recurs (e.g. after rebasing multiple times), it resolves automatically.

---

### git pull --rebase

```bash
# Instead of: fetch + merge (creates merge commits)
git pull                    # default: fetch + merge

# Use: fetch + rebase (linear history, no merge commits)
git pull --rebase           # rebase local commits on top of fetched changes
git pull --rebase=interactive  # interactive rebase during pull

# Set as default globally (recommended for feature branch workflows)
git config --global pull.rebase true

# Set for current branch only
git config branch.main.rebase true

# Stash automatically before pull --rebase and re-apply after
git config --global rebase.autostash true
# Then: git pull --rebase works even with uncommitted local changes
```

---

## 7. Remote Repositories

### git remote

```bash
# --- List remotes ---
git remote                          # list remote names
git remote -v                       # list with fetch/push URLs
git remote show origin              # detailed info: branches, tracking, stale refs

# --- Add / remove / rename ---
git remote add origin https://github.com/user/repo.git   # add remote named "origin"
git remote add upstream https://github.com/original/repo.git  # add "upstream" (fork pattern)
git remote remove upstream          # remove a remote
git remote rename origin old-origin # rename a remote

# --- Change URLs ---
git remote set-url origin git@github.com:user/repo.git   # switch HTTPS → SSH
git remote set-url --add origin https://mirror.example.com/repo.git  # add a second push URL
git remote set-url --delete origin https://old.url/repo  # remove a specific URL

# --- Prune stale remote-tracking refs ---
git remote prune origin             # delete local refs for remote branches that no longer exist
git fetch --prune                   # fetch + prune in one command
```

---

### fetch vs pull

```bash
# git fetch — download changes, do NOT touch your working tree or branches
git fetch origin                    # fetch all branches from origin
git fetch origin main               # fetch only main branch
git fetch --all                     # fetch from ALL configured remotes
git fetch --prune                   # also prune deleted remote branches
git fetch --tags                    # fetch all tags

# After fetch, you can inspect before integrating:
git log HEAD..origin/main           # see what's new on remote main
git diff HEAD origin/main           # see changes coming in
git merge origin/main               # integrate manually when ready

# git pull — fetch + integrate (merge or rebase) in one step
git pull                            # fetch + merge (or rebase if pull.rebase=true)
git pull origin main                # pull specific remote + branch
git pull --rebase                   # fetch + rebase
git pull --no-rebase                # fetch + merge (override pull.rebase)
git pull --ff-only                  # only if fast-forward is possible

# Key difference:
# fetch = safe, read-only, never breaks your work
# pull  = fetch + automatic integration (can cause conflicts or unexpected merges)
```

> 💡 **Tip:** Many experienced Git users prefer `git fetch` + manual review + `git merge`/`git rebase` over `git pull`. It gives you control over when and how you integrate remote changes.

---

### git push

```bash
# Push current branch to its tracked remote branch
git push

# Push to a specific remote and branch
git push origin main
git push origin feature/login       # push local feature branch to remote

# Push and set upstream tracking in one step (first push of a new branch)
git push -u origin feature/login    # -u = --set-upstream
# After this, bare "git push" works for this branch

# --- Deleting remote branches ---
git push origin --delete old-feature    # delete remote branch
git push origin :old-feature           # older syntax (colon prefix = delete)

# --- Pushing tags ---
git push origin v1.0.0              # push a specific tag
git push origin --tags              # push ALL local tags to remote
git push origin --follow-tags       # push tags reachable from pushed commits

# --- Deleting remote tags ---
git push origin --delete v1.0.0-beta
git push origin :refs/tags/v1.0.0-beta

# --- Force push (DANGEROUS on shared branches) ---
git push --force                    # overwrite remote history (dangerous!)
git push --force-with-lease         # safer: fails if remote has new commits you don't have
git push --force-with-lease=main:abc1234  # specify expected remote ref
```

---

### Remote Tracking Branches

```bash
# Remote-tracking branches are LOCAL references to the state of remote branches
# They live in .git/refs/remotes/ and are named origin/branchname

# They are READ-ONLY — you cannot commit to them directly
# They are updated automatically by git fetch/pull/push

git branch -r                       # list all remote-tracking branches
# output: origin/main, origin/develop, origin/HEAD -> origin/main

# See where remote-tracking branches diverge from local
git log main..origin/main           # commits on remote not in local
git log origin/main..main           # commits in local not on remote

# Create a local branch that tracks a remote-tracking branch
git switch -c feature origin/feature         # local "feature" tracks "origin/feature"
git checkout --track origin/feature          # equivalent older syntax

# Check upstream tracking for all local branches
git branch -vv
# output: * main abc1234 [origin/main] Last commit message
#           dev  def567 [origin/dev: ahead 2, behind 1] Another commit
```

---

### Force Push Safely

```bash
# --force-with-lease is almost always better than --force
# It fails if the remote has been updated since your last fetch
# (protects against accidentally overwriting someone else's push)

git push --force-with-lease origin feature/my-branch

# Make it the default with an alias:
git config --global alias.pushf "push --force-with-lease"
git pushf origin feature/my-branch

# --force-if-includes (Git 2.30+) — even safer
# Also checks that the remote-tracking ref is in your reflog (you've seen those changes)
git push --force-with-lease --force-if-includes
```

> ⚠️ **Warning:** Never `git push --force` to `main`, `master`, or any shared branch. You will overwrite other people's commits and cause serious headaches. Use `--force-with-lease` on feature branches only, and only when rebasing your own commits.

---

### Multiple Remotes

```bash
# Fork & upstream pattern (common for open source contribution)

# 1. You have: origin (your fork) + upstream (original repo)
git remote add upstream https://github.com/original/project.git
git remote -v
# origin    https://github.com/you/project.git (fetch)
# origin    https://github.com/you/project.git (push)
# upstream  https://github.com/original/project.git (fetch)
# upstream  https://github.com/original/project.git (push)

# 2. Keep your fork's main up to date with upstream
git fetch upstream                          # download upstream changes
git switch main
git merge upstream/main                     # or: git rebase upstream/main
git push origin main                        # push updated main to your fork

# 3. Work on a feature branch
git switch -c feature/my-pr
# ... make commits ...
git push -u origin feature/my-pr            # push to YOUR fork

# 4. Open a PR from origin/feature/my-pr → upstream/main

# Push to multiple remotes at once (mirror pattern)
git remote set-url --add origin https://gitlab.com/you/project.git
git push origin     # now pushes to both GitHub and GitLab
```

---

## 8. History & Inspection

### git log

```bash
# --- Basic log ---
git log                             # full log (author, date, message, hash)
git log --oneline                   # condensed: hash + subject line
git log --oneline --graph           # ASCII art branch graph
git log --oneline --graph --all     # all branches + tags in graph
git log --oneline --graph --all --decorate  # with ref labels

# Useful alias (add to .gitconfig):
# lg = log --oneline --graph --all --decorate

# --- Filtering ---
git log -5                          # last 5 commits only
git log --since="2 weeks ago"       # since a relative time
git log --since="2024-01-01" --until="2024-06-30"  # date range
git log --author="Alice"            # by author name (partial match)
git log --author="alice@example.com"  # by email
git log --grep="fix:"               # commits whose message matches regex
git log --grep="fix:" --grep="feat:" --all-match  # must match BOTH

# --- Diff filtering ---
git log -S "getUserById"            # commits that ADD or REMOVE this string (pickaxe)
git log -G "getUserBy.*"            # commits where diff matches regex
git log --diff-filter=M -- api.js   # only commits that MODIFIED api.js
git log --follow -- old-name.txt    # follow renames (track a file through rename)

# --- Formatting ---
git log --format="%h %an %s"        # custom: hash, author name, subject
git log --format="%H"               # full SHAs only (useful for scripts)
git log --stat                      # files changed per commit + line counts
git log --patch                     # full diff for each commit (verbose)
git log --patch -3                  # full diff for last 3 commits
git log --word-diff                 # word-level diff instead of line-level
git log --name-only                 # only filenames changed
git log --name-status               # filenames + status (A/M/D)

# --- Comparing ranges ---
git log main..feature               # commits in feature not in main (what's on feature?)
git log feature..main               # commits in main not in feature (what would I merge?)
git log main...feature              # symmetric diff (commits unique to either side)

# --- Specific file history ---
git log -- path/to/file.py          # all commits that touched this file
git log --follow -- old/path.py     # follow renames
git log -p -- config.yaml           # full diff history for one file
```

---

### git show & git diff

```bash
# --- git show ---
git show                            # show last commit (diff + metadata)
git show abc1234                    # show a specific commit
git show HEAD~2                     # show commit 2 before HEAD
git show v1.0.0                     # show a tag
git show HEAD:src/main.py           # show file contents at a specific commit
git show HEAD~1:package.json        # see what package.json looked like before

# --- git diff ---
git diff                            # working tree vs staging area (unstaged changes)
git diff --staged                   # staging area vs last commit (what will be committed)
git diff HEAD                       # working tree vs last commit (all changes)

git diff main feature               # between two branches
git diff abc1234 def5678            # between two commits
git diff HEAD~3 HEAD                # last 3 commits worth of changes
git diff HEAD~1                     # changes since last commit

git diff -- file.py                 # diff only a specific file
git diff --stat main feature        # summary: files changed, lines +/-
git diff --word-diff                # word-level changes (good for prose)
git diff --ignore-space-change      # ignore whitespace changes
git diff -w                         # ignore ALL whitespace

# View diff in external tool
git difftool                        # opens configured difftool
git difftool main feature -- app.py # diff specific file between branches
```

---

### git blame & Line History

```bash
# Show who last modified each line of a file
git blame file.py
git blame -L 10,25 file.py          # only lines 10-25
git blame -L 10,+15 file.py         # lines 10 to 10+15
git blame --since="6 months ago" file.py  # ignore commits older than 6 months
git blame -w file.py                # ignore whitespace changes
git blame -C file.py                # detect code moved from other files (copy detection)
git blame -M file.py                # detect code moved within same file

# git log -L — show the history of a function or line range
git log -L 10,25:file.py            # history of lines 10-25 in file.py
git log -L :functionName:file.py    # history of a function by name (language-aware)
git log -L '/def get_user/,/^def/:api.py'  # regex-based range
```

> 💡 **Tip:** Use `git blame -w -C -C -C` to ignore whitespace changes AND aggressively detect code copied from other files. This shows the commit that ORIGINALLY wrote the code, not just the commit that moved it.

---

### git bisect

`git bisect` performs a **binary search** through commit history to find the first commit that introduced a bug.

```bash
# --- Manual bisect ---
git bisect start                    # start bisect session
git bisect bad                      # mark current commit as BAD (has the bug)
git bisect good v1.2.0              # mark last known GOOD commit
# Git now checks out a commit halfway between good and bad

# Test the commit — does it have the bug?
git bisect bad                      # this commit has the bug
git bisect good                     # this commit is fine
# Repeat — Git checks out the midpoint each time
# After ~log₂(N) steps, Git identifies the FIRST bad commit

git bisect reset                    # end bisect session, return to original HEAD

# --- Automated bisect (with a test script) ---
git bisect start
git bisect bad                      # current is bad
git bisect good v1.2.0              # last known good

# Run bisect automatically with a test script
# Script must exit 0 for "good", non-zero for "bad"
git bisect run ./test-script.sh

# Or use a single command:
git bisect run pytest tests/test_auth.py -x  # pytest exits non-zero on failure

# After bisect, Git prints: "abc1234 is the first bad commit"
git bisect reset
git show abc1234                    # inspect the offending commit

# --- Useful bisect commands ---
git bisect log                      # show bisect history so far
git bisect replay bisect.log        # replay a saved bisect session
git bisect skip                     # skip current commit (e.g. broken build)
git bisect skip abc1234..def5678    # skip a range of commits
```

> 💡 **Tip:** With 1,000 commits between good and bad, `git bisect` finds the culprit in just **10 steps** (log₂ 1000 ≈ 10). Always use `git bisect run` with an automated test when possible — it's much faster than manual testing.

---

### git shortlog & git describe

```bash
# git shortlog — summarise commits by author (useful for changelogs/releases)
git shortlog                        # group commits by author
git shortlog -sn                    # just author name + commit count, sorted by count
git shortlog -sn --no-merges        # exclude merge commits
git shortlog HEAD~20..HEAD          # just the last 20 commits
git shortlog v1.0.0..HEAD           # commits since v1.0.0 tag (release notes)

# git describe — human-readable name for a commit based on tags
git describe HEAD                   # e.g: v1.2.0-14-gabc1234
                                    # (tag v1.2.0 + 14 commits after + short SHA)
git describe --tags HEAD            # use lightweight tags too
git describe --abbrev=8             # use 8-char SHA abbreviation
git describe --exact-match HEAD     # error if HEAD is not exactly on a tag
git describe --always HEAD          # fallback to short SHA if no tag found
git describe --dirty                # append "-dirty" if working tree has changes
```

---

### Commit References

```bash
# --- Relative references ---
HEAD                    # current commit (tip of current branch)
HEAD~1  or  HEAD~       # 1 commit before HEAD (first parent)
HEAD~2                  # 2 commits before HEAD
HEAD~N                  # N commits before HEAD

HEAD^                   # same as HEAD~1 (first parent)
HEAD^2                  # second parent of HEAD (only on merge commits!)
HEAD^1^2                # second parent of HEAD's first parent

# --- Tilde vs Caret ---
# ~ always follows FIRST parent (linear history)
# ^ selects among multiple parents (for merge commits)
#
#       C
#      / \
#     B   |    HEAD = C
#     |   |    HEAD~1 = B (first parent of C)
#     A   D    HEAD^2 = D (second parent of C — the merged branch)
#         |    HEAD^2~1 = D's parent

# --- Named branch/tag references ---
main                    # tip of main branch
origin/main             # remote-tracking branch
v1.0.0                  # tag

# --- Reflog references ---
HEAD@{1}                # where HEAD was 1 move ago (before last checkout/reset etc.)
HEAD@{5}                # where HEAD was 5 moves ago
main@{yesterday}        # where main was yesterday
main@{2 weeks ago}      # where main was 2 weeks ago
HEAD@{0}                # same as HEAD (current position)

# --- Range references ---
main..feature           # commits reachable from feature but NOT from main
main...feature          # symmetric difference (commits unique to either side)
^main feature           # same as main..feature (exclude main)

# --- Other special refs ---
ORIG_HEAD               # previous HEAD before a merge/rebase/reset
MERGE_HEAD              # the commit being merged (during a merge)
CHERRY_PICK_HEAD        # the commit being cherry-picked
FETCH_HEAD              # last fetched commit
```

---

## 9. Undoing & Rewriting History

### git restore

```bash
# git restore — discard changes (Git 2.23+, replaces checkout for this purpose)

# Discard unstaged changes in a file (restore from staging area)
git restore file.txt                # working tree matches staged version
git restore .                       # discard ALL unstaged changes

# Unstage a file (restore staging area from last commit)
git restore --staged file.txt       # move file from staged → modified
git restore --staged .              # unstage everything

# Both: unstage AND discard changes (back to last commit)
git restore --staged --worktree file.txt

# Restore a file from a specific commit
git restore --source=HEAD~2 file.txt     # from 2 commits ago
git restore --source=abc1234 file.txt    # from a specific commit
git restore --source=main -- config.yaml # from another branch
```

---

### git reset

`git reset` moves the branch pointer (HEAD) to a different commit, optionally affecting the Index and Working Tree.

```
                    ┌──────────────────────────────────────────────────┐
                    │           git reset modes                        │
                    ├──────────────┬───────────────┬───────────────────┤
                    │   --soft     │   --mixed     │     --hard        │
                    │              │  (default)    │                   │
  ┌───────────────┐ │              │               │                   │
  │ HEAD / Branch │ │   MOVED      │   MOVED       │   MOVED           │
  └───────────────┘ │              │               │                   │
  ┌───────────────┐ │              │               │                   │
  │  Index        │ │   UNCHANGED  │   RESET       │   RESET           │
  │ (Staging)     │ │              │               │                   │
  └───────────────┘ │              │               │                   │
  ┌───────────────┐ │              │               │                   │
  │ Working Tree  │ │   UNCHANGED  │   UNCHANGED   │   RESET           │
  │   (files)     │ │              │               │                   │
  └───────────────┘ └──────────────┴───────────────┴───────────────────┘
```

```bash
# --- git reset --soft (safest — only moves HEAD) ---
git reset --soft HEAD~1             # undo last commit, keep changes STAGED
git reset --soft HEAD~3             # undo last 3 commits, all changes staged
# Use when: you want to recommit with a different message or combine commits

# --- git reset --mixed (default — moves HEAD + resets Index) ---
git reset HEAD~1                    # undo last commit, changes back to MODIFIED (unstaged)
git reset HEAD~3                    # undo 3 commits, all changes unstaged in working tree
git reset abc1234                   # reset to a specific commit
# Use when: you want to recommit the changes differently, choosing what to stage

# --- git reset --hard (nuclear — moves HEAD + resets Index + Working Tree) ---
git reset --hard HEAD~1             # undo last commit AND discard all changes
git reset --hard origin/main        # reset local branch to match remote exactly
git reset --hard abc1234            # jump to a specific commit, discard everything after
```

> ⚠️ **Warning:** `git reset --hard` permanently discards uncommitted changes from your working tree. There is no undo (unless you have the SHA in `git reflog`). Never use on a shared branch.

```bash
# --- Resetting specific files (doesn't move HEAD) ---
git reset HEAD file.txt             # unstage file.txt (HEAD~N works too)
git reset abc1234 -- file.txt       # restore file.txt's index state from that commit
```

---

### git revert

`git revert` creates a **new commit** that undoes a previous commit — it never rewrites history, making it safe for shared branches.

```bash
# Revert the last commit (creates a new "revert" commit)
git revert HEAD
git revert HEAD --no-edit           # don't open editor, use default message

# Revert a specific commit by SHA
git revert abc1234

# Revert a range of commits (creates one revert commit per reverted commit)
git revert HEAD~3..HEAD             # revert last 3 commits individually
git revert abc1234..def5678         # revert a range

# Revert multiple commits as one single new commit (--no-commit + commit manually)
git revert --no-commit HEAD~3..HEAD  # stage all reversals without committing
git commit -m "revert: undo last 3 commits"

# Revert a merge commit (requires specifying which parent is "mainline")
git revert -m 1 abc1234             # -m 1 = keep parent 1's side (the branch you merged into)
git revert -m 2 abc1234             # keep parent 2's side (the branch that was merged)
```

> 💡 **Tip:** On shared/protected branches (`main`, `production`), always use `git revert`. On local or personal feature branches, `git reset` is fine. The question to ask: "Have others already pulled these commits?"

---

### git clean

```bash
# Remove UNTRACKED files from working tree

git clean -n                        # dry run: show what WOULD be deleted
git clean -f                        # force delete untracked files
git clean -fd                       # delete untracked files AND directories
git clean -fx                       # delete untracked + ignored files
git clean -fdx                      # delete everything not tracked or committed

git clean -i                        # interactive mode (select what to delete)
```

> ⚠️ **Warning:** `git clean -f` is irreversible — deleted files are NOT sent to trash. Always run `git clean -n` (dry run) first to see what will be deleted.

---

### git reflog

The reflog is a **local log of everywhere HEAD has pointed** — your safety net for recovering "lost" commits.

```bash
# Show reflog for HEAD (every time HEAD moved)
git reflog
git reflog show HEAD                # same as above
# output:
# abc1234 HEAD@{0}: commit: feat: add login
# def5678 HEAD@{1}: checkout: moving from feature to main
# 789abc  HEAD@{2}: reset: moving to HEAD~1
# ...

# Show reflog for a specific branch
git reflog show main
git reflog show feature/login

# --- Recovering lost commits ---

# Scenario: You ran "git reset --hard" and lost commits
git reflog                          # find the SHA of the lost commit
git reset --hard abc1234            # restore to that commit
# or: create a branch at the lost commit
git switch -c rescue-branch abc1234

# Reflog entries expire after 90 days by default (configurable)
git config gc.reflogExpire 180.days          # extend to 180 days
git config gc.reflogExpireUnreachable 60.days
```

> 💡 **Tip:** `git reflog` is your ultimate undo. Almost any mistake in Git can be recovered using the reflog within the expiry window. The only things you truly cannot recover are uncommitted changes lost to `git reset --hard` or `git clean -f`.

---

### Undo Strategy Guide

```
What happened?                           What to use?
──────────────────────────────────────────────────────────────────────
Uncommitted file changes (unstaged)      git restore <file>
Staged changes (not yet committed)       git restore --staged <file>
Last commit — keep changes staged        git reset --soft HEAD~1
Last commit — keep changes unstaged      git reset HEAD~1  (--mixed)
Last commit — discard all changes        git reset --hard HEAD~1  ⚠️
Last commit — already pushed to shared   git revert HEAD
Multiple commits — local only            git rebase -i HEAD~N  (squash/drop)
Multiple commits — already pushed        git revert <range>
Accidentally deleted branch              git switch -c name <SHA from reflog>
Lost commits after hard reset            git reflog → git reset --hard <SHA>
Wrong commit message (local)             git commit --amend
Wrong commit message (shared)            git revert (creates a new fix commit)
```

---

### fixup & autosquash

```bash
# Create a fixup commit targeting a previous commit
git commit --fixup abc1234          # commit message becomes "fixup! <original message>"
git commit --squash abc1234         # like fixup but prompts to edit combined message

# Then use autosquash to automatically organise fixups in rebase -i
git rebase -i --autosquash HEAD~10  # fixup! commits are moved and set to "fixup" automatically

# Or set globally (always autosquash in interactive rebase):
git config --global rebase.autosquash true
# Then just: git rebase -i HEAD~10
```

---

## 10. Tagging

### Lightweight vs Annotated

```
Lightweight tag:  A simple pointer to a commit (like a branch that doesn't move)
                  Stored as: .git/refs/tags/v1.0 → abc1234

Annotated tag:    A full Git object with its own SHA, message, author, date, GPG sig
                  Stored as: .git/refs/tags/v1.0 → tag object → commit abc1234
```

**Use annotated tags for releases.** Lightweight tags are for temporary/personal use.

---

### git tag Commands

```bash
# --- List tags ---
git tag                             # list all tags alphabetically
git tag -l "v1.*"                   # list tags matching a pattern
git tag -l --sort=-version:refname  # sort by semantic version (descending)
git show v1.0.0                     # show tag details + commit

# --- Create tags ---
# Lightweight tag (just a name)
git tag v1.0.0                      # tag current commit
git tag v1.0.0 abc1234              # tag a specific commit

# Annotated tag (full object with message — use for releases)
git tag -a v1.0.0 -m "Release version 1.0.0"
git tag -a v1.0.0 abc1234 -m "Release 1.0.0"  # tag a specific commit

# --- Delete tags ---
git tag -d v1.0.0-beta              # delete local tag
git push origin --delete v1.0.0-beta  # delete remote tag

# --- Push tags ---
git push origin v1.0.0             # push a specific tag
git push origin --tags             # push ALL tags
git push origin --follow-tags      # push only annotated tags reachable from pushed commits

# --- Checkout at a tag (enters detached HEAD) ---
git checkout v1.0.0                 # check out code at this tag
git switch -c hotfix/1.0.x v1.0.0  # create a branch from a tag
```

---

### Signing Tags

```bash
# Sign a tag with your GPG key
git tag -s v1.0.0 -m "Release 1.0.0"      # -s = sign with default GPG key
git tag -u ABC123KEY v1.0.0 -m "Release"  # -u = specify key ID

# Verify a signed tag
git tag -v v1.0.0

# Set default signing key
git config --global user.signingkey ABC123DEF456

# List GPG keys
gpg --list-secret-keys --keyid-format LONG
```

---

### SemVer Conventions

```
MAJOR.MINOR.PATCH[-PRERELEASE][+BUILD]

v1.2.3
│ │ └─ PATCH: backwards-compatible bug fixes
│ └─── MINOR: backwards-compatible new features
└───── MAJOR: breaking changes

Pre-release examples:
v1.0.0-alpha.1
v1.0.0-beta.2
v1.0.0-rc.1          # release candidate

Build metadata (ignored for precedence):
v1.0.0+20240315
v1.0.0-beta.1+exp.sha.5114f85

# Common tagging workflow:
git tag -a v1.0.0-rc.1 -m "Release candidate 1"
# ... testing ...
git tag -a v1.0.0 -m "Release v1.0.0 — stable"
git push origin v1.0.0
```

---

## 11. Advanced Features

### git cherry-pick

Cherry-pick applies the changes from a specific commit onto the current branch — creating a new commit with a different SHA but the same changes.

```bash
# Apply a single commit to current branch
git cherry-pick abc1234

# Apply multiple commits
git cherry-pick abc1234 def5678 789abcd     # three specific commits
git cherry-pick abc1234..def5678            # range: commits AFTER abc1234 up to def5678
git cherry-pick abc1234^..def5678           # range: including abc1234

# Options
git cherry-pick abc1234 --no-commit         # apply changes but don't commit (stage only)
git cherry-pick abc1234 -x                  # append "cherry picked from commit abc1234" to message
git cherry-pick abc1234 -e                  # edit commit message before committing

# Handling cherry-pick conflicts
git cherry-pick --continue                  # after resolving conflicts
git cherry-pick --abort                     # abandon the cherry-pick
git cherry-pick --skip                      # skip this commit, continue with next
```

> 💡 **Tip:** Cherry-pick is great for backporting bug fixes to maintenance branches. Workflow: fix on `main`, then `git cherry-pick` the fix commit onto `release/1.x`. Use `-x` to record where the cherry-pick came from.

---

### git worktree

Worktrees let you have **multiple working directories** from a single repository — check out different branches simultaneously without stashing or cloning.

```bash
# List current worktrees
git worktree list

# Add a new worktree (checked out to a branch)
git worktree add ../project-feature feature/login     # existing branch
git worktree add -b hotfix/security ../project-hot    # create new branch

# Add worktree for a specific commit (detached HEAD)
git worktree add --detach ../project-review abc1234

# Remove a worktree (also deletes the directory)
git worktree remove ../project-feature

# Remove a worktree that has changes (force)
git worktree remove --force ../project-feature

# Clean up stale worktree references (directory was manually deleted)
git worktree prune
```

> 💡 **Tip:** Use worktrees when you need to quickly fix a production bug while mid-feature. Instead of stashing and switching, just `git worktree add ../hotfix hotfix/critical-fix` and work in a separate directory — your feature branch is untouched.

---

### git submodule

Submodules let you include another Git repository as a subdirectory of your repo.

```bash
# --- Adding a submodule ---
git submodule add https://github.com/lib/libfoo lib/libfoo
# Creates: .gitmodules file + lib/libfoo directory (at a specific commit)

# --- Cloning a repo that has submodules ---
git clone --recurse-submodules https://github.com/user/repo.git
# or for an existing clone:
git submodule init                  # initialise submodule config
git submodule update                # checkout submodule at pinned commit

# Combined:
git submodule update --init --recursive  # init + update all nested submodules

# --- Updating submodules ---
# Update all submodules to their latest remote commit
git submodule update --remote
git submodule update --remote --merge   # merge latest into local submodule

# Update a specific submodule
git submodule update --remote lib/libfoo

# --- Working inside a submodule ---
cd lib/libfoo
git checkout main                   # submodules start in detached HEAD
git pull                            # update to latest
cd ../..
git add lib/libfoo                  # stage the updated submodule pointer
git commit -m "chore: update libfoo to latest"

# --- Removing a submodule ---
git submodule deinit lib/libfoo     # unregister
git rm lib/libfoo                   # remove from index and disk
rm -rf .git/modules/lib/libfoo      # remove cached submodule data
```

> ⚠️ **Warning:** Submodules can be tricky — they pin to a specific commit (not a branch), and forgetting `--recurse-submodules` when cloning is a common mistake. Consider alternatives like `git subtree` or package managers if the complexity isn't worth it.

---

### git sparse-checkout

Sparse checkout lets you check out **only a subset of a large repository's files**, saving disk space and I/O.

```bash
# --- Enable sparse checkout (cone mode — fastest) ---
git clone --filter=blob:none --sparse https://github.com/user/large-repo.git
cd large-repo
git sparse-checkout init --cone      # cone mode: specify directories (not globs)

# Specify which directories to materialise
git sparse-checkout set src/ docs/   # only src/ and docs/ on disk

# Add more directories
git sparse-checkout add tests/

# List active sparse-checkout paths
git sparse-checkout list

# Disable sparse checkout (materialise all files)
git sparse-checkout disable

# --- Non-cone mode (pattern-based, slower but more flexible) ---
git sparse-checkout init
git sparse-checkout set '*.md' 'src/**/*.py'   # glob patterns
```

---

### git archive

```bash
# Export repo contents to a tar/zip archive (without .git/)
git archive HEAD --format=tar.gz > project.tar.gz
git archive HEAD --format=zip > project.zip

# Archive a specific branch/tag/commit
git archive v1.0.0 --format=tar.gz > release-v1.0.0.tar.gz
git archive --prefix=myproject-1.0/ v1.0.0 > myproject-1.0.tar.gz
                                    # --prefix adds a directory prefix inside the archive

# Archive only a specific subdirectory
git archive HEAD:src/ --format=tar > src.tar

# Export via remote (without cloning)
git archive --remote=git@github.com:user/repo.git HEAD > remote-snapshot.tar
```

---

### git bundle

```bash
# Bundle a repo for offline transfer (creates a single binary file)
git bundle create repo.bundle HEAD main    # bundle HEAD + main branch
git bundle create repo.bundle --all        # bundle everything

# Verify a bundle
git bundle verify repo.bundle

# Clone from a bundle
git clone repo.bundle my-repo

# Fetch from a bundle (incremental update)
git bundle create updates.bundle v1.0.0..HEAD   # only changes since v1.0.0
git fetch repo.bundle HEAD:remote-main           # fetch from bundle into branch
```

---

### git filter-repo

`git filter-repo` (install separately) is the modern replacement for `git filter-branch` — rewrite history to remove files, move files, or modify commit messages at scale.

```bash
# Install: pip install git-filter-repo

# --- Remove a file from ALL history (e.g. accidentally committed secret) ---
git filter-repo --path secrets.env --invert-paths   # remove secrets.env from all commits
git filter-repo --path-glob '*.key' --invert-paths  # remove all .key files

# --- Keep only specific paths (extract a subdirectory as its own repo) ---
git filter-repo --path src/mymodule/       # keep only src/mymodule/ in history

# --- Rename/move paths in history ---
git filter-repo --path-rename src/:lib/    # rename src/ to lib/ across all history

# --- Rewrite commit messages ---
git filter-repo --message-callback '
  return message.replace(b"JIRA-", b"TICKET-")
'

# After filter-repo, force push to all remotes (everyone must re-clone or rebase)
git remote add origin https://github.com/user/repo.git
git push origin --force --all
git push origin --force --tags
```

> ⚠️ **Warning:** `git filter-repo` rewrites ALL commit SHAs. This is a destructive, irreversible operation on shared history. Take a backup first (`git clone --mirror`), and coordinate with all contributors — they must all re-clone after this.

---

### git notes

Notes attach metadata to commits without rewriting them (the commit SHA stays the same).

```bash
# Add a note to the current commit
git notes add -m "Reviewed by Alice on 2024-03-15"

# Add a note to a specific commit
git notes add -m "Deploy notes: requires DB migration" abc1234

# Show notes with log
git log --show-notes

# Edit an existing note
git notes edit HEAD

# Remove a note
git notes remove HEAD

# Push notes to remote (not pushed by default)
git push origin refs/notes/*

# Fetch notes from remote
git fetch origin refs/notes/*:refs/notes/*
```

---

## 12. Internals & Plumbing

### .git/ Directory

```
.git/
├── HEAD              ← Current branch or commit SHA (e.g: "ref: refs/heads/main")
├── config            ← Repo-local git config
├── description       ← Used by GitWeb (rarely relevant)
├── index             ← The staging area (binary file)
│
├── objects/          ← All Git objects (blobs, trees, commits, tags)
│   ├── info/         ← Object pack information
│   ├── pack/         ← Packfiles (.pack + .idx) for compressed storage
│   ├── ab/           ← Loose objects: first 2 chars of SHA = dir name
│   └── cd/ef.../     ← remaining 38 chars = filename
│
├── refs/             ← Named references (pointers to SHAs)
│   ├── heads/        ← Local branches (e.g. heads/main = tip of main)
│   ├── tags/         ← Tags (e.g. tags/v1.0.0)
│   └── remotes/      ← Remote-tracking branches
│       └── origin/
│           └── main
│
├── logs/             ← Reflogs (history of where refs have pointed)
│   ├── HEAD          ← HEAD reflog
│   └── refs/heads/   ← Per-branch reflog
│
├── hooks/            ← Executable hook scripts
│   ├── pre-commit.sample
│   ├── commit-msg.sample
│   └── ...
│
├── info/
│   └── exclude       ← Local .gitignore (not committed)
│
├── COMMIT_EDITMSG    ← Last commit message (for re-use)
├── MERGE_HEAD        ← Exists during a merge (SHA of incoming commit)
├── MERGE_MSG         ← Pre-filled merge commit message
├── ORIG_HEAD         ← Previous HEAD before merge/rebase/reset
└── packed-refs       ← Packed format for refs (replaces individual ref files)
```

---

### Object Types

```bash
# Git stores 4 types of objects. Each is identified by SHA-1 of its content.

# --- BLOB: file content (no metadata) ---
git hash-object -w file.txt               # store a file as a blob, print its SHA
git cat-file -t abc1234                   # show object type: "blob"
git cat-file -p abc1234                   # show blob content (raw file text)
git cat-file blob abc1234                 # same as -p for blobs

# --- TREE: directory snapshot ---
git ls-tree HEAD                          # list tree at HEAD
git ls-tree HEAD src/                     # list tree of a subdirectory
git ls-tree -r HEAD                       # recursive — list all files
git ls-tree -r --name-only HEAD           # just file paths
git cat-file -p HEAD^{tree}               # show root tree of HEAD

# ls-tree output format:
# <mode> <type> <SHA>    <filename>
# 100644 blob abc1234    README.md
# 100755 blob def5678    run.sh     (executable)
# 040000 tree 789abcd    src/
# 160000 commit aaabbb   lib/sub    (submodule)

# --- COMMIT: snapshot with metadata ---
git cat-file -p HEAD                      # show commit object
# output:
# tree   a1b2c3d4...      (root tree SHA)
# parent f0e1d2c3...      (parent commit SHA — missing for root commit)
# author Alice <a@b.com> 1700000000 +0000
# committer Alice <a@b.com> 1700000000 +0000
#
# feat: add login endpoint

# --- TAG: annotated tag object ---
git cat-file -p v1.0.0                    # show annotated tag object
# output:
# object abc1234...       (commit SHA this tag points to)
# type commit
# tag v1.0.0
# tagger Alice <a@b.com> 1700000000 +0000
#
# Release version 1.0.0
```

---

### Plumbing Commands

```bash
# hash-object — compute/store object SHA
git hash-object file.txt               # compute SHA without storing
git hash-object -w file.txt            # compute AND write to object store

# ls-files — list files in the index (staging area)
git ls-files                           # all tracked files
git ls-files -s                        # with SHA and stage number
git ls-files -u                        # only unmerged (conflicted) files
git ls-files --others                  # untracked files
git ls-files --ignored --exclude-standard  # ignored files

# rev-parse — translate names to SHAs
git rev-parse HEAD                     # full SHA of HEAD
git rev-parse HEAD~3                   # SHA of 3 commits before HEAD
git rev-parse --short HEAD             # short SHA (7 chars)
git rev-parse --abbrev-ref HEAD        # current branch name
git rev-parse --show-toplevel          # absolute path to repo root

# for-each-ref — iterate over refs (useful for scripting)
git for-each-ref refs/heads/           # all local branches with metadata
git for-each-ref --format="%(refname:short) %(objectname:short)" refs/heads/

# update-ref — directly set a ref
git update-ref refs/heads/new-branch HEAD  # create a branch at HEAD (low-level)

# symbolic-ref — read/set symbolic refs
git symbolic-ref HEAD                  # shows "refs/heads/main"
git symbolic-ref HEAD refs/heads/main  # set HEAD to point to main
```

---

### Packfiles & GC

```bash
# Git initially stores objects as loose files (one file per object)
# Packfiles consolidate objects for efficient storage using delta compression

# --- Garbage collection (pack + prune) ---
git gc                                 # runs full GC: pack-refs, repack, prune, rerere gc
git gc --aggressive                    # more thorough repacking (slow — for maintenance)
git gc --auto                          # run GC only if thresholds are exceeded (default behaviour)

# --- Prune unreachable objects ---
git prune                              # remove loose unreachable objects (default: >2 weeks old)
git prune --expire=now                 # remove ALL unreachable objects immediately
git remote prune origin                # remove stale remote-tracking refs

# --- Check object store integrity ---
git fsck                               # verify all objects are intact and reachable
git fsck --unreachable                 # show unreachable (dangling) objects
git fsck --lost-found                  # write dangling objects to .git/lost-found/

# --- Manual repacking ---
git repack -ad                         # repack all loose objects into a single packfile
git repack -adf                        # also remove redundant objects

# --- Count objects (check repo size) ---
git count-objects -vH                  # verbose object count in human-readable sizes
```

---

### Git Hooks

Hooks are executable scripts in `.git/hooks/` that run at specific Git events.

```bash
# List available hook samples
ls .git/hooks/
# pre-commit.sample  commit-msg.sample  pre-push.sample  post-receive.sample ...

# Enable a hook: remove the .sample extension and make it executable
cp .git/hooks/pre-commit.sample .git/hooks/pre-commit
chmod +x .git/hooks/pre-commit

# Hooks in .git/hooks/ are LOCAL and NOT committed to the repo
# For shareable hooks, use a tool like Husky (Node.js) or pre-commit framework
```

```bash
#!/bin/sh
# .git/hooks/pre-commit — runs before each commit
# Exit non-zero to ABORT the commit

# Example: run linting before every commit
npm run lint
if [ $? -ne 0 ]; then
  echo "❌ Lint failed. Fix errors before committing."
  exit 1
fi

# Example: prevent committing directly to main
branch=$(git rev-parse --abbrev-ref HEAD)
if [ "$branch" = "main" ]; then
  echo "❌ Direct commits to main are not allowed. Use a feature branch."
  exit 1
fi
```

```bash
#!/bin/sh
# .git/hooks/commit-msg — validates commit message format
# $1 = path to file containing the commit message

commit_msg=$(cat "$1")

# Enforce Conventional Commits format
if ! echo "$commit_msg" | grep -qE "^(feat|fix|docs|style|refactor|test|chore|perf|ci|build|revert)(\(.+\))?: .{1,72}"; then
  echo "❌ Commit message must follow Conventional Commits format:"
  echo "   type(scope): description"
  echo "   e.g: feat(auth): add OAuth2 login"
  exit 1
fi
```

```bash
#!/bin/sh
# .git/hooks/pre-push — runs before git push
# Prevents pushing if tests fail

echo "🧪 Running tests before push..."
npm test
if [ $? -ne 0 ]; then
  echo "❌ Tests failed. Push aborted."
  exit 1
fi
```

```bash
#!/bin/sh
# .git/hooks/post-receive — SERVER-SIDE: runs after a push is received
# Used for: deploy triggers, notifications, CI integration
# Runs on the remote (server) repo, not locally

while read oldrev newrev refname; do
  branch=$(echo $refname | sed 's/refs\/heads\///')
  echo "📦 Received push to branch: $branch"

  if [ "$branch" = "main" ]; then
    echo "🚀 Deploying main to production..."
    cd /var/www/myapp && git pull origin main && systemctl restart myapp
  fi
done
```

**Client-side hooks:**

| Hook | When it runs | Common use |
|------|-------------|-----------|
| `pre-commit` | Before commit is made | Lint, tests, format |
| `prepare-commit-msg` | Before editor opens for message | Prepopulate message |
| `commit-msg` | After message is typed | Validate format |
| `post-commit` | After commit completes | Notifications |
| `pre-rebase` | Before rebase starts | Safety checks |
| `post-checkout` | After switch/checkout | Setup, build |
| `pre-merge-commit` | Before merge commit | Lint |
| `pre-push` | Before push to remote | Tests |
| `post-merge` | After merge completes | Install deps |

**Server-side hooks:**

| Hook | When it runs | Common use |
|------|-------------|-----------|
| `pre-receive` | Before any refs are updated | Access control |
| `update` | Once per ref being updated | Per-branch access control |
| `post-receive` | After all refs updated | Deploy, notify |
| `post-update` | After `update` hook | Update server info |

---

### Special Refs

```bash
# ORIG_HEAD — set by commands that dramatically move HEAD
# (merge, rebase, reset, am)
# Use it to undo such operations:
git reset --hard ORIG_HEAD          # undo last merge/rebase/reset

# MERGE_HEAD — the commit being merged (exists during an active merge)
git cat-file -p MERGE_HEAD          # inspect the incoming commit

# CHERRY_PICK_HEAD — commit being cherry-picked (during active cherry-pick)
git show CHERRY_PICK_HEAD

# REBASE_HEAD — commit being applied (during active rebase)

# FETCH_HEAD — last commit fetched from any remote
git log FETCH_HEAD                  # see what was just fetched
git merge FETCH_HEAD                # merge what was just fetched

# All special refs can be used anywhere a commit SHA is accepted:
git diff ORIG_HEAD HEAD             # what changed since before the merge?
git log ORIG_HEAD..HEAD             # what commits came in during the merge?
```

---

## 13. Workflows & Collaboration

### Gitflow

Gitflow is a branching model for release-heavy projects with scheduled releases.

```
main (or master)   ──────────────────────────────●────────────●──────────▶
                                                 ↑  v1.0      ↑  v1.1
                                    merge ──────┘            │
                                                              │
release/1.0       ─────────────────────────────●────────────┘
                                               ↑ (bugfix only)
develop           ──────────────────────────────────────────────────────▶
                       ↑          ↑                 ↑
                  feature/A  feature/B         feature/C

hotfix/1.0.1      (from main → fix → merge to main + develop)
```

```bash
# Main branches:
# main (or master) — production-ready code, always stable, tagged with versions
# develop          — integration branch for features, next release base

# Supporting branches:
# feature/*        — from develop, merged back to develop
# release/*        — from develop, merged to main + develop, then tagged
# hotfix/*         — from main, merged to main + develop

# --- Feature branch ---
git switch -c feature/user-auth develop    # branch from develop
# ... work ...
git switch develop
git merge --no-ff feature/user-auth        # always merge commit for history
git branch -d feature/user-auth

# --- Release branch ---
git switch -c release/1.2 develop
# (bump versions, update changelog — no new features)
git switch main && git merge --no-ff release/1.2
git tag -a v1.2.0 -m "Release 1.2.0"
git switch develop && git merge --no-ff release/1.2
git branch -d release/1.2

# --- Hotfix branch ---
git switch -c hotfix/1.2.1 main
# (fix the critical bug)
git switch main && git merge --no-ff hotfix/1.2.1
git tag -a v1.2.1 -m "Hotfix 1.2.1"
git switch develop && git merge --no-ff hotfix/1.2.1
git branch -d hotfix/1.2.1
```

> 💡 **Tip:** Use the `gitflow` CLI tool (`brew install git-flow`) to automate the branch creation/merge workflow. `git flow feature start my-feature` handles everything automatically.

---

### GitHub Flow

Simple, lightweight flow for continuous delivery — one main branch, short-lived feature branches, PRs.

```
main: ──────────────────────────────────────────────────────────────────▶
         ↑                 ↑                      ↑
   PR merged         PR merged               PR merged
         │                 │                      │
feature: └──(2-3 days)─────┘  feature-B: └──────┘
```

```bash
# 1. Create a feature branch from (up-to-date) main
git switch main && git pull
git switch -c feature/add-payments

# 2. Commit early and often (push often too)
git commit -m "feat: add Stripe integration"
git push -u origin feature/add-payments

# 3. Open a Pull Request (in GitHub/GitLab) — for review + discussion
# 4. Address review comments with additional commits
# 5. Merge PR (squash merge recommended for clean main history)
# 6. Delete the feature branch
git branch -d feature/add-payments
git push origin --delete feature/add-payments

# 7. Deploy main (GitHub Flow assumes main is always deployable)
```

---

### GitLab Flow

Adds environment branches to GitHub Flow — bridges the gap between GitHub Flow and Gitflow.

```
main ──────────────────────────────────────────────────▶ (development)
         ↓ (promote)
pre-production ───────────────────────────────────────▶ (staging)
         ↓ (promote)
production ───────────────────────────────────────────▶ (live)

For versioned releases, also use:
main → release/2.x → (cherry-pick hotfixes)
```

```bash
# Environment branches track deployment environments
# Merges flow downstream: main → pre-production → production

# Cherry-pick hotfixes to release branches:
git switch release/2.x
git cherry-pick abc1234 -x           # -x traces origin
git push origin release/2.x
```

---

### Trunk-Based Development

Everyone commits frequently (multiple times/day) to a single `main`/`trunk` branch. Features are hidden behind feature flags, not long-lived branches.

```
trunk: A─B─C─D─E─F─G─H─I─J─K─L─M ▶  (everyone commits here, often)
              │               │
              release/1.0     release/1.1  (cut from trunk when ready)
```

```bash
# Short-lived feature branches (max 1-2 days)
git switch -c feat/short-lived-work main
# ... small, incremental commits ...
git push -u origin feat/short-lived-work
# Open PR → CI passes → squash merge → delete branch
# Typically: < 1 day lifecycle

# Feature flags hide incomplete features
# (no long-lived branches for WIP features)

# Integration happens continuously (CI must be fast and reliable)
```

---

### Fork & Pull Request

```bash
# 1. Fork the repo on GitHub (creates your personal copy)

# 2. Clone YOUR fork
git clone git@github.com:yourusername/project.git
cd project

# 3. Add the original repo as "upstream"
git remote add upstream https://github.com/original/project.git
git remote -v  # verify: origin (your fork) + upstream (original)

# 4. Keep your fork's main in sync with upstream
git fetch upstream
git switch main
git rebase upstream/main            # or: git merge upstream/main
git push origin main                # update your fork

# 5. Create a feature branch (from fresh main)
git switch -c fix/improve-docs

# 6. Make changes, commit, push to YOUR fork
git push -u origin fix/improve-docs

# 7. Open a Pull Request: your fork's branch → upstream's main
# 8. Address review comments (push more commits to the same branch)
# 9. Maintainer merges your PR
# 10. Clean up
git switch main
git fetch upstream && git rebase upstream/main
git push origin main
git branch -d fix/improve-docs
git push origin --delete fix/improve-docs
```

---

### Branch Naming Conventions

```bash
# Common prefixes:
feature/description     # new feature
feat/description        # (shorter variant)
fix/description         # bug fix
hotfix/description      # urgent production fix
bugfix/description      # (synonym for fix/)
release/version         # release preparation
chore/description       # maintenance, dependency updates
docs/description        # documentation only
test/description        # test additions/fixes
refactor/description    # code restructuring
experiment/description  # exploratory/spike work

# Include ticket numbers for traceability:
feature/JIRA-123-add-user-login
fix/GH-456-correct-email-validation
hotfix/TICKET-789-fix-payment-timeout

# Rules:
# - Use lowercase and hyphens (not underscores or spaces)
# - Be descriptive but concise
# - Avoid special characters
# - Max ~50 characters

# Examples:
git switch -c feature/PROJ-101-oauth-login
git switch -c fix/null-pointer-in-cart-checkout
git switch -c release/2.4.0
git switch -c hotfix/critical-xss-vulnerability
```

---

### Protecting Branches

These are platform settings (GitHub/GitLab/Bitbucket), but here's what to configure:

```
Branch protection rules for "main" / "production":

✅ Require pull request reviews before merging (min 1-2 approvals)
✅ Dismiss stale pull request approvals when new commits pushed
✅ Require review from Code Owners (CODEOWNERS file)
✅ Require status checks to pass before merging (CI must pass)
✅ Require branches to be up to date before merging
✅ Require signed commits
✅ Include administrators (don't bypass rules for admins)
✅ Restrict who can push to matching branches
✅ Allow force pushes: OFF
✅ Allow deletions: OFF
```

```bash
# CODEOWNERS file (in repo root, .github/, or docs/)
# Automatically requests review from owners when their files change

# Global owner (reviews everything)
*                   @org/core-team

# Backend: reviewed by backend team
src/api/            @org/backend-team
src/db/             @org/backend-team @alice

# Frontend: reviewed by frontend team
src/ui/             @org/frontend-team

# Security-sensitive files
.github/workflows/  @org/devops-team
*.key               @org/security-team
```

---

## 14. Performance & Maintenance

### gc, prune, fsck

```bash
# --- git gc (garbage collection) ---
git gc                              # routine GC: pack loose objects, prune expired reflog, rerere gc
git gc --aggressive                 # deep optimisation (slow — use rarely, e.g. once/month for big repos)
git gc --prune=now                  # also prune ALL unreachable objects immediately
git gc --auto                       # only GC if auto thresholds exceeded (run by git commands internally)
git gc --quiet                      # suppress output

# --- git prune ---
git prune                           # remove unreachable loose objects older than 2 weeks
git prune --expire=now              # remove ALL unreachable loose objects immediately
git prune -n                        # dry run: show what would be pruned
git remote prune origin             # prune stale remote-tracking branches

# --- git fsck (filesystem check) ---
git fsck                            # full integrity check (all reachable objects)
git fsck --unreachable              # also check unreachable objects
git fsck --dangling                 # report dangling (unreachable) objects
git fsck --lost-found               # write dangling objects to .git/lost-found/
                                    # (useful for recovering deleted content)

# Diagnose repo size issues
git count-objects -vH               # count loose + packed objects, total size
git gc && git count-objects -vH     # see how much GC saves
```

---

### Shallow & Partial Clones

```bash
# --- Shallow clones (truncated history) ---
# Ideal for CI/CD where you only need the latest code

git clone --depth 1 https://github.com/user/repo.git         # latest commit only
git clone --depth 50 https://github.com/user/repo.git        # last 50 commits
git clone --shallow-since="6 months ago" https://...         # last 6 months

# Deepen a shallow clone
git fetch --deepen=100              # fetch 100 more commits of history
git fetch --unshallow               # download FULL history (makes it a full clone)

# --- Partial clones (lazy blob loading) ---
# Download commits + trees, but fetch file contents (blobs) on demand

# Skip all blobs (fastest download — only get files when you need them)
git clone --filter=blob:none https://github.com/user/repo.git

# Skip large blobs (only fetch blobs > 1MB on demand)
git clone --filter=blob:limit=1m https://github.com/user/repo.git

# Skip all trees and blobs (blobless + treeless — for fastest history browsing)
git clone --filter=tree:0 https://github.com/user/repo.git

# Fetch missing objects on demand automatically
git config core.partialCloneFilter blob:none  # make partial clone setting permanent

# --- Combine for minimal CI clone ---
git clone --depth 1 --filter=blob:none --single-branch --branch main \
  https://github.com/user/repo.git
```

> 💡 **Tip:** For most CI pipelines, `--depth 1` is sufficient. For monorepos where you need `git blame` or `git log -- path`, prefer `--filter=blob:none` without `--depth` — it gives you full commit history with lazy blob fetching.

---

### git maintenance

`git maintenance` (Git 2.29+) runs background optimisation tasks on a schedule.

```bash
# Start background maintenance (registers with OS scheduler: cron / launchd / Task Scheduler)
git maintenance start               # enables daily GC, hourly prefetch, etc.

# Stop background maintenance
git maintenance stop

# Run maintenance tasks immediately
git maintenance run                 # run all enabled tasks
git maintenance run --task=gc       # run only garbage collection
git maintenance run --task=fetch    # run only prefetch
git maintenance run --task=commit-graph   # update commit-graph file (faster log/blame)
git maintenance run --task=loose-objects  # pack loose objects
git maintenance run --task=incremental-repack  # incremental packfile repack

# Configure maintenance schedule
git config maintenance.fetch.enabled true
git config maintenance.commit-graph.enabled true
git config maintenance.loose-objects.enabled true
git config maintenance.incremental-repack.enabled true

# Check maintenance registration
git maintenance list
cat ~/.config/git/schedule           # view schedule file
```

---

### Git LFS

Git LFS (Large File Storage) replaces large files with text pointers inside Git, storing actual file contents on a separate LFS server.

```bash
# Install Git LFS (brew install git-lfs / apt install git-lfs)
git lfs install                     # initialise LFS for your user account

# --- Track file types with LFS ---
git lfs track "*.psd"               # track all Photoshop files
git lfs track "*.mp4"               # track all videos
git lfs track "*.zip"               # track all ZIP files
git lfs track "assets/**"           # track everything in assets/
# This updates .gitattributes automatically

# Always commit .gitattributes first!
git add .gitattributes
git commit -m "chore: configure Git LFS tracking"

# --- Normal workflow (LFS is transparent) ---
git add design.psd                  # LFS intercepts and stores in LFS
git commit -m "add design mockup"
git push origin main                # LFS files pushed to LFS server separately

# --- LFS management commands ---
git lfs ls-files                    # list all files tracked by LFS
git lfs status                      # show LFS file status
git lfs track                       # list currently tracked patterns
git lfs untrack "*.zip"             # stop tracking a pattern
git lfs pull                        # download LFS files for current checkout
git lfs fetch --all                 # download ALL LFS objects
git lfs prune                       # remove old cached LFS objects locally

# --- Migrate existing repo to use LFS ---
git lfs migrate import --include="*.psd,*.mp4" --everything
git push origin --force --all
git push origin --force --tags
```

> 💡 **Tip:** LFS requires server-side support (GitHub, GitLab, Bitbucket all support it). Without LFS, store large binaries in object storage (S3, GCS) and reference them by URL — never commit large binaries directly to Git.

---

## 15. Quick Reference Tables

### Core Commands Summary

| Command | What It Does | Common Flags |
|---------|-------------|--------------|
| `git init` | Initialise a new repo | `--bare`, `--initial-branch=main` |
| `git clone` | Clone a remote repo | `--depth`, `--branch`, `--filter`, `--sparse` |
| `git status` | Show working tree status | `-s` (short), `-b` (branch) |
| `git add` | Stage changes | `-A`, `-p` (patch), `-u` |
| `git commit` | Create a commit | `-m`, `-a`, `--amend`, `-v`, `--fixup` |
| `git diff` | Show changes | `--staged`, `--stat`, `-w`, `--word-diff` |
| `git log` | Show commit history | `--oneline`, `--graph`, `--all`, `-p`, `--follow`, `-S` |
| `git show` | Show a commit/object | `HEAD:file`, `--stat` |
| `git branch` | Manage branches | `-a`, `-d`, `-D`, `-m`, `-vv`, `--merged` |
| `git switch` | Switch branches | `-c` (create), `--detach` |
| `git checkout` | Switch/restore (legacy) | `-b`, `--`, `-p` |
| `git merge` | Merge branches | `--no-ff`, `--squash`, `--ff-only`, `--abort` |
| `git rebase` | Rebase commits | `-i`, `--onto`, `--autosquash`, `--abort`, `--continue` |
| `git cherry-pick` | Apply specific commits | `-x`, `--no-commit`, `--abort` |
| `git stash` | Shelve changes | `push -m`, `pop`, `apply`, `list`, `drop`, `-u` |
| `git restore` | Discard/unstage changes | `--staged`, `--source=` |
| `git reset` | Move HEAD/index | `--soft`, `--mixed`, `--hard` |
| `git revert` | Undo via new commit | `--no-edit`, `--no-commit`, `-m` |
| `git clean` | Remove untracked files | `-n` (dry run), `-f`, `-fd`, `-fdx` |
| `git tag` | Manage tags | `-a`, `-s`, `-d`, `-l`, `-v` |
| `git remote` | Manage remotes | `-v`, `add`, `remove`, `set-url`, `prune` |
| `git fetch` | Download remote changes | `--all`, `--prune`, `--tags`, `--depth` |
| `git pull` | Fetch + integrate | `--rebase`, `--ff-only`, `--no-rebase` |
| `git push` | Upload to remote | `-u`, `--force-with-lease`, `--tags`, `--delete` |
| `git reflog` | Show HEAD history | `show`, `expire`, `delete` |
| `git bisect` | Binary search history | `start`, `good`, `bad`, `run`, `reset` |
| `git blame` | Show line authorship | `-L`, `-w`, `-C`, `--since` |
| `git worktree` | Multiple working dirs | `add`, `list`, `remove`, `prune` |
| `git submodule` | Manage submodules | `add`, `update --init --recursive`, `deinit` |
| `git gc` | Garbage collection | `--aggressive`, `--auto`, `--prune=now` |
| `git fsck` | Integrity check | `--unreachable`, `--lost-found` |
| `git archive` | Export repo as archive | `--format`, `--prefix` |
| `git bundle` | Offline repo transport | `create`, `verify`, `clone` |

---

### git reset Modes

```
Starting state:
  HEAD → commit C3
  Index: contains C3's snapshot
  Working Tree: has your edits

  C1 ── C2 ── C3   ← HEAD, main
                └── (your edits in working tree)
```

| Mode | HEAD | Index (Staging) | Working Tree | Use When |
|------|------|-----------------|--------------|----------|
| `--soft` | ✅ Moved | ❌ Unchanged | ❌ Unchanged | Undo commit, keep everything staged. Recommit differently. |
| `--mixed` (default) | ✅ Moved | ✅ Reset | ❌ Unchanged | Undo commit + unstage. Choose what to re-stage. |
| `--hard` | ✅ Moved | ✅ Reset | ✅ Reset ⚠️ | Completely discard commits AND all changes. |

```
git reset --soft HEAD~1:
  HEAD → C2  |  Index still has C3 changes staged  |  Working tree unchanged
  → git commit again to recommit

git reset HEAD~1 (--mixed):
  HEAD → C2  |  Index reset to C2  |  Working tree has your edits + C3's changes unstaged
  → choose what to re-stage

git reset --hard HEAD~1:
  HEAD → C2  |  Index reset to C2  |  Working tree reset to C2 (edits LOST) ⚠️
```

---

### git diff Variants

| Command | Compares | Shows |
|---------|----------|-------|
| `git diff` | Working tree vs Index | Unstaged changes |
| `git diff --staged` | Index vs HEAD | Staged changes (what will be committed) |
| `git diff HEAD` | Working tree vs HEAD | All changes since last commit |
| `git diff HEAD~1` | Working tree vs 2 commits ago | All changes in last 2 commits + working tree |
| `git diff abc def` | Two commit SHAs | Changes between those two commits |
| `git diff main feature` | Two branches | Changes feature has over main |
| `git diff main...feature` | Merge base vs feature | Only feature's changes (ignoring main's changes) |
| `git diff HEAD -- file` | Working tree vs HEAD for one file | Changes to that file since last commit |
| `git diff --stat` | Any of the above | Summary only (file names + line counts) |
| `git diff --word-diff` | Any of the above | Word-level changes (better for prose) |

---

### Commit Reference Syntax

| Syntax | Means | Example |
|--------|-------|---------|
| `HEAD` | Current commit | `git show HEAD` |
| `HEAD~` or `HEAD~1` | 1 commit before HEAD (first parent) | `git diff HEAD~` |
| `HEAD~N` | N commits before HEAD | `git reset HEAD~3` |
| `HEAD^` | Same as `HEAD~` (first parent) | `git log HEAD^` |
| `HEAD^2` | Second parent of HEAD (merge commits) | `git diff HEAD^2` |
| `HEAD^^` | First parent's first parent | `git show HEAD^^` |
| `HEAD~2^2` | Second parent of 2-commits-back | `git log HEAD~2^2` |
| `@{upstream}` or `@{u}` | Upstream tracking branch | `git diff @{u}` |
| `@{push}` | Configured push destination | `git log @{push}..` |
| `HEAD@{1}` | Previous HEAD position (reflog) | `git diff HEAD@{1}` |
| `main@{yesterday}` | Where main was yesterday | `git show main@{yesterday}` |
| `main@{3}` | Where main was 3 moves ago | `git log main@{3}` |
| `v1.0.0` | Annotated or lightweight tag | `git checkout v1.0.0` |
| `origin/main` | Remote tracking branch | `git log origin/main` |
| `abc1234` | Full or partial SHA | `git show abc1234` |
| `:/fix typo` | Most recent commit matching message | `git show :/fix typo` |
| `HEAD^{tree}` | The tree of HEAD | `git ls-tree HEAD^{tree}` |
| `HEAD:path/file` | File at HEAD | `git show HEAD:src/main.py` |
| `a..b` | Commits reachable from b, not a | `git log main..feature` |
| `a...b` | Symmetric difference | `git diff main...feature` |

---

### Most Useful Aliases

```ini
# Add these to ~/.gitconfig under [alias]

[alias]
  # --- Navigation & status ---
  st   = status -sb                                        # compact status with branch
  lg   = log --oneline --graph --all --decorate           # visual branch graph
  ll   = log --oneline --decorate -20                     # last 20 commits
  lp   = log --patch --follow                             # log with full diff + follow renames
  who  = shortlog -sn --no-merges                         # contributor leaderboard

  # --- Branching ---
  sw   = switch
  br   = branch -vv                                       # branches with tracking info
  bra  = branch -vva                                      # all branches including remote
  gone = !git branch -vv | grep ': gone]' | awk '{print $1}' | xargs git branch -d
                                                          # delete branches whose remote is gone

  # --- Committing ---
  ci   = commit
  ca   = commit --amend --no-edit                         # amend without editing message
  cm   = commit -m                                        # quick commit with message

  # --- Staging ---
  ap   = add -p                                           # interactive patch staging
  unstage = restore --staged                              # unstage all
  discard = restore                                       # discard working tree changes

  # --- Undo ---
  undo  = reset HEAD~1 --mixed                            # undo last commit, keep changes
  nuke  = reset --hard HEAD                               # discard ALL local changes ⚠️

  # --- Remote ---
  pushf = push --force-with-lease                         # safe force push
  pub   = push -u origin HEAD                             # publish current branch

  # --- Inspection ---
  aliases = config --get-regexp alias                     # list all aliases
  root    = rev-parse --show-toplevel                     # show repo root directory
  current = rev-parse --abbrev-ref HEAD                   # current branch name
  ahead   = log @{upstream}..HEAD --oneline               # commits not yet pushed
  behind  = log HEAD..@{upstream} --oneline               # commits not yet pulled

  # --- Cleanup ---
  prune-all = !git remote prune origin && git gc
  cleanup   = !git branch --merged main | grep -v main | xargs git branch -d
```

---

*Generated with ❤️ — For Git 2.39+*
*Official docs: [git-scm.com/docs](https://git-scm.com/docs) | Pro Git book (free): [git-scm.com/book](https://git-scm.com/book)*
