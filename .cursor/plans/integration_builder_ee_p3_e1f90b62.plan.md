---
name: Integration Builder EE
overview: P3 — Stored agents and Agent Builder pin Integration tools and trigger bindings. Coordinate with Mastra EE owners. Parent unified_integration_layer_7f3a91c2.
todos:
  - id: schemas
    content: "Extend stored-agent schemas for integration connections + bindings alongside toolProviders"
    status: pending
  - id: runtime
    content: "Resolve Integration tools + bindings when stored agent loads"
    status: pending
  - id: studio-ui
    content: "Playground/Builder UI — connect scopes, pick tools, pick trigger→sink; reconnect-for-scope UX"
    status: pending
  - id: msw-tests
    content: "Playground MSW tests for integrations UI (per playground-msw-tests skill)"
    status: pending
isProject: true
---

# Integration Builder / stored agents (P3)

**Parent:** [unified_integration_layer_7f3a91c2.plan.md](unified_integration_layer_7f3a91c2.plan.md)  
**Depends on:** [integration_events_triggers_p1_9a2f6e31.plan.md](integration_events_triggers_p1_9a2f6e31.plan.md)

## Current baseline

- Stored agents pin **`toolProviders`** only ([`stored-agents.ts`](packages/server/src/server/schemas/stored-agents.ts) + tool-providers schema) — no signal subs, no workflow trigger bindings.
- Builder Integrations UI = ToolProvider connections (Composio/Arcade).
- Channels (Slack) are a separate opt-in — not the same as automation triggers.

## Goal

EE Agent Builder moves from “integrations = tools only” to **tools + trigger bindings** on stored agents.

## Ship

- Schema: `integrationConnections` / `integrationBindings` (compat with `toolProviders`)
- Runtime resolve on stored agent load
- UI: connect (grant scopes) → select tools → bind trigger → sink
- Show “reconnect required for missing OAuth scope”

## Ownership

Coordinate Mastra EE / playground owners. Collaborator may pair on schemas/runtime; UI often team-owned.

## Exit

Stored agent snapshot portable with tools + bindings; MSW coverage for critical flows.

## Effort

Team-owned or paired · playground MSW tests required
