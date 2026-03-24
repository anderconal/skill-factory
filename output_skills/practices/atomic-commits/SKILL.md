---
name: atomic-commits
description: Enforces atomic commits and conventional commit format. Apply before every git commit — one logical change, staged with git add -p, typed as feat/fix/refactor.
---

# Conventional Commits

STARTER_CHARACTER = 📝

When starting, announce: "📝 Using CONVENTIONAL-COMMITS skill".

## Before Every Commit

**One logical change per commit.** If staged changes mix concerns (refactor + feature, formatting + logic), split them.

Never use:
- `git commit -am`
- `git add .` or `git add -A` without reviewing what is staged

Stage by hunks, not by file:
```bash
git add -p
```

Review exactly what will be committed before writing the message:
```bash
git diff --staged
```

## Commit Message Format

```
type(scope): short description

Optional body explaining WHY, not what. Blank line separates it from subject.
```

- Subject: lowercase, imperative mood, no period, max 72 characters
- Scope: optional — the module or area affected (`auth`, `orders`, `api`)
- Body: explain the reason. The diff already shows what changed.

## Types

- `feat` — new capability → bumps MINOR in SemVer (1.1.0 → 1.2.0)
- `fix` — bug correction → bumps PATCH (1.1.0 → 1.1.1)
- `refactor` — restructure without changing behavior → no version bump
- `test` — add or correct tests → no version bump
- `docs` — documentation only → no version bump
- `chore` — tooling, dependencies, build config → no version bump
- `style` — formatting, whitespace → no version bump
- `perf` — performance improvement → bumps PATCH

Breaking changes: add `!` after the type (`feat!:`, `fix!:`) or include `BREAKING CHANGE:` in the footer → bumps MAJOR (1.1.0 → 2.0.0).

## Examples

```
feat(auth): add JWT refresh token rotation
fix(orders): prevent duplicate charge on retry
refactor(user): extract validation into UserValidator
chore: update dotnet SDK to 8.0.4
test(payment): add characterization tests for LegacyGateway
feat!: remove support for API v1
```

## Before Pushing a Branch

Clean local history before merge/PR. Merge "wip" and correction commits into their parent:

```bash
git rebase -i origin/main
```

- `fixup` — merge commit into parent, discard its message
- `squash` — merge commit into parent, combine messages
- `reword` — fix message format without changing the change

Never rebase commits already pushed to a shared branch.

## Where This Leads

A history of atomic semantic commits enables full automation: CI reads `feat:` and bumps minor version, generates CHANGELOG, tags and publishes the release — with no human deciding version numbers or writing release notes. This is the direction tools like semantic-release and release-please take you.

## Credits

Conventional Commits spec: conventionalcommits.org
Semantic Versioning: semver.org — Tom Preston-Werner
Interactive Rebase discipline: Dave Farley, *Continuous Delivery* (Addison-Wesley, 2010)
