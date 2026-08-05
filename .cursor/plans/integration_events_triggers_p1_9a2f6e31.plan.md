---
name: Integration events and triggers P1
overview: P0b/P1 — IntegrationEvent, bindings, webhook ingress, workflow.start. Grounded in current SignalProvider gaps and triple GitHub subscription persistence. Parent unified_integration_layer_7f3a91c2.
todos:
  - id: event-types
    content: "Land IntegrationEvent, IntegrationBinding, TriggerAdapter, BindingRouter unit tests"
    status: pending
  - id: dispatcher-route
    content: "Ship real webhook ingress via createRoute (closes undocumented /api/signals/:providerId gap)"
    status: pending
  - id: workflow-sink
    content: "workflow.start sink via getWorkflow → createRun → startAsync"
    status: pending
  - id: signal-sink
    content: "signal sink via sendNotificationSignal (existing notifications path)"
    status: pending
  - id: adapter-demo
    content: "Native or Composio adapter demo — tool + trigger→workflow + tool in step on same connection"
    status: pending
  - id: docs-changeset
    content: "Docs Connect / Use tools / Bind triggers + changeset @experimental"
    status: pending
isProject: true
---

# Integration events + triggers (P1)

**Parent:** [unified_integration_layer_7f3a91c2.plan.md](unified_integration_layer_7f3a91c2.plan.md)  
**Depends on:** [integration_oauth_tools_p0_b4e8c1d0.plan.md](integration_oauth_tools_p0_b4e8c1d0.plan.md)

## Current baseline (signals)

| Mechanism | Behavior today |
|-----------|----------------|
| `SignalProvider` registry | In-memory Maps; cleared on `stop()` — [`signal-provider.ts`](packages/core/src/signals/signal-provider.ts) |
| `handleWebhook?` | Documented as `POST /api/signals/:providerId` — **no server route** |
| Agent HTTP | `SEND_AGENT_SIGNAL_ROUTE` only (inject into agent, not provider webhooks) |
| `@mastra/github-signals` | Poll + tools; PR subs in **thread metadata** |
| Factory | **`github_signal_subscriptions`** table in app Postgres — third store |
| Delivery | `sendNotificationSignal` → notifications storage → agent thread |
| Workflows | Cron/manual/`createRun` — **no** SignalProvider → `workflow.start` |
| Composio | No trigger APIs — P1 may use native webhook kit first |

## Goal

One event path: adapters → **`IntegrationEvent`** → bindings → signal **or** workflow. Same `connectionId` as tools (P0). Collapse the three GitHub sub mechanisms over time onto bindings.

## Ship

```text
SaaS webhook/poll → TriggerAdapter → IntegrationEvent → BindingRouter
  → workflow.start | signal | workflow.resume
```

- Types + BindingRouter
- HTTP ingress (`createRoute` + `SERVER_ROUTES`) — implement what signals docs/comments promised
- Sinks: existing `createRun`/`startAsync` + `sendNotificationSignal`
- Vendor-neutral core; Composio/Nango as optional adapters

## Exit / demo scenarios

1. **Email→order:** trigger → workflow → step `getMessage` (latest email) on same connection → create order  
2. **Slack→agent:** trigger → triage workflow → then signal durable agent; agent posts via Slack tools on same connection  
3. Generic: tool call + trigger→workflow + tool in step on connection C

## Effort

~1 week types · ~2–4 weeks ingress + demo
