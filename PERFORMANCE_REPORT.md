# Performance & Scalability Report

## Observed Hotspots
- Supplier and order endpoints perform Python-side aggregation over large item sets.
- Multiple list APIs appear unpaginated, risking large payloads.
- Feed and dashboard flows may produce high query volume under growth.

## Query Efficiency Notes
- Positive: some endpoints use `select_related` / `prefetch_related`.
- Gap: ownership and pagination filters are inconsistent, amplifying result cardinality.
- Gap: aggregation should be shifted from Python loops to DB annotations/materialized daily facts.

## Recommended Performance Actions
1. Global DRF pagination + explicit ordering indexes on high-cardinality tables.
2. Query instrumentation (django-debug-toolbar in dev, query count assertions in tests).
3. Cache read-mostly dashboards with short TTL and invalidation hooks.
4. Add async workers for heavy non-request-critical tasks.

## Estimated Impact
- P95 list endpoint latency reduction: 25-50% after pagination/aggregation fixes.
- DB load reduction: 20-40% with query consolidation and caching.
