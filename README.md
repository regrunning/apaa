# APAA — Authorization Protocol for Autonomous Agents

APAA is a protocol for authorizing, constraining, and auditing
side-effectful actions performed by autonomous AI agents.

As AI agents increasingly issue refunds, modify data, and communicate
externally, most systems rely on implicit trust, prompt discipline,
or ad-hoc checks. APAA introduces an explicit, machine-enforceable
authorization layer between agent reasoning and real-world effects.

## What APAA Provides

- Explicit authorization for agent-initiated actions
- Short-lived, least-privilege authorization leases
- Parameter-level constraints (e.g. max amount, allowed targets)
- Immutable audit trails
- Revocation and incident-response support

## What APAA Is Not

- Not an agent framework
- Not a policy language
- Not a replacement for OAuth, IAM, or RBAC

APAA is a **control-plane protocol** designed to work alongside existing
identity, policy, and execution systems.

## Status

**Draft v0.1** — experimental and under active discussion.

## Read the Specification

➡️ [SPEC.md](SPEC.md)

## Related Work

APAA is adjacent to, but distinct from:
- OAuth 2.0 / OIDC (identity & delegation)
- Cloud IAM / STS systems
- Capability-based security models
- Agent communication protocols (e.g. A2A)

See SPEC.md for a more detailed comparison.

## Contributing

Issues and design feedback are welcome.
This project is intentionally protocol-first and implementation-agnostic.
