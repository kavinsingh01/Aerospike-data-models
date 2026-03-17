# Aerospike Collection Data Type (CDT) API — Reference

**Summary:** Reference for the List and Map APIs, nested context, and ordering/comparison rules from the [Aerospike documentation](https://aerospike.com/docs/develop/data-types/collections/). Use alongside `aerospike-data-modeling.md` when designing or implementing data models.

**Status:** Reference.

**Source docs:**
- [Collections overview](https://aerospike.com/docs/develop/data-types/collections/)
- [List](https://aerospike.com/docs/develop/data-types/collections/list/)
- [Map](https://aerospike.com/docs/develop/data-types/collections/map/)
- [Nested context](https://aerospike.com/docs/develop/data-types/collections/context/)
- [Order and compare](https://aerospike.com/docs/develop/data-types/collections/ordering/)

---

## Collections overview

- **CDTs** are List and Map: schema-free containers that can hold scalars or nest other lists/maps. Elements can be mixed types.
- Bins hold one value per bin; that value can be a scalar, a CDT, or another supported type (e.g. HyperLogLog, GeoJSON).
- CDTs are a superset of JSON (e.g. integer map keys, bytes).
- **Multi-op transactions:** multiple list, map, or scalar ops can be combined in a single record operation; policies control rollback vs partial success on error.
- HyperLogLog can live inside a collection; blob/HLL ops inside CDTs use read-/write-expressions.

---

## List API

### List types and terminology

- **Unordered list:** Keeps insertion order as long as elements are appended. Rank-based access uses just-in-time value order.
- **Ordered list:** Keeps value (rank) order; re-sorts on insert. Mixed-type elements ordered by type then value.
- **index** — 0-based position (e.g. index 2 = third element). Negative = from end (-1 = last).
- **rank** — Value order (rank 0 = smallest value). Negative = from end.
- Same API for both; convert with `set_order()`.

### List operations (representative)

| Category   | Operations |
|-----------|------------|
| Order     | `set_order()`, `sort()`, `clear()` |
| Write     | `append()`, `append_items()`, `insert()`, `set()`, `increment()` |
| Size      | `size()` |
| By index  | `get_by_index()`, `get_by_index_range()` |
| By rank   | `get_by_rank()`, `get_by_rank_range()` |
| By value  | `get_all_by_value()`, `get_all_by_value_list()`, `get_by_value_interval()`, `get_by_value_rel_rank_range()` |
| Remove    | `remove_by_index()`, `remove_by_index_range()`, `remove_by_rank_range()`, `remove_all_by_value()`, `remove_all_by_value_list()`, `remove_by_value_interval()`, `remove_by_value_rel_rank_range()` |

- List bin is created when a list value is written to a bin or when using list `append`, `insert`, `set`, or `increment`.
- Insertion at either end is fast; append/delete at end is generally efficient.
- **Limitations:** Bound by max record size. List commands are not supported in Lua UDFs.

---

## Map API

### Map types and terminology

- **Unordered** — No guaranteed order.
- **K-ordered** — Stored in key order; in-memory namespaces get a key-offset index for faster access.
- **KV-ordered** — Stored in key order; in-memory also has rank (value) index. On SSD, value index persistence and performance depend on version (e.g. DB 7+ can persist value index).
- **key** — Element identifier. **value** — Element value. **index** — Key order (0 = smallest key). **rank** — Value order (0 = smallest value); ties broken by index.
- **From DB 7.1.0:** Map keys restricted to **integer, string, blob**.
- Same API for all map types; convert with `set_type()`.

### Map operations (representative)

| Category   | Operations |
|-----------|------------|
| Type      | `set_type()` |
| Write     | `put()`, `put_items()`, `increment()`, `decrement()`, `clear()` |
| Size      | `size()` |
| By key    | `get_by_key()`, `get_by_key_interval()`, `get_all_by_key_list()` |
| By index  | `get_by_index()`, `get_by_index_range()`, `get_by_key_rel_index_range()` |
| By value  | `get_by_value_interval()`, `get_by_rank_range()`, `get_all_by_value()`, `get_all_by_value_list()`, `get_by_value_rel_rank_range()` |
| Remove    | `remove_by_key()`, `remove_by_key_interval()`, `remove_by_index()`, `remove_by_index_range()`, `remove_by_value_interval()`, `remove_by_rank_range()`, `remove_all_by_value()`, etc. |

### Map op flags and return types

- **INVERTED** — Inverts the selection (e.g. “remove all but the 10 largest by index”).
- **Return types:** None, Index, RevIndex, Rank, RevRank, Count, Key, Value, KeyValue.

### Map guidelines

- Multiple ops can be combined in one record command. Map bin created on `put`/`put_items`/`increment` or when a map value is written to the bin.
- For **data-on-SSD**, **K-ordered** generally gives the best performance for all map ops (cost: ~4 extra bytes per entry).
- Nested list/map ops: from DB 4.6.0. Relative rank ops: from 4.3.0.
- **Limitations:** Bound by max record size. Map commands are not supported in Lua UDFs.

---

## Nested context

- **Context** identifies the path to the element an operation targets. Top-level = no context.
- Context is a list of (type, value) selectors applied from the top level or from the previous step.

### Context types

| Type                 | Description |
|----------------------|-------------|
| `BY_LIST_INDEX(idx)` | List element at index |
| `BY_LIST_RANK(rank)` | List element at rank |
| `BY_LIST_VALUE(val)` | List element by value (first match in value order) |
| `BY_MAP_INDEX(idx)`  | Map element at index |
| `BY_MAP_RANK(rank)`  | Map element at rank |
| `BY_MAP_KEY(key)`    | Map element by key |
| `BY_MAP_VALUE(val)`  | Map element by value |
| `MAP_KEY_CREATE(key)`   | Create map key if missing, then select (4.9+) |
| `LIST_INDEX_CREATE(idx)` | Create list slot if missing, then select (4.9+) |

- **CDT context** feature: DB 4.6.0+. Create-if-missing context types: 4.9.0+.

### Example: drilling into a list

List: `[0, 1, [2, [3, 4], 5, 6], 7, [8, 9]]`

- Target `[3, 4]` with context `[BY_LIST_INDEX(2), BY_LIST_INDEX(1)]` (third top-level element, then second element of that list).
- Then e.g. `list_append('l', 100, context=[BY_LIST_INDEX(2), BY_LIST_INDEX(1)])` appends 100 to `[3, 4]`.

### Example: operating on map value (list)

- Map: segment_id → `[ttl, attrs]`. To increment the TTL (first list element) at a given segment_id, use **list** op with context **by map key**: e.g. `list_increment("u", 0, 5, ctx=[cdt_ctx_map_key(segment_id)])`.

### Example: create path then operate

- `map_increment('m', 'jokes', 317, context=[MAP_KEY_CREATE('stats'), MAP_KEY_CREATE('accolades')])` creates `stats` and `accolades` if missing, then increments `jokes` under `accolades`.

---

## Order and compare

### Type order (ascending)

1. NIL  
2. BOOLEAN  
3. INTEGER  
4. STRING  
5. LIST  
6. MAP  
7. BYTES  
8. DOUBLE  
9. GEOJSON  
10. INF  

- Different types compare by type first; same type by value. Unordered maps have no guaranteed order; ordered maps use this for value ordering.
- **INF** (4.3.1+): singleton, highest type; used in intervals, not for storage.

### Value comparison

- **List:** Compare element by element from index 0; then by length. E.g. `[1,2] < [1,3]`, `[1,2] < [1,2,1]`.
- **Map:** By element count, then keys in stored order, then values when keys match. (Pre-4.3.1: known issues for different-length maps/lists.)

### Wildcard and intervals

- **Wildcard (`*`):** In operations (e.g. `get_all_by_value([1, *])`) matches any value in that position. Not a storage type. 4.3.1+.
- **Intervals:** Default **inclusive-exclusive** `[start, end)`.
- **INF** for inclusive end: e.g. `list_get_by_value_interval(start=[1, NIL], end=[2, INF])` gives elements ≥ `[1, NIL]` and ≤ `[2, INF]` at the second level. 4.3.1+.

---

## Relation to data modeling (this project)

- **User profile store:** Bin is a **map** (segment_id → value). Value = **list** `[ttl, attrs]`. Range ops use **list-to-list** comparison: e.g. `map_get_by_value_range` with `[now, null]`..`[infinity, null]` for “non-expired” segments. **Context** `BY_MAP_KEY(segment_id)` used with **list_increment** to update TTL in place.
- **IoT sensors:** Bin is a **list** of `[minute, temp_x10]`. **get_by_value_interval** for time ranges; ordering and interval rules apply.
- When comparing list or map values in range ops, wrap scalars in a list if the stored value is a list (e.g. segment value `[ttl, {}]` compared with `[ttl, null]` or wildcard).
