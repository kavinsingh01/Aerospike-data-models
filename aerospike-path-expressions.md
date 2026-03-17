# Aerospike Path Expressions — Research Summary

**Summary:** Path expressions enable granular querying and indexing of nested List and Map structures (context stack, loop variables, selectByPath/modifyByPath). Use with [aerospike-cdt-api.md](aerospike-cdt-api.md) and [aerospike-data-modeling.md](aerospike-data-modeling.md) when designing document-style or nested-CDT models.

**Status:** Preview (non-production). Requires **Aerospike Database 8.1.1** or later. Check [client compatibility](https://aerospike.com/docs/develop/client-matrix).

**Source docs:**
- [Path expressions overview](https://aerospike.com/docs/develop/expressions/path/)
- [Quickstart](https://aerospike.com/docs/develop/expressions/path/quickstart/)
- [Advanced usage](https://aerospike.com/docs/develop/expressions/path/advanced/)
- [FAQ](https://aerospike.com/docs/develop/expressions/path/faq/)

---

## Overview

Path expressions add **contextual expressions** over nested [List](https://aerospike.com/docs/develop/data-types/collections/list/) and [Map](https://aerospike.com/docs/develop/data-types/collections/map/) CDTs. They address:

- **Performance:** Server-side filtering and selection so only matching elements (or subtrees) are returned — less network transfer than fetch-whole-record-then-filter.
- **Flexibility:** Native support for JSON-like document models (e.g. map of products → map/list of variants) without denormalizing or writing client-side loops.

Combined with [expression indexes](https://aerospike.com/docs/database/learn/tutorials/quick-start-expression-index/), path expressions allow **indexing and querying** data inside deeply nested structures.

### Key capabilities

- **Multi-element selection:** Select, retrieve, or manipulate multiple elements from a nested CDT in one operation.
- **Iteration + loop variables:** Iterate over map key-value pairs or list elements; use **loop variables** (MAP_KEY, VALUE, LIST_INDEX) in expressions to filter or read from the current element.
- **Expression indexes:** Select multiple elements from a document (nested CDT) to build an index; query and traverse deep structures without denormalization.

---

## Concepts

### Context stack

Path operations use a **context stack** that defines how to traverse the bin’s CDT:

1. Start at the bin (e.g. `inventory`).
2. Each step is a **context** (e.g. “all children”, “all children matching filter”, “map key `variants`”).
3. Filters are **expressions** evaluated at each level; they can use **loop variables** to read the current iteration’s key, value, or list index.

Example shape (from Quickstart):

```
inventory (bin)
 └── product (map entry, keyed by productId)
     ├── category, featured, name, description
     └── variants  (map of SKU→attrs or list of variant objects)
         └── variant-level filter (e.g. quantity > 0)
```

### Loop variables

When iterating over CDT elements, the server exposes the current element via loop variables so expressions can reference it:

| Loop variable   | Meaning (map)        | Meaning (list)     |
|-----------------|----------------------|--------------------|
| **MAP_KEY**     | Key of current entry | N/A                |
| **VALUE**       | Value of current entry (often a map or list) | Element at current index |
| **LIST_INDEX**  | N/A                  | Index in the list  |

Example: “product-level” filter `featured == true` uses `MapExp.getByKey(..., "featured", Exp.mapLoopVar(LoopVarPart.VALUE))` — the loop variable is the product map, and the expression reads its `featured` field.

### Context types (representative)

- **allChildren()** — Descend into all children (map entries or list elements).
- **allChildrenWithFilter(filter)** — Same, but keep only elements for which the expression evaluates true.
- **mapKey(key)** — Descend into the map entry for the given key (e.g. `"variants"`).

Same idea as CDT nested context (BY_MAP_KEY, etc.), but used here with **expressions** and **filters** instead of fixed paths.

---

## Operations

### selectByPath (read)

- **Purpose:** Select a subset of the nested CDT and return it.
- **Parameters:** Bin name, **selectFlags**, and a list of context steps (with optional filters).
- **Result:** Depends on selectFlags (see below).

### modifyByPath (write)

- **Purpose:** Update selected elements in place (e.g. increment `quantity` for all in-stock variants of featured products).
- **Parameters:** Bin name, modify flags, an **expression** that computes the new value (often using loop variables to read current value and `MapExp.put` / similar to write back), and the same kind of context stack.
- **Use case:** Bulk updates on filtered subsets of a nested structure without read-modify-write in the client.

---

## SelectFlags (return mode)

Control what the server returns from `selectByPath`:

| Flag            | Description |
|-----------------|-------------|
| **MATCHING_TREE** | Return subtree from bin to leaves, including only nodes that passed filters. Good for “return filtered document.” |
| **VALUE**       | Return a list of the **values** of the nodes finally selected. |
| **MAP_KEY**     | For final nodes that are map elements, return only the **map keys** (e.g. list of SKU ids). |
| **MAP_KEY_VALUE** | Return key-value pairs for final selected map elements. |
| **NO_FAIL**     | On type mismatch (e.g. expecting map, got string), skip that element instead of failing the whole operation. |

Without **NO_FAIL**, any type mismatch aborts the operation with no partial results.

---

## Example: list of structs (e.g. user’s vehicles)

Without path expressions, iterating over a list of map elements (e.g. vehicles as structs with make, color, license) to filter or index by a field is not supported; the only option is to **denormalize** (e.g. a separate bin holding a list of license strings) to index or query by license. Path expressions allow a single **list of maps** in one bin (e.g. `vehicles`: `[{make, color, license}, ...]`) and:

- **Filter in place:** Use `selectByPath` with a context that iterates over the list’s children and a filter on `license` (e.g. `MapExp.getByKey(..., "license", Exp.mapLoopVar(LoopVarPart.VALUE)) == "ABC123"`) to return only vehicles (or only matching vehicle structs) where license equals a given string.
- **Index by field:** Create an **expression index** that iterates over the vehicle map structures inside the list and indexes the `license` (or other) field — so you can query “all users with license plate X” without a separate bin holding a list of license strings.

This **list-of-maps** pattern (list of structs with named fields) is directly useful for the standard example application: e.g. one record per user, one bin `vehicles` = list of `{make, color, license}`, with server-side filter by license or expression index on license. See [aerospike-data-modeling.md](aerospike-data-modeling.md) for how this fits with other modeling choices.

---

## Data modeling implications

- **Document-style records:** One record (e.g. catalog) can hold a large map of entities (e.g. productId → product). Path expressions let you “query” that document server-side: e.g. “featured products with at least one in-stock variant,” returning only those products and only in-stock variants.
- **List of structs:** A bin can hold a **list of maps** (each map = one “struct” with fields like make, color, license). Path expressions let you iterate over that list and filter or project by field; expression indexes let you index a field across all list elements — no need for a separate denormalized bin (e.g. list of license strings) just to query or index.
- **Avoid denormalization:** You can keep a single source of truth (one record per catalog or user) and use path expressions to project subsets (e.g. by segment, by time window, by flag) instead of maintaining separate flattened records.
- **IN-style membership:** There is no cluster-level SQL-style `IN`. Within a **single record**, you can emulate IN over map keys (or values) using expressions: e.g. `ListExp.getByValue(COUNT, Exp.stringLoopVar(MAP_KEY), Exp.val(roomIds)) > 0` to keep only entries whose map key is in `roomIds` (see [FAQ](https://aerospike.com/docs/develop/expressions/path/faq/)).
- **Combining filters:** Multiple conditions (e.g. quantity > 0 AND price < 50) use `Exp.and(...)` / `Exp.or(...)` with each condition reading from the same loop variable (e.g. variant map).
- **Return shape:** Use MATCHING_TREE for “filtered document,” MAP_KEY for “list of ids,” VALUE for “list of values” — keeps payloads small when you only need keys or a flat list.

---

## Advanced usage (high level)

- **Loop vars for keys:** Filter by map key (e.g. productId) using `Exp.stringLoopVar(LoopVarPart.MAP_KEY)` in a regex or comparison (e.g. keys starting with `"10000"`).
- **Alternative return modes:** Same context stack, different selectFlags — e.g. MAP_KEY to get only in-stock variant SKUs for featured products.
- **modifyByPath:** One operation to update many nested elements (e.g. increment quantity by 10 for all in-stock variants of featured products) using an expression that reads from `Exp.mapLoopVar(VALUE)` and writes back with `MapExp.put`.
- **Chained filters:** AND/OR over expressions that all refer to the current element via the same loop variable.

---

## Limits and performance (FAQ)

- **Nesting depth:** Up to **15 levels** (same as CDT context limit).
- **Elements:** No hard limit on number of elements, but very large CDTs (e.g. millions of elements) can increase latency; consider partitioning across records or using secondary indexes to narrow scope before path expressions.
- **Performance factors:** Result size (MATCHING_TREE vs MAP_KEY), filter complexity, nesting depth, CDT size. Prefer server-side path filtering over full-record fetch + client filter when possible; benchmark with realistic data.

---

## Relation to other research

- **CDT API ([aerospike-cdt-api.md](aerospike-cdt-api.md)):** Path expressions use the same underlying List/Map types and ordering; context steps are analogous to nested CDT context (BY_MAP_KEY, BY_LIST_INDEX, etc.), but with expression-based filtering and loop variables.
- **Data modeling ([aerospike-data-modeling.md](aerospike-data-modeling.md)):** Path expressions are a good fit for “one record, one big map/list” patterns (e.g. user profile store, product catalog) when you need server-side filtering, projection, or bulk updates on nested subsets without denormalizing into many records or bins. The **list-of-structs** pattern (e.g. user’s vehicles as list of maps with make, color, license) is called out there for the standard example application: one bin, path expressions to filter by field, expression index to query by field, no separate denormalized bin needed.
