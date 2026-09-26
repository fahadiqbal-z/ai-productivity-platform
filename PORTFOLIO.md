# Cadence — Portfolio Case Study

## Project overview

Cadence is an AI-powered productivity platform: a personal command center that
carries a user from a rough goal to finished work. It combines a task manager,
project workspaces, goal hierarchies, notes, a calendar, focus mode, and an AI
planner — with the AI designed as an assistant that drafts and suggests, never
as an autopilot that decides.

## Problem

Goal-setting tools tend to fail in one of two directions: blank lists that
leave all structuring to the user, or AI features that generate content the
user didn't ask to keep. The gap is a workflow where AI does the tedious
structuring work while the human keeps full control and full context.

## Solution

A full-stack workspace where the core loop is **suggest → review → edit →
confirm**: the AI planner drafts milestones, tasks, and dates from a goal
description, and nothing enters the workspace until the user reviews and
confirms it. The same pattern repeats for task breakdowns and note-to-task
extraction. A workspace-aware assistant answers “what should I focus on?”
exclusively from the user's real data, citing the facts it used.

## Architecture

- **Frontend:** React 18 SPA (Vite), React Router, hand-written CSS design
  system — warm paper, ink, hairline rules, serif display type. No UI framework.
- **Backend:** Node.js + Express REST API, route-per-resource modules.
- **Database:** SQLite (WAL mode) with 9 relational tables, foreign keys, and
  indexes; every row user-scoped.
- **AI layer:** provider abstraction (`local` heuristics + OpenAI-compatible
  hosted models) with prompt registry, Zod-validated structured outputs, and
  automatic fallback. Works with zero API keys.
- **Deploy:** one Node process serves API + static client (`npm run build && npm start`).

## Important technical decisions

1. **SQLite over Postgres** — zero-ops relational storage that keeps the demo
   deployable anywhere; the query layer is centralized for a future adapter.
2. **Confirm-step writes** — AI suggestion endpoints are read-only by
   construction; separate confirm endpoints transact the writes. This makes
   “AI never deletes anything” a structural property, not a promise.
3. **Validated structured outputs** — hosted-model JSON is Zod-parsed and
   discarded on mismatch; the local provider guarantees the same schemas.
4. **Safe deletion semantics** — deleting a project or goal preserves the
   user's tasks and notes (unlinked, not destroyed).
5. **No scores or streaks** — insights are factual counts; the product
   deliberately refuses to quantify “productivity.”

## AI implementation

- Goal planning with domain-aware phasing and date spreading.
- Task breakdown into ordered sub-two-hour steps.
- Note assistance: summarize, key-points memo, sentence-level task extraction,
  simplify.
- Workspace assistant with verified-facts context and fact/suggestion labeling.
- Full request logging (kind + excerpt) for transparency.

## Security considerations

bcrypt hashing, JWT sessions with production secret enforcement, per-user query
scoping, Zod validation, parameterized queries, Helmet/CORS/rate-limiting,
uniform auth errors, password-confirmed account deletion, and a verified
no-secrets frontend bundle.

## Links

- Live demo: _add when deployed_
- Repository: _this repo_

_Only links that exist are listed. Nothing here is fabricated._
