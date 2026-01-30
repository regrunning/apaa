# APAA - Authorization Protocol for AI Agents

**Version:** 0.1 (Draft)  
**Status:** Request for Comments  
**Author:** Sara Almeida ([@regrunning](https://twitter.com/regrunning))  
**Reference Implementation:** [Reg.Run](https://reg.run)

---

## Abstract

APAA (Authorization Protocol for AI Agents) defines a standardized protocol for AI agents to request, receive, and audit authorization for actions in production systems.

Unlike existing authorization systems designed for human users (OAuth, IAM), APAA is purpose-built for autonomous agents that operate with business logic, approval workflows, and compliance requirements.

---

## Table of Contents

1. [Motivation](#motivation)
2. [Core Concepts](#core-concepts)
3. [Protocol Specification](#protocol-specification)
4. [Implementation Requirements](#implementation-requirements)
5. [Security Considerations](#security-considerations)
6. [Examples](#examples)
7. [FAQ](#faq)
8. [Contributing](#contributing)

---

## Motivation

### The Problem

AI agents are being deployed in production with **ambient authority** - full credentials to perform any action. This creates:

- **Security risks:** Agents can be prompt-injected or exploited
- **Compliance issues:** No audit trail of who authorized what
- **Operational chaos:** Manual approval of every action doesn't scale
- **Business risk:** No enforcement of business policies

### Why Existing Solutions Don't Work

| Solution | Why It Doesn't Work for AI Agents |
|----------|----------------------------------|
| **IAM (Okta, Auth0)** | Designed for human users, session-based, no business logic |
| **Capability Tokens (Tenuo)** | Cryptographic scope, but no approval workflows or business rules |
| **Sandboxing** | Prevents technical exploits, not business policy violations |
| **Prompts** | "Just use better prompts" is not a security model |

### The APAA Solution

APAA provides:
- ✅ **Task-scoped authorization** (not session-based)
- ✅ **Business policy enforcement** (not just access control)
- ✅ **Human-in-the-loop workflows** (approval queues)
- ✅ **Compliance-ready audit trails** (SOX, GDPR, HIPAA)
- ✅ **Standard protocol** (any system can implement)

---

## Core Concepts

### 1. Authorization Flow
```
┌─────────────┐
│  AI Agent   │
└──────┬──────┘
       │ 1. Authorization Request
       ▼
┌─────────────────────┐
│  APAA Server        │
│  (Policy Engine)    │
└──────┬──────────────┘
       │ 2. Policy Evaluation
       ▼
┌─────────────────────┐
│  Decision:          │
│  • Auto-approve     │
│  • Require approval │
│  • Deny             │
└──────┬──────────────┘
       │
       ▼
┌─────────────────────┐
│  If approval needed:│
│  Human approves     │
│  via dashboard      │
└──────┬──────────────┘
       │ 3. Final Decision
       ▼
┌─────────────┐
│  AI Agent   │
│  Proceeds   │
│  (or not)   │
└─────────────┘
```

### 2. Key Principles

**Task-Scoped:** Authorization is per-action, not per-session  
**Policy-Based:** Business rules, not just permissions  
**Auditable:** Every decision logged with full context  
**Human-Friendly:** Ops teams configure policies without code  
**Developer-Friendly:** Simple API integration

---

## Protocol Specification

### Message Format

All APAA messages use JSON and include:
- `protocol`: Always `"apaa/0.1"`
- `timestamp`: ISO 8601 timestamp
- `id`: Unique identifier for tracking

### 1. Authorization Request

**Endpoint:** `POST /apaa/authorize`
```json
{
  "protocol": "apaa/0.1",
  "id": "req-abc123",
  "timestamp": "2025-01-31T10:30:00Z",
  
  "agent": {
    "id": "agent-customer-service-001",
    "type": "ai_agent",
    "name": "CustomerServiceBot",
    "version": "2.1.0"
  },
  
  "action": {
    "type": "refund",
    "resource": "order/12345",
    "parameters": {
      "amount": 250.00,
      "currency": "USD",
      "reason": "defective_product"
    }
  },
  
  "context": {
    "customer_id": "cust-456",
    "customer_lifetime_value": 5000.00,
    "order_date": "2025-01-15",
    "days_since_purchase": 15,
    "customer_tier": "gold"
  }
}
```

**Required Fields:**
- `protocol`, `id`, `timestamp`
- `agent.id`, `agent.type`
- `action.type`, `action.resource`

**Optional Fields:**
- `action.parameters` (action-specific data)
- `context` (business context for policy evaluation)

---

### 2. Authorization Response

**Response:** `200 OK`
```json
{
  "protocol": "apaa/0.1",
  "request_id": "req-abc123",
  "timestamp": "2025-01-31T10:30:01Z",
  
  "decision": "requires_approval",
  
  "policy": {
    "name": "refund_policy",
    "version": "1.2",
    "rule_matched": "require_approval_100_to_500"
  },
  
  "reason": "Amount $250 requires team lead approval per policy v1.2",
  
  "approval": {
    "status": "pending",
    "approver_role": "team_lead",
    "approval_url": "https://dashboard.example.com/approvals/req-abc123",
    "expires_at": "2025-01-31T11:30:00Z",
    "timeout_action": "deny"
  },
  
  "metadata": {
    "evaluation_time_ms": 12,
    "policy_version": "1.2",
    "server_id": "apaa-server-01"
  }
}
```

**Decision Types:**

| Decision | Meaning | Agent Action |
|----------|---------|--------------|
| `approved` | Action authorized | Proceed immediately |
| `denied` | Action not authorized | Do not proceed |
| `requires_approval` | Pending human decision | Wait for approval or timeout |

---

### 3. Policy Format

Policies are defined in YAML (or JSON):
```yaml
policy:
  name: "refund_policy"
  version: "1.2"
  description: "Authorization rules for customer refunds"
  
  action_type: "refund"
  
  rules:
    auto_approve:
      conditions:
        - "action.parameters.amount < 100"
        - "context.days_since_purchase < 30"
        - "NOT context.customer.flagged_fraud"
      
    require_approval:
      conditions:
        - "action.parameters.amount >= 100"
        - "action.parameters.amount < 500"
        - "context.days_since_purchase < 30"
      approver: "team_lead"
      timeout: "1h"
      timeout_action: "deny"
      
    deny:
      conditions:
        - "action.parameters.amount >= 500"
        - "OR context.days_since_purchase >= 30"
        - "OR context.customer.flagged_fraud"
      reason: "Exceeds refund limits or policy violations"
```

**Rule Evaluation Order:**
1. Check `deny` rules first (fail-fast)
2. Check `require_approval` rules
3. Check `auto_approve` rules
4. Default: `deny` (fail-secure)

---

### 4. Approval Submission

**Endpoint:** `POST /apaa/approve/{request_id}`
```json
{
  "protocol": "apaa/0.1",
  "request_id": "req-abc123",
  "timestamp": "2025-01-31T10:45:00Z",
  
  "approver": {
    "id": "user-manager-001",
    "email": "manager@company.com",
    "role": "team_lead",
    "name": "Jane Manager"
  },
  
  "decision": "approved",
  "comment": "Customer is VIP tier, approved refund"
}
```

**Response:**
```json
{
  "protocol": "apaa/0.1",
  "request_id": "req-abc123",
  "status": "approved",
  "timestamp": "2025-01-31T10:45:01Z",
  
  "audit_id": "audit-xyz789"
}
```

---

### 5. Audit Trail

**Endpoint:** `GET /apaa/audit?from=2025-01-01&to=2025-01-31`
```json
{
  "protocol": "apaa/0.1",
  "entries": [
    {
      "audit_id": "audit-xyz789",
      "request_id": "req-abc123",
      "timestamp": "2025-01-31T10:45:01Z",
      
      "agent": {
        "id": "agent-customer-service-001",
        "name": "CustomerServiceBot"
      },
      
      "action": {
        "type": "refund",
        "resource": "order/12345",
        "parameters": {"amount": 250.00}
      },
      
      "decision": "approved",
      
      "policy": {
        "name": "refund_policy",
        "version": "1.2",
        "rule": "require_approval_100_to_500"
      },
      
      "approver": {
        "id": "user-manager-001",
        "email": "manager@company.com",
        "comment": "Customer is VIP tier, approved refund"
      },
      
      "compliance": {
        "retention_period": "7_years",
        "data_classification": "sensitive",
        "regulations": ["SOX", "PCI-DSS"]
      }
    }
  ]
}
```

---

## Implementation Requirements

APAA-compliant systems **MUST** implement:

### 1. Core Endpoints

- `POST /apaa/authorize` - Authorization requests
- `POST /apaa/approve/{id}` - Approval submission  
- `POST /apaa/deny/{id}` - Denial submission
- `GET /apaa/audit` - Audit trail retrieval

### 2. Policy Engine

- Evaluate requests against policies
- Support auto-approve, require-approval, deny decisions
- Return decision within **100ms p99 latency**
- Support policy versioning

### 3. Approval Workflow

- Human approval interface (dashboard, CLI, API)
- Timeout mechanism (default: 1 hour)
- Notification system (email, Slack, webhook)
- Approval delegation

### 4. Audit Trail

- Immutable log of all decisions
- Queryable by date, agent, action, decision
- Retention per compliance requirements
- Export capabilities (JSON, CSV)

### 5. Security

- TLS 1.3 for all communications
- Agent authentication (API keys, mTLS, OAuth)
- Request signature verification
- Replay attack prevention (nonce + timestamp)
- Rate limiting (default: 100 req/sec per agent)

---

## Security Considerations

### Agent Authentication

APAA implementations SHOULD support:
- API keys (Bearer tokens)
- mTLS (certificate-based)
- OAuth 2.0 client credentials

### Replay Attack Prevention

- Include `timestamp` in all requests
- Reject requests older than 5 minutes
- Maintain nonce cache for deduplication

### Policy Security

- Policy changes require authentication
- Policy versioning (immutable history)
- Rollback capabilities
- Audit policy changes

### Data Privacy

- PII in context fields should be minimal
- Support data masking in audit logs
- GDPR right-to-deletion compliance

---

## Examples

### Example 1: Simple Refund

**Request:**
```json
{
  "protocol": "apaa/0.1",
  "id": "req-001",
  "agent": {"id": "bot-001", "type": "ai_agent"},
  "action": {
    "type": "refund",
    "resource": "order/123",
    "parameters": {"amount": 50}
  }
}
```

**Response:**
```json
{
  "protocol": "apaa/0.1",
  "request_id": "req-001",
  "decision": "approved",
  "reason": "Amount < $100, auto-approved"
}
```

### Example 2: Requires Approval

**Request:**
```json
{
  "protocol": "apaa/0.1",
  "id": "req-002",
  "agent": {"id": "bot-001", "type": "ai_agent"},
  "action": {
    "type": "refund",
    "resource": "order/456",
    "parameters": {"amount": 250}
  }
}
```

**Response:**
```json
{
  "protocol": "apaa/0.1",
  "request_id": "req-002",
  "decision": "requires_approval",
  "approval": {
    "status": "pending",
    "approver_role": "manager",
    "approval_url": "https://dashboard.example.com/approve/req-002"
  }
}
```

### Example 3: Denied

**Request:**
```json
{
  "protocol": "apaa/0.1",
  "id": "req-003",
  "agent": {"id": "bot-001", "type": "ai_agent"},
  "action": {
    "type": "delete",
    "resource": "database/production",
    "parameters": {}
  }
}
```

**Response:**
```json
{
  "protocol": "apaa/0.1",
  "request_id": "req-003",
  "decision": "denied",
  "reason": "Production database deletion not allowed"
}
```

---

## FAQ

### How is APAA different from OAuth?

**OAuth:** Authentication + authorization for human users  
**APAA:** Authorization for AI agents with business logic

OAuth is session-based, APAA is task-based.

### How is APAA different from capability tokens (Tenuo)?

**Capability tokens:** Cryptographic scope enforcement  
**APAA:** Business policy + approval workflows

They're complementary - you could implement APAA on top of capability tokens.

### Can APAA integrate with existing IAM systems?

Yes! APAA can use IAM for agent authentication, then add the authorization layer.

### What policy languages does APAA support?

APAA is policy-language agnostic. Implementations can use:
- Simple condition syntax (like examples above)
- Cedar (AWS's policy language)
- OPA/Rego
- Custom DSL

### Is APAA production-ready?

APAA v0.1 is a **draft specification**. We're seeking feedback before v1.0.

[Reg.Run](https://reg.run) is the reference implementation.

### How do I implement APAA?

See [Implementation Guide](./IMPLEMENTATION.md) and [Reference Implementation](https://github.com/regrunning/regrun_mvp).

## Contributing

APAA is an open specification. Contributions welcome!

**Feedback:**
- GitHub Issues: [apaa-spec/issues](https://github.com/regrunning/apaa-spec/issues)
- Email: sara@reg.run
- Twitter: [@regrunning](https://twitter.com/regrunning)

**Implementations:**
If you build an APAA-compliant system, let us know! We'll list it here.

## Reference Implementation

**[Reg.Run](https://reg.run)** - First APAA-compliant authorization system

- ✅ Full APAA v0.1 support
- ✅ Dashboard for approval workflows
- ✅ Policy management UI
- ✅ Compliance-ready audit trails

Try it: https://regrunmvp.replit.app

---

## License

**APAA Specification:** CC0 1.0 Universal (Public Domain)

Anyone can implement APAA. Implementations may use any license.

## Changelog

**v0.1 (2025-01-31)** - Initial draft specification


## Acknowledgments

Inspired by:
- OAuth 2.0 (authorization for users)
- Tenuo (capability tokens for agents)
- AWS Cedar (policy language)
- Open Policy Agent (policy engine)

Special thanks to the AI safety and security community for feedback.

---
**Don't sweat it, reg it.** 😎

Built with ❤️ by [@regrunning](https://twitter.com/regrunning)
