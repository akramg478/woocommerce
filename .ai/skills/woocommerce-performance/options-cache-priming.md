# Options cache priming

Use when a consumer reads several option keys or transients. Distinguish existing autoloaded options, missing options, non-autoloaded options and the active transient backend before adding work.

## Batch the keys the consumer will actually read

`wp_prime_option_caches()` can batch cold reads of known option names, including missing names. Existing autoloaded options normally need no explicit prime. Cached and negatively cached keys may also need no SQL; many get_option calls alone do not establish an N+1 problem.

```php
$keys = array_map(
    static fn( $id ) => "woocommerce_{$id}_settings",
    $setting_ids
);
if ( ! empty( $keys ) ) {
    wp_prime_option_caches( $keys );
}
```

Place the prime before the first consumer, including constructors that read their settings. Derive keys from the final registry/settings structure; normalize nested or bracketed settings references to actual option names. Prime a bounded relevant group rather than every option a request might conceivably use. [#62853](https://github.com/woocommerce/woocommerce/pull/62853) batches email/settings reads; [#65883](https://github.com/woocommerce/woocommerce/pull/65883) covers registry-derived feature/gateway keys.

WooCommerce settings APIs commonly save options as autoloaded, but explicit overrides and historic state exist. Before first save an option may be absent even if its definition defaults to autoload. WordPress's priming helper checks autoload/cache state itself; avoid treating a definition or a large call count as proof of live SQL work.

## Use the correct transient storage

With an external object cache, transients use their cache group rather than the options table. Do not prime `_transient_*`/timeout option names on that path:

```php
wp_prime_option_caches( $regular_option_names );
if ( ! wp_using_ext_object_cache() ) {
    wp_prime_option_caches( $transient_option_names );
}
```

Priming nonexistent transient option names can grow the `notoptions` negative cache as **new unique names** accumulate. Repeating the same names does not add entries each time; eviction and legacy database transient rows also affect the behavior. The useful guard comes from [#65440](https://github.com/woocommerce/woocommerce/pull/65440) and its [skill correction #65441](https://github.com/woocommerce/woocommerce/pull/65441). It does not imply all external-cache access is free or that no backend-specific batching can help.

Keep ordinary option priming outside that guard when the key list is mixed. Site transients have a separate storage path: on multisite their database fallback is network options, not the current site's options table. Use the appropriate network-option cache API and network ID, checking availability in supported WordPress versions. Match the API and cache namespace to the consumer rather than merely matching a key prefix.

## Keep autoload changes consistent across time

Autoload moves cost into bootstrap payload; use it for suitably bounded, frequently read values with evidence. Check installation defaults, existing stored values, migration, and every later writer. In [#69693](https://github.com/woocommerce/woocommerce/pull/69693), settings saves undid an earlier migration because definitions still disabled autoload. Prefer WordPress option APIs for cache coherence and target the options that still exist.

Do not materialize a missing option just to avoid a read until its absence semantics are understood. Absence can select a dynamic default, unlike a stored false/zero. The [#70409 review](https://github.com/woocommerce/woocommerce/pull/70409#discussion_r4423848691) identified that distinction for HPOS background synchronization; check absence behavior before adopting the proposed stored default.

Validate query-count changes and end-to-end cost separately, with fresh/unconfigured and previously saved settings, external cache on/off, and relevant multisite scope. For invalidation, background warming and persistent counters, also read [database operations](database-operations.md#make-caching-and-scheduling-correct-across-their-lifecycle).
