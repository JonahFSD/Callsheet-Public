# Engineering decisions

[Overview](../README.md) · [Architecture](architecture.md) · [Workflows](workflows.md) · [Validation](validation.md)

These notes explain choices visible in the implementation. Tradeoff analysis is retrospective; it is not presented as historical architecture decision records.

## Keep workflows in services

HTTP handlers form the entry boundary, while services coordinate contacts, calendars, estimates, messaging and billing. Repository and provider interfaces give much of that work explicit dependencies.

This permits tests of a billing or customer operation with a substitute rather than a real payment or SMS call. The cost is additional interfaces and dependency wiring. Some concrete integration dependencies remain; the documentation describes those instead of claiming complete interchangeability.

## Use deterministic ranking

The priority calculation uses business signals the product already collects. A call outcome, service interval or viewed estimate can affect the next suggested action without a model inference request.

This makes the relationship between inputs and ranking tractable and supports focused scoring tests. Hand-built rules require maintenance and do not establish that a higher score causes better customer outcomes. This is a heuristic, not a calibrated prediction system.

## Persist before contacting the provider

Persisting the draft separates the estimate from successful SMS submission. A failure leaves an object to inspect instead of making the attempted estimate disappear.

The database and provider are separate systems. A persisted draft, queued SMS and recipient acceptance are distinct facts. The state diagram makes those differences visible; it does not itself prove a comprehensive duplicate-delivery or crash-recovery guarantee.

## Evaluate expiry on access

The estimate service checks expiry during reads and acceptance, persisting expired status when detected. The documented path does not require a scheduled expiry worker.

Stored status may therefore not advance until an access triggers the check, and a read can cause a write. Reporting or background-processing needs might justify a separate reconciliation mechanism later; this page does not imply one is already implemented.

## Carry account scope through operations

A team member's identity and business account are distinct concepts. Repository calls carry account scope, while role checks constrain operations within that account. Purpose-specific public estimate access is separate.

This supports multiple businesses in a shared application. The engineering obligation is verification at request boundaries, including attempts to reference another account's records. Existing tests cover several such scenarios; [validation](validation.md) explains the evidence limits.

## Make localization part of the workflow

English and Spanish resources live alongside the client, applying translation to the daily product flow. The maintenance cost is keeping labels, errors and changing workflow text aligned in both languages. Locale resources establish implementation scope, not proof that every string received professional translation review.
