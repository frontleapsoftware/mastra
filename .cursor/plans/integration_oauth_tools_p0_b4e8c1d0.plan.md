---
name: Integration OAuth and tools P0
overview: P0 — IntegrationConnection, CredentialBackend, OAuth building blocks, resolveTools. Grounded in current ToolProvider + factory Linear OAuth + Slack crypto. Parent unified_integration_layer_7f3a91c2.
todos:
  - id: rfc-contract
    content: "RFC — Integration contract, tenancy vs OAuth grant scopes, CredentialBackend variants"
    status: pending
  - id: oauth-helper
    content: "Land OAuth helper — generalize factory Linear refresh + GitHub/Linear state HMAC patterns"
    status: pending
  - id: vaults
    content: "NativeOAuthVault (from Slack crypto) + ExternalVault (Composio today) + SecretStore"
    status: pending
  - id: resolve-tools
    content: "Integration.resolveTools wrapping ToolProvider; prove agent + workflow step"
    status: pending
  - id: docs-building-blocks
    content: "Docs — Building a Mastra Integration (e.g. Jira) on these blocks"
    status: pending
isProject: true
---

# Integration OAuth + tools (P0)

**Parent:** [unified_integration_layer_7f3a91c2.plan.md](unified_integration_layer_7f3a91c2.plan.md)  
**Next:** [integration_events_triggers_p1_9a2f6e31.plan.md](integration_events_triggers_p1_9a2f6e31.plan.md)

## Current baseline

- **ToolProvider** ([`packages/core/src/tool-provider`](packages/core/src/tool-provider)): tools + authorize; connection rows are labels/scopes only; tenancy `per-author` | `shared` | `caller-supplied` — **no `org`**.
- **Composio/Arcade** ([`packages/editor/src/providers`](packages/editor/src/providers)): ExternalVault-style secrets; **no triggers**.
- **Factory Linear** ([`mastracode/web/src/web/linear`](mastracode/web/src/web/linear)): best in-tree native OAuth lifecycle (refresh single-flight, grant `scope`) but **plaintext** tokens in app Postgres — not a platform API.
- **Slack channel crypto** ([`channels/slack/src/crypto.ts`](channels/slack/src/crypto.ts)): best encrypt-at-rest pattern for NativeOAuthVault.
- **Legacy** [`integration.ts`](packages/core/src/integration/integration.ts): unused — do not extend.

## Goal

Extract reusable credential + tools APIs from those patterns so factory/Studio/customers stop forking OAuth.

## Ship

- `IntegrationConnection` + storage (inmemory/libsql first)
- `CredentialBackend`: ExternalVault | NativeOAuthVault | SecretStore
- OAuth helper (state, exchange, refresh, grant scopes)
- `listTools` / `resolveTools` for agents **and** workflow steps
- Optional tenancy `org` for factory-style ownership
- ToolProvider remains compat until callers migrate

## Exit

Agent + workflow step call a tool via Integration + `connectionId`. Jira-like pack = compose blocks, not new vault design.

## Maintainer checks

Merge vs sit-beside ToolProvider connections · native vault in OSS · `org` · SecretStore v1
