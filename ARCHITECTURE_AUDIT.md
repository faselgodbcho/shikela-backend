# Shikela Backend Architecture Audit

## Executive Summary
- Architecture score: **41/100**
- Maintainability score: **38/100**
- Scalability score: **43/100**
- Performance score: **46/100**
- Technical debt score: **72/100** (higher is worse)

The codebase delivers core ecommerce flows but shows principal-level concerns around domain coupling, multitenant boundaries, lifecycle-state governance, and unbounded data access patterns.

## Findings
### ARC-001 — Cross-app domain leakage and weak bounded contexts
- Severity: **High**
- Type: **Architecture Problem**
- Affected files: `order/services.py, payment/services/service.py, analytics/services.py, notifications/services.py`
- Problematic snippet: `OrderService.create_order(...) triggers notifications and payment side effects indirectly; PaymentService mutates Order/Inventory/Earnings.`
- Why problematic: Core order, payment, analytics, and notification domains are tightly coupled via direct model/service calls, creating ripple effects and high change risk.
- Real-world consequences: Small payment-flow changes can break order/inventory logic, reducing release safety and team parallelism.
- Recommended refactor: Introduce domain events (OrderCreated, PaymentSettled, OrderDelivered) and asynchronous handlers; isolate write models per app boundary.
- Example improved implementation: Publish event in order app; subscribers in payment/analytics/notifications consume event with idempotency keys.
- Estimated impact of fixing: High: reduces regression blast radius and enables scale-out ownership.

### ARC-002 — Fat API views containing orchestration/business rules
- Severity: **High**
- Type: **Refactor Opportunity**
- Affected files: `payment/views.py, order/views.py, supliers/views.py, marketer/views.py`
- Problematic snippet: `APIView methods perform validation, querying, state transitions, and response shaping in one layer.`
- Why problematic: Violates separation of concerns; hard to test deterministically; duplicates validation logic between views/services/serializers.
- Real-world consequences: Higher defect rate, inconsistent behavior across endpoints, expensive maintenance.
- Recommended refactor: Adopt use-case/application service layer + thin DRF views; serializers for I/O only, services for workflows.
- Example improved implementation: class CreateOrderUseCase.execute(command) -> DTO; view only maps request/response.
- Estimated impact of fixing: High: testability and maintainability improve materially.

### ARC-003 — Multitenancy access model is inconsistent across apps
- Severity: **Critical**
- Type: **Logic Error**
- Affected files: `shop/views.py, catalog/views.py, inventory/views.py, account/views.py`
- Problematic snippet: `queryset = Shop.objects.all(); queryset = Product.objects.all(); queryset = Inventory.objects.all()`
- Why problematic: Object-level ownership filters are missing/uneven, while platform is multi-vendor.
- Real-world consequences: Cross-tenant reads/writes and data corruption risk.
- Recommended refactor: Introduce shared tenant-aware mixins/permissions and mandatory get_queryset ownership filters.
- Example improved implementation: class OwnerScopedQuerysetMixin: def get_queryset(self): return super().get_queryset().filter(shop__owner=self.request.user)
- Estimated impact of fixing: Critical: protects core data integrity and compliance.

### ARC-004 — State machine rules distributed and partially duplicated
- Severity: **High**
- Type: **Maintainability Issue**
- Affected files: `order/services.py, payment/services/service.py, payment/views.py, payment/signals.py`
- Problematic snippet: `Order status and payment status transitions are enforced in multiple places with partially overlapping checks.`
- Why problematic: Transition invariants are not centralized in a single state machine.
- Real-world consequences: Impossible states and drift during feature additions (refund/cancel/deliver race paths).
- Recommended refactor: Create explicit transition policy module with allowed transitions + guard conditions and audit logging.
- Example improved implementation: OrderStateMachine.transition(order, target, actor, context)
- Estimated impact of fixing: High: fewer lifecycle bugs in payments/orders.

### ARC-005 — Potential N+1 and heavy in-memory aggregation in dashboards
- Severity: **Medium**
- Type: **Performance Issue**
- Affected files: `supliers/views.py, order/views.py, hub/views.py`
- Problematic snippet: `Looping through items/orders and computing totals in Python after broad queryset materialization.`
- Why problematic: Loads large datasets into memory; repeated related-object accesses can explode query count under scale.
- Real-world consequences: Slow dashboards and higher DB/API latency at production volume.
- Recommended refactor: Push aggregation to DB using annotate/Sum/Case, paginate result sets, and precompute daily facts.
- Example improved implementation: OrderItem.objects.filter(...).values(...).annotate(total=Sum(F('price')*F('quantity')))
- Estimated impact of fixing: Medium-high performance gain at scale.

### ARC-006 — Repository-level configuration drift and environment mismatch
- Severity: **Medium**
- Type: **Scalability Risk**
- Affected files: `core/settings.py, requirements.txt`
- Problematic snippet: `SQLite default in settings while platform context implies PostgreSQL production; requirements encoding malformed.`
- Why problematic: Local/prod behavior diverges; tooling (linters/scanners) can fail on malformed dependency file.
- Real-world consequences: Hard-to-reproduce bugs and unstable CI/CD quality gates.
- Recommended refactor: Use env-driven DB config with explicit prod profiles; normalize requirements to UTF-8 and lock with hashes.
- Example improved implementation: DATABASE_URL parsing + separate settings modules (base/dev/prod).
- Estimated impact of fixing: Medium: improved deployment reliability.

### ARC-007 — Service naming and module layout inconsistency
- Severity: **Low**
- Type: **Maintainability Issue**
- Affected files: `supliers/*, payment/services/service.py, catalog/services.py`
- Problematic snippet: `Misspelled app name `supliers`, generic file names (`service.py`), mixed conventions.`
- Why problematic: Inconsistent naming impairs discoverability and onboarding.
- Real-world consequences: Higher cognitive load and accidental import mistakes.
- Recommended refactor: Rename modules to intent-revealing names (payout_service.py, order_checkout_service.py) and fix app naming.
- Example improved implementation: payment/services/payouts.py, payment/services/reconciliation.py
- Estimated impact of fixing: Low-medium ongoing productivity benefit.

### ARC-008 — Transaction boundaries not uniformly aligned with side effects
- Severity: **High**
- Type: **Logic Error**
- Affected files: `order/services.py, payment/views.py, payment/signals.py, notifications/services.py`
- Problematic snippet: `DB writes and external effects (payment gateway, notifications) can occur without outbox pattern.`
- Why problematic: Failure between DB commit and external call causes inconsistent external/internal state.
- Real-world consequences: Ghost notifications, duplicate payouts, reconciliation mismatches.
- Recommended refactor: Use transactional outbox + worker processing for external IO; ensure idempotent consumers.
- Example improved implementation: Store outbox_event within atomic transaction, Celery worker dispatches reliably.
- Estimated impact of fixing: High operational reliability uplift.

### ARC-009 — Serializer responsibilities are mixed with domain defaults
- Severity: **Medium**
- Type: **Refactor Opportunity**
- Affected files: `account/serializers.py, shop/serializers.py, payment/serializers.py`
- Problematic snippet: `create() methods assign roles and defaults; validation duplicates model/business invariants.`
- Why problematic: Domain rules in serializers complicate reuse outside HTTP and create hidden coupling.
- Real-world consequences: Inconsistent behavior in management commands, tests, and future async workflows.
- Recommended refactor: Move domain invariants to services/model methods; keep serializer for schema validation only.
- Example improved implementation: UserRegistrationService.register_supplier(payload)
- Estimated impact of fixing: Medium maintainability/testability gain.

### ARC-010 — Insufficient pagination/filter standards across list endpoints
- Severity: **Medium**
- Type: **Scalability Risk**
- Affected files: `catalog/views.py, notifications/views.py, marketer/views.py, hub/views.py`
- Problematic snippet: `Several list endpoints return full result sets without pagination controls.`
- Why problematic: Unbounded responses increase DB load, memory, and network costs.
- Real-world consequences: Latency spikes and API instability under high cardinality datasets.
- Recommended refactor: Adopt global pagination policy + filter backends + max page size.
- Example improved implementation: REST_FRAMEWORK['DEFAULT_PAGINATION_CLASS']='rest_framework.pagination.PageNumberPagination'
- Estimated impact of fixing: Medium-high at growth stage.
