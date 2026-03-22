# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

A shared library of universal Claude Code skills, agents, and research that can be transferred between projects to build agentic flows. Skills and agents here are project-agnostic — designed to work in any codebase.

## Structure

- `.claude/skills/` — Reusable skills (each with SKILL.md + reference files):
  - `meta-prompt/` — Prompt architect: creates, chains, and audits prompts for Claude 4.6
  - `meta-workflow/` — Workflow architect: designs multi-agent pipelines with steps, gates, and context flow
  - `meta-agent/` — Agent architect: creates agent definitions (workers, orchestrators, reviewers) from task descriptions or workflow documents
  - `meta-hook/` — Hook architect: creates, audits, and debugs Claude Code hooks (lifecycle guardrails, context injection, validation)
  - `meta-architect/` — Architecture analyst: reviews projects, detects smells, generates diagrams, designs architecture, creates project-specific architect skills
  - `meta-research/` — Research planner: decomposes vague questions into structured plans with sub-questions, sources, strategies, and sufficiency criteria
- `.claude/agents/` — Reusable agent definitions (to be added)

## Conventions

### Skills
- Each skill lives in `.claude/skills/<name>/` with a `SKILL.md` (YAML frontmatter: name, description) and optional reference files
- Skills inject instructions, not code — they guide Claude's behavior in-conversation
- The `description` field in SKILL.md frontmatter controls when the skill activates — write it precisely
- Follow prompt design principles from `.claude/skills/meta-prompt/prompt-principles.md`: specific roles, XML tags for structure, examples over descriptions, positive framing, self-check sections
- Keep core instructions under 300 words; use reference files for additional context (progressive disclosure)

### Agents
- Agent definitions go in `.claude/agents/<name>.md`
- Each agent should have one clear task, explicit tool permissions, and defined output format

