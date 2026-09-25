# sydia-forge init — Design Spec

**Date:** 2026-09-25
**Status:** Approved (conversational), ready for implementation plan
**Scope:** v1 of the `sydia-forge init` generator

## Intent

`sydia-forge init` is the onboarding generator of the forge (blank spot #1 in
`docs/02-blank-spots.md`). You **choose the technologies**, and `init`
**adapts** — scaffolding a project with the right files, conventions, git
history, and optional deployment wiring, so a project starts configured instead
of from scratch.

The generator is **brick-driven**. This spec defines the brick/combo model, the
engine that composes them, and the concrete v1 slice.

## Core concepts

- **Brick** — the atomic unit. One technology concern that knows how to generate
  *its own* files and (optionally) *its own* deployment method. Kinds: `front`,
  `backend`, `hosting`, `db`, `auth`. Examples: `angular`, `supabase`, `vercel`.
- **Combo** — a named preset of bricks **plus the integration glue** between
  them. Example: `angular-supabase` = `angular` + `supabase` + `vercel` + the
  tested files that wire them together.
- **Glue** — the integration files a combo owns because a lone brick cannot know
  its neighbours (e.g. `environment.prod.ts` carrying the Supabase `anon` key,
  the Supabase client provider inside Angular).

A single mechanism serves both usage models the user asked for: pick a **combo**
(preset), or select **bricks à la carte** (engine composes them; glue may be
combo-provided only).

## Goals

- Scaffold an Angular + Supabase project deployable to Vercel Hobby + Supabase
  Free, for **both phases**: simple (single repo, proto hardens in place) and
  complex (monorepo `apps/proto` + `apps/app`).
- Make **adding a brick** a matter of dropping a folder (declarative manifest +
  templates), with an optional `deploy.ts` code hook for the deployment method.
- Initialize git at step 0, with a first commit.
- Emit the forge's own context scaffolding (`CLAUDE.md`, `.claude/`, `openspec/`).

## Non-goals (YAGNI for v1)

- **Capability negotiation engine.** `provides`/`consumes` are recorded in the
  manifest but v1 glue for `angular-supabase` is **explicit/hardcoded** for that
  trio — no dynamic resolution. Generalize when a second real combo
  (Symfony/MariaDB) reveals the right abstraction.
- Other stacks (React, Vue, Next, Symfony, MariaDB) — architecture must allow
  them, v1 does not ship them.
- À-la-carte with more than one combo available (the mode exists; only
  `angular-supabase` is offered).
- KEV/JEV review triage, agent observability, real cloud deploy executed in CI.

## Architecture

CLI repository layout:

```
sydia-forge/
  src/
    cli.ts              bin entry (`sydia-forge`), arg parsing, subcommand dispatch
    commands/init.ts    the init flow orchestration
    engine/
      prompts.ts        interactive prompt runner
      resolver.ts       resolve bricks from a combo or an à-la-carte selection
      renderer.ts       template rendering (token substitution)
      glue.ts           apply a combo's integration files
      git.ts            git init + first commit
      deploy.ts         orchestrate bricks'/combo's deploy hooks (opt-in)
    types.ts            Brick, Combo, Manifest, Context types
  bricks/
    angular/  { brick.json, templates/ }
    supabase/ { brick.json, templates/, deploy.ts }
    vercel/   { brick.json, deploy.ts }
  combos/
    angular-supabase/ { combo.json, glue/, deploy.ts }
  test/               vitest unit + integration tests
  package.json        bin: { "sydia-forge": "dist/cli.js" }
```

### Runtime / tech choices

- **Node.js 20+ / TypeScript**, distributed on npm, invoked via `npx sydia-forge`.
- **Prompts:** `@clack/prompts` (small, good UX).
- **Templates:** `eta` (lightweight, logic-capable) with `{{token}}`-style data.
- **Build:** `tsup` (esbuild) → `dist/`, `bin` shebang entry.
- **Tests:** `vitest`.
- Templates and manifests ship inside the package (bundled), resolved relative
  to the package root at runtime.

### `brick.json` (declarative)

```json
{
  "id": "supabase",
  "kind": "backend",
  "label": "Supabase",
  "provides": ["db", "auth", "backend-client"],
  "consumes": [],
  "prompts": [
    { "id": "projectRef", "message": "Supabase project ref (cloud)", "when": "deploy" }
  ],
  "templates": [
    { "from": "templates/config.toml", "to": "supabase/config.toml" },
    { "from": "templates/0001_init.sql", "to": "supabase/migrations/0001_init.sql" }
  ],
  "deploy": "deploy.ts"
}
```

### `combo.json`

```json
{
  "id": "angular-supabase",
  "label": "Angular + Supabase (Vercel)",
  "bricks": ["angular", "supabase", "vercel"],
  "phases": {
    "simple":  { "topology": "single-repo" },
    "complex": { "topology": "monorepo", "apps": ["proto", "app"] }
  },
  "glue": [
    { "from": "glue/environment.prod.ts.eta", "to": "src/environments/environment.prod.ts" },
    { "from": "glue/supabase.client.ts.eta",  "to": "src/app/core/supabase.client.ts" }
  ],
  "deploy": "deploy.ts"
}
```

### `deploy.ts` hook (code, optional)

Each brick/combo may export a default async function receiving the resolved
`Context` (answers, target dir, topology) and running its deployment method. The
combo's `deploy.ts` orchestrates the bricks' hooks in order. Examples:
- `vercel` brick: `git push` (relies on Vercel Git integration) or `vercel deploy`.
- `supabase` brick: `supabase link --project-ref <ref>` then `supabase db push`.
- `angular-supabase` combo: run supabase deploy, then vercel deploy.

Deploy is **opt-in** — `init` scaffolds and initializes git by default; deploying
is a confirmed extra step.

## `init` execution flow

1. **Mode** — prompt: use a **combo preset** or pick **bricks à la carte**?
2. **Combo/bricks** — if combo, pick from available combos (v1: only
   `angular-supabase`); if à-la-carte, multi-select bricks.
3. **Phase** — simple or complex → determines topology (single-repo / monorepo).
   For complex, sub-choice: shared Supabase schema (monorepo) or discard (2 repos).
4. **Resolve** bricks (from combo or selection) via `resolver.ts`.
5. **Prompts** — run each brick's prompts (skip `when: "deploy"` prompts unless
   deploying).
6. **Render** — write each brick's templates into the target dir (topology-aware
   destination paths).
7. **Glue** — apply the combo's integration files.
8. **Forge scaffolding** — emit `CLAUDE.md`, `.claude/` (agents, context,
   settings), `openspec/`.
9. **Git** — `git init`, first commit.
10. **Deploy (opt-in)** — if confirmed, run the deploy orchestration.

## Data flow & isolation

Each engine module has one purpose and a typed interface:
- `resolver` : (combo|selection) → `Brick[]`
- `renderer` : (`Brick`, `Context`) → files on disk
- `glue`     : (`Combo`, `Context`) → integration files on disk
- `git`/`deploy` : side-effecting steps behind explicit calls

Bricks never import each other. The only cross-brick knowledge lives in a combo's
`glue/` and `deploy.ts`.

## Topology handling

- **single-repo (simple):** templates render at repo root; one Angular app.
- **monorepo (complex):** Angular workspace with `apps/proto` + `apps/app`,
  shared `libs/` (ui, data) and shared `supabase/`. Renderer prefixes destination
  paths per app; combo glue wires the shared Supabase client into a `libs/data`
  library consumed by both apps.

## Error handling

- Validate manifests on load (fail fast with a clear message naming the brick).
- Refuse to write into a non-empty target dir unless `--force`.
- Deploy hooks surface the underlying CLI's stderr; a failed deploy never
  corrupts the already-generated scaffold (generation and deploy are separate
  phases).

## Testing strategy

- **Unit:** `resolver` (combo → bricks, missing brick error), `renderer` (token
  substitution, topology path prefixing), `glue` application.
- **Integration:** run `init` into a temp dir with a fixed answer set for both
  phases; assert the generated file tree and key file contents (e.g.
  `environment.prod.ts` contains the anon-key token, `vercel.json` has the SPA
  rewrite).
- **Deploy hooks:** tested with the child-process runner mocked — CI never
  performs a real deploy.

## Open questions (deferred, not blocking)

- Exact `libs/` structure for the monorepo phase (Nx vs plain Angular workspace)
  — decided during implementation of the complex-phase templates.
- Whether à-la-carte without a combo falls back to a generic glue or just warns
  — v1 warns; revisit with the second combo.
```

