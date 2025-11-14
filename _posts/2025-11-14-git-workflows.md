---
layout: default
title:  "Git Workflows That Don't Drive You Crazy"
date:   2025-11-14 15:00:00
categories: Development Git Workflow
---

I've worked with teams using every Git workflow imaginable. Feature branches that live for months. Rebasing nightmares. Merge conflicts that made developers weep. And workflows so simple they were actually elegant.

Here's what I've learned about Git workflows that actually work without making everyone miserable.

## The Two Main Approaches

In 2025, most teams use one of two strategies:

**Trunk-Based Development**: Everyone works on main (or trunk), committing small changes frequently.

**GitFlow**: Separate branches for features, releases, and hotfixes with a structured merging process.

There's no "best" workflow—only what works for your team. Let me explain both and when to use each.

## Trunk-Based Development (What I Recommend for Most Teams)

### The Core Idea

Everyone commits to `main` (the trunk) frequently—ideally multiple times per day. Feature branches are short-lived (hours or days, not weeks).

```
main: ---*---*---*---*---*---*---*---*--->
           ↑   ↑   ↑   ↑   ↑   ↑   ↑
        Dev1 Dev2 Dev1 Dev3 Dev2 Dev1 Dev3
```

### Why It Works

**Reduces merge conflicts**: Small, frequent merges mean conflicts are tiny and easy to resolve.

**Encourages CI/CD**: You can't have real continuous integration without trunk-based development.

**Faster feedback**: Code gets to main quickly, so issues are caught early.

**Simpler**: No complex branching strategies to remember.

### How to Do It

**1. Work in small increments**

```bash
# Start work
git pull origin main

# Make small change (< 1 day of work)
# ... edit files ...

# Commit and push frequently
git add .
git commit -m "Add user validation for email field"
git push origin main
```

**2. Use feature flags for incomplete features**

```javascript
// Feature not ready for users, but code is in main
if (featureFlags.isEnabled('new-checkout-flow')) {
    return <NewCheckout />;
}
return <OldCheckout />;
```

**3. Short-lived feature branches (optional)**

For code review, create short branches that live < 24 hours:

```bash
# Create feature branch
git checkout -b fix/user-validation

# Make changes, commit
git add .
git commit -m "Fix email validation bug"

# Push and create PR
git push origin fix/user-validation

# Get review, merge, delete branch (same day!)
```

### When Trunk-Based Development Works Best

- Teams practicing CI/CD
- Fast-paced development with frequent releases
- Experienced teams comfortable with small commits
- Products that don't need to support multiple versions

**2025 data**: Teams using trunk-based development report 28% faster project delivery times.

## GitFlow (For Structured Releases)

### The Core Idea

Multiple long-lived branches with specific purposes:

- `main` (or `master`): Production-ready code
- `develop`: Integration branch for features
- `feature/*`: Individual features
- `release/*`: Preparing releases
- `hotfix/*`: Emergency production fixes

```
main:     ---*-----------*-----------*--->
               ↖       ↗   ↖       ↗
release:         *---*       *---*
                   ↖       ↗
develop:  ---*---*---*---*---*---*---*--->
           ↗ ↖ ↗ ↖ ↗ ↖
feature:  *   *   *   *
```

### The Workflow

**1. Start a feature**

```bash
# Create feature branch from develop
git checkout develop
git pull origin develop
git checkout -b feature/user-profiles

# Work on feature
# ... make changes ...
git add .
git commit -m "Add user profile page"
git push origin feature/user-profiles
```

**2. Finish a feature**

```bash
# Merge feature into develop
git checkout develop
git merge feature/user-profiles
git push origin develop

# Delete feature branch
git branch -d feature/user-profiles
git push origin --delete feature/user-profiles
```

**3. Create a release**

```bash
# Create release branch from develop
git checkout develop
git checkout -b release/1.2.0

# Fix bugs, update version numbers
git commit -am "Bump version to 1.2.0"

# Merge to both main and develop
git checkout main
git merge release/1.2.0
git tag -a v1.2.0 -m "Version 1.2.0"
git push origin main --tags

git checkout develop
git merge release/1.2.0
git push origin develop

# Delete release branch
git branch -d release/1.2.0
```

**4. Hotfix production**

```bash
# Create hotfix from main
git checkout main
git checkout -b hotfix/critical-bug

# Fix the bug
git commit -am "Fix critical security issue"

# Merge to both main and develop
git checkout main
git merge hotfix/critical-bug
git tag -a v1.2.1 -m "Hotfix 1.2.1"
git push origin main --tags

git checkout develop
git merge hotfix/critical-bug
git push origin develop

# Delete hotfix branch
git branch -d hotfix/critical-bug
```

### When GitFlow Works Best

- Products with scheduled releases (quarterly, monthly)
- Multiple versions in production (SaaS with version support)
- Teams that need strict separation between development and production
- Projects where breaking changes need careful management

## The Hybrid Approach (What I Actually Use)

Most teams don't need full GitFlow's complexity, but pure trunk-based can feel risky. Here's a middle ground:

### The Strategy

- `main`: Always deployable production code
- Short feature branches (< 3 days)
- Direct hotfixes to main
- Feature flags for work-in-progress

```bash
# Feature branch for code review
git checkout -b feature/add-search
# ... work ...
git push origin feature/add-search
# Create PR, get review, merge

# Hotfix goes straight to main
git checkout main
git pull
# ... fix ...
git commit -m "Fix search crash"
git push origin main
```

### The Rules

1. **Feature branches live < 3 days** - If it's taking longer, break it down
2. **Main is always green** - CI must pass before merging
3. **Deploy from main frequently** - At least daily
4. **Use feature flags** - Hide incomplete features
5. **No develop branch** - It just adds complexity

This gives you:
- Code review (via PRs from feature branches)
- Fast integration (short-lived branches)
- Safety (main stays stable)
- Simplicity (no complex branching model)

## Branch Naming Conventions

Whatever workflow you choose, consistent naming helps:

```
feature/user-authentication
feature/add-payment-gateway

fix/memory-leak-in-parser
fix/broken-login-redirect

hotfix/security-vulnerability
hotfix/production-crash

docs/api-documentation
docs/deployment-guide

refactor/database-queries
refactor/auth-module

chore/update-dependencies
chore/configure-linter
```

## Commit Message Best Practices

Good commits make history useful:

**Bad commits:**
```
git commit -m "fix"
git commit -m "updates"
git commit -m "wip"
git commit -m "forgot to add file"
```

**Good commits:**
```
git commit -m "Fix email validation to allow plus signs"
git commit -m "Add user authentication with JWT"
git commit -m "Refactor database queries to use prepared statements"
git commit -m "Update React to v18.2.0"
```

**The format I use:**
```
<type>: <subject>

<body>

<footer>
```

**Example:**
```
feat: Add password reset functionality

Implement password reset flow with email verification.
Users can request reset link via email, which expires
after 1 hour.

Closes #234
```

**Types:**
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation
- `refactor`: Code restructuring
- `test`: Adding tests
- `chore`: Maintenance

## Handling Merge Conflicts

Conflicts are inevitable. Here's how to not hate them:

### Prevention

**1. Pull frequently**
```bash
# Before starting work
git pull origin main

# Every few hours
git pull origin main
```

**2. Keep changes small**
Smaller changes = fewer conflicts

**3. Communicate**
"Hey, I'm refactoring the auth module today" prevents two people changing the same code.

### Resolution

When you hit a conflict:

```bash
git pull origin main
# Auto-merging src/auth.js
# CONFLICT (content): Merge conflict in src/auth.js

# Open the file, look for conflict markers:
<<<<<<< HEAD
function login(email, password) {
    return authenticate(email, password);
}
=======
async function login(email, password) {
    const user = await authenticate(email, password);
    return user;
}
>>>>>>> main

# Resolve by picking one or combining
async function login(email, password) {
    const user = await authenticate(email, password);
    return user;
}

# Mark as resolved
git add src/auth.js
git commit -m "Merge main and resolve auth.js conflict"
```

**Tools that help:**
- VSCode built-in merge editor
- `git mergetool` with diff tools
- GitHub/GitLab conflict resolution UI

## CI/CD Integration

Your Git workflow should integrate with CI/CD:

**On every PR/push to main:**
```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run tests
        run: npm test
      - name: Run linter
        run: npm run lint
      - name: Check types
        run: npm run type-check
```

**Branch protection rules:**
- Require PR reviews (at least 1)
- Require CI to pass
- Require branches to be up-to-date before merging
- No direct pushes to main (except for emergency hotfixes)

## Common Mistakes

### Mistake 1: Long-Lived Feature Branches

**Problem**: Feature branch lives for weeks or months

**Result**: Massive merge conflicts, branch drifts from main, integration problems

**Fix**: Break work into smaller pieces, merge frequently

### Mistake 2: Not Pulling Before Starting Work

**Problem**: Start working on old code

**Result**: Conflicts guaranteed

**Fix**: Always `git pull` before starting

### Mistake 3: Force-Pushing to Shared Branches

**Problem**: `git push --force` on a branch others are using

**Result**: Everyone's local branches are now broken

**Fix**: Never force-push shared branches. Only force-push your own feature branches if you must.

### Mistake 4: Merge vs. Rebase Confusion

**When to merge:**
- Integrating a feature branch into main
- You want to preserve history

**When to rebase:**
- Updating your feature branch with changes from main
- You want clean, linear history

```bash
# Update feature branch with main changes
git checkout feature/my-feature
git rebase main  # Replays your commits on top of main

# Integrate feature into main
git checkout main
git merge feature/my-feature  # Preserves feature branch history
```

### Mistake 5: No Branch Cleanup

**Problem**: Dozens of merged branches still in the repo

**Result**: Clutter, confusion

**Fix**: Delete branches after merging

```bash
# Delete local branch
git branch -d feature/old-feature

# Delete remote branch
git push origin --delete feature/old-feature

# Prune deleted remote branches from local
git fetch --prune
```

## 2025 Trends

### AI-Enhanced Git

Tools now offer:
- AI-powered code reviews on PRs
- Automated conflict resolution suggestions
- Predictive merge conflict detection

### Automated Branch Management

Some teams use bots to:
- Auto-delete merged branches
- Auto-rebase feature branches
- Auto-merge when CI passes and reviews approve

### Better Integration with Project Management

Git commits/PRs now auto-update:
- Jira tickets
- Linear issues
- GitHub Projects

## My Workflow (Hybrid, Simplified)

Here's what I actually do:

```bash
# Daily routine
git checkout main
git pull origin main

# Small change (< 4 hours)
# ... make changes ...
git add .
git commit -m "Descriptive message"
git push origin main

# Larger change (needs review)
git checkout -b feature/new-thing
# ... make changes ...
git commit -m "Descriptive message"
git push origin feature/new-thing
# Create PR, get review, merge, delete branch

# Weekly cleanup
git fetch --prune
git branch -vv | grep ': gone]' | awk '{print $1}' | xargs git branch -D
```

**Principles:**
1. Main is always deployable
2. Merge to main at least daily
3. Feature branches live < 3 days
4. Use feature flags for WIP features
5. CI must pass before merging

This gives me the benefits of trunk-based development (fast integration, simple workflow) with the safety of code review.

## Sources and Further Reading

- [Trunk-based Development - Atlassian](https://www.atlassian.com/continuous-delivery/continuous-integration/trunk-based-development)
- [Gitflow Workflow - Atlassian](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow)
- [Git Workflows Guide 2025 - Peerlist](https://peerlist.io/teamcamp/articles/mastering-git-workflows-efficient-collaboration-guide-2025)
- [Trunk-Based Development vs. Git Flow - Toptal](https://www.toptal.com/software/trunk-based-development-git-flow)
- [GitFlow vs. Trunk-Based Development - Medium (Oct 2025)](https://medium.com/girishmk/gitflow-vs-trunk-based-development-a-comparative-study-of-branching-strategies-9835ceef377a)
- [Choosing the Right Git Branching Strategy - Medium](https://medium.com/@sreekanth.thummala/choosing-the-right-git-branching-strategy-a-comparative-analysis-f5e635443423)

---

*Questions about Git workflows? [Let me know](mailto:jordan@jordananderson.us).*
