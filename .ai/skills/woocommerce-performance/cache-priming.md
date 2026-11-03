# Cache priming

Use when loops hydrate products, orders, terms or images. Batch proven cache misses close to the consumer; avoid loading data that the caller will not use. Priming, caching hydrated objects, and retaining mutable instances are different optimizations.

## Follow the actual loading path

1. Identify the repeated reads and their cache layer. `_prime_post_caches()` warms WordPress post data, optionally terms and metadata; it is not an HPOS order-table loader. Use the existing WooCommerce bulk-order/data-store path for the active backend.
2. Check what the source already loads. A full-object WP_Query with default cache settings normally primes its results; ID-only queries and disabled cache flags require separate inspection. Never pass WP_Post objects to an API expecting integer IDs.
3. Reuse native batching before building another layer. For example, `get_comments()` supports `update_comment_post_cache`; inspect whether the later consumer also needs terms/meta. Existing order-item bulk/lazy loading already covers many product reads.
4. Prime the remaining IDs needed by the actual consumer. An object-cache hit can eliminate hydration entirely; prime misses after that decision. `_prime_post_caches()` itself skips cached posts, but building redundant ID lists still costs work.

```php
if ( ! empty( $product_ids ) ) {
    _prime_post_caches( $product_ids );
    $products = array_filter( array_map( 'wc_get_product', $product_ids ) );
}
```

Use integer IDs from a validated source. Empty guards clarify optional batches; the WordPress helper also short-circuits empty input. Select term/meta flags from downstream reads, not a universal minimal-fetch rule. [#54984](https://github.com/woocommerce/woocommerce/pull/54984) benefited from post-only priming; [#61057](https://github.com/woocommerce/woocommerce/pull/61057) found full term/meta priming faster for product responses. Query count varies with cache state and term work; it is not always exactly two queries.

## Prime images after resolving the products that will render

Product metadata must be available before discovering image IDs. When objects are loaded, prefer the existing helper:

```php
wc_get_container()->get( \Automattic\WooCommerce\Internal\Utilities\ProductUtil::class )
    ->prime_image_caches( $products );
```

Scope `$products` to the collection actually rendered. Match the consumer's getter context: variation image getters can fall back to the parent's image and apply view filters. Reading only raw variation `_thumbnail_id` does not cover that behavior. Raw metadata extraction is appropriate when the consumer itself uses those raw values; gallery metadata is a comma-separated ID list, not a serialized array.

Prime attachments when the consumer loads attachment records, metadata or URLs. Merely returning integer image IDs does not need attachment hydration. Audit classic templates, blocks and REST separately because they can consume the same products through different paths. [#63006](https://github.com/woocommerce/woocommerce/pull/63006) and [#69319](https://github.com/woocommerce/woocommerce/pull/69319) illustrate collection-specific placement.

## Prove that existing priming covers the final consumed IDs

`WC_Abstract_Order::get_items()` and order data stores already prime products on common load paths. Avoid another collect-and-prime loop when it duplicates that work. But check the exact path: lazy loading may prime a variation rather than its parent; in-memory items and items appended by `woocommerce_order_get_items` can differ from the set primed earlier. Remove a prime only after proving those consumed IDs are covered.

Keep priming before the first read, including a hook that actually consumes the same IDs. Preserve hook timing and overrides. Prefer a shared loading boundary when its bounded callers benefit; do not move broad hydration into global initialization simply because it reduces a later count. [#62820](https://github.com/woocommerce/woocommerce/pull/62820#discussion_r2710344921) narrows warming after object-cache hits; [#65276](https://github.com/woocommerce/woocommerce/pull/65276) warms only the page records needed by later checks.

## Check the cost and lifetime

Large batches can reduce round trips while increasing memory, bytes fetched or unwanted hydration. Measure the actual endpoint in cold/warm states and with/without persistent object caching. Do not infer a full-page improvement from an isolated method benchmark; [#69100](https://github.com/woocommerce/woocommerce/pull/69100#issuecomment-5912552732) rejected such a fast path after consumer-level measurements.

For hierarchical term priming, maintain a visited set and termination behavior for malformed/cyclic data. A few extra queries are preferable to a nonterminating traversal ([#70278](https://github.com/woocommerce/woocommerce/pull/70278#discussion_r4386236231)).

When moving beyond priming into value or instance caching, use the lifecycle checks in [database operations](database-operations.md#make-caching-and-scheduling-correct-across-their-lifecycle): identity/cloning, cache scope, mutation, invalidation, eviction and upgrade behavior. Add comments for these non-obvious constraints, not a mandatory comment that merely restates the prime call.
