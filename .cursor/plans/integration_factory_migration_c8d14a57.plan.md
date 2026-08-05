---
name: Integration factory migration
overview: P2/P4 — Migrate current MastraCode factory (Linear/GitHub/board as implemented) onto Integration connections, tools, and bindings. Parent unified_integration_layer_7f3a91c2.
todos:
  - id: persist-bindings
    content: "Persist integration bindings; retire dual GitHub sub stores over time"
    status: pending
  - id: migrate-linear
    content: "Migrate linear_connections plaintext tokens → NativeOAuthVault; replace linear/agent-tools"
    status: pending
  - id: github-adapter
    content: "GitHub Integration — unify github-signals metadata + github_signal_subscriptions behind triggers/bindings"
    status: pending
  - id: bindings-pack
    content: "Factory bindings pack for intake / default PR / optional Slack"
    status: pending
  - id: customer-pr-example
    content: "Example — customer PR workflow binding on same GitHub Integration"
    status: pending
isProject: true
---

# Integration factory migration (P2/P4)

**Parent:** [unified_integration_layer_7f3a91c2.plan.md](unified_integration_layer_7f3a91c2.plan.md)  
**Depends on:** [integration_events_triggers_p1_9a2f6e31.plan.md](integration_events_triggers_p1_9a2f6e31.plan.md)

## Current baseline (MastraCode Web factory)

| Area | As implemented |
|------|----------------|
| Board | [`factory/`](mastracode/web/src/web/factory) `work_items` — org + `githubProjectId`; sources `github-issue` \| `github-pr` \| `linear-issue` \| `manual` |
| Stages (UI) | intake → triage → planning → execute → review → done ([`stages.ts`](mastracode/web/src/web/ui/domains/factory/stages.ts)) |
| Linear | [`linear/`](mastracode/web/src/web/linear) — org OAuth, plaintext tokens, `agent-tools.ts`, routes/client |
| GitHub | App install + projects; **`github_signal_subscriptions`** ([`subscriptions.ts`](mastracode/web/src/web/github/subscriptions.ts)) |
| Intake | Prefs which projects sync — not credentials |
| Runs | Board/UI starts agent sessions — not declarative event→workflow bindings |

## Goal

Keep board + prefs + org UI. Move auth, tools, webhooks, and PR subscription glue onto Integration.

## Today → after

| Keep | Move to Integration layer |
|------|---------------------------|
| `work_items`, stage UX, metrics | `linear_connections` tokens → vault |
| Intake project/repo prefs | `linear/agent-tools` → Integration tools |
| Org tenancy UI | GitHub webhooks + both sub stores → triggers/bindings |
| | One-off PR path → customer-replaceable workflows |

## Default bindings pack

- `linear.issue.created` → `factoryIntake`
- `github.issues.opened` → `factoryIntake`
- `github.pull_request.opened` → `factoryPrReview`
- Optional `slack.message.channels` → `factorySlackTriage`

## Customer PR workflows

Same GitHub Integration; org bindings → custom workflows / signal — no factory fork.

## Exit

Linear tokens off plaintext columns · intake/PR use Integration tools · dual GitHub sub stores deprecated or dual-written · customer override documented
