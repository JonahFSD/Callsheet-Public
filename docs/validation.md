# Validation and status

[Overview](../README.md) · [Architecture](architecture.md) · [Workflows](workflows.md) · [Decisions](engineering-decisions.md)

Documentation snapshot: **September 21, 2026**. Inspected private implementation revision: `318f3cf37b1b2736dcd65b445a770aa572eb00bc`.

## Evidence

| Claim | Basis | What it establishes |
|---|---|---|
| Co-built by Jonah Elliott and Evan Pursley | Existing public attribution and implementation history | Collaborative product authorship |
| Served two paying customers | Cofounder Jonah Elliott's confirmation | Historical self-reported outcome; no public receipts or revenue audit |
| React/FastAPI with provider integrations | Source inspection | Implemented components and integration paths |
| Deterministic follow-up scoring | Scoring implementation and test sources | Rule-based ranking, not a learned model |
| Estimate lifecycle and failure result | Estimate service inspection | Draft/submission/view/acceptance behavior |
| Account and role boundaries | Repository conventions and authorization tests | Mechanisms and test scenarios in source, not blanket certification |

The public documentation is independently inspectable. The private application and tests cannot be reproduced from this repository, and code-review access is not promised here.

## Test topics present

- Account isolation and role-restricted operations.
- Priority scoring and boundary inputs.
- Subscription status, checkout and portal operations with provider substitutes.
- Billing event verification behavior.
- Message drafting, send-result handling and conversation retrieval.

Some tests are skipped; some exercise mocks or permissive success/failure expectations. A test topic does not imply complete coverage. The suite was **not rerun for this documentation update**; no fresh all-green product result is asserted here.

## Product limits

The two-customer outcome is historical. Current subscriptions, revenue, active usage and production deployment were not verified for these pages.

Calendar integration exists, but automatic event creation on estimate acceptance remains unfinished in the inspected service. Provider SMS acceptance is not handset delivery. The ranking has not been represented as a calibrated prediction model.

A historical automated security scan exists. It does not amount to an independent penetration test, certification or comprehensive security assurance.

## Documentation maintenance

When the private implementation changes, review the affected workflow, redraw its diagram and update the inspected revision and date. Recheck numerical claims separately from code changes. New examples or screenshots should use synthetic data and be labeled accordingly.

These pages are freshly authored explanatory material. Application code, credentials, customer records and operational configuration are outside publication scope. The existing repository license covers the material included here; it does not grant access to or license the separate private application.
