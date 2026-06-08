For a **single Knowledge Base domain** (e.g. Cinematography KB), I would split it into **10 phases**.

Not because we like phases, but because each phase produces a different asset.

---

# Phase 1 — Knowledge Architecture

Goal:

```text
Define how knowledge is represented.
```

Outputs:

```text
Knowledge model
Knowledge taxonomy
Knowledge lifecycle
Approval states
Governance model
Knowledge health model
Audit model
Versioning model
```

Deliverables:

```text
knowledge contracts
schemas
state machine
governance standards
```

---

# Phase 2 — Source Intelligence System

Goal:

```text
Define where knowledge comes from.
```

Outputs:

```text
Source registry
Authority scoring
Source classifications
Trust framework
Evidence framework
```

Example:

```text
ASC Manual = 98
Random blog = 20
```

Deliverables:

```text
source registry
source ingestion rules
source validation rules
```

---

# Phase 3 — Knowledge Extraction System

Goal:

```text
Convert sources into structured knowledge.
```

Outputs:

```text
Rules
Patterns
Principles
Anti-patterns
Failure modes
Case studies
```

Example:

```text
Book chapter
↓
Structured pattern
```

Deliverables:

```text
extractors
templates
extraction contracts
```

---

# Phase 4 — Knowledge Review & Approval System

Goal:

```text
Prevent bad knowledge entering KB.
```

Outputs:

```text
Review workflow
Approval workflow
Conflict detection
Evidence checks
```

States:

```text
draft
candidate
approved
canonical
deprecated
archived
```

Deliverables:

```text
review engine
approval engine
conflict checker
```

---

# Phase 5 — Core Knowledge Content

Goal:

```text
Actually build cinematography intelligence.
```

Outputs:

```text
Shot Language Library
Camera Pattern Library
Composition Library
Lens Library
Movement Library
Shot Sequence Grammar
```

This will be the largest phase.

---

# Phase 6 — Validation & Failure Intelligence

Goal:

```text
Teach Rafiki what bad cinematography looks like.
```

Outputs:

```text
Failure modes
Detection rules
Correction rules
Validation targets
```

Examples:

```text
weak reveal
confusing geography
bad eyeline
crossed line
unmotivated movement
```

---

# Phase 7 — Agent Access Layer

Goal:

```text
Allow agents to safely use KB.
```

Outputs:

```text
retrieval APIs
query system
pattern selector
reasoning layer
```

Agent flow:

```text
Scene
↓
KB Query
↓
Approved Patterns
↓
Camera Blueprint
```

---

# Phase 8 — Knowledge Testing System

Goal:

```text
Prove knowledge works.
```

Outputs:

```text
Golden scene tests
Pattern tests
Regression tests
Knowledge certification tests
```

Example:

```text
Dramatic irony scene
↓
Generated camera packet
↓
Matches expected strategy
```

---

# Phase 9 — Audit & Governance System

Goal:

```text
Keep KB healthy over years.
```

Outputs:

```text
Knowledge health scoring
Periodic audits
Staleness detection
Usage analytics
Conflict analytics
```

Questions:

```text
Still valid?
Still useful?
Still trusted?
```

---

# Phase 10 — Studio Learning System

Goal:

```text
Allow Rafiki to improve itself.
```

Outputs:

```text
Operator feedback ingestion
Production learnings
Pattern success tracking
Pattern evolution
Canonical promotion system
```

Example:

```text
Pattern used 200 times
Success rate 96%

↓

Promote to Canonical
```

---

# What This Means For Rafiki

For **every major creative KB**:

```text
Cinematography KB
Acting KB
Lighting KB
Editing KB
Music KB
Storytelling KB
```

the architecture is almost identical.

So after we design Cinematography KB, we can reuse about:

```text
70-80%
```

of the infrastructure for the others.

The only thing that changes is the actual knowledge content.

So the Cinematography KB becomes the **template knowledge system** for the entire Rafiki Creative Intelligence Platform. Once it works, the other KBs follow the same governance, approval, audit, testing, and retrieval architecture.
