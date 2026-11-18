---
name: sync-docs-adr
description: Keep StackShop's design docs true to the code. Use when an implementation deviated from a feature spec or architecture doc, when an architectural decision changes (new library, different store/broker, changed flow), or when the user asks to "update the docs" or "write an ADR". Updates every affected doc consistently and records decisions as ADRs.
---

# Sync docs and write ADRs

The docs in `docs/` are the source of truth. Code and docs must never silently disagree.

## 1. Decide what kind of change this is

| Change | Action |
|---|---|
| Detail inside one feature (an endpoint shape, an extra field, a renamed error code, a rule clarified) | Update the feature spec only (+ the API/event tables it references) |
| Something cross-cutting (event name/topic, store ownership, an auth flow, CI flow, a repo layout, a library choice, a new or removed service) | Update every affected doc **and** write an ADR |
| Reversing an existing ADR | New ADR with `Supersedes ADR-NNN`, and mark the old one `Superseded by ADR-MMM` |

## 2. Find every place to update

Grep the docs for the changed term (event type, topic, endpoint path, table/collection, service name, error code):

```bash
grep -rn "<term>" docs/ CLAUDE.md .claude/agents .claude/skills
```

Where things are usually documented:
- Events/topics/queues → `04-messaging-and-events.md` (catalogue + topic tables) + the feature specs that publish or consume them
- Schemas, collections, Redis keys, ES mapping → `03-data-architecture.md`
- Endpoints, gateway routes, error codes, permissions → `05-api-design.md` + the feature spec
- Service responsibilities/stores → `02-architecture.md` §3 service catalogue
- Pipelines, compose, K8s → `07-devops-cicd.md`. Metrics/alerts → `08-observability.md`
- Rules for agents → `CLAUDE.md`, `.claude/agents/*.md`, `.claude/skills/*`

Update all of them in the same change so they agree.

## 3. Write the ADR (if needed)

1. Next number = the highest `docs/adr/NNN-*.md` + 1, zero-padded to 3 digits.
2. Copy `docs/adr/000-template.md` to `docs/adr/NNN-<kebab-title>.md`, and fill in Status (`Accepted`), today's date, related features, context, options considered (at least 2), decision and consequences.
3. Add a row to the ADR summary table in `docs/02-architecture.md` §10 linking the file.
4. Link it from `docs/README.md` if it's a major decision.

## 4. Check

- Re-run the grep. No stale mentions should remain, except intentional ones in superseded ADRs.
- Mermaid diagrams still match the text.
- Report the list of files changed and a one-line summary per file.

Docs live in the workspace root git repo (`stack-app`). Commit there only if the user asks, with `docs(<area>): …`.
