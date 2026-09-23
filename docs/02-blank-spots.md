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
