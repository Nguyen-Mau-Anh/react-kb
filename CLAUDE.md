# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A personal React + Next.js knowledge base. One topic per session. Structure mirrors `springboot-kb`: **group folders by theme, multiple `.md` files per folder**.

```
react-kb/
├── 00.meta/roadmap.md        ← single source of truth for what's done / pending
├── 01.fundamentals/          ← group of related topic files
│   ├── 01-how-react-works.md
│   ├── 02-jsx-and-rendering.md
│   └── 03-components-props.md
├── 02.hooks/
│   ├── 01-usestate.md
│   └── ...
└── README.md                 ← human-friendly index linking into groups
```

## Session workflow

1. Open `00.meta/roadmap.md`. Find the next `[ ]` unchecked box.
2. Write the file at the path it specifies (e.g. `02.hooks/04-usememo-usecallback.md`).
3. Flip the box to `[x]` in the roadmap.
4. Update the matching row in `README.md` from ⏳ to ✅.
5. Commit and push immediately. One topic = one commit.

## When the roadmap is exhausted

Append a new "Group N" section to `00.meta/roadmap.md` with five new `[ ]` topics, plus matching folder if needed. Then continue.

## Content rules (non-negotiable)

- **What / Why / How** structure — every file must have all three sections in that order.
- **Real names only** — name every tool, library, and exact version (e.g. `react-hook-form@7.51`, `@tanstack/react-query@5.28`, `next@14.2`). Never write "some libraries", "certain tools", "various frameworks", or "etc."
- **Real project use cases** — ground every explanation in a concrete scenario (e.g. "a Next.js e-commerce site that needs ISR for product pages") — not abstract generalities.
- **Named comparisons** — every comparison table must list actual named alternatives with versions and a clear trade-off statement.
- **Exact npm package names** — include the npm package name and version every time a library is mentioned.
- **Official docs only** in further reading — react.dev, nextjs.org, the library's own README. Not random Medium posts.

## Git rules

- Use `gh` (GitHub CLI) for all GitHub operations: creating repos, PRs, pushing.
- Commit and push after every completed topic — do not batch.
- One commit per topic file.
