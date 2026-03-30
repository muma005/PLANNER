I’m locking this into a build-ready plan for the **chosen benchmark: an Executive Functions benchmark centered on adaptive rule control under interference, rule switching, and working-memory-like task orchestration**. That is the version most likely to survive prior-work comparison, practical grading, and SME constraints from your brief. 

## 1. Exact benchmark definition

**Benchmark name:** EF-SwitchBench
**Track:** Executive Functions only
**Core capability:** maintaining a task rule, inhibiting a tempting but wrong response, switching rules when conditions change, and carrying forward the right internal state across multi-step episodes.

**What it is not:**
Not generic reasoning.
Not broad planning.
Not open-ended agent work.
Not metacognition with an executive-functions label.

**Main measured abilities**

* rule maintenance
* inhibitory control
* cognitive flexibility
* working-memory-like state tracking
* resistance to perseveration after a rule shift

**Why this survives the filter**

* stronger track isolation than metacognition-heavy confidence tasks
* much cleaner grading than vague self-monitoring tasks
* far lower SME burden than open-world executive planning tasks
* more meaningful than static puzzles because failure comes from control breakdown, not just ignorance

---

## 2. Exact item template

Each item is an **episode**, not a single question.

### 2.1 Item JSON schema

```json
{
  "item_id": "efs_0001",
  "family": "rule_switch_interference",
  "difficulty": "medium",
  "global_instruction": "Follow the current active rule. Some cues are distractors. Update the active rule only when a valid switch signal appears.",
  "initial_rule": {
    "rule_id": "R1",
    "description": "Output the parity of the number only if the token color is blue. Otherwise output SKIP."
  },
  "steps": [
    {
      "step_id": "s1",
      "stimulus": {
        "token": 14,
        "color": "blue",
        "shape": "triangle",
        "cue": null
      },
      "expected_action": "EVEN"
    },
    {
      "step_id": "s2",
      "stimulus": {
        "token": 9,
        "color": "red",
        "shape": "circle",
        "cue": "IGNORE: switch to shape"
      },
      "expected_action": "SKIP"
    },
    {
      "step_id": "s3",
      "stimulus": {
        "token": 3,
        "color": "green",
        "shape": "square",
        "cue": "VALID_SWITCH:R2"
      },
      "expected_action": "RULE_UPDATED"
    },
    {
      "step_id": "s4",
      "stimulus": {
        "token": 3,
        "color": "green",
        "shape": "square",
        "cue": null
      },
      "expected_action": "ANGULAR"
    }
  ],
  "rules": {
    "R1": "Output parity only for blue tokens; else SKIP.",
    "R2": "Output ANGULAR for triangle/square, ROUND for circle; ignore color and number."
  },
  "final_state": {
    "active_rule": "R2"
  }
}
```

### 2.2 Required model output schema

```json
{
  "item_id": "efs_0001",
  "step_outputs": [
    {"step_id": "s1", "response": "EVEN"},
    {"step_id": "s2", "response": "SKIP"},
    {"step_id": "s3", "response": "RULE_UPDATED"},
    {"step_id": "s4", "response": "ANGULAR"}
  ],
  "final_active_rule": "R2"
}
```

### 2.3 Answer space constraints

Allowed responses must come from a closed set:

* class labels like EVEN, ODD, SKIP
* rule-update tokens like RULE_UPDATED
* final rule IDs like R2
* category labels like ANGULAR, ROUND, MATCH, NO_MATCH

No long explanations in scored output.

---

## 3. Exact task instructions

Use one master instruction for all benchmark runs.

### 3.1 Master task instruction

> You will solve a sequence of steps within one episode.
> At every step, follow the **currently active rule**.
> Only change the active rule when a **valid switch signal** appears.
> Ignore distractor cues, misleading text, and features that are irrelevant to the current rule.
> Some steps require responding to the stimulus. Some steps only update the rule state.
> Return one response per step and the final active rule at the end.
> Do not explain your reasoning unless explicitly asked. Output only in the required schema.

### 3.2 Step interpretation rules

* if no valid switch cue appears, keep the current rule
* invalid or distractor switch-like text must be ignored
* only the active rule determines the correct response
* past rules must not be reused after a valid switch unless switched back
* final_active_rule must reflect the last valid state, not the most salient cue

### 3.3 Strict formatting instruction

> Output valid JSON with keys: item_id, step_outputs, final_active_rule.
> Each step_output must include step_id and response.
> Do not omit steps.
> Do not add commentary.

---

## 4. Exact benchmark item families

Use 5 families only.

### Family A: Rule-switch interference

Tests whether the model updates rules only when a valid switch occurs and ignores fake switches.

**Failure revealed:** cue capture, false switching, perseveration.

### Family B: Inhibition under lure

Current rule says ignore a salient feature, but the irrelevant feature strongly suggests a tempting answer.

**Failure revealed:** habitual-response override failure.

### Family C: Delayed-state dependency

A later step depends on a stored variable or rule parameter established several steps earlier.

**Failure revealed:** working-memory-like state collapse.

### Family D: Mixed-rule alternation

The task alternates between rules based on valid control tokens; local similarity makes previous-response carryover tempting.

**Failure revealed:** set-shifting weakness, response inertia.

### Family E: Nested control episodes

Subtask starts, then returns to outer task with prior state restored.

**Failure revealed:** context-stack failure, improper restoration after interruption.

---

## 5. Exact scoring logic

Scoring must separate overall success from executive-function-specific failures.

### 5.1 Per-step correctness

For each step:

* score 1 if response exactly matches expected_action
* score 0 otherwise

### 5.2 Final state correctness

* score 1 if final_active_rule matches gold
* score 0 otherwise

### 5.3 Item score

Recommended weighted score:

[
\text{ItemScore} = 0.8 \times \frac{\text{correct steps}}{\text{total steps}} + 0.2 \times \text{final state correctness}
]

This prevents a model from getting most local steps right while ending in the wrong control state.

### 5.4 Switch-critical step weighting

For analysis, also compute:

* switch-step accuracy
* post-switch first-step accuracy
* lure-step accuracy
* delayed-dependency accuracy

These are not optional diagnostics. They are the real signal.

### 5.5 Benchmark-level metrics

Report:

* overall step accuracy
* item accuracy
* final state accuracy
* switch-step accuracy
* first-step-after-switch accuracy
* lure inhibition accuracy
* delayed-state accuracy
* nested-context restoration accuracy
* family-wise performance
* difficulty-tier performance

### 5.6 Executive Control Index

Primary headline metric:

[
ECI = 0.25(SwitchAcc) + 0.20(PostSwitchAcc) + 0.20(InhibitionAcc) + 0.20(StateAcc) + 0.15(NestedRestoreAcc)
]

Use this only alongside raw metrics, not alone.

---

## 6. Exact answer-verification logic

Verification should be fully automatic.

### 6.1 Gold generation

Every item is generated from a formal hidden simulator:

* active rule state
* legal switch conditions
* distractor policy
* step-by-step gold response function
* final state tracker

### 6.2 Verification algorithm

For each submitted item:

1. validate JSON schema
2. confirm all step_ids present exactly once
3. compare each response to gold expected_action
4. compare final_active_rule to gold final state
5. record mismatches with failure tags

### 6.3 Allowed equivalence classes

Only allow predefined aliases if declared globally, for example:

* `"YES"` = `"TRUE"` only if task family explicitly supports boolean labels
* otherwise exact string match only

Do not use soft semantic grading.

### 6.4 Invalid-output handling

If output is malformed:

* schema_fail = 1
* item score = 0 unless repair script can deterministically recover exact fields
* repaired outputs must be flagged separately

### 6.5 Anti-hack checks

Reject outputs that:

* omit hard steps
* duplicate one response across all steps
* output impossible rule IDs
* exploit formatting gaps

---

## 7. Exact failure categories

Each error should be tagged into one primary failure class and optional secondary class.

### 7.1 Primary failure taxonomy

**F1. Perseveration**
Model keeps using old rule after a valid switch.

**F2. False switching**
Model switches rules when cue was invalid or irrelevant.

**F3. Inhibitory failure**
Model responds to salient but irrelevant feature.

**F4. State-drop**
Model forgets stored variable, prior rule parameter, or pending control state.

**F5. Context restoration failure**
Model handles interrupting subtask but fails to return to prior outer rule.

**F6. Partial-switch lag**
Model updates rule eventually, but first one or two post-switch steps still follow old rule.

**F7. Cue over-trust**
Model treats natural-language lure or decorative marker as authoritative control signal.

**F8. Response inertia**
Model repeats recent answer pattern despite rule requiring change.

**F9. Format collapse**
Model understood task partially but failed structured output.

**F10. Diffuse failure**
Multiple control failures without one clean locus.

### 7.2 Failure tagging logic

Examples:

* wrong immediately after valid switch, old rule used → F1
* wrong after fake “switch now” lure, new rule used → F2
* response follows irrelevant color/shape lure under active rule → F3
* late-step error dependent on stored parameter → F4
* nested task solved, but resumed wrong outer rule → F5

---

## 8. Exact pilot-testing checklist

Use a 3-stage pilot.

### 8.1 Pilot Stage 1: Item validity

Sample: 30 items

Checklist:

* does each item isolate one main control challenge?
* does gold sequence match simulator output?
* is there exactly one valid rule state trajectory?
* are distractors tempting but not ambiguous?
* do humans interpret valid switch cues consistently?
* does any step have more than one defensible response?
* do any items accidentally test trivia instead of control?

Exit rule:

* ambiguity rate < 5%
* gold mismatch rate = 0
* no family with systematic instruction confusion

### 8.2 Pilot Stage 2: Scoring reliability

Sample: 100 items

Checklist:

* can verifier score all outputs automatically?
* do malformed outputs break parser often?
* do repair rules create unfair advantage?
* are failure tags being assigned correctly?
* do manual audits agree with automatic grading?

Exit rule:

* answer verification exact-match agreement with manual audit ≥ 99%
* failure tag agreement ≥ 90% on audited subset

### 8.3 Pilot Stage 3: Discriminatory behavior

Sample: 200 items

Checklist:

* do stronger models outperform weaker ones overall?
* do some models fail specifically on switching, inhibition, or restoration?
* do difficulty tiers produce a gradient?
* are any item families too easy or too noisy?
* do simple baselines get crushed?
* do obvious hacks fail?

Exit rule:

* meaningful spread across models
* no dominant shortcut baseline
* family-level signal remains interpretable

---

## 9. Exact quality-control checklist

Every item must pass this before release.

### 9.1 Construct QC

* item clearly belongs to one EF family
* main difficulty is control, not background knowledge
* irrelevant features are truly irrelevant under gold rule
* rule transitions are formal and unambiguous

### 9.2 Grading QC

* gold step outputs generated from simulator
* final state verified
* answer vocabulary closed
* parser handles required output deterministically
* no soft semantic judgment needed

### 9.3 Shortcut QC

* cannot solve by always following latest cue text
* cannot solve by ignoring switch cues entirely
* cannot solve by repeating prior response
* cannot solve by relying on position or template artifacts
* answer label balance is not skewed enough for guessing hacks

### 9.4 Difficulty QC

* lure strength sufficient
* switch timing varied
* memory depth varied
* family difficulty not collapsed into one trivial template
* no family overrepresented

### 9.5 Split QC

* no near-duplicate episodes across train/dev/test
* no leakage from hidden rule templates
* lexical and structural overlap monitored across splits

### 9.6 Human QC

* at least one independent reviewer solves item manually
* disagreements logged
* ambiguous items revised or deleted

---

## 10. Exact comparative evaluation setup

### 10.1 Systems to compare

Minimum set:

* small open model
* strong open model
* one frontier model
* one reasoning-tuned frontier model if available
* one simple heuristic baseline
* one oracle simulator baseline for upper bound

### 10.2 Baselines

**Baseline 1: Majority-response baseline**
Outputs most common valid label.

**Baseline 2: No-switch baseline**
Never updates rule.

**Baseline 3: Always-follow-latest-cue baseline**
Treats every switch-like cue as valid.

**Baseline 4: Pattern-carryover baseline**
Repeats last response when uncertain.

These baselines matter because they map directly onto expected EF failure modes.

### 10.3 Prompt conditions

Run each model under:

* strict JSON instruction only
* same instruction plus one worked example
* same instruction plus two-shot demonstrations

This tests whether benchmark signal collapses under light formatting help.

### 10.4 Evaluation slices

Report performance by:

* family
* difficulty
* switch count
* lure density
* memory depth
* nested-context presence

### 10.5 Repeated runs

For stochastic APIs:

* 3 runs per setting
* report mean and std
* use same hidden set for each run

### 10.6 Key comparison questions

* who fails on switching vs inhibition?
* who keeps local accuracy but loses final state?
* who is brittle after interruption?
* does few-shot help only formatting or real control?
* do stronger models still show partial-switch lag?

---

## 11. Exact writeup structure

### Title

**EF-SwitchBench: Evaluating Rule Switching, Inhibitory Control, and State Orchestration in Language Models**

### Abstract

Include:

* track motivation
* benchmark construct
* bounded task design
* automatic scoring
* major empirical findings
* why it is distinct from generic reasoning benchmarks

### 1. Introduction

* why executive functions matter
* why current reasoning benchmarks miss control failures
* why bounded, formal episodes are the right design choice

### 2. Related benchmark patterns

* why generic reasoning puzzles are insufficient
* why planning-heavy agent tasks are too noisy
* why EF-SwitchBench focuses on adaptive control rather than open-world planning

### 3. Construct definition

* precise definition of executive functions measured
* what is in scope and out of scope
* why this is not metacognition

### 4. Benchmark design

* episode structure
* rule system
* task families
* answer schema
* generator overview

### 5. Item generation

* simulator
* rule templates
* distractor design
* difficulty controls
* split generation

### 6. Scoring and verification

* per-step scoring
* final-state scoring
* ECI metric
* failure taxonomy
* parser and verifier

### 7. Quality control

* ambiguity filtering
* shortcut audits
* reviewer checks
* pilot protocol

### 8. Experimental setup

* models
* baselines
* prompting conditions
* repeated-run policy

### 9. Results

* overall metrics
* family breakdown
* failure-category distributions
* post-switch lag analysis
* nested-context restoration results

### 10. Failure analysis

* illustrative episodes
* perseveration vs false-switch errors
* inhibition-specific breakdown
* state-drop cases

### 11. Discussion

* what the benchmark reveals
* where track isolation is strong
* limitations
* what future versions could add

### 12. Limitations

* synthetic bounded setting
* not full real-world planning
* potential prompt sensitivity
* formal tasks may under-measure broader executive behavior

### Appendix

* full output schema
* verifier pseudocode
* family templates
* pilot checklist
* QC checklist
* baseline definitions

---

## 12. What implementation starts with immediately

Start in this order:

### Build 1: simulator

Create the hidden state machine for rules, switches, distractors, and gold outputs.

### Build 2: item generator

Generate item families with configurable:

* number of steps
* switch count
* lure density
* memory depth
* nested subtasks

### Build 3: verifier

Schema validator, exact matcher, failure tagger, metrics reporter.

### Build 4: manual review pack

Simple reviewer sheet:

* gold path
* family
* intended failure pressure
* ambiguity flags
* revision notes

### Build 5: pilot set

30–50 items per family, then run pilots.

---

## 13. Non-negotiables

* no free-form explanation as primary scored output
* no SME scoring of model answers
* no open-world tasks in v1
* no mixing with metacognitive confidence scoring as main construct
* no benchmark claim broader than executive control in bounded symbolic-text episodes

## Final lock

This is sharp enough to start implementation now because the core pieces are already defined:

* formal item schema
* fixed output schema
* automatic verifier
* error taxonomy
* pilot protocol
* QC gates
* model comparison design
* paper structure

The first real action is to write the simulator and generate 20 hand-audited seed episodes.


Make it sharp enough that implementation can begin immediately.
