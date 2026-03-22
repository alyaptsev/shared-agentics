# Workflow Design Patterns

A decision-oriented reference for designing multi-agent workflows in Claude Code. Each pattern has selection criteria, structure, and trade-offs.

---

## 1. Pattern selection matrix

Start here. Match the task to the simplest pattern that handles it.

| Pattern | When to use | Structure | Trade-off |
|---------|-------------|-----------|-----------|
| **Sequential chain** | Task decomposes into ordered steps, each needs the previous result | Step → Gate → Step → Gate → Output | High reliability; grows latency linearly |
| **Routing** | Input falls into distinct categories needing different handling | Classifier → Branch A / Branch B / ... | Clean separation; needs clear categories |
| **Parallelization (sectioning)** | Independent subtasks with no data dependency | Fork → [Task A, Task B, Task C] → Join | Fast; results need aggregation logic |
| **Parallelization (voting)** | Need quality through redundancy | Fork → [Same task ×N] → Select best | N× cost; good for creative tasks |
| **Orchestrator-workers** | Subtasks not known in advance — LLM decides decomposition | Orchestrator → dynamic spawn → aggregate | Flexible; harder to predict cost and behavior |
| **Evaluator-optimizer** | Quality matters more than speed; clear evaluation criteria exist | Generator → Evaluator (score + feedback) → loop | Catches errors; adds latency per iteration |

**Default rule**: start with sequential chain. Add parallelization where steps are independent. Add evaluator-optimizer where quality is critical. Use orchestrator-workers only when the decomposition itself requires reasoning.

## 2. Claude Code building blocks

Each workflow step maps to one of these execution mechanisms:

| Block | What it is | When to use | Context |
|-------|------------|-------------|---------|
| **Skill** (`/name`) | In-conversation instructions loaded on demand | Specialized generation tasks (writing, analysis, formatting) | Shares the main conversation context |
| **Agent** (subagent) | Isolated subprocess with its own context window | Heavy tasks, parallel work, tasks needing fresh context | Returns only a summary to caller |
| **Hook** (shell command) | Deterministic code triggered at lifecycle points | Validation, format checks, auto-injection, blocking rules | No LLM involved — pure code |
| **CLAUDE.md routing** | Static rules in project config | Directing Claude to skills/agents based on task type | Always loaded, costs context tokens |
| **Main conversation** | Direct Claude interaction | Simple tasks, coordination, final review | Full context available |

**Key constraint**: two agents must never write to the same file. If they need shared state, use a handoff file with clear ownership per step.

## 3. Step specification format

Every step in a workflow needs these fields:

```
### Step N: [name]
Goal: [one sentence — what this step achieves]
Executor: [skill / agent / hook / main]
Model: [opus / sonnet / haiku — only for agent steps]
Input: [what this step reads — files, prior step output, user input]
Output: [what this step produces — file path or data format]
Gate: [what must be true before proceeding to next step]
On failure: [what happens if the gate fails — retry / alternative / escalate]
```

## 4. Context passing

How data flows between steps in Claude Code file-based workflows:

| Method | When | Format |
|--------|------|--------|
| **File handoff** | Between agents (isolated contexts) | Step N writes → file → Step N+1 reads |
| **In-conversation** | Between skills (shared context) | Previous skill output is in conversation history |
| **Structured file** | When downstream steps need specific fields | YAML/JSON frontmatter in markdown |
| **Summary return** | Agent → caller | Agent produces detailed work, returns condensed summary |

**File naming convention**: `{pipeline-name}/{STEP-NN}-{step-name}.md`

## 5. Gate types

| Gate | Checks | Implementation |
|------|--------|---------------|
| **Format gate** | Structure, length, required fields present | Hook (deterministic script) |
| **Semantic gate** | Brand voice, quality, relevance | LLM-as-Judge (skill or inline prompt) |
| **Completeness gate** | All items processed, no gaps | Hook (checklist verification) |
| **Human gate** | Approval, creative direction | Pause workflow, present for review |
| **Composite gate** | Format + semantic | Hook first (fast, cheap), then LLM if format passes |

**Default**: use format gates (hooks) wherever possible — they're free, instant, and deterministic. Add semantic gates only where meaning matters.

## 6. Model tiering

| Role in workflow | Model | Why |
|-----------------|-------|-----|
| Orchestrator / lead agent | Opus | Needs reasoning for decomposition and delegation |
| Worker agents (generation, analysis) | Sonnet | Balance of quality and speed |
| Format validation, classification, routing | Haiku | 90% quality of Sonnet, 2× speed, 3× cheaper |
| Creative generation with voting | Sonnet ×N | Multiple candidates, select best |

## 7. Error handling

| Failure type | Strategy |
|-------------|----------|
| Format error (wrong structure) | Auto-retry with error feedback in prompt (max 2 retries) |
| Quality error (gate fails) | Route to evaluator-optimizer loop |
| Partial completion | Decompose further — identify what's missing, regenerate only that |
| Agent timeout / crash | Checkpoint-resume — last saved state + retry |
| Ambiguity in input | Escalate to human — ask before guessing |

**Anti-pattern**: infinite retry loops. If two retries fail, escalate or try a different approach.

## 8. Parallelization checklist

Before marking steps as parallel, verify:
- [ ] No data dependency between them (Step A doesn't need Step B's output)
- [ ] They don't write to the same file
- [ ] Their outputs can be aggregated without order dependency
- [ ] The join step is explicitly defined (who combines results and how)

## 9. Workflow complexity guide

| Complexity | Steps | Agents | Pattern |
|-----------|-------|--------|---------|
| Simple | 2-3 | 0-1 | Sequential chain |
| Medium | 3-5 | 1-3 | Sequential + parallel sections |
| Complex | 5-8 | 3-5 | Orchestrator-workers or pipeline with branches |
| System | 8+ | 5+ | Hierarchical — decompose into sub-workflows first |

**Rule**: if a workflow exceeds 8 steps, it should be split into sub-workflows that are composed.

## 10. Pre-send checklist

Before presenting a workflow design:

1. [ ] Every step has a single clear goal (not "generate and validate")
2. [ ] Context flow is explicit — no step assumes data it hasn't been given
3. [ ] Gates are concrete (not "check quality" but "verify all 5 required fields are present")
4. [ ] Error handling defined for every gate (what happens on failure)
5. [ ] No two agents write to the same file
6. [ ] Model choice is justified per step (not everything on Opus)
7. [ ] Parallel steps are truly independent (checked against parallelization checklist)
8. [ ] Human checkpoints are placed where creative judgment is needed
9. [ ] The simplest viable pattern was chosen (not orchestrator-workers when a chain suffices)
10. [ ] Output format of each step is specified precisely enough for the next step to consume
