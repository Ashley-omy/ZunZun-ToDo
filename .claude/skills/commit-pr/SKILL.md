---
name: commit-pr
description: Create commits and pull requests for ZunZun ToDo following project conventions. Invoke explicitly with /commit-pr.
disable-model-invocation: true
---

# Commits and Pull Requests

## Current state

!`git status --short`

!`git branch --show-current`

## Steps

### 1. Pre-checks (required)

Tests must pass before committing.

```bash
cd backend && ./mvnw verify                         # if backend changed
cd frontend && npm run typecheck && npm run lint    # if frontend changed
```

**Never merge to main on red.** On a feature branch, a work-in-progress commit at the end of a
work block is fine (prefix it with `wip(scope):`) so the work is backed up — but it must be
green before the PR is merged.

### 2. Secret scan

Review `git diff` and confirm none of the following are present:

- Passwords, API keys, tokens
- Real `.env` or `.env.local` files
- AWS access keys
- Production database credentials

Once committed, secrets live in history forever — **check before pushing.**

### 3. Branches

Never commit directly to main. One branch per change.

```
feature/xxx    new functionality
fix/xxx        bug fix
docs/xxx       documentation only
chore/xxx      config, dependencies
```

### 4. Commit messages

Conventional Commits. One logical change per commit.

```
<type>(<scope>): <short summary>

<body explaining WHY, when it isn't obvious>
```

Types: `feat` `fix` `docs` `test` `refactor` `chore`
Scopes: `auth` `user` `project` `task` `progress` `stats` `social` `ui` `infra` `docs`

Examples:

```
feat(progress): add progress record endpoints

The one-completion-per-task rule is enforced by a partial unique
index rather than an application check, because an application-level
check races under concurrent requests.
```

```
fix(stats): weekly chart started on Sunday instead of Monday
```

**Explain why.** The diff already shows what changed.

### 5. Pull request

```bash
git push -u origin <branch>
gh pr create --title "<title>" --body "..."
```

PR body template:

```markdown
## Summary
What this PR accomplishes (1–3 lines)

## Changes
- Bullet list of changes

## Verification
- [ ] Verified locally (describe the steps)
- [ ] Tests pass

## References
- SDS: §x.y
- Issue: #n
```

### 6. Merging

- Merge only once CI is green
- Even working solo, open a PR and merge it yourself — history stays readable in feature-sized units

## Milestone tags

Tag the plan's milestones as you reach them.

```bash
git tag m4-core
git push origin m4-core
```

| Tag | Meaning |
|---|---|
| `m1-setup` | Backend and frontend run locally and talk to each other (Day 1) |
| `m2-auth` | Password and Google sign-in work end to end (Day 3) |
| `m3-calendar` | Projects, tasks, and calendar records work (Day 7) |
| `m4-core` | Progress charts complete — core loop done, most important (Day 8) |
| `m5-social` | Profiles, follow, feed, reactions, comments (Day 11) |
| `v1.0.0` | Production release on Vercel and AWS (Day 14) |

## Checklist

- [ ] Tests green (or clearly marked `wip` on a feature branch)
- [ ] `docs/dev-log.md` updated with what was done and the next step
- [ ] No secrets in the diff
- [ ] On a feature branch, not main
- [ ] Message follows Conventional Commits
- [ ] One logical change per commit
