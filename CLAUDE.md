# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A personal React + Next.js knowledge base. One topic per session, written for the owner to read any time. Topics 00–18 cover React core; 19–25 cover Next.js 14 App Router; 26–29 cover the broader ecosystem. After topic 29, define 5 new topics, append them to `progress.json`, and keep running.

## Session workflow

1. Read `progress.json` to find the next `"status": "pending"` topic.
2. Write `NN-slug/README.md` for that topic following the content rules below.
3. Set its status to `"done"` and increment `current_topic` in `progress.json`.
4. Update the status row in the root `README.md` table (✅ / ⏳).
5. Commit and push immediately after completing the topic.

## Content rules (non-negotiable)

- **What / Why / How structure** — every topic must have all three sections in that order.
- **Real names only** — name every tool, library, and version (e.g. "TanStack Query v5", "Zustand 4.5", "Next.js 14.2"). Never write "some libraries", "certain tools", "various frameworks", or "etc."
- **Real project use cases** — ground every explanation in a concrete scenario (e.g. "a Next.js e-commerce site that needs ISR for product pages") — not abstract generalities.
- **Named comparisons** — every comparison table must list actual named alternatives with versions and a clear trade-off statement.
- **Exact package names** — include the npm package name and version for every library mentioned (e.g. `react-hook-form@7.51`, `@tanstack/react-query@5.28`).

## Git rules

- Use `gh` (GitHub CLI) for all GitHub operations: creating repos, PRs, pushing to remote.
- Commit and push after every completed topic — do not batch.
