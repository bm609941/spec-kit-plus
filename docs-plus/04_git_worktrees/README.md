# Git Worktree Support for SpecKit Plus

SpecKit Plus now supports git worktrees, allowing you to work on multiple features simultaneously in separate directories while sharing the same git history, specs, and prompt history.

## What are Git Worktrees?

Git worktrees let you check out multiple branches at once, each in its own directory. This is perfect for feature development where you need to:

- Work on multiple features without switching branches
- Test features side-by-side
- Keep a clean main branch while developing
- Avoid stashing changes when switching contexts

## Directory Structure

```
my-project/              ← Main repository (stays on main branch)
├── specs/               ← Shared across all worktrees
├── history/             ← Shared across all worktrees
├── templates/           ← Shared across all worktrees
├── scripts/             ← Shared across all worktrees
└── src/                 ← Main branch source code

worktrees/               ← Sibling directory with feature worktrees
├── 001-user-auth/       ← Worktree for feature 001
│   ├── src/             ← Feature 001 source code
│   └── .git             ← Points to main repo
├── 002-dashboard/       ← Worktree for feature 002
│   ├── src/             ← Feature 002 source code
│   └── .git             ← Points to main repo
└── 003-api-endpoint/    ← Worktree for feature 003
    ├── src/             ← Feature 003 source code
    └── .git             ← Points to main repo
```

**Important**: The `specs/` and `history/` directories are automatically accessed from the main repo root, even when working in a worktree. This means all features share the same specs and prompt history.

## Quick Start

### 1. List Existing Worktrees

```bash
# From your main repo
/sp.worktree list
```

This shows all active worktrees and their branches.

### 2. Create a New Worktree for a Feature

```bash
# Option A: Let SpecKit Plus create the worktree and generate branch name
/sp.worktree create "User authentication system"

# Option B: Create with explicit branch name
/sp.worktree create 001-auth
```

This will:
- Create a new branch (e.g., `001-user-auth`)
- Create a worktree at `../worktrees/001-user-auth`
- Set up the directory structure

### 3. Switch to the Worktree

```bash
cd ../worktrees/001-user-auth
```

### 4. Create Specification in Worktree

```bash
# In the worktree directory
/sp.specify "User authentication with OAuth2 and JWT tokens"
```

SpecKit Plus will automatically detect that you're in a worktree with an existing branch and:
- Skip branch creation (already done)
- Use the branch name to determine the spec directory
- Create the spec at `specs/001-user-auth/spec.md` in the main repo

### 5. Develop in the Worktree

Work on your feature normally:
- Edit code in the worktree
- Run `/sp.plan`, `/sp.implement`, etc.
- Commit and push from the worktree
- All specs and history remain in the main repo

### 6. Clean Up When Done

```bash
# After merging your feature
cd /path/to/main-repo
/sp.worktree remove ../worktrees/001-user-auth
```

## Workflows

### Workflow 1: Create Worktree with /sp.worktree

Best for starting a new feature:

```bash
# In main repo
/sp.worktree create "payment processing"

# Output shows:
# ✓ Worktree created: /path/to/worktrees/001-payment
# Branch: 001-payment

cd ../worktrees/001-payment
/sp.specify "Implement Stripe payment processing"
# ... develop feature ...
```

### Workflow 2: Enable Worktree Mode for All Features

Make all `/sp.specify` commands create worktrees automatically:

```bash
# In main repo
/sp.worktree enable

# This sets SPECIFY_WORKTREE_MODE=true

# Now /sp.specify creates worktrees instead of branches
/sp.specify "New dashboard analytics"

# Output:
# ✓ Worktree created at: /path/to/worktrees/002-dashboard
# To switch to this worktree, run: cd /path/to/worktrees/002-dashboard
```

To make worktree mode permanent, add to your shell profile:

**Bash/Zsh** (`~/.bashrc` or `~/.zshrc`):
```bash
export SPECIFY_WORKTREE_MODE=true
```

**PowerShell** (`$PROFILE`):
```powershell
$env:SPECIFY_WORKTREE_MODE = "true"
```

### Workflow 3: Multiple Features Simultaneously

```bash
# Create multiple worktrees
cd /path/to/main-repo

/sp.worktree create "user authentication"
# → ../worktrees/001-user-auth

/sp.worktree create "dashboard UI"
# → ../worktrees/002-dashboard

/sp.worktree create "API endpoints"
# → ../worktrees/003-api-endpoints

# Work on feature 1
cd ../worktrees/001-user-auth
/sp.specify "OAuth2 authentication with Google and GitHub"
# ... develop ...
git commit -am "Add OAuth2 providers"

# Switch to feature 2 (no stashing needed!)
cd ../worktrees/002-dashboard
/sp.specify "Analytics dashboard with real-time charts"
# ... develop ...

# Back to feature 1
cd ../worktrees/001-user-auth
# Your changes are still there, no git checkout needed!
```

## Commands Reference

### `/sp.worktree list`

List all active worktrees:

```bash
/sp.worktree list
```

Output:
```
/path/to/main-repo        (main)
/path/to/worktrees/001-auth  (001-auth)
/path/to/worktrees/002-dashboard  (002-dashboard)
```

### `/sp.worktree create <branch-or-description>`

Create a new worktree:

```bash
# With feature description (generates branch name)
/sp.worktree create "payment processing feature"

# With explicit branch name
/sp.worktree create 001-payment

# With custom path
/sp.worktree create 001-payment /path/to/custom/location
```

### `/sp.worktree remove <path>`

Remove a worktree:

```bash
/sp.worktree remove ../worktrees/001-auth

# With force (if there are uncommitted changes)
git worktree remove --force ../worktrees/001-auth
```

### `/sp.worktree enable`

Enable worktree mode for all future `/sp.specify` commands:

```bash
/sp.worktree enable
```

This sets `SPECIFY_WORKTREE_MODE=true` for the current session.

### `/sp.worktree help`

Show help and usage information:

```bash
/sp.worktree help
```

## How It Works

### Automatic Repo Root Detection

SpecKit Plus scripts automatically detect when you're in a worktree and access `specs/` and `history/` from the main repo root:

**In Bash** (`scripts/bash/common.sh`):
```bash
get_repo_root() {
    if is_worktree; then
        # Return main repo root, not worktree directory
        get_git_common_dir
    else
        # Normal git repo
        git rev-parse --show-toplevel
    fi
}
```

**In PowerShell** (`scripts/powershell/common.ps1`):
```powershell
function Get-RepoRoot {
    if (Test-IsWorktree) {
        # Return main repo root
        return Get-GitCommonDir
    }
    # Normal git repo
    return (git rev-parse --show-toplevel)
}
```

### Branch Detection in /sp.specify

The `/sp.specify` command now detects existing worktree branches:

```bash
# Check current branch
CURRENT_BRANCH=$(git branch --show-current)

# Detect feature branch pattern (001-name or feature-001-name)
if [[ "$CURRENT_BRANCH" =~ ^([0-9]{3})-(.+)$ ]]; then
    # Extract number and name
    # Use existing branch instead of creating new one
fi
```

If you're in a worktree with a feature branch:
- Skips branch creation
- Uses the existing branch name
- Creates spec at the correct location
- Proceeds with spec generation

## Benefits

### 1. Work on Multiple Features Simultaneously

No more `git stash` or losing context when switching between features:

```bash
# Feature 1: OAuth implementation
cd worktrees/001-auth
vim src/auth/oauth.ts

# Feature 2: Dashboard (without stashing!)
cd worktrees/002-dashboard
vim src/dashboard/charts.tsx

# Back to feature 1 - all changes preserved
cd worktrees/001-auth
# oauth.ts still has your changes
```

### 2. Easy Side-by-Side Testing

Test multiple features together:

```bash
# Terminal 1: Run feature 1
cd worktrees/001-auth
npm run dev -- --port 3001

# Terminal 2: Run feature 2
cd worktrees/002-dashboard
npm run dev -- --port 3002

# Compare features running simultaneously
```

### 3. Clean Main Repository

Your main repo stays on the main branch, always ready for hotfixes or production deploys:

```bash
# Main repo always on main
cd main-repo
git status  # → On branch main, working tree clean

# Features in worktrees
cd worktrees/001-auth
git status  # → On branch 001-auth
```

### 4. Shared Specs and History

All features share the same `specs/` and `history/` directories:

```
main-repo/
├── specs/
│   ├── 001-user-auth/spec.md      ← Feature 1 spec
│   └── 002-dashboard/spec.md      ← Feature 2 spec
└── history/
    └── prompts/
        ├── 001-user-auth/         ← Feature 1 history
        └── 002-dashboard/         ← Feature 2 history

# Both accessible from either worktree!
```

## Troubleshooting

### Worktree Already Exists

**Error**: `fatal: '/path/to/worktrees/001-auth' already exists`

**Solution**: The worktree directory already exists. Either:
1. Remove the existing directory: `rm -rf /path/to/worktrees/001-auth`
2. Use a different path: `/sp.worktree create 001-auth /different/path`

### Branch Already Exists

**Error**: `fatal: a branch named '001-auth' already exists`

**Solution**: Create a worktree from the existing branch:
```bash
git worktree add ../worktrees/001-auth 001-auth
```

Or use `/sp.worktree create 001-auth` which handles existing branches automatically.

### Can't Find Specs Directory

**Error**: Specs directory not found when in worktree

**Solution**: Make sure you're using the latest version of SpecKit Plus. The `get_repo_root()` function should automatically detect worktrees and point to the main repo.

Verify with:
```bash
# In worktree
source scripts/bash/common.sh
get_repo_root
# Should output main repo path, not worktree path
```

### Uncommitted Changes When Removing

**Error**: `fatal: 'worktree' contains modified or untracked files`

**Solution**: Either commit your changes or force remove:
```bash
# Option 1: Commit changes
cd worktrees/001-auth
git add .
git commit -m "WIP: Save progress"

# Option 2: Force remove (loses changes!)
git worktree remove --force ../worktrees/001-auth
```

## Advanced Usage

### Custom Worktree Locations

By default, worktrees are created at `../worktrees/<branch-name>`. You can customize:

```bash
# Use different base directory
export WORKTREE_BASE_DIR="/path/to/my-worktrees"
/sp.worktree create "feature"

# Or specify full path
/sp.worktree create 001-feature /custom/path/001-feature
```

### Cleanup Script

Remove all stale worktree references:

```bash
# In main repo
git worktree prune

# List remaining worktrees
git worktree list
```

### Integration with IDEs

Most IDEs work with worktrees automatically:

**VS Code**:
```bash
# Open worktree in VS Code
code ../worktrees/001-auth

# Or use workspaces to manage multiple worktrees
```

**JetBrains IDEs** (IntelliJ, WebStorm, etc.):
- Open each worktree as a separate project
- Share run configurations via version control

## Migration Guide

### Migrating Existing Projects

If you have an existing SpecKit Plus project without worktrees:

```bash
# 1. Ensure you're on main branch with clean working tree
git checkout main
git status  # Should be clean

# 2. Enable worktree mode
/sp.worktree enable

# 3. For each active feature branch, create a worktree
git worktree add ../worktrees/001-auth 001-auth
git worktree add ../worktrees/002-dashboard 002-dashboard

# 4. Continue working in worktrees
cd ../worktrees/001-auth
/sp.specify "Continue user authentication development"
```

### Reverting to Branch-Based Workflow

If you want to go back to the traditional branch-based workflow:

```bash
# 1. Disable worktree mode
unset SPECIFY_WORKTREE_MODE

# 2. Commit and push all changes from worktrees
cd worktrees/001-auth
git push

cd ../002-dashboard
git push

# 3. Remove all worktrees
cd /path/to/main-repo
git worktree remove ../worktrees/001-auth
git worktree remove ../worktrees/002-dashboard

# 4. Use regular git checkout
git checkout 001-auth
# Continue with traditional workflow
```

## Best Practices

### 1. One Worktree Per Feature

Keep worktrees focused on single features:

✅ **Good**:
```
worktrees/001-user-auth
worktrees/002-dashboard
worktrees/003-api-endpoints
```

❌ **Avoid**:
```
worktrees/all-features-combined
```

### 2. Clean Up Merged Worktrees

Remove worktrees after merging:

```bash
# After PR merged
cd main-repo
git worktree remove ../worktrees/001-auth
git branch -d 001-auth  # Delete local branch
git fetch --prune  # Clean up remote references
```

### 3. Use Descriptive Branch Names

Follow SpecKit Plus naming conventions:

✅ **Good**: `001-oauth-implementation`, `002-payment-stripe`
❌ **Avoid**: `001-feature`, `002-stuff`

### 4. Keep Main Repo Clean

Never develop directly in the main repo when using worktrees:

```bash
# Always work in worktrees
cd worktrees/001-auth  # ✅ Good

# Not in main repo
cd main-repo
vim src/auth.ts  # ❌ Avoid
```

## FAQ

**Q: Can I use worktrees with /sp.plan and /sp.implement?**

A: Yes! All SpecKit Plus commands work seamlessly in worktrees. They automatically detect that you're in a worktree and access specs from the main repo.

**Q: Do worktrees work with remote collaboration?**

A: Yes. Worktrees are local-only - your teammates don't need to use them. Each developer can choose their preferred workflow (branches or worktrees).

**Q: Can I have different worktrees on different machines?**

A: Yes. Worktrees are local to each machine. Clone the repo on multiple machines and create different worktrees on each.

**Q: What happens to worktrees when I pull changes?**

A: Worktrees pull changes like normal branches. Run `git pull` in each worktree to update.

**Q: Can I create a worktree from a remote branch?**

A: Yes:
```bash
git worktree add ../worktrees/001-auth origin/001-auth
```

**Q: Do worktrees increase disk usage significantly?**

A: No. Git uses hard links to share objects between worktrees. Only the working directory files are duplicated, not the entire git history.

## See Also

- [Git Worktree Documentation](https://git-scm.com/docs/git-worktree)
- [SpecKit Plus Commands](/templates/commands/)
- [Feature Specification Workflow](/docs-plus/01_getting_started/)

---

**Need Help?**

If you encounter issues with worktrees, try:

1. Check your git version: `git --version` (requires git 2.15+)
2. Verify SpecKit Plus installation: `/sp.worktree help`
3. Review worktree status: `git worktree list`
4. Check common.sh functions: `source scripts/bash/common.sh && get_repo_root`

For bugs or feature requests, please open an issue in the SpecKit Plus repository.
