# sydia-forge

> A modular meta-framework for AI-driven software development — from rapid prototype to production delivery.

**The idea:** stop rebuilding the same AI infrastructure on every project. Instead, assemble maintained community bricks into a personal system you deploy once and configure per-project.

Like using Symfony to build a product — you don't rewrite the framework, you configure your layer on top of it.

---

## The problem

Every AI-driven project ends up rebuilding the same things:
- Agent definitions
- Context injection systems
- Workflow orchestration
- Spec management
- Git discipline

This is infrastructure work, not product work. sydia-forge externalizes it.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    YOUR PROJECT                         │
│         context files · domain skills · ADRs           │
├─────────────────────────────────────────────────────────┤
│                   sydia-forge                           │
│         onboarding generator · brick selector          │
├──────────┬──────────┬──────────┬───────────────────────┤
│  BMAD    │   ECC    │ OpenSpec │  MCP ecosystem         │
│ workflow │ infra    │ planning │  extensions            │
├──────────┴──────────┴──────────┴───────────────────────┤
│              community-maintained bricks                │
└─────────────────────────────────────────────────────────┘
```

**You only maintain:** your project delta (context files, domain conventions).  
**The community maintains:** everything below.

---

## Brick map

### Prototype
| Brick | Role | Source |
|---|---|---|
| Lovable | Full app from a prompt (React + Supabase) | lovable.dev |
| v0.dev | UI components from a prompt | Vercel |
| Bolt.new | Fullstack generative with live preview | StackBlitz |

### Planning
| Brick | Role | Source |
|---|---|---|
| BMAD (Analyst/PM) | PRD, stories, structured backlog | github.com/bmad |
| OpenSpec | Feature lifecycle versioned with code | openspec |
| claude-mem | Cross-session and cross-project memory | claude-mem |

### Architecture
| Brick | Role | Source |
|---|---|---|
| BMAD (Architect) | Architecture decisions, ADR templates | github.com/bmad |
| CodeGraph | Code knowledge graph, call paths, impact radius | codegraph.io |
| RepoPrompt | Token-efficient codebase context for LLMs | open source |

### Implementation
| Brick | Role | Source |
|---|---|---|
| ECC | Thin agents + hooks + context files + official skills | community patterns |
| BMAD (Developer) | Story-driven development personas | github.com/bmad |
| MCP ecosystem | Claude Code extensions (Supabase, GitHub, Linear…) | community |
| Conventional Commits | Git discipline readable by agents and humans | conventionalcommits.org |

### Quality
| Brick | Role | Source |
|---|---|---|
| BMAD (QA) | QA persona with validation checklist | github.com/bmad |
| CodeRabbit | Automated inline PR review | coderabbit.ai |
| AgentOps | Agent observability (traces, costs, errors) | agentops.ai |

### Delivery
| Brick | Role | Source |
|---|---|---|
| OpenSpec (archive) | Versioned decisions alongside shipped code | openspec |
| MCP GitHub | PR, releases, changelogs from Claude Code | GitHub |

---

## Modular combinations

```
SOLO FAST     → Lovable + ECC + Conventional + MCP GitHub
SOLO FULL     → + OpenSpec + CodeGraph + BMAD Developer
TEAM STANDARD → + BMAD (Analyst + Architect + QA) + claude-mem
TEAM FULL     → + CodeRabbit + AgentOps + Linear
```

---

## What sydia-forge builds (the unique layer)

Every brick above is maintained externally. The only thing sydia-forge adds is the **onboarding generator** — the glue that installs and configures the right bricks for your project:

```bash
npx sydia-forge init

# → "What's your stack?" → generates context/front.md, context/back.md
# → "Solo or team?"     → activates/deactivates team bricks
# → "Prototype or prod?" → selects the right brick combination
# → Deploys BMAD + ECC + selected bricks → 0 infrastructure to maintain
```

---

## Status

🔬 **Concept phase** — research complete, design in progress.

See [`docs/`](./docs/) for the full analysis.

---

## Contributing

This is a personal meta-framework. Feedback and discussion welcome via issues.
