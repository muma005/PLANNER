# Phase-by-Phase Implementation Plan

## Ariadne – AI‑Enabled Progress Tracker + Notion Agent + Smart Work Planner

This plan breaks down the entire 25‑week build into **manageable weekly tasks**. Each phase ends with clear **success criteria** and **tests** you can run manually. Follow the order strictly. Do not skip ahead.

---

## Phase 0: Foundation & Identity Lock  
**Duration: 0.5 week (3–4 days)**

### Goal
Establish the project skeleton, database schema, and the rule that one project = one GitHub repo + one Notion database.

### Tasks
1. **Set up the development environment**
   - Create a Python virtual environment.
   - Install base dependencies: `psycopg2`, `redis`, `python-dotenv`, `click` (CLI skeleton).
   - Initialize Git repository.

2. **Configure PostgreSQL and Redis**
   - Run PostgreSQL locally (Docker or native).
   - Create database `ariadne`.
   - Run Redis locally.

3. **Create the core tables** (run raw SQL)
   - `projects` with unique constraints on `github_repo_url` and `notion_database_id`.
   - `user_preferences` (basic: work hours, timezone).
   - `sync_logs`.

4. **Write a CLI skeleton**
   - Command: `ariadne project add --name --github --notion-db`
   - Command: `ariadne project list`

5. **Implement project addition logic**
   - Validate that GitHub repo exists (quick API check).
   - Validate that Notion database exists (API check).
   - Insert into `projects`.

### Success Criteria (Test manually)
- `ariadne project add` works and creates a record.
- Trying to add the same GitHub repo again fails (unique constraint).
- Trying to add the same Notion DB again fails.
- `ariadne project list` shows the added project.

---

## Phase 1: Data Ingestion – GitHub Sync  
**Duration: 2 weeks**

### Goal
Fetch commits from a GitHub repo (one project) and store them in PostgreSQL, with rate limiting and caching.

### Tasks

**Week 1 – Basic GitHub connector**
1. Install `PyGithub` and `tenacity` (for retries).
2. Implement GitHub authentication (Personal Access Token from `.env`).
3. Write a function `fetch_commits(repo_full_name, since=None)` that returns a list of commit dicts.
4. Store commits in `commits` table (project_id, sha, author, date, message, files_changed JSON).
5. Add CLI command: `ariadne sync --project <key>`.
6. Implement simple rate limit detection (print warning if near limit).

**Week 2 – Caching & incremental sync**
1. Add Redis caching for GitHub API responses (TTL 1 hour).
2. Store last sync timestamp per project in `projects.last_synced_at`.
3. Modify `fetch_commits` to use `since=last_synced_at` (only new commits).
4. Add `sync_logs` table and log each sync attempt.
5. Implement pagination (auto‑fetch all pages).

### Success Criteria
- `ariadne sync --project X` fetches all commits from the repo.
- Running it again fetches only new commits.
- If rate limit is hit, script waits and retries (exponential backoff).
- `SELECT * FROM commits WHERE project_id = X` returns correct data.

---

## Phase 1.5: Enhanced GitHub – Branch/Path Filters & Commit Parsing  
**Duration: 1 week**

### Goal
Allow projects to be scoped to specific branches or file paths, and parse commit messages for task IDs.

### Tasks
1. Create `project_scopes` table.
2. Extend `ariadne project add` to accept optional `--branch` and `--path` flags.
3. Modify GitHub connector: before storing a commit, check if its branch or changed files match the scope. Ignore if not.
4. Add commit message parser:
   - Regex for `[TASK-123]`, `#123`, `fixes #456`, `closes #789`.
   - Store extracted ID in `commits.parsed_task_id`.
5. Add `needs_classification` flag for commits with no parsed ID (orphan detection).
6. Write CLI command: `ariadne orphans` – lists commits without task links.

### Success Criteria
- Commit from allowed branch → stored; from other branch → ignored.
- Commit containing `[TASK-42]` → `parsed_task_id` = `TASK-42`.
- `ariadne orphans` shows unlinked commits.

---

## Phase 2: Data Ingestion – Notion Sync & Enrichment  
**Duration: 2 weeks**

### Goal
Fetch tasks from Notion database, link to commits, and build basic progress metrics.

### Tasks

**Week 1 – Notion connector**
1. Install `notion-client`.
2. Implement authentication (integration token).
3. Write function `fetch_tasks(database_id)` that returns pages with properties.
4. Store tasks in `notion_tasks` table (project_id, notion_page_id, title, status, priority, due_date, etc.).
5. Add Notion sync to `ariadne sync` command (syncs both GitHub and Notion for a project).

**Week 2 – Linking commits to tasks**
1. Create `commit_task_links` table.
2. For each commit with `parsed_task_id`, try to find a matching `notion_task` (by `notion_page_id` or custom ID). Create link with confidence=1.
3. For commits without parsed ID, use embedding similarity (generate embeddings using sentence‑transformers) to suggest links. Store with confidence<1.
4. Add CLI command: `ariadne link-suggestions` – shows low‑confidence links for manual approval.

### Success Criteria
- `ariadne sync` fetches Notion tasks correctly.
- Commit with `[TASK-123]` is linked to the correct Notion task.
- `commit_task_links` table contains entries.

---

## Phase 2.5: Enhanced Enrichment – Dependencies, Sub‑tasks, Size Tags  
**Duration: 1 week**

### Goal
Extract task dependencies, sub‑tasks, and automatic size classification.

### Tasks
1. Create `task_dependencies` table (supports cross‑project).
2. Extend Notion parser to read relation properties (e.g., “Blocks” column) and populate dependencies.
3. Create `sub_tasks` table.
4. Parse Notion checklists or child pages as sub‑tasks; store in `sub_tasks`.
5. Add `size_tag` to `notion_tasks` (quick/medium/large) based on estimated minutes or keywords.
6. Add CLI command: `ariadne task show <id>` – displays task, sub‑tasks, dependencies.

### Success Criteria
- Notion relation property “Blocked by” creates a dependency.
- A checklist inside a Notion page appears as sub‑tasks.
- Task with estimated 10 minutes is tagged `quick`.

---

## Phase 3: Progress Calculator & Report Generator (Multi‑Agent)  
**Duration: 2 weeks**

### Goal
Calculate overall progress % and generate resumption reports using a multi‑agent LLM system.

### Tasks

**Week 1 – Progress metrics**
1. Create `project_snapshots` table (daily snapshot of tasks done, commits, completion %).
2. Write function `calculate_progress(project_id)`:
   - Simple % = completed tasks / total tasks.
   - Weighted % = sum(priority weight of completed) / sum(priority weight of all).
3. Store snapshot daily via scheduled job (Celery Beat).
4. Add CLI command: `ariadne progress <project>` – shows %, breakdown.

**Week 2 – Report agents (Redis queue)**
1. Set up Redis as message broker.
2. Create four worker types:
   - **Dispatcher** – receives user request, fetches project config.
   - **Context Retriever** – queries DB for commits, tasks, snapshots.
   - **LLM Analyzer** – calls OpenRouter with structured prompt (include citation rule).
   - **Validator** – checks that cited commit SHAs exist, flags hallucinations.
3. Wire them together via Redis queues.
4. Add CLI command: `ariadne report <project>` – triggers the workflow and prints report.

### Success Criteria
- `ariadne progress` shows correct % (matches manual count).
- `ariadne report` produces a 6‑section report with commit citations.
- Validator flags if LLM invents a commit SHA.

---

## Phase 4: Output & Escalation Engine  
**Duration: 1.5 weeks**

### Goal
Deliver reports in multiple formats and alert when projects become stale.

### Tasks

**Week 1 – Output formats**
1. Extend `ariadne report` to accept `--format markdown` (saves to file) and `--format json`.
2. Create simple web dashboard (Streamlit) showing project list, last report, progress chart.
3. Add `ariadne dashboard` command to launch locally.

**Week 2 (0.5) – Escalation engine**
1. Add `abandonment_thresholds` to `user_preferences` (warning_days, critical_days, archive_days).
2. Create background job (daily) that checks `last_commit_date` for each project.
3. If >warning_days, send Notion comment (via API) to the project page.
4. If >critical_days, also send Slack/email (configurable).
5. If >archive_days, move project to `archived_projects` table (exclude from plans).

### Success Criteria
- `ariadne report --format markdown` creates a file.
- Web dashboard shows at least one project.
- After 7 days of no commits, you receive a warning in Notion.

---

## Phase 5: Notion AI Agent (Basic)  
**Duration: 2 weeks**

### Goal
Allow `@ai` commands from inside Notion pages.

### Tasks

**Week 1 – Trigger detection (polling)**
1. Set up a script that runs every minute (cron or Celery Beat).
2. Query Notion for pages in tracked databases that contain the string `@ai`.
3. Extract the command after `@ai`.
4. Send command to AI Brain (simple wrapper for now).

**Week 2 – Response writing**
1. AI Brain processes command (using OpenRouter) and returns answer.
2. Script appends answer as a new callout block below the command.
3. Add basic command types: `summarize this page`, `what is the status of X?`, `update status to Done`.
4. Handle errors (e.g., “Could not find project”).

### Success Criteria
- Type `@ai summarize this page` in Notion → within 1 minute, a summary appears.
- `@ai what is the status of Project A?` returns correct progress.

---

## Phase 5.5: Enhanced Notion AI Agent – Cost Control & Caching  
**Duration: 1 week**

### Goal
Reduce LLM costs by routing to cheaper models and caching responses.

### Tasks
1. Install `ollama` and pull `llama3` (local).
2. Implement tiered router:
   - Status queries → local Llama 3 (free).
   - Summaries → Gemini 2.0 Flash (cheap).
   - Planning → GPT-4o mini (medium).
   - Debugging → Claude (expensive).
3. Add Redis cache for LLM responses (TTL 24h, keyed by prompt + context hash).
4. Create `budget_tracking` table; log each call.
5. Add CLI command: `ariadne budget` – shows spend today/week/month.

### Success Criteria
- `@ai what is the status?` uses local LLM (zero cost).
- Repeated identical question returns cached answer.
- Budget log shows correct costs.

---

## Phase 6: Smart Work Planner – Core  
**Duration: 3 weeks**

### Goal
Generate daily/weekly plans from project constraints.

### Tasks

**Week 1 – User profile & constraints**
1. Extend `user_preferences` with `max_parallel_projects`, `constant_project_id`, `deep_work_minutes`.
2. Add `project_constraints` table (deadline, estimated_remaining_hours, priority, is_constant).
3. Create CLI: `ariadne project estimate --project X --hours 40 --deadline YYYY-MM-DD`.

**Week 2 – Scheduling algorithm**
1. Implement a simple time‑weighted round robin scheduler:
   - Constant project gets 40% of daily hours.
   - Remaining hours divided among active projects based on urgency (days to deadline).
2. Output a daily plan (list of tasks per project with hours).

**Week 3 – Task breaker & daily generator**
1. Fetch incomplete tasks from Notion for each project.
2. Split tasks into “work units” of 2–4 hours (combine small tasks, split large ones).
3. Generate timeline: deep work in mornings, shallow afternoons, add lunch and breaks.

### Success Criteria
- `ariadne plan today` shows a schedule with projects and hours.
- Constant project appears every day.
- Total planned hours ≤ user’s `work_end – work_start` minus breaks.

---

## Phase 6.5: Enhanced Planner – Dependencies, Calendar, Switch Costs  
**Duration: 2 weeks**

### Goal
Make planner aware of meetings, holidays, dependencies, and context switching.

### Tasks

**Week 1 – Calendar & holidays**
1. Integrate Google Calendar API (OAuth) – read‑only.
2. Fetch events for next 7 days, block those time slots in planner.
3. Add `user_time_off` table; planner skips those dates.
4. If meetings fill >50% of day, reduce planned work and alert.

**Week 2 – Dependencies & switch costs**
1. Modify scheduler to respect `task_dependencies` (topological order).
2. Add cross‑project dependency support (if Task B depends on Project A’s task, schedule B after A’s estimated completion).
3. Implement `switch_costs` table; add penalty minutes between different projects.
4. Add quick task batching (collect `#quick` tasks into 30‑min slots).

### Success Criteria
- Meeting at 2 PM → no work scheduled during that hour.
- Holiday marked → no tasks assigned that day.
- Task that depends on another is scheduled after it.
- Switching from A to B adds a 10‑min buffer.

---

## Phase 7: Verification & Auto‑Reassignment  
**Duration: 1.5 weeks**

### Goal
Compare planned vs. actual work and reschedule unfinished tasks.

### Tasks

**Week 1 – Verification worker**
1. Create `planned_task_verification` table.
2. At end of day, for each task in `daily_plans`:
   - Query GitHub for commits in that project since the task’s start time.
   - Query Notion for task status.
   - Determine: Complete / Partial / No progress.
3. Store result with `was_completed`, `partial_progress_percentage`.

**Week 2 (0.5) – Auto‑reassignment**
1. For partial progress, estimate remaining hours (use pattern matching or LLM).
2. For missed/partial tasks, remove completed portion and add remaining hours back to project backlog.
3. Re‑run scheduler for remaining days (or just tomorrow) to insert the leftover work.
4. Send user a proposal (CLI or Notion) to accept/reject.

### Success Criteria
- You plan a task, do 1 commit but don’t finish → end‑of‑day verification detects partial (e.g., 40% done).
- Remaining 60% is rescheduled to the next day.
- You receive a message explaining the reassignment.

---

## Phase 7.5: Enhanced Verification – Untracked Work & Prompts  
**Duration: 1 week**

### Goal
Detect work that was done but not logged or committed.

### Tasks
1. Monitor system activity (optional: integrate with keyboard/mouse tracker or IDE plugin).
2. If >2 hours of activity with no commits or task updates, trigger end‑of‑day prompt.
3. Prompt via CLI or Notion: “You had 3 hours of activity. Which project(s)?”
4. Store user response in `time_logs` (source = `prompted`).
5. Adjust future plans based on discovered work (reduce remaining hours).

### Success Criteria
- You work on Project B for 2 hours without committing → at end of day, you are asked to assign that time.
- After assigning, Project B’s remaining hours decrease.

---

## Phase 8: Learning & Personalization  
**Duration: 1 week**

### Goal
Improve estimates, detect patterns, and personalize schedules.

### Tasks
1. Create `learned_patterns` table.
2. Implement duration learning: after each completed task, compare estimated vs actual minutes; update multiplier for that task type.
3. Implement focus peak learning: track hour of day when commits happen most; adjust schedule to put deep work there.
4. Implement empty promise detector: compare week 1 actual velocity vs. initial estimate; if >2×, flag and apply multiplier for future estimates.
5. Store all patterns in `learned_patterns`.

### Success Criteria
- After 5 “unit test” tasks, the estimate becomes more accurate (error <20%).
- Planner schedules deep work at your historically productive hours.

---

## Phase 9: AI Brain – Conversation & Memory  
**Duration: 3 weeks**

### Goal
Unify all functionality into a single conversational AI.

### Tasks

**Week 1 – Conversation manager**
1. Create `conversations` table with message history and vector embeddings.
2. Implement CLI entry: `ariadne brain "query"`.
3. Maintain session state per user.

**Week 2 – Tool registry & ReAct loop**
1. Wrap all existing functions as tools (40+ tools: `get_project_report`, `update_task`, `reschedule`, etc.).
2. Implement ReAct loop: LLM decides which tool to call, calls it, observes result, iterates.
3. Add deterministic ID guard: tool calls must include project_id; if ambiguous, LLM asks user.

**Week 3 – Memory & proactive intelligence**
1. Add semantic memory: retrieve past conversations relevant to current query (vector similarity).
2. Add morning briefing scheduler (8 AM): fetches plan, stale projects, blockers, sends to user channel.
3. Add celebration messages (e.g., when a project is finished early).

### Success Criteria
- `ariadne brain "why was task X reassigned?"` retrieves verification logs and explains.
- Morning briefing arrives automatically.
- Conversation memory works: follow‑up “and what about Project B?” understands context.

---

## Phase 9.5: Brain Hardening – Cost, Cache & Single Interface  
**Duration: 1 week**

### Goal
Optimize LLM costs, enforce single interface, and add gamification (optional).

### Tasks
1. Enforce that all user interaction goes through `ariadne brain` (deprecate direct commands).
2. Add budget alert: when monthly spend reaches 80% of limit, send warning.
3. Implement gamification: points for plan adherence, streaks, badges; store in `user_achievements`.
4. Add `ariadne brain --channel slack` for future Slack integration (placeholder).

### Success Criteria
- You can no longer run `ariadne report` directly; you must use `ariadne brain "generate report"`.
- Budget alert triggers at 80% spend.
- Gamification badges appear after 7 consecutive days of plan adherence.

---

## Phase 10: Cross‑Project Orchestration & Scaling  
**Duration: 2 weeks**

### Goal
Handle dependencies and resource leveling across many projects.

### Tasks

**Week 1 – Global dependency resolution**
1. Extend scheduler to consider all projects simultaneously (not one‑by‑one).
2. Build a global dependency graph across all projects.
3. Implement critical path method to identify which tasks drive the timeline.

**Week 2 – Resource leveling & global queue**
1. Since you are the only resource, ensure total planned hours per day ≤ user’s available hours.
2. Create a global priority queue (mix tasks from all projects based on urgency, not per‑project).
3. Add CLI: `ariadne brain "show my global backlog"` – lists next 10 tasks regardless of project.
4. Performance test with 50 projects (simulated data) – ensure sync and planning finish within 5 minutes.

### Success Criteria
- Project B task that depends on Project A task is scheduled after A’s estimated completion.
- Daily plan never exceeds available hours.
- `ariadne brain "global backlog"` returns a meaningful list.

---

## Final Integration & Testing (1 week after Phase 10)

### Tasks
1. Run the entire system for 2 weeks on your real projects (start with 3 projects, then add more).
2. Verify all success criteria from earlier phases.
3. Document any bugs and fix them.
4. Write user documentation (README, CLI help, Notion setup guide).
5. Optional: Deploy via Docker Compose for easy startup.

### Success
- You use the system daily without frustration.
- You no longer lose context when switching projects.
- Your productivity feels 2–3× higher because you never wonder “what next?”

---

## Summary of Phases (Table)

| Phase | Weeks | Focus |
|-------|-------|-------|
| 0 | 0.5 | Foundation & identity lock |
| 1 | 2 | GitHub sync |
| 1.5 | 1 | Branch filters & commit parsing |
| 2 | 2 | Notion sync & linking |
| 2.5 | 1 | Dependencies, sub‑tasks, size tags |
| 3 | 2 | Progress & multi‑agent reports |
| 4 | 1.5 | Output formats & escalation |
| 5 | 2 | Notion AI Agent (basic) |
| 5.5 | 1 | Cost control & caching |
| 6 | 3 | Planner core |
| 6.5 | 2 | Enhanced planner (calendar, dependencies, switch costs) |
| 7 | 1.5 | Verification & auto‑reassignment |
| 7.5 | 1 | Untracked work prompts |
| 8 | 1 | Learning & personalization |
| 9 | 3 | AI Brain (conversation, memory, tools) |
| 9.5 | 1 | Brain hardening, gamification |
| 10 | 2 | Cross‑project orchestration |
| Final | 1 | Integration & testing |

**Total: 25–26 weeks**

---

## How to Track Progress

Create a checklist in your project management tool (Notion, of course) with each phase as a heading and each task as a checkbox. After completing a task, test it immediately. Only check it off when the test passes.

**Example for Phase 0:**
- [ ] Python virtual environment created.
- [ ] PostgreSQL and Redis running.
- [ ] `projects` table created with unique constraints.
- [ ] `ariadne project add` works.
- [ ] Duplicate GitHub repo rejected.

Then move to Phase 1.

---

## Final Advice

**Never skip testing at the end of a phase.** If you do, you will accumulate technical debt that will take longer to fix than writing the tests would have.

**Celebrate each green checkmark.** This is a long journey. Each working piece is a victory.

Now start with Phase 0. Good luck. 🧵
