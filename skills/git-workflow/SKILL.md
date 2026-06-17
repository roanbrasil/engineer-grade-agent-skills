---
name: git-workflow
description: Expert Git workflow guidance — branching strategies, commit discipline, PR practices, merge strategies, and advanced Git operations for production teams.
---

# Git Workflow — Expert Reference

## Branching Strategies

### Decision Matrix

```
Team size / CI maturity →  Low             Medium           High
                        ┌──────────────┬────────────────┬─────────────────┐
Release cadence         │              │                │                 │
  Continuous deploy  →  │  GitHub Flow │  GitHub Flow   │  Trunk-Based    │
  Weekly/sprint     →  │  Gitflow     │  Gitflow       │  Trunk-Based    │
  Quarterly/versioned→  │  Gitflow     │  Gitflow       │  Gitflow        │
                        └──────────────┴────────────────┴─────────────────┘
```

### Trunk-Based Development (recommended default)

Everyone integrates to `main` at least once per day. Feature flags gate incomplete work. Short-lived branches (< 2 days) acceptable.

```
main ──────────────────────────────────────────────►
      ↑     ↑     ↑     ↑     ↑     ↑     ↑
      |     |     |     |     |     |     |
   feat/A  fix/B  chore feat/C fix/D feat/E  ...
  (hours) (hours)(min) (hours)(min)(hours)
```

**Why trunk-based wins with CI/CD:**
- Eliminates merge hell from long-lived branches
- Forces integration continuously — you find conflicts when they're small
- Feature flags decouple deploy from release
- Smaller diffs = faster, higher-quality code review
- Supports rollback via flag toggle, not git revert

**Prerequisites:**
- Robust CI gate (tests must pass before merge)
- Feature flag infrastructure (LaunchDarkly, Unleash, env vars)
- Culture of small, complete commits

### GitHub Flow

```
main ─────────────────────────────────────────────►
       \                              /
        feat/user-auth ──────────────
         (PR review + CI gate)
```

Single long-lived branch (`main`). Feature branches merged via PR. Deploy from `main`. Good for teams not ready for trunk-based — simpler than Gitflow.

### Gitflow

```
main     ──────────────────────────────────────────►
           ↑ tag v1.0          ↑ tag v1.1
develop  ──┴──────────────────┴──────────────────►
              \  feat/A  /  \  feat/B  /
hotfix   ────────────────────────────────── (rare)
release  ──────────── (stabilization) ─────────────
```

Use when: shipping versioned software (libraries, mobile apps, firmware), multiple maintained versions in production simultaneously, external customer releases on a cadence.

**Anti-pattern**: Using Gitflow for a web app that deploys daily. It is overhead without benefit.

---

## Branch Naming

```
<type>/<short-description>
<type>/<ticket-id>-<short-description>
```

| Prefix | When to use |
|--------|-------------|
| `feat/` | New capability |
| `fix/` | Bug fix |
| `chore/` | Tooling, deps, config (no prod behavior change) |
| `refactor/` | Behavior-preserving code restructure |
| `docs/` | Documentation only |
| `ci/` | CI/CD pipeline changes |
| `test/` | Test additions or fixes |
| `perf/` | Performance improvements |

**Examples:**
```
feat/user-authentication
fix/JIRA-1234-null-pointer-on-checkout
chore/upgrade-spring-boot-3.2
refactor/extract-payment-service
```

**Rules:**
- Lowercase, hyphens only (no underscores, no slashes except the prefix)
- Describe the change, not the person (`feat/auth`, not `johns-auth-work`)
- Ticket ID is useful but not mandatory if the branch name is self-documenting

---

## Conventional Commits

### Format

```
<type>(<scope>): <imperative-mood subject, max 72 chars>
<blank line>
<body: WHY this change, not what — wrap at 72 chars>
<blank line>
<footer: BREAKING CHANGE, Fixes #, Closes #, Co-authored-by>
```

### Types

| Type | Description | SemVer bump |
|------|-------------|-------------|
| `feat` | New feature | MINOR |
| `fix` | Bug fix | PATCH |
| `perf` | Performance improvement | PATCH |
| `refactor` | No behavior change | none |
| `test` | Tests only | none |
| `docs` | Documentation only | none |
| `chore` | Build, deps, tooling | none |
| `ci` | CI configuration | none |
| `build` | Build system | none |

### Breaking Changes

Breaking change in footer (triggers MAJOR bump in semver tools):
```
feat(auth): replace session cookies with JWT

Stateless tokens simplify horizontal scaling. All clients must update
to send Authorization: Bearer <token> header instead of relying on
cookies. The /api/v1/session endpoint is removed.

BREAKING CHANGE: Cookie-based session authentication removed. Clients
must migrate to Bearer token auth before upgrading to this version.
Closes #442
```

### Subject Line Discipline

```
# GOOD — imperative mood, specific, under 72 chars
feat(checkout): add idempotency key support for payment API

# BAD — past tense
feat(checkout): added idempotency key support

# BAD — present continuous
feat(checkout): adding idempotency key support

# BAD — describes WHAT not WHY (reserve that for body)
fix(auth): change token expiry from 7d to 1h

# GOOD — with WHY in subject
fix(auth): shorten token expiry to 1h to limit breach window
```

### Commit Body — WHY Not WHAT

```
fix(payments): retry on 429 before propagating error

Stripe's API enforces rate limits during Black Friday traffic spikes.
Previously we surfaced a 429 immediately as a checkout failure. Now
we retry up to 3 times with exponential backoff (1s, 2s, 4s) before
failing. This matches Stripe's recommended integration pattern.

Fixes #891
```

---

## PR Discipline

### Size

- Target < 400 lines changed (excluding generated code, migrations, lock files)
- Single responsibility: one logical change per PR
- If you need to refactor to land a feature, land the refactor first in a separate PR

### Self-Review Checklist (do before requesting review)

```
[ ] Read every line of the diff as if you're the reviewer
[ ] No debug code, no commented-out code, no TODOs without tickets
[ ] New behavior has tests; deleted behavior has deleted tests
[ ] PR description explains WHY the change is needed, not just WHAT
[ ] Link to ticket, design doc, or Slack thread if context is non-obvious
[ ] All CI checks pass locally before pushing
[ ] Screenshots/recordings for UI changes
```

### PR Description Template

```markdown
## Why

<!-- What problem does this solve? Link to ticket if applicable. -->

## What Changed

<!-- High-level summary. What's the approach? Why this approach? -->

## How to Test

<!-- Concrete steps. What should the reviewer exercise? -->

## Risks

<!-- What could go wrong? Any backward compatibility concerns? -->
```

### Reviewer Expectations

- Respond within one business day
- Block on correctness, security, and architectural concerns
- Comment (non-blocking) on style; don't require style changes unless they affect readability significantly
- Approve after concerns are addressed, not just acknowledged

---

## Merge Strategies

```
Strategy        History          When to use
─────────────────────────────────────────────────────────
Merge commit    Preserves all    Never: noisy, harder to bisect
                branch commits
                + adds a
                merge commit

Squash merge    All branch       Short-lived feature branches
                commits → 1      where WIP commits are noise;
                commit on main   trunk-based development

Rebase merge    Branch commits   When each commit on the branch
                replayed onto    is meaningful and you want linear
                main (no merge   history; requires clean commits
                commit)
```

**Recommendation for trunk-based:** Squash merge. One logical change = one commit on `main`. This makes `git bisect` and `git log` useful.

**Anti-pattern**: Merge commits on main from every PR. `git log --oneline` becomes unreadable.

---

## Feature Flags vs Feature Branches

Feature flags enable trunk-based development:

```python
# Python — LaunchDarkly or simple env var flag
def create_order(cart):
    if feature_enabled("new-checkout-flow", user):
        return new_checkout_service.process(cart)
    return legacy_checkout_service.process(cart)
```

```java
// Java — feature flag via environment variable
if (featureFlags.isEnabled("new-checkout-flow", userId)) {
    return newCheckoutService.process(cart);
}
return legacyCheckoutService.process(cart);
```

**Flag lifecycle:**
1. Create flag, default OFF in all environments
2. Merge code behind flag (flag is OFF, production unaffected)
3. Enable in staging, test
4. Gradual rollout in production (1% → 10% → 100%)
5. Remove flag and dead code path (this step is mandatory — flag debt kills you)

**Types of flags:**
| Type | Lifetime | Example |
|------|----------|---------|
| Release flag | Days-weeks | New checkout flow |
| Experiment | Weeks | A/B test on CTA copy |
| Ops/kill switch | Permanent | Disable expensive feature under load |
| Permission | Permanent | Beta features for paying tiers |

---

## Git Bisect

Binary search through commit history to find the commit that introduced a bug.

```bash
# Start bisect
git bisect start

# Mark current commit as bad
git bisect bad

# Mark last known good commit (tag, SHA, or relative ref)
git bisect good v2.3.1

# Git checks out midpoint — test manually or run a command
git bisect run pytest tests/test_checkout.py

# Git reports the first bad commit
# b3a4f91 is the first bad commit
# commit b3a4f91...

# Always clean up
git bisect reset
```

**With automated test:**
```bash
# bisect run exits 0 = good, non-zero = bad
git bisect run ./scripts/check_regression.sh
```

---

## Interactive Rebase

Clean up WIP commits before merging. Never rebase commits already pushed to shared branches.

```bash
# Rebase last 5 commits interactively
git rebase -i HEAD~5

# In the editor:
pick a1b2c3 feat(auth): add JWT validation
squash d4e5f6 wip: fix test
squash g7h8i9 wip: actually fix the test
pick j1k2l3 test(auth): add JWT expiry edge case
reword m4n5o6 fix typo

# Commands:
# pick   = keep commit as-is
# squash = melt into previous commit, merge commit messages
# fixup  = melt into previous, discard this commit message
# reword = keep commit but edit message
# drop   = delete commit entirely
# edit   = pause to amend the commit
```

**Workflow for clean history:**
```bash
# During feature work — commit freely with WIP messages
git commit -m "wip: getting auth to work"
git commit -m "wip: tests failing, investigate tomorrow"
git commit -m "fix: found the issue, token was base64 encoded twice"

# Before merging — clean up
git rebase -i origin/main

# Result: one clean commit lands on main
# feat(auth): add JWT validation with proper base64 handling
```

---

## Stash Discipline

```bash
# GOOD — always name your stash
git stash push -m "WIP: half-done payment refactor, needs error handling"

# GOOD — list with descriptions
git stash list
# stash@{0}: On feat/payments: WIP: half-done payment refactor...
# stash@{1}: On main: quick spike on auth approach

# GOOD — apply specific stash
git stash pop stash@{0}   # apply and remove
git stash apply stash@{1} # apply, keep in list

# GOOD — drop a stash you no longer need
git stash drop stash@{1}

# ANTI-PATTERN: anonymous stash accumulation
git stash  # no message — forget what it was in 2 days
```

**Rule:** Stashes older than 3 days are either committed, branched, or dropped. Stash is not a parking lot.

---

## Cherry-Pick

Use sparingly. Justified for backporting a fix to a release branch.

```bash
# Find the commit SHA
git log --oneline main | grep "fix(payments)"
# a1b2c3f fix(payments): handle Stripe timeout correctly

# Apply to release branch
git checkout release/v2.3
git cherry-pick a1b2c3f

# If conflicts:
git cherry-pick --continue  # after resolving
git cherry-pick --abort     # to cancel
```

**When cherry-pick is appropriate:**
- Backporting a security fix to an older release branch
- Porting a hotfix from `main` to `release/v1.x`

**When it's a smell:**
- Regularly cherry-picking between feature branches → your branch strategy is broken
- Cherry-picking across unrelated codebases → extract the logic into a shared library

---

## Revert vs Reset

```
Situation                           Command
──────────────────────────────────────────────────────────────
Undo a commit on a shared branch    git revert <sha>
(creates a new "undo" commit,
preserves history)

Undo your local commits not yet     git reset HEAD~2  (keep files)
pushed (move HEAD pointer,          git reset --hard HEAD~2  (discard)
history is rewritten)

Remove a file from staging          git restore --staged <file>

Discard file changes in working     git restore <file>
tree
```

```bash
# SAFE — revert commit abc123 on main (creates inverse commit)
git revert abc123
git push origin main

# DANGEROUS on shared branches — rewrites history
git reset --hard abc123
git push --force origin main  # breaks everyone else's clone
```

**Rule:** Never `reset` + `force push` to `main`. Use `revert` on any branch others have pulled.

---

## Tags and Releases

```bash
# Annotated tag (always use annotated — they carry metadata)
git tag -a v1.4.2 -m "Release v1.4.2: fix Stripe timeout handling"

# Tag a specific commit
git tag -a v1.4.1 a1b2c3f -m "Release v1.4.1"

# Push tags (not pushed by default)
git push origin --tags
git push origin v1.4.2  # single tag

# Semantic versioning via git describe
git describe --tags --always
# v1.4.1-23-ga1b2c3f  (23 commits since v1.4.1, current SHA a1b2c3f)
```

### Semantic Versioning

```
MAJOR.MINOR.PATCH[-prerelease][+build]
  │      │     │
  │      │     └── Backward-compatible bug fix
  │      └──────── Backward-compatible new feature
  └─────────────── Breaking change

Examples:
  1.0.0         Initial stable release
  1.1.0         New feature added
  1.1.1         Bug fix
  2.0.0-rc.1    Release candidate for breaking change
  2.0.0         Breaking change shipped
```

---

## Monorepo Considerations

### Sparse Checkout (only check out what you need)

```bash
# Initialize sparse checkout
git sparse-checkout init --cone
git sparse-checkout set services/payment-service shared/auth-lib

# Check out only those directories
git pull
```

### Path Filters in CI

```yaml
# GitHub Actions — only run payment service CI when its code changes
jobs:
  payment-service:
    if: |
      contains(github.event.head_commit.modified, 'services/payment/')
      || contains(github.event.head_commit.modified, 'shared/auth-lib/')
    steps:
      - uses: actions/checkout@v4
      - run: cd services/payment-service && ./gradlew test
```

### Commit Scope in Monorepo

```
feat(payment-service): add idempotency keys
fix(auth-lib): handle clock skew in JWT validation
chore(ci): add path filter for shared libs
```

---

## Anti-Patterns Catalog

| Anti-pattern | Why it hurts | Fix |
|---|---|---|
| Long-lived feature branches | Merge conflicts compound daily; integration risk grows | Trunk-based + feature flags |
| Force-push to main | Rewrites history for everyone, loses work | `git revert`; protect main branch |
| "WIP" commits on main | Bisect finds WIP as first bad commit; CI may fail | Interactive rebase before merge |
| Huge commits (1000+ lines) | Impossible to review meaningfully; hard to bisect | Single responsibility commits |
| Committing secrets | Permanent in history even after revert | Pre-commit hook with gitleaks/trufflehog |
| No commit messages ("fix", "update") | `git log` becomes useless | Conventional Commits; PR template |
| Skipping CI with `--no-verify` | Bypasses safety net | Fix the hook; never skip it |
| Branching off a feature branch | Cascading rebase pain | Always branch off `main`/`develop` |
| Stale stashes | Forgotten work; shadow of a branch | Weekly stash hygiene |
| Merge commits on every PR | History becomes a tree, not a line; bisect harder | Squash merge policy |

---

## Quick Reference Cheatsheet

```bash
# Start work
git switch -c feat/my-feature origin/main

# Clean commit
git add -p                          # patch-stage: review each hunk
git commit -m "feat(scope): subject"

# Sync with main during long work
git fetch origin
git rebase origin/main              # keeps linear history

# Clean up before PR
git rebase -i origin/main           # squash WIPs

# Undo last commit (keep changes)
git reset HEAD~1

# Find who introduced a bug
git bisect start && git bisect bad && git bisect good v1.3.0
git bisect run pytest tests/

# Backport fix
git cherry-pick <sha>

# Safe undo on shared branch
git revert <sha>

# Stash with name
git stash push -m "description"
git stash pop
```
