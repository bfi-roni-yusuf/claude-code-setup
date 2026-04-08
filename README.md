# Claude Code Setup

My [Claude Code](https://claude.com/product/claude-code) agentic workflow setup for Java/Spring Boot development on [bravo-bpm-service](https://github.com/bfi-finance/bravo-bpm-service) — a Camunda BPM orchestration service at BFI Finance.

This repo contains the full configuration: project instructions, custom skills, automated hooks, specialized agents, and workflow documentation. See the [Confluence post](https://bfifinance.atlassian.net/wiki/spaces/~638eb37f77acd224b3428329/pages/2331574276) for the narrative walkthrough.

## What's Inside

```
.
├── CLAUDE.md                         # Project instruction file (354 lines)
├── settings.local.json               # Hook definitions (17 hooks)
├── skills/                           # 21 skill definitions (17 custom + 3 baseline + 1 sub-skill)
│   ├── analyze-task/SKILL.md         # Interactive requirement analysis
│   ├── start-task/SKILL.md           # Branch creation + plan dispatch
│   ├── create-plan/SKILL.md          # Implementation plan generation
│   ├── review-plan/SKILL.md          # Plan validation
│   ├── create-pr/SKILL.md            # Draft PR from changes + plan
│   ├── review-pr/SKILL.md            # Multi-agent PR review
│   ├── address-review/SKILL.md       # PR review comment handler
│   ├── bpm-conventions/SKILL.md      # Java/Spring convention reference
│   ├── test-branch/SKILL.md          # Branch testing (REST + unit)
│   ├── create-jira-task/SKILL.md     # Jira issue creation
│   ├── fetch-jira-context/SKILL.md   # Jira/TRD sub-skill
│   ├── create-migration/SKILL.md     # Flyway migration generator
│   ├── generate-seed-data/SKILL.md   # SQL INSERT generator
│   ├── call-api/SKILL.md             # Local API testing
│   ├── cleanup-feature-flag/SKILL.md # Feature flag removal
│   ├── fix-dto/SKILL.md              # DTO annotation fixer
│   ├── update-workflow/SKILL.md      # Workflow status updater
│   ├── workflow/SKILL.md             # Workflow status viewer
│   ├── java-springboot/SKILL.md      # Spring Boot baseline
│   ├── java-junit/SKILL.md           # JUnit 5 baseline
│   └── java-docs/SKILL.md           # Javadoc baseline
├── agents/                           # 4 specialized agents
│   ├── plan-creator.md               # Autonomous plan generation
│   ├── plan-reviewer.md              # Plan validation
│   ├── implementation-reviewer.md    # Post-implementation review
│   └── migration-reviewer.md         # Flyway SQL review
└── docs/
    └── agent-workflow.md             # 10-phase workflow reference
```

## The Workflow

A 10-phase development cycle from Jira ticket to merged PR:

| Phase | Command | What Happens |
|-------|---------|--------------|
| 0 | `/analyze-task` | Requirement analysis (during/after grooming) |
| 1 | `/start-task` | Branch + plan-creator agent dispatch |
| 2 | Review plan | Approve or revise the generated plan |
| 3 | Execute | Step-by-step implementation |
| 4 | `/simplify` | Code quality review (built-in) |
| 5 | `/test-branch` | REST + unit testing |
| 6 | `/create-pr` | Draft PR generation |
| 7 | `/review-pr` | 6+ parallel review agents |
| 8 | `/address-review` | Handle review findings |
| - | `/auto-revise-claude-md` | Runs every session — captures learnings |

## Setup Overview

| Component | Count | Purpose |
|-----------|-------|---------|
| CLAUDE.md | 354 lines | Project conventions, architecture, testing rules |
| Skills | 25 (17 custom + 3 baseline + 5 global) | Reusable slash commands for workflow phases |
| Hooks (PreToolUse) | 6 | Block dangerous actions before they happen |
| Hooks (PostToolUse) | 11 | Warn about convention violations after writes |
| Agents | 4 | Autonomous specialists (planning, review, migration) |

## How to Use This Repo

This setup is **project-specific** — it's built for a Java/Spring Boot + Camunda BPM codebase. You can't copy-paste it into a different project and expect it to work. Instead, use it as a reference for building your own:

1. **Start with `CLAUDE.md`** — document your project's conventions, gotchas, and patterns
2. **Add hooks as you notice repeated mistakes** — each hook here started as a correction I kept making
3. **Build skills for your workflow** — start simple (e.g., `/create-pr`), iterate as you use them
4. **Agents come last** — only add them when a task benefits from isolation and autonomy

## Context

- **Project**: bravo-bpm-service (Java 17, Spring Boot 3.5.x, Camunda Platform 7)
- **Company**: BFI Finance (officially uses GitHub Copilot — this is a personal experiment)
- **Timeline**: ~7 weeks of iterative development (Feb–Apr 2026)
