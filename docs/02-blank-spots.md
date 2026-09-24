# Blank Spots

Layers of the development cycle with no adequate community-maintained brick. These are the areas sydia-forge must address with custom work.

---

## 1. Project onboarding generator ← Primary blank spot

**What it is:** The tool that installs and configures the right bricks for a new project, generates the project-specific delta (context files, stack conventions, selected bricks), and asks the questions that determine the configuration.

**Why it matters:** Without this, every project still starts from scratch — manually installing BMAD, configuring ECC, writing context files. The generator is what makes the meta-framework reusable.

**What to build:**
```bash
npx sydia-forge init

Prompts:
- Stack? (Angular, React, Vue, Next.js, Django, Rails…)
- Backend? (Supabase, Firebase, custom API, none)
- Team size? (solo, 2-3, 4+)
- Phase? (prototype, product, mature codebase)
- Bricks to activate?

Outputs:
- CLAUDE.md (pre-configured)
- .claude/agents/ (BMAD personas as ECC-style skeletons)
- .claude/context/ (stack-specific context files)
- .claude/skills/ (selected skills)
- .claude/settings.json (hooks)
- openspec/ (if OpenSpec selected)
```

---

## 2. Agent observability

**What it is:** Tracing which agents ran, what they did, what it cost, where they failed — across sessions.

**Why it matters:** Without observability, debugging multi-agent workflows is blind. You don't know which agent produced a bad output or why.

**Available bricks:** AgentOps, LangSmith — but no native Claude Code integration.

**What to build:** Hook-based bridge that sends Claude Code agent events to AgentOps via HTTP. One `PostToolUse` hook + one `Stop` hook.

---

## 3. Test generation pipeline

**What it is:** Automatic generation and maintenance of tests from the implementation, integrated in the Claude Code workflow.

**Why it matters:** BMAD has a QA persona but no automated test generation. Qodo generates tests but as a separate CLI.

**What to build:** A `test-gen` skill that calls the `back`/`front` agents with a test-writing prompt after implementation, following the project's existing test patterns.

---

## 4. Multi-agent shared state (cross-session)

**What it is:** A state machine that persists across sessions, so agents can pick up where the previous session left off without re-reading everything.

**Why it matters:** Today's multi-agent coordination is session-scoped. If a feature spans 3 sessions, agents restart from scratch each time.

**Available bricks:** claude-mem handles observations; LangGraph handles state machines (Python-only).

**What to build:** A lightweight JSON state file (`.claude/state/current-feature.json`) written by agents, read at session start. Simple, no dependencies.

---

## 5. Prototyping layer (owned-code, no vendor lock-in)

**What it is:** Fast scaffolding that produces a running prototype in hours — but as real, owned code in the target stack (Angular / SvelteKit / Nuxt + Supabase), living in the same git repo the release will ship from. Git is initialized at step 0, before the first line of prototype code.

**Why it matters:** SaaS builders (Lovable, v0, bolt) get you to a demo fast but trap the code in their runtime; escaping them means a rewrite. The whole point of the forge is proto = release, no tool rupture, no lock-in.

**Anti-brick:** Lovable / v0 / bolt — explicitly NOT recommended. They own the code; we want to own it.

**What to build:** A `prototype` path in `sydia-forge init` that scaffolds the chosen stack via native CLIs (`ng` / `nuxi` / `create-svelte`) + the Supabase CLI (local stack in Docker), wired by Claude Code agents. A throwaway-or-keep prototype that is already the real project.

---

## 6. Deployment layer (Vercel Hobby ↔ Supabase Free)

**What it is:** The default deployment target and the glue that makes a personal project publishable in one step. Front-end propagates to Vercel via git (Git integration); the database propagates to Supabase via CLI migrations.

**Why it matters:** Without it, every project re-solves the same friction: SPA rewrites, Supabase Auth redirect URLs, the `anon` key placement, the Free-tier inactivity pause. For strictly personal use, Vercel Hobby has no commercial restriction — a single Hobby/Free tier carries a project from prototype to daily-use app.

**Available bricks:** Vercel Git integration + presets, Supabase CLI (`link` / `db push` / `functions deploy`), Supabase MCP server, `@supabase/ssr` — but nothing wires them together per project.

**What to build:** A `deploy` path (init output + skill) that generates `vercel.json` (SPA rewrite to `index.html`), places the `anon` key in `environment.prod.ts` (public by design; security = RLS, never `service_role` on the front), registers the `*.vercel.app` URL in Supabase Auth (Site URL + Redirect URLs), and adds an optional keep-alive cron against the ~1-week Free-tier pause.

**Constraints (not gates, for personal use):** Vercel Hobby allows up to 200 projects and 100 deployments/day, but only 1 concurrent build and ~100 GB/month bandwidth. Supabase Free caps at 2 active projects, 500 MB DB, and pauses after ~1 week of inactivity.

---

## 7. Release / hardening layer

**What it is:** The transition from prototype to release. Not a rewrite by default — a hardening checklist applied to the same repo (or the clean base, for complex projects).

**Why it matters:** "It runs" is not "it ships." A prototype typically has partial RLS, unwatched quotas, and no operational safety net. Nothing today formalizes what must be true before a project is trusted for daily use.

**Repo topology (complexity-driven, decided at `init`):**
- **Simple project** → one repo, the prototype hardens in place (branches to experiment). No rupture.
- **Complex project** → the prototype clears the ground, then a clean rebuild in a separate base — monorepo (`apps/proto` + `apps/app`, shared `supabase/`) when the schema/libs survive the rebuild, or two repos when everything is discarded. This sub-choice is itself an `init` question per project.

**What to build:** A `release` checklist skill: complete RLS on every table, verify quota headroom (bandwidth, DB size), confirm keep-alive, and — only if the project ever becomes non-personal — flag the Vercel Hobby→Pro migration.

---

## 8. Decision-model review tier (System One)

**What it is:** A fast triage tier placed in front of generative code review, using System One decision models — **KEV** (open-source, self-hostable, Qwen3.5-based) and **JEV** (TypeSafe AI, proprietary; open-source clone via DiffusionGemma). These return typed, calibrated decisions in one parallel pass (70–500 ms) instead of generating prose.

**Why it matters:** A generative reviewer (Claude, GPT — "System Two") is powerful but slow and costly on every PR. Decision models answer typed questions cheaply ("does this diff touch auth?", "breaking change?", "risk score?") so the generative reviewer is invoked only when triage raises a flag. Big latency and cost savings, review quality kept where it counts.

**Available bricks:** KEV (self-hosted System One endpoint, `kev-finetune`), JEV (API, ~70–500 ms typed calls), plus existing generative-review skills (`code-review-skill`, `fcbm-pr-review`). None are wired into a two-tier PR pipeline.

**Anti-lock-in note:** Prototype the triage with JEV (fast to plug in), then switch to self-hosted KEV to depend on no one — the same proto→release pattern the forge defends elsewhere.

**What to build:** A `review-triage` skill: on a PR, call the decision-model endpoint with the project's typed review questions; on any flag, hand the diff to the generative review skill; otherwise pass. Relates to blank spot #3 (test generation) — same "automate a quality gate in the Claude Code workflow" logic.
