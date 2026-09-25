# sydia-forge init Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build v1 of the `sydia-forge init` CLI: a brick-driven generator that scaffolds an Angular + Supabase project (simple or monorepo) deployable to Vercel Hobby + Supabase Free.

**Architecture:** A generic TypeScript engine composes **bricks** (atomic tech concerns declared by `brick.json` + templates) selected via a **combo** (named preset + integration glue). The engine loads manifests, resolves bricks, renders templates into the target dir (topology-aware), applies combo glue, inits git, and optionally runs per-brick deploy hooks.

**Tech Stack:** Node.js 20+, TypeScript, `@clack/prompts` (prompts), `tsup` (build), `vitest` (tests). Minimal built-in `{{token}}` template renderer (no runtime template engine — see Global Constraints).

**Spec:** `docs/superpowers/specs/2026-09-25-sydia-forge-init-design.md`

## Global Constraints

- Node.js **>= 20**; package is ESM (`"type": "module"`).
- Bin name is exactly `sydia-forge`; entry is `dist/cli.js`.
- **v1 renderer:** minimal `{{key}}` string substitution (refines the spec's `eta` mention — eta's logic isn't needed yet; YAGNI). Unknown `{{key}}` with no answer → empty string, and a collected warning.
- Bricks never import other bricks. Cross-brick knowledge lives only in a combo's `glue/` and `deploy.ts`.
- v1 ships bricks `angular`, `supabase`, `vercel` and the combo `angular-supabase` only. À-la-carte mode exists but offers only these bricks.
- `provides`/`consumes` are recorded but NOT resolved dynamically in v1 (glue is explicit).
- Generation and deploy are separate phases; a failed deploy must never corrupt an already-generated scaffold.
- Refuse to write into a non-empty target dir unless `--force`.

## Review Focus

- **Non-empty target dir:** running `init` into a dir that already has files must abort with a clear message unless `--force` — covered in Task 10.
- **Unknown brick id in a combo:** `combo.bricks` referencing a missing brick must fail fast naming the brick — covered in Task 3.
- **Malformed manifest:** invalid/missing-field `brick.json`/`combo.json` must fail fast naming the file — covered in Task 2.
- **Monorepo path prefixing:** a template `to` containing `{{app}}` must render once per app into `apps/<app>/...`; single-repo renders once at root — covered in Task 4.
- **Deploy declined:** when the user declines deploy, `when: "deploy"` prompts are skipped and no deploy hook runs — covered in Task 10.

---

### Task 1: Project scaffold + core types

**Files:**
- Create: `package.json`, `tsconfig.json`, `tsup.config.ts`, `vitest.config.ts`
- Create: `src/cli.ts`, `src/types.ts`
- Test: `test/cli.test.ts`

**Interfaces:**
- Consumes: nothing.
- Produces: all shared types (`BrickKind`, `Topology`, `Phase`, `BrickPrompt`, `TemplateSpec`, `Brick`, `ComboPhase`, `Combo`, `Context`); a runnable `dist/cli.js` printing usage.

- [ ] **Step 1: Write the failing test**

```ts
// test/cli.test.ts
import { execFileSync } from 'node:child_process';
import { describe, it, expect } from 'vitest';

describe('cli', () => {
  it('prints usage with --help', () => {
    const out = execFileSync('node', ['dist/cli.js', '--help']).toString();
    expect(out).toContain('sydia-forge');
    expect(out).toContain('init');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm run build && npx vitest run test/cli.test.ts`
Expected: FAIL (no `dist/cli.js` / build not configured yet).

- [ ] **Step 3: Write minimal implementation**

`package.json`:
```json
{
  "name": "sydia-forge",
  "version": "0.1.0",
  "type": "module",
  "bin": { "sydia-forge": "dist/cli.js" },
  "engines": { "node": ">=20" },
  "scripts": {
    "build": "tsup",
    "test": "vitest run"
  },
  "dependencies": { "@clack/prompts": "^0.7.0" },
  "devDependencies": { "tsup": "^8.0.0", "typescript": "^5.4.0", "vitest": "^1.4.0", "@types/node": "^20.0.0" }
}
```

`tsconfig.json`:
```json
{
  "compilerOptions": {
    "target": "ES2022", "module": "ESNext", "moduleResolution": "Bundler",
    "strict": true, "esModuleInterop": true, "skipLibCheck": true,
    "outDir": "dist", "rootDir": "src"
  },
  "include": ["src"]
}
```

`tsup.config.ts`:
```ts
import { defineConfig } from 'tsup';
export default defineConfig({
  entry: ['src/cli.ts'],
  format: ['esm'],
  target: 'node20',
  banner: { js: '#!/usr/bin/env node' },
  clean: true,
});
```

`vitest.config.ts`:
```ts
import { defineConfig } from 'vitest/config';
export default defineConfig({ test: { environment: 'node' } });
```

`src/types.ts`:
```ts
export type BrickKind = 'front' | 'backend' | 'hosting' | 'db' | 'auth';
export type Topology = 'single-repo' | 'monorepo';
export type Phase = 'simple' | 'complex';

export interface BrickPrompt { id: string; message: string; when?: 'always' | 'deploy'; }
export interface TemplateSpec { from: string; to: string; }

export interface Brick {
  id: string; kind: BrickKind; label: string;
  provides: string[]; consumes: string[];
  prompts: BrickPrompt[]; templates: TemplateSpec[];
  deploy?: string; dir: string;
}
export interface ComboPhase { topology: Topology; apps?: string[]; }
export interface Combo {
  id: string; label: string; bricks: string[];
  phases: Record<Phase, ComboPhase>;
  glue: TemplateSpec[]; deploy?: string; dir: string;
}
export interface Context {
  targetDir: string; phase: Phase; topology: Topology;
  apps: string[]; answers: Record<string, string>; deploy: boolean;
}
```

`src/cli.ts`:
```ts
const USAGE = `sydia-forge — brick-driven project generator

Usage:
  sydia-forge init [dir] [--force]   Scaffold a new project
  sydia-forge --help                 Show this help`;

const [, , cmd] = process.argv;
if (!cmd || cmd === '--help') { console.log(USAGE); process.exit(0); }
if (cmd === 'init') { console.log('init: not yet implemented'); process.exit(0); }
console.error(`Unknown command: ${cmd}`); console.log(USAGE); process.exit(1);
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm run build && npx vitest run test/cli.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add package.json tsconfig.json tsup.config.ts vitest.config.ts src/cli.ts src/types.ts test/cli.test.ts
git commit -m "feat(cli): scaffold sydia-forge CLI and core types"
```

---

### Task 2: Manifest loader + validation

**Files:**
- Create: `src/engine/manifest.ts`
- Test: `test/manifest.test.ts`

**Interfaces:**
- Consumes: types from `src/types.ts`.
- Produces: `loadBrick(dir: string): Brick` and `loadCombo(dir: string): Combo`. Both read a JSON file (`brick.json` / `combo.json`) in `dir`, validate required fields, attach `dir`, and throw `Error` naming the file on any problem.

- [ ] **Step 1: Write the failing test**

```ts
// test/manifest.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { mkdtempSync, writeFileSync, mkdirSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { loadBrick, loadCombo } from '../src/engine/manifest.js';

function tmp() { return mkdtempSync(join(tmpdir(), 'sf-')); }

describe('loadBrick', () => {
  it('loads a valid brick and attaches dir', () => {
    const d = tmp();
    writeFileSync(join(d, 'brick.json'), JSON.stringify({
      id: 'supabase', kind: 'backend', label: 'Supabase',
      provides: ['db'], consumes: [], prompts: [], templates: [],
    }));
    const b = loadBrick(d);
    expect(b.id).toBe('supabase');
    expect(b.dir).toBe(d);
  });

  it('throws naming the file when a required field is missing', () => {
    const d = tmp();
    writeFileSync(join(d, 'brick.json'), JSON.stringify({ id: 'x' }));
    expect(() => loadBrick(d)).toThrow(/brick\.json/);
  });
});

describe('loadCombo', () => {
  it('loads a valid combo', () => {
    const d = tmp();
    writeFileSync(join(d, 'combo.json'), JSON.stringify({
      id: 'angular-supabase', label: 'A+S', bricks: ['angular'],
      phases: { simple: { topology: 'single-repo' }, complex: { topology: 'monorepo', apps: ['proto', 'app'] } },
      glue: [],
    }));
    expect(loadCombo(d).bricks).toEqual(['angular']);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/manifest.test.ts`
Expected: FAIL with "Cannot find module '../src/engine/manifest.js'".

- [ ] **Step 3: Write minimal implementation**

```ts
// src/engine/manifest.ts
import { readFileSync } from 'node:fs';
import { join } from 'node:path';
import type { Brick, Combo } from '../types.js';

function readJson(file: string): any {
  try { return JSON.parse(readFileSync(file, 'utf8')); }
  catch (e) { throw new Error(`Invalid JSON in ${file}: ${(e as Error).message}`); }
}
function require_(obj: any, fields: string[], file: string) {
  for (const f of fields) if (obj[f] === undefined) throw new Error(`Missing "${f}" in ${file}`);
}

export function loadBrick(dir: string): Brick {
  const file = join(dir, 'brick.json');
  const m = readJson(file);
  require_(m, ['id', 'kind', 'label'], file);
  return {
    id: m.id, kind: m.kind, label: m.label,
    provides: m.provides ?? [], consumes: m.consumes ?? [],
    prompts: m.prompts ?? [], templates: m.templates ?? [],
    deploy: m.deploy, dir,
  };
}

export function loadCombo(dir: string): Combo {
  const file = join(dir, 'combo.json');
  const m = readJson(file);
  require_(m, ['id', 'label', 'bricks', 'phases'], file);
  return {
    id: m.id, label: m.label, bricks: m.bricks, phases: m.phases,
    glue: m.glue ?? [], deploy: m.deploy, dir,
  };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/manifest.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/engine/manifest.ts test/manifest.test.ts
git commit -m "feat(engine): manifest loader with validation"
```

---

### Task 3: Brick resolver

**Files:**
- Create: `src/engine/resolver.ts`
- Test: `test/resolver.test.ts`

**Interfaces:**
- Consumes: `Brick`, `Combo` types.
- Produces: `resolveBricks(ids: string[], all: Brick[]): Brick[]` — returns the bricks matching `ids` in order; throws `Error` naming the missing id if any id has no brick. Used both for combos (`combo.bricks`) and à-la-carte selections.

- [ ] **Step 1: Write the failing test**

```ts
// test/resolver.test.ts
import { describe, it, expect } from 'vitest';
import { resolveBricks } from '../src/engine/resolver.js';
import type { Brick } from '../src/types.js';

const b = (id: string): Brick => ({ id, kind: 'front', label: id, provides: [], consumes: [], prompts: [], templates: [], dir: '/x' });

describe('resolveBricks', () => {
  it('resolves ids in order', () => {
    const all = [b('angular'), b('supabase'), b('vercel')];
    expect(resolveBricks(['vercel', 'angular'], all).map(x => x.id)).toEqual(['vercel', 'angular']);
  });
  it('throws naming a missing brick', () => {
    expect(() => resolveBricks(['ghost'], [b('angular')])).toThrow(/ghost/);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/resolver.test.ts`
Expected: FAIL ("Cannot find module").

- [ ] **Step 3: Write minimal implementation**

```ts
// src/engine/resolver.ts
import type { Brick } from '../types.js';

export function resolveBricks(ids: string[], all: Brick[]): Brick[] {
  return ids.map(id => {
    const found = all.find(b => b.id === id);
    if (!found) throw new Error(`Unknown brick: "${id}"`);
    return found;
  });
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/resolver.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/engine/resolver.ts test/resolver.test.ts
git commit -m "feat(engine): brick resolver"
```

---

### Task 4: Template renderer (token substitution + topology paths)

**Files:**
- Create: `src/engine/renderer.ts`
- Test: `test/renderer.test.ts`

**Interfaces:**
- Consumes: `TemplateSpec`, `Context`.
- Produces:
  - `renderString(tpl: string, data: Record<string, string>): { out: string; missing: string[] }` — replaces `{{key}}`; unknown keys → `''` and collected in `missing`.
  - `destPaths(to: string, ctx: Context): string[]` — if `to` contains `{{app}}` and topology is monorepo, returns one path per `ctx.apps` under `apps/<app>/`; otherwise a single path (root; `{{app}}` stripped).
  - `renderTemplates(fromDir: string, specs: TemplateSpec[], ctx: Context): string[]` — reads each `from` under `fromDir`, renders, writes to every `destPaths` under `ctx.targetDir`, returns the list of written absolute paths.

- [ ] **Step 1: Write the failing test**

```ts
// test/renderer.test.ts
import { describe, it, expect } from 'vitest';
import { mkdtempSync, writeFileSync, readFileSync, mkdirSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { renderString, destPaths, renderTemplates } from '../src/engine/renderer.js';
import type { Context } from '../src/types.js';

const ctx = (over: Partial<Context> = {}): Context => ({
  targetDir: '/t', phase: 'simple', topology: 'single-repo',
  apps: ['.'], answers: {}, deploy: false, ...over,
});

describe('renderString', () => {
  it('substitutes and reports missing', () => {
    const r = renderString('hi {{name}} {{gone}}', { name: 'sy' });
    expect(r.out).toBe('hi sy ');
    expect(r.missing).toEqual(['gone']);
  });
});

describe('destPaths', () => {
  it('single-repo strips {{app}} to root', () => {
    expect(destPaths('{{app}}/src/main.ts', ctx())).toEqual(['src/main.ts']);
  });
  it('monorepo expands per app', () => {
    const c = ctx({ topology: 'monorepo', apps: ['proto', 'app'] });
    expect(destPaths('{{app}}/src/main.ts', c)).toEqual(['apps/proto/src/main.ts', 'apps/app/src/main.ts']);
  });
});

describe('renderTemplates', () => {
  it('writes rendered files into targetDir', () => {
    const from = mkdtempSync(join(tmpdir(), 'sf-from-'));
    const target = mkdtempSync(join(tmpdir(), 'sf-t-'));
    writeFileSync(join(from, 'env.tmpl'), 'url={{url}}');
    const written = renderTemplates(from, [{ from: 'env.tmpl', to: 'env.ts' }], ctx({ targetDir: target, answers: { url: 'http://x' } }));
    expect(readFileSync(written[0], 'utf8')).toBe('url=http://x');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/renderer.test.ts`
Expected: FAIL ("Cannot find module").

- [ ] **Step 3: Write minimal implementation**

```ts
// src/engine/renderer.ts
import { readFileSync, writeFileSync, mkdirSync } from 'node:fs';
import { dirname, join } from 'node:path';
import type { TemplateSpec, Context } from '../types.js';

export function renderString(tpl: string, data: Record<string, string>): { out: string; missing: string[] } {
  const missing: string[] = [];
  const out = tpl.replace(/\{\{(\w+)\}\}/g, (_m, key: string) => {
    if (key === 'app') return _m; // handled by destPaths, not here
    if (data[key] === undefined) { missing.push(key); return ''; }
    return data[key];
  });
  return { out, missing };
}

export function destPaths(to: string, ctx: Context): string[] {
  if (to.includes('{{app}}') && ctx.topology === 'monorepo') {
    return ctx.apps.map(a => to.replace('{{app}}', `apps/${a}`));
  }
  return [to.replace(/\{\{app\}\}\/?/, '')];
}

export function renderTemplates(fromDir: string, specs: TemplateSpec[], ctx: Context): string[] {
  const written: string[] = [];
  for (const spec of specs) {
    const raw = readFileSync(join(fromDir, spec.from), 'utf8');
    const { out } = renderString(raw, ctx.answers);
    for (const rel of destPaths(spec.to, ctx)) {
      const abs = join(ctx.targetDir, rel);
      mkdirSync(dirname(abs), { recursive: true });
      writeFileSync(abs, out);
      written.push(abs);
    }
  }
  return written;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/renderer.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/engine/renderer.ts test/renderer.test.ts
git commit -m "feat(engine): template renderer with topology-aware paths"
```

---

### Task 5: Glue applier

**Files:**
- Create: `src/engine/glue.ts`
- Test: `test/glue.test.ts`

**Interfaces:**
- Consumes: `Combo`, `Context`, `renderTemplates` from Task 4.
- Produces: `applyGlue(combo: Combo, ctx: Context): string[]` — renders the combo's `glue` specs (resolved under `<combo.dir>/`) into `ctx.targetDir`; returns written paths. Reuses `renderTemplates`.

- [ ] **Step 1: Write the failing test**

```ts
// test/glue.test.ts
import { describe, it, expect } from 'vitest';
import { mkdtempSync, writeFileSync, mkdirSync, readFileSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { applyGlue } from '../src/engine/glue.js';
import type { Combo, Context } from '../src/types.js';

describe('applyGlue', () => {
  it('renders combo glue files into targetDir', () => {
    const comboDir = mkdtempSync(join(tmpdir(), 'sf-combo-'));
    mkdirSync(join(comboDir, 'glue'));
    writeFileSync(join(comboDir, 'glue', 'env.eta'), 'anon={{anonKey}}');
    const target = mkdtempSync(join(tmpdir(), 'sf-t-'));
    const combo: Combo = { id: 'c', label: 'c', bricks: [], phases: { simple: { topology: 'single-repo' }, complex: { topology: 'monorepo' } }, glue: [{ from: 'glue/env.eta', to: 'src/environments/environment.prod.ts' }], dir: comboDir };
    const ctx: Context = { targetDir: target, phase: 'simple', topology: 'single-repo', apps: ['.'], answers: { anonKey: 'abc' }, deploy: false };
    const written = applyGlue(combo, ctx);
    expect(readFileSync(written[0], 'utf8')).toBe('anon=abc');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/glue.test.ts`
Expected: FAIL ("Cannot find module").

- [ ] **Step 3: Write minimal implementation**

```ts
// src/engine/glue.ts
import type { Combo, Context } from '../types.js';
import { renderTemplates } from './renderer.js';

export function applyGlue(combo: Combo, ctx: Context): string[] {
  return renderTemplates(combo.dir, combo.glue, ctx);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/glue.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/engine/glue.ts test/glue.test.ts
git commit -m "feat(engine): combo glue applier"
```

---

### Task 6: Git initializer

**Files:**
- Create: `src/engine/git.ts`
- Test: `test/git.test.ts`

**Interfaces:**
- Consumes: nothing beyond node.
- Produces: `initGit(targetDir: string): void` — runs `git init`, `git add -A`, `git commit -m "chore: sydia-forge init"` in `targetDir` via `execFileSync`. Idempotent-safe: if already a repo, skip `init`.

- [ ] **Step 1: Write the failing test**

```ts
// test/git.test.ts
import { describe, it, expect } from 'vitest';
import { mkdtempSync, writeFileSync } from 'node:fs';
import { execFileSync } from 'node:child_process';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { initGit } from '../src/engine/git.js';

describe('initGit', () => {
  it('creates a repo with one commit', () => {
    const d = mkdtempSync(join(tmpdir(), 'sf-git-'));
    writeFileSync(join(d, 'file.txt'), 'x');
    initGit(d);
    const log = execFileSync('git', ['-C', d, 'log', '--oneline']).toString();
    expect(log).toContain('sydia-forge init');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/git.test.ts`
Expected: FAIL ("Cannot find module").

- [ ] **Step 3: Write minimal implementation**

```ts
// src/engine/git.ts
import { execFileSync } from 'node:child_process';
import { existsSync } from 'node:fs';
import { join } from 'node:path';

export function initGit(targetDir: string): void {
  const run = (...args: string[]) => execFileSync('git', ['-C', targetDir, ...args], { stdio: 'ignore' });
  if (!existsSync(join(targetDir, '.git'))) run('init');
  run('add', '-A');
  run('commit', '-m', 'chore: sydia-forge init');
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/git.test.ts`
Expected: PASS. (If CI lacks a git identity, the test env must set `GIT_AUTHOR_*`/`GIT_COMMITTER_*`; document in README.)

- [ ] **Step 5: Commit**

```bash
git add src/engine/git.ts test/git.test.ts
git commit -m "feat(engine): git initializer with first commit"
```

---

### Task 7: Deploy orchestrator (opt-in, injectable runner)

**Files:**
- Create: `src/engine/deploy.ts`
- Test: `test/deploy.test.ts`

**Interfaces:**
- Consumes: `Brick`, `Combo`, `Context`.
- Produces: `runDeploy(bricks: Brick[], combo: Combo | null, ctx: Context, load?: HookLoader): Promise<void>` where `type DeployHook = (ctx: Context) => Promise<void>` and `type HookLoader = (absPath: string) => Promise<{ default: DeployHook }>`. Runs each brick's `deploy` hook (in brick order), then the combo's hook if present. `load` defaults to dynamic `import()`; tests inject a mock. Does nothing when `ctx.deploy` is false.

- [ ] **Step 1: Write the failing test**

```ts
// test/deploy.test.ts
import { describe, it, expect, vi } from 'vitest';
import { runDeploy } from '../src/engine/deploy.js';
import type { Brick, Combo, Context } from '../src/types.js';

const ctx = (deploy: boolean): Context => ({ targetDir: '/t', phase: 'simple', topology: 'single-repo', apps: ['.'], answers: {}, deploy });
const brick = (id: string, deploy?: string): Brick => ({ id, kind: 'hosting', label: id, provides: [], consumes: [], prompts: [], templates: [], deploy, dir: `/bricks/${id}` });

describe('runDeploy', () => {
  it('does nothing when deploy is false', async () => {
    const load = vi.fn();
    await runDeploy([brick('vercel', 'deploy.js')], null, ctx(false), load as any);
    expect(load).not.toHaveBeenCalled();
  });

  it('runs brick hooks in order then the combo hook', async () => {
    const calls: string[] = [];
    const load = async (p: string) => ({ default: async () => { calls.push(p); } });
    const combo: Combo = { id: 'c', label: 'c', bricks: ['supabase', 'vercel'], phases: { simple: { topology: 'single-repo' }, complex: { topology: 'monorepo' } }, glue: [], deploy: 'deploy.js', dir: '/combos/c' };
    await runDeploy([brick('supabase', 'deploy.js'), brick('vercel', 'deploy.js')], combo, ctx(true), load as any);
    expect(calls).toEqual(['/bricks/supabase/deploy.js', '/bricks/vercel/deploy.js', '/combos/c/deploy.js']);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/deploy.test.ts`
Expected: FAIL ("Cannot find module").

- [ ] **Step 3: Write minimal implementation**

```ts
// src/engine/deploy.ts
import { join } from 'node:path';
import type { Brick, Combo, Context } from '../types.js';

export type DeployHook = (ctx: Context) => Promise<void>;
export type HookLoader = (absPath: string) => Promise<{ default: DeployHook }>;

const defaultLoader: HookLoader = (absPath) => import(absPath);

export async function runDeploy(bricks: Brick[], combo: Combo | null, ctx: Context, load: HookLoader = defaultLoader): Promise<void> {
  if (!ctx.deploy) return;
  for (const b of bricks) {
    if (b.deploy) { const mod = await load(join(b.dir, b.deploy)); await mod.default(ctx); }
  }
  if (combo?.deploy) { const mod = await load(join(combo.dir, combo.deploy)); await mod.default(ctx); }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/deploy.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/engine/deploy.ts test/deploy.test.ts
git commit -m "feat(engine): opt-in deploy orchestrator"
```

---

### Task 8: Bricks content (angular, supabase, vercel)

**Files:**
- Create: `bricks/angular/brick.json`, `bricks/angular/templates/{{app}}.vercel.json`, `bricks/angular/templates/README.md`
- Create: `bricks/supabase/brick.json`, `bricks/supabase/templates/config.toml`, `bricks/supabase/templates/0001_init.sql`, `bricks/supabase/deploy.ts`
- Create: `bricks/vercel/brick.json`, `bricks/vercel/deploy.ts`
- Test: `test/bricks.test.ts`

**Interfaces:**
- Consumes: `loadBrick` (Task 2).
- Produces: three on-disk bricks that load and validate. Each `deploy.ts` exports `default: DeployHook` (Task 7 type).

- [ ] **Step 1: Write the failing test**

```ts
// test/bricks.test.ts
import { describe, it, expect } from 'vitest';
import { join } from 'node:path';
import { loadBrick } from '../src/engine/manifest.js';

const root = join(process.cwd(), 'bricks');

describe('shipped bricks', () => {
  it('all load with expected kinds', () => {
    expect(loadBrick(join(root, 'angular')).kind).toBe('front');
    expect(loadBrick(join(root, 'supabase')).kind).toBe('backend');
    expect(loadBrick(join(root, 'vercel')).kind).toBe('hosting');
  });
  it('supabase declares a deploy hook and a migration template', () => {
    const b = loadBrick(join(root, 'supabase'));
    expect(b.deploy).toBe('deploy.ts');
    expect(b.templates.some(t => t.to.includes('migrations'))).toBe(true);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/bricks.test.ts`
Expected: FAIL (bricks not created).

- [ ] **Step 3: Write minimal implementation**

`bricks/angular/brick.json`:
```json
{
  "id": "angular", "kind": "front", "label": "Angular",
  "provides": ["spa"], "consumes": ["backend-client"],
  "prompts": [{ "id": "appName", "message": "Application name" }],
  "templates": [
    { "from": "templates/vercel.json", "to": "{{app}}/vercel.json" },
    { "from": "templates/README.md", "to": "{{app}}/README.md" }
  ]
}
```
`bricks/angular/templates/vercel.json`:
```json
{ "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }] }
```
`bricks/angular/templates/README.md`:
```md
# {{appName}}

Angular app scaffolded by sydia-forge. Deploy: push to the linked Vercel project.
```
`bricks/supabase/brick.json`:
```json
{
  "id": "supabase", "kind": "backend", "label": "Supabase",
  "provides": ["db", "auth", "backend-client"], "consumes": [],
  "prompts": [
    { "id": "supabaseUrl", "message": "Supabase project URL" },
    { "id": "anonKey", "message": "Supabase anon key" },
    { "id": "projectRef", "message": "Supabase project ref (for db push)", "when": "deploy" }
  ],
  "templates": [
    { "from": "templates/config.toml", "to": "supabase/config.toml" },
    { "from": "templates/0001_init.sql", "to": "supabase/migrations/0001_init.sql" }
  ],
  "deploy": "deploy.ts"
}
```
`bricks/supabase/templates/config.toml`:
```toml
# Supabase local config (managed by supabase CLI)
project_id = "{{appName}}"
```
`bricks/supabase/templates/0001_init.sql`:
```sql
-- initial migration (empty; add tables here). RLS must be enabled per table.
```
`bricks/supabase/deploy.ts`:
```ts
import { execFileSync } from 'node:child_process';
import type { Context } from '../../src/types.js';
export default async function deploy(ctx: Context): Promise<void> {
  const ref = ctx.answers.projectRef;
  execFileSync('supabase', ['link', '--project-ref', ref], { cwd: ctx.targetDir, stdio: 'inherit' });
  execFileSync('supabase', ['db', 'push'], { cwd: ctx.targetDir, stdio: 'inherit' });
}
```
`bricks/vercel/brick.json`:
```json
{
  "id": "vercel", "kind": "hosting", "label": "Vercel",
  "provides": ["hosting"], "consumes": ["spa"],
  "prompts": [], "templates": [], "deploy": "deploy.ts"
}
```
`bricks/vercel/deploy.ts`:
```ts
import { execFileSync } from 'node:child_process';
import type { Context } from '../../src/types.js';
export default async function deploy(ctx: Context): Promise<void> {
  // Relies on Vercel Git integration: push triggers the build.
  execFileSync('git', ['-C', ctx.targetDir, 'push'], { stdio: 'inherit' });
}
```

Also add `bricks/**` to `tsup` externals or a separate build so `deploy.ts` files are shipped as-is: set `"files": ["dist", "bricks", "combos"]` in `package.json`, and load hooks from source at runtime via a `tsx`/`node --import` note in README. (v1: deploy hooks run through `node` with `--experimental-strip-types` or are pre-compiled by `tsup` with entry globs `bricks/*/deploy.ts` and `combos/*/deploy.ts`.) Update `tsup.config.ts` entry to include them:
```ts
entry: ['src/cli.ts', 'bricks/*/deploy.ts', 'combos/*/deploy.ts'],
```
and have Task 7's default loader resolve `deploy.js` from `dist` (change brick `deploy` values to `deploy.js` at load time by stripping `.ts`). Adjust `manifest.ts` `loadBrick` to normalize: `deploy: m.deploy?.replace(/\.ts$/, '.js')`.

- [ ] **Step 4: Run test to verify it passes**

Run: `npm run build && npx vitest run test/bricks.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add bricks package.json tsup.config.ts src/engine/manifest.ts test/bricks.test.ts
git commit -m "feat(bricks): angular, supabase, vercel bricks with deploy hooks"
```

---

### Task 9: Combo content (angular-supabase) + glue

**Files:**
- Create: `combos/angular-supabase/combo.json`
- Create: `combos/angular-supabase/glue/environment.prod.ts.eta`, `combos/angular-supabase/glue/supabase.client.ts.eta`
- Create: `combos/angular-supabase/deploy.ts`
- Test: `test/combo.test.ts`

**Interfaces:**
- Consumes: `loadCombo` (Task 2), `resolveBricks` (Task 3), `applyGlue` (Task 5).
- Produces: the `angular-supabase` combo on disk; its glue renders `environment.prod.ts` containing the anon-key value and the Supabase client provider.

- [ ] **Step 1: Write the failing test**

```ts
// test/combo.test.ts
import { describe, it, expect } from 'vitest';
import { mkdtempSync, readFileSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { loadCombo } from '../src/engine/manifest.js';
import { loadBrick } from '../src/engine/manifest.js';
import { resolveBricks } from '../src/engine/resolver.js';
import { applyGlue } from '../src/engine/glue.js';
import type { Context } from '../src/types.js';

const combo = loadCombo(join(process.cwd(), 'combos/angular-supabase'));

describe('angular-supabase combo', () => {
  it('references the three shipped bricks', () => {
    const all = ['angular', 'supabase', 'vercel'].map(id => loadBrick(join(process.cwd(), 'bricks', id)));
    expect(resolveBricks(combo.bricks, all).map(b => b.id)).toEqual(['angular', 'supabase', 'vercel']);
  });
  it('glue writes the anon key into environment.prod.ts', () => {
    const target = mkdtempSync(join(tmpdir(), 'sf-c-'));
    const ctx: Context = { targetDir: target, phase: 'simple', topology: 'single-repo', apps: ['.'], answers: { supabaseUrl: 'http://x', anonKey: 'KEY123' }, deploy: false };
    const written = applyGlue(combo, ctx);
    const env = written.find(p => p.endsWith('environment.prod.ts'))!;
    expect(readFileSync(env, 'utf8')).toContain('KEY123');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/combo.test.ts`
Expected: FAIL (combo not created).

- [ ] **Step 3: Write minimal implementation**

`combos/angular-supabase/combo.json`:
```json
{
  "id": "angular-supabase", "label": "Angular + Supabase (Vercel)",
  "bricks": ["angular", "supabase", "vercel"],
  "phases": {
    "simple": { "topology": "single-repo" },
    "complex": { "topology": "monorepo", "apps": ["proto", "app"] }
  },
  "glue": [
    { "from": "glue/environment.prod.ts.eta", "to": "{{app}}/src/environments/environment.prod.ts" },
    { "from": "glue/supabase.client.ts.eta", "to": "{{app}}/src/app/core/supabase.client.ts" }
  ],
  "deploy": "deploy.ts"
}
```
`combos/angular-supabase/glue/environment.prod.ts.eta`:
```ts
export const environment = {
  production: true,
  supabaseUrl: '{{supabaseUrl}}',
  supabaseAnonKey: '{{anonKey}}',
};
```
`combos/angular-supabase/glue/supabase.client.ts.eta`:
```ts
import { createClient } from '@supabase/supabase-js';
import { environment } from '../../environments/environment.prod';
export const supabase = createClient(environment.supabaseUrl, environment.supabaseAnonKey);
```
`combos/angular-supabase/deploy.ts`:
```ts
import type { Context } from '../../src/types.js';
// Brick hooks already ran (supabase db push, then vercel git push).
// Combo-level deploy is a no-op placeholder for future cross-brick steps.
export default async function deploy(_ctx: Context): Promise<void> {}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/combo.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add combos test/combo.test.ts
git commit -m "feat(combo): angular-supabase combo with integration glue"
```

---

### Task 10: init command orchestration + generation pipeline

**Files:**
- Create: `src/commands/init.ts`
- Modify: `src/cli.ts` (wire `init` to the command)
- Test: `test/init.test.ts`

**Interfaces:**
- Consumes: everything above.
- Produces:
  - `runGeneration(opts: GenerationOptions): GenerationResult` where
    `type GenerationOptions = { targetDir: string; combo: Combo; bricks: Brick[]; ctx: Context; force: boolean }`
    and `type GenerationResult = { written: string[]; warnings: string[] }`. Pure pipeline: guard non-empty dir (throw unless `force`), render each brick's templates (skipping `when:"deploy"` prompt values already handled by caller), apply glue, emit forge scaffolding files (`CLAUDE.md`, `.claude/context/stack.md`), init git.
  - `initCommand(argv: string[]): Promise<void>` — parses `[dir] [--force]`, runs prompts (mode, phase, deploy y/n, brick prompts filtered by `when`), builds `Context`, calls `runGeneration`, then `runDeploy`.

- [ ] **Step 1: Write the failing test**

```ts
// test/init.test.ts
import { describe, it, expect } from 'vitest';
import { mkdtempSync, writeFileSync, existsSync, readFileSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { runGeneration } from '../src/commands/init.js';
import { loadCombo, loadBrick } from '../src/engine/manifest.js';
import { resolveBricks } from '../src/engine/resolver.js';
import type { Context } from '../src/types.js';

const combo = loadCombo(join(process.cwd(), 'combos/angular-supabase'));
const bricks = resolveBricks(combo.bricks, ['angular', 'supabase', 'vercel'].map(id => loadBrick(join(process.cwd(), 'bricks', id))));

function gen(topology: 'single-repo' | 'monorepo', apps: string[]) {
  const target = mkdtempSync(join(tmpdir(), 'sf-init-'));
  const ctx: Context = { targetDir: target, phase: topology === 'monorepo' ? 'complex' : 'simple', topology, apps, answers: { appName: 'demo', supabaseUrl: 'http://x', anonKey: 'KEY123' }, deploy: false };
  const res = runGeneration({ targetDir: target, combo, bricks, ctx, force: false });
  return { target, res };
}

describe('runGeneration', () => {
  it('single-repo: writes vercel.json, env with anon key, CLAUDE.md, and a git repo', () => {
    const { target } = gen('single-repo', ['.']);
    expect(existsSync(join(target, 'vercel.json'))).toBe(true);
    expect(readFileSync(join(target, 'src/environments/environment.prod.ts'), 'utf8')).toContain('KEY123');
    expect(existsSync(join(target, 'CLAUDE.md'))).toBe(true);
    expect(existsSync(join(target, '.git'))).toBe(true);
  });

  it('monorepo: writes per-app files under apps/', () => {
    const { target } = gen('monorepo', ['proto', 'app']);
    expect(existsSync(join(target, 'apps/proto/vercel.json'))).toBe(true);
    expect(existsSync(join(target, 'apps/app/vercel.json'))).toBe(true);
  });

  it('refuses a non-empty target dir without force', () => {
    const target = mkdtempSync(join(tmpdir(), 'sf-ne-'));
    writeFileSync(join(target, 'existing.txt'), 'x');
    const ctx: Context = { targetDir: target, phase: 'simple', topology: 'single-repo', apps: ['.'], answers: { appName: 'demo', supabaseUrl: 'u', anonKey: 'k' }, deploy: false };
    expect(() => runGeneration({ targetDir: target, combo, bricks, ctx, force: false })).toThrow(/not empty/i);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/init.test.ts`
Expected: FAIL ("Cannot find module '../src/commands/init.js'").

- [ ] **Step 3: Write minimal implementation**

```ts
// src/commands/init.ts
import { readdirSync, existsSync, mkdirSync, writeFileSync } from 'node:fs';
import { join } from 'node:path';
import * as p from '@clack/prompts';
import type { Brick, Combo, Context, Phase } from '../types.js';
import { loadBrick, loadCombo } from '../engine/manifest.js';
import { resolveBricks } from '../engine/resolver.js';
import { renderTemplates } from '../engine/renderer.js';
import { applyGlue } from '../engine/glue.js';
import { initGit } from '../engine/git.js';
import { runDeploy } from '../engine/deploy.js';

export interface GenerationOptions { targetDir: string; combo: Combo; bricks: Brick[]; ctx: Context; force: boolean; }
export interface GenerationResult { written: string[]; warnings: string[]; }

const BRICK_IDS = ['angular', 'supabase', 'vercel'];

function emitForgeScaffold(ctx: Context): string[] {
  const written: string[] = [];
  const claude = join(ctx.targetDir, 'CLAUDE.md');
  writeFileSync(claude, `# ${ctx.answers.appName ?? 'project'}\n\nAngular + Supabase project scaffolded by sydia-forge.\n`);
  written.push(claude);
  const ctxDir = join(ctx.targetDir, '.claude', 'context');
  mkdirSync(ctxDir, { recursive: true });
  const stack = join(ctxDir, 'stack.md');
  writeFileSync(stack, `# Stack\n\nFront: Angular. Backend: Supabase. Hosting: Vercel.\nTopology: ${ctx.topology}.\n`);
  written.push(stack);
  return written;
}

export function runGeneration(opts: GenerationOptions): GenerationResult {
  const { targetDir, combo, bricks, ctx, force } = opts;
  if (existsSync(targetDir) && readdirSync(targetDir).length > 0 && !force) {
    throw new Error(`Target directory is not empty: ${targetDir} (use --force)`);
  }
  mkdirSync(targetDir, { recursive: true });
  const written: string[] = [];
  for (const b of bricks) written.push(...renderTemplates(b.dir, b.templates, ctx));
  written.push(...applyGlue(combo, ctx));
  written.push(...emitForgeScaffold(ctx));
  initGit(targetDir);
  return { written, warnings: [] };
}

export async function initCommand(argv: string[]): Promise<void> {
  const force = argv.includes('--force');
  const dir = argv.find(a => !a.startsWith('--')) ?? '.';
  const targetDir = join(process.cwd(), dir);
  const combo = loadCombo(join(process.cwd(), 'combos/angular-supabase'));
  const allBricks = BRICK_IDS.map(id => loadBrick(join(process.cwd(), 'bricks', id)));
  const bricks = resolveBricks(combo.bricks, allBricks);

  const phase = (await p.select({ message: 'Project complexity', options: [
    { value: 'simple', label: 'Simple — proto hardens in place (single repo)' },
    { value: 'complex', label: 'Complex — proto + app (monorepo)' },
  ] })) as Phase;
  const cfg = combo.phases[phase];
  const apps = cfg.apps ?? ['.'];
  const deploy = (await p.confirm({ message: 'Deploy now (Supabase db push + Vercel git push)?' })) as boolean;

  const answers: Record<string, string> = {};
  for (const b of bricks) for (const q of b.prompts) {
    if (q.when === 'deploy' && !deploy) continue;
    answers[q.id] = (await p.text({ message: q.message })) as string;
  }

  const ctx: Context = { targetDir, phase, topology: cfg.topology, apps, answers, deploy };
  runGeneration({ targetDir, combo, bricks, ctx, force });
  await runDeploy(bricks, combo, ctx);
  p.outro('Done.');
}
```

Modify `src/cli.ts`:
```ts
import { initCommand } from './commands/init.js';
// ...
if (cmd === 'init') { await initCommand(process.argv.slice(3)); process.exit(0); }
```
(Make the top-level of `cli.ts` an `async` IIFE so `await` is legal.)

- [ ] **Step 4: Run test to verify it passes**

Run: `npm run build && npx vitest run test/init.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/commands/init.ts src/cli.ts test/init.test.ts
git commit -m "feat(init): generation pipeline and interactive command"
```

---

### Task 11: End-to-end integration test + README

**Files:**
- Create: `test/e2e.test.ts`
- Create: `README.md` (usage, git identity note for CI)

**Interfaces:**
- Consumes: `runGeneration` and the shipped bricks/combo.
- Produces: an e2e test asserting the full generated tree for both phases; user-facing README.

- [ ] **Step 1: Write the failing test**

```ts
// test/e2e.test.ts
import { describe, it, expect } from 'vitest';
import { mkdtempSync, existsSync, readFileSync } from 'node:fs';
import { execFileSync } from 'node:child_process';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { runGeneration } from '../src/commands/init.js';
import { loadCombo, loadBrick } from '../src/engine/manifest.js';
import { resolveBricks } from '../src/engine/resolver.js';
import type { Context } from '../src/types.js';

const combo = loadCombo(join(process.cwd(), 'combos/angular-supabase'));
const bricks = resolveBricks(combo.bricks, ['angular', 'supabase', 'vercel'].map(id => loadBrick(join(process.cwd(), 'bricks', id))));

describe('e2e init', () => {
  it('simple phase produces a committed, buildable tree', () => {
    const target = mkdtempSync(join(tmpdir(), 'sf-e2e-'));
    const ctx: Context = { targetDir: target, phase: 'simple', topology: 'single-repo', apps: ['.'], answers: { appName: 'demo', supabaseUrl: 'http://x', anonKey: 'KEY123' }, deploy: false };
    runGeneration({ targetDir: target, combo, bricks, ctx, force: false });
    // vercel SPA rewrite present
    expect(readFileSync(join(target, 'vercel.json'), 'utf8')).toContain('index.html');
    // supabase migration present
    expect(existsSync(join(target, 'supabase/migrations/0001_init.sql'))).toBe(true);
    // git has exactly one commit
    const log = execFileSync('git', ['-C', target, 'log', '--oneline']).toString().trim().split('\n');
    expect(log.length).toBe(1);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/e2e.test.ts`
Expected: FAIL until README exists is irrelevant; test should PASS already if Tasks 8–10 done. If it fails, fix the responsible task. (Write the README regardless in Step 3.)

- [ ] **Step 3: Write minimal implementation**

`README.md`:
```md
# sydia-forge

Brick-driven project generator. `npx sydia-forge init` scaffolds an
Angular + Supabase project (single-repo or monorepo) ready for Vercel Hobby
+ Supabase Free.

## Usage
    npx sydia-forge init [dir] [--force]

## Bricks & combos
A **brick** is one tech concern (`bricks/<id>/brick.json` + templates + optional
`deploy.ts`). A **combo** is a preset of bricks plus integration glue
(`combos/<id>/`). v1 ships the `angular-supabase` combo.

## CI note
Tests run `git commit`; set a git identity in CI:
    git config --global user.email ci@example.com
    git config --global user.name CI
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm run build && npm test`
Expected: PASS (all suites).

- [ ] **Step 5: Commit**

```bash
git add test/e2e.test.ts README.md
git commit -m "test(e2e): full init tree assertions + README"
```

---

## Self-Review

**Spec coverage:** brick/combo/glue model → Tasks 2,3,5,8,9; hybrid engine (declarative + deploy.ts) → Tasks 7,8; both phases/topology → Tasks 4,9,10,11; git at step 0 → Task 6,10; forge scaffolding (CLAUDE.md/.claude) → Task 10; runtime choices → Task 1; error handling (non-empty dir, missing brick, malformed manifest) → Tasks 10,3,2; deploy opt-in + separate phase → Tasks 7,10; testing strategy → all tasks + Task 11. Non-goals (capability negotiation, other stacks, real CI deploy) respected. `openspec/` emission from the spec's step 8 is deferred (documented gap: v1 emits CLAUDE.md + .claude/context only; add `openspec/` when OpenSpec brick lands).

**Placeholder scan:** no TBD/TODO/"add error handling" left; all code steps contain real code.

**Type consistency:** `Brick`/`Combo`/`Context` used identically across tasks; `DeployHook`/`HookLoader` defined in Task 7 and reused in Tasks 8/9; `runGeneration`/`GenerationOptions` defined in Task 10 and reused in Task 11; `renderTemplates` signature stable Tasks 4→5,10.

**Review Focus:** all five lines have owning tasks with tests (non-empty dir → Task 10; unknown brick → Task 3; malformed manifest → Task 2; monorepo path prefixing → Tasks 4 & 10; deploy declined → Tasks 7 & 10).
