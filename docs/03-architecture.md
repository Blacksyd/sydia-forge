# Architecture

How the forge's tools are arranged across the project lifecycle, where each
lives, and how they communicate. **Claude Code is the orchestrator**: it reads
context files, runs CLIs and MCP tools, and calls models. The tools rarely talk
to each other peer-to-peer — Claude sits in the middle.

## Tools by stage

| Stage | Tools | Where it lives | How it communicates |
|---|---|---|---|
| **0. Setup** | `sydia-forge init`, **`git init`** | Local CLI, driven by Claude | Generates repo files (CLAUDE.md, `.claude/`, `openspec/`); git is initialized here, not at deploy time |
| **1. Plan** | OpenSpec, BMAD (Analyst / PM / Architect) | Repo files (`openspec/`, `.claude/agents/`) | Context files read by agents (ECC pattern) |
| **2. Build / Proto** | Angular CLI (`ng`), Supabase CLI (local Docker stack), CodeGraph, ECC agents | Local repo + local Docker (Supabase), Claude Code | Claude runs CLIs via Bash; agents read context files |
| **3. Review** | KEV / JEV (System One triage), `code-review-skill` / `fcbm-pr-review` (System Two), test-gen | Self-hosted KEV endpoint / JEV API; skills in Claude Code | Claude calls the decision model over HTTP to triage → invokes generative review **only** on a flag |
| **4. Deploy** | **git → Vercel** (front), **Supabase CLI `db push`** (DB), Vercel | GitHub, Vercel cloud, Supabase cloud | `git push` triggers the Vercel build (webhook); Supabase CLI/MCP → Management API |
| **5. Release / Ops** | Hardening checklist, keep-alive cron, agent observability, claude-mem, shared-state JSON | Supabase (cron), Vercel, Claude memory | Cron HTTP anti-pause; hooks → AgentOps; claude-mem = local store |

Git is active from stage 0 to the end — it is bus #1, not a deploy-time tool.

## Local → prod propagation

The app has two halves that propagate differently. This is the core mechanism.

### ① Front-end (Angular) → Vercel = via git

Not an MCP, not a custom API. Claude commits and runs `git push` to GitHub;
Vercel's Git integration detects the commit, runs `ng build`, and deploys.
Alternative: `vercel deploy` (CLI). Front-end deploys are immutable and
disposable — a fresh build per push.

### ② Database (local Supabase) → Supabase cloud = via CLI migrations

The database is **stateful** — you don't "push" it, you apply **versioned
migrations** so real cloud data is never overwritten.

```bash
supabase start                              # local stack (Postgres + Auth + Storage) in Docker
supabase db diff -f <name>                  # capture local schema changes as a migration
supabase link --project-ref <ref>           # once: link local ↔ cloud
supabase db push                            # apply migrations to the cloud DB
supabase functions deploy                   # deploy Edge Functions
```

Under the hood the CLI talks to the Supabase Management API. Claude can run this
via Bash **or** through the Supabase MCP server (managing migrations/SQL/project
as MCP tools instead of raw CLI). The MCP is a cleaner way for Claude to drive
Supabase — not a magic channel between Vercel and Supabase.

## The communication buses

1. **Git** — source of truth; triggers Vercel; carries OpenSpec specs + context files.
2. **CLIs run by Claude (Bash)** — `ng`, `supabase`, `vercel`, `git`.
3. **MCP** — Supabase (DB / project); optionally Vercel (else its API/CLI).
4. **Shared context files** — `.claude/context/`, `openspec/`, `.claude/state/current-feature.json`, claude-mem. This is how one Claude session communicates with the next and how agents coordinate.
5. **HTTP/REST at runtime** — Angular ↔ Supabase (`anon` key + RLS); Claude ↔ KEV/JEV endpoints.

At the center: **Claude Code**, which reads the files (1, 4), actuates the arms
(2, 3), and calls the models (5).

## Repo topology (complexity-driven)

Decided at `init`, per project — see blank spot #7.

- **Simple project** → one repo; the prototype hardens in place. No rupture.
- **Complex project** → prototype clears the ground, then a clean rebuild:
  monorepo (`apps/proto` + `apps/app`, shared `supabase/`) when schema/libs
  survive, or two repos when everything is discarded.

Vercel does not constrain this choice (up to 200 projects on Hobby). The real
criterion is code sharing vs isolation, not infrastructure.
