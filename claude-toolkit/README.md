# Claude Code Toolkit

A reusable collection of Claude Code agents and skills for software development workflows.
Project-agnostic — works with any codebase as long as a `CLAUDE.md` is present.

---

## Installation

Copy the `agents/` and `skills/` directories into your project's `.claude/` folder:

```bash
# Clone this repo
git clone https://github.com/gabrielfst30/super-gabriel-claude-toolkit.git

# Copy into your project
cp -r super-gabriel-claude-toolkit/agents your-project/.claude/
cp -r super-gabriel-claude-toolkit/skills your-project/.claude/
```

Your `.claude/` should look like:

```
.claude/
├── agents/
│   ├── architecture-guardian.md
│   ├── nextjs-react-expert.md
│   ├── task-executor.md
│   ├── task-verifier.md
│   ├── typescript-error-fixer.md
│   ├── prisma-migrator.md
│   ├── natspec-writer.md
│   └── natspec-validator.md
└── skills/
    ├── enhance/
    ├── planner/
    ├── execute-todo/
    ├── system-verifier/
    ├── create-skill-agent/
    ├── natspec/
    └── prisma-postgresql-schema/
```

**Requirement:** Create a `CLAUDE.md` at the project root. The `architecture-guardian`,
`task-executor`, and `system-verifier` agents all read it to derive project-specific rules.

---

## Agents

Agents are specialized Claude subagents invoked programmatically — not by the user directly.
They run in isolated context with restricted tool access.

| Agent | Model | Role |
|---|---|---|
| `architecture-guardian` | opus | Validates code adherence to `CLAUDE.md` conventions |
| `nextjs-react-expert` | opus | Expert in Next.js App Router, React, TypeScript patterns |
| `task-executor` | opus | Implements a single atomic task from a todo spec |
| `task-verifier` | opus | Verifies a completed task against requirements and architecture |
| `typescript-error-fixer` | opus | Fixes TypeScript/React/Next.js type errors |
| `prisma-migrator` | sonnet | Executes Prisma CLI commands (migrate, generate, db pull) |
| `natspec-writer` | opus | Writes NatSpec-style docs for a single file in any language |
| `natspec-validator` | haiku | Validates NatSpec completeness (read-only) |

---

## Skills

Skills are workflows invoked by the user via `/slash-commands` or triggered by keywords.

### `/enhance [request-name]`
Transforms a raw development request into a structured `task.md` spec.
Reads `developer/requests/[name].md` or an inline description, consults
`architecture-guardian`, and creates `developer/tasks/[name]/task.md`.

### `/planner [task-name]`
Decomposes a `task.md` spec into atomic, executable tasks with IDs and
dependency mapping. Creates `MAIN.md` + `todo/*.md` files for `/execute-todo`.

### `/execute-todo [task-name]`
Orchestrates parallel task execution. Reads the plan, launches `task-executor`
agents respecting dependencies, verifies each with `task-verifier`, and runs
type-check at the end.

### `/system-verifier`
Full architectural audit. Runs type-check + lint, then spawns parallel
`architecture-guardian` agents per domain. Aggregates into a unified report.

### `/natspec [file or directory]`
Generates NatSpec-style documentation comments for any language (TypeScript,
JavaScript, Solidity, Python, Rust, Go). Parallelizes by file, validates output.

### `/create-skill-agent [description]`
Creates a new skill+agent combination from a workflow description. Follows the
architecture formula in `skills/create-skill-agent/references/architecture-formula.md`.

### `prisma-postgresql-schema` (keyword-triggered)
Handles Prisma schema design, analysis, migration, and type generation.
Triggers on: `prisma`, `schema.prisma`, `prisma model`, `prisma migrate`, etc.

---

## Standard Workflow

```
1. Describe the feature → /enhance feature-name
   Creates: developer/tasks/feature-name/task.md

2. Plan the implementation → /planner feature-name
   Creates: developer/tasks/feature-name/MAIN.md
            developer/tasks/feature-name/todo/*.md

3. Execute in parallel → /execute-todo feature-name
   Runs: task-executor agents (parallel where possible)
         task-verifier agents (after each task)
         type-check (at the end)

4. Audit the result → /system-verifier
   Checks: architecture compliance, type errors, lint issues
```

---

## Architecture Formula

Skills and agents follow a strict separation:

```
SKILL  = orchestrates + decides + has conversation context
AGENT  = executes + is isolated + does one thing
```

Four layers (top-down only — no upward invocation):

```
Layer 1: User → /slash-commands (Skills)
Layer 2: Orchestration (Skills) — reads plans, decides order
Layer 3: Execution (Agents) — implements, writes files, runs CLI
Layer 4: Specialists (Agents) — audits, validates, reads only
```

See `skills/create-skill-agent/references/architecture-formula.md` for the full guide.

---

## CLAUDE.md Requirements

The following agents read `CLAUDE.md` to derive project rules:
- `architecture-guardian` — reads it for validation checklist
- `task-executor` — reads it for coding conventions
- `task-verifier` — reads it for architecture compliance checks
- `system-verifier` — reads it to identify project domains

Minimum recommended `CLAUDE.md` sections:
- Stack and technology versions
- Directory structure
- Import conventions (aliases, relative paths rules)
- Mandatory patterns (error handling, exports, etc.)
- Prohibited patterns
- Build/lint/type-check commands

---

## Adding New Skills and Agents

Use `/create-skill-agent [description]` — it reads the architecture formula
and generates well-structured files following all conventions.

Or follow the checklist manually:

**For each agent:**
- [ ] `name` is unique, in kebab-case
- [ ] `description` includes specific keywords for auto-matching
- [ ] `model` is proportional to task complexity (haiku/sonnet/opus)
- [ ] `allowed-tools` is the absolute minimum needed
- [ ] Response format is documented in the body

**For each skill:**
- [ ] `name` is unique, in kebab-case
- [ ] `description` includes trigger keywords
- [ ] `user-invocable: true` if user-facing
- [ ] `allowed-tools` includes `Agent` if it spawns subagents
- [ ] Workflow is numbered steps — no ambiguity
