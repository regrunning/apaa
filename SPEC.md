# Authorization Protocol for Autonomous Agents (APAA) v0.1

## Status

Draft

## 1. Overview

APAA defines a protocol for authorizing autonomous agents to perform
side-effectful actions in a controlled, auditable, and revocable manner.

The protocol introduces explicit authorization decisions and short-lived
leases that bind an agent’s intent to a constrained permission.

## 2. Goals and Non-Goals

### Goals
- Explicit authorization before side effects occur
- Least-privilege, time-bounded permissions
- Enforceable constraints on action parameters
- End-to-end auditability
- Support for revocation and incident response

### Non-Goals
- Defining execution semantics
- Replacing identity or authentication protocols
- Standardizing a policy language

## 3. Conventions

The terms **MUST**, **MUST NOT**, **SHOULD**, and **MAY** are to be
interpreted as described in RFC 2119.

All timestamps MUST be RFC 3339 UTC.

---

## 4. Core Concepts

### 4.1 ActionRequest

An ActionRequest represents an agent’s intent to perform a specific
side-effectful action.

Agents MUST submit an ActionRequest before execution.

```json
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
  "timestamp": "2026-01-24T14:02:21Z"
}
