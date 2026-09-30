# Generic Claude Session Handoff / Context Switch Command

My current Claude session has become very long, and I want to continue the work in a completely new Claude session **without losing important context, decisions, business rules, implementation details, constraints, reasoning, or project history**.

Before continuing any implementation or analysis, create a comprehensive Markdown **handoff/context document** from the current conversation.

## Configuration

I will only provide these three configuration values:

- **Project:** HMS
    
- **Module / Area:** HCK
    
- **Handoff File:** F:\Office Projects\HMS\docs\KnowledgeBase\Hck_Module_Analysis.md
    
- **senior's architecture/design guideline :** F:\Office Projects\HMS\docs\modules\HCK.md
Everything else must be determined by Claude from the **actual conversation and, where necessary, the actual codebase**.

Claude must independently determine and document:

- what the original objective was
    
- what has been investigated
    
- what has been decided
    
- what is LOCKED
    
- what has already been implemented
    
- what is currently being implemented
    
- what is partially implemented
    
- what remains to be done
    
- what the immediate next step is
    
- what is waiting for a later phase
    
- what phases/stages have already been established
    
- what phase/stage the work is currently in
    
- what known bugs, risks, and gaps remain
    
- what decisions were rejected or superseded
    
- what is still unconfirmed
    
- what dependencies exist between remaining tasks
    

**Do not ask me to manually provide the current state, implementation status, next task, or phase status if that information can be determined from the conversation or codebase.**

---

# Purpose

This document is **not a simple summary**.

Its purpose is to allow me to start a completely new Claude session and continue the work as if the new session had access to the important context of the previous session.

The new Claude session must be able to understand:

- what we are working on
    
- why we are working on it
    
- how the current system works
    
- what we discovered
    
- what we decided
    
- what we rejected
    
- what has already been implemented
    
- what is currently in progress
    
- what remains
    
- what should happen next
    
- what should wait for a later phase
    

---

# Required Handoff Structure

## 1. Project Context

Document:

- project/application
    
- module/area
    
- technology stack
    
- framework/runtime versions where relevant
    
- architecture and patterns
    
- relevant application layers
    
- important entities
    
- services
    
- repositories
    
- controllers
    
- UI components
    
- database
    
- external integrations
    

Preserve exact names whenever they appeared in the conversation.

---

## 2. Original Problem / Objective

Explain:

- the original problem
    
- why it matters
    
- the desired final behavior
    
- business/technical reasons
    
- how the objective evolved during the session
    

If the objective changed, preserve the evolution and identify the latest authoritative objective.

---

## 3. Current Architecture / Existing Implementation

Describe the existing implementation relevant to this work.

Include where applicable:

- request/response flow
    
- controller → service → repository flow
    
- entity relationships
    
- database relationships
    
- important methods
    
- DTOs
    
- UI → backend flow
    
- validation
    
- authorization
    
- existing project patterns
    
- existing implementation conventions
    

Do not redesign the architecture merely for documentation purposes.

---

## 4. Confirmed / LOCKED Decisions

Create a clearly marked section:

> **LOCKED — DO NOT CHANGE WITHOUT EXPLICIT USER APPROVAL**

Record every decision explicitly confirmed during the conversation.

Pay special attention to statements such as:

- "confirmed"
    
- "locked"
    
- "use this"
    
- "we will use..."
    
- "don't do this"
    
- "I don't want this"
    
- "remove this"
    
- "this is not acceptable"
    
- "final decision"
    

Preserve exact:

- business rules
    
- formulas
    
- data types
    
- validation rules
    
- lifecycle rules
    
- status rules
    
- UI rules
    
- security rules
    
- RBAC rules
    
- database rules
    
- architectural decisions
    
- naming decisions
    
- compatibility requirements
    

Do not reinterpret or weaken a locked decision.

---

## 5. Rejected / Superseded Approaches

Create a clearly marked section:

> **DO NOT REINTRODUCE**

Record approaches that were:

- rejected
    
- replaced
    
- explicitly forbidden
    
- superseded
    
- previously recommended but later overruled
    

For important decisions, explain:

1. Previous approach
    
2. Why it was rejected, if known
    
3. What replaced it
    
4. Current authoritative decision
    

This prevents a new Claude session from proposing the same rejected solution again.

---

## 6. Important Constraints

Preserve all explicit constraints, including:

- UI restrictions
    
- security restrictions
    
- RBAC restrictions
    
- database constraints
    
- backward compatibility
    
- performance requirements
    
- concurrency requirements
    
- data integrity
    
- API/network restrictions
    
- information that must not be exposed to users
    
- deployment limitations
    
- architectural constraints
    
- explicit "must not" requirements
    

Do not invent additional constraints.

---

## 7. Current Implementation State

Claude must determine the current implementation state from the conversation and codebase.

Clearly separate:

### Completed

What has actually been implemented or verified.

### In Progress

What is currently being worked on.

### Partially Implemented

What has started but is incomplete.

### Not Started

What has not yet been implemented.

### Blocked / Waiting

What cannot or should not proceed yet because of a dependency, decision, phase, or unresolved issue.

### Current Phase / Stage

Determine which phase/stage the project is currently in.

If formal phases were established, preserve their original names.

If no formal phases exist, infer a logical current stage **only when strongly supported by the conversation**, and mark the inference appropriately.

---

## 8. Phase / Implementation Roadmap

Preserve the phase structure established during the conversation.

For each phase, document:

- objective
    
- required changes
    
- affected files/classes
    
- completed work
    
- remaining work
    
- dependencies
    
- known risks
    
- current status
    

Use status such as:

- `COMPLETED`
    
- `IN PROGRESS`
    
- `PARTIALLY COMPLETED`
    
- `NOT STARTED`
    
- `BLOCKED`
    
- `WAITING FOR PREVIOUS PHASE`
    

Do not invent new phases.

If future work was explicitly deferred, preserve it under the appropriate future phase.

---

## 9. Current State → Next Step → Future Work

This section is especially important.

Claude must derive these three things automatically:

### CURRENT STATE

What is the exact state of the implementation right now?

### NEXT STEP

What is the next concrete task that should be performed when the new Claude session starts?

### WAITING / FUTURE PHASE

What work has been identified but intentionally belongs to a later phase or depends on another task?

Do not mix future-phase work with the immediate next task.

---

## 10. Critical Business / Technical Rules

Preserve important rules exactly.

Depending on the project, this may include:

- calculations
    
- quantity rules
    
- UOM rules
    
- conversion rules
    
- pricing rules
    
- cost rules
    
- stock rules
    
- allocation rules
    
- lifecycle rules
    
- confirmation rules
    
- cancellation rules
    
- FEFO/FIFO rules
    
- concurrency rules
    
- validation rules
    
- accounting rules
    
- authorization rules
    

Preserve formulas and exact meanings.

---

## 11. Current Bugs / Risks / Gaps

For each discovered issue, document:

- problem
    
- symptoms
    
- root cause, if established
    
- affected component
    
- proposed fix
    
- whether the fix is confirmed or proposed
    
- unresolved questions
    
- dependencies
    
- potential side effects
    

Clearly distinguish:

`CONFIRMED FACT`

`PROPOSED SOLUTION`

`UNCONFIRMED`

---

## 12. Code-Level Context

Preserve exact technical details that may be required by the next Claude session.

Include where relevant:

- class names
    
- method names
    
- entity names
    
- DTO names
    
- property names
    
- database columns
    
- repository methods
    
- service methods
    
- controller actions
    
- JavaScript functions
    
- Razor views
    
- middleware
    
- attributes
    
- EF Core behavior
    
- LINQ behavior
    
- query/filter behavior
    
- transactions
    
- concurrency mechanisms
    
- indexes
    
- constraints
    

Do not reproduce entire files unless necessary.

---

## 13. UI ↔ Backend Contract

Document:

- what the UI currently sends
    
- what the backend currently expects
    
- what the backend should receive
    
- what must not be exposed
    
- important IDs/identifiers
    
- hidden fields
    
- route parameters
    
- request DTOs
    
- response DTOs
    
- JavaScript behavior
    
- Razor behavior
    
- AJAX/API behavior
    
- serialization requirements
    

Clearly identify any current mismatch.

---

## 14. Authentication / Authorization / RBAC

If applicable, preserve:

- authentication mechanism
    
- authorization attributes
    
- middleware
    
- permission resolution
    
- route/method matching
    
- role behavior
    
- permission records
    
- seed requirements
    
- known permission gaps
    
- existing module authorization patterns
    

Follow existing project patterns unless a confirmed decision says otherwise.

---

## 15. Database / Migration Requirements

Document:

- relevant tables
    
- entities
    
- columns
    
- data types
    
- nullable/non-nullable state
    
- relationships
    
- foreign keys
    
- indexes
    
- unique constraints
    
- migrations
    
- seed data
    
- existing schema assumptions
    
- required future schema changes
    

Clearly distinguish implemented changes from required future changes.

---

## 16. Decision History

For major decisions, preserve the evolution.

Use:

|Topic|Earlier Decision|Later Decision|Current Authority|
|---|---|---|---|
|`[topic]`|`[old decision]`|`[new decision]`|`[current decision]`|

The latest explicitly confirmed decision is authoritative.

Older conflicting decisions must be marked `SUPERSEDED`.

---

## 17. Unconfirmed / Open Questions

Create a dedicated section containing:

- unresolved questions
    
- assumptions
    
- information that was discussed but never confirmed
    
- areas requiring user confirmation
    
- technical uncertainties
    
- business-rule uncertainties
    

Never silently convert these into requirements.

---

# Accuracy Rules

## Do Not Invent

Never invent:

- requirements
    
- business rules
    
- implementation status
    
- files
    
- classes
    
- methods
    
- architecture
    
- decisions
    
- test results
    
- database behavior
    
- user preferences
    

If something is unknown:

`UNKNOWN`

---

## Unconfirmed Information

If something was discussed but never confirmed:

`UNCONFIRMED`

---

## Conflicting Decisions

If decisions conflict:

1. identify the conflict
    
2. preserve the older decision
    
3. preserve the newer decision
    
4. identify the newer confirmed decision as authoritative
    
5. mark the older decision `SUPERSEDED`
    

---

## Preserve Explicit User Decisions

Any explicit user statement such as:

- "locked"
    
- "confirmed"
    
- "don't do this"
    
- "I don't want this"
    
- "remove this"
    
- "use this instead"
    
- "this is not acceptable"
    

must be preserved prominently.

---

## Preserve Important Reasoning

Do not preserve meaningless conversational noise.

However, preserve the reasoning behind major decisions whenever losing that reasoning could cause a future Claude session to repeat a rejected approach.

---

## Separate Facts From Assumptions

Use explicit labels:

- `CONFIRMED`
    
- `LOCKED`
    
- `IMPLEMENTED`
    
- `IN PROGRESS`
    
- `PROPOSED`
    
- `UNCONFIRMED`
    
- `SUPERSEDED`
    
- `REJECTED`
    
- `BLOCKED`
    
- `WAITING`
    
- `UNKNOWN`
    

---

# Final Section

At the very end, create:

# NEXT SESSION START HERE

This must be concise and immediately actionable.

Include:

### Where We Are Now

The current implementation state.

### What We Are Solving

The current objective.

### LOCKED Decisions

Only the most important decisions the new Claude must know immediately.

### Rejected Approaches

Only the important rejected approaches that Claude must not reintroduce.

### What Remains

The remaining implementation/analysis work.

### Immediate Next Step

The exact next action Claude should take.

### Waiting / Future Phase

Work that has been intentionally deferred and should not be started yet.

---

# Instructions for the New Claude Session

When I provide this handoff document in a new Claude session, Claude must:

1. Read the entire handoff document before making implementation decisions.
    
2. Treat `LOCKED` decisions as authoritative.
    
3. Treat `REJECTED`, `SUPERSEDED`, and `DO NOT REINTRODUCE` items as constraints.
    
4. Never silently override a locked decision.
    
5. Never reintroduce a rejected approach unless I explicitly ask to reconsider it.
    
6. Inspect the actual codebase before making implementation claims.
    
7. Follow existing project patterns where appropriate.
    
8. Avoid unnecessary architectural redesign.
    
9. Continue from the documented current implementation state.
    
10. Use `CURRENT STATE`, `NEXT STEP`, and `WAITING / FUTURE PHASE` to determine where to continue.
    
11. Clearly distinguish facts from assumptions.
    
12. Mark uncertain information as `UNCONFIRMED`.
    
13. Ask questions only when genuinely necessary.
    
14. Before changing a locked decision, explicitly ask for approval.
    
15. Do not start unrelated refactoring.
    
16. Do not restart analysis that has already been completed.
    
17. Do not treat a future-phase task as the immediate task unless explicitly instructed.
    
18. Inspect the actual implementation before claiming that something is completed, missing, or broken.
    

---

# Verification Before Finishing

Before finishing the handoff document, verify that it contains:

-  Project context
    
-  Original objective
    
-  Current architecture
    
-  LOCKED decisions
    
-  Rejected/superseded approaches
    
-  Important constraints
    
-  Current implementation state
    
-  Current phase
    
-  Phase roadmap
    
-  Current State → Next Step → Future Work
    
-  Critical business rules
    
-  Bugs/risks/gaps
    
-  Code-level context
    
-  UI/backend contract
    
-  Authentication/RBAC
    
-  Database/migration requirements
    
-  Decision history
    
-  Unconfirmed/open questions
    
-  New-session continuation instructions
    
-  `NEXT SESSION START HERE`
    

After creating the file, **verify it for missing, contradictory, or incorrectly classified information**.

Then provide me with a short report containing:

1. What information was preserved
    
2. What was marked `LOCKED`
    
3. What was marked `REJECTED` / `SUPERSEDED`
    
4. What remains `UNCONFIRMED` / unresolved
    
5. What Claude identified as the current state
    
6. What Claude identified as the immediate next step
    
7. What Claude identified as future/waiting work
    

**Do not start new implementation work yet.**

