# Technical Architecture

Deep dive into CallSheet's system design, patterns, and technical decisions.

---

## System Overview

CallSheet is a multi-tenant SaaS application. Each "account" is a separate business (plumber, HVAC company, etc.) with complete data isolation.

```
┌─────────────────────────────────────────────────────────────────┐
│                         DRIVING ADAPTERS                        │
│  (Things that initiate requests into our system)                │
├─────────────────────────────────────────────────────────────────┤
│  FastAPI Routes    │  Clerk Webhooks    │  Stripe Webhooks      │
│  /api/today        │  user.created      │  checkout.completed   │
│  /api/estimates    │  user.updated      │  subscription.updated │
│  /api/customers    │  user.deleted      │  invoice.failed       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        SERVICE LAYER                            │
│  (Business logic and use case orchestration)                    │
├─────────────────────────────────────────────────────────────────┤
│  ContactsService   │  EstimateService   │  BillingService       │
│  - get_today_view  │  - create_and_send │  - create_checkout    │
│  - record_action   │  - accept_estimate │  - handle_webhook     │
│  - import_csv      │  - check_expiration│  - get_status         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                           PORTS                                 │
│  (Abstract interfaces - what the services need)                 │
├─────────────────────────────────────────────────────────────────┤
│  ContactsRepository    │  PaymentProvider    │  SMSProvider     │
│  TimelineRepository    │  CalendarProvider   │  EmailProvider   │
│  EstimateRepository    │                     │                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                       DRIVEN ADAPTERS                           │
│  (Concrete implementations of ports)                            │
├─────────────────────────────────────────────────────────────────┤
│  PostgreSQL repos  │  StripeProvider    │  TwilioAdapter       │
│  (SQLAlchemy)      │  GoogleCalendar    │  ResendEmail         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Multi-Tenant Data Model

### Entity Relationship Diagram

```
accounts (tenant root)
    │
    ├── users (team members)
    │       └── user_credentials (Google OAuth tokens, 1:1)
    │
    ├── customers (CRM contacts)
    │       ├── timeline_events (activity history)
    │       ├── user_tasks (scheduled follow-ups)
    │       ├── notes (with pinning)
    │       └── estimates (SMS estimates)
    │
    ├── sync_logs (Google import history)
    ├── audit_logs (security trail)
    └── analytics_events (product usage)
```

### Tenant Isolation Strategy

**Layer 1 - Authentication:**
Every request must have a valid JWT. The JWT contains `sub` (Clerk user ID), which maps to a User row, which has `account_id`.

**Layer 2 - Repository Pattern:**
Every database query includes `account_id` in the WHERE clause:

```python
# This pattern is in every repository method
def get_customers(self, account_id: int) -> List[Customer]:
    return self.db.query(Customer).filter(
        Customer.account_id == account_id,
        Customer.deleted_at.is_(None)
    ).all()
```

**Layer 3 - Database Constraints:**
- `account_id` is NOT NULL on all tenant tables
- Foreign key constraints ensure referential integrity
- No RLS (application-enforced isolation only)

**Exception - Public Estimate Access:**
The `get_estimate_by_token()` method does NOT filter by account_id. This is intentional - customers viewing estimates aren't authenticated. The token itself (48-byte cryptographic random) is the authorization.

---

## Authentication Deep Dive

### JWT Verification Flow

```python
def verify_clerk_token(token: str) -> dict:
    # 1. Get JWKS client (cached, auto-refreshes keys)
    jwks_client = get_jwks_client()

    # 2. Extract signing key from JWT header
    signing_key = jwks_client.get_signing_key_from_jwt(token)

    # 3. Verify signature and decode claims
    claims = jwt.decode(
        token,
        signing_key.key,
        algorithms=["RS256"],
        options={
            "verify_exp": True,
            "verify_iat": True,
            "require": ["sub", "exp", "iat"],
        }
    )

    return claims  # Contains 'sub' = clerk_user_id
```

### JWKS URL Discovery

The Clerk publishable key encodes the JWKS domain:

```
pk_test_<base64-encoded-domain>$
         ↓
Decode → clerk-abc123.clerk.accounts.dev
         ↓
JWKS URL: https://clerk-abc123.clerk.accounts.dev/.well-known/jwks.json
```

For custom domains, a proxy URL overrides this.

### Webhook Fallback Pattern

Problem: Clerk webhook fires on signup, creates User+Account. But webhooks can fail (network, deploy timing, outages).

Solution: Lazy user creation in the auth dependency:

```python
async def get_current_user(token, db):
    claims = verify_clerk_token(token)
    clerk_user_id = claims["sub"]

    user = db.query(User).filter(User.clerk_id == clerk_user_id).first()

    if not user:
        # Webhook failed - create user on demand
        clerk_data = await fetch_from_clerk_api(clerk_user_id)

        # Check for invited user (email exists, no clerk_id)
        existing = db.query(User).filter(User.email == clerk_data["email"]).first()
        if existing:
            existing.clerk_id = clerk_user_id  # Link accounts
            return existing

        # Create new account + user
        account = Account(name=f"{clerk_data['name']}'s Account")
        user = User(account_id=account.id, clerk_id=clerk_user_id, ...)
        db.commit()

    return user
```

---

## Priority Scoring Algorithm

### Input Metrics

For each customer, we calculate:

| Metric | Calculation |
|--------|-------------|
| `avg_invoice` | Mean of all service_completed event values |
| `days_since_last_service` | Now - most recent service date |
| `expected_interval` | Mean gap between services (default 90 days) |
| `overdue_ratio` | days_since_last_service ÷ expected_interval |
| `missed_contacts` | Call attempts without completion within 7 days |
| `gap_deviation` | Std dev of service intervals ÷ mean (irregularity) |

### Scoring Components

**Value Score (0-40 points):**
```
Invoice percentile (0-20):  Where does avg_invoice rank vs all customers?
Repeat frequency (0-10):    10+ services = 10pts, 5+ = 7, 3+ = 5, 2+ = 3
Contract status (0-10):     Active = 10, Expired = 5, None = 0
```

**Urgency Score (0-30 points):**
```
Based on overdue_ratio:
  ≥ 2.0 → 30 pts (way overdue)
  ≥ 1.5 → 25 pts
  ≥ 1.2 → 20 pts
  ≥ 1.0 → 15 pts (just due)
  ≥ 0.8 → 10 pts
  ≥ 0.5 → 5 pts
  < 0.5 → 0 pts (recently serviced)
```

**Risk Score (0-20 points):**
```
Missed contacts:        3+ = 10pts, 2 = 7, 1 = 4
Gap deviation:          High irregularity = up to 10pts
Stale sent estimate:    72+ hours without view = +15pts
Stale viewed estimate:  72+ hours without accept = +10pts
Expired estimate:       +20pts
Multiple expired:       +30pts
Accepted estimate:      -20pts (good sign, lower risk)
```

**Opportunity Score (0-10 points):**
```
Seasonal window:        Spring/Fall = 5pts, other = 2pts
Upsell potential:       Below 75th percentile invoice = 5pts
Recently viewed est:    <24h = +20pts, >24h = +10pts
Accepted estimate:      +25pts (ready to schedule)
```

### Filtering Logic

```python
# Score everyone
scored = [(customer, calculate_score(customer)) for customer in customers]

# Sort by score descending
scored.sort(key=lambda x: x[1], reverse=True)

# Filter: show score >= 50, unless fewer than 5 qualify
high_priority = [c for c in scored if c[1] >= 50]
final_list = high_priority if len(high_priority) >= 5 else scored

# Exclude customers with pending scheduled tasks
final_list = [c for c in final_list if c not in customers_with_pending_tasks]
```

---

## Estimate State Machine

### States

| State | Description | Terminal? |
|-------|-------------|-----------|
| DRAFT | Created but SMS not sent (or failed) | No |
| SENT | SMS delivered, waiting for customer | No |
| VIEWED | Customer opened the link | No |
| ACCEPTED | Customer clicked accept | Yes |
| EXPIRED | Past expires_at timestamp | Yes |

### Transitions

| From | To | Trigger | Side Effects |
|------|-----|---------|--------------|
| (new) | DRAFT | User creates estimate | Generate token, persist |
| DRAFT | SENT | Twilio SMS succeeds | Set sent_at, expires_at, sms_sid |
| SENT | VIEWED | Customer opens link | Set viewed_at |
| SENT | EXPIRED | Time check on read | (lazy) |
| VIEWED | ACCEPTED | Customer clicks accept | Set responded_at, optionally create calendar event |
| VIEWED | EXPIRED | Time check on read | (lazy) |

### Invariants

1. ACCEPTED and EXPIRED are terminal - no transitions out
2. Cannot skip states (DRAFT cannot jump to ACCEPTED)
3. Token is immutable after creation
4. expires_at is only set when SMS succeeds
5. An expired estimate cannot be accepted

### Lazy Expiration

```python
def _check_expiration(self, estimate: Estimate) -> Estimate:
    """Called on every read. Updates state if expired."""
    if estimate.status in (Status.SENT, Status.VIEWED):
        if estimate.expires_at and datetime.now(UTC) > estimate.expires_at:
            estimate.status = Status.EXPIRED
            self.db.commit()
    return estimate
```

**Tradeoff:** No background worker needed, simpler ops. But creates write-on-read, and expired estimates stay in old state until someone reads them.

---

## External Service Integration

### Adapter Pattern

Each external service is wrapped in an adapter that:
1. Handles configuration (API keys, etc.)
2. Translates between domain types and service-specific types
3. Handles errors gracefully
4. Masks sensitive data in logs

Example - Twilio adapter:

```python
class TwilioAdapter:
    @property
    def is_configured(self) -> bool:
        return bool(self.account_sid and self.auth_token and self.phone_number)

    async def send_sms(self, to: str, body: str) -> Tuple[bool, Optional[str], Optional[str]]:
        """Returns: (success, message_sid, error_message)"""
        if not self.is_configured:
            return (False, None, "Twilio not configured")

        try:
            message = self.client.messages.create(
                body=body,
                from_=self.phone_number,
                to=to
            )
            return (True, message.sid, None)
        except TwilioRestException as e:
            logger.error(f"SMS failed to ***{to[-4:]}: {e}")  # Mask phone
            return (False, None, str(e))
```

### Graceful Degradation

| Service | Failure Mode | Degradation |
|---------|--------------|-------------|
| Twilio | SMS fails | Estimate stays DRAFT, error returned, user can retry |
| Stripe | API down | Fall back to local DB subscription status |
| Clerk (webhook) | Doesn't fire | User created on-demand via API call |
| Google Calendar | API fails | Calendar events skipped, core features work |

---

## Security Considerations

### What's Implemented

- JWT verification with RS256 (asymmetric keys)
- Webhook signature verification (HMAC-SHA256)
- CORS restricted to frontend domain
- Rate limiting on all endpoints
- Audit logging (who did what, when, from where)
- Soft deletes (data recoverable)
- Encrypted Google OAuth tokens at rest (Fernet)
- Security headers (CSP, HSTS, X-Frame-Options, etc.)

### What's Missing (Tech Debt)

- No database-level RLS (tenant isolation is app-enforced only)
- No request signing between services
- No field-level encryption for PII
- No penetration testing
- Webhook handlers not idempotent (Stripe can redeliver)

---

## Performance Characteristics

### Current Bottlenecks

1. **Today View** - Fetches all customers, all events, scores everyone on every request. O(n) where n = customers per account.

2. **CSV Import** - Synchronous, single-threaded. Large files (1000+ rows) risk timeout.

3. **No caching** - Every request hits the database. Redis is in docker-compose but unused.

### What Would Break at Scale

- 10K+ customers per account: Today View becomes slow
- High concurrent reads on same estimate: Write-on-read expiration creates race conditions
- Multiple regions: Single database, no read replicas

### Scaling Path (If Needed)

1. Cache Today View results (invalidate on action)
2. Pre-compute priority scores in background
3. Add read replicas for reporting queries
4. Move CSV import to job queue
5. Replace lazy expiration with scheduled task
