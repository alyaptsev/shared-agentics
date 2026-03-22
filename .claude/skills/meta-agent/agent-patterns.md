# Agent Design Patterns

A decision-oriented reference for creating Claude Code agent definitions. Each section covers a specific aspect of agent design with concrete guidelines.

---

## 1. Agent types

| Type | Purpose | Context | Output | Model |
|------|---------|---------|--------|-------|
| **Worker** | Executes one specific task | Isolated — gets only what it needs | Returns summary to caller | Sonnet (default) or Haiku |
| **Orchestrator** | Coordinates other agents | Reads workflow state, delegates tasks | Aggregated results + status report | Opus |
| **Researcher** | Gathers and synthesizes information | Broad read access, web tools | Structured findings document | Sonnet |
| **Reviewer** | Evaluates output against criteria | Receives artifact + rubric | Pass/fail + specific feedback | Sonnet |
| **Transformer** | Converts data between formats | Input format + output spec | Reformatted data | Haiku or Sonnet |

**Default**: most agents are workers. Only create an orchestrator when you need dynamic delegation across multiple agents.

## 2. Agent definition structure

Every agent definition (`.claude/agents/<name>.md`) follows this skeleton:

```markdown
# Agent Name

[Role — one line, specific domain and specialization]

## Task

[What this agent does — single clear goal]

## Input

[What this agent receives — files to read, data format, context]

## Output

[What this agent produces — format, file path, structure]

## Tools

[Which tools this agent may use — be explicit]
- Read, Glob, Grep — for research agents
- Read, Write, Edit — for generation agents
- Bash — only when shell execution is required
- Agent — only for orchestrators
- WebSearch, WebFetch — only when external data is needed

## Constraints

[Boundaries — what this agent must NOT do, with reasons]

## Escalation

[When to stop and return to caller — uncertainty, scope creep, errors]
```

## 3. Model selection

| Factor | Opus | Sonnet | Haiku |
|--------|------|--------|-------|
| **Reasoning depth** | Deep analysis, multi-step planning | Standard analysis, generation | Classification, formatting |
| **Cost** | ~15× Haiku | ~5× Haiku | 1× (baseline) |
| **Speed** | Slowest | Medium | Fastest |
| **Context window** | 1M tokens | 1M tokens | 200k tokens |
| **When** | Orchestrators, complex reasoning, lead agents | Most worker agents, research, generation | Routing, validation, simple transforms |

**Rule**: start with the cheapest model that handles the task. Upgrade only when quality is insufficient.

## 4. Worker agent design

### Principles

1. **One goal** — if you can't describe the task in one sentence, split into multiple agents
2. **Explicit input** — list every file/data source the agent needs. Don't assume it knows the project
3. **Defined output** — specify format, location, and structure. "Write a report" is too vague
4. **Minimal tools** — grant only the tools needed for the task. Read-only agents don't need Write
5. **Clear boundaries** — what the agent should NOT do is as important as what it should do

### Input specification

```markdown
## Input

Before starting, read:
1. `path/to/file.md` — [what to look for in this file]
2. `path/to/data/` — [which files in this directory, what pattern]
3. The task description provided by the caller: [what variable data comes in]
```

### Output specification

```markdown
## Output

Write results to `path/to/output.md` in this format:

### Summary
[2-3 sentences — what was found/produced]

### Details
[structured content — tables, lists, sections as appropriate]

### Issues
[anything that needs human attention — flagged items, uncertainties]
```

## 5. Orchestrator agent design

### Principles

1. **Read and route only** — an orchestrator must never do the actual work. It reads state, decides what to do, and delegates
2. **Explicit delegation** — each delegation includes: goal, input, output format, boundaries
3. **State tracking** — maintains a progress file showing what's done, what's in progress, what's next
4. **Error handling** — defines what happens when a worker fails (retry, alternative, escalate)
5. **Completion criteria** — knows when the workflow is done

### Delegation template

When the orchestrator spawns a worker, it should provide:

```
Goal: [one sentence]
Input files: [explicit paths]
Output: write to [path] in [format]
Constraints: [boundaries]
When done: return a summary of what you produced
```

### State tracking

```markdown
## Progress

| Step | Agent | Status | Output |
|------|-------|--------|--------|
| 1. Research | research-agent | done | pipeline/STEP-01-research.md |
| 2. Draft | writer-agent | in progress | pipeline/STEP-02-draft.md |
| 3. Review | reviewer-agent | pending | — |
```

## 6. Common constraints by agent type

### Worker constraints
```markdown
## Constraints

- Do not modify files outside your output path — you own only `path/to/output.md`
- Do not make assumptions about data you haven't read — if a file is missing, report it
- Do not attempt tasks outside your goal — if you discover related work, note it in Issues and return
- Stay within [N] tool calls — if the task requires more, something is wrong
```

### Orchestrator constraints
```markdown
## Constraints

- Do not perform work directly — delegate to worker agents
- Do not spawn more than [N] agents in parallel — resource constraint
- Do not retry a failed agent more than 2 times — escalate to the user after that
- Do not modify worker output files — they are owned by their respective agents
```

### Reviewer constraints
```markdown
## Constraints

- Do not rewrite the artifact — only evaluate and provide feedback
- Score against the provided rubric only — do not invent criteria
- If the artifact passes all criteria, say so briefly — don't find problems where there are none
```

## 7. Naming conventions

| Component | Format | Example |
|-----------|--------|---------|
| Agent file | `kebab-case.md` | `content-writer.md` |
| Output files | `{PIPELINE}-{STEP-NN}-{name}.md` | `weekly-content/STEP-01-research.md` |
| Agent role in prompt | Specific, one line | "You are a technical writer specializing in API documentation for developer audiences" |

## 8. Escalation patterns

| Situation | Agent should |
|-----------|-------------|
| Missing input file | Report which file is missing and return |
| Ambiguous instructions | List interpretations and ask caller to clarify |
| Task too large | Suggest decomposition and return |
| Repeated failures (2+) | Stop retrying, report what's failing and why |
| Out-of-scope discovery | Note it in Issues section, continue with main task |
| Uncertainty about correctness | Flag with `[UNCERTAIN: reason]`, don't guess |

## 9. Anti-patterns

| Anti-pattern | Problem | Fix |
|-------------|---------|-----|
| **Swiss army agent** | One agent does research, writes, reviews, publishes | Split into focused workers |
| **Implicit input** | "Use the project context" without specifying files | List every file path explicitly |
| **God orchestrator** | Orchestrator also does generation work | Orchestrator only reads, decides, delegates |
| **Tool buffet** | Agent has access to all tools "just in case" | Grant only needed tools |
| **Vague output** | "Write a good report" | Show template with exact structure |
| **No escalation** | Agent guesses when unsure | Define uncertainty handling |
| **Shared files** | Two agents write to same file | Each file has one owner |
| **Opus everywhere** | All agents run on Opus | Use cheapest model that works |

## 10. Pre-send checklist

Before presenting an agent definition:

1. [ ] Agent has one clear goal — describable in one sentence
2. [ ] Role is specific (not "expert" but "who, in what domain, for what audience")
3. [ ] Every input file is listed by path with what to look for
4. [ ] Output format shown with template — not described in words
5. [ ] Tools are minimal — only what's needed for the task
6. [ ] Constraints include file ownership (what the agent may and may not write)
7. [ ] Escalation rules cover: missing input, ambiguity, failure, out-of-scope
8. [ ] Model choice is the cheapest that handles the task
9. [ ] For orchestrators: delegation template is explicit, state tracking is defined
10. [ ] Agent doesn't duplicate an existing skill or agent in the project
