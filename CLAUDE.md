# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Hard Rules

- Never generate spec files — `skipTests: true` is set globally in `angular.json`.
- Never write to `context/archive/` — archived changes are immutable; abort with "This change is archived" if a target path starts there.
- Destructive production actions (drop DB, rotate secrets, delete a Workers project) are human-only — suggest but do not execute.
- All repository files must be written in English — no Polish in any file content, comments, or documentation. Polish is allowed only in chat responses.

## Project: MarketPulse

Stock market alert web app. Users set price or RSI-based alerts on VIX and NASDAQ-100 indices and receive email notifications when thresholds are crossed. Market data is fetched once daily from Stooq via a cron job; RSI is calculated server-side from recent daily closes.

## Architecture

Split deployment — two separate services:

- **Frontend**: Angular 22 SPA → Cloudflare Pages (`src/`)
- **Backend**: Cloudflare Workers (Hono) + D1 (SQLite) + Cron Triggers + Resend (not yet scaffolded)

Key flows:
1. **Daily cron** (Cloudflare Cron Trigger): fetches closes from Stooq → calculates RSI → evaluates all active alerts → sends email via Resend for each triggered alert → records trigger events in D1.
2. **Alert CRUD**: Angular SPA calls Worker endpoints; D1 is the source of truth.
3. **Auth**: username + password, flat role model — each user manages only their own alerts.

## Commands

```bash
npm start           # dev server at http://localhost:4200 (live reload)
npm run build       # production build → dist/
npm run watch       # incremental dev build
npm test            # Karma unit tests
ng generate component path/to/name   # scaffold a component (skipTests is on by default)
```

## Angular Conventions

From `angular.json` schematics config:
- Standalone components — no NgModule.
- Styles: SCSS. File naming: `name.ts` / `name.html` / `name.scss`.

## Formatting

See `@.prettierrc`.

<!-- BEGIN @przeprogramowani/10x-cli -->

## 10xDevs AI Toolkit - Module 2, Lesson 1

Move from sprint-zero setup to project orchestration with the **roadmap chain**:

```
(Module 1 foundation docs) -> /10x-roadmap -> backlog-ready roadmap items
```

`/10x-roadmap` is the lesson focus. `/10x-new` is intentionally introduced in Module 2, Lesson 2, when a selected roadmap item becomes an implementation change folder.

### Task Router - Where to start

| Skill | Use it when |
| --- | --- |
| **Roadmap (lesson focus)** | |
| `/10x-roadmap` | You have `context/foundation/prd.md` and a scaffolded project baseline, and you need a vertical-first MVP roadmap. The skill reads the PRD, inspects the code baseline, uses available foundation docs such as `tech-stack.md`, `infrastructure.md`, and `deploy-plan.md`, then writes `context/foundation/roadmap.md`. Use it BEFORE creating per-change folders or implementation plans. |
| **Re-run upstream if needed** | |
| `/10x-shape` / `/10x-prd` / `/10x-tech-stack-selector` / `/10x-bootstrapper` / `/10x-agents-md` / `/10x-infra-research` | Bundled from Module 1 so foundation contracts can be fixed before roadmap sequencing. If roadmap generation exposes a PRD gap, repair the PRD before pretending the backlog is ready. |

### How the chain hands off

- `/10x-roadmap` bridges product and implementation. It does not choose frameworks, design schemas, or write a per-change implementation plan.
- The output is `context/foundation/roadmap.md`: ordered milestones, vertical slices, bounded foundations, dependencies, unknowns, risk, and backlog handoff fields.
- Roadmap items should receive stable human-readable identifiers in backlog tools. The actual `context/changes/<change-id>/` folder is created in Lesson 2 with `/10x-new`.

### Roadmap boundaries

- Default to vertical slices: user-visible outcomes that cross UI, data, business logic, and integrations.
- Horizontal work is allowed only as a bounded enabler that names the downstream vertical milestone it unlocks.
- Avoid orphan horizontal work such as "build the whole database", "build all API endpoints", or "design the whole UI" before the first user-visible flow.
- Roadmap is not a calendar estimate. Do not invent dates, story points, or sprint velocity unless the user explicitly asks for a separate planning artifact.

### Foundation paths used by this lesson

- `context/foundation/prd.md` - input
- `context/foundation/tech-stack.md` - optional input
- `context/foundation/infrastructure.md` - optional input
- `context/deployment/deploy-plan.md` - optional input
- `context/foundation/roadmap.md` - output
- `context/foundation/lessons.md` - recurring rules and pitfalls
- `docs/reference/contract-surfaces.md` - load-bearing names registry

Skills must not write to `context/archive/`. Archived changes are immutable; if a resolved target path starts with `context/archive/`, abort with: "This change is archived. Open a new change with `/10x-new` instead."

<!-- END @przeprogramowani/10x-cli -->
