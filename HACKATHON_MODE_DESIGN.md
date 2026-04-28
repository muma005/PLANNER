# Professor Hackathon Mode — Full Design

> **This is an EXTENSION to v2, not a rewrite.**
> Triggered by: `professor run --mode hackathon --competition {name}`
> Reuses: Self-Debugging Engine, HITL, Cost Governor, Data Integrity, Code Ledger, Freeform Sandbox, Statistical Gates
> Adds: 4 new components, 2 modified components, 1 new orchestration graph

---

## The Core Problem

Professor v2 optimizes a SINGLE NUMBER. Hackathons judge on 4-6 weighted criteria simultaneously (clinical relevance, technical quality, documentation, insight, novelty). A system that maximizes AUC while ignoring writeup quality scores 30/100 on Triagegeist. A system that writes beautiful narrative with a broken notebook scores 20/100.

Professor needs to read what the judges are scoring on and allocate effort proportionally across ALL criteria.

---

## What Wins Hackathons — The Invariant Pattern

After analyzing competitions across domains, every winning hackathon entry follows the same structure regardless of field:

**1. The entrant chose a SHARP angle within a broad objective.** The competition says "address a problem in emergency triage." The winner doesn't build a generic triage predictor — they identify that ESI undertriages elderly patients with atypical cardiac presentations, and prove it with data. The sharpness of the angle is what scores 21-25 on clinical relevance instead of 14-20.

**2. The entrant brought data the competition didn't provide.** Triagegeist says "use any publicly available dataset." The winner uses MIMIC-IV-ED (recommended) PLUS published ESI inter-rater reliability studies to quantify the human baseline error rate. The external data enables the novel analysis.

**3. The entrant's features tested a HYPOTHESIS, not just improved a score.** Instead of "these 50 features improve AUC," the winner shows "this specific interaction between age × chief_complaint_category × time_of_day predicts ESI disagreement between raters, and here's why that matters clinically."

**4. The entrant's visualizations argued for the finding.** Not a correlation heatmap. A comparison plot showing: "Model accuracy on typical presentations (left) vs atypical presentations (right), stratified by patient age. The accuracy gap widens for patients over 65 — exactly the population where undertriage is most dangerous."

**5. The writeup reads like a clinical research abstract crossed with a compelling blog post.** Sharp problem statement, clear methodology, honest results with effect sizes, limitations acknowledged, clinical implications stated.

This pattern is DOMAIN-AGNOSTIC. Swap "clinical relevance" for "soccer domain knowledge" and the structure is identical.

---

## Architecture — What Changes, What Stays

### Unchanged (reused from v2 traditional mode)

| Component | Hackathon Use |
|---|---|
| ProfessorState | Same — add hackathon-specific fields |
| Self-Debugging Engine | Same — catches code errors |
| HITL (CLI + Telegram) | CRITICAL — operator directs thesis selection |
| Cost Governor | Same — budget management |
| Data Integrity Checkpoints | Same — row count + schema validation |
| State Schema Enforcement | Same — typed state |
| Code Ledger | Same — provenance for notebook generation |
| Pre-Flight Checks | Same — data profiling |
| Freeform Sandbox | MORE IMPORTANT — operator directs novel analyses |
| Statistical Gates | REPURPOSED — validate hypotheses, not just features |
| Metric Verification Gate | Modified — verify competition-specific evaluation if metric exists |
| Solution Assembler | Modified — generates hackathon-format notebook |

### New Components (4)

1. **Rubric Parser** — reads competition page, extracts judging criteria + weights, builds effort allocation plan
2. **Thesis Generator** — proposes 3-5 novel research angles from domain + data + rubric
3. **External Data Scout** — finds, evaluates, and integrates external datasets
4. **Narrative Engine** — argument-driven visualizations + template-aware writeup

### Modified Components (2)

5. **Hypothesis Feature Factory** — Feature Factory with modified prompt purpose and gate question
6. **Hackathon Graph Builder** — alternative LangGraph graph with different agent ordering

---

## Component 1 — Rubric Parser (tools/rubric_parser.py)

### Failure mode combated

The operator enters a hackathon and allocates effort evenly across all activities. But Triagegeist weights Technical Quality at 30% and Novelty at only 10%. The operator spends 40% of their time on novelty and 15% on technical quality. Effort is misallocated. Score suffers.

### What it does

Reads the competition description page (scraped by Competition Intel or provided by operator) and extracts:

```python
@dataclass
class HackathonRubric:
    criteria: list  # [{name, weight, description, scoring_levels}]
    submission_requirements: list  # ["notebook", "writeup", "cover_image", "project_link"]
    writeup_template: dict  # Required sections from the competition
    data_policy: str  # "any_public" | "provided_only" | "specific_sources"
    recommended_datasets: list  # [{name, url, description}]
    deliverable_type: str  # "notebook_and_writeup" | "application" | "analysis"
    max_writeup_words: int  # e.g., 2000 for Triagegeist
    tracks: list  # Competition tracks if multiple exist
```

### How it works

**Step 1 — Extract rubric from competition text:**

The competition description (scraped or pasted by operator via HITL) is fed to `llm_call()`:

```
Read this hackathon competition page and extract the judging rubric.

COMPETITION TEXT:
{competition_description}

Extract as JSON:
{
    "criteria": [
        {
            "name": "Clinical Relevance",
            "weight": 25,
            "max_points": 25,
            "description": "Does the submission address a real problem...",
            "top_score_description": "Problem is sharply defined...",
            "bottom_score_description": "Problem has negligible relevance..."
        }
    ],
    "submission_requirements": ["notebook", "writeup", "cover_image", "project_link"],
    "writeup_sections": ["clinical_problem_statement", "methodology", "results", "limitations", "reproducibility"],
    "data_policy": "any_public",
    "recommended_datasets": [{"name": "MIMIC-IV-ED", "url": "physionet.org", "description": "..."}],
    "max_writeup_words": 2000
}

If no explicit rubric exists, infer criteria from the description.
If no point weights exist, assign equal weights.
```

**Step 2 — Build effort allocation plan:**

```python
def build_effort_plan(rubric: HackathonRubric) -> dict:
    """
    Maps rubric criteria to Professor agent effort allocation.
    
    Returns:
    {
        "thesis_depth": "deep" | "standard" | "light",
        "technical_depth": "marathon" | "standard" | "sprint",
        "writeup_depth": "deep" | "standard" | "light",
        "external_data_priority": "high" | "medium" | "skip",
        "visualization_count": int,
        "narrative_polish_passes": int,
    }
    """
    # Map criteria to effort dimensions
    technical_weight = sum(c["weight"] for c in rubric.criteria 
                          if any(kw in c["name"].lower() for kw in ["technical", "methodology", "code", "model"]))
    novelty_weight = sum(c["weight"] for c in rubric.criteria 
                         if any(kw in c["name"].lower() for kw in ["novel", "impact", "creative", "original"]))
    documentation_weight = sum(c["weight"] for c in rubric.criteria 
                               if any(kw in c["name"].lower() for kw in ["document", "writeup", "write-up", "clarity"]))
    domain_weight = sum(c["weight"] for c in rubric.criteria 
                        if any(kw in c["name"].lower() for kw in ["clinical", "domain", "relevance", "problem"]))
    insight_weight = sum(c["weight"] for c in rubric.criteria 
                         if any(kw in c["name"].lower() for kw in ["insight", "finding", "result", "analysis"]))
    
    total = technical_weight + novelty_weight + documentation_weight + domain_weight + insight_weight
    if total == 0:
        total = 100  # Fallback: equal distribution
    
    return {
        # Technical Quality drives pipeline depth
        "technical_depth": (
            "marathon" if technical_weight / total > 0.30 else
            "standard" if technical_weight / total > 0.15 else
            "sprint"
        ),
        
        # Domain Relevance + Novelty drive thesis depth
        "thesis_depth": (
            "deep" if (domain_weight + novelty_weight) / total > 0.30 else
            "standard" if (domain_weight + novelty_weight) / total > 0.15 else
            "light"
        ),
        
        # Documentation drives writeup polish
        "writeup_depth": (
            "deep" if documentation_weight / total > 0.20 else
            "standard" if documentation_weight / total > 0.10 else
            "light"
        ),
        
        # Novelty + Insight drive external data acquisition
        "external_data_priority": (
            "high" if (novelty_weight + insight_weight) / total > 0.20 else
            "medium" if (novelty_weight + insight_weight) / total > 0.10 else
            "skip"
        ),
        
        # Insight drives visualization count
        "visualization_count": (
            7 if insight_weight / total > 0.15 else
            5 if insight_weight / total > 0.10 else
            3
        ),
        
        # Documentation weight drives polish passes
        "narrative_polish_passes": (
            3 if documentation_weight / total > 0.20 else
            2 if documentation_weight / total > 0.10 else
            1
        ),
    }
```

**Example — Triagegeist rubric parsed:**

```
Clinical Relevance:   25 pts → domain_weight = 25
Technical Quality:    30 pts → technical_weight = 30
Documentation:        20 pts → documentation_weight = 20
Insight & Findings:   15 pts → insight_weight = 15
Novelty & Impact:     10 pts → novelty_weight = 10

Result:
  technical_depth:       "marathon"   (30% — full ML pipeline with deep Optuna)
  thesis_depth:          "deep"       (25+10=35% domain+novelty — aggressive thesis generation)
  writeup_depth:         "deep"       (20% — 3 polish passes on writeup)
  external_data_priority: "high"      (10+15=25% novelty+insight — find supplementary data)
  visualization_count:   7            (15% insight)
  narrative_polish_passes: 3          (20% documentation)
```

**Example — a hypothetical novelty-heavy art hackathon:**

```
Novelty:            40 pts → novelty_weight = 40
Presentation:       30 pts → documentation_weight = 30
Technical:          15 pts → technical_weight = 15
Impact:             15 pts → insight_weight = 15

Result:
  technical_depth:       "sprint"     (15% — minimal model, just needs to work)
  thesis_depth:          "deep"       (40% novelty — invest heavily in unique angle)
  writeup_depth:         "deep"       (30% — maximum polish)
  external_data_priority: "high"      (40+15=55% — novel data IS the differentiator)
  visualization_count:   7
  narrative_polish_passes: 3
```

The same system handles both competitions correctly because it reads the rubric and adapts.

### HITL integration

After parsing, present the rubric interpretation to the operator:

```
📋 RUBRIC ANALYSIS

Competition: Triagegeist
Total points: 100

Criteria breakdown:
  Clinical Relevance:   25 pts (25%) → DEEP thesis depth + domain research
  Technical Quality:    30 pts (30%) → MARATHON pipeline depth
  Documentation:        20 pts (20%) → 3-pass writeup polish
  Insight & Findings:   15 pts (15%) → 7 narrative visualizations
  Novelty & Impact:     10 pts (10%) → External data search HIGH priority

Required deliverables: notebook, writeup (max 2000 words), cover image, project link
Writeup sections: clinical problem statement, methodology, results, limitations, reproducibility
Data policy: any publicly available dataset

Effort plan looks correct? Reply /continue or adjust
```

### State additions

```python
hackathon_rubric: dict = Field(default_factory=dict)       # Parsed rubric
hackathon_effort_plan: dict = Field(default_factory=dict)   # Effort allocation
hackathon_mode: bool = False                                # True when in hackathon mode
hackathon_writeup_template: dict = Field(default_factory=dict)  # Required sections
```

### Contract tests

1. `test_triagegeist_rubric_parsed` — Triagegeist competition text → 5 criteria extracted with correct weights
2. `test_effort_plan_technical_heavy` — 30% technical weight → technical_depth="marathon"
3. `test_effort_plan_novelty_heavy` — 40% novelty → thesis_depth="deep", external_data="high"
4. `test_equal_weights_when_no_rubric` — no explicit rubric → equal weights inferred
5. `test_writeup_sections_extracted` — Triagegeist → ["clinical_problem_statement", "methodology", "results", "limitations", "reproducibility"]
6. `test_recommended_datasets_extracted` — MIMIC-IV-ED appears in recommended_datasets

---

## Component 2 — Thesis Generator (agents/thesis_generator.py)

### Failure mode combated

The participant builds "a generic triage prediction model." Every other team does the same thing. All score 14-20 on Clinical Relevance (relevant but not sharply defined) and 3-5 on Novelty (follows established approaches). The winner chose a SHARP angle that scores 21-25 on Clinical Relevance and 9-10 on Novelty.

The thesis is the single highest-leverage decision in a hackathon. Professor must generate strong thesis candidates and present them to the operator for selection.

### What it does

Takes domain research + EDA + rubric and generates 3-5 thesis candidates ranked by estimated score across ALL rubric criteria.

### The thesis generation prompt

```python
THESIS_PROMPT = """You are simultaneously:
1. A senior practitioner in {domain} with 15 years of clinical/field experience
2. A data scientist who wins Kaggle hackathons
3. A judge evaluating submissions against this rubric: {rubric_criteria}

COMPETITION OBJECTIVE: {competition_description}
DATA AVAILABLE: {schema + EDA summary}
DOMAIN KNOWLEDGE: {domain_brief}
EXTERNAL DATA AVAILABLE: {recommended_datasets}
OPERATOR CONTEXT: {hitl_injections}

Generate exactly 5 THESIS CANDIDATES for this hackathon.

Each thesis must:
- Address a SPECIFIC problem, not the general objective
- Be CONDITIONAL: "X behaves differently under condition Y" or "Z is systematically biased for population W"
- Be TESTABLE with available + acquirable data
- Score HIGH on the rubric's top-weighted criterion ("{top_criterion_name}" at {top_criterion_weight}%)
- Matter to a PRACTITIONER who would read this analysis and change their behavior

For each thesis, estimate scores on EACH rubric criterion:

{for each criterion in rubric:}
  - {criterion.name} ({criterion.weight} pts): estimated score /  {criterion.max_points}
    Justification: why this thesis scores this on {criterion.name}

Respond with JSON:
[
    {
        "thesis_id": 1,
        "statement": "One-sentence thesis",
        "angle": "The specific novel angle — what makes this different from the obvious approach",
        "target_audience": "Who would use this finding (e.g., ED nurses, triage algorithm designers, hospital administrators)",
        "data_plan": {
            "primary_dataset": "What competition/recommended data to use",
            "external_needed": ["What additional data strengthens this"],
            "join_strategy": "How to combine primary + external"
        },
        "condition_variable": "The variable that creates the conditional split",
        "hypothesis": "Specific testable prediction — what we expect to find",
        "estimated_scores": {
            "Clinical Relevance": {"score": 22, "justification": "..."},
            "Technical Quality": {"score": 25, "justification": "..."},
            ...
        },
        "estimated_total": 82,
        "feasibility": "high" | "medium" | "low",
        "risk": "What could go wrong — what if the data doesn't support the thesis"
    }
]

RANK by estimated_total (highest first).
"""
```

### Thesis presentation to operator (GATE — operator MUST choose)

```
🔬 THESIS PROPOSALS (ranked by estimated total score)

1. [EST: 84/100] "ESI undertriages elderly patients presenting 
   with atypical cardiac symptoms — standard vital sign thresholds 
   miss age-adjusted cardiac risk"
   
   Angle: Not generic triage prediction — specifically targets 
   the intersection of age × atypical presentation where ESI 
   is known to fail in clinical literature
   
   Clinical Relevance: 23/25 — sharply defined, documented problem
   Technical Quality: 26/30 — standard ML approach is sufficient
   Documentation: 16/20 — clear narrative arc
   Insight: 12/15 — age-adjusted thresholds would be actionable
   Novelty: 7/10 — applies known clinical concern to data analysis
   
   Data: MIMIC-IV-ED + AHA cardiac risk thresholds + ESI protocol docs
   Risk: MIMIC-IV-ED may not have enough atypical cardiac cases

2. [EST: 79/100] "Night shift triage accuracy degrades independent 
   of patient volume — cognitive fatigue creates systematic 
   undertriage of medium-acuity patients"
   
   Angle: Isolates cognitive fatigue from volume effects by 
   controlling for census. Medium-acuity patients are the most 
   judgment-dependent and most affected.
   
   Clinical Relevance: 21/25 — relevant but less clinical urgency
   Technical Quality: 25/30 — requires careful causal analysis
   Documentation: 16/20 — compelling narrative about shift effects
   Insight: 11/15 — staffing implications are real
   Novelty: 6/10 — shift effects are known, but per-acuity is new
   
   Data: MIMIC-IV-ED + published shift fatigue literature
   Risk: MIMIC-IV-ED timestamps may not distinguish shift boundaries

3. [EST: 76/100] "Chief complaint text contains undertriage 
   signals that structured vital signs miss — NLP extraction 
   of symptom trajectory from free text improves severity prediction"
   
   ...

4. ...
5. ...

Select a thesis (1-5) or describe your own: ___
```

The operator chooses. This is the creative direction decision. Professor generates candidates, the operator's domain intuition picks the winner.

### If operator describes their own thesis

Professor validates it against the rubric:

```python
# Operator types: "I want to analyze whether certain chief complaint 
# keywords predict which patients will be upgraded in severity after 
# initial triage"

# Professor evaluates:
custom_thesis_eval = llm_call(
    f"Evaluate this thesis against the rubric:\n"
    f"Thesis: {operator_thesis}\n"
    f"Rubric: {rubric_criteria}\n"
    f"Estimate scores for each criterion with justification.\n"
    f"Flag any risks or feasibility concerns."
)

# Present evaluation to operator for confirmation
```

### AUTONOMOUS mode (no operator)

When running unattended, Professor selects the highest-scoring thesis automatically. This is suboptimal (operator judgment is the competitive advantage) but functional for batch/benchmark runs.

### State additions

```python
thesis_candidates: list = Field(default_factory=list)   # All generated theses
active_thesis: dict = Field(default_factory=dict)        # Selected thesis
thesis_selected_by: str = ""                             # "operator" | "auto"
```

### Contract tests

1. `test_generates_5_candidates` — exactly 5 thesis candidates generated
2. `test_each_has_required_fields` — thesis_id, statement, angle, data_plan, estimated_scores, feasibility
3. `test_scores_cover_all_rubric_criteria` — each thesis has estimated score for every criterion in rubric
4. `test_ranked_by_total` — first thesis has highest estimated_total
5. `test_condition_variable_present` — each thesis has a condition_variable (the conditional split)
6. `test_hypothesis_is_testable` — each thesis has a specific, testable hypothesis
7. `test_operator_selection_stored` — after GATE response, active_thesis is populated
8. `test_auto_selects_highest_when_autonomous` — in AUTONOMOUS mode, thesis 1 is auto-selected

---

## Component 3 — External Data Scout (agents/external_data_scout.py)

### Failure mode combated

The winner uses MIMIC-IV-ED plus two supplementary datasets. The loser uses only MIMIC-IV-ED. The supplementary data enables the novel analysis that scores 9-10 on Novelty instead of 3-5.

### What it does

Given the active thesis's `data_plan.external_needed`, searches for, evaluates, and integrates external datasets.

### Search strategy (ordered by reliability)

```python
SEARCH_SOURCES = [
    {
        "name": "competition_recommended",
        "description": "Datasets explicitly recommended by the competition",
        "method": "direct_download",  # URLs provided in competition description
        "reliability": "highest",
    },
    {
        "name": "kaggle_datasets",
        "description": "Kaggle Datasets API search",
        "method": "api",  # kaggle datasets list -s "{query}"
        "reliability": "high",
    },
    {
        "name": "government_portals",
        "description": "data.gov, WHO, CDC, Eurostat, NHS Digital",
        "method": "web_search",  # Search for specific domain data
        "reliability": "high",
    },
    {
        "name": "academic_repositories",
        "description": "UCI ML Repository, PhysioNet, OpenML",
        "method": "web_search",
        "reliability": "medium",
    },
    {
        "name": "domain_specific",
        "description": "Sports: FBref, Statsbomb. Health: WHO, PubMed data. Finance: FRED, SEC.",
        "method": "web_search",
        "reliability": "medium",
    },
    {
        "name": "reference_tables",
        "description": "Published thresholds, clinical guidelines, lookup tables",
        "method": "web_search + manual_creation",
        "reliability": "medium",  # Often needs manual structuring
    },
]
```

### Evaluation per candidate dataset

For each found dataset, compute a feasibility score:

```python
@dataclass
class DatasetCandidate:
    name: str
    source_url: str
    description: str
    
    # Feasibility dimensions
    join_feasibility: float   # 0-1: can it be joined to competition data?
    relevance_to_thesis: float  # 0-1: does it help test the thesis?
    size_compatible: bool     # Will it fit in memory alongside competition data?
    license_compatible: bool  # Does license allow use in competition?
    download_accessible: bool # Can we actually download it programmatically?
    
    # Join details
    join_key: str             # What column to join on (date, ID, category code)
    join_type: str            # "left" | "inner" | "lookup_table" | "enrichment"
    estimated_match_rate: float  # What % of competition rows will match?
    
    overall_score: float      # Weighted combination of all dimensions

def evaluate_candidate(candidate, thesis, competition_data_schema) -> DatasetCandidate:
    """
    LLM evaluates whether this dataset helps test the thesis.
    """
    eval_prompt = f"""
    THESIS: {thesis.statement}
    THESIS DATA NEED: {thesis.data_plan.external_needed}
    COMPETITION DATA SCHEMA: {competition_data_schema}
    
    CANDIDATE DATASET:
    Name: {candidate.name}
    Description: {candidate.description}
    
    Evaluate:
    1. Can this dataset be JOINED to the competition data? On what key?
    2. Does this dataset help TEST the thesis specifically?
    3. What percentage of competition rows would match after join?
    
    Respond with JSON: {{join_feasibility, relevance, join_key, join_type, match_rate_estimate}}
    """
```

### Integration (download + join)

For approved datasets (top 1-3 by overall_score):

```python
def integrate_external_dataset(
    candidate: DatasetCandidate,
    competition_data_path: str,
    session_dir: str,
) -> dict:
    """
    Download external dataset and join to competition data.
    Executed via run_in_sandbox() with full safety infrastructure.
    """
    # Generate join code
    join_prompt = f"""
    Write Polars code to:
    1. Load the external dataset from {candidate.download_path}
    2. Load the competition data from {competition_data_path}
    3. Join on key: {candidate.join_key} using {candidate.join_type} join
    4. Verify: row count of competition data is PRESERVED (no duplication)
    5. Save enriched dataset to {session_dir}/enriched_data.parquet
    6. Print summary: matched rows, new columns added, any unmatched rows
    """
    
    result = run_in_sandbox(
        code=generated_join_code,
        agent_name="external_data_scout",
        purpose=f"Join external dataset: {candidate.name}",
        expected_row_change="none",  # Row count MUST be preserved
    )
```

### Safety caps (NON-NEGOTIABLE)

```python
MAX_EXTERNAL_DATASETS = 3        # More adds complexity without proportional value
ROW_COUNT_MUST_PRESERVE = True   # Joins cannot duplicate or drop competition rows
MUST_CITE_ALL_SOURCES = True     # Every external dataset cited in writeup with URL
MUST_VERIFY_LICENSE = True       # Flagged to operator if license is unclear
```

### Graceful degradation

If no relevant external data is found:
- The thesis is tested with competition data only
- The writeup includes a "Future Work" section: "This analysis could be strengthened with [specific data type]. We were unable to acquire [data] within the competition timeframe."
- This is actually GOOD in a hackathon writeup — it shows the entrant thought beyond the provided data

### State additions

```python
external_datasets: list = Field(default_factory=list)       # Acquired datasets
external_data_paths: list = Field(default_factory=list)      # File paths
thesis_data_sufficient: bool = True                          # Whether enough data to test thesis
enriched_data_path: str = ""                                 # Path to joined dataset
```

### Contract tests

1. `test_recommended_datasets_searched_first` — competition-recommended datasets checked before web search
2. `test_join_feasibility_evaluated` — each candidate has join_key and match_rate_estimate
3. `test_row_count_preserved_after_join` — enriched data has same row count as competition data
4. `test_max_3_external_datasets` — even with 10 candidates, only top 3 integrated
5. `test_sources_tracked_for_citation` — each dataset has source_url for writeup citation
6. `test_graceful_when_none_found` — no external data → thesis_data_sufficient may be False but no crash
7. `test_license_flagged_when_unclear` — dataset with no clear license → HITL warning

---

## Component 4 — Narrative Engine (tools/narrative_engine.py)

### What this does

Two capabilities in one component:
A) Generates argument-supporting visualizations (not diagnostic EDA plots)
B) Generates a template-aware argumentative writeup (not documentation)

### Part A — Argument Visualizations

The EDA Artifact Export generates 7 diagnostic plots for the OPERATOR to understand the data. The Narrative Engine generates 3-7 THESIS-SPECIFIC plots for the JUDGES to understand the finding.

**The key difference:** EDA plots ask "what does this data look like?" Narrative plots ask "does this data support the thesis?"

```python
def generate_narrative_plots(
    thesis: dict,
    thesis_features: list,
    effect_sizes: dict,
    enriched_data_path: str,
    session_dir: str,
    n_plots: int,  # From effort_plan.visualization_count
) -> list[dict]:
    """
    Generate thesis-supporting visualizations.
    Each plot supports ONE claim in the narrative.
    
    Returns list of:
    {
        "path": str,
        "plot_type": str,
        "claim_supported": str,  # The specific claim this plot supports
        "caption": str,
        "figure_number": int,
    }
    """
```

**Plot type selection driven by thesis structure:**

```python
def _select_plot_types(thesis: dict, data_schema: dict) -> list[str]:
    """
    Choose plot types based on what the thesis needs to show.
    """
    plot_types = []
    
    # Every thesis gets a conditional comparison (the core finding)
    plot_types.append("conditional_comparison")
    
    # If thesis has a condition variable that's categorical:
    # → grouped bar chart or side-by-side distributions
    if thesis["condition_variable_type"] == "categorical":
        plot_types.append("grouped_bar")
    
    # If thesis has a condition variable that's temporal:
    # → time series showing effect over time
    if thesis["condition_variable_type"] == "temporal":
        plot_types.append("temporal_effect")
    
    # If thesis involves interaction between two conditions:
    # → heatmap of effect by condition1 × condition2
    if len(thesis.get("moderator_variables", [])) > 0:
        plot_types.append("interaction_heatmap")
    
    # If thesis compares model vs baseline:
    # → calibration plot or ROC comparison
    if thesis.get("involves_model_comparison"):
        plot_types.append("model_comparison")
    
    # Every thesis gets a summary infographic
    plot_types.append("finding_summary")
    
    # Fill remaining slots with exploratory plots
    while len(plot_types) < n_plots:
        plot_types.append("supporting_evidence")
    
    return plot_types[:n_plots]
```

**Plot generation via LLM + sandbox:**

For each plot, build a SPECIFIC prompt:

```python
def _build_plot_prompt(plot_type, thesis, effect_sizes, data_path):
    return f"""
    Generate matplotlib/seaborn code for a {plot_type} visualization.
    
    THESIS: "{thesis['statement']}"
    CLAIM THIS PLOT SUPPORTS: "{thesis['hypothesis']}"
    KEY STATISTIC: effect_size={effect_sizes['primary']}, p={effect_sizes['p_value']}
    
    DATA: loaded from {data_path}
    CONDITION VARIABLE: {thesis['condition_variable']}
    
    REQUIREMENTS:
    - Title states the FINDING, not the plot type
      GOOD: "Elderly patients with atypical symptoms are undertriaged 2.3x more often"
      BAD: "Figure 1: Distribution of triage accuracy by age group"
    - Annotate the key statistic directly on the plot (effect size, p-value)
    - Use colorblind-friendly palette (seaborn 'colorblind' or 'Set2')
    - Font sizes: title=14, axis labels=12, tick labels=10, annotations=11
    - Figure size: (10, 6) at 150 DPI
    - Remove chart junk: no gridlines unless they aid reading, no box around legend
    - If comparing groups: highlight the GAP between them, not just the individual values
    
    Save to: {session_dir}/narrative_plots/fig_{figure_number}_{plot_type}.png
    """
```

### Part B — Argumentative Writeup

The existing Solution Writeup Generator documents what Professor DID. The Narrative Engine writes an ARGUMENT for what Professor FOUND.

**Structure follows the competition's writeup template:**

```python
def generate_hackathon_writeup(
    state: ProfessorState,
    session_dir: str,
    rubric: HackathonRubric,
    thesis: dict,
    effect_sizes: dict,
    narrative_plots: list,
    polish_passes: int,
) -> str:
    """
    Generate the hackathon writeup as the PRIMARY deliverable.
    
    Adapts structure to the competition's required sections.
    Reads rubric.writeup_template for required sections.
    """
```

**For Triagegeist (clinical writeup template):**

```markdown
# [Compelling Title — States the Finding]
"Unseen Risk: How Atypical Symptom Presentations Lead to Systematic 
Undertriage in Elderly Emergency Patients"

## Clinical Problem Statement (scores on: Clinical Relevance, 25 pts)

[NOT: "Triage is important." 
 YES: "Emergency Severity Index (ESI) assigns acuity levels based 
 on vital sign thresholds calibrated for typical symptom presentations. 
 When elderly patients present with atypical cardiac symptoms — 
 fatigue and confusion rather than chest pain — their vital signs 
 may remain within ESI's 'normal' ranges despite elevated cardiac risk. 
 We quantify this undertriage gap using MIMIC-IV-ED data and 
 demonstrate that age-adjusted thresholds reduce severity 
 misclassification by {effect_size}%."]

## Methodology (scores on: Technical Quality, 30 pts)

[Data sources with citations]
[Feature engineering — specifically the THESIS-TESTING features]
[Model architecture with justification]
[Validation strategy — why this approach is rigorous]
[How we defined "atypical presentation" operationally]

## Results (scores on: Insight & Findings, 15 pts)

[Figure 1: The core finding visualization]
[Statistical evidence: effect sizes, confidence intervals, p-values]
[NOT: "AUC was 0.87." YES: "The model identifies 73% of undertriaged 
elderly patients that ESI misses, with a false positive rate of 12%.
Figure 2 shows the accuracy gap widens for patients over 75."]

## Discussion & Clinical Implications

[Who would use this: ED triage nurses, ESI algorithm designers]
[How it would change practice: age-adjusted thresholds for atypical presentations]
[What the foundation could do with this: pilot study design]

## Limitations (scores on: Documentation, 20 pts — honesty here is REWARDED)

[MIMIC-IV-ED is single-center Boston data — generalization unknown]
["Atypical presentation" definition is operational, may miss some cases]
[Retrospective analysis — cannot prove causal effect of undertriage on outcomes]

## Reproducibility Notes

[All code in attached notebook — runs end-to-end]
[External data sources cited with access instructions]
[Random seeds pinned throughout]
```

**The writeup word budget:**

Triagegeist allows 2000 words. Professor must allocate words proportionally to rubric weights:

```python
def _allocate_word_budget(max_words: int, rubric: HackathonRubric) -> dict:
    """
    Allocate writeup word budget based on rubric weights.
    """
    total_weight = sum(c["weight"] for c in rubric.criteria)
    
    # Map criteria to writeup sections
    section_weights = {}
    for criterion in rubric.criteria:
        # Find which writeup section this criterion maps to
        section = _map_criterion_to_section(criterion, rubric.writeup_template)
        section_weights[section] = section_weights.get(section, 0) + criterion["weight"]
    
    # Allocate words proportionally (with minimums)
    word_budget = {}
    for section, weight in section_weights.items():
        words = max(150, int(max_words * weight / total_weight))
        word_budget[section] = words
    
    return word_budget
```

**Polish passes:**

The effort plan specifies how many LLM passes to spend on writeup quality:

```python
for pass_num in range(polish_passes):
    writeup = llm_call(
        f"You are an expert editor for data science competition writeups.\n\n"
        f"RUBRIC (what judges score on):\n{rubric_criteria}\n\n"
        f"CURRENT WRITEUP (pass {pass_num + 1} of {polish_passes}):\n{writeup}\n\n"
        f"IMPROVE the writeup. Specifically:\n"
        f"- Sharpen the problem statement (scores on Clinical Relevance)\n"
        f"- Ensure methodology is clearly justified (scores on Technical Quality)\n"
        f"- Make findings SPECIFIC with exact numbers (scores on Insight)\n"
        f"- Acknowledge limitations honestly (scores on Documentation)\n"
        f"- Remove any generic filler that doesn't serve the argument\n"
        f"- Ensure word count is under {max_words}\n"
        f"\nReturn the improved writeup only.",
        agent_name="narrative_engine",
    )
```

### State additions

```python
narrative_plots: list = Field(default_factory=list)         # Argument visualizations
hackathon_writeup_path: str = ""                            # Path to final writeup
hackathon_writeup_word_count: int = 0                       # Current word count
hackathon_writeup_polish_pass: int = 0                      # Current polish pass
```

### Contract tests

1. `test_narrative_plots_generated` — n_plots matching effort_plan.visualization_count
2. `test_plot_titles_state_findings` — no plot title starts with "Figure N:" — must state the finding
3. `test_plot_has_claim_supported` — each plot dict has non-empty claim_supported field
4. `test_writeup_follows_template` — writeup contains all sections from rubric.writeup_template
5. `test_writeup_under_word_limit` — word count ≤ max_writeup_words
6. `test_writeup_contains_effect_sizes` — writeup contains at least 3 numeric results
7. `test_writeup_cites_external_data` — every external dataset URL appears in writeup
8. `test_polish_passes_executed` — writeup goes through effort_plan.narrative_polish_passes iterations
9. `test_limitations_section_present` — writeup has a limitations section (honesty scores points)

---

## Component 5 — Hypothesis Feature Factory (MODIFIED Feature Factory)

### What changes

The Feature Factory's code and infrastructure are UNCHANGED. The PROMPT changes. In traditional mode, the prompt says "generate features that improve CV." In hackathon mode, the prompt says "generate features that test the thesis."

```python
def _build_hackathon_feature_prompt(state, round_num):
    """
    Modified feature generation prompt for hackathon mode.
    """
    return f"""
    You are testing this thesis: "{state.active_thesis['statement']}"
    
    Hypothesis: {state.active_thesis['hypothesis']}
    Condition variable: {state.active_thesis['condition_variable']}
    
    Generate features in 4 categories:
    
    1. CONDITION FEATURES — identify WHEN/WHERE the thesis condition applies
       Create binary or categorical variables that split the data into
       "condition present" vs "condition absent" groups.
       Example: is_atypical_presentation, is_elderly (age > 65), is_night_shift
    
    2. OUTCOME FEATURES — measure the outcome WITHIN each condition group
       Compute the target-related metric separately for each condition.
       Example: triage_accuracy_typical, triage_accuracy_atypical
    
    3. DELTA FEATURES — quantify the DIFFERENCE between groups
       This is the feature that DIRECTLY measures the thesis.
       Example: accuracy_gap = accuracy_typical - accuracy_atypical
       A large, statistically significant gap PROVES the thesis.
    
    4. MODERATOR FEATURES — find what makes the effect STRONGER or WEAKER
       Cross the condition with other variables.
       Example: accuracy_gap × patient_age → does the gap widen for older patients?
    
    Available data: {state.enriched_data_path or state.features_train_path}
    External data integrated: {state.external_datasets}
    Existing features (DO NOT duplicate): {existing_feature_names}
    
    Use Polars. Handle nulls. Preserve row count.
    """
```

### Gate modification

The statistical gates still run. But the QUESTION changes:

**Traditional mode:** "Does this feature improve the model's CV score?" (Wilcoxon on CV with vs without)

**Hackathon mode:** "Does this feature show a statistically significant conditional effect?" 

```python
def _hackathon_gate(feature_values, condition_values, target_values, gate_config):
    """
    Instead of Wilcoxon on CV scores, test whether the feature
    reveals a significant conditional effect.
    """
    # Split data by condition
    group_a = feature_values[condition_values == 1]
    group_b = feature_values[condition_values == 0]
    
    # Mann-Whitney U test: do the groups differ?
    from scipy.stats import mannwhitneyu
    stat, p_value = mannwhitneyu(group_a, group_b, alternative="two-sided")
    
    # Effect size: Cohen's d
    effect_size = (group_a.mean() - group_b.mean()) / np.sqrt(
        (group_a.std()**2 + group_b.std()**2) / 2
    )
    
    # Pass if: significant AND meaningful effect size
    passed = p_value < gate_config["wilcoxon_p"] and abs(effect_size) > 0.2
    
    return passed, p_value, effect_size
```

Features that pass the hackathon gate get their effect sizes stored for the Narrative Engine:

```python
thesis_effect_sizes[feature_name] = {
    "p_value": p_value,
    "effect_size": effect_size,
    "group_a_mean": float(group_a.mean()),
    "group_b_mean": float(group_b.mean()),
    "n_group_a": len(group_a),
    "n_group_b": len(group_b),
}
```

### What stays the same

- The code generation via `run_in_sandbox()`
- The Self-Debugging Engine retry cascade
- The Code Ledger provenance capture
- The adaptive gate thresholds from Fix 1A
- The KEPT/PARKED/REJECTED three-state system

### Contract tests

1. `test_hackathon_prompt_mentions_thesis` — prompt contains active_thesis.statement
2. `test_hackathon_prompt_has_4_categories` — prompt asks for condition, outcome, delta, moderator features
3. `test_hackathon_gate_tests_conditional_effect` — gate uses Mann-Whitney, not CV-based Wilcoxon
4. `test_effect_sizes_stored` — features that pass gate have effect sizes recorded
5. `test_traditional_gates_still_work` — in traditional mode, regular Wilcoxon gates unchanged

---

## Component 6 — Hackathon Graph Builder (graph/hackathon_builder.py)

### The pipeline order in hackathon mode

```python
def build_hackathon_graph() -> StateGraph:
    """
    Alternative LangGraph graph for hackathon mode.
    Different agent order — thesis-driven instead of metric-driven.
    """
    graph = StateGraph(ProfessorState)
    
    # Phase 0: Parse the competition (SAME as traditional)
    graph.add_node("preflight_checks", run_preflight_checks)
    graph.add_node("competition_intel", competition_intel)
    
    # Phase 1: Understand the judging criteria (NEW)
    graph.add_node("rubric_parser", rubric_parser)
    
    # Phase 2: Understand the domain + data (SAME agents, different order)
    graph.add_node("data_engineer", data_engineer)
    graph.add_node("eda_agent", eda_agent)
    graph.add_node("domain_research", domain_research)
    
    # Phase 3: Generate and select thesis (NEW)
    graph.add_node("thesis_generator", thesis_generator)
    
    # Phase 4: Acquire external data (NEW)
    graph.add_node("external_data_scout", external_data_scout)
    
    # Phase 5: Test the thesis (MODIFIED Feature Factory)
    graph.add_node("hypothesis_feature_factory", hypothesis_feature_factory)
    
    # Phase 6: Build the model (SAME — technical quality still scores 30%)
    graph.add_node("ml_optimizer", ml_optimizer)
    graph.add_node("red_team_critic", red_team_critic)
    
    # Phase 7: Generate deliverables (NEW — narrative engine)
    graph.add_node("narrative_engine", narrative_engine)
    
    # Phase 8: Assemble notebook + final outputs (MODIFIED)
    graph.add_node("publisher", hackathon_publisher)
    
    # === EDGES ===
    graph.set_entry_point("preflight_checks")
    graph.add_edge("preflight_checks", "competition_intel")
    graph.add_edge("competition_intel", "rubric_parser")
    graph.add_edge("rubric_parser", "data_engineer")
    graph.add_edge("data_engineer", "eda_agent")
    graph.add_edge("eda_agent", "domain_research")
    graph.add_edge("domain_research", "thesis_generator")  # Thesis needs domain + EDA
    graph.add_edge("thesis_generator", "external_data_scout")  # Scout needs thesis
    graph.add_edge("external_data_scout", "hypothesis_feature_factory")
    graph.add_edge("hypothesis_feature_factory", "ml_optimizer")
    graph.add_edge("ml_optimizer", "red_team_critic")
    graph.add_edge("red_team_critic", "narrative_engine")  # Narrative needs results
    graph.add_edge("narrative_engine", "publisher")
    graph.add_edge("publisher", END)
    
    return graph
```

### What's DIFFERENT from traditional graph

| Phase | Traditional | Hackathon |
|---|---|---|
| After Competition Intel | Metric Verification Gate | Rubric Parser |
| After Domain Research | Validation Architect → Feature Factory | Thesis Generator → External Data Scout |
| Feature Factory purpose | Improve CV score | Test thesis hypothesis |
| After Critic | Self-Reflection → Ensemble → Post-Processing | Narrative Engine (plots + writeup) |
| Publisher output | submission.csv + notebook + writeup | notebook + writeup + cover image + project link |

### What's REMOVED in hackathon mode

Not every traditional agent makes sense:
- **Metric Verification Gate** — only if the hackathon has a leaderboard metric. If it's writeup-judged, skip.
- **Shift Detector** — still useful for understanding data but doesn't drive remediation
- **Problem Reframer** — the thesis IS the reframing
- **Pseudo-Labels** — typically irrelevant for hackathons
- **Post-Processing** — only if there's a metric to post-process
- **Ensemble** — only if technical quality weight > 25%
- **Submission Strategist** — no leaderboard-based submission strategy

### What's ADDED

- Rubric Parser (after Competition Intel)
- Thesis Generator (after Domain Research)
- External Data Scout (after Thesis Generator)
- Narrative Engine (after Critic)
- Modified Publisher (generates all required deliverables)

---

## The Hackathon Publisher (agents/hackathon_publisher.py)

### Deliverables generated

Based on `rubric.submission_requirements`, the publisher generates:

```python
def hackathon_publisher(state: ProfessorState) -> dict:
    """
    Generate ALL required hackathon deliverables.
    """
    deliverables = {}
    
    # 1. ALWAYS: Kaggle Notebook (clean, runs end-to-end)
    notebook_path = assemble_hackathon_notebook(state, session_dir)
    deliverables["notebook"] = notebook_path
    
    # 2. ALWAYS: Project Writeup (from Narrative Engine)
    writeup_path = state.hackathon_writeup_path
    deliverables["writeup"] = writeup_path
    
    # 3. IF REQUIRED: Cover Image (560x280px for Triagegeist)
    if "cover_image" in state.hackathon_rubric.get("submission_requirements", []):
        cover_path = _generate_cover_image(state, session_dir)
        deliverables["cover_image"] = cover_path
    
    # 4. IF REQUIRED: Project Link (GitHub repo)
    if "project_link" in state.hackathon_rubric.get("submission_requirements", []):
        # Professor can't create a GitHub repo, but can generate the README
        readme_path = _generate_github_readme(state, session_dir)
        deliverables["readme"] = readme_path
        emit_to_operator(
            "📎 Project link required. Generated README.md for GitHub repo. "
            "Create the repo manually and paste the URL.",
            level="CHECKPOINT"
        )
```

### Hackathon notebook format

Different from traditional `solution_notebook.py`:

```python
def assemble_hackathon_notebook(state, session_dir):
    """
    The hackathon notebook is BOTH code AND narrative.
    It interleaves markdown cells with code cells.
    """
    # Structure:
    # 1. Title + introduction (markdown)
    # 2. Data loading + description (code + markdown)
    # 3. EDA with inline discussion (code + markdown)
    # 4. External data integration (code + markdown)
    # 5. Thesis-testing features (code + markdown)
    # 6. Model training + evaluation (code + markdown)
    # 7. Results visualization (code + narrative plots)
    # 8. Discussion + limitations (markdown)
    
    # Generate as .ipynb format (JSON) for Kaggle compatibility
    # Each cell is either "code" or "markdown"
```

---

## Integration with HITL and Operator Playbook

### Operator's role in hackathon mode

The operator is MORE involved in hackathon mode than traditional mode:

| Checkpoint | Operator Action |
|---|---|
| Rubric parsed | Verify effort allocation is correct |
| Thesis proposals | SELECT the thesis (this is the creative direction) |
| External data candidates | Approve/reject datasets, suggest others |
| After hypothesis features | Review effect sizes, adjust thesis if needed |
| After model training | Review technical results |
| Narrative plots | Review visualizations, request changes |
| Writeup draft | Edit/improve the writeup (this IS the deliverable) |
| Final deliverables | Verify everything is complete, submit |

### Hackathon-specific HITL commands

```
/thesis select [1-5]     — select a thesis candidate
/thesis custom "..."     — provide custom thesis
/thesis refine           — regenerate thesis candidates with new context
/data search "..."       — search for specific external data
/data approve [id]       — approve an external dataset for integration
/narrative replot [n]    — regenerate specific narrative plot
/writeup edit "section"  — regenerate a specific writeup section
/writeup polish          — run another polish pass
/rubric show             — display the parsed rubric and effort allocation
```

---

## Build Plan (within v2 timeline)

```
Day 1:  tools/rubric_parser.py + tests (Commit 1)
Day 2:  agents/thesis_generator.py + tests (Commit 2)
Day 3:  agents/external_data_scout.py + tests (Commit 3)
Day 4:  tools/narrative_engine.py Part A (plots) + tests (Commit 4)
Day 5:  tools/narrative_engine.py Part B (writeup) + tests (Commit 5)
Day 6:  Hypothesis Feature Factory modifications + tests (Commit 6)
Day 7:  graph/hackathon_builder.py + hackathon_publisher + integration test (Commit 7)
```

7 commits, 7 days. Can run in parallel with Layer 4 traditional components since the infrastructure (sandbox, HITL, state, code ledger) is shared.

---

## What This Design Does NOT Do

- Does NOT require rewriting any traditional-mode agent
- Does NOT modify the Self-Debugging Engine, Cost Governor, or Data Integrity system
- Does NOT change the state schema enforcement or ownership model
- Does NOT replace the traditional pipeline — hackathon mode is an ALTERNATIVE graph
- Does NOT guarantee a win — the operator's thesis selection and domain judgment remain the competitive edge
- Does NOT automate the creative insight — it automates the EXECUTION of creative insight

The system generates thesis candidates. The operator provides the editorial judgment. Professor executes the analysis, validates the findings, generates the plots, and writes the narrative. The operator reviews and refines. The combination is a data science team compressed into one person plus an AI system.
