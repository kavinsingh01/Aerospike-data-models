# Aerospike research

Reference and patterns for Aerospike data modeling and API usage. Use this folder when you need Aerospike concepts, patterns, or API details.

**URL tracking:** Before processing a doc URL for research, check [urls-processed.md](urls-processed.md). If the URL is already listed, ask whether to reprocess before fetching again.

**Data modeling updates:** Validate any new data modeling information against the existing content in this folder. Do not add to the data modeling knowledge unless the information is new. If new material seems to conflict with what is already documented, ask for clarification before adding or changing items.

## Files in this folder

| File | Purpose |
|------|---------|
| [aerospike-data-modeling.md](aerospike-data-modeling.md) | Data model concepts, primary/secondary indexes, distribution; Goldilocks Principle (record size 1–128 KiB); **data modeling tips** (denormalization, namespace-wide SI, unique lookup table, sample population); applied patterns from IoT, profiles, relationships, leaderboards, time series, etc. |
| [aerospike-one-to-many-relationships.md](aerospike-one-to-many-relationships.md) | One-to-many patterns: list on parent, consolidate (one record per parent), or child-held reference + secondary index. Choice by cardinality and who drives the read; record size (Goldilocks). |
| [aerospike-subscription-scale-user-set.md](aerospike-subscription-scale-user-set.md) | Scaling subscribers/following: why lists on the user record don’t scale; consolidated alternative (e.g. `user_subscribers` / `user_following` per user with list of handles). |
| [aerospike-cdt-api.md](aerospike-cdt-api.md) | List and Map APIs, nested context, ordering/comparison. Reference for CDT operations when designing or implementing data models. |
| [aerospike-expressions.md](aerospike-expressions.md) | Filter and operation expressions (WHERE clause, computed bins). Research summary from Summit talk; use official Expressions docs for current API. |
| [aerospike-path-expressions.md](aerospike-path-expressions.md) | Path expressions (8.1.1+): nested CDT query/index, selectByPath/modifyByPath. For document-style or nested-CDT models. |

Project-specific research (e.g. sizing, benchmarks) lives in `_dev/projects/<ProjectName>/research/`.
