# Prompt Design Principles for Claude 4.6

A checklist for writing effective prompts. Each principle has an example and an explanation of why it matters.

---

## 1. Structure: "prompt as contract"

Every prompt follows this skeleton. Order matters — data on top, instructions at the bottom (up to 30% quality improvement on long context).

```
[Role — 1-2 lines, specific]

[Context — who, what, why. Concrete, not abstract]

[Input data — files to read, references, prior results]

[Task — what exactly to do]

[Success criteria — what good output looks like]

[Constraints — what to avoid and WHY]

[Output format — exact structure, template or example]
```

Why: Claude 4.6 follows instructions literally. Precise contract = precise execution. Missing section = model fills the gap with assumptions.

## 2. Role — one line, specific

```
# Good:
You are a backend engineer specializing in PostgreSQL performance
optimization for high-traffic SaaS applications.

# Bad:
You are an experienced expert in databases and engineering.
```

Why: a specific role focuses tone, vocabulary, and depth. Generic role = generic result.

## 3. Context and motivation — explain "why"

For every non-trivial rule or constraint — explain the reason.

```
# Good:
API responses use snake_case — because the mobile team's JSON parser
auto-maps snake_case to native object properties. CamelCase would
require manual mapping on every endpoint.

# Bad:
API responses use snake_case.
```

Why: Claude generalizes from explanations to analogous cases. A bare rule is a stop sign; a motivated rule is a compass.

## 4. Positive framing — "do", not "don't"

```
# Good:
Write in flowing prose with short paragraphs.

# Bad:
Don't use markdown, don't make lists, don't format.
```

Why: negative instructions constrain without directing. The model knows what to avoid but not what to do instead.

## 5. XML tags — for separating prompt sections

```xml
<context>
  Acme Corp is a B2B SaaS company selling analytics tools...
</context>

<task>
  Write release notes for version 3.2...
</task>

<constraints>
  Maximum 500 words. No internal codenames.
</constraints>
```

Why: XML tags are Claude's signature technique. They unambiguously separate instructions, context, examples, and data. Especially important in long prompts (3000+ tokens).

## 6. Examples (few-shot) — the most reliable lever

3-5 examples in `<example>` tags. Examples should be:
- **Relevant** — reflect real use cases, not abstract scenarios
- **Diverse** — cover different cases and edge cases
- **Structured** — clear input/output format

```xml
<example>
  <input>Feature: user can reset password via email</input>
  <output>
    ## Password Reset via Email
    **As a** registered user
    **I want to** reset my password using my email address
    **So that** I can regain access when I forget my credentials

    ### Acceptance Criteria
    - [ ] User receives reset link within 60 seconds
    - [ ] Link expires after 24 hours
    - [ ] Old password is invalidated on reset
  </output>
</example>
```

Why: examples control format, tone, and structure more reliably than text descriptions. Claude generalizes the pattern.

## 7. Calibrated language — match intensity to stakes

```
# For true invariants (security, correctness, data loss):
User input is always sanitized before SQL queries — unsanitized input
enables injection attacks that can expose the entire database.

# For preferences and style:
Use present tense in commit messages. It reads more naturally
in git log and matches the convention of most open-source projects.
```

Reserve strong language (MUST, NEVER) for genuine safety or correctness constraints. For everything else, explain the reasoning — Claude 4.6 is highly responsive to system prompts and doesn't need aggressive phrasing. Overuse of MUST/NEVER causes overtriggering — the model overcompensates and loses naturalness.

## 8. Specificity — numbers, names, paths

```
# Good:
Read docs/api-reference.md, the "Authentication" section.
Generate 5 test cases, each under 20 lines.

# Bad:
Review the documentation. Write some test cases.
```

Why: "some" = anywhere from 2 to 20. Numbers and paths eliminate ambiguity.

## 9. Output format — show, don't describe

Provide a template or example of the result instead of describing it in words.

```
# Good:
Format each entry as:

| Endpoint | Method | Auth | Description |
|----------|--------|------|-------------|
| /users   | GET    | JWT  | List users  |

# Bad:
Present the result as a table with columns for endpoint, method, and description.
```

## 10. Files and tools — explicit instructions

```
# Good:
Before starting, read:
1. src/config/database.ts — connection settings — required
2. docs/schema.md — if the task involves data modeling
3. tests/fixtures/ — for examples of test data format

# Bad:
Use knowledge from available project files.
```

Why: Claude reads what you ask for. Without explicit instructions it may skip a key file or read an irrelevant one.

## 11. Prompt length — sweet spot

- Core instructions: 150-300 words
- With context and examples: up to 1000-1500 words
- Reasoning quality degrades past ~3000 tokens of instructions

If the prompt is longer — split into a chain.

## 12. Prompt chaining — when the task is big

If the task has 3+ distinct goals or naturally breaks into steps — decompose:

```
Step 1: Research → intermediate result (saved to file)
  Gate: does the result contain what's needed?
Step 2: Analyze result → conclusions (saved to file)
  Gate: are conclusions grounded in the data?
Step 3: Based on conclusions → final artifact
```

Why: each step with one goal produces better quality than one prompt with five goals. Gates catch errors early before they cascade.

## 13. Self-check — built-in verification

For tasks with clear criteria, ask the model to verify its own result:

```
Before presenting the result, verify:
- [ ] Every API endpoint has an error response example
- [ ] No endpoint requires auth without documenting the auth method
- [ ] Response schemas match the TypeScript types in src/types/
```

Why: self-check catches errors the model would notice on re-reading. Especially effective for structural requirements.

## 14. Uncertainty handling

Explicitly define what the model should do when unsure:

```
If you're not sure about an implementation detail, flag it with
[UNCERTAIN: reason] rather than guessing. It's better to surface
a question than to silently make an assumption.
```

Why: without explicit permission to express uncertainty, Claude may fabricate plausible-sounding answers rather than admit gaps.

## 15. Context engineering — load only what's needed

For prompts that work within a larger system (agents, skills, pipelines):

- Reference files by path instead of pasting their contents
- Specify which sections of a file to read, not the whole document
- Use progressive disclosure: start with a summary, load details only when the task requires them
- For multi-step workflows, each step should receive only the context it needs — not the full history

Why: every token competes for the model's attention. Irrelevant context dilutes the signal and degrades quality.

---

## Pre-send checklist

Before presenting a prompt, verify each point:

1. [ ] Role is specific (not "expert" but "who exactly, in what domain, for what audience")
2. [ ] Context is concrete (real names, paths, constraints — not "a company" but the actual entity)
3. [ ] Files to read listed explicitly with paths and sections
4. [ ] Task uses positive framing ("do X") not negative ("don't do Y")
5. [ ] Non-trivial rules are motivated (explain why)
6. [ ] Output format shown via template or example
7. [ ] Strong language (MUST/NEVER) reserved for true invariants only
8. [ ] Numbers are concrete (not "several" but "5")
9. [ ] Core instruction length < 300 words (excluding context and examples)
10. [ ] Self-check included for tasks with verifiable criteria
11. [ ] Uncertainty handling defined (what to do when unsure)
12. [ ] If the task has 3+ goals — chain proposed, not a mega-prompt
