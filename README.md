# CallSheet

A CRM for blue-collar service businesses (HVAC, plumbing, electrical, etc.) that helps them prioritize customer outreach and send SMS estimates.



![Status](https://img.shields.io/badge/status-production-green)
![Stack](https://img.shields.io/badge/stack-FastAPI%20%2B%20React%20%2B%20PostgreSQL-blue)

---

## What It Does

A plumber opens CallSheet in the morning and sees: "Here's who you should call today, ranked by priority."

The system scores every customer based on:
- **Value** - How much revenue do they bring?
- **Urgency** - How overdue are they for service?
- **Risk** - Are we losing them?
- **Opportunity** - Is now a good time to reach out?

They can then send SMS estimates directly from the app. Customers receive a link, view the estimate, and accept it - all tracked in real-time.

---

## Architecture

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────┐
│   React SPA     │────▶│  FastAPI Backend │────▶│ PostgreSQL  │
│   (Vite)        │     │  (Hexagonal)     │     │             │
└─────────────────┘     └──────────────────┘     └─────────────┘
        │                    │   │   │
        │                    │   │   │
   ┌────▼────┐          ┌───▼┐ ┌▼──┐ ┌▼────────┐
   │  Clerk  │          │Twi-│ │St-│ │Google   │
   │  Auth   │          │lio │ │ri-│ │Calendar │
   └─────────┘          └────┘ └───┘ └─────────┘
```

### Tech Stack

| Layer | Technology | Why |
|-------|------------|-----|
| Frontend | React 18 + Vite | Fast dev experience, mainstream ecosystem |
| Backend | FastAPI (Python) | Modern async Python, auto-generated API docs |
| Database | PostgreSQL | Relational data, industry standard |
| Auth | Clerk | Hosted auth, JWT-based, handles OAuth/MFA |
| SMS | Twilio | Reliable, good API |
| Payments | Stripe | Industry standard for SaaS billing |
| Hosting | Vercel (frontend) + Railway (backend + DB) | Simple deployment from Git |

### Hexagonal Architecture

The backend uses ports and adapters (hexagonal architecture):

```
app/
├── domain/           # Business logic, no framework dependencies
├── ports/            # Abstract interfaces (what I need)
├── services/         # Use cases, orchestration
└── adapters/
    ├── driving/      # Things that call us (API routes)
    └── driven/       # Things we call (DB, Twilio, Stripe)
```

**Why this pattern:**
- Business logic is testable without mocking frameworks
- External services are swappable (mock Twilio in tests)
- Dependencies flow inward (domain never imports adapters)

---

## Core Features

### 1. Priority Scoring Algorithm

Each customer gets a score from 0-100:

| Component | Points | What It Measures |
|-----------|--------|------------------|
| Value | 0-40 | Invoice size percentile, repeat frequency, contract status |
| Urgency | 0-30 | Days since last service ÷ expected interval |
| Risk | 0-20 | Missed contacts, irregular service patterns, stale estimates |
| Opportunity | 0-10 | Seasonal factors, upsell potential, recent estimate activity |

The "Today View" shows customers with score ≥ 50, sorted highest first.

### 2. SMS Estimates

State machine for estimate lifecycle:

```
      ┌──────────┐
      │  DRAFT   │
      └────┬─────┘
           │ SMS sent successfully
      ┌────▼─────┐
      │   SENT   │──── expires_at passed ────┐
      └────┬─────┘                           │
           │ customer clicks link            │
      ┌────▼─────┐                      ┌────▼─────┐
      │  VIEWED  │──── expires_at ──────▶│ EXPIRED  │
      └────┬─────┘                      └──────────┘
           │ customer accepts
      ┌────▼─────┐
      │ ACCEPTED │
      └──────────┘
```

**Key design decisions:**
- **Token-based public access**: Customers view/accept estimates without logging in. Each estimate has a cryptographic token in the URL.
- **Lazy expiration**: No cron job. Expiration is checked on read. Simpler ops, works at this scale.
- **Graceful degradation**: If Twilio fails, estimate stays in DRAFT. User sees error, can retry.

### 3. Multi-Tenant Isolation

Every table has `account_id`. Every query filters by it.

```python
# Every repository method looks like this
def get_customer_by_id(self, customer_id, account_id):
    return self.db.query(Customer).filter(
        Customer.id == customer_id,
        Customer.account_id == account_id  # Always present
    ).first()
```

A user from Business A cannot see Business B's data. Period.

---

## Auth Flow

```
Browser                 Clerk                Backend
  │                       │                    │
  ├──Sign in/up──────────▶│                    │
  │◀──Session + JWT───────┤                    │
  │                       ├──webhook: user.created──▶│
  │                       │                    ├──Create Account + User
  │                       │                    │
  ├──GET /api/today───────────────────────────▶│
  │  Authorization: Bearer <jwt>               │
  │                       │                    ├──Verify JWT via JWKS
  │                       │                    ├──Extract clerk_user_id
  │                       │                    ├──Query User by clerk_id
  │                       │                    ├──Filter data by account_id
  │◀──200 + data──────────────────────────────┤
```

**The webhook fallback pattern:**

Webhooks can fail. If a user signs up and the webhook doesn't fire, they'd be locked out.

Solution: `get_current_user()` checks if the user exists. If not, it calls Clerk's API directly and creates the account on-demand. User never knows anything went wrong.

---

## Production Challenges

### The Clerk Proxy Problem

On localhost, Clerk just works. In production with a custom domain, auth requests need to route through your domain.

**The fix:**
1. Frontend configured with `proxyUrl` and `clerkJSUrl` pointing to custom domain
2. Reverse proxy forwards `/__clerk/*` to Clerk's servers
3. Backend JWKS fetching updated to use proxy URL
4. Webhook signatures still verified correctly

This took real debugging. Auth failures were silent - had to trace the full flow to find the break.

### Other Production Work

- **Environment validation**: Startup checks 20+ settings, blocks deployment if misconfigured
- **Docker networking**: Database only accessible internally, API exposed via reverse proxy
- **Migrations**: Alembic runs before container starts, with health checks

---

## Database Schema

12 tables, key ones:

| Table | Purpose |
|-------|---------|
| `accounts` | Tenants (businesses). Root of multi-tenant isolation. |
| `users` | Team members within an account. Links to Clerk via `clerk_id`. |
| `customers` | The business's customers. Soft-delete via `deleted_at`. |
| `estimates` | SMS estimates with state machine. Has public `token` for customer access. |
| `timeline_events` | Activity history per customer (services, calls, etc.) |
| `audit_logs` | Security compliance trail. Who did what, when, from where. |

**Key constraints:**
- `account_id` is NOT NULL and FK-constrained on all tenant-scoped tables
- CHECK constraints on all enum columns (status, role, etc.)
- Unique constraint on estimate tokens
- Numeric(10,2) for all currency fields (no floating-point bugs)

---

## What I'd Do Differently

1. **Add database-level RLS** - Multi-tenant isolation is currently application-enforced. PostgreSQL Row-Level Security would add defense in depth.

2. **Real observability from day one** - Structured logging, metrics, distributed traces. Currently just basic logging. Posthog and sentry. 

3. **Background workers for expiration** - Lazy expiration works but creates write-on-read. A simple cron would be cleaner.

4. **TypeScript on backend** - Python's lack of compile-time type checking has caused runtime bugs that TS would catch.

5. **Idempotent webhook handlers** - Stripe webhooks can be delivered multiple times. Should deduplicate by event ID.

---

## Lessons Learned

1. **Shipping to production is a different skill than building features.** Literally everything possible broke when I took it from localhost to production. 

2. **Webhooks are unreliable.** Always have a fallback. Never assume they'll fire.

3. **AI-assisted coding works, but you have to understand what's being built.** I used Claude Code heavily. Anthropic released some data recently that said people who used AI and understand what they're doing / ask calrifying questions don't actively degrade neurons, so I'm going to do that. There's really no point in doing any of this unless I'm learning. 

---


