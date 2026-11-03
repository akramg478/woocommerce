# SQL query patterns

Use for WooCommerce query generation and review. These are conditional optimization candidates, not automatic rewrites. For indexes, large writes and benchmarking, also read [database operations](database-operations.md).

## Establish the question before changing its SQL

Record the eligible entities and the unit being counted: orders, parent products, variations, terms, or metadata rows. Check publication, visibility, stock, custom statuses/types, NULLs, duplicate metadata, missing rows, ordering, pagination, and result types. HPOS and posts storage can require different plans and predicates.

A count must describe the same population as its corresponding ID selector. Two counts added together may intentionally count an overlapping row twice; one ordered CASE counts it at most once. Moving a branch's condition into a shared WHERE can remove rows another branch previously included. Exercise a truth table of the old and new predicates before pursuing small per-row CPU savings. The [#69360 review](https://github.com/woocommerce/woocommerce/pull/69360#discussion_r4445338840) supplies a counterexample to a proposed CASE simplification; the proposal alone is not an equivalence proof.

SQL equality also depends on collation and coercion. Case-insensitive comparisons can merge distinct custom status strings. A row count equal to the number of children does not prove each child is represented when metadata allows duplicates. For exact agreement, establish identity coverage and comparison semantics; a bounded ID/value read followed by strict PHP comparison may be clearer. The [child-stock review](https://github.com/woocommerce/woocommerce/pull/68546#discussion_r4445639644) illustrates these identity/collation questions; its proposed rewrite is not a general equivalence proof.

Moving grouping/deduplication between SQL and PHP also changes equality and key normalization. [#66853](https://github.com/woocommerce/woocommerce/pull/66853) repaired batched variation-attribute keys using the existing WordPress canonicalization, not the SQL LOWER described in its stale body. Validate filter results before introducing stricter array operations, while preserving the established return contract: [#66810](https://github.com/woocommerce/woocommerce/pull/66810) restored legacy pass-through rather than silently converting malformed values to an empty array.

## Preserve identity unions and key/value pairing

A known account ID does not subsume guest purchases by billing email, even when the email matches that account. Preserve every identity the API promises. A boolean existence check can stop after a positive branch; a negative branch must fall through. A cached collection or count needs the complete qualifying union, with overlap deduplicated where required. The later [guest-order correction #65517](https://github.com/woocommerce/woocommerce/pull/65517) qualifies the earlier optimization in [#63995](https://github.com/woocommerce/woocommerce/pull/63995).

When keys have different value domains, keep their pairs together:

```sql
WHERE (meta_key = '_customer_user' AND meta_value = '12')
   OR (meta_key = '_billing_email' AND meta_value = 'buyer@example.test')
```

Independent key and value IN lists also match unintended cross-pairs. Pairing restores the intended predicate; it does not guarantee a faster plan when both branches remain necessary. Likewise, OR across indexed columns can use index merge or another plan: neither textual predicate order nor a resolved identity guarantees a particular access method.

## Remove joins only after proving their contribution is redundant

A posts join may enforce existence, type, publication, or other eligibility even if SELECT only uses its ID. Remove it only when the final incoming ID set already guarantees every such condition, including after filters and cache invalidation. The fact that IDs originated from a WP_Query is not sufficient by itself.

[#65141](https://github.com/woocommerce/woocommerce/pull/65141) removes a posts join from counts over already-filtered product IDs. That does not authorize dropping publication checks from unfiltered lookup queries. Compare actual selectivity and cardinality; neither FROM order nor the name of the smallest table dictates the optimizer's driving table.

When a framework filter is expensive over the whole store, first derive the smaller valid candidate set, then keep the framework's eligibility checks. In [#66786](https://github.com/woocommerce/woocommerce/pull/66786#discussion_r3614242300), the query narrowed role checks to token authors instead of enumerating all users with a role. Eligibility still precedes pagination so pages and totals stay correct. The same query count can do much less work; do not bypass role/site checks or mistake this for a universal constant-time plan.

## Express membership without multiplying rows

When related rows only establish eligibility, compare an IN/EXISTS form with the current join. Joining several one-to-many relations can multiply intermediate rows before DISTINCT or GROUP BY removes duplicates. A membership predicate expresses that duplicates in the qualifying relation need not multiply the result.

Preserve unmatched-row behavior: replacing a LEFT JOIN with membership loses unmatched rows unless existing predicates already require a match. Preserve taxonomy transformation, child expansion, empty-list behavior and extension hooks when replacing WP_Tax_Query-generated SQL. If related columns or multiplicity are needed, retain the appropriate join.

The engine chooses the semijoin strategy. IN does not force materialization or guarantee linear work. In MySQL EXPLAIN, `Start temporary`/`End temporary` describe Duplicate Weedout; `MATERIALIZED`/`<subqueryN>` indicate a materialized subquery. Verify the selected plan. [MySQL semijoin documentation](https://dev.mysql.com/doc/refman/8.4/en/semijoins-antijoins.html).

The [#65119](https://github.com/woocommerce/woocommerce/pull/65119) taxonomy rewrite improved dense matching sets but regressed its sparse 50K-product sample. Test relationship density as well as table size. [#69321](https://github.com/woocommerce/woocommerce/pull/69321#discussion_r4193316278) illustrates proposing membership before forcing join order; neither shape is universally faster.

## Consolidate repeated work over the same bounded population

Compare GROUP BY or conditional aggregation with repeated queries when they scan substantially the same population. Keep requested status/term predicates, zero buckets, original result types, and COUNT versus COUNT DISTINCT semantics. Do not replace several selective point lookups with an unrestricted full-table aggregation merely to reduce statement count.

In [#69320](https://github.com/woocommerce/woocommerce/pull/69320), conditional DISTINCT counts preserve duplicate-metadata behavior and custom status keys. An empty status list remains a valid filter result and returns before SQL construction. The [review accepted that guard](https://github.com/woocommerce/woocommerce/pull/69320#discussion_r4337958438). Test empty, missing, duplicate and custom values rather than only ordinary populated fixtures.

Remove ORDER BY when the caller needs only existence or an unordered set and ordering does not determine membership through LIMIT/OFFSET or another consumer. A top-N set still needs its ordering to choose the rows. Remove grouping only when it cannot affect multiplicity or aggregate meaning after all query filters run. Preserve pagination offsets. [#65413](https://github.com/woocommerce/woocommerce/pull/65413) targets a narrow primary-key/no-join case, not arbitrary single-table GROUP BY; [#70121](https://github.com/woocommerce/woocommerce/pull/70121) removes unneeded sorting from IDs used as a set.

## Keep specialized fast paths narrow and bounded

A per-status UNION can use a `(type,status,date)` index for each fixed-status branch and merge a small ordered page. That does not make it useful for full counts, deep offsets, arbitrary filters or all database setups. In [#65663](https://github.com/woocommerce/woocommerce/pull/65663#pullrequestreview-4478443894), a large Vitess store benefited while ordinary setups were slightly slower. The accepted implementation validates query shape, caps branch/page work, preserves externally changed SQL and retains the original fallback. Its size threshold is a local policy, not a universal crossover point. Include the cost of deciding whether to use the fast path.

Compare the **executed** SQL after framework hooks, not just the generated string. In [#66111](https://github.com/woocommerce/woocommerce/pull/66111#discussion_r3543439014), a request filter accidentally disabled WordPress's ID-only split query and fetched full rows. The fix preserved the framework optimization; the overall PR later closed unmerged. Restrict textual SQL edits to the intended clause so literals and extension-supplied syntax are not rewritten accidentally.

## Use lookup tables when their meaning and freshness fit

WooCommerce's product lookup tables can avoid expensive metadata access, but they are derived data:

- `wc_product_meta_lookup` supplies product/variation scalars. It does not carry post status or type; retain the necessary posts predicates or a proven eligible ID set.
- `wc_product_attributes_lookup` includes taxonomy, parent/variation identity and stock information. Choose the correct identity for counts and preserve DISTINCT/visibility behavior. It is not automatically faster than term relationships.
- Check the existing lifecycle: product-meta regeneration is indicated by `woocommerce_product_lookup_table_is_generating`; attribute lookup use requires `woocommerce_attribute_lookup_enabled === 'yes'`. These are different conditions. Deferred updates or out-of-band imports can still make an enabled lookup stale.
- Reuse the area's established fallback and acceptable freshness. Regeneration code must read authoritative source data, not the derived rows it is rebuilding.
- For fields used by the fast path, compare normal-save and full-regeneration derivations and test their intended agreement. A completed rebuild does not prove equivalent meaning: [#43011](https://github.com/woocommerce/woocommerce/pull/43011) corrected rebuilt sale flags that omitted the regular-price comparison.

The [#63043](https://github.com/woocommerce/woocommerce/pull/63043) downloadable lookup serves an editor capability heuristic; its omission of publish/type predicates is not a general equivalence proof. [#65141](https://github.com/woocommerce/woocommerce/pull/65141) retained term relationships for one attribute-count path after comparing alternatives. Read the current schema in `WC_Install` and the relevant lookup data store/regenerator instead of copying a static index catalogue.

## Compare MIN/MAX with ordered endpoint probes

First inspect the existing plan: the engine may already optimize MIN/MAX through an index. If it scans many rows, test two ordered LIMIT 1 probes while retaining all eligibility predicates and aggregate result shape:

```sql
SELECT MIN(date_created_gmt), MAX(date_created_gmt)
FROM (
    (SELECT date_created_gmt FROM wp_wc_orders
     WHERE type = 'shop_order' AND status != 'trash'
       AND date_created_gmt IS NOT NULL
     ORDER BY date_created_gmt ASC LIMIT 1)
    UNION ALL
    (SELECT date_created_gmt FROM wp_wc_orders
     WHERE type = 'shop_order' AND status != 'trash'
       AND date_created_gmt IS NOT NULL
     ORDER BY date_created_gmt DESC LIMIT 1)
) AS bounds;
```

Table names here are illustrative; use the runtime table name. NULL guards preserve MIN/MAX's non-NULL semantics; the outer aggregate preserves one NULL-valued result row on empty/all-NULL input. Backdated orders mean ID order cannot substitute for date order.

A composite `(type,status,date)` index is not globally date-ordered when status varies. In [#63039](https://github.com/woocommerce/woocommerce/pull/63039), the probes selected the separate `date_created` index and applied residual filters. Two returned dates do not mean two examined rows: a long run of ineligible records can make a probe expensive. Do not add an index automatically to enable this rewrite; evaluate its operational cost separately. [MySQL ORDER BY optimization](https://dev.mysql.com/doc/refman/8.4/en/order-by-optimization.html).

## Construct new fragments safely without re-preparing existing SQL

Guard empty dynamic lists using the caller's established empty-result semantics. Normalize integer IDs and use `$wpdb->prepare()` placeholders for values; escaping alone is not a substitute for value placeholders. Prepare only newly introduced fragments when incorporating SQL that has already been prepared: preparing the whole statement again can reinterpret literal percent syntax. The [#69320 preparation discussion](https://github.com/woocommerce/woocommerce/pull/69320#discussion_r4316217493) includes a literal `50%s` search regression case.

Preserve supported hooks, data-store overrides, and filterable query paths. A fast path that stops calling an override changes behavior even if signatures remain intact. When skipping the old path cannot preserve those contracts, retain a compatible fallback or design the change as an explicit compatibility migration.
