# Build spec: agentic data pipeline workflow

Hand this file to Claude Code. It describes what to build, in what order, and what must stay true.

## 1. Goal

Turn a task ticket into a tested pull request for a Snowflake + dbt + Airflow project.
Three agents do the work. Two human gates approve intent. Tests are written before the SQL (TDD).

## 2. Read this first (instructions to Claude Code)

- Do not invent project details. Section 12 lists open questions. Ask them before you write code that depends on the answers.
- Build one phase at a time (section 11). Stop after each phase and wait for confirmation.
- Keep it simple. Prefer existing MCP servers over custom tool code.
- Never give any agent credentials that can write to production.

## 3. Assumptions (change if the owner says so)

- Orchestration: LangGraph, Python 3.11+.
- LLM provider: configurable through one settings file. Default: Claude.
- Ticket source for v1: a manual note (a Markdown file). Jira and ClickUp come later behind the same interface.
- dbt and Airflow access: the dbt MCP server and the Airflow MCP server from `astronomer/agents`.
- Human gate 1 is a pause inside the graph. Human gate 2 is the normal GitHub pull request review.

## 4. Flow

```
Ticket
  -> Requirements agent      (names sources; code profiles them; spec, or questions + stop)
  -> Tester agent, pass 1    (contract tests from the spec and the profile, plus a stub model)
  -> HUMAN GATE 1            (approve spec + tests, with the data profile)
  -> Builder agent           (dbt models + DAG, loop until tests pass, max 3 tries)
  -> Tester agent, pass 2    (checks the output data: reconciliation, sample rows)
  -> CI checks               (GitHub Actions on the pull request)
  -> HUMAN GATE 2            (review PR with sample output and totals, then merge)
  -> Ticket updated with PR link and run log
```

Routing rules:

| Situation | Next step |
|---|---|
| Ticket has no `agent-ready` label | Stop. Do nothing. |
| Spec status is `blocked` | Post questions on the ticket. Stop. Resume when the ticket changes. |
| Gate 1 rejected | Back to Requirements agent with the reviewer's comments. |
| Tests fail and attempts < 3 | Back to Builder agent with the failure log. |
| Tests fail and attempts = 3 | Escalate to a human. Stop. |
| Output check (Tester pass 2) fails | Back to Builder agent with the report. Counts as one attempt. |
| CI fails | Back to Builder agent. Counts as one attempt. |

## 5. Shared state

One typed object passed through the graph. Every agent reads it. Each agent writes only its own part.

```python
class RunState(TypedDict):
    ticket: Ticket                 # id, title, body, labels, source
    spec: Spec | None              # written by Requirements agent
    open_questions: list[str]      # written by Requirements agent
    spec_status: Literal["ready", "blocked"] | None   # set by code, not by the model
    tests: list[TestFile]          # written by Tester agent
    tests_checksum: str | None     # set by code after gate 1
    gate1: Literal["approved", "rejected"] | None
    gate1_comments: str | None
    build_attempts: int            # 0..3
    changed_files: list[str]       # written by Builder agent
    test_results: TestResults | None
    profile: dict[str, TableProfile]   # written by code, keyed by source table
    output_report: OutputReport | None # written by Tester pass 2
    pr_url: str | None
    run_log: list[LogEntry]        # append-only, every step
```

## 5a. Profiling

Profiling is a fixed step run by code. No agent writes profiling SQL.

Who does what:

1. The Requirements agent returns the list of source tables, and may pick extra checks from the menu below.
2. Code validates each table and column name against Snowflake metadata.
3. Code runs fixed, parameterised queries and stores the results in `profile`.
4. The Requirements agent and the Tester read `profile`. Gate 1 shows it to the reviewer.

Default profile for each source table:

- Row count.
- For each column: null rate, distinct count, min and max.
- Top values for low-cardinality columns.
- Freshness: the latest value of the main timestamp column.

Menu of extra checks (the agent picks, code runs):

- `key_uniqueness(table, columns)`
- `join_match_rate(left, right, keys)`
- `value_list(table, column)`
- `date_range(table, column)`

Rules:

- Aggregates only. Profiling never returns raw rows.
- No top values or value lists for columns flagged as sensitive.
- Set a timeout and a row sample limit so cost stays low.
- Each surprise becomes a question on the ticket. Example: the expected key has duplicates.

## 6. The spec

The spec is the contract. Validate it with Pydantic.

Every field is a `Sourced` value:

```python
class Sourced(BaseModel):
    value: str | list[str] | None
    tag: Literal["from_ticket", "verified", "from_repo", "missing"]
    evidence: str | None   # ticket quote, query / metadata result, or repo file path
```

Evidence is checked by code. The model's claim is not trusted:

- `from_ticket`: the quote must appear word for word in the ticket body (ignore whitespace differences). If not, reset the tag to `missing`.
- `verified`: code re-runs the metadata or profile check itself and stores that result as the evidence.
- `from_repo`: allowed for optional fields only. The evidence is a file path that exists and contains the value. Use it for defaults that follow repo conventions, such as layer or materialisation.

| Field | Required | Meaning |
|---|---|---|
| `business_goal` | yes | Why it is needed and who asked |
| `source_tables` | yes | Must be verified to exist in Snowflake |
| `grain` | yes | What one row means |
| `columns` | yes | Name, definition, business rule, for each column |
| `target_model` | yes | Model name and layer |
| `schedule` | no | Refresh time and freshness need |
| `acceptance_criteria` | yes | How we know it is right. Each one must be testable. |
| `sensitive_data` | yes | Flags for personal or regulated fields |

`spec_status` is computed in code:

- `ready` if every required field has a tag other than `missing` and has evidence.
- `blocked` otherwise.

## 7. Agents

### 7.1 Requirements agent

- Persona: a careful business analyst. Asks before assuming.
- Reads: the ticket, the dbt project (models, sources, lineage), Snowflake metadata.
- Writes: `spec`, `open_questions`. Its only external write is a comment on the ticket.
- Steps:
  1. Fill the spec template from the ticket text.
  2. Check that each source table and column exists.
  3. Name the source tables to profile (section 5a) and read the results.
  4. Check whether a similar model already exists.
  5. Tag each field and attach evidence.
  6. Write one clear question for each missing required field and each profile surprise.
- Rules:
  - No proof means `missing`. Never a guess.
  - Never define a business term itself. Ask, and offer options if helpful.
  - Never set `spec_status`. Code does that.
  - Keep questions few. Ask only about required fields and real profile surprises. Take optional fields from repo conventions where possible.
  - Post all questions in one comment, not one at a time.

### 7.2 Tester agent

- Persona: a sceptical QA engineer. Trusts the spec and the data, not the code.
- Works twice.

Pass 1, before the build (contract tests):

- Reads: the spec, the data profile, existing tests in the dbt project.
- Writes: dbt test files, and a stub model so the tests parse.
- Produces:
  - Schema tests: `unique`, `not_null`, `accepted_values`, `relationships`.
  - dbt unit tests with small mock inputs and expected outputs.
  - A stub model: the target model file with the right column names and no logic. The Builder fills it in.
- Use the profile to keep tests realistic. Example: only add `not_null` where the rule requires it or the source supports it.

Pass 2, after the build (output check):

- Reads: the built table in dev, the source tables, the spec. Not the model SQL.
- Writes: `output_report`.
- Produces:
  - Reconciliation: row counts and totals against source.
  - Sample output rows, with sensitive columns masked.
  - A pass or fail for each acceptance criterion.

Rules:

- Pass 1 runs before the Builder. No model SQL exists yet.
- The Tester never reads the Builder's SQL, in either pass.
- Every acceptance criterion maps to at least one test or output check. List any it could not cover.

### 7.3 Builder agent

- Persona: a senior analytics engineer. Follows the repo's conventions.
- Reads: the approved spec, the approved tests, repo conventions, test and run logs.
- Writes: dbt models and docs, the Airflow DAG change, the pull request.
- Steps:
  1. Read the repo conventions (naming, layers, materialisation).
  2. Write the models.
  3. Run `dbt build` in dev.
  4. On failure: read the log, fix, retry. Max 3 attempts.
  5. Add the model to the DAG (or create a DAG if none fits). Trigger a dev run.
  6. Open a pull request linked to the ticket.
- Rules:
  - Builds only what the spec says.
  - Must not change test files. Code compares `tests_checksum` before and after each attempt and fails the run if it differs.
  - Works in dev only. Never merges. Never pushes to prod.
  - Must not special-case test inputs. No logic that exists only to make a mock case pass.
  - Green unit tests do not finish the build. The Tester's output check on real dev data must also pass.

## 8. Human gates

- Gate 2 must show data, not only green ticks: put the `output_report` (totals and masked sample rows) in the pull request description.
- Gate 1 (in the graph): show the spec, the tests and the data profile together. Accept `approve` or `reject` with comments. Use a LangGraph interrupt so the run can pause and resume. A CLI prompt is enough for v1.
- Gate 2 (outside the graph): a person reviews and merges the pull request on GitHub.

## 9. Tools and access

| Tool | Used by | Access |
|---|---|---|
| Ticket source (manual note, later Jira / ClickUp) | Requirements | Read ticket, post comment |
| dbt MCP server | All three | Requirements and Tester: read. Builder: read and run in dev. |
| Snowflake | All three | Requirements and Tester: metadata read-only. Builder: dev schema only. |
| Airflow MCP (`astronomer/agents`) | Builder | Dev environment only |
| GitHub | Builder | Feature branch and pull request. No merge rights. |
| GitHub Actions | none (not an agent) | Runs CI on the pull request |

Put the ticket source behind one interface (`get_ticket`, `post_comment`) so Jira and ClickUp can be added without touching the agents.

## 10. Guardrails (must always hold)

1. Every spec value has evidence, or it is tagged `missing`.
2. A missing required field stops the run and posts a question.
3. Code decides `ready` or `blocked`. The model does not.
4. The Tester never sees the Builder's SQL. The Builder never edits tests.
5. Three build attempts at most, then escalate.
6. No agent holds production credentials.
7. Every step appends to `run_log` (who, what, when, result).
8. Profiling SQL is fixed code. It returns aggregates only.
9. Sample rows shown to people have sensitive columns masked.
10. Code verifies every piece of evidence. An unverified quote resets the field to `missing`.
11. A build is done only when the tests and the output check on real data both pass.

## 11. Build phases

Stop after each phase and wait for confirmation.

| Phase | Build | Done when |
|---|---|---|
| 1. Skeleton | Repo layout, `RunState`, the graph with stub agents, routing rules, run log | A fake ticket runs end to end with stubs, and each routing rule has a test |
| 2. Requirements agent | Spec schema, profiling step (5a), evidence tags, `spec_status` check, question posting | A complete ticket gives `ready`. An incomplete ticket gives `blocked` with correct questions. No field has a value without evidence. A made-up quote is rejected by code. Questions arrive in one comment. Profiling returns aggregates only. |
| 3. Tester agent | Pass 1: tests and stub model from the spec and profile. Pass 2: output report. | Every acceptance criterion maps to a test or output check. The project parses with dbt. Pass 2 fails on a deliberately wrong model. |
| 4. Builder agent | Model writing, `dbt build` loop, retry cap, checksum check, DAG change | A simple ticket goes green in dev within 3 tries. A forced failure escalates. A model that passes the mocks but is wrong on real data fails the output check. |
| 5. Gates and CI | Gate 1 interrupt, pull request creation, GitHub Actions workflow | A run pauses at gate 1, resumes on approve, and opens a PR that CI checks |
| 6. Pilot | One real ticket on the banking project | The owner reviews the PR and the run log |

Suggested layout:

```
agentic-pipeline/
  README.md
  pyproject.toml
  config/settings.yaml        # model provider, paths, limits
  src/
    graph.py                  # nodes, edges, routing
    state.py                  # RunState, Spec, Sourced
    agents/
      requirements.py
      tester.py
      builder.py
    prompts/                  # one persona prompt per agent
    tools/
      tickets.py              # manual note now, Jira / ClickUp later
      dbt.py
      snowflake.py
      airflow.py
      github.py
    gates.py                  # gate 1 interrupt
    checks.py                 # spec_status, tests_checksum, retry cap
  tests/
  .github/workflows/ci.yml
```

## 12. Open questions (ask the owner before building)

1. Where is the dbt project? Is it dbt Core or the dbt platform?
2. How does Airflow run today: local, Docker, or Astro? Is Cosmos in use?
3. Which Snowflake role and schema should agents use for dev?
4. Which LLM provider and model should be the default?
5. Which ticket source comes first after the manual note: Jira or ClickUp?
6. Should the agent code live in the pipeline repo or in a separate repo?
7. What CI already exists in GitHub Actions for the dbt project?
8. Is the first pilot a new model, or a change to an existing one? (Changes need the impact check below.)
9. What is the cost budget per run (LLM tokens and warehouse credits)?

## 12a. Gaps found in review (design before phase 4)

- **Changing an existing model.** The spec assumes a new model. For a change, the Requirements agent must list downstream models from dbt lineage, and the Tester must run their tests too.
- **Incremental models and backfills.** The spec needs a `materialisation` field and, for incremental models, the unique key and a backfill note.
- **The ticket changes mid-run.** Store a hash of the ticket body at the start. If it changes, stop and restart from the Requirements agent.
- **Success metrics.** Log for each run: time to PR, number of questions, build attempts, gate 1 rejections, and whether the PR was merged without changes.
- **Cost guard.** Stop a run that passes its token or credit budget.

## 13. Out of scope for v1

- Exploratory analysis with no clear rules. This flow is for well-defined transformations.
- Production deploys by an agent.
- Skipping human gates for low-risk tickets.
- More than one ticket in parallel.
- BI or dashboard changes.

## 14. Later (not v1)

- Fast path for small tickets, such as adding one column: one agent and one gate.
- A different model for the Tester than for the Builder, to reduce shared blind spots.
- A short gate summary that highlights only the risky items for the reviewer.

## 15. Known open issues

Do not try to solve these in code. Keep them visible in the README.

1. A wrong spec gives a wrong model with green tests. Only a person who knows the business can catch it.
2. Reviewers can approve without reading. The gates are only as good as the review.
3. The Tester and the Builder can misread the spec the same way.
4. Sending bank data to a model provider needs compliance sign-off before any pilot on real data.
5. Exploratory work does not fit this flow.
6. Whoever merges the pull request owns the result.
