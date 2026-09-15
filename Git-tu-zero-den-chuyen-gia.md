# 🌿 Tài liệu Git: Từ Zero đến Chuyên gia

> **Mục tiêu**: Thành thạo Git: branching, merge, rebase, hooks, submodules, workflows GitFlow & trunk-based.

---

## 📑 Mục lục

| Phần | Nội dung |
|------|----------|
| [P0. Cơ bản](#p0) | Cài Git, config, init |
| [P1. Làm việc cơ bản](#p1) | add, commit, status, log |
| [P2. Branching & Merging](#p2) | Branch, merge, conflict |
| [P3. Remote](#p3) | push, pull, fetch, clone |
| [P4. Undo & Rewriting](#p4) | reset, revert, rebase, amend |
| [P5. Stashing & Tags](#p5) | stash, tag, release |
| [P6. Submodules](#p6) | submodule, subtree |
| [P7. Hooks](#p7) | pre-commit, post-commit |
| [P8. Workflows](#p8) | GitFlow, trunk-based |
| [P9. Troubleshooting](#p9) | Recover, debug |

---

<a id="p0"></a>
## P0. Cơ bản

### Bước 1: Cài đặt

```bash
# Linux
sudo apt install git

# macOS
brew install git

# Windows
winget install Git.Git
# hoặc tải từ https://git-scm.com/

# Verify
git --version
```

### Bước 2: Cấu hình ban đầu

```bash
# Identity (BẮT BUỘC cho commit)
git config --global user.name "Your Name"
git config --global user.email "[email protected]"

# Default editor
git config --global core.editor "vim"
# hoặc nano, code, subl

# Default branch
git config --global init.defaultBranch main

# Line ending
# Windows
git config --global core.autocrlf true
# macOS/Linux
git config --global core.autocrlf input

# Push behavior
git config --global push.default simple
git config --global push.autoSetupRemote true

# Pull behavior
git config --global pull.rebase true   # Rebase thay vì merge

# Aliases
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
git config --global alias.unstage 'reset HEAD --'
git config --global alias.last 'log -1 HEAD'
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.amend 'commit --amend --no-edit'

# Pretty log
git config --global log.decorate full
git config --global log.abbrevCommit true

# Credential helper
# Cache 1 giờ
git config --global credential.helper cache
# Hoặc lưu (không khuyến nghị)
git config --global credential.helper store
# Hoặc dùng SSH (khuyến nghị)
```

### Bước 3: SSH key cho GitHub/GitLab

```bash
# Generate
ssh-keygen -t ed25519 -C "[email protected]"
# Default: ~/.ssh/id_ed25519

# Add to ssh-agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# Copy public key
cat ~/.ssh/id_ed25519.pub
# Paste vào GitHub Settings > SSH keys

# Test
ssh -T [email protected]
ssh -T [email protected
```

### Bước 4: Khởi tạo repo

```bash
# Tạo mới
mkdir my-project && cd my-project
git init

# Clone có sẵn
git clone https://github.com/user/repo.git
git clone [email protected]:user/repo.git
git clone https://github.com/user/repo.git my-folder-name
git clone --depth 1 https://github.com/user/repo.git  # Shallow clone
git clone --branch develop https://github.com/user/repo.git
git clone --recurse-submodules https://github.com/user/repo.git

# Khám phá
ls -la
cat .git/HEAD       # Ref hiện tại
```

### Bước 5: Cấu trúc Git

```
.git/
├── HEAD              # Branch/commit hiện tại
├── config            # Repo-local config
├── description       # Mô tả (cho gitweb)
├── hooks/            # Hook scripts
│   ├── pre-commit
│   ├── post-commit
│   └── ...
├── info/
│   └── exclude       # Local .gitignore
├── objects/          # Git objects (blob, tree, commit, tag)
│   ├── 00/
│   ├── 1f/
│   └── ...
├── refs/
│   ├── heads/        # Local branches
│   ├── remotes/      # Remote branches
│   └── tags/         # Tags
└── packed-refs       # Optimized refs file
```

---

<a id="p1"></a>
## P1. Làm việc cơ bản

### Bước 1: Three areas

```
Working Directory  →  Staging Area (Index)  →  Repository (.git)
       edit       │        git add         │     git commit
                   │                        │
         untracked │     unstaged            │     committed
         staged    │     modified            │
```

### Bước 2: Add & Commit

```bash
# Status
git status
git status -s            # Short format

# Add files
git add file.txt              # File cụ thể
git add *.txt                  # Wildcard
git add .                      # Tất cả
git add -p                     # Interactive - chọn từng hunk
git add -A                     # All (including deleted)

# Commit
git commit -m "Initial commit"
git commit -m "Title" -m "Description body"
git commit -am "msg"           # Add tracked + commit
git commit --amend             # Sửa commit cuối
git commit --amend --no-edit   # Thêm file vào commit cuối (không đổi message)

# Format commit message (Conventional Commits)
git commit -m "feat: add user login"
git commit -m "fix: resolve race condition"
git commit -m "docs: update README"
git commit -m "chore: bump version"
```

### BƯớc 3: Status & Diff

```bash
# Status
git status
git status -sb     # Branch + short status

# Diff (working dir vs index)
git diff
git diff file.txt
git diff --stat           # Chỉ filenames
git diff --word-diff      # Word-level diff

# Diff (index vs last commit)
git diff --staged
git diff --cached

# Diff (working dir vs last commit)
git diff HEAD

# Diff between commits
git diff commit1 commit2
git diff branch1 branch2
git diff main..feature

# Diff specific file
git diff main -- file.txt

# Visual diff
git difftool             # Mở external tool
```

### BƯớc 4: Log

```bash
# Basic
git log
git log --oneline
git log --graph --oneline --all

# Decorate (branch & tag info)
git log --oneline --decorate
git log --oneline --graph --decorate --all
# Hoặc alias 'lg'

# Filter
git log --author="John"
git log --since="2 weeks ago"
git log --until="2024-01-01"
git log --before="1 month ago"
git log --grep="fix"
git log -S "function_name"          # Pickaxe - tìm code change
git log -- file.txt
git log --follow -- file.txt       # Theo dõi rename

# Format
git log --pretty=format:"%h %an %ar %s"
# %h - short hash
# %H - full hash
# %an - author name
# %ae - author email
# %ar - author date relative
# %ad - author date
# %cn - committer name
# %s - subject
# %b - body

# Show specific commit
git show <hash>
git show HEAD
git show HEAD~1
git show HEAD^                  # Parent
git show HEAD~5                 # 5 commits ago

# Range
git log main..feature           # Commits in feature not in main
git log main...feature          # Symmetric difference
```

### BƯớc 5: .gitignore

```bash
# ~/.gitignore_global (apply cho tất cả repos)
git config --global core.excludesFile ~/.gitignore_global
```

```gitignore
# Node
node_modules/
npm-debug.log
yarn-error.log
.env
.env.local

# Python
__pycache__/
*.pyc
.venv/
venv/

# Java
*.class
*.jar
target/
build/

# IDE
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Logs
*.log
logs/

# Secrets
*.pem
*.key
secrets.yaml

# Build
dist/
build/
*.min.js

# Specific
*.local
coverage/
```

```bash
# Force add (khi file bị ignore)
git add -f file.txt

# Track empty directory
mkdir logs
touch logs/.gitkeep

# Check why ignored
git check-ignore -v file.txt
git check-ignore -v *
```

---

<a id="p2"></a>
## P2. Branching & Merging

### Bước 1: Branch cơ bản

```bash
# List
git branch
git branch -a                # Tất cả (local + remote)
git branch -r                # Remote only
git branch --list "feat/*"

# Tạo
git branch feature-login
git checkout -b feature-login      # Tạo + switch
git switch -c feature-login        # Mới hơn (Git 2.23+)

# Chuyển branch
git checkout main
git switch main

# Rename
git branch -m old-name new-name
git branch -m new-name              # Rename current

# Delete
git branch -d feature-login         # Safe (check merged)
git branch -D feature-login         # Force

# Delete remote
git push origin --delete feature-login
git push origin :feature-login
```

### BƯớc 2: Merge

```bash
# Fast-forward merge (default nếu no divergence)
git checkout main
git merge feature-login

# No-fast-forward (luôn tạo merge commit)
git merge --no-ff feature-login -m "Merge feature-login"

# Squash merge (1 commit thay vì nhiều)
git merge --squash feature-login
git commit -m "Add login feature"

# Resolve conflicts
# 1. Edit files
# 2. git add <file>
# 3. git commit (merge sẽ auto-commit khi hết conflict)

# Abort merge
git merge --abort

# Continue after fixing
git add .
git merge --continue
```

### Bước 3: Conflict resolution

```bash
# Conflict markers
<<<<<<< HEAD
  code from current branch
=======
  code from merging branch
>>>>>>> feature-branch

# Tools
git mergetool                # Mở merge tool (vimdiff, meld, kdiff3)
git config --global merge.tool vimdiff

# Mark as resolved
git add resolved-file.txt

# Show conflict status
git diff --name-only --diff-filter=U

# Take one side
git checkout --ours file.txt
git checkout --theirs file.txt
```

### Bước 4: Rebase

```bash
# Rebase current branch lên main
git checkout feature-login
git rebase main

# Interactive rebase (chỉnh sửa history)
git rebase -i HEAD~5
git rebase -i main

# Trong editor:
# pick abc1234 Commit 1
# pick def5678 Commit 2
# edit ghi9012 Commit 3  <- sẽ dừng ở đây
# squash jkl3456 Commit 4
# drop mno7890 Commit 5

# Commands:
# pick/reword/edit/squash/fixup/drop/exec/label/reset/merge

# Reorder
git rebase -i HEAD~3
# Sắp xếp lại thứ tự commit

# Edit author
git commit --amend --author="New Name <[email protected]>"

# Auto-squash
git commit -m "fixup! Some commit"  # Auto-squash vào commit match
git rebase -i --autosquash main
```

### BƯớc 5: Merge vs Rebase

```
Merge:
   A---B---C  (main)
    \
     D---E---F  (feature)

Sau merge:
   A---B---C---M  (main)
    \         /
     D---E---F    (feature)


Rebase:
   A---B---C  (main)
    \
     D---E---F  (feature)

Sau rebase:
   A---B---C  (main)
            \
             D'--E'--F'  (feature)
```

```bash
# Rule of thumb:
# - Merge: cho shared branches (main, develop)
# - Rebase: cho local feature branches (trước khi push)

# Workflow: rebase local, merge shared
git checkout feature
git rebase main           # Update feature với main changes
git checkout main
git merge --no-ff feature  # Merge feature vào main
```

---

<a id="p3"></a>
## P3. Remote

### Bước 1: Remote cơ bản

```bash
# List
git remote -v

# Add
git remote add origin https://github.com/user/repo.git
git remote add upstream https://github.com/original/repo.git

# Remove
git remote remove origin
git remote rm origin

# Rename
git remote rename origin old-origin

# Change URL
git remote set-url origin [email protected]:user/repo.git

# Show info
git remote show origin

# Fetch (download without merge)
git fetch origin
git fetch origin main
git fetch --all                  # Tất cả remotes
git fetch --prune                # Xoá remote-tracking deleted refs
```

### Bước 2: Push & Pull

```bash
# Push
git push
git push origin main
git push -u origin main           # Set upstream
git push --tags
git push --follow-tags           # Push tags cho annotated tags
git push --force                 # Force (cẩn thận!)
git push --force-with-lease      # Safer force (check upstream)

# Pull
git pull                         # Fetch + merge
git pull --rebase                # Fetch + rebase
git pull --autostash             # Stash, pull, unstash

# Clone
git clone https://github.com/user/repo.git
git clone --branch develop --single-branch https://github.com/user/repo.git
git clone --depth 1 --branch main https://github.com/user/repo.git  # Shallow
git clone --mirror https://github.com/user/repo.git                # Mirror
```

### BƯớc 3: Tracking branches

```bash
# Tracking (upstream) - biết push/pull từ đâu
git branch -vv

# Set tracking
git branch -u origin/main main
git checkout --track origin/main

# Unset
git branch --unset-upstream
```

### Bước 4: Fork workflow

```bash
# 1. Fork trên GitHub/GitLab
# 2. Clone fork
git clone [email protected]:myuser/repo.git
cd repo

# 3. Add upstream
git remote add upstream https://github.com/original/repo.git

# 4. Work trên branch
git checkout -b fix-bug main

# 5. Commit & push
git commit -am "Fix bug"
git push -u origin fix-bug

# 6. Create PR trên GitHub/GitLab

# 7. Sync với upstream
git fetch upstream
git checkout main
git merge upstream/main
git push origin main

# 8. Rebase feature branch
git checkout fix-bug
git rebase main
```

### BƯớc 5: Submodules

```bash
# Add submodule
git submodule add https://github.com/user/lib.git libs/lib

# Clone với submodules
git clone --recurse-submodules https://github.com/user/repo.git
git submodule update --init --recursive

# Update
git submodule update --remote
git submodule foreach 'git pull origin main'

# Remove submodule
git submodule deinit libs/lib
git rm libs/lib
rm -rf .git/modules/libs/lib
```

---

<a id="p4"></a>
## P4. Undo & Rewriting

### Bước 1: Undo working changes

```bash
# Discard changes trong working directory
git checkout -- file.txt        # Old syntax
git restore file.txt            # New (Git 2.23+)
git checkout -- .               # Tất cả files
git restore .

# Unstage (keep changes trong working dir)
git reset HEAD file.txt
git restore --staged file.txt
```

### BƯớc 2: Undo commits

```bash
# Soft reset (giữ changes trong staging)
git reset --soft HEAD~1

# Mixed reset (default - giữ changes trong working dir)
git reset HEAD~1
git reset --mixed HEAD~1

# Hard reset (xoá everything - CẨN THẬN)
git reset --hard HEAD~1
git reset --hard <commit-hash>

# Revert (tạo commit mới undo commit cũ)
git revert <commit-hash>
git revert HEAD
git revert HEAD~3..HEAD           # Revert range
git revert --no-commit HEAD       # Revert without commit (cho cherry-pick)
```

### Bước 3: Cherry-pick

```bash
# Apply specific commit từ branch khác
git cherry-pick <commit-hash>
git cherry-pick <hash1> <hash2>
git cherry-pick <hash1>..<hash3>

# No commit
git cherry-pick --no-commit <hash>

# Continue after conflict
git cherry-pick --continue
git cherry-pick --abort
```

### Bước 4: Reflog - cứu mạng

```bash
# Xem lịch sử HEAD
git reflog
git reflog show HEAD
git reflog show main

# Khôi phục commit đã mất (sau reset --hard)
git reflog
# Tìm commit hash muốn khôi phục
git reset --hard <hash-from-reflog>

# Reflog expire
git reflog expire --expire=30.days.ago --all
git gc --prune=now --aggressive
```

### Bước 5: Amend & Author

```bash
# Amend last commit
git commit --amend
git commit --amend -m "New message"
git commit --amend --no-edit   # Add files vào commit cuối

# Amend author
git commit --amend --author="Name <[email protected]>" --no-edit

# Fix nhiều commits author (interactive rebase)
git rebase -i HEAD~5 --exec 'git commit --amend --author="Name <email>" --no-edit'

# Filter-branch (mass rewrite - deprecation warning)
# Dùng git-filter-repo thay thế
```

---

<a id="p5"></a>
## P5. Stashing & Tags

### Bước 1: Stash

```bash
# Save working changes
git stash
git stash save "WIP: working on feature"
git stash push -m "WIP message"
git stash push file.txt          # Stash specific file
git stash push -p               # Interactive (chọn hunk)
git stash push --keep-index     # Chỉ stash unstaged

# List
git stash list

# Show
git stash show
git stash show -p               # Show diff
git stash show stash@{0}        # Specific stash

# Apply
git stash apply                 # Apply latest
git stash apply stash@{2}       # Specific
git stash pop                   # Apply + drop
git stash drop                  # Delete latest
git stash drop stash@{2}

# Apply to branch
git stash branch new-branch stash@{0}

# Clear all
git stash clear
```

### BƯớc 2: Tags - releases

```bash
# Lightweight tag
git tag v1.0
git tag v1.0 <commit-hash>

# Annotated tag (recommended - có author, date, message)
git tag -a v1.0 -m "Release 1.0"
git tag -a v1.0 <hash> -m "Release notes"

# Signed tag (GPG)
git tag -s v1.0 -m "Signed release"

# List
git tag
git tag -l "v1.*"
git tag -n                        # Show messages

# Show
git show v1.0

# Push
git push origin v1.0
git push origin --tags            # Tất cả tags
git push --follow-tags            # Push annotated tags only

# Delete
git tag -d v1.0                   # Local
git push origin :v1.0             # Remote
git push origin --delete v1.0

# Checkout tag
git checkout v1.0                 # Detached HEAD
git checkout -b branch-from-tag v1.0
```

### Bước 3: Tag với changelog

```bash
git tag -a v1.0 -m "Release 1.0

Features:
- User authentication
- Dashboard
- API endpoints

Bugfixes:
- Memory leak in worker
- Race condition in cache

Breaking changes:
- /v1/users endpoint removed"

# Xem changelog
git tag -n10 v1.0
```

---

<a id="p6"></a>
## P6. Submodules & Worktrees

### Bước 1: Submodules

```bash
# Add
git submodule add https://github.com/user/lib.git path/to/lib

# Status
git submodule status

# Init khi clone
git submodule update --init --recursive

# Update to latest
git submodule update --remote
git submodule update --remote --merge
git submodule update --remote --rebase

# Foreach
git submodule foreach 'git checkout main && git pull'
git submodule foreach 'git status'

# Remove
git submodule deinit path/to/lib
git rm path/to/lib
rm -rf .git/modules/path/to/lib
```

```gitignore
# Trong .gitmodules (Git config)
[submodule "path/to/lib"]
    path = path/to/lib
    url = https://github.com/user/lib.git
    branch = main
```

### Bước 2: Git Subtree (alternative)

```bash
# Add remote as subtree
git subtree add --prefix=libs/lib https://github.com/user/lib.git main --squash

# Pull updates
git subtree pull --prefix=libs/lib https://github.com/user/lib.git main --squash

# Push changes back
git subtree push --prefix=libs/lib https://github.com/user/lib.git main

# Split out subtree
git subtree split --prefix=libs/lib -b lib-branch
```

### Bước 3: Worktrees (parallel working)

```bash
# Add worktree (parallel branch checkout)
git worktree add ../repo-hotfix hotfix-branch
git worktree add ../repo-experiment -b experiment-branch main
git worktree add ../repo-pr --detach

# List
git worktree list

# Remove
git worktree remove ../repo-hotfix
git worktree prune

# Use case:
# - Work on 2 branches cùng lúc
# - Test trên branch khác mà không cần stash
# - Compare 2 versions side-by-side
```

---

<a id="p7"></a>
## P7. Hooks & Automation

### Bước 1: Hook types

```
Client-side (local):
- pre-commit       # Trước commit message
- prepare-commit-msg
- commit-msg       # Sau message, trước commit
- post-commit      # Sau commit thành công
- pre-rebase
- post-merge
- pre-push         # Trước push

Server-side (remote):
- pre-receive
- update
- post-receive
```

### BƯớc 2: Setup hook

```bash
# Hooks trong .git/hooks/
ls .git/hooks/

# Enable: chmod +x .git/hooks/<hook-name>

# Dùng template để share hooks qua team
mkdir -p .githooks
git config core.hooksPath .githooks
```

### Bước 3: Pre-commit hook

```bash
#!/bin/bash
# .githooks/pre-commit

# Lint check
echo "Running lint..."
npm run lint
if [ $? -ne 0 ]; then
    echo "Lint failed. Commit aborted."
    exit 1
fi

# Tests
echo "Running tests..."
npm test
if [ $? -ne 0 ]; then
    echo "Tests failed. Commit aborted."
    exit 1
fi

# Check for secrets
echo "Checking for secrets..."
if git diff --cached | grep -E "password|api[_-]?key|secret" -i; then
    echo "Possible secret detected!"
    exit 1
fi

exit 0
```

```bash
chmod +x .githooks/pre-commit
```

### Bước 4: Pre-commit framework

```yaml
# .pre-commit-config.yaml
repos:
- repo: https://github.com/pre-commit/pre-commit-hooks
  rev: v4.5.0
  hooks:
  - id: trailing-whitespace
  - id: end-of-file-fixer
  - id: check-yaml
  - id: check-json
  - id: check-merge-conflict
  - id: detect-private-key

- repo: https://github.com/psf/black
  rev: 23.12.1
  hooks:
  - id: black
    language_version: python3.11

- repo: https://github.com/pre-commit/mirrors-eslint
  rev: v8.56.0
  hooks:
  - id: eslint
    files: \.jsx?$
```

```bash
pip install pre-commit
pre-commit install
pre-commit run --all-files
```

### Bước 5: Commit-msg hook

```bash
#!/bin/bash
# .githooks/commit-msg

commit_msg_file=$1
commit_msg=$(cat "$commit_msg_file")

# Conventional Commits regex
pattern="^(feat|fix|docs|style|refactor|test|chore|perf|ci|build|revert)(\(.+\))?: .{1,100}"

if ! echo "$commit_msg" | grep -qE "$pattern"; then
    echo "ERROR: Commit message does not follow Conventional Commits!"
    echo "Format: <type>(<scope>): <subject>"
    echo "Types: feat, fix, docs, style, refactor, test, chore, perf, ci, build, revert"
    exit 1
fi
```

### Bước 6: Husky (Node.js)

```bash
npm install --save-dev husky lint-staged

npx husky init
```

```bash
# .husky/pre-commit
npx lint-staged

# .husky/commit-msg
npx commitlint --edit $1
```

```json
// package.json
{
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{json,md}": ["prettier --write"]
  }
}
```

```bash
# commitlint
npm install --save-dev @commitlint/cli @commitlint/config-conventional
```

```js
// commitlint.config.js
module.exports = {
    extends: ['@commitlint/config-conventional'],
    rules: {
        'header-max-length': [2, 'always', 100]
    }
};
```

---

<a id="p8"></a>
## P8. Workflows

### BƯớc 1: GitFlow

```bash
# Setup
git flow init
# Branch names: master, develop, feature/*, release/*, hotfix/*

# Feature
git flow feature start login
# Tạo feature/login từ develop

git flow feature finish login
# Merge vào develop + delete branch

# Release
git flow release start 1.0
# Tạo release/1.0 từ develop

git flow release finish 1.0
# Merge vào master + develop + tag

# Hotfix
git flow hotfix start critical-bug
# Tạo hotfix/critical-bug từ master

git flow hotfix finish critical-bug
# Merge vào master + develop
```

```
Branches:
- master (main): production code
- develop: integration branch
- feature/*: new features
- release/*: preparing release
- hotfix/*: urgent fixes
```

```
master ────●─────●────────●── (tags)
            \    │         /
             \   │        /
develop ──────●───●───●──●──
              \  /     /
feature/login  ●─────●
                     /
feature/api    ●───●
```

### Bước 2: GitHub Flow

```bash
# Main branch = main (production-ready)

# Workflow:
# 1. Create branch from main
git checkout -b feat/login

# 2. Make commits
git commit -am "Login UI"
git commit -am "Login API"

# 3. Open PR (Pull Request)
# GitHub UI: Open PR từ feat/login -> main

# 4. Review & discuss
# - CI/CD runs automatically
# - Reviewers approve

# 5. Merge (squash)
# GitHub: "Squash and merge" button

# 6. Deploy (auto với CI/CD)
```

### BƯớc 3: Trunk-Based Development

```bash
# Main branch = main/trunk

# Workflow:
# - Short-lived feature branches (1-2 days max)
# - Feature flags for incomplete features
# - Frequent merges to main

# Workflow:
git checkout main
git pull
git checkout -b feat/small-change

# Work & commit
git commit -am "Small change"
git commit -am "Another commit"

# Quick rebase + merge
git fetch origin
git rebase origin/main
git push origin feat/small-change

# Merge vào main (squash hoặc rebase)
# CI auto-deploys
```

```bash
# Feature flags (config-driven)
if config.feature.newLogin {
    newLoginUI()
} else {
    legacyLoginUI()
}
```

### BƯớc 4: GitLab Flow

```
Production ───●────●────●
                \      \
Staging ──────●──●────●──●
                \    \
Feature ──────●───●───●──
```

```bash
# 3 branches:
# - production (current production)
# - stable (sẵn sàng cho release tiếp theo)
# - feature/* (working branches)

# Merge flow:
# feature → stable (test) → production (release)
```

### BƯớc 5: So sánh workflows

| Workflow | Best for | Pros | Cons |
|---------|----------|------|------|
| GitFlow | Versioned releases | Stable releases | Complex, slow |
| GitHub Flow | SaaS, continuous deploy | Simple, fast | Less control |
| Trunk-Based | High-performing teams | Fast, less conflict | Need CI/CD maturity |
| GitLab Flow | Mix of release-based + continuous | Flexible | Need planning |

---

<a id="p9"></a>
## P9. Troubleshooting

### Bước 1: Undo various scenarios

```bash
# "I committed but want to undo"
git revert HEAD                    # New commit undo
git reset --soft HEAD~1             # Undo, keep changes

# "I committed to wrong branch"
git reset --hard HEAD~3
git checkout correct-branch
git cherry-pick <hash1> <hash2> <hash3>

# "I want to change commit message"
git commit --amend -m "New msg"     # Last commit
git rebase -i HEAD~3                # Older commits

# "I lost my code"
git reflog
git reset --hard <hash>

# "Merge conflict"
git status                          # See conflicted files
# Edit manually
git add .
git commit                          # Complete merge

# "Want to undo merge"
git revert -m 1 <merge-commit-hash> # Revert merge commit
git reset --hard HEAD~1             # If not pushed
```

### BƯớc 2: Bisect - tìm bug commit

```bash
# Start bisect
git bisect start

# Mark current as bad
git bisect bad

# Mark known good commit
git bisect good v1.0

# Git sẽ checkout giữa, test và mark
git bisect good   # Hoặc bad

# Run automated test
git bisect run npm test

# Kết thúc
git bisect reset
```

### Bƛớc 3: Blame & Annotate

```bash
# Tìm ai đổi dòng nào
git blame file.txt
git blame -L 10,20 file.txt          # Range
git blame -C file.txt                # Detect copies

# Tìm commit thay đổi dòng cụ thể
git log -S "function_name" --source --all

# Tìm khi nào bug xuất hiện
git log --all --source --remotes --oneline -- file.txt
```

### BƯớc 4: Clean up

```bash
# Remove untracked files
git clean -n                          # Dry-run
git clean -fd                         # Force + directories
git clean -fdx                        # Including ignored

# Remove merged branches
git branch --merged main | grep -v "main" | xargs git branch -d

# Prune remote
git remote prune origin
git fetch --prune

# Garbage collection
git gc --aggressive
git prune

# Pack refs
git pack-refs --all
```

### BƯớc 5: Common errors & fixes

```bash
# 1. "fatal: refusing to merge unrelated histories"
git pull origin main --allow-unrelated-histories

# 2. "Your branch is ahead by X commits"
git pull   # Or rebase

# 3. "Your branch is behind"
git pull

# 4. "CONFLICT (content): Merge conflict"
# Edit files, then:
git add .
git commit

# 5. "Permission denied (publickey)"
ssh-add ~/.ssh/id_ed25519
# Check ssh config

# 6. "Repository not found"
# Check URL, credentials

# 7. "fatal: refusing to push"
# Push with force (cẩn thận)
git push --force-with-lease

# 8. "Detached HEAD"
git checkout -b new-branch    # Save current state

# 9. "Already up to date"
git fetch origin
git rebase origin/main

# 10. Lock file
rm .git/index.lock     # Only if safe (no other git process)
```

### Bước 6: Performance

```bash
# Shallow clone
git clone --depth 1 https://github.com/user/repo.git

# Partial clone (Blobless)
git clone --filter=blob:none https://github.com/user/repo.git

# Sparse checkout
git clone --no-checkout https://github.com/user/repo.git
cd repo
git sparse-checkout init --cone
git sparse-checkout set src tests docs

# Maintenance
git maintenance run
git maintenance start --schedule=daily

# Large repo
git config --global feature.manyFiles true
git config --global index.threads 8

# Speed up log
git log --oneline -20
```

---

### Bước 7: Useful commands cheat sheet

```bash
# Quick reference
git status
git add -A
git commit -m "msg"
git push
git pull --rebase

git checkout -b feat/x
git checkout main
git merge feat/x

git log --oneline -10
git diff
git stash
git stash pop

# Show what changed
git diff main..HEAD --stat
git log main..HEAD --oneline

# Emergency
git reflog                          # Find lost commits
git reset --hard <hash>             # Restore
git revert <hash>                   # Undo safely
```

---

## 🎯 Bài tập P0-P9

1. Init repo, commit, push lên GitHub
2. Branching: feature branch, merge vào main
3. Resolve merge conflict thực tế
4. Rebase feature branch lên main
5. Interactive rebase squash 5 commits thành 1
6. Stash work, switch branch, restore
7. Setup pre-commit hook kiểm tra syntax
8. GitFlow: feature → release → hotfix
9. Cherry-pick commit từ branch khác
10. Submodule add + clone với submodules

---

> **💡 Tip cuối**: Git là công cụ mạnh nhưng dễ confuse. Học từ từ: add/commit/push trước, branch/merge sau, rebase/cherry-pick khi đã vững. Luôn backup bằng push lên remote. Reflog là cứu mạng. Không force-push lên shared branch.

---

*Tạo bởi tài liệu học Git - Chúc bạn thành công! 🚀*
