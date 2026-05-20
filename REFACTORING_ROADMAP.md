# Refactoring Roadmap

## Phase 1 (0-2 weeks)
1. Enforce tenant-aware querysets and object permissions for Shop/Catalog/Inventory/Account detail endpoints.
2. Introduce centralized order/payment transition policy module.
3. Add pagination defaults and response limits globally.

## Phase 2 (2-6 weeks)
1. Extract use-case services from fat views (`order`, `payment`, `supliers`, `marketer`).
2. Introduce transactional outbox for notifications/payment side effects.
3. Normalize settings into `base/dev/prod` modules and env contracts.

## Phase 3 (6-12 weeks)
1. Split payment service into subdomains (initiation, reconciliation, payout, earnings).
2. Add selector/repository layer for read-heavy dashboards and feed APIs.
3. Rename `supliers` app and align naming conventions platform-wide.

## Phase 4 (ongoing)
1. Add architecture tests (permissions, state machines, idempotency).
2. Add performance budgets (query count + latency SLAs) in CI.
3. Establish ADR process for cross-domain changes.
