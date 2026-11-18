---
name: implement-feature
description: Implement a StackShop feature spec end to end across the three repos. Use when the user says "implement F07", "build the cart feature", "do Fxx", or asks for a feature that spans backend, frontend and infra. Splits the spec by repo, runs backend-engineer → frontend-engineer → devops-engineer in order, and verifies every acceptance criterion and the Definition of Done.
---

# Implement a feature spec (Fxx)

Run this in the **main session**. It coordinates the three subagents. It doesn't write app code itself.

## 1. Load and check the spec

1. Read `docs/features/Fxx-*.md` completely, then `docs/10-roadmap.md` to find its phase.
2. Check prerequisites:
   - **Dependencies:** the spec's summary table and `docs/features/README.md` graph list the features it depends on. If one isn't done, tell the user and stop, or ask whether to stub it.
   - **Phase:** if the feature belongs to a later phase than the current one, say so and confirm before continuing.
   - **Missing services/infra:** if the services or infrastructure it needs don't exist yet (check the repo folders), plan `new-microservice` / `add-infra-resource` first.
3. If anything in the spec is ambiguous or contradicts another doc, ask the user before starting. Don't guess.

## 2. Split the work by repo

Make a work breakdown table and show it with `update_plan`:

| Repo / agent | Work items (from the spec's "Tasks" + API/Events/Frontend sections) | Spec IDs covered |
|---|---|---|
| backend-engineer | domain, endpoints, events, consumers, gRPC, migrations, tests | BR-…, AC-… |
| frontend-engineer | pages, components, forms, BFF routes, tests | US-…, AC-… |
| devops-engineer | topics/queues/schemas, env vars, secrets, manifests, dashboards/alerts | – |

Map every `AC-n` to the repo that proves it (unit/integration test, Playwright, or manual demo).

## 3. Run the agents in order

Order: **backend → (client publish) → frontend → devops**. Infra resources the backend needs locally (a new topic/queue/schema) may go first. In that case run devops-engineer for `local/init` only, then backend.

For each agent, the prompt must contain:
- the spec path and the exact work items + spec IDs for that repo
- the handoff list from the previous agent (verbatim)
- the skill(s) it should use (e.g. `add-api-endpoint`, `add-integration-event`, `new-page`, `k8s-service-manifests`)
- "Report test output, anything not done, and a handoff list."

Wait for each agent's result before starting the next one. Read its report critically. If it says something is unverified or failing, resolve that before moving on.

## 4. Verify

1. Make an AC checklist: for each `AC-n`, the test name or evidence that proves it. Tick it only when you've seen the evidence (test output, a screenshot, a command result).
2. Go through the **Definition of Done** at the end of `docs/10-roadmap.md` item by item.
3. If the implementation differs from the spec, run the `sync-docs-adr` skill.
4. Optionally run the `local-env` skill and walk through the spec's demo.

## 5. Report

- Per repo: what changed, test results, commits (only if the user asked to commit)
- AC checklist with evidence. Clearly mark anything **not verified**
- Doc/ADR updates
- Follow-ups (remaining handoffs, deferred stretch goals)

Never commit or push in any repo unless the user asked. Each agent commits only inside its own repo, using Conventional Commits that reference the spec ID.
