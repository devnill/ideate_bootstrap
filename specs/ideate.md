# Ideate Plugin — Specification

## Overview

Ideate is a Claude Code plugin that provides a structured workflow for planning, building, and validating software projects. It addresses three specific limitations of ad-hoc plan mode:

1. Plan mode produces a single monolithic plan. It doesn't decompose work into units that agents can execute in parallel.
2. Plan mode doesn't probe intent deeply. It takes requirements at face value instead of asking clarifying questions.
3. Plan mode doesn't iterate. Once a plan is written, there's no structured loop for execution, review, and refinement.

Ideate replaces this with a four-phase cycle: **plan**, **execute**, **review**, **refine**. Each phase is a user-invocable skill. The plugin also ships agents specialized for research, architecture, and review.

---

## Plugin Structure

```
ideate/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   ├── plan/
│   │   └── SKILL.md
│   ├── execute/
│   │   └── SKILL.md
│   ├── review/
│   │   └── SKILL.md
│   └── refine/
│       └── SKILL.md
├── agents/
│   ├── researcher.md
│   ├── architect.md
│   ├── code-reviewer.md
│   ├── spec-reviewer.md
│   ├── gap-analyst.md
│   └── journal-keeper.md
└── README.md
```

### Plugin Manifest

```json
{
  "name": "ideate",
  "description": "Structured workflow for planning, building, and validating software projects through iterative interview, execution, and review.",
  "version": "0.1.0",
  "author": {
    "name": "Dan"
  },
  "license": "MIT",
  "keywords": ["planning", "development", "review", "workflow", "agents"]
}
```

---

## Artifact Directory

All phases read from and write to a shared artifact directory. The user chooses this location at the start of `plan`. The structure is:

```
{artifact-dir}/
├── steering/
│   ├── interview.md              # Full Q&A transcript
│   ├── guiding-principles.md     # The "why" — intent, goals, success criteria
│   ├── constraints.md            # Technology, design, and process constraints
│   └── research/                 # Research findings from background agents
│       ├── {topic-slug}.md
│       └── ...
├── plan/
│   ├── overview.md               # High-level plan summary
│   ├── architecture.md           # Technical architecture and key decisions
│   ├── execution-strategy.md     # Agent strategy: teams, subagents, worktrees, sequencing
│   └── work-items/               # Granular parallelizable tasks
│       ├── 001-{name}.md
│       ├── 002-{name}.md
│       └── ...
├── reviews/
│   ├── incremental/              # Per-work-item reviews (written during execute)
│   │   ├── 001-{name}.md
│   │   └── ...
│   └── final/                    # Comprehensive review (written during review)
│       ├── code-quality.md
│       ├── spec-adherence.md
│       ├── gap-analysis.md
│       └── summary.md
└── journal.md                    # Running log: decisions, challenges, unresolved issues
```

### Artifact Conventions

- All artifacts are Markdown.
- Each work item in `plan/work-items/` is a self-contained unit of work. It must include: objective, acceptance criteria, file scope (which files it touches), dependencies on other work items (by ID), and estimated complexity.
- Work items must not have overlapping file scope. If two items need the same file, they must be sequenced via a dependency, or the scope must be split.
- `journal.md` is append-only. Every phase appends entries. Entries are timestamped and tagged with the phase that wrote them (`[plan]`, `[execute]`, `[review]`, `[refine]`).

---

## Skills

### `/ideate:plan`

**Purpose**: Interview the user, research the problem space, and produce a complete plan.

**Invocation**: User runs `/ideate:plan` optionally with a one-line idea as argument.

**Frontmatter**:

```yaml
---
name: plan
description: >
  Interview the user to explore a software idea, research the problem space,
  and produce a structured plan with guiding principles, architecture,
  parallelizable work items, and an execution strategy.
user-invocable: true
argument-hint: "[initial idea or topic]"
---
```

**Behavior — Step by Step**:

1. **Ask for artifact directory.** Default to `./specs/{project-name}/`. Create the directory structure if it doesn't exist.

2. **Open the interview.** If the user provided an argument, treat it as the initial idea. Otherwise, ask for one. Then begin a structured interview. The interview has three tracks that interleave naturally:

   - **Intent track**: What does the user actually want? What problem does this solve? Who is it for? What does success look like? What are the non-goals?
   - **Design track**: Technology choices, architecture preferences, scale, performance requirements, deployment targets, existing systems to integrate with.
   - **Process track**: How should execution work? Parallel vs. sequential? How many agents? What's the acceptable tradeoff between speed and token cost? Does the user want to approve each phase or run autonomously?

   The interview is a conversation, not a questionnaire. Ask one or two questions at a time. Wait for the answer. Use the answer to decide what to ask next. Do not ask questions the user has already answered.

3. **Spawn research agents during the interview.** When the user mentions a technology, domain, or design pattern worth investigating, spawn a `researcher` agent in the background. Inform the user that research is underway. When results come back, integrate findings into follow-up questions. Do not ask the user to repeat information the researchers already found.

4. **Detect completion.** The interview is done when: (a) intent, design, and process tracks are all sufficiently covered, or (b) the user says to move on. Before closing, present a brief summary of what was captured and list any open questions or risks. The user can address these or defer them.

5. **Produce artifacts.** Write the following files:
   - `steering/interview.md` — Full Q&A transcript, formatted as alternating **Q:** and **A:** blocks.
   - `steering/guiding-principles.md` — 5-15 principles that capture the user's intent. Each principle has a short name and a one-paragraph explanation of *why* it matters. These are the sanity checks used later to verify the project stays on track.
   - `steering/constraints.md` — All technology, design, and process constraints the user specified or agreed to.
   - `steering/research/*.md` — One file per research topic, written by the researcher agents.
   - `plan/overview.md` — High-level summary of what will be built.
   - `plan/architecture.md` — Technical architecture. Key components, data flow, interfaces between components.
   - `plan/execution-strategy.md` — How agents will be used during execute. This document is based on the user's answers in the process track. It specifies: number of parallel agents, whether to use agent teams or sequential subagents, whether to use worktrees, and the sequencing of work item groups.
   - `plan/work-items/*.md` — One file per work item. See format below.
   - `journal.md` — Initial entry recording the planning session.

6. **Present the plan.** After writing artifacts, present a summary: number of work items, dependency graph, estimated parallelism, and any unresolved concerns. Ask if the user wants to revise anything or proceed to execute.

**Work Item Format** (`plan/work-items/NNN-name.md`):

```markdown
# NNN: {Title}

## Objective
What this work item accomplishes.

## Acceptance Criteria
- [ ] Criterion 1
- [ ] Criterion 2

## File Scope
- `src/components/Foo.tsx` (create)
- `src/utils/bar.ts` (modify)

## Dependencies
- Depends on: 002, 005
- Blocks: 010, 011

## Implementation Notes
Relevant technical details, edge cases, or constraints.

## Complexity
Low | Medium | High
```

### `/ideate:execute`

**Purpose**: Build the project by following the plan, using the agent strategy the user specified.

**Invocation**: User runs `/ideate:execute` optionally with the artifact directory path.

**Frontmatter**:

```yaml
---
name: execute
description: >
  Execute the plan produced by ideate:plan. Follows the execution strategy
  to build work items using agents, tracks progress, and spawns reviewers
  for each completed item.
user-invocable: true
argument-hint: "[artifact directory path]"
---
```

**Behavior — Step by Step**:

1. **Locate artifacts.** If argument provided, use it. Otherwise, look for the most recently modified artifact directory under `./specs/`. If ambiguous, ask.

2. **Read the plan.** Load `plan/execution-strategy.md`, `plan/overview.md`, `plan/architecture.md`, `steering/guiding-principles.md`, and the full list of work items. Validate that dependencies form a DAG (no cycles).

3. **Present the execution plan.** Show the user: total work items, dependency groups (items that can run in parallel), the agent strategy that will be used. Ask for confirmation before starting.

4. **Execute work items.** Follow the execution strategy:

   - **Sequential mode**: Work through items one at a time in dependency order. After each item, spawn a `code-reviewer` subagent to validate the output. Write the review to `reviews/incremental/NNN-name.md`. Report status to the user after each completion.

   - **Parallel mode (subagents)**: Identify the next batch of unblocked items. Spawn one subagent per item (up to the configured parallelism limit). Each subagent receives: the work item, architecture doc, guiding principles, and any relevant steering research. When a subagent completes, spawn a `code-reviewer` to validate. Write incremental reviews. Report batch status to the user.

   - **Team mode**: Create an agent team. Assign work items as tasks. The lead (this skill's session) monitors progress and reports to the user. Teammates claim tasks from the shared list. On task completion, the lead spawns review agents. Use worktrees if specified in the execution strategy.

   Regardless of mode, after each work item completes:
   - Update `journal.md` with a completion entry noting any deviations from plan.
   - If the reviewer finds issues, log them and either fix immediately (if minor) or flag to the user (if the issue changes scope or violates guiding principles).

5. **Report completion.** When all work items are done, present a summary: items completed, items that required rework, outstanding issues from incremental reviews. Suggest running `/ideate:review` next.

### `/ideate:review`

**Purpose**: Comprehensive multi-perspective review of the completed project against the plan and guiding principles.

**Invocation**: User runs `/ideate:review` optionally with the artifact directory path.

**Frontmatter**:

```yaml
---
name: review
description: >
  Comprehensive review of completed work against the plan, guiding principles,
  and original intent. Spawns specialized reviewers for code quality,
  spec adherence, gap analysis, and improvement opportunities.
user-invocable: true
argument-hint: "[artifact directory path]"
---
```

**Behavior — Step by Step**:

1. **Locate artifacts.** Same logic as execute.

2. **Read all context.** Load: guiding principles, constraints, architecture, all work items, all incremental reviews, journal. Also read the actual project source code.

3. **Spawn review agents in parallel.** Four specialized reviews, each run as a subagent:

   - **`code-reviewer`** — Code quality: correctness, readability, maintainability, security (OWASP top 10), performance, error handling, test coverage.
   - **`spec-reviewer`** — Spec adherence: does each work item's acceptance criteria pass? Does the implementation match the architecture doc? Are there undocumented deviations?
   - **`gap-analyst`** — Gap analysis: are there requirements from the guiding principles or interview that aren't addressed? Are there edge cases not handled? Missing error states? Incomplete integrations?
   - **`journal-keeper`** — Reads the journal and all review outputs, produces a decision log and open questions list.

4. **Synthesize.** Once all reviewers return, write their outputs to `reviews/final/`. Then produce `reviews/final/summary.md` which:
   - Lists all findings by severity (critical / significant / minor / suggestion).
   - Maps each finding to the guiding principle or work item it relates to.
   - Identifies which findings can be addressed automatically and which need user input.
   - Proposes a refinement plan (feed into `/ideate:refine` or another `/ideate:execute` cycle).

5. **Present to user.** Show the summary. For any finding that requires a decision not covered by existing steering documents, ask the user directly. Record their answers in `journal.md`.

### `/ideate:refine`

**Purpose**: Run the planning process against an existing codebase to produce a change plan.

**Invocation**: User runs `/ideate:refine` optionally with a description of what to change.

**Frontmatter**:

```yaml
---
name: refine
description: >
  Plan changes to an existing codebase. Analyzes current code, interviews the
  user about desired changes, and produces a structured plan that accounts for
  existing architecture and constraints.
user-invocable: true
argument-hint: "[description of desired changes]"
---
```

**Behavior — Step by Step**:

1. **Ask for artifact directory.** Same as plan.

2. **Analyze existing codebase.** Before interviewing, spawn an `architect` agent to survey the project: directory structure, languages, frameworks, key modules, existing tests, configuration, dependencies. Write findings to `steering/research/existing-codebase.md`.

3. **Load existing artifacts if present.** If this artifact directory already has artifacts from a previous cycle (guiding principles, prior plans, journal), load them as context. This is what makes refine iterative — it builds on prior work.

4. **Interview.** Same three-track interview as plan, but with additional context:
   - The architect's codebase analysis informs questions. Don't ask about technology choices the code already makes.
   - If prior guiding principles exist, confirm whether they still hold or need revision.
   - Focus on what's changing and why, rather than re-establishing the full project vision.

5. **Produce artifacts.** Same set as plan, but:
   - Append to existing `journal.md` rather than overwriting.
   - Update (don't replace) `steering/guiding-principles.md` if principles changed — note what changed and why.
   - Work items reference existing files to modify rather than creating everything from scratch.
   - `plan/overview.md` is a change plan, not a full project plan.

6. **Present the plan.** Same as plan phase.

---

## Agents

### `researcher`

Investigates a specific topic in the background during the interview. Returns structured findings.

```yaml
---
name: researcher
description: >
  Background research agent. Spawned during ideate:plan or ideate:refine
  to investigate a technology, pattern, domain, or design question.
  Returns structured findings with sources.
tools: Read, Grep, Glob, WebSearch, WebFetch
model: sonnet
background: true
maxTurns: 15
---
```

**System Prompt**: You are a research agent. You've been asked to investigate a specific topic for a software planning session. Search the web, read documentation, and examine any relevant code in the current project. Return a structured report:

1. **Summary** — 2-3 sentence overview of findings.
2. **Key Facts** — Bullet list of relevant facts, capabilities, limitations.
3. **Recommendations** — If applicable, what approach to take and why.
4. **Risks** — Known issues, gotchas, or limitations.
5. **Sources** — URLs or file paths consulted.

Be thorough but concise. Do not pad findings. If information is uncertain, say so.

### `architect`

Analyzes a codebase and designs the technical plan.

```yaml
---
name: architect
description: >
  Analyzes project structure and designs technical architecture.
  Used during ideate:plan to create the architecture doc and decompose
  work into parallelizable items, and during ideate:refine to survey
  existing code.
tools: Read, Grep, Glob, Bash
model: opus
maxTurns: 30
---
```

**System Prompt**: You are a software architect. Your job is to analyze codebases and design technical plans.

When analyzing an existing codebase: map directory structure, identify languages and frameworks, understand module boundaries, trace key data flows, note test coverage and configuration, and identify architectural patterns in use.

When designing architecture: decompose the system into modules with clear interfaces. Identify which pieces can be built independently (parallel) and which have strict ordering (sequential). Minimize cross-module dependencies. Flag any areas where the design has inherent tension or where tradeoffs need to be made.

Do not advocate for any particular technology. Present options with tradeoffs and let the plan reflect the user's stated preferences. If the user's preferences conflict with practical constraints, note the conflict without editorializing.

### `code-reviewer`

Reviews a completed work item for correctness and quality.

```yaml
---
name: code-reviewer
description: >
  Reviews code produced by a single work item. Checks correctness,
  quality, security, and adherence to the work item's acceptance criteria.
tools: Read, Grep, Glob, Bash
model: sonnet
maxTurns: 20
---
```

**System Prompt**: You are a code reviewer. You will receive a work item specification and a set of files to review.

Check:
1. **Acceptance criteria** — Does the code satisfy every criterion in the work item?
2. **Correctness** — Are there logic errors, off-by-one errors, race conditions, null handling gaps?
3. **Security** — Injection vulnerabilities, auth issues, data exposure, OWASP top 10.
4. **Quality** — Readability, naming, dead code, unnecessary complexity.
5. **Tests** — Are there tests? Do they cover the acceptance criteria? Are edge cases tested?

Output a structured review:
- **Pass/Fail** — Does this work item meet its acceptance criteria?
- **Findings** — List each issue with severity (critical/significant/minor) and specific file:line references.
- **Suggested fixes** — For each finding, describe the fix.

Do not praise good code. Only report problems and open questions.

### `spec-reviewer`

Reviews completed work against the full plan and guiding principles.

```yaml
---
name: spec-reviewer
description: >
  Reviews the complete project against the plan, architecture doc, and
  guiding principles. Identifies deviations and unmet requirements.
tools: Read, Grep, Glob
model: sonnet
maxTurns: 25
---
```

**System Prompt**: You are a specification reviewer. You will receive the project's guiding principles, architecture document, and work item specifications. You will also have access to the implemented code.

Your job is to verify:
1. **Architecture adherence** — Does the implementation match the architecture doc? Are module boundaries respected? Are interfaces as specified?
2. **Principle adherence** — For each guiding principle, is it reflected in the implementation? Are there places where the code contradicts a stated principle?
3. **Completeness** — Is every work item's acceptance criteria satisfied? Are there work items that were partially implemented or skipped?
4. **Consistency** — Are naming conventions, patterns, and error handling consistent across the codebase?

Output:
- **Deviations** — List each deviation from spec with severity and evidence.
- **Unmet criteria** — List acceptance criteria that are not satisfied.
- **Principle violations** — List guiding principles that are not reflected or contradicted.

Do not assess code quality (that's a different reviewer's job). Focus only on whether what was built matches what was specified.

### `gap-analyst`

Identifies missing requirements and unhandled scenarios.

```yaml
---
name: gap-analyst
description: >
  Identifies missing requirements, unhandled edge cases, incomplete
  integrations, and blind spots in the implementation.
tools: Read, Grep, Glob
model: sonnet
maxTurns: 25
---
```

**System Prompt**: You are a gap analyst. You look for what's missing — not what's wrong with what exists, but what doesn't exist at all.

Given the project's guiding principles, interview transcript, and implementation, identify:
1. **Missing requirements** — Things the user asked for or implied that aren't implemented.
2. **Unhandled edge cases** — Error conditions, empty states, boundary conditions, concurrent access, malformed input.
3. **Incomplete integrations** — External systems referenced in the plan that aren't fully connected.
4. **Missing infrastructure** — Logging, monitoring, configuration management, deployment, documentation.
5. **Implicit requirements** — Things not stated but expected by any reasonable user of this software (e.g., a web app should handle browser back button).

Output:
- **Gaps by category** — Each gap with severity and rationale.
- **Recommendations** — For each gap, whether it should be addressed now or deferred and why.

### `journal-keeper`

Maintains the decision journal and synthesizes review findings.

```yaml
---
name: journal-keeper
description: >
  Reads all review outputs, the journal, and the plan to produce a
  synthesis of decisions made, challenges encountered, and open questions.
tools: Read, Grep, Glob
model: sonnet
maxTurns: 15
---
```

**System Prompt**: You are a project historian and synthesizer. Your job is to read all available project artifacts — journal, reviews, plan, guiding principles — and produce two things:

1. **Decision Log** — A chronological record of significant decisions made during the project. For each: what was decided, why, what alternatives were considered, and what the implications are.

2. **Open Questions** — Issues, risks, or ambiguities that remain unresolved. For each: what the question is, why it matters, who needs to answer it (the user or a future development cycle), and what the consequences are of leaving it unresolved.

Do not duplicate the content of other reviews. Synthesize across them.

---

## Tone and Language

All skills and agents follow these communication rules:

- **Neutral tone.** No encouragement, validation, praise, or enthusiasm. Do not say "great idea" or "that's a solid approach." Just process information and respond.
- **Direct.** State things plainly. "This won't work because X" not "There might be some challenges with this approach, though it has merit."
- **No hedging qualifiers.** Don't say "I think", "perhaps", "it might be worth considering". If something is uncertain, state the uncertainty directly: "This is unverified" or "There are two possible interpretations."
- **Critical by default.** The tool's job is to find problems, not confirm the user's expectations. A bad idea should be identified as a bad idea with a clear explanation of why.
- **No filler.** No "Let me help you with that" or "I'd be happy to." Just do the work.

---

## Execution Strategy Design

The `plan` skill's interview includes a process track where the user specifies how execution should work. The resulting `execution-strategy.md` codifies these choices:

### Strategy Options

| Strategy | Description | Token Cost | Speed | When to Use |
|----------|-------------|-----------|-------|-------------|
| **Sequential** | One work item at a time, one subagent per item | Low | Slow | Small projects, tight budgets, highly interdependent work |
| **Batched parallel** | Groups of independent items via subagents, batches run sequentially | Medium | Medium | Medium projects, moderate parallelism |
| **Full parallel (teams)** | Agent team with shared task list, teammates claim work items | High | Fast | Large projects with many independent modules |

### Execution Strategy Document Format

```markdown
# Execution Strategy

## Mode
Sequential | Batched parallel | Full parallel (teams)

## Parallelism
Max concurrent agents: {N}

## Worktrees
Enabled: yes | no
Reason: {why or why not}

## Review Cadence
After every item | After every batch | At end only

## Work Item Groups
Group 1 (parallel): 001, 002, 003
Group 2 (parallel, depends on group 1): 004, 005
Group 3 (sequential): 006 → 007 → 008
...

## Agent Configuration
Model for workers: sonnet | opus
Model for reviewers: sonnet
Permission mode: acceptEdits | dontAsk
```

---

## Lifecycle — Full Cycle

A typical ideate session:

```
/ideate:plan "A CLI tool for managing dotfiles"
  ↓
  Interview (10-20 exchanges)
  ├── researcher agents investigate topics in background
  ├── artifacts written to specs/dotfiles-manager/
  └── plan with 12 work items, 3 parallel groups
  ↓
/ideate:execute specs/dotfiles-manager/
  ↓
  Execute work items per strategy
  ├── incremental reviews after each item
  ├── journal updated throughout
  └── status reports to user
  ↓
/ideate:review specs/dotfiles-manager/
  ↓
  4 parallel reviewers
  ├── code quality review
  ├── spec adherence review
  ├── gap analysis
  └── synthesis + open questions
  ↓
  Summary with findings and refinement suggestions
  ↓
  (optional) /ideate:refine "Add support for encrypted secrets"
  ↓
  New interview + updated plan → /ideate:execute → /ideate:review
```

---

## Constraints and Assumptions

1. **Claude Code features required**: Agent tool, background agents, `WebSearch` and `WebFetch` for researcher agents. Agent teams and worktrees are optional and depend on user's execution strategy choice.
2. **Agent teams are experimental.** If the user chooses full parallel mode with teams, the plan skill should note this and inform the user that the feature requires `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`.
3. **No external services.** The plugin uses only built-in Claude Code tools and web search. No MCP servers, no custom HTTP endpoints.
4. **Artifact directory is the contract.** All communication between phases happens through files in the artifact directory. There is no in-memory state carried between skill invocations. Each skill reads what it needs from disk.
5. **The user controls pacing.** Skills do not automatically chain. The user decides when to move from plan → execute → review → refine. The skills suggest the next step but don't invoke it.
