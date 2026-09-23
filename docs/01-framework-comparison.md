# Framework Comparison

Analysis of AI development frameworks and their fit in a modular meta-framework.

## BMAD (Breakthrough Method of Agile AI-Driven Development)

**Layer:** Workflow — phases, personas, handoffs  
**Maintained by:** Community (GitHub)  
**Maturity:** Medium

BMAD structures AI-driven development around specialized agent personas (Analyst, PM, Architect, Developer, QA) that produce concrete artifacts (PRD, stories, ADR, QA reports) and hand off to each other through defined phases.

**Unique value:** The only framework that covers the full workflow cycle with formalized handoffs between phases. Designed for small teams amplified by AI — counter-intuitively, fewer humans = more need for structure, not less.

**Limitations:** Designed for from-scratch projects with formal sprint cycles. Adopting it mid-project requires refactoring to match its role model.

**Integration:** Use BMAD's workflow and personas; implement agents as thin ECC-style shells that read context files rather than fat embedded-knowledge agents.

---

## ECC (Everything Claude Code)

**Layer:** Infrastructure — how agents are built  
**Maintained by:** Community patterns  
**Maturity:** Medium

ECC is a methodology (not an installable tool) for organizing Claude Code projects: thin agent definitions that read shared context files, official skills for procedural knowledge, hooks for automation, memory for persistence.

**Unique value:** The only layer that defines HOW to build the infrastructure. Complements BMAD's WHAT with the HOW.

**Limitations:** Not installable — it's a set of patterns. Value depends on the team's discipline.

---

## OpenSpec

**Layer:** Planning — feature lifecycle management  
**Maintained by:** Community  
**Maturity:** Medium

OpenSpec manages the full feature lifecycle through structured artifacts: proposal → spec (delta) → design → tasks → implementation → archive. All versioned alongside the code.

**Unique value:** Versioned decision trail that grows with the codebase. Neither BMAD nor ECC cover this.

**Integration:** Use as the planning layer between BMAD's Analyst phase and Developer phase.

---

## SpecKit

**Layer:** Planning — spec format standard  
**Maintained by:** Small community  
**Maturity:** Low

Standardized spec document format (context, requirements, acceptance criteria, constraints) designed for direct agent consumption.

**Unique value:** The spec format idea. The actual tool is immature.

**Integration:** Use as inspiration for context file format, not as a formal dependency.

---

## Kiro (Amazon)

**Layer:** Implementation — IDE-integrated spec generation  
**Maintained by:** Amazon  
**Maturity:** Medium (preview)

VS Code extension that auto-generates specs from natural language, with steering rules (persistent project instructions) and real-time validation.

**Integration:** Not recommended — proprietary, IDE-locked, competes with Claude Code. Steering rules = CLAUDE.md done by Amazon.

---

## CodeGraph

**Layer:** Architecture — codebase understanding  
**Maintained by:** Commercial (codegraph.io)  
**Maturity:** Medium

Builds a semantic knowledge graph of the codebase: call paths, dependency trees, impact radius. Answers "what calls X", "what would break if I change Y" more accurately than grep.

**Integration:** Architecture layer. Activate for projects with >50 files where grep becomes unreliable.

---

## GSD (Get Shit Done)

**Layer:** N/A — anti-pattern for meta-frameworks  
**Integration:** Not recommended as a framework. Use its philosophy (bias toward shipping) as a calibration signal, not a structure.

---

## Compatibility matrix

| | BMAD | ECC | OpenSpec | CodeGraph | claude-mem |
|---|---|---|---|---|---|
| BMAD | — | ✅ | ✅ | ✅ | ✅ |
| ECC | ✅ | — | ✅ | ✅ | ✅ |
| OpenSpec | ✅ | ✅ | — | ⚠️ | ✅ |
| CodeGraph | ✅ | ✅ | ⚠️ | — | ✅ |
| claude-mem | ✅ | ✅ | ✅ | ✅ | — |

All five are complementary. No conflicts. The full stack is: BMAD (workflow) + ECC (infrastructure) + OpenSpec (planning) + CodeGraph (architecture) + claude-mem (memory).
