# Cadence — AI-Powered Productivity Platform

> A personal command center for getting serious work done.

Cadence takes a user from **idea → plan → tasks → focus → progress → completion**.
You state a goal in plain words, the AI planner drafts milestones, tasks, and dates,
you review and edit every line, and then you work the list — one task at a time —
with projects, notes, a calendar, focus mode, and a workspace-aware assistant.

**Live with no API keys required:** the built-in local AI provider drafts plans,
breakdowns, note assistance, and chat answers on the server. Point it at any
OpenAI-compatible endpoint later without changing application code.

---

## Overview

Most productivity tools are either a blank list (all structure is your problem)
or an autopilot (the AI decides, you watch). Cadence sits in the middle:

- **AI suggests, you decide.** Every AI output lands in a review screen first.
  Nothing AI-made enters the workspace without explicit confirmation.
- **Goals stay connected to work.** The chain goal → milestone → project → task
  is visible everywhere, so a task always has a reason to exist.
- **Facts, not theater.** Insights are plain counts (“tasks completed this week:
  24”). There are no productivity scores, streaks, or gamified distractions.

## Features

| Area | What it does |
|---|---|
| Dashboard | Today's focus, active projects, 7-day outlook, recent activity, quick actions |
| Tasks | Inbox → Planned → In Progress → Completed, priorities, due dates, labels, subtasks, search / filter / sort, quick-add |
| Projects | Own workspace with tasks, notes, deadlines, progress, safe delete (work is kept, never destroyed) |
| Goals | Goal → milestones → projects → tasks hierarchy with progress and target dates |
| AI planner | Goal brief → drafted plan → **review & edit** → confirm into goal + project + dated tasks |
| AI breakdown | Turn any large task into ordered steps; pick, edit, then add as subtasks |
| Notes | Two-pane notes with autosave, project links, archive; AI summarize / key points / extract-to-tasks / simplify |
| Assistant | Workspace-aware chat that cites the facts it used and clearly separates facts from suggestions |
| Focus mode | Minimal full-screen session: one task, timer, steps, session notes saved back to the task |
| Calendar | Day / week / month views of due dates, deadlines, and milestones |
| Search | Global `⌘K` palette across tasks, projects, goals, notes — plus creation commands |
| Insights | Factual counts, 14-day completion chart, workload distribution, overdue list |
| Account | Register, login, profile, full JSON export, sample-data loader, password-confirmed delete |

## Product Architecture

```
┌─────────────┐     HTTPS/JSON      ┌──────────────────┐
│  React SPA  │ ◄─────────────────► │  Express API     │
│  (Vite)     │   Bearer JWT        │  + SQLite (WAL)  │
└─────────────┘                     └────────┬─────────┘
                                             │ validated JSON only
                                    ┌────────▼─────────┐
                                    │   AI layer       │
                                    │ local │ openai*  │
                                    └──────────────────┘
                                    * any OpenAI-compatible endpoint
```

- **Frontend** (`client/`): React 18 + React Router, one coherent design system
  (`styles.css`), no component library, no chart library, no CSS framework.
- **Backend** (`server/src/`): Express 4, route modules per resource, Zod
  validation on every write, `better-sqlite3` with WAL mode.
- **Production deploy**: `npm run build` then `npm start` — one Node process
  serves the API and the static client. No separate frontend host needed.

## AI Architecture

All AI access goes through `server/src/ai/`:

- `prompts.js` — every hosted-model prompt in one reviewable registry.
- `providers: local.js` — deterministic heuristics (domain-aware plan templates,
  sentence-level task extraction, fact-cited chat). Zero dependencies, works offline.
- `providers: openai.js` — OpenAI-compatible chat completions with JSON mode,
  30s timeouts, strict parsing.
- `index.js` — `planGoal / breakdownTask / assistNote / chat`. Tries the hosted
  model when configured, **validates with Zod**, falls back to local on any
  failure, and logs each request (kind + 200-char excerpt) to `ai_requests`.

Safety rules enforced in code:

1. AI routes that *suggest* never write to user tables; separate `/confirm`
   routes write, inside transactions, after user review.
2. Hosted-model JSON is schema-validated; invalid output is discarded, never saved.
3. The assistant receives only the requesting user's workspace facts and is
   instructed (and UI-labeled) to separate facts from suggestions.
4. No delete/overwrite capability exists anywhere in the AI path.

## Security

Practical measures for a real deployment (proportionate to a portfolio project):

- Passwords hashed with bcrypt (cost 12); login uses uniform error messages and
  a dummy comparison to avoid account enumeration.
- JWT sessions (7-day TTL); server refuses to boot in production without `JWT_SECRET`.
- Every query scoped by `user_id`; cross-user access returns 404 (no ID oracle).
- Zod validation on all writes; parameterized queries only; JSON body limit 256KB.
- Helmet headers, CORS allowlist, rate limits (strict on auth, moderate on AI).
- Destructive actions require confirmation in UI and re-authentication (password)
  for account deletion; project/goal deletion preserves user content by design.
- No secrets in frontend code or the bundle (verified); `.env.example` documents
  every variable; `.env` is gitignored.

Not claimed: this has not had a professional security audit. It is
security-conscious student work, not a certified system.

## Privacy

- Stored: account, workspace content (goals, projects, tasks, notes), activity
  log, AI request log (kind + short excerpt). Nothing else.
- With the default `local` provider, no workspace text leaves the server.
- With a hosted provider configured, only the text you submit per request is
  sent — and Settings tells you which mode is active.
- You can export everything as JSON or delete the account (cascades to all data).

## Tech Stack

| Layer | Choice | Why |
|---|---|---|
| Frontend | React 18, React Router, Vite | Component model + routing without framework weight; instant dev loop |
| Styling | Hand-written CSS system | Original identity, no framework look, tiny bundle (~38KB CSS) |
| Backend | Node 20, Express 4 | Simple, well-understood, easy to secure and deploy |
| Database | SQLite (better-sqlite3, WAL) | Real relational DB, zero-ops, file-backed; migratable to Postgres later |
| Validation | Zod (shared patterns) | One source of truth, field-level errors |
| Auth | bcrypt + JWT Bearer | Stateless API, standard practice |
| AI | Abstraction over local + OpenAI-compatible | Works with no keys; swappable provider |

## Database Structure

`users` → `goals` → `milestones`; `goals` → `projects` → `tasks` → `subtasks`;
`notes` optionally linked to `projects`; plus `activity` and `ai_requests` logs.
Foreign keys with `ON DELETE CASCADE` for owned children and `SET NULL` for
links that must survive deletion (tasks outlive their project). See
`server/src/db.js` for the full schema with indexes.

## Installation

Prerequisites: **Node 20+** and npm.

```bash
git clone <this-repo> && cd cadence
npm run install:all        # installs server/ and client/
cp .env.example server/.env
# edit server/.env — at minimum set JWT_SECRET to a long random string
```

## Environment Variables

| Variable | Required | Default | Purpose |
|---|---|---|---|
| `PORT` | no | `4000` | API port |
| `JWT_SECRET` | **yes in prod** | dev fallback (refuses prod boot without it) | Session signing |
| `CORS_ORIGIN` | no | `http://localhost:5173` | Allowed frontend origin(s), comma-separated |
| `DATA_DIR` | no | `./data` | SQLite file location |
| `AI_PROVIDER` | no | `local` | `local` or `openai` |
| `OPENAI_API_KEY` | iff `openai` | — | Hosted model key (server-side only) |
| `OPENAI_BASE_URL` | no | `https://api.openai.com/v1` | Compatible endpoint |
| `OPENAI_MODEL` | no | `gpt-4o-mini` | Model name |

## Running Locally

```bash
# Terminal 1 — API (http://localhost:4000)
npm run dev:server

# Terminal 2 — client (http://localhost:5173)
npm run dev:client
```

Then open http://localhost:5173, create an account, and either describe a goal
in the **AI planner** or load **sample data** from Settings to explore.

Single-process production run:

```bash
npm run build
npm start              # serves API + built client on $PORT
```

## Screenshots

Screenshots live in `docs/` (capture from a local run; none are checked in by
default to avoid stale images). Key screens to look at: Dashboard, AI planner
review, Task drawer with breakdown, Goal hierarchy, Focus mode, Insights.

## Future Improvements

- Password reset via email (requires a mail provider; schema-ready).
- Shared projects / collaboration (data model is currently single-user by design).
- Recurring tasks and reminders.
- Postgres adapter behind the same query layer for multi-instance hosting.
- Offline-first client with background sync.
- Semantic search over notes (labeled as AI search, separate from keyword search).

## Known Limitations

- No email-based password reset yet (by design — no mail provider configured).
- Single-user workspaces; no sharing or teams.
- JWTs are stateless: “sign out everywhere” isn't possible without a denylist.
- The local AI provider is heuristic — genuinely useful for structure, not a
  replacement for a large model on open-ended questions.
- SQLite suits single-server deploys; see Postgres note above for scaling.

## Author

**Fahad Iqbal** — B.S. Cybersecurity student. Full-stack development,
cybersecurity, AI engineering, creative technology.

This repository is original portfolio work. No employers, clients, metrics, or
testimonials are claimed — the code is the credential.
