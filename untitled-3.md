---
type: Note
_owner_id: 7f1940e1-74d1-4585-b2a5-5d7f4df09e09
---
# ADR-001: Event Sourcing for Inventory Context

**Status:** Proposed

## Context

Inventory is the single most critical domain in Ginesys. Stock position across locations is the #1 reason customers choose Ginesys over competitors. Every stock movement -- receipt, transfer, sale, return, adjustment, audit variance -- must be fully traceable. Stock position is not a static value; it is the sum of every movement that has ever occurred.

The current system treats stock as a mutable balance that gets updated in place. When a stock audit finds a variance, there is no reliable way to reconstruct what happened. During peak season (Diwali, EOFY), high write throughput on stock tables causes locking contention.

The question is whether to use Event Sourcing (events as the source of truth, state derived by replay) or simpler CRUD with an audit log (update-in-place with an outbox record of what changed). Event Sourcing adds significant complexity: event store schema, snapshot management, projection handlers, and event versioning. A team that has never worked with ES will need training.

However, several contexts have immutable-data semantics baked into their domain. A stock position *is* a projection. A retail transaction *cannot* be edited after completion. A journal entry *must never* be altered by law. POS-specific event-sourcing constraints are now covered by [ADR-021](./ADR-021-event-sourcing-pos.md); this ADR remains the binding decision only for Inventory.

## Decision

We adopt Event Sourcing with CQRS for the Inventory Context. Events are appended to an `event_store` table within each tenant's PostgreSQL schema. The current stock position is a projection maintained by event handlers, cached in Redis for sub-100ms reads. Snapshots are taken every 100 events per aggregate to avoid replaying long event streams.

CRUD with outbox audit is used for simpler contexts (Catalog, Procurement, Production, Customer, Pricing) where writes are infrequent and the domain does not have immutable-data semantics.

## Alternatives Considered

- **CRUD + audit log for everything:** Simpler to implement but cannot reconstruct stock position from first principles. Audit logs record *that* something changed but not in a replayable structure. Would require a separate reconciliation engine anyway.
- **Event Sourcing for all contexts:** Over-engineering for contexts like Catalog where writes are rare and the domain is simple master data.

## Consequences

**Positive:**

- Full traceability of every stock movement. Any stock position can be reconstructed by replaying events.
- High write throughput without locking -- appends to event store are never blocked by reads.
- Natural fit for the domain: stock position IS a derived projection in retail.
- Enables time-travel queries ("what was the stock position on March 15?") without additional infrastructure.

**Negative:**

- ES expertise required. Teams must understand event store design, snapshot strategies, projection idempotency, and event versioning.
- Event schema evolution is harder than table schema migration. Events written in version N must be readable by code in version N+50.
- Read model latency: projections are eventually consistent. Stock position in Redis may be up to 1 second behind the event store. This is acceptable for operational queries but must be clearly documented.
- Debugging is harder. You cannot "look at the stock position table" to understand state -- you must understand the projection logic.

**Irreversibility:** Once Inventory is event-sourced, reverting to CRUD requires rebuilding the current-state tables and losing the ability to replay history. This decision is effectively irrevocable.
