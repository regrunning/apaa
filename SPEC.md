Authorization Protocol for Autonomous Agents (APAA)
Version 0.1 — Draft
Status of This Document

This document defines APAA v0.1, a draft protocol.
It is not a standard and is subject to change.

1. Introduction

Autonomous agents increasingly perform actions with real-world side effects,
including financial transactions, data modification, and external communication.
In most current systems, such actions are authorized implicitly, through
coarse-grained permissions, or via informal safeguards.

APAA defines a protocol for explicitly authorizing autonomous agent actions
before they occur, constraining those actions, and recording their outcomes
for audit and incident response.

APAA is a control-plane protocol. It does not execute actions, define policy
languages, or replace identity systems. Instead, it standardizes how agents
request authorization, how authorization is granted, and how execution
outcomes are recorded.

2. Goals and Non-Goals
2.1 Goals

APAA aims to:

Require explicit authorization for side-effectful agent actions

Bind authorization to a specific action intent

Enforce least-privilege through constraints and expiration

Provide end-to-end auditability

Support revocation and incident response

2.2 Non-Goals

APAA does not:

Define how actions are executed

Replace authentication or identity protocols

Standardize a policy language

Govern read-only or non-side-effectful operations

3. Terminology and Conventions

The keywords MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY
are to be interpreted as described in RFC 2119.

All timestamps MUST be RFC 3339 formatted UTC timestamps.

4. Actors

APAA defines the following actors:

Agent — An autonomous system that proposes actions

Authorization Server — Evaluates requests and issues decisions

Executor — Performs side-effectful actions

Audit Log — An append-only record of protocol events

5. Protocol Overview

At a high level:

An Agent submits an ActionRequest

The Authorization Server evaluates the request

A decision (ALLOW or DENY) is returned

If allowed, a short-lived Lease is issued

The Agent presents the Lease to an Executor

The Executor enforces constraints and executes the action

The Executor submits an ExecutionReceipt

6. Data Models
6.1 ActionRequest

An ActionRequest represents an agent’s intent to perform a specific
side-effectful action.
{
  "request_id": "req_123",
  "agent_id": "support-agent-001",
  "action_type": "refund_user",
  "resource": "users/user_123",
  "parameters": {
    "user_id": "user_123",
    "amount": 50.00,
    "currency": "USD"
  },
  "context": {
    "reason": "damaged item complaint",
    "ticket_id": "TICK-456"
  },
  "dry_run": false,
  "timestamp": "2026-01-24T14:02:21Z"
}

Requirements

request_id MUST be unique per Agent

Authorization Servers MUST treat request_id as idempotent

Agents MUST NOT execute side-effectful actions without authorization

If dry_run is true, a Lease MUST NOT be issued

6.2 AuthorizationDecision

The Authorization Server evaluates an ActionRequest and returns a decision.

ALLOW
{
  "decision": "ALLOW",
  "lease_id": "lease_abc",
  "lease_token": "opaque-or-jwt",
  "expires_at": "2026-01-24T14:12:21Z",
  "constraints": {
    "max_amount": 100.00,
    "allowed_parameters": ["user_id", "amount", "currency"],
    "resource_allowlist": ["users/*"]
  },
  "constraint_hash": "sha256:abcd...",
  "audit_id": "audit_789"
}
DENY
{
  "decision": "DENY",
  "reason": "Policy violation",
  "audit_id": "audit_790"
}
Requirements

Default decision MUST be DENY

All decisions MUST be logged

ALLOW decisions MUST include a Lease

expires_at MUST be enforced
6.3 Lease

A Lease is a short-lived, unforgeable authorization granting permission
to perform a specific action under constraints.

Requirements

Leases MUST be bound to:

agent_id

action_type

constraints

expiration

Leases MUST be revocable

Executors MUST reject expired or revoked Leases

6.4 Constraints

Constraints restrict how an authorized action may be executed.

Constraint types MAY include:

Parameter constraints (e.g. max_amount)

Resource scope constraints

Rate or count limits

Template or method allowlists

Executors MUST enforce all applicable constraints.

6.5 ExecutionReceipt

An ExecutionReceipt records the outcome of an executed action.
{
  "lease_id": "lease_abc",
  "audit_id": "audit_789",
  "executor_id": "payments-service",
  "executed_at": "2026-01-24T14:07:10Z",
  "outcome": "SUCCESS",
  "side_effect_ids": {
    "refund_id": "rf_556677"
  }
}
Requirements

Executors MUST submit a receipt for executed actions

Receipts MUST reference the Lease and audit_id
7. Endpoints
7.1 POST /authorize

Requests authorization for an ActionRequest.

MUST be idempotent by request_id

MUST return an AuthorizationDecision

7.2 POST /revoke

Revokes an active Lease.

Revocation MUST be idempotent

Revoked Leases MUST NOT be honored

7.3 POST /receipt

Submits an ExecutionReceipt.

MAY be asynchronous

MUST be logged

7.4 GET /audit/{audit_id}

Retrieves audit records.

Access SHOULD be restricted

8. Audit Logging

Authorization Servers MUST maintain append-only, tamper-evident logs.

Audit events include:

ActionRequested

DecisionMade

LeaseRevoked

ExecutionReceiptLogged

9. Security Considerations

All endpoints MUST use HTTPS

Clients MUST authenticate

Leases MUST be unforgeable

Systems MUST fail closed if authorization cannot be obtained

Sensitive data SHOULD be protected at rest

10. Error Handling

Authorization Servers MUST NOT return partial ALLOW responses

Constraint violations MUST result in execution failure

Revocation MUST take effect immediately

11. Privacy Considerations

Context fields SHOULD support redaction

Audit access SHOULD be limited

Receipts MUST NOT include secrets

12. Related Work (Informative)

APAA is adjacent to:

OAuth 2.0 and OIDC

Cloud IAM and STS systems

Capability-based security models

Agent communication protocols

APAA focuses on action-level authorization, not identity delegation.
