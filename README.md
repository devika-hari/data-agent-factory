# data-agent-factory [WIP]

> **Work in progress.** This is a design and build plan. Nothing here has been tested yet.

A spec-first, multi-agent workflow that turns a data ticket into a tested dbt + Airflow pull request on Snowflake.

## The idea

A ticket goes in. A tested pull request comes out. People approve the two points where intent matters.

1. **Requirements agent** turns the ticket into a spec. Every field needs evidence. Anything missing becomes a question on the ticket.
2. **Tester agent** writes dbt tests from the spec, before any SQL exists (test-first).
3. **Human gate 1** approves the spec and tests together.
4. **Builder agent** writes the dbt models and the Airflow DAG change until the tests pass (max 3 tries).
5. **Tester agent** checks the real output data.
6. **CI** (GitHub Actions) runs on the pull request.
7. **Human gate 2** reviews and merges. No agent can touch production.

## Stack

Snowflake · dbt · Airflow · GitHub Actions · LangGraph (planned) · dbt MCP server · Airflow MCP server

## Status

| Phase | Status |
|---|---|
| Design and build spec | Done |
| 1. Skeleton | Not started |
| 2. Requirements agent | Not started |
| 3. Tester agent | Not started |
| 4. Builder agent | Not started |
| 5. Gates and CI | Not started |
| 6. Pilot | Not started |

## Files

- [`BUILD_SPEC.md`](BUILD_SPEC.md): the full design and build plan, written so a coding agent can build it phase by phase.
- [`docs/agentic-pipeline-deck.pdf`](docs/agentic-pipeline-deck.pdf): the slide deck with the flow, agent personas, architecture and open issues.

## Known limits

- A wrong spec gives a wrong model with green tests. Only a person who knows the business can catch it.
- Built for well-defined transformations, not exploratory analysis.
- Any use on real or sensitive data needs a compliance review first.

## Author

Devika Hari · [GitHub](https://github.com/devika-hari) · [LinkedIn](https://www.linkedin.com/in/devika-hari/)
