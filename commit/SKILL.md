---
name: commit
description: Use this skill when the user asks to commit changes. Triggers on "commit", "commit changes", "/commit". Enforces the project's commit granularity rules — one concept per commit unless changes share the same file.
user_invocable: true
---

# Commit Skill

## Commit Granularity

Each commit should contain a single, minimal concept. Do NOT mix unrelated concepts in the same commit.

**Exception:** if multiple concepts touch the same file, they may be committed together.

When staged changes span multiple unrelated concepts across different files, split them into separate commits.

## Procedure

1. Run `git status` and `git diff` to review all changes.
2. Run `git log --oneline -5` to match the repo's commit message style.
3. Group changes by concept. If unrelated concepts are in separate files, plan multiple commits.
4. For each commit:
   - Stage only the relevant files (`git add <file>...` — never `git add -A` or `git add .`)
   - Write a concise commit message in the repo's style (typically `type: description`)
   - End the message with: `Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>`
   - Use a HEREDOC to pass the commit message
5. Run `git status` after all commits to verify clean state.

## Commit Message Style

Follow the conventional commit format used in this repo:

```
type: short description
```

Common types: `docs`, `feat`, `fix`, `refactor`, `test`, `chore`

## What NOT to Do

- Don't use `git add -A` or `git add .`
- Don't amend existing commits unless explicitly asked
- Don't push unless explicitly asked
- Don't skip hooks (`--no-verify`)
- Don't mix unrelated concepts in one commit (unless they share a file)