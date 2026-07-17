---
id: adrs-adr001
title: 'ADR001: Self-hosted Firewall Support'
# prettier-ignore
description: Architecture Decision Record (ADR) for adding self-hosted firewall endpoint support to the Socket GitHub Action
---

## Context

The Socket GitHub Action currently requires a `socket-token` (Socket API key) when running in enterprise firewall mode. This API key is used to authenticate the `sfw` CLI binary against the Socket API for package threat analysis.

We operate a self-hosted Socket Registry Firewall proxy which already holds and manages the Socket API key centrally. Requiring a separate API key in every GitHub Actions workflow that uses this action creates a duplication of secrets management and a divergence between our local development and CI flows — local dev routes through the proxy automatically, but CI would need its own API key.

We reached out to Socket about a native `SOCKET_SECURITY_FIREWALL_ENDPOINT` env var for the `sfw` CLI but no such capability exists upstream. The only documented configuration is `SOCKET_SECURITY_API_TOKEN`.

## Decision

We will add a `socket-firewall-endpoint` input to this action. When provided alongside enterprise mode, the action will:

1. Skip downloading the `sfw` binary — the deployed firewall proxy handles all package validation.
2. Export `SOCKET_SECURITY_FIREWALL_ENDPOINT` as an environment variable so any tooling that does support it can consume it.
3. Allow enterprise mode to be activated by either `socket-token` or `socket-firewall-endpoint` — the endpoint takes conceptual priority since it replaces the need for a per-deployment API key.

The firewall endpoint value will be the fully qualified URL of the proxy.

## Consequences

**Positive:**
- Single source of truth for the Socket API key — it lives only in the firewall proxy deployment, not spread across CI workflows.
- Local development and CI use the same proxy, providing consistent behavior.
- No binary download in firewall-endpoint mode, reducing CI step time and eliminating dependency on the `socketdev/firewall-release` GitHub repo at runtime.

**Negative:**
- The action no longer installs the `sfw` binary when using a firewall endpoint. If the `sfw` CLI later adds native firewall endpoint support, the action would need to resume downloading the binary.
- The `SOCKET_SECURITY_FIREWALL_ENDPOINT` env var is not yet recognized by upstream `sfw`. It is exported preemptively in anticipation of upstream support or for use by our own wrapper tooling that respects it.
- Users must ensure the firewall proxy is reachable from their CI runners — network policy may need adjustment.

**Neutral:**
- The existing `socket-token` path is unchanged. Users who prefer or need the API key approach can continue using it.