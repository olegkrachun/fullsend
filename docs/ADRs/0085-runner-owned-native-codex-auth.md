---
title: "85. Runner-owned native Codex auth with stable placeholders"
status: Accepted
relates_to:
  - security-threat-model
  - agent-infrastructure
topics:
  - runtime
  - security
  - credentials
---

# 85. Runner-owned native Codex auth with stable placeholders

Date: 2026-11-17

## Status

Accepted

## Context

The Codex runtime uses a custom API provider, `auth.command`, a runner-seeded
placeholder file and runner-held integrity digests
([ADR 0099](0099-codex-agent-runtime.md)). Native ChatGPT subscription routing
needs a different credential path while preserving the isolation boundary of
[ADR 0025](0025-provider-credential-delivery-for-sandboxed-agents.md).

The [threat model](../problems/security-threat-model.md) and
[infrastructure problem](../problems/agent-infrastructure.md) motivate keeping
reusable credentials outside agent control. This ADR aims to enable local
subscription use while the runner owns account access and can independently
revoke the sandbox's capability.

## Decision

Adopt a Go-owned broker around official Codex authentication and an OpenShell
stable placeholder to keep subscription credentials outside the sandbox.
Initial scope is explicit trusted-operator opt-in, local-only, Codex-only,
single-context and nondelegating; repository configuration, agent environment
and resume artifacts cannot authorize it. Incompatible modes fail closed,
without API, workload-identity federation (WIF) or gateway fallback; existing
API/gateway behavior and defaults stay unchanged.

Official Codex owns login, OAuth and refresh-token rotation. The broker uses a
dedicated persistent host profile, never an automatic import of the personal
profile, at a canonical non-symlink path with private permissions, verified
ownership, replacement protection and an exclusive cross-process lock. It pins
the child environment, origins and file credential store, and validates
case-sensitive stored refresh authority before and after refresh: only the
pinned official issuer/client pair or supported absent/null legacy provenance.
Independently of RPC success, it requires strict JWT syntax, integer expiry
and exact operator-selected workspace/user consistency across access/identity
claims, the RPC token and refreshed profile; these checks prove neither
signatures nor entitlement.

The native ChatGPT provider receives only the gateway-issued stable handle as
its access token, an empty refresh token and a synthetic noncredential identity
stub, never a real ID token or invented plan claim. Runner-held integrity
checks and session overrides pin auth/configuration and native routing after
agent environment loading. Verified OpenShell support and activation bind the
handle to the intended sandbox/provider attachment, revisions, policy and expiry.
Rotation replaces the host-side credential behind that handle, not the cached
native auth file. Generations expire no later than their validated tokens;
credential rejection, permanent refresh failure, expiry and teardown revoke
the capability even with `--keep-sandbox`, while detach/replacement invalidates
the old handle and transient retries cannot extend expired or revoked access.

Subscription credentials and matching inference egress rules are limited to
`chatgpt.com:443`, binary `**/codex`, and `POST /backend-api/codex/responses`
plus `GET /backend-api/wham/accounts/check`. The GET supports native workspace
routing even with a forced workspace; Responses-only scope is insufficient.
Other hosts, ports, methods and paths, including models/settings and WebSocket
GETs, stay denied; the native transport uses HTTPS server-sent events (SSE)
instead. Keep the private profile outside uploads and output collection, guard
final paths and bytes against credential disclosure, and redact diagnostics
and collected logs. Same-user host scripts remain inside the host trust
boundary: require operator-approved trusted pre/post/validation scripts or
reject the run, rather than claim environment filtering isolates the profile.

## Forge scope

Forge-neutral for GitHub, GitLab and Forgejo; no forge behavior changes.
CI and remote/delegated execution are outside initial scope and gain no
subscription access from this decision.

## Consequences

- Official Codex owns OAuth, but its deprecated auth RPC and native auth-file
  behavior require pinned compatibility tests before upgrades.
- Stable handles permit same-process rotation only with verified activation,
  bounded expiry and independent revocation.
- Profile isolation protects the sandbox boundary, not the host from trusted
  same-user scripts or a compromised operator account.
- Synthetic HTTP and verified-TLS probes cover same-process A-to-B substitution,
  expiry/recovery, detach/replacement and endpoint denials, not real-account
  OAuth refresh, entitlement, model access or fullsend integration.
- This ADR-only change ships no usable mode; implementation is separate and a
  real-account smoke test requires separate approval.

## Source provenance

Compatibility baseline: Codex 0.166.0 at
`6ff1e1ca3b30fc8b914720681fcf5fe62e95f856`
([legacy auth RPC](https://github.com/openai/codex/blob/6ff1e1ca3b30fc8b914720681fcf5fe62e95f856/codex-rs/app-server/src/request_processors/account_processor.rs),
[refresh authority](https://github.com/openai/codex/blob/6ff1e1ca3b30fc8b914720681fcf5fe62e95f856/codex-rs/login/src/auth/manager.rs))
and OpenShell 0.1.4 at `5a7728c304b5528e954e9ae900cf21f84e1a12b1`
([stable provider credentials](https://github.com/NVIDIA/OpenShell/blob/5a7728c304b5528e954e9ae900cf21f84e1a12b1/crates/openshell-core/src/provider_credentials.rs)).
