---
name: buildproject
description: Use this skill when the user asks to build the project, implement the checklist, or continue building. Triggers on "build", "build project", "implement checklist", or "/buildproject".
user-invocable: true
---

# Build Project

Before starting, commit any uncommitted changes using the `/commit` skill.

Follow these three steps in order.

## Step 1 — Update BUILD_CHECKLIST.md

Read every spec doc in the project root and sync the checklist so it fully reflects them.

**Rules:**
- Add checklist items for any requirement in the docs that is not yet represented.
- Remove or update items that no longer match the docs.
- Preserve existing checked/unchecked state — never uncheck a completed item.
- Keep the existing phase structure; add new phases only when a requirement doesn't fit an existing one.

## Step 2 — Implement the Checklist

Work through every unchecked `[ ]` item in `BUILD_CHECKLIST.md`:

1. Implement the feature or fix.
2. Mark it `[x]` in `BUILD_CHECKLIST.md` when done.
3. Follow existing project skills for the relevant domain.

## Step 3 — Commit

After all steps are complete, commit all changes using the `/commit` skill.
