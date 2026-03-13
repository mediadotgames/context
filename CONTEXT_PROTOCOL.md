# Context Maintenance Protocol

## When to Update
Update the project context files **after every significant decision, implementation change, or design evolution** — don't wait to be asked. If you changed how something works, decided against an approach, or discovered a constraint, update the docs immediately.

## What to Maintain

### 1. Decisions Log (`DECISIONS.md`)
For every decision, record:
- **What** was decided
- **Why** (the reasoning, tradeoffs considered, alternatives rejected)
- **When** (date or session reference)
- **Scope of impact** (what other parts of the system this affects)

Never overwrite a decision — if it's reversed, add a new entry referencing the old one.

### 2. Specifications (`SPECS.md`)
Keep a living spec for each major component/feature:
- **Inputs**: exact shape, types, sources, validation rules
- **Outputs**: exact shape, types, destinations, formats
- **Processing logic**: transformation rules, business logic, edge case handling
- **Data flow**: how data moves between components, what triggers what
- **API contracts**: request/response shapes, error codes, auth requirements

When any of these change, update the spec AND add a note to the decisions log explaining why.

### 3. Requirements (`REQUIREMENTS.md`)
Maintain these sections:
- **User stories**: written as "As a [user], I want [goal] so that [reason]" — accumulate these, never delete
- **Acceptance criteria**: concrete, testable conditions for each story
- **Test cases**: specific scenarios to validate against later, including:
  - Happy path cases
  - Edge cases discovered during implementation
  - Regression cases (things that broke or almost broke)
- **UX principles**: long-term UX rationale — why we chose an interaction pattern, what we're optimizing for, what we'd never want to sacrifice
- **Non-functional requirements**: performance targets, accessibility, security, etc.

### 4. Open Questions (`OPEN_QUESTIONS.md`)
Track unresolved items:
- **Decision needed**: what choice is pending and what's blocking it
- **Options on the table**: with rough pros/cons
- **Who/what unblocks it**: more info needed? User input? Prototype first?
- **Priority**: is this blocking work now or can it wait?

Remove items as they're resolved (move them to the decisions log).

### 5. Constraints & Risks (`CONSTRAINTS.md`)
- **Technical constraints**: platform limits, dependency quirks, performance bottlenecks
- **Known risks**: things that might break, scale poorly, or cause problems later
- **Tech debt**: shortcuts taken consciously, with notes on what a proper fix looks like

## Update Format
When updating, keep diffs minimal and meaningful. Use timestamps or session markers. Structure for scannability — someone should be able to open any file and get oriented in 30 seconds.

## On-Demand Sync Command
If I say **"sync docs"**, do a full pass:
1. Review everything we've discussed and built in this session
2. Update all five files above (DECISIONS.md, SPECS.md, REQUIREMENTS.md, OPEN_QUESTIONS.md, CONSTRAINTS.md)
3. Call out anything that contradicts or supersedes prior entries
4. Surface any new open questions that emerged
5. Print a short summary of what changed
