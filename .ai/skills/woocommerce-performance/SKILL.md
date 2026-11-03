---
name: woocommerce-performance
description: Use when writing or reviewing WooCommerce performance changes involving SQL, indexes, bulk data loading, cache priming or invalidation, maintenance jobs, repeated computation, or deferred initialization.
---

# WooCommerce Performance

Optimize the work required by the caller while preserving its data and extension contracts. A smaller query count, a familiar SQL shape, or a merged precedent is not proof of an improvement on another store.

Read only the references relevant to the change:

| Work | Reference |
| --- | --- |
| SQL predicates, joins, aggregates, lookup tables, query plans | [SQL query patterns](sql-query-patterns.md) |
| Indexes, large writes, cleanup, concurrency, profiling or deferred work | [Database operations and measurement](database-operations.md) |
| Product/order collections, images, N+1 hydration | [Cache priming](cache-priming.md) |
| Options, transients, autoload state | [Options cache priming](options-cache-priming.md) |

For SQL changes, establish the result contract before the plan: eligible entities, duplicates, ordering, NULLs, result types, storage authority, and extension filters. For cache changes, also establish the key, lifetime, mutation paths, and acceptable staleness. Check the current implementation and schema rather than copying a historical PR verbatim.

In the change description, identify the workload improved, evidence supporting that conclusion, compatibility preserved, and untested conditions. Use the project's testing and verification workflows; these references add performance-specific checks, not a separate approval process.
