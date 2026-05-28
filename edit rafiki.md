Yes — the order matters a lot.

You should **not** feed the implementation agent the 22 pillar prompts first and then later give architecture rules. That causes the agent to make wrong assumptions about directories, contracts, tech stack, orchestration, tests, and UI.

The correct approach is:

# The implementation agent needs the “constitution” first

Before any pillar-specific implementation prompt, the agent should receive the **global system instructions** that govern every implementation.

Think of it like this:

```text
System rules first.
Architecture second.
Then individual components.
```

If you give a pillar prompt first, the agent may start inventing folders, contracts, database models, API styles, test patterns, and orchestration logic before the global rules exist.

That is exactly what we are trying to avoid.

---

# Best order to feed the implementation agent

## Phase 0 — Repository inspection instruction

First prompt should tell the agent:

```text
Inspect the repository before editing anything.
Identify existing architecture, framework, tests, database, API style, workers, contracts, and UI patterns.
Do not invent file paths.
Do not implement until you understand the codebase.
```

This is the first thing every coding agent should see.

---

## Phase 1 — Global implementation constitution

This should include:

```text
AGENTS.md rules
coding standards
commit protocol
test rules
no guessing file paths
read before editing
run tests before claiming success
schema/versioning rules
audit/security rules
migration rules
documentation rules
```

This becomes the agent’s operating law.

This should come before every pillar.

---

## Phase 2 — Tech stack and architecture decisions

Next give:

```text
tech stack
backend architecture
frontend architecture
database choice
queue/workflow choice
object storage choice
contract/schema system
testing tools
CI/CD expectations
deployment assumptions
```

This prevents the agent from building Pillar 5 in one style and Pillar 18 in another style.

---

## Phase 3 — File directory / monorepo architecture

After tech stack, give the file structure.

This should tell the agent:

```text
where backend modules live
where frontend modules live
where contracts live
where workers live
where agents live
where database migrations live
where tests live
where docs live
where shared types live
```

Important: this should come **before implementation**, but **after** the 22 pillars have been planned.

You were right earlier: we should not have written directories inside each pillar prompt. The directory should be designed once globally.

---

## Phase 4 — Shared contracts and global data model

Before individual pillars, give:

```text
shared package contract
handoff payload contract
job contract
report contract
repair ticket contract
accepted risk contract
operator decision contract
audit log contract
timeline version contract
asset ref contract
media ref contract
event contract
```

This is critical.

Otherwise every pillar will create its own slightly different version of:

```text
status
handoff_allowed
package_id
episode_id
timeline_version
repair_ticket
accepted_risk
```

That becomes a disaster.

---

## Phase 5 — Orchestration and agent runtime

Then give:

```text
Agent Graph Orchestration & Workflow Runtime
Agent Registry
workflow DAG rules
state machine rules
retry rules
repair loop rules
handoff rules
operator approval gates
```

This tells the agent **how the pillars connect**.

The individual pillar prompt says what Pillar 12 does.
The orchestration prompt says when Pillar 12 runs, what it waits for, how it fails, how it retries, and how it hands off.

---

## Phase 6 — UI/UX system and design system

Before building pillar UIs, give:

```text
global UI layout
studio console design
navigation model
episode workspace
package explorer
workflow graph viewer
repair ticket board
risk registry
role-based UI controls
component design system
```

Otherwise each pillar UI becomes a random dashboard.

---

## Phase 7 — Global testing, CI/CD, observability, security

Before implementation begins, give:

```text
global testing strategy
contract test rules
integration test rules
E2E pipeline tests
golden episode tests
CI/CD rules
observability standards
security rules
permissions/governance
failure recovery
```

This ensures each pillar is implemented to production-level standards.

---

## Phase 8 — Then feed individual pillar implementation prompts

Now the agent is ready for:

```text
Pillar 1 implementation prompt
Pillar 2 implementation prompt
Pillar 3 implementation prompt
...
Pillar 22 implementation prompt
```

The pillar prompts should be fed **one at a time**, not all at once.

For each pillar, the agent should receive:

```text
global constitution
architecture reference
shared contracts reference
orchestration reference
then the specific pillar prompt
```

Not necessarily pasted in full every time — but available as reference docs in the repo.

---

# Best practical order

Here is the clean order I recommend:

```text
0. Repo Inspection Prompt
1. AGENTS.md / Implementation Constitution
2. Tech Stack Decision
3. Monorepo / File Directory Architecture
4. Shared Contracts & Schema System
5. Global Data Model & Database Architecture
6. Agent Graph Orchestration & Workflow Runtime
7. Agent Management & Agent Registry
8. UI/UX Design System & Studio Console
9. Global Testing Strategy
10. Evaluation Harness & Quality Benchmarks
11. CI/CD Pipeline & Release Management
12. Observability, Logging & Monitoring
13. Permissions, Roles & Governance
14. Security & Data Protection
15. Asset Storage & Media Infrastructure
16. Import/Export & External Tool Integration Layer
17. Notification & Review Workflow System
18. Deployment, Environments & Infrastructure
19. Performance & Scalability Plan
20. Failure Recovery & Disaster Recovery
21. Seed Data, Demo Projects & Golden Episodes
22. Documentation System
23. Then Pillar 1
24. Pillar 2
25. Pillar 3
...
44. Pillar 22
```

But there is one important correction:

For planning, we planned the 22 pillars first.
For implementation, the **system-level components come first**.

---

# The correct implementation order in simple terms

```text
1. Rules
2. Architecture
3. Contracts
4. Data model
5. Orchestration
6. UI system
7. Testing/CI/security/observability
8. Then pillars
9. Then full pipeline E2E hardening
```

That is the safest order.

---

# What not to do

Do **not** give the agent:

```text
Pillar 1 prompt
Pillar 2 prompt
Pillar 3 prompt
...
then later AGENTS.md
then later tech stack
then later file directory
```

That creates rework.

Do **not** give all 45 prompts at once either.

That overloads the agent and causes shallow implementation.

Do **not** let the agent implement a pillar before these exist:

```text
contracts
status enums
package format
handoff format
audit format
repair ticket format
job format
event format
directory rules
test rules
```

---

# Best implementation strategy

Use three layers:

## Layer 1 — Foundation build

Implement:

```text
repo structure
contracts
database base models
auth/permissions base
audit logs
job system
event bus
worker runtime
agent registry
workflow orchestrator
UI shell
testing harness
CI/CD
observability
```

## Layer 2 — Pillar-by-pillar build

Implement pillars in dependency order:

```text
1 → 2 → 3 → 4 → ... → 22
```

Each pillar plugs into the foundation.

## Layer 3 — Full system hardening

After all pillars exist:

```text
full pipeline E2E tests
golden episode regression tests
workflow graph UI
load tests
failure recovery tests
security review
documentation
demo project
operator manual
```

---

# Best “agent feeding” format

Each implementation task should give the agent a bundle like this:

```text
A. AGENTS.md rules
B. architecture overview
C. relevant shared contracts
D. relevant database models
E. relevant orchestration rules
F. relevant UI rules
G. specific component/pillar prompt
H. expected tests
I. expected deliverable summary
```

For example, when implementing Pillar 12:

```text
Use:
- AGENTS.md
- tech stack
- file structure
- shared Package/Handoff/Job/Event schemas
- orchestration rules
- UI design system
- Pillar 12 implementation prompt
```

---

# My recommended next step

Before continuing to Component 23 deeply, we should create a master document called:

# Implementation Order & Agent Feeding Protocol

This document will tell future coding agents:

```text
what docs to read first
what order to implement components
what must exist before pillars
what to never do
how to handle dependencies
how to handle missing architecture
how to report completion
```

That should probably come before writing more component prompts.
