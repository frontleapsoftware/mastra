# Proposal: Unified Integration Layer for Tools + Triggers

**Author:** Mastra collaborator  
**Audience:** Mastra core / product  
**Goal:** One reusable integration model so SaaS events (email, Slack, GitHub, Linear, …) can start workflows *or* signal agents — and the same connection powers tools — without per-product hardcoding.

Engineering execution plans: [`.cursor/plans/unified_integration_layer_7f3a91c2.plan.md`](.cursor/plans/unified_integration_layer_7f3a91c2.plan.md)

---

## Problem

Mastra currently has **parallel integration stacks** that do not share an event model or credential vault:

1. **ToolProvider** (Composio / Arcade) — OAuth + actions → agent tools / Agent Builder  
2. **ChannelProvider** (Slack) — messaging ingress → agent chat  
3. **SignalProvider** (only `@mastra/github-signals` shipped) — external events → **agent-thread notifications only**

Workflows start via manual run, **cron**, or custom/Inngest webhooks. There is **no first-class path** from a SaaS event → `workflow.start`.

**Concrete gaps (current codebase):**

- “Email → create order” or “Slack message → triage workflow” still requires bespoke glue  
- Linear / GitHub / Slack are often **hardcoded** in product surfaces (software-factory) instead of composed from a shared layer  
- Stored agents / Agent Builder pin **tools + channels**, but not signal subscriptions or workflow trigger bindings  
- Nango is not in-tree; Composio today is **tools-only** (no trigger APIs on `ComposioToolProvider`)  
- Legacy `Integration` class in core is unused and must not be the foundation  
- SignalProvider webhook route is documented (`POST /api/signals/:providerId`) but **not registered** in `packages/server`  
- SignalProvider subscriptions are **in-memory only**; `@mastra/github-signals` uses thread metadata; factory adds a third store (`github_signal_subscriptions`)  
- **No shared OAuth/secret vault** — factory Linear stores tokens plaintext in app Postgres; ToolProvider only stores connection labels/scopes while secrets stay in Composio  

**Result:** N SaaS systems × M Mastra entities (tools, signals, workflows, factory, Builder) grows as custom code, not configuration.

---

## Vision

Treat **integrations as configuration**. Signal providers and workflow event triggers are **sinks** on the same integration — not separate provider implementations per SaaS product. Tools are a **peer** surface, not an afterthought.

> One connection (e.g. Gmail / Slack / GitHub / Linear) powers:
> - **tools** (agent can act; workflow steps can act — e.g. re-fetch latest email)
> - **triggers** (events start process logic in a workflow)
> - **signals** (after workflow gating, wake a durable agent when appropriate)

Software-factory becomes easier to **extend without duplication**: shared OAuth/integration components + bindings/workflows, not copied Linear/GitHub modules. Same pattern for custom agents, stored agents, and Agent Builder.

**`IntegrationEvent`** is the reusable event abstraction. Downstream depends on bindings — not Slack/Linear/GitHub shapes.

---

## Proposed architecture

### 1. Integration (shared identity)

- Toolkit slug: `gmail`, `slack`, `github`, `linear`, …  
- Connection + tenancy scope: reuse/extend ToolProvider model (`per-author` / `shared` / `caller-supplied` + proposed `org`)  
- OAuth **grant** scopes recorded separately from tenancy  
- Capability catalogs: **actions** (tools) and **events** (triggers)  

### 2. IntegrationEvent (normalized envelope)

```
adapter / integrationId
toolkit
connectionId
eventType          e.g. gmail.message.received, slack.message.channels
externalResourceId e.g. channel, issue, PR
payload
receivedAt
```

### 3. Binding (declarative routing)

```
ON event E FROM connection C
  WHERE filters…
  MAP payload → input / notification
  THEN sink:
    - signal          → wake / notify agent thread
    - workflow.start  → start run with inputData
    - workflow.resume → resume suspended run
```

### 4. Credential backends + building blocks

- **ExternalVault** — Composio / Nango (Mastra stores labels; vendor holds tokens)  
- **NativeOAuthVault** — encrypt + refresh (patterns from factory Linear + Slack crypto)  
- **SecretStore** — DB URLs / API keys (not fake OAuth)  
- Mastra ships **reusable OAuth / webhook / poll / tool helpers** so adding Jira is compose-blocks, not a new credential design  

### 5. Integration sources (glue into platform later)

| Source | Examples |
|--------|----------|
| External / marketplace | Composio, Nango adapters |
| Mastra-provided | Slack, GitHub, … packs |
| Customer-built | Private ERP/CRM |

```ts
new Mastra({
  integrations: {
    composio: new ComposioIntegration({ ... }),
    slack: new SlackIntegration({ ... }),
    'acme-erp': new AcmeErpIntegration({ ... }),
  },
})
```

Core depends on the Integration interface only — never on Composio/Nango SDKs.

### 6. Runtime

```
SaaS webhook/poll
  → Event adapter (Composio / Nango / native)
  → IntegrationEvent bus
  → Binding router
      ├─ signal sink        → sendNotificationSignal
      ├─ workflow.start     → createRun + startAsync
      └─ tools              → resolveTools (same connectionId)
```

**SignalProvider becomes thin config** (or compat) over an event filter + signal sink — not a new package per SaaS.

---

## Business scenarios (why tools + triggers must share one Integration)

### Email → order (trigger starts logic; tools refresh data)

Webhook payloads are often thin or stale. The Integration must support **both**:

1. **Trigger** — `gmail.message.received` → `workflow.start: processOrder` (process logic on new email)  
2. **Tools on the same connection** — workflow steps call e.g. `gmail.getMessage` / fetch attachments to load the **latest** email body and metadata before creating the order  

Without a unified Integration, the workflow either trusts a partial event payload or re-implements Gmail auth beside ToolProvider. With Integration: one `connectionId` for the event binding and for `resolveTools` inside the run.

```text
Gmail trigger (new mail)
  → workflow processOrder
      → Integration tool: fetch latest email by id
      → validate / enrich / create order
      → optional: Integration tool: label or reply
```

### Slack → workflow → durable agent (gate before wake)

Not every Slack message should hit a durable agent immediately. Preferred pattern:

1. **Trigger** — `slack.message.channels` → **`workflow.start`** (filter, classify, redact, policy, fan-out)  
2. Workflow decides whether / how to continue  
3. Only then **signal sink** (or workflow step that signals) — message context delivered to a **durable agent** thread  
4. Agent replies using the **same Slack Integration tools** (`slack.postMessage`, etc.)  

ChannelProvider can remain conversational chat UX; this path is **automation**: event → workflow orchestration → durable agent, reusing one Slack connection.

```text
Slack message event
  → workflow slackTriage
      → filters / enrichment / human policy steps
      → signal durable agent (notification)
      → agent uses Slack tools on same connection
```

### Factory: extend without duplication; OAuth as a shared component base

Factory today duplicates Linear/GitHub OAuth, tools, and subscription stores. The Integration layer is the **code abstraction** so factory (and customers) extend by:

- Registering Integrations (or using Mastra packs)  
- Declaring **bindings** + **workflows** (intake, PR review, custom customer PR policy)  
- Reusing Mastra **building blocks** — especially **OAuth** (authorize, refresh, grant scopes, vault) — instead of copying `linear/connection.ts` for every new system  

Adding Jira (or a customer SaaS) to factory-style intake should mean: OAuth blocks + tools + triggers + bindings — **not** another app-local auth module.

---

## Example use cases (summary)

| Use case | Trigger | Same Integration tools | Sink chain |
|----------|---------|------------------------|------------|
| Email → order | `gmail.message.received` | Fetch latest email, label, reply | `workflow.start` (process logic) |
| Slack → durable agent | `slack.message.channels` | Post/reply in channel | **workflow first**, then signal agent |
| GitHub PR | `pull_request.opened` | Comment, labels, checks | workflow and/or signal (customer policy) |
| Linear factory intake | `issue.created` | Read issue, comment | `workflow.start: factoryIntake` |

---

## Software factory — today vs with Integration layer

**Today:** `mastracode/web/linear/*` and `github/*` own OAuth, tools, webhooks; board is UI-driven; `github_signal_subscriptions` + github-signals metadata are parallel; one-size PR path; extending factory means **code duplication**.

**After:**

- **Keep:** board (`work_items`), stages (intake→triage→planning→execute→review→done), intake prefs, org UI  
- **Leave:** token tables, per-SaaS webhook glue, `agent-tools.ts`, dual GitHub sub stores, one-off PR automation  
- **Add:** shared Integration **component base** (OAuth vault, tool resolve, triggers) + bindings pack + workflows  
- **Extend:** new intake source or customer PR workflow = new binding/workflow (and optional Integration pack) — not a fork of factory auth code

---

## Requirements (acceptance)

| ID | Requirement | Success criteria |
|----|-------------|------------------|
| R1 | One connection for tools + events | Same `connectionId` for Tool resolve and bindings |
| R2 | Tools on agents and workflow steps | `resolveTools` works in both contexts |
| R3 | Event can start a workflow | Binding → run with mapped `inputData`; visible in Studio |
| R3b | Workflow can re-use Integration tools | e.g. email→order run fetches latest message via Gmail tools on same `connectionId` |
| R3c | Workflow-before-agent | Slack (or similar) event can start a workflow that later signals a durable agent |
| R4 | Event can signal an agent | Binding → notification signal; idle thread wakes |
| R5 | Vendor-neutral core | Adapter interface; no Composio SDK in `@mastra/core` |
| R6 | Discoverable catalogs | `listTools` + `listTriggers` for Studio / Builder |
| R7 | Filter + map | Declarative filters and payload → input mapping |
| R8 | Portable config | Bindings serialize on stored agents and templates |
| R9 | Building blocks | OAuth helper + vault; Jira-like pack needs no new credential design |
| R10 | Three sources | External, Mastra-provided, customer-built registerable |
| R11 | Observability | Ingress + dispatch traces; retry / DLQ policy |

**P1 demo bar:** (1) email→order with re-fetch of latest email via Integration tools (2) Slack message → workflow → then durable agent signal (3) same `connectionId` throughout.

---

## Business case

**Product**

- Ships the default enterprise story: *when X happens, run Y* — inside Mastra  
- Moves EE Agent Builder from “integrations = tools” to event-driven agents + workflows  
- Makes software-factory a **reusable template**, not an app one-off  

**Engineering**

- Stops N×M growth across tools / signals / workflows / factory  
- Completes the incomplete SignalProvider webhook story via one ingress  
- Reuses auth already paid for by ToolProvider; collapses triple GitHub subscription paths  

**Who wins**

| Audience | Pain today | Value if shipped |
|----------|------------|------------------|
| App builders | Cron or custom webhooks to start workflows | Declarative event → workflow |
| Long-running agents | Only GitHub signals packaged | Any connected toolkit can wake a thread |
| Platform / factory | Linear/GitHub/Slack duplicated | Composable intake packs |
| EE / Builder | Integrations = tools only | Configurable agents with triggers + tools |

---

## Why Composio / Nango for events

| Need | Build every SaaS in-house | Composio / Nango events |
|------|---------------------------|-------------------------|
| OAuth + refresh | Per-SaaS forever | Reuse tool connections |
| Webhook verify + delivery | Per-provider packages | Normalized trigger catalog |
| Event schema discovery | Hand-maintained | Listable event types per toolkit |
| Time to new integration | Weeks | Config + binding |

**Constraint:** Mastra stays adapter-based. Composio/Nango are accelerators, not a hard core dependency. Mastra-provided and customer Integrations use the same contract (often NativeOAuthVault).

---

## Suggested phasing

| Phase | Scope | Outcome |
|-------|--------|---------|
| **P0** | Connection + OAuth building blocks + `resolveTools` | Tools work via Integration; path to Jira-like packs |
| **P1** | Ingress + `workflow.start` sink + adapter demo | Tools **and** event→workflow on same connection |
| **P2** | Signal sink + persist bindings; factory Linear → vault | Dual-write / migrate plaintext tokens |
| **P3** | Builder UI + stored-agent bindings (EE) | Configurable event-driven agents |
| **P4** | Factory bindings pack + customer PR example | Reusable software-factory abstraction |

---

## Ask

Endorse **P0/P1** as a core roadmap item:

> Treat SignalProvider and workflow triggers as **sinks on a shared Integration layer**, with tools as a peer surface, Composio/Nango as first adapters, and Mastra OAuth building blocks so Linear, GitHub, Slack, Gmail — and later Jira — become reusable configuration for agents, workflows, Builder, and factory templates.

Collaborator delivers RFC + spike PR series (see engineering plans).

---

## Non-goals

- Replacing ChannelProvider conversational Slack UX  
- Reviving legacy `Integration` class  
- Hard dependency on Composio/Nango inside `@mastra/core`  
- Rewriting MastraCode Web Factory UI in the first PR series  
- Re-adding `waitForEvent` — use `workflow.resume` sink instead  
