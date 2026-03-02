# Stop forgetting your best customers.

CallSheet is a CRM for 1-3 truck service contractors that ranks customers by who needs attention most. Open the app, see the list, tap to call. That's it.

[callsheet.cv](https://callsheet.cv)

---

## What It Does

**Priority Scoring** -- AI ranks every customer by urgency and value. The one who needs you most is always first.

**One-Tap Calls** -- Call, log the outcome, schedule the next follow-up. One flow.

**SMS Estimates** -- Create a quote, tap send. Customer gets a branded estimate page via text.

**Google Calendar Sync** -- Schedule a callback and it lands on your calendar automatically.

**Performance Dashboard** -- Track call volume, conversion rates, and revenue booked.

**Team Management** -- Invite your crew, assign roles, see who called whom.

Also includes CSV import, customer notes, search and filtering, dark mode, and English/Spanish localization.

---

## Progress

- Live product with paying customers
- 27K+ lines of production code across 178 commits
- First commit: January 17, 2026
- Open-source contributor to Gymnasium (Farama Foundation) and browser-use
- 1517 Fund community member

---

## Tech Stack

| Layer | Tech |
|-------|------|
| Frontend | React 18, Vite |
| Backend | FastAPI, Python 3.11 |
| Database | PostgreSQL (prod), SQLite (dev) |
| Auth | Clerk (JWT/JWKS) |
| Payments | Stripe |
| SMS | Twilio |
| Calendar | Google Calendar API |
| Analytics | PostHog |
| Email | Resend |
| Hosting | Vercel (frontend), Railway (backend) |

---

## Architecture

```
backend/app/
├── adapters/
│   ├── driving/api/       → Route handlers (thin: parse → service → respond)
│   └── driven/            → Concrete integrations
│       ├── database/      → SQLAlchemy ORM, repositories
│       ├── stripe/        → Payment provider
│       ├── google/        → Calendar + Contacts OAuth
│       ├── twilio/        → SMS delivery
│       ├── clerk/         → Auth provider
│       └── email/         → Transactional email
├── domain/                → Business entities & exceptions
├── ports/                 → Abstract interfaces (repository contracts)
└── services/              → All business logic (constructor-injected deps)
```

Hexagonal architecture with dependency injection via FastAPI's `Depends()`. Routes are thin -- they parse input, call a service, and map to HTTP responses. All business logic lives in services. DB access goes through repository ports. External integrations are swappable adapters. Multi-tenant -- every query scopes to `account_id`.

---

## AI Development Workflow

CallSheet is developed using an autonomous AI-driven workflow. Task specifications are written as structured prompts (context, goal, files to modify, specific changes, constraints, verification steps). A bash harness spawns a fresh AI coding agent for each task, monitors progress via a shared PLAN.md checklist, and kills the process on completion to prevent context degradation. A second agent automatically audits every batch of changes against project conventions and task specs before anything gets committed.

The system enforces constraints at the harness level, not the prompt level -- a fake git wrapper blocks all write operations, a process monitor prevents scope creep, and the filesystem serves as shared memory between iterations. This lets one person produce the engineering throughput of a small team with consistent code quality.

---

## Scoring Algorithm

Customers are scored 0-100 based on four weighted factors: revenue value (contract size and service history), service urgency (time since last contact, overdue thresholds), customer risk (contract expiration, churn signals), and outreach opportunity (seasonal patterns, estimate follow-ups). Scores map to red/amber/green urgency buckets with specific action recommendations.

---

## Running Locally

```bash
# Full stack (Docker)
make dev                # Starts postgres, redis, backend, frontend
make dev-down           # Stop everything

# Without Docker
cd backend && pip install -r requirements.txt && uvicorn main:app --reload
cd frontend && npm install && npm run dev
```

Frontend: `localhost:5173` | Backend: `localhost:8000` | API docs: `localhost:8000/docs`

---

Built by Jonah Elliott and Evan Pursley.
