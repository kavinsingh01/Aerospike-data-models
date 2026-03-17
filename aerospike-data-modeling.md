# Aerospike Data Modeling — Project Notes

**Summary:** Data modeling concepts from Aerospike documentation and example articles (IoT sensors, user profile store), the Document-Oriented Modeling Workshop (Summit '20), and the use-case-cookbook (relationships, leaderboards, player matching, time series, recent events across DCs, advanced expressions). Consolidated foundation, applied patterns, and takeaways for defining the standard example application’s data model.

**Status:** Research. Informs the data model for the standard example application.

---

## Foundational Concepts (Aerospike Docs)

Background from the [Aerospike documentation](https://aerospike.com/docs/) on data storage and clustering. References: [Data model](https://aerospike.com/docs/database/learn/architecture/data-storage/data-model/), [Primary index](https://aerospike.com/docs/database/learn/architecture/data-storage/primary-index/), [Secondary index](https://aerospike.com/docs/database/learn/architecture/data-storage/secondary-index/), [Set index](https://aerospike.com/docs/database/learn/architecture/data-storage/set-index/), [Resilience](https://aerospike.com/docs/database/learn/architecture/data-storage/resilience/), [Data distribution](https://aerospike.com/docs/database/learn/architecture/clustering/data-distribution/).

### Data model

Aerospike uses a **schemaless** data model: no fixed schema; structure is determined by how the application uses the system.

| Component | Description |
|-----------|-------------|
| **Physical storage** | Per-namespace choice: NVMe Flash, DRAM, or PMem. Different namespaces in the same cluster can use different storage engines. |
| **Namespace** | Top-level container (like a tablespace): records that share one storage engine and policies (replication factor, encryption, etc.). A database can have multiple namespaces. |
| **Set** | Optional logical grouping of records within a namespace (table-like, but no explicit schema). Records can belong to a set or only to the namespace. Scans and secondary indexes can be scoped to a set. **Set name cannot exceed 63 bytes** (UTF-8). |
| **Record** | Unit of storage: uniquely identified by a key. Contains **metadata** (generation, TTL, LUT) and **bins** (name + value). |
| **Bin** | Name + value; type is defined by the value. **Bin names are limited to 15 characters.** No schema: each record can have different bins; bins can be added/removed. Values use [native data types](https://aerospike.com/docs/develop/data-types/blob) (scalars, blob, list, map, etc.). |

**Key and digest:** The application operates on records by key (or digest). The client hashes the key (e.g. RIPEMD160) to a 20-byte digest. The server can work with either; the digest is used for partition placement and index metadata.

**Key send policy and omitting the key bin:** When the key is sent on a write (`POLICY_KEY_SEND`), the server can store it in the record (if it wasn't stored yet). Once stored, it is available on read (e.g. in batch responses); using `POLICY_KEY_DIGEST` (or not sending the key) on a later operation does not clear it — it simply doesn't send the key on that operation. You can **omit storing the same value as a bin** and rely on the key from metadata — a small space-saving technique. Trade-offs: (1) **Clarity** — the "primary key" is not visible in the bin set; it lives in metadata. (2) **Consistency** — applications should use the send-key policy on writes that create or update records so the key gets stored where needed; document this in the application-level contract so all clients stay consistent.

**Record metadata:** Generation (modification count, for CAS), TTL (optional expiration; as of 4.9 expirations are off by default), LUT (last-update-time, used internally for conflict resolution, backup, XDR, etc.).

### Primary index

- **One per namespace:** Automatically created; one metadata entry per record (~64 bytes per entry). Enables consistent, low-latency access to any record by key.
- **Index metadata** includes: 20-byte digest, generation, void-time (expiration), LUT, replication state, and **location of the record on storage** — so a get is a single read I/O.
- **Structure:** Keyspace is hashed into **4096 partitions**; partitions are distributed across nodes. Per-partition index is implemented as multiple red–black trees (**sprigs**) to reduce traversal depth and lock contention. Replication factor determines how many copies of each partition exist; replicas are on different nodes.
- **Persistence:** Index can be rebuilt from data (e.g. cold restart). With fast restart (EE), index can live in shared memory and reattach on planned restart without a full data scan.
- **Storage:** In EE, `index-type` can be `shmem`, `flash`, or `pmem`. CE uses volatile process memory.

### Secondary index

- **Purpose:** Locate records by **bin value** (or by computed value with expression indexes in 8.1+) instead of only by primary key. Enables queries like “all records where `status` = `active`” without scanning the whole namespace.
- **Bin-based:** Index on a bin (optionally with list/map context). **Expression indexes** (DB 8.1+): index on a computed value from an expression.
- **Name limit:** **Secondary index name cannot exceed 63 bytes** (UTF-8). Deployment-specific index names must stay within this limit.
- **Behavior:** Stored per node, co-located with primary index; each SI entry references only local records. Created/removed dynamically (e.g. via asadm or API). On write, SI is updated atomically with the record. Index metadata is keyed by type (integer, string, blob, geospatial) and index type (e.g. list, map-keys, map-values).
- **Queries:** SI query scatters to all nodes, each node reads from its SI and storage, results are gathered by the client. During migration, clients (e.g. Java 6.0+) can reschedule to get correct results. SI queries can feed **aggregation** (stream UDFs).
- **Storage:** `sindex-type` (EE): `shmem`, `flash`, or `pmem`. CE: volatile memory.

**Secondary index capacity planning (Database 6.0.0 and later):** The following applies only to 6.0+; see [Secondary index capacity planning](https://aerospike.com/docs/database/manage/planning/capacity/secondary-indexes) for the full reference. “Space” means memory (shmem or process), PMem, or disk depending on `sindex-type`. Only records where the indexed bin exists and has the correct data type are indexed.

- **Structure:** One B-Tree per partition (4096 partitions, collocated with primary index). Each B-Tree node uses 4 KiB.
- **Entry cost:** Each secondary index entry is **14 bytes** (8 bytes for the bin value or its hash, 6 bytes to reference the record in the primary index). Scalar bins: one entry per indexed record. List bins: one entry per list element that matches the index type. Map bins: one entry per key or value being indexed. GeoJSON Point: one entry; Polygon or AeroCircle: `max-cells` entries (configurable, default 12).
- **Sparseness:** B-Tree average sparseness is ~1.5× entry space, worst case ~2×. Example: 10M entries → ~200 MiB average, ~267 MiB worst case. Multiply by namespace replication factor for cluster total.
- **Allocation:** Each secondary index has an **initial 16 MiB** per node (e.g. four SIs = 64 MiB per node). From 6.1.0, growth is controlled by **sindex-stage-size** (minimum 128 MiB, default 1 GiB). The first SI defined gets the first stage; SIs grow into it; when full, the next stage is allocated. Plan capacity in `sindex-stage-size` increments.
- **Max size:** If an SI can exceed **2 TiB**, `sindex-stage-size` must be increased from the default 1 GiB (max 2048 arenas per SI, so larger stages are required for >2 TiB).

**Secondary index lifecycle and behavior** (from [Secondary index](https://aerospike.com/docs/database/learn/architecture/data-storage/secondary-index/) architecture): Index metadata is maintained in **System Metadata (SMD)**; create/remove flows from client → any node → SMD principal → all nodes → each node’s sindex module. On create, each node builds the SI in write-only (WO) mode via a background scan; when the scan finishes, the index is marked read-active on that node. New nodes that join with data but missing index definitions get indexes from SMD and populate them (queries not allowed until population completes). **On writes:** if the bin is missing, or the value doesn’t match the declared data type and index type, or (for expression indexes) the expression fails to produce a value of the declared type, **no secondary index action** is performed; all SI updates are atomic with the record under single-lock. **Garbage collection:** for non-durable deletes (client delete, expiry, eviction, truncation, migration drop), the SI entry is removed without reading the record from disk; a background thread periodically cleans remaining entries. **Distributed SI queries:** request scatters to all nodes; each node reads from its SI and storage in parallel; results are aggregated per node and gathered by the client. SI queries are batched (key lists and response flushing) to bound space. With Database 6.0+ clients, partitions that are migrating are rescheduled so results stay correct. **Aggregation:** SI query results can feed stream UDFs; global queues and thread pools manage the pipeline; batching and UDF state caching limit overhead. **Storage:** `sindex-type` (EE/SE 6.1+): default **shmem** (shared memory); or `flash`, `pmem`. CE: volatile process memory. All types can use fast restart.

### Set index

- **Purpose:** Index **set membership** so that “all records in set S” can be found without traversing the full primary index.
- **Use case:** When a set is small relative to the namespace, a set index avoids scanning the whole PI. ~16 bytes per record in the set. Most useful when the set is &lt; ~1% of the namespace; can be tried and removed if not helpful.
- **Management:** Dynamic (enable/disable per set). Avoids the need for a secondary index on a bin containing set name; no extra bin needed, and scans on the set are correct and support pagination.

### Resilience (writes and throttling)

- **Writes:** Records are grouped into an 8 MiB streaming write buffer (SWB); flush to device when full or when `flush-max-ms` expires, in units of `flush-size`. Copy-on-write for create/update/replace.
- **Throttling:** When the write cache (pending write blocks) exceeds baseline and thresholds, the server returns errors (e.g. device overload) and logs “queue too deep” for different write types (master, UDF, replica, migration, defrag, etc.) so that overload doesn’t take down the node.

### Data distribution

- **Shared-nothing:** All nodes are peers; no single point of failure. Data is distributed evenly across nodes.
- **Partitions:** Each namespace is divided into **4096 partitions**. Record key → hash (RIPEMD160) → 20-byte digest → 12 bits determine partition ID. Partitions are distributed across nodes (e.g. ~1024 per node in a 4-node cluster).
- **Replication:** Replication factor (e.g. RF2 = one master + one replica) is per namespace. Master and replica(s) are on different nodes. Writes propagate to replicas (synchronous replication option); client normally talks to the master for the partition.
- **Rebalancing:** On node add/remove, the cluster rebalances; partitions migrate. Clients can be rebalance-tolerant (e.g. reschedule SI queries during migration). Succession list defines which node holds master and replicas for each partition; with `prefer-uniform-balance` (default), balance is preferred and succession can change.

**Record sizing and primary index cost:** Every record incurs a **64-byte primary index metadata** entry. Many tiny records therefore cost a lot of index memory and don't use storage efficiently. We aim to **consolidate data into records that are a few KiB in size**. The maximum record size is **8 MiB**, so we must stay under that. Aerospike is very efficient at **batch reads**, so a design with many records of several KiB each is preferable to a single multi-MiB "giant" record: we get better parallelism, more granular access, and avoid hot spots while still keeping per-record index cost reasonable.

**Goldilocks Principle (ideal record size):** The "by cardinality" aspect of one-to-many modeling is like Goldilocks and the Three Bears. **Too small** — when the ratio between record size and the 64-byte index metadata is poor, cost-efficiency suffers. In most deployments the primary index lives in memory and data on SSD, so many tiny records increase index memory cost without using storage efficiently. **Too big** — very large records hurt read/write performance and defragmentation. **Just right** — records in the **1–128 KiB** range tend to balance index-to-data ratio, I/O, and defrag impact. When choosing list-on-parent vs consolidate vs inverse index, aim for record sizes in this band where possible.

**Implications for data modeling:** Key design determines partition (and thus which node and how data is distributed). Aerospike does **not** support collocation of related records via key design; keys are uniformly distributed for load balancing. Key design should still support **access patterns** (e.g. compose the key so related objects can be looked up or batched efficiently). No schema means our “standard” data model is an application-level contract: we agree on namespace, set(s), key format, and bin names/types so that all clients read/write the same logical model; the server does not enforce it. Record granularity and target size (few KiB, under 8 MiB) should be part of that contract; batch reads make many medium-sized records a good fit.

---

## Capacity planning (Database 7.1.0 and later)

The following summarizes the [Capacity planning guide](https://aerospike.com/docs/database/manage/planning/capacity/) for 7.1.0 and later only. Older version notes (e.g. pre-7.0.0 memory-size, pre-7.0.0 in-memory storage format, pre-6.4 single-bin) are omitted.

- **Required memory:** Reserve enough memory for OS, namespace overhead, and other software to avoid OOM. Use configurable stop-writes and eviction thresholds; see [Configuring namespace data retention](https://aerospike.com/docs/database/manage/namespace/retention).
- **Transactions (CP mode):** With transactions enabled, writes create extra tombstones at the rate of transactions ended per second. Tune `tomb-raider-period` (and related config) if transaction throughput is high; durable deletes required for deletes in a transaction.
- **Primary index:** **64 bytes × replication factor × number of records** (cluster total). For **index on flash**, primary index is in 4 KiB blocks; each sprig uses **10 bytes RAM**. Size index device with `mounts-budget` (7.0+); formula: `((4096 × replication-factor / min-cluster-size) × partition-tree-sprigs) × 4 KiB`. Sprig count and fill fraction affect reads; see guide. If primary index exceeds **2 TiB**, increase `index-stage-size` (max 2048 arenas).
- **Set index capacity:** Set indexes are **always in memory**. Per set index: **16 bytes × RF** per record in the indexed set; **16 MiB × RF** pre-allocated when the set index is created (covers first million records), distributed across nodes; **4 MiB × RF** overhead per set index, distributed across nodes. Beyond the first million records in a set, memory grows in **4 KiB micro-arena** increments (each 4 KiB indexes up to 256 records in one partition). Example: 1000 sets with set index, RF2 → 31.25 GiB initial stage + 8 GiB overhead across the cluster.
- **Secondary index:** See [Secondary index capacity planning (6.0+)](#secondary-index) above and the [Secondary index capacity planning](https://aerospike.com/docs/database/manage/planning/capacity/secondary-indexes) page.
- **Data storage:** From 7.0.0, in-memory data storage is **pre-allocated and static**. From **7.1.0**, **indexes-memory-budget** limits how much memory indexes can use. Record overhead is **39 bytes** (6.0+); storage formula includes bin overhead, key, set name, TTL, etc. — see the capacity guide. **64 MiB** (8 × 8 MiB write-blocks) is reserved per device; recommended minimum device size 128 MiB.
- **Defragmentation:** Plan to use no more than ~50% of storage by default (`defrag-lwm-pct`). Write-blocks in the **post-write-cache** (7.1.0+) are not defragmented; keep post-write-cache small relative to device size.
- **Throughput:** `(records accessed per second) × (record size)`; plan so the cluster can handle full load with one node down.
- **Provisioning:** See [Provisioning a cluster](https://aerospike.com/docs/database/manage/planning/capacity/provisioning) for examples.

---

## Data modeling tips

The following are practical tips for Aerospike data modeling. They are not sourced from URLs (articles, GitHub repos); they are guidance for the architect.

**Relational vs NoSQL mindset:** Relational modeling emphasizes normalization and minimizing storage through deduplication. NoSQL (including Aerospike) does the opposite: denormalization is used in data modeling to prioritize low-latency access over storage compaction. When modeling for Aerospike in particular, the architect should maximize performance (consistent low latency and high throughput) while minimizing memory consumption for cost efficiency.

**Namespace-wide secondary index for sparse sets:** Assume the secondary index (SI) is stored in memory. If many sets share a repeating bin and the application regularly queries on ranges of that bin’s values, a **single SI over the whole namespace** (no set name in the SI definition) can be more memory-efficient when many sets are sparsely populated per node. Reason: minimum index pre-allocation (~16 MiB per node) is sized for on the order of a million indexed values; one namespace-wide SI shares that allocation across all sets. Example: an integer bin `created_at` with range queries on creation time, and a large number of sets that are sparsely populated per node. If the application instead queries range values on the same bin for only a **subset** of sets, use a **namespace-wide expression index** that indexes the bin’s values and uses `cond(..., unknown())` to skip records in sets that are never queried — effectively a multi-set selective index with one index.

**Unique “index” via lookup table, not SI:** The relational concept of a unique index should be expressed in Aerospike as a **lookup table**: the external unique ID is the user key, and the record digest is stored in a single bytes bin. To find a record by that external ID, the application does one `get()` on the lookup table, then accesses the record by the retrieved digest. Building a secondary index on a “unique” bin in the record itself is worse: a query for that value scatters to all nodes and returns at most one record. The lookup table gives lower latency and better throughput than such an SI query.

**Consistent sample population with expression index:** To target a consistent sample population (e.g. for analytics or sampling), use an expression index that indexes using **digest modulo** (or similar expression), so a deterministic subset of records is indexed and queryable.

**Strong-consistency namespace: expunge volatile records:** In a strong-consistency namespace, **volatile records** (records deleted without the durable-delete policy) should be **expunged** rather than left as tombstones. Expunging frees capacity faster and reduces CPU/IOPS used by the tomb-raiding process. Set the namespace configuration parameter **strong-consistency-allow-expunge** to `true` to enable expunge for volatile records in that namespace.

**Auto-sharding hot keys (counter pattern):** When a record's counter bin is under high write concurrency (e.g. view counts, like counts), a single record can become a hot key and return `KEY_BUSY` on writes. Distribute the counter across **N sub-keys** so concurrent increments spread across records. The original record holds a `subkeys` bin (integer, value = N) that tells readers and writers how many sub-keys exist; each sub-key holds a partial counter. Sub-key record key = original key + sub-key index (e.g. `{originalKey}:{i}` for i in 0..N−1).

- **Read:** Get the original record. If it has a `subkeys` bin, also **batch read** all N sub-key records and **sum** the sub-key counter values with the original record's counter value.
- **Write (normal path):** Increment the original record's counter using a **filter expression** that checks for the **absence** of a `subkeys` bin; fail the write if `subkeys` exists (FILTERED_OUT). On FILTERED_OUT: the record is already sharded — compute a sub-key index via `nanotime % N` and increment that sub-key's counter instead.
- **Write (KEY_BUSY on original):** If incrementing the original key returns KEY_BUSY, the record is hot but not yet sharded. Assume N sub-keys (application-side config, e.g. 100). Compute sub-key index via `nanotime % N` and increment that sub-key's counter. Then attempt **once** to update the original record with a `subkeys` bin set to N (so future writes and reads know the key is sharded).
- **Pre-sharding:** If the application knows the key will be hot (e.g. a viral post), create the original record with a `subkeys` bin set to N in advance. All writes will immediately shard.
- **Dynamic scaling:** If a sub-key update itself hits KEY_BUSY, increase N for all subsequent writes (application-side), recompute which sub-key to increment (new `nanotime % N`), and attempt to update the original record's `subkeys` bin to the new N' value. Existing sub-keys remain valid; new sub-keys are created on first write.

**From blog [Data Modeling for Speed at Scale](https://aerospike.com/blog/data-modeling-for-speed-at-scale):**

- **Split object across namespaces:** The same (set, user-key) can be used in **multiple namespaces**; the digest is identical, so the same logical key refers to related records in each namespace. Example: one namespace for the latest version of an object, another for archived versions; application uses the same key to read from either. Useful when different namespaces have different storage policies (e.g. fast vs cold) or retention.

- **Trigger-like behavior via same digest across namespaces:** The identical digest for the same (set-name, userKey) in different namespaces can be used to mimic triggers with XDR/Change Notification. Use two namespaces: **storage** (full object) and **triggers** (sparse record: bin `event` and optional metadata). Configure the triggers namespace to ship events via XDR/Change Notification to an Aerospike ESP connector (or similar), which forwards to an endpoint. Any write that should fire an event is written to the triggers namespace with the **same key** as the storage record; the handler receives the notification, reads the full record from storage by key, and combines it with the trigger payload. For **on-delete handling:** delete the record in the triggers namespace (same key). The handler gets the delete notification; the storage record is still present, so the handler can read the full record from storage, process it (e.g. cascade, audit), then delete it from storage. This avoids the limitation that XDR/Change Notification delivers only the delete event, not the deleted record body — the “deleted” record is the trigger record, while the payload remains in storage until the handler processes and removes it.

**From blog [Data Modeling for Speed at Scale (Part 2)](https://aerospike.com/blog/data-modeling-for-speed-at-scale-part-2/) (CDTs):**

- **Object ID design for direct access:** When objects live inside a CDT and must be reachable by object id, **embed the record key (or collection id) in the object id** (e.g. as a prefix). The application can then derive the record key from the object id, do one get by key, then target the object inside the CDT (by value or by map key). Example: if the record key is region id and the CDT holds all stores in that region, store ids can be `regionId:storeId` so the record key is always known.
- **Direct access by container type:** **List of Lists** (object = list tuple with id as first field): use get-by-value with id + wildcard (e.g. `get-by-value(outerList, ["id1", *])`). **Map of Maps** (object id = map key): use get-by-key(outerMap, "id1"). **Map of Lists** (object = list tuple inside map value): there is no wildcard for map *values*, so you cannot do "get by object id" inside the list; use Map of Maps if you need direct access by object id within the container.
- **Value-based access: prefer List tuple over Map:** List tuples support value and value-range selection with **wildcard** and **NIL/INF** (e.g. get-by-value-range on first field). Maps do not support wildcard in value comparison; exact or range match on map values requires specifying all keys. When you need value-based or range-based selection (including in filter expressions), model objects as **list tuples** with the selection field first; use Map when you need key-based access or self-describing schema (e.g. JSON).
- **Filter expressions on CDT value predicates:** Value-based predicates in filter expressions (e.g. on a list of tuples) can only use the **first field** of the tuple for comparison; wildcard/NIL/INF apply as in CDT ops. See [aerospike-cdt-api.md](aerospike-cdt-api.md) for ordering and interval rules.

**From blog [Taking advantage of probabilistic data structures](https://aerospike.com/blog/taking-advantage-of-probabilistic-data-structures/) (HyperLogLog):**

- **HLL for approximate distinct count:** Aerospike’s native **HyperLogLog (HLL)** type (and HyperMinHash for set similarity) supports approximate cardinality with fixed, small storage per bin. Use when you need “how many distinct X?” (or union/intersect of such sets) and approximate is acceptable. **Data model:** One record per (dimension, time bucket), e.g. key `tag:month`; one HLL bin holds the set of identifiers. **Ingest:** Add each identifier to the HLL (idempotent); no need to store raw events. **Query:** **hll_get_count** for one set; **hll_get_union_count** across buckets (e.g. distinct users over a year); **hll_get_intersect_count** for multi-tag (e.g. users matching tag A and tag B). Index bit count trades size vs accuracy (e.g. 12 bits → ~3 KB per HLL, ~1.6% standard error). See [aerospike-expressions.md](aerospike-expressions.md) § HyperLogLog bin operations for the expression API; [HLL data type](https://aerospike.com/docs/develop/data-types/hll) for semantics. Example code: [aerospike-examples/hll-python](https://github.com/aerospike-examples/hll-python).

---

## Source 1: IoT Sensors (Time-Series / Roll-Up by Day)

**Article:** [Aerospike Modeling: IoT Sensors](https://medium.com/aerospike-developer-blog/aerospike-modeling-iot-sensors-c74e1411d493) (Medium, Dec 2019)  
**Code:** [aerospike-examples/modeling-iot-sensors](https://github.com/aerospike-examples/modeling-iot-sensors)

This example applies the **Foundational Concepts** above: schemaless namespace/set/record model, key-driven partition distribution, target record size (few KiB, under 8 MiB), primary index cost (64 bytes per record), and efficient batch reads.

### Use case

One million sensors, one temperature reading per minute for a year (525,600 readings per sensor per year). Contrast with a Cassandra-style model where each (sensor-id, timestamp) is one row: that yields 526 billion rows and forces full scans for most access patterns. Aerospike models with **fewer, larger records** to enable single-read and batch-read access without scanning — aligning with the foundation's guidance to **consolidate data into records of a few KiB** and avoid many tiny records that inflate primary index cost.

### Data model

| Aspect | Choice | Foundational link |
|--------|--------|-------------------|
| **Namespace** | `test` (example) | Top-level container; one storage engine and policies for all sensor data. |
| **Set** | `sensor_data` | Logical grouping within the namespace; scans and secondary indexes can be scoped to this set. |
| **Record key** | String: `sensor{sensor_id}-{YYYY-MM-DD}` (e.g. `sensor1-2018-12-31`) — one record per sensor per day. | Key (or its digest) determines **partition** (RIPEMD160 → 12 bits) and thus distribution across nodes. Key design drives access: one key per sensor-day. |
| **Bins** | Single bin `t` (list of readings for that day). | Schemaless: no fixed schema; bin name + value (here, a list type). |

**Bin `t` (readings):**

- **Type:** List (CDT list).
- **Element type:** 2-element list `[minute_since_midnight, temperature_x10]`.
  - Minute: int, 0–1439 (minutes since midnight for that day).
  - Temperature: int, stored as value×10 (e.g. 62.1° → 621) for MessagePack compaction.
- **List type:** Unordered (O(1) append); range reads use `list_get_by_value_interval` on the list.
- **Size:** Up to 1440 tuples per record (~10KB uncompressed per record; compression reduces significantly). Keeps each record in the **target range of a few KiB** and well under the **8 MiB** record maximum; avoids the primary index cost of one 64-byte entry per reading.

### Foundational concepts applied

- **Record granularity and size:** One record per sensor per day (not per reading) keeps record count and primary index metadata manageable and fits the "few KiB per record" target. Batch reads then operate over a known, bounded key set.
- **Key design and partition distribution:** Key format `sensor{id}-{date}` makes partition placement deterministic; same sensor across days spreads across partitions (and nodes), avoiding hot spots while allowing efficient batch get by key list.
- **No schema:** The "schema" (namespace, set, key format, bin `t` as list of [minute, temp_x10]) is an application-level contract; the server does not enforce it.
- **Batch reads:** When the key set is known (e.g. 365 keys for one sensor/year, or 1M keys for all sensors one day), batch get replaces scans and aligns with Aerospike's efficient batch read behavior.

### Design decisions

- **Roll-up by day:** One record per sensor per day instead of one record per reading. Enables one read for “one sensor, one day” and batch reads for “one sensor, full year” or “all sensors, one day” without scanning. Directly applies the foundation's **record consolidation** guidance and keeps **primary index cost** (64 bytes per record) reasonable.
- **Compact in-record encoding:** Minute since midnight (not full epoch) and temperature×10 (int) keep MessagePack small (e.g. 7 bytes per tuple). Supports the target record size and good compression within the **schemaless bin value** model.
- **CDT list choice:** Unordered list favors fast append; ordered list would favor range operations; both support the same ops with different performance.
- **Key in application key (send key):** Key is sent to server (`POLICY_KEY_SEND`) so key can be used in batch and for debugging.

### API patterns used (in code)

- **Single-record write (append):** `client.operate(key, [lh.list_append_items("t", readings)], policy=policy)` — append a chunk of `[minute, temp_x10]` tuples to bin `t`.
- **Single-record read:** `client.get(key)` — get full day’s list.
- **In-record range:** `list_get_by_value_interval("t", ..., [starts, aerospike.null()], [ends, aerospike.null()])` via `operate()` — get 3 hours of readings within one record (e.g. 8–11am).
- **Batch read (one sensor, one year):** Build 365 keys, `client.get_many(keys)`.
- **Batch read (all sensors, one day):** Build 1000 keys (e.g. `sensor{i}-06-19`), `client.get_many(keys)`.
- **Scan with filter:** `client.scan(namespace, set)` (or query) with a filter expression (e.g. digest modulo for ~0.25% sample); `scan.foreach(callback)`.

### Takeaways for our standard example

- **Record key design** can encode entity + time slice (e.g. `sensor{sensor_id}-{date}`) so access patterns map to single gets or bounded batch gets.
- **CDT lists** support high-density time-series within one record (append, value-interval reads); list order (ordered vs unordered) is a performance tradeoff, not a capability limit.
- **Compaction in value:** Store minimal representation (minute not epoch, int not float) to reduce size and improve read/write and compression.
- **Batch reads** replace scans when the key set is known (e.g. all keys for “sensor X, year” or “all sensors, day Y”).
- **Filter expressions** on scan allow server-side filtering without reading full records (e.g. digest modulo for sampling).

---

## Source 2: User Profile Store (User Segmentation)

**Article:** [Aerospike Modeling: User Profile Store](https://medium.com/aerospike-developer-blog/aerospike-modeling-user-profile-store-dc3c1464b60a) (Medium, Feb 2020); same article: [DEV Community](https://dev.to/aerospike/aerospike-modeling-user-profile-store-4k8k). See also [Building a Fast User Profile Store for Real-Time Bidding](https://aerospike.com/blog/user-profile-store-rtb) (Aerospike blog).  
**Code:** [aerospike-examples/modeling-user-segmentation](https://github.com/aerospike-examples/modeling-user-segmentation)

This example applies the **Foundational Concepts**: one record per user (not per segment) to avoid primary index blow-up, schemaless bin as map, and single-read + CDT map operations for in-record querying. Contrast with a "one record per user-segment" design, which would create huge record counts and primary index cost (e.g. 50B users × 1000 segments → 50T records → ~3 PB of index metadata if stored as separate records).

### Use case

User profile store for **audience segmentation** (e.g. AdTech, real-time bidding). Each user (cookie/device ID) has many **segments** (interests); segments have **per-segment expiry** (e.g. 30–60 days). Access pattern: given a user ID, read all (or filtered) segments in &lt;10 ms for bid decisions. Segments must be expirable at the element level; Aerospike TTL is record-level only, so expiry is implemented in the value and maintained by the application.

### Data model

| Aspect | Choice | Foundational link |
|--------|--------|-------------------|
| **Namespace** | `test` (example) | Top-level container; one storage engine and policies. |
| **Set** | `profiles` | Logical grouping; scans (e.g. background trim) can be scoped to this set. |
| **Record key** | User ID string (e.g. `u1`, `"1234"`) — **one record per user**. | Key determines **partition**; one key per user gives one read for all segments and keeps record count = user count. |
| **Bins** | Single bin `u` (or `segments`): **map** from segment_id → value. | Schemaless: bin holds one map; no fixed schema for value structure beyond application contract. |

**Bin `u` (segment map):**

- **Type:** Map (CDT map). Key = segment ID (integer, e.g. 0–81999 in the example). Value = list `[segment_ttl, attributes]`.
  - **segment_ttl:** Expiry encoded as integer — in the code, **hours since a fixed epoch** (e.g. 2019-01-01) for compact storage and range queries.
  - **attributes:** Optional map/list for per-segment metadata (e.g. source, flags); can be empty `{}`.
- **Map order:** Options are UNORDERED, K-ORDERED, KV-ORDERED; all support the same ops with different performance. For SSD namespaces, **K-ORDERED** is often best; value-ordered (or key-value-ordered) enables efficient `map_get_by_value_range` for "all segments with TTL ≥ now" and `map_remove_by_value_range` for trim. In DB 7+, value index can be persisted to avoid re-sorting on SSD.
- **Size:** One record holds all segments for one user. MessagePack serialization reduces size; EE compression (e.g. Zstandard) can yield ~0.69–0.25 ratio depending on data. Large users (e.g. thousands of segments) still fit under 8 MiB per record.

### Foundational concepts applied

- **Record granularity and primary index cost:** One record per user (not per segment) keeps record count equal to user count. Avoids the alternative of one record per (user, segment), which would multiply records by average segments per user and explode primary index size (e.g. 50B users × 1K segments → 50T records → ~3 PB of 64-byte index entries).
- **Key design:** Key = user ID; single get returns the full profile. No need to discover segment keys or batch by unknown key set; no secondary index required for "all segments for user X."
- **In-record expiry:** Record-level TTL is not granular enough. Storing expiry as the first element in each map value allows **map_get_by_value_range** for "segments not yet expired" and **map_remove_by_value_range** for cleanup. Application owns expiry logic; Aerospike CDTs provide the range operations.
- **Batch reads:** When the set of user IDs is known (e.g. batch bid request), batch get by user keys is efficient; no scans.

### Design decisions

- **One record per user, map of segments:** Single read fetches all segments; record count scales with users, not segments. Aligns with **record consolidation** and **primary index cost** (64 bytes per record).
- **Expiry in value, not record TTL:** Segment TTL stored as first element in map value; retrieval via `map_get_by_value_range(bin, [now, null], [infinity, null])`; trim via `map_remove_by_value_range`. List wrapper in range args required so Aerospike compares list-to-list.
- **Compact TTL encoding:** Hours (or similar) since a fixed epoch keeps values small and supports range queries; same idea as IoT (minute since midnight, int not float).
- **Background trim:** Stale segments can be removed on write (when adding a segment) or via a **background scan** with the same `map_remove_by_value_range` op, optionally throttled (e.g. records per second per node) to avoid overloading the cluster.

### API patterns used (in code)

- **Upsert one segment:** `client.operate_ordered(key, [mh.map_put("u", segment_id, [segment_ttl, {}]), ...])`.
- **Upsert multiple segments:** `mh.map_put_items("u", {segment_id: [ttl, {}], ...})`.
- **Get segments by TTL:** `mh.map_get_by_value("u", [ttl, aerospike.CDTWildcard()], MAP_RETURN_KEY_VALUE)` or `map_get_by_value_range` for "TTL ≥ now" (non-expired).
- **Update segment TTL in place:** `lh.list_increment("u", 0, 5, ctx=[cdt_ctx_map_key(segment_id)])` — increment first element of the list value at that map key.
- **Count segments in key range:** `mh.map_get_by_key_range("u", 8000, 9000, MAP_RETURN_COUNT)`.
- **Trim expired (single record):** `mh.map_remove_by_value_range("u", [0, null], [end_ttl, null], ...)` then `mh.map_size("u")`.
- **Trim expired (namespace/set):** As of 4.7 (CE and EE), the remove op can be attached to a **background scan**; `scan.add_ops([mh.map_remove_by_value_range(...)])`, `scan.execute_background()`, then `client.job_info(job_id, JOB_SCAN)` to wait. Trim is not recommended on every read—run periodically (e.g. hourly) or via background scan.

### Takeaways for our standard example

- **One record per entity (user), many items in a CDT (map)** avoids record explosion and primary index cost when the natural access is "get everything for this key." Same consolidation principle as IoT (one record per sensor per day) but with a map instead of a list.
- **Application-level expiry** when record TTL is too coarse: store expiry in the value (e.g. first element of list), use **map_get_by_value_range** / **map_remove_by_value_range** for read and trim. Value-ordered (or key-value-ordered) map enables this; in DB 7+ persist the value index to reduce CPU on SSD.
- **Compact time encoding** (hours since epoch) supports range queries and keeps bins small.
- **Background scan + operation** for cluster-wide trim: one scan with `map_remove_by_value_range` op, optionally rate-limited per node.

---

## Source 3: Document-Oriented Modeling Workshop (Summit '20)

**Workshop:** [Document-Oriented Data Modeling in Aerospike](https://www.youtube.com/watch?v=7rVk0WCRJtQ) (Aerospike Summit '20)  
**Code:** [aerospike-examples/aerospike-document-modeling-workshop](https://github.com/aerospike-examples/aerospike-document-modeling-workshop)

Companion code for the Summit '20 workshop. Builds on the same foundational concepts and reinforces the IoT and CDT patterns already captured in Source 1 and the CDT API reference. The workshop frames the **record as a document**: one record key, multiple bins as document "fields," with optional nested CDTs (map of lists, list of maps). Prerequisites: Aerospike ≥ 4.7, Python ≥ 3.5, aerospike client ≥ 3.10.0.

### Record as document (1)

- One record = one logical document. Key identifies the document (e.g. `("test", "workshop", "record-as-document")`).
- Bins are document fields: scalar (`i`), list (`l`), map (`m`). Multiple bins updated in one `operate_ordered` (e.g. increment, list_append, map_put_items).
- Record can be read by key or by digest (`get_key_digest` → get with tuple `(ns, set, None, digest)`). Send key policy so key is available in batch and for debugging.
- Map policy: `map_set_policy("m", {"map_order": aerospike.MAP_KEY_ORDERED})` for K-ordered map when order matters.

### List ordering and uniqueness (2)

- **Ordered list:** `list_set_order("scores", aerospike.LIST_ORDERED)` so elements are stored in value order. Supports rank-based and value-interval ops.
- **Merge with uniqueness:** `list_append_items(..., list_policy={ list_order: LIST_ORDERED, write_flags: LIST_WRITE_ADD_UNIQUE | LIST_WRITE_PARTIAL | LIST_WRITE_NO_FAIL })` merges new elements, deduplicates by value, keeps list ordered. Partial + no_fail avoid failing the op on duplicates.
- **Relative rank:** `list_get_by_value_rank_range_relative("scores", [10.00, null], -1, LIST_RETURN_VALUE, 2, False)` returns the two closest elements to the value 10.0 (from rank -1, i.e. below). Useful for "nearest" or "closest to X" queries on ordered lists.

### Map index vs rank (3)

- **Index** = position in key order (0 = smallest key, -1 = largest key). **Rank** = position in value order (0 = smallest value, -1 = largest value).
- K-ordered map: `map_get_by_index("m", -1, MAP_RETURN_KEY)` (last key), `map_get_by_rank("m", -1, MAP_RETURN_KEY_VALUE)` (largest value and its key). `map_get_by_value("m", 1, MAP_RETURN_KEY)` returns keys whose value equals 1.
- See [aerospike-cdt-api.md](aerospike-cdt-api.md) for map types (unordered, K-ordered, KV-ordered) and return types.

### List of maps: comparison order (4)

- A list can hold **map elements**. Ordering (for ordered list or rank ops) uses Aerospike's map comparison: by element count, then by keys in stored order, then by values when keys match. Workshop stores several small maps in a list and uses `list_get_by_rank` to observe sort order (rank 0 = "lowest" map, rank -1 = "highest").
- Relevant when storing list-of-structs (e.g. list of vehicle maps) and using rank or value-interval ops; see [aerospike-cdt-api.md](aerospike-cdt-api.md) (Order and compare).

### Nested structure: map of lists (5)

- **Top-level bin** is a **map** (e.g. `user`). One map key (e.g. `"cards"`) holds a **list** of card maps (each card: last_six, expires, cvv, zip, default).
- **Context** targets the nested list: `ctx = [cdt_ctx_map_key("cards")]`. List ops (append, get_by_rank, size) use this context so they apply to `user["cards"]`, not the top-level bin.
- **Multi-step context:** To change a field on one list element, use context that selects that element: e.g. `ctxh.cdt_ctx_map_key("cards")` + `ctxh.cdt_ctx_list_rank(-1)` (last card by rank) or `ctxh.cdt_ctx_list_index(1)` (second card). Then `map_remove_by_key("user", "default", ..., ctx=default_card_ctx)` and `map_put("user", "default", True, ctx=second_card_ctx)` to move "default" from one card to another.
- **Create-if-missing:** `map_put("user", "cards", [], { MAP_WRITE_FLAGS_CREATE_ONLY | MAP_WRITE_FLAGS_NO_FAIL })` creates the `cards` key with an empty list if absent. Then `list_append("user", card1, ..., cards_ctx)` adds the first card.
- Same nested-context idea as User Profile Store (Source 2): map key → list value, with context used for list_increment on TTL. See [aerospike-cdt-api.md](aerospike-cdt-api.md) (Nested context).

### IoT modeling and querying (6, 7)

- Aligns with **Source 1: IoT Sensors**: key `sensor1-MM-DD`, bin `t` (list of `[minute, temp_x10]`), `client.get` / `client.get_many` for single-day and multi-day reads. `list_get_by_value_range("t", ..., [starts, null], [ends, null])` for in-record time slice (e.g. 9–11am). Scan with **filter expression** (e.g. digest modulo and integer comparison) for ~1% sampling.
- No new data-model content beyond Source 1; workshop uses the same key design and CDT list patterns.

### Takeaways for our standard example

- **Record as document:** One record key, multiple bins (scalar, list, map) updated in one `operate_ordered`; key or digest for read. Fits "one entity per record" and consolidation.
- **Ordered list + write flags:** Use `LIST_ORDERED` and `LIST_WRITE_ADD_UNIQUE` (with PARTIAL/NO_FAIL) when merging and deduplicating ordered sequences (e.g. time-series or score lists). Relative rank ops (`get_by_value_rank_range_relative`) for "closest to X" on ordered lists.
- **Map index vs rank:** Index = key order; rank = value order. K-ordered (or KV-ordered) map when you need by-key or by-value access; see CDT API for SSD guidance.
- **List of maps:** Comparison order (count, keys, values) matters for rank and value-interval ops on lists of structs.
- **Nested map → list:** Top-level bin = map; a map key's value = list. Use **CDT context** (e.g. `cdt_ctx_map_key("cards")`) to run list ops on that list; use context with list_rank or list_index to target one element for map_put/map_remove. Complements path expressions (8.1.1+): context is fixed path; path expressions add expression-based iteration and filtering.

---

## Summary: Data modeling learnings

Consolidated takeaways for defining the standard example application's data model.

### Foundation

- **Schemaless:** No server-enforced schema. The application-level contract is namespace, set(s), key format, bin names, and bin types so all clients share the same logical model.
- **Key → partition:** Key (or digest) determines partition (RIPEMD160 → 12 bits). Key design drives data distribution and access patterns.
- **Record size:** Target **a few KiB** per record. Avoid many tiny records (each costs **64 bytes** in the primary index). Stay **under 8 MiB** max. Prefer **many medium-sized records** over one giant record: batch reads are efficient and give better parallelism and granularity.
- **Batch reads:** Use when the key set is known; they replace scans and are a good fit for the "many records, few KiB each" design.

### Indexes

- **Primary index:** One per namespace, automatic, 64 bytes per record. Use record consolidation to keep index cost reasonable.
- **Secondary index:** For lookups by bin value (or expression in 8.1+). Create when query-by-value is needed; consider selectivity.
- **Set index:** For "all records in set S" when the set is small relative to the namespace (~&lt;1%). Prefer over an SI on a "set name" bin.

### Distribution and resilience

- **Partitions:** 4096 per namespace; even distribution across nodes. Replication (e.g. RF2) for availability.
- **Access:** Single-hop to partition master for read/write. Batch and query scatter/gather across nodes as needed.
- **Writes:** 8 MiB streaming write buffer; server throttles by write type when overloaded.

### From applied examples (IoT sensors, user profile store, document workshop)

- **Key design** can encode entity + slice (e.g. `entity{id}-{date}`) or entity only (e.g. user ID); design so access maps to single gets or bounded batch gets.
- **Roll-up / consolidation:** Fewer, larger records (e.g. one record per sensor per day, or one record per user with a map of segments) instead of one record per event or per sub-entity; use CDTs (list, map) to pack data within a record and avoid primary index explosion.
- **Compact encoding:** Minimal representation in bins (int not float, minute/hour since epoch not full timestamp) to reduce size, improve compression, and support range operations.
- **Batch when keys are known;** use scan + filter only when necessary (e.g. sampling). Batch get by user IDs is the main read pattern for profile store.
- **Application-level expiry:** When record TTL is too coarse (e.g. per-segment expiry), store expiry in the value (e.g. first element of map value list) and use **map_get_by_value_range** / **map_remove_by_value_range** for reads and trim; value-ordered map and (in DB 7+) persisted value index make this efficient. Background scan with the same remove op can trim cluster-wide, with optional rate limiting.
- **Record as document, nested CDTs:** One record = one document (key + bins). Bins can be scalar, list, or map; map values can be lists (e.g. user.cards). Use **CDT context** (e.g. `cdt_ctx_map_key("cards")`) to run list ops on nested lists; combine with list_rank/list_index context to update one list element. Ordered list + **LIST_WRITE_ADD_UNIQUE** (and PARTIAL/NO_FAIL) for merge-and-dedup; **list_get_by_value_rank_range_relative** for "closest to X" on ordered lists. Map **index** = key order, **rank** = value order (see CDT API).
- **Expressions:** **Filter expressions** act as the WHERE clause for scans, secondary index queries, and batch reads; use metadata (TTL, since_update_time) in filters when possible to avoid storage reads (two-phase execution). **Operation expressions** allow computed bins and atomic cross-bin operations in a single operate transaction (prefer over UDFs). See [aerospike-expressions.md](aerospike-expressions.md).
- **One-to-many (top-level entities):** When both parent and child are first-class records (association, not aggregation), choose by cardinality: **large "many"** → child stores parent ID + secondary index on that bin; **smaller "many" or lowest latency** → parent stores list of child keys (ordered list, ADD_UNIQUE + NO_FAIL); optionally child stores parent ID for child→parent traversal. When one logical operation updates both records, use **transactions** (v8+, strong-consistency). See Source 4 (use-case-cookbook one-to-many).
- **Many-to-many (top-level entities):** Each entity holds a **list of the other's keys** (no join table, no secondary index). Add association: put new record with its list + batch operate to append this key to each related record's list; remove association: two operate (remove id from list on each side); all multi-record updates in a **transaction** (v8+, strong-consistency). Same list policy (ordered, ADD_UNIQUE, NO_FAIL). For two-record updates, sequential point writes can be faster than batch. See Source 5 (use-case-cookbook many-to-many).
- **Leaderboards (sorted rank at scale):** **KEY_ORDERED map** with **composite key** (e.g. score-playerId) so key order = score order and index = rank; no value index. **Shard by score range** (buckets) to avoid 8 MiB limit and hot keys. Update: remove from old bucket + put in new bucket + update player record (transaction; same-bucket = one operate). "Scores around player": **expressions** (MapExp.getByKey INDEX, MapExp.getByIndexRange) + ExpOperation.read; overflow to adjacent buckets when window spans boundaries. See Source 6 (use-case-cookbook leaderboard).
- **Player matching (similar score + criteria):** Reuse **leaderboard rank window** (getScoresAroundPlayer) as candidate set; no extra index for "similar score." Encode match criteria (online, shield, beingAttackedBy, score threshold) in a **filter expression**; use **filterExp** on batch get and on the "reserve" write so only eligible records are returned and the reserve write is **conditional** (optimistic). Game outcome: transaction updating leaderboard (both players) + both Player records. See Source 7 (use-case-cookbook player-matching).
- **Time series (account–bucket, map by eventId):** One record per **account per time bucket** (e.g. day or N hours); **KEY_ORDERED map** key = eventId (timestamp + uniqueness suffix), value = [deviceId, details]. Time range = getByKeyRange; **device filter** = getByValueList with list [deviceId, WILDCARD]; combine both via **expression** (getByKeyRange then getByValueList) + ExpOperation.read. Pagination via exclusive end or next eventId; multi-day = loop bucket keys, collate. Insert sets TTL; update does not change TTL. High variance in events per bucket → see Time Series with Large Variance. See Source 8 (use-case-cookbook timeseries).
- **Time series with large variance:** When events per bucket vary greatly (bell curve), add a **threshold**; below it use root map only (Source 8). At threshold: **continuation bin** = list of event IDs (each = min eventId for a block); sub-record key = bucketKey:eventId. Insert: find block = largest list value &lt; new eventId, write to that sub-record. Use **filterExp** on root write (no continuation bin, map size &lt; threshold) with **failOnFilteredOut = false** so overflowed buckets don't fail — app uses continuation path. Read: bucket record + batch get sub-records, merge. See Source 9 (use-case-cookbook timeseries-large-variance).
- **Recent events across DCs (top-N per account):** **One record per account** with a KEY_ORDERED map (key = timestamp+txnId, value = txnId) as "top N" index; **one bin per DC** so each DC only updates its bin (XDR replicates bins; no last-writer-wins on same bin). Write: put + removeByIndexRange(-N, INVERTED). Read: **merge both bins in expression** (MapExp.putItems of DC1 and DC2 maps), getByIndexRange(-(N+k)) on merged map, batch get txn records, return up to N (over-fetch k to tolerate XDR lag). See Source 10 (use-case-cookbook top-transactions-across-dcs).
- **Advanced expressions (cookbook techniques):** **IN (value in list bin):** ListExp.getByValue(EXISTS, Exp.val(x), Exp.listBin("b")). **IN reverse (bin in passed list):** ListExp.getByValue(EXISTS, Exp.stringBin("field") or other type-specific bin, Exp.val(list)). **List as set:** UNORDERED list + ADD_UNIQUE + NO_FAIL. **Multi-step conditional write on one bin:** Exp.let/Exp.def chain results (each step returns modified copy); ExpOperation.write final var to bin — one operate, dependent logic atomic; reuse expensive sub-expressions via def/var. See Source 11 (use-case-cookbook advanced-expressions).

### Path expressions and list-of-structs (example application)

**Path expressions** (Aerospike DB 8.1.1+, preview; see [aerospike-path-expressions.md](aerospike-path-expressions.md)) allow modeling a **list of structs** in one bin (e.g. a user’s vehicles as a list of maps with `make`, `color`, `license`) and querying or indexing by a field. Without path expressions, the only way to index or filter by such a field is to denormalize (e.g. a separate bin holding a list of license strings). With path expressions:

- **Filter server-side:** Iterate over the list of vehicle maps and return only vehicles where `license` equals a given string.
- **Index by field:** Build an expression index over the vehicle map structures and index `license` (or another field), enabling queries like “all users with license plate X” without a second bin.

Useful for the standard example application when the model includes repeated structured data per user (e.g. vehicles, addresses, devices): one bin as list of maps, path expressions for filtering, expression indexes for lookup by field.

---

## Source 4: One-to-many (top-level entities) — Use Case Cookbook

**Source:** [aerospike-examples/use-case-cookbook](https://github.com/aerospike-examples/use-case-cookbook) — [Managing One to Many relationships](https://github.com/aerospike-examples/use-case-cookbook/blob/main/UseCases/one-to-many-relationships.md)  
**Code:** [OneToManyRelationships.java](https://github.com/aerospike-examples/use-case-cookbook/blob/main/source/src/main/java/com/aerospike/examples/onetomany/OneToManyRelationships.java)

Use case: two related entity types where one has many of the other; **both entities have business value in their own right**, so the "many" are not aggregated (nested) inside the "one." Examples: Department has many Employees (each employee in one department); Agent has many Listings (each listing has one agent). Query patterns can run **parent → child** (e.g. all listings for an agent) and **child → parent** (e.g. listing → agent). This is association, not aggregation.

### Two implementation options (cardinality-driven)

| Situation | Approach | Rationale |
|-----------|----------|-----------|
| **"Many" is very large** (thousands or more) | Child record stores parent ID; create a **secondary index** on that parent-ID bin. | Query by parent ID to discover all child keys; then batch get children. Same idea as relational FK + index. Keeps parent record small and avoids storing huge key lists. |
| **Smaller cardinality** or need **fastest retrieval** of the "many" | Parent record stores a **list of child keys** (in a bin). Optionally, child stores parent ID for reverse lookup. | One read of parent yields all child keys; batch get by those keys. No secondary index query; minimal latency when cardinality is modest. |

The cookbook implements the second approach (parent holds list of child keys) with **bidirectional** references so both traversal directions are supported.

### Data model (cookbook example: Agent, Listing)

| Entity | Set | Record key | Key bins | Purpose |
|--------|-----|------------|----------|---------|
| **Parent (Agent)** | `agent` | `agentId` (long) | `listings` (list of listing IDs) | One record per agent; list of child keys for "all listings for this agent." |
| **Child (Listing)** | `listing` | `listingId` (string) | `agentId` (long), plus address/description/etc. | One record per listing; parent reference for "which agent lists this." |

- **Parent → child:** Get agent (e.g. bin `listings` only) → use returned IDs as keys → batch get listing records.
- **Child → parent:** Get listing → read `agentId` → get agent by key.

Both entities are **top-level records** (own keys, own sets). No embedding of one in the other.

### Cross-record updates and transactions

Any operation that **updates both** parent and child (e.g. add listing, delete listing) must keep them consistent. The cookbook uses **Aerospike transactions** (v8+, namespace with strong-consistency) so that:

- **Add listing to agent:** (1) Append `listingId` to agent's `listings` bin, (2) Put listing record with `agentId`. Both in one transaction; commit or abort.
- **Delete listing:** (1) Operate on listing: read `agentId` and delete record (single operate), (2) Operate on agent: remove `listingId` from `listings` (e.g. `list_remove_by_value`). Both in one transaction.

If the model were **one-way** (only parent holds list; child does not store parent ID), then adding a listing would touch only the parent record — one write, no transaction. Transactions are needed when **two records** are updated for one logical operation.

### List policy for parent's child-key list

- **Ordered list** (`ListOrder.ORDERED`) for predictable iteration and potential range/rank use.
- **Set-like behavior:** `ListWriteFlags.ADD_UNIQUE | ListWriteFlags.NO_FAIL` so the same child ID is not duplicated, and duplicate add does not fail the operation (NO_FAIL). For append-items with dedup, `PARTIAL` can be used as well.
- List is top-level in a bin; Aerospike creates it on first append. For nested CDTs, create the list explicitly before appending.

### API patterns used (cookbook)

- **Append child key to parent:** `ListOperation.append(listPolicy, "listings", Value.get(listingId))` with ordered + ADD_UNIQUE + NO_FAIL.
- **Get child keys from parent:** `client.get(readPolicy, agentKey, "listings")` (single bin); then `client.get(batchPolicy, listingKeys)` for batch read of child records.
- **Remove child key from parent:** `ListOperation.removeByValue("listings", Value.get(listingId), ListReturnType.EXISTS)` (or equivalent).
- **Delete child and get parent id:** `client.operate(writePolicy, listingKey, Operation.get("agentId"), Operation.delete())` — one round-trip to read agentId and delete listing; then remove listingId from agent's list.
- **Transaction retry:** On `MRT_BLOCKED`, `MRT_EXPIRED`, `MRT_VERSION_MISMATCH`, `TXN_FAILED`, abort and retry (with optional short backoff).

### Takeaways for our standard example

- **Top-level one-to-many:** When both parent and child are first-class entities (separate sets, separate keys), choose by cardinality: (a) **large "many"** → child stores parent ID + secondary index on that bin; (b) **smaller "many" or lowest latency** → parent stores list of child keys; optionally child stores parent ID for reverse lookup.
- **Bidirectional vs one-way:** Storing parent ID on the child enables child→parent traversal and allows deletes to find the parent (to remove the key from the parent's list). One-way (list only on parent) avoids transactions on add but limits navigation and delete handling.
- **Consistency:** When both records are updated for one logical operation, use **transactions** (v8+, strong-consistency namespace) and retry on transient transaction failures.
- **Parent's list of keys:** Use an **ordered list** with **ADD_UNIQUE** and **NO_FAIL** (and optionally **PARTIAL** for multi-value append) to keep a set-like list of child keys; one read of the parent plus batch get by key replaces a secondary-index query when cardinality is modest.
- **Batch read:** After reading the parent's key list, batch get child records by key; for large batches, parallel batch (e.g. `maxConcurrentThreads`) can be used. Aligns with the foundation's **batch reads when key set is known**.
- **Contrast with aggregation:** If the "many" were only ever accessed via the parent (e.g. addresses of a customer), they could be nested in the parent (e.g. list of maps in one record). One-to-many with **two top-level entity types** is for when both sides are queried independently or in both directions.

---

## Source 5: Many-to-many (top-level entities) — Use Case Cookbook

**Source:** [aerospike-examples/use-case-cookbook](https://github.com/aerospike-examples/use-case-cookbook) — [Managing Many to Many relationships](https://github.com/aerospike-examples/use-case-cookbook/blob/main/UseCases/many-to-many-relationships.md)  
**Code:** [ManyToManyRelationships.java](https://github.com/aerospike-examples/use-case-cookbook/blob/main/source/src/main/java/com/aerospike/examples/manytomany/ManyToManyRelationships.java)

Use case: two entity types where **each** has many of the other; both entities have business value in their own right. Example: Customer has many Accounts, and each Account has many owners (Customers). In SQL this would be a join table plus indexes; in Aerospike we avoid both by storing **lists of related keys** on each record. No secondary indexes and no separate join records are required.

### Data model (cookbook example: Customer, Account)

| Entity | Set | Record key | Key bins | Purpose |
|--------|-----|------------|----------|---------|
| **Customer** | `customer` | `custId` (string) | `accounts` (list of account IDs) | One record per customer; list of account keys for "all accounts this customer owns." |
| **Account** | `account` | account id (e.g. UUID) | `owners` (list of customer IDs) | One record per account; list of customer keys for "all owners of this account." |

- **Customer → accounts:** Get customer's `accounts` bin → batch get account records by those keys.
- **Account → owners:** Get account's `owners` bin → batch get customer records by those keys.

Both sides hold the association; traversal works in either direction with a single read plus batch get. Same list policy as one-to-many: **ordered list**, **ADD_UNIQUE**, **NO_FAIL** for set-like, deduplicated key lists.

### Adding an association (e.g. add account with owners)

One logical operation updates **1 + N records**: the new account record and every owner's record.

- **In a transaction:** (1) Put the account record with bins (id, accountName, balanceInCents, dateOpened, **owners** = list of owner customer IDs). (2) For each owner customer, append the account ID to that customer's `accounts` bin (same list policy: ordered, ADD_UNIQUE, NO_FAIL). The cookbook uses a **batch operate** over the customer keys with `ListOperation.append(..., "accounts", Value.get(accountId))` so all owner updates are sent in one batch. Commit or abort the transaction so the account and all owner lists stay consistent.

### Breaking an association (remove one customer–account link)

Removing the link updates **two records**: the customer and the account.

- **In a transaction:** (1) Remove the account ID from the customer's `accounts` list (`list_remove_by_value`, return EXISTS to confirm presence). (2) Remove the customer ID from the account's `owners` list (same op). If either side did not contain the ID, throw (referential integrity). The cookbook does these as **two sequential operate** calls (not a single batch) and notes that for very small operation counts (e.g. two writes), sequential point writes can be faster than batch due to batch startup overhead; batch pays off for larger counts.

### Querying and derived relationships

- **Direct traversal:** "Accounts for this customer" or "Owners for this account" match the one-to-many pattern: get one bin (the ID list), then batch get by those keys. Optionally request only needed bins to reduce payload.
- **Related entities (e.g. "customers who share an account with this customer"):** Get customer's `accounts` bin → batch get those account records (e.g. `owners` bin only) → flatten all owner IDs, exclude self, group/count. All read-only; no transaction required unless consistency across the read is required.

### API patterns used (cookbook)

- **Put new entity with association list:** `client.put(writePolicy, accountKey, new Bin("owners", ownerIds), ...)` then batch append account ID to each owner's `accounts` bin within the same transaction.
- **Batch append to many records:** `client.operate(batchPolicy, null, customerKeys, Operation.array(ListOperation.append(setLikeListPolicy, "accounts", Value.get(accountId))))` — one batch to update all owner records.
- **Remove from both sides:** `ListOperation.removeByValue("accounts", Value.get(accountId), ListReturnType.EXISTS)` on customer key; `ListOperation.removeByValue("owners", Value.get(customerId), ListReturnType.EXISTS)` on account key; both in one transaction, with validation that both returned true.

### Takeaways for our standard example

- **Many-to-many without join table or SI:** Each entity record holds a **list of the other entity's keys**. Traversal in either direction is one read (to get the list) plus batch get. No secondary index and no separate "join" set; CDTs replace the join table.
- **Consistency:** Any operation that updates more than one record (add account + update all owners, or remove link on both sides) should run in a **transaction** (v8+, strong-consistency) so both sides stay in sync.
- **Same list discipline as one-to-many:** Ordered list, ADD_UNIQUE, NO_FAIL for key lists on both sides to avoid duplicates and to tolerate duplicate-add safely.
- **Add pattern:** One new record (e.g. account) with its list of related keys (owners); then batch operate to append this record's key to each related record's list. Both in one transaction.
- **Remove-association pattern:** Two operate calls (remove id from list on each side) in one transaction; use return type EXISTS to verify the link was present and to enforce referential integrity.
- **Batch vs sequential:** For a small number of cross-record updates (e.g. two), sequential point operations can be faster than batch; for larger counts, batch operate is beneficial.
- **Derived/aggregate queries:** More complex reads (e.g. "related customers") are built from the same primitives: get one side's list, batch get the other side's records (or specific bins), then process in application code. Read-only flows often do not need transactions.

---

## Source 6: Leaderboards — Use Case Cookbook

**Source:** [aerospike-examples/use-case-cookbook](https://github.com/aerospike-examples/use-case-cookbook) — [Leaderboards](https://github.com/aerospike-examples/use-case-cookbook/blob/main/UseCases/leaderboard.md)  
**Code:** [Leaderboard.java](https://github.com/aerospike-examples/use-case-cookbook/blob/main/source/src/main/java/com/aerospike/examples/gaming/Leaderboard.java)

Use case: gaming leaderboards — top scores/players, a player's position, and players directly above/below. Scores update as players play; the structure must support sorted order, insert, and delete. Scale: tens of millions of players, many concurrent updates; a single record would hit the 8 MiB limit and become a hot key.

### Why not a single record or value-ordered map

- **Single sorted structure:** Aerospike has no native sorted list. Maps can be key-ordered or value-ordered. Value-ordered maps require a value index (extra storage and CPU on read/write); key-ordered is natural. Map keys must be unique — score alone is not unique.
- **Single record:** One record holding all scores would exceed 8 MiB at scale and create a hot key under high write load. So the model **shards by score range** (buckets).

### Composite key so key order = score order

Use a **composite map key** so that key order equals score order without a value index: e.g. `score-playerId` (zero-padded) as string. Uniqueness comes from playerId. Map value can be playerId again. **KEY_ORDERED** map → index in the map = rank within that bucket. Tie-breaking: for "most recent score wins" use `score-date-playerId`; for "first to score wins" use a decreasing time component (e.g. time-until-offset) in the key.

### Data model: score buckets + player records

| Component | Set | Record key | Key bin | Purpose |
|-----------|-----|------------|---------|---------|
| **Scoreboard** | `scoreboard` (example) | Bucket id (e.g. 0..N for score ranges 0–24, 25–49, …) | Map: composite key (e.g. `04982-000000001`) → playerId | One record per score bucket; KEY_ORDERED map so index = rank within bucket. |
| **Player** | `player` | playerId | `score`, plus name, etc. | One record per player; score duplicated here and updated when score changes. |

Bucket size (e.g. 25 points per bucket) trades record size and write contention for number of records and boundary handling. Can split buckets or use skip-list style indexing if needed (adds complexity and O(log N) vs O(1) access).

### Updating a player's score

In a **transaction:** (1) If the player had a previous score, remove the old map entry from the **old bucket** (`map_remove_by_key` on composite key). (2) Insert the new map entry in the **new bucket** (`map_put`). If old and new scores fall in the **same bucket**, one operate (remove + put); if **different buckets**, two operates (one on old bucket, one on new). (3) Update the Player record's `score` bin. This keeps the scoreboard and player in sync and avoids hot single-record updates by spreading writes across buckets.

### Querying: scores around a player (rank window)

Target: "N players above and below this player (and optionally the player)."

1. **Bucket:** Determine the bucket from the player's score; use that scoreboard record.
2. **Within-bucket index and range:** Use **expressions** (no full-record read): `MapExp.getByKey(..., MapReturnType.INDEX, ...)` to get the player's index in the bucket map; then `MapExp.getByIndexRange(..., startIndex, count, ...)` to get keys for "lower" and "higher" players. Handle underflow for the low side (index &lt; N → startIndex 0, count = index); high side can request index..index+N+1 (include self). Run both via `ExpOperation.read` so only the computed slices are returned.
3. **Bucket overflow:** If the player is near a bucket boundary, the window may span two (or more) buckets. After reading the home bucket, check if more "lower" or "higher" entries are needed; if so, read from the **previous** bucket (e.g. last K entries via negative index) or **next** bucket (first K entries) and prepend/append to the result. Repeat until the window is filled or no more buckets.
4. **Resolve to players:** Map keys encode score and playerId; parse to get details or batch get Player records for full data.

### API patterns used (cookbook)

- **Map policy:** KEY_ORDERED so index = rank; composite key format e.g. `getMapKey(playerId, score)` → `"04982-000000001"`.
- **Score update (same bucket):** `MapOperation.removeByKey(bin, oldMapKey)`, `MapOperation.put(MAP_POLICY, bin, newMapKey, playerId)` in one operate.
- **Score update (different buckets):** Operate on old bucket (removeByKey), then operate on new bucket (put); then put Player record score.
- **Expressions for rank window:** `Exp.let`, `Exp.def("index", MapExp.getByKey(INDEX, ...))`, `Exp.def("startIndex", Exp.cond(...))`, `Exp.def("count", ...)`, `MapExp.getByIndexRange(KEY, startIndex, count, mapBin)`; `ExpOperation.read("lowerPlayers", ...)`, `ExpOperation.read("higherPlayers", ...)` in one operate.
- **Adjacent bucket read:** `MapOperation.getByIndexRange(bin, negativeOffset, count, MapReturnType.KEY)` for overflow from previous/next bucket.

### Concurrency and consistency

Demo synchronizes "update this player's score" and "show scoreboard for this player" so the score used for the query (bucket + map key) matches the current state. Otherwise a score change during the query could point at the wrong bucket. Transactions could enforce consistency but add locking cost; application-level coordination is used for the read path.

### Takeaways for our standard example

- **Sorted rank structure without value index:** Use a **KEY_ORDERED map** with a **composite key** (e.g. score-playerId) so key order = score order; index in map = rank within the record. Avoids value-index storage and CPU cost.
- **Shard by range (buckets):** Multiple records per "leaderboard," each holding a score range, to avoid 8 MiB limit and hot keys. Bucket size is a tuning knob; exact global rank may require summing counts in lower buckets; top-N is easy (highest bucket first).
- **Score update = remove from old bucket + put in new bucket + update player:** Use a transaction when updating both scoreboard and player; same-bucket update is one operate (remove + put); cross-bucket is two operates plus player put.
- **Rank window (above/below):** Use **expressions** to compute index and index range inside the map (MapExp.getByKey INDEX, MapExp.getByIndexRange) and return only those entries via ExpOperation.read; handle bucket boundaries by reading from adjacent bucket(s) and merging.
- **Tie-breaking:** Encode tie-breaker in the composite key (e.g. date or time-until-offset) so key order reflects business rule (most recent vs first).
- **See also:** [aerospike-expressions.md](aerospike-expressions.md) for expression API; [aerospike-cdt-api.md](aerospike-cdt-api.md) for map order and index/rank.

---

## Source 7: Player Matching — Use Case Cookbook

**Source:** [aerospike-examples/use-case-cookbook](https://github.com/aerospike-examples/use-case-cookbook) — [Player Matching](https://github.com/aerospike-examples/use-case-cookbook/blob/main/UseCases/player-matching.md)  
**Code:** [PlayerMatching.java](https://github.com/aerospike-examples/use-case-cookbook/blob/main/source/src/main/java/com/aerospike/examples/gaming/PlayerMatching.java)

Use case: match players for head-to-head (or similar) by **similar score/ability**, with extra criteria (e.g. not online, no active shield, not already being attacked, score above a threshold). Scale: tens of millions of players, thousands active; matching must be efficient. The cookbook **builds on the Leaderboard** use case (Source 6) for "similar score" and adds **filter expressions** for the remaining criteria.

### Role of the leaderboard

The leaderboard's **getScoresAroundPlayer** (scores above and below a given player) supplies the **candidate set** of similar-score players. No separate index or scan is needed for "similar score" — the bucketed score map plus expression-based rank window already provides a small, ordered set of player IDs. Player Matching then narrows that set by **match criteria** on the Player records.

### Match criteria and filter expression

Criteria that must hold for a defender to be matchable (example):

| Criterion | Bin / logic | Expression idea |
|-----------|-------------|-----------------|
| Defender not online | `online` (bool) | `Exp.not(Exp.boolBin("online"))` |
| No active shield | `shieldExpiry` (timestamp) | `Exp.lt(Exp.intBin("shieldExpiry"), Exp.val(now))` |
| Not being attacked | `beingAttackedBy` (string, empty if none) | `Exp.or(Exp.eq(Exp.val(""), Exp.stringBin("beingAttackedBy")), Exp.not(Exp.binExists("beingAttackedBy")))` |
| Not beginner | `score` | `Exp.gt(Exp.intBin("score"), Exp.val(400))` |

Combined with `Exp.and(...)`. This expression is used as **filterExp** on batch get and on write so that only records satisfying the criteria are returned or updated.

### Matching flow

1. **Candidates by score:** `leaderboard.getScoresAroundPlayer(client, attackerId, score, N)` → list of players (e.g. 20 above, 20 below); exclude the attacker; turn into keys.
2. **Filter to valid defenders:** Batch get those Player records with **batchPolicy.filterExp =** the match-criteria expression. Server applies the filter (can avoid returning or reading full records for non-matching candidates). Collect records that pass.
3. **Choose one:** From the valid set, pick one (e.g. randomly).
4. **Reserve defender:** Operate to set `beingAttackedBy` on the chosen defender. Use **writePolicy.filterExp** with the same criteria so the write succeeds only if the record still matches (e.g. not already attacked in the meantime) — **conditional write** for optimistic concurrency.
5. **Play game:** Apply outcome (e.g. Elo), then in a **transaction**: update both players' scores in the leaderboard (remove from old bucket, put in new), and update both Player records (score, shieldExpiry, clear `beingAttackedBy` for defender).

### Data model (Player record)

Player record holds: `id`, `score`, `userName`, `online`, `shieldExpiry`, `beingAttackedBy`, etc. The leaderboard (score buckets) is separate; score is duplicated on the Player record and kept in sync on updates. Match criteria are all bins on the same record, so one filter expression covers them.

### Concurrency and consistency

- **Reserving a defender:** Setting `beingAttackedBy` to the attacker ID prevents other matchers from selecting the same defender; the filter excludes records where `beingAttackedBy` is non-empty. Clearing it when the battle ends releases the defender.
- **Conditional write:** Write policy filterExp ensures the "set beingAttackedBy" operate only commits if the record still meets criteria (e.g. still not under attack), avoiding double-booking.
- **Game outcome:** Transaction updates leaderboard (both players' bucket entries) and both Player records so scores and state stay consistent.

### Takeaways for our standard example

- **Similar-score matching reuses leaderboard:** Use the leaderboard's rank-window query (scores around player) as the candidate set; no extra index for "similar score." Then apply match criteria on the candidate records.
- **Filter expression for match criteria:** Encode eligibility (online, shield, beingAttackedBy, score threshold) in an **Exp.and(...)** and set **filterExp** on batch get and on the reserve write. Server-side filtering reduces payload and supports conditional writes.
- **Conditional write (optimistic reserve):** When updating a record to "reserve" it (e.g. set `beingAttackedBy`), use **writePolicy.filterExp** with the same eligibility expression so the write succeeds only if the record still qualifies; retry or pick another candidate on failure.
- **Multi-record outcome in a transaction:** Game outcome touches leaderboard (two score updates) and two Player records; run in one transaction (v8+, strong-consistency) so leaderboard and player state stay consistent.
- **See also:** Source 6 (Leaderboards) for score buckets and rank window; [aerospike-expressions.md](aerospike-expressions.md) for filter and expression API.

---

## Source 8: Time Series Data — Use Case Cookbook

**Source:** [aerospike-examples/use-case-cookbook](https://github.com/aerospike-examples/use-case-cookbook) — [Time Series](https://github.com/aerospike-examples/use-case-cookbook/blob/main/UseCases/timeseries.md)  
**Code:** [TimeSeriesDemo.java](https://github.com/aerospike-examples/use-case-cookbook/blob/main/source/src/main/java/com/aerospike/examples/timeseries/TimeSeriesDemo.java)

Use case: devices (e.g. cameras) record events at random times; events belong to an account. Scale: many accounts, few devices per account (&lt;50), relatively low events per device per day (&lt;1000). Queries: insert, update, and **query by date range** with ordered results (asc/desc), **pagination** (forward/backward), optional start/end/cursor, and **filter by one or more device IDs**. Insert sets record TTL (e.g. 14 days); update must not alter TTL. Contrast with Source 1 (IoT sensors), which rolls up by sensor-day with a list of [minute, value]; here the cookbook uses **one record per account per time bucket** with a **map** keyed by eventId for flexible range and device filtering.

### Data model

| Aspect | Choice | Purpose |
|--------|--------|---------|
| **Record key** | `accountId + dateOffset` (e.g. days since fixed epoch) | One record per account per day (or per configurable bucket, e.g. N hours). Empty days have no record. |
| **Bin** | Single bin (e.g. `map`): **KEY_ORDERED map** | Map key = eventId (see below); map value = list `[deviceId, eventDetailsMap]`. Key order = time order. |
| **EventId** | Timestamp (e.g. 13 digits) + random suffix (e.g. 12 digits) | Uniqueness when timestamps collide across devices; sortable so key range = time range. Used for pagination (exclusive end or "next id" for asc). |

Record TTL set on insert (e.g. 14 days); updates use a policy that does not modify TTL so existing expiration is preserved.

### Time range and pagination

- **Key range:** `getByKeyRange(oldestEventId, newestEventId)` on the map returns events in that range. Start inclusive, end **exclusive** in the API — so for "next page" descending, use last returned eventId as exclusive end; for ascending, compute "next" eventId and use as inclusive start.
- **Pagination parameters:** Start timestamp, end timestamp, and optional "last returned eventId" + direction combine into earliestEventId and latestEventId; when eventId is provided, adjust range for asc vs desc (e.g. run from after eventId to end, or from start to eventId exclusive).
- **Multi-record range:** A query spanning several days touches multiple record keys. Compute startRecord and endRecord from day offset; loop over record keys in asc or desc order, run the same operate (expression) on each, and collate results. Order of iteration determines global asc/desc.

### Filtering by device

Map value is a **list** `[deviceId, eventDetailsMap]` (list so Aerospike can compare for value-based filter; map comparison is different). To filter by device IDs use **getByValueList**: pass a list of values to match. Comparing a bare string to a list fails (type mismatch); pass a list that matches the first element and wildcards the rest: e.g. `Value.get(List.of(deviceId, Value.WILDCARD))` per device. Aerospike compares list-to-list and matches the first element; WILDCARD allows the rest to differ.

### Combining time range and device filter

**MapOperation** does not support both getByKeyRange and getByValueList in one call. Use **ExpOperation.read** with an expression: (1) **MapExp.getByKeyRange** (oldestEventId, newestEventId, mapBin) → filtered map by time; (2) **MapExp.getByValueList** (device value list, *result of step 1*) → filter by device. Apply key range first for efficiency (data is key-ordered). Return type KEY_VALUE so the result is the filtered map.

### Insert and update

- **Insert:** Put or operate to add map entry (eventId → [deviceId, details]); set record TTL (e.g. 14 days). Record and map are created on first write if absent.
- **Update:** Operate to put the same key with new value; use write policy that does not update TTL so expiration is unchanged.

### Bucket size and limits

- **Configurable bucket:** Use a time bucket (e.g. `BUCKET_WIDTH_HOURS`) instead of fixed one day so record size can be tuned. Smaller buckets = smaller records, more keys to scan for long ranges.
- **Limitations:** If events per bucket have **high variance** (some buckets tiny, others huge), a single bucket can approach 8 MiB or cause large copy-on-write on every update. The cookbook points to **Time Series with Large Variance** (separate use case) for that scenario.

### Takeaways for our standard example

- **Account–bucket time series:** One record per account per time bucket (e.g. day or N hours); record key = accountId + bucket offset. KEY_ORDERED map: key = eventId (timestamp + uniqueness suffix), value = [deviceId, details]. Key range = time range; pagination via exclusive end or next-id.
- **EventId for uniqueness and order:** Timestamp + random (or synthetic) suffix gives sortable, unique keys for range and cursor-based pagination.
- **Device filter with list value:** Store deviceId as first element of list value; use **getByValueList** with `List.of(deviceId, Value.WILDCARD)` so list comparison matches device. Combine with key range via **expression** (MapExp.getByKeyRange then MapExp.getByValueList) and ExpOperation.read.
- **Multi-bucket query:** Iterate over bucket keys in asc/desc, operate on each with the same expression, collate; iteration order gives global sort.
- **TTL:** Set on insert; use policy that does not update TTL on update so retention is unchanged.
- **High variance:** When events per bucket vary greatly, see Time Series with Large Variance (Source 9). See also Source 1 (IoT sensors) for roll-up-by-day list model.

---

## Source 9: Time Series with Large Variance — Use Case Cookbook

**Source:** [aerospike-examples/use-case-cookbook](https://github.com/aerospike-examples/use-case-cookbook) — [Time Series with Large Variance](https://github.com/aerospike-examples/use-case-cookbook/blob/main/UseCases/timeseries-large-variance.md)  
**Code:** [TimeSeriesLargeVarianceDemo.java](https://github.com/aerospike-examples/use-case-cookbook/blob/main/source/src/main/java/com/aerospike/examples/timeseries/TimeSeriesLargeVarianceDemo.java)

Use case: extends the time series model (Source 8) for **high variance** in events per bucket — e.g. bell curve where many devices have few events but a few have huge volumes (corporate card 100K tx/day, emergency line spiking in a disaster). No fixed upper bound on events per account/day; a single-bucket record would hit 8 MiB or cause large copy-on-write. Same query interface (insert, update, query by date range); only the storage layout changes when a bucket exceeds a **threshold**.

### Two-phase storage per bucket

| Phase | Condition | Storage | Writes |
|-------|-----------|---------|--------|
| **Root only** | Map size &lt; threshold (e.g. MAX_RECORDS_PER_BUCKET) and no continuation bin | All events in the bucket record's root map (same as Source 8) | Single-record operate; no transaction. |
| **Continuation** | Map size ≥ threshold (or continuation already exists) | Root map is emptied; a **continuation bin** (list of event IDs) is added. Each list entry points to a **sub-record** that holds a block of events. | New events go to the appropriate sub-record; root record holds only the index (continuation list). |

Once the continuation bin exists, the root map is no longer used for new events; the list in the continuation bin is **append-only** (existing entries immutable). New blocks may be appended to the list.

### Continuation list semantics

- **Continuation bin:** A list of **event IDs**. Each element serves two roles:
  1. **Sub-record key:** Sub-record primary key = bucket key + `":"` + eventId (e.g. `acct-1:568-1753175980346001295536836`). One Aerospike record per block.
  2. **Block minimum:** The eventId is the **smallest** eventId in that block. To decide which sub-record to insert a new event into: find the **largest** value in the list that is **less than** the new event's id; insert into that block's record. If none (new id is smaller than first list value), use the first block or create a new block depending on design (e.g. first element can be the smallest possible timestamp for the bucket to handle out-of-order arrival).
- **Ordering:** List is ordered by these minimum eventIds so "largest &lt; newId" is a well-defined block. First element is often the bucket's minimum timestamp for out-of-order support.

### Write path: filter to choose root vs continuation

Before writing, the application must choose "write to root map" vs "write to continuation block." Use a **write policy with filterExp** so the server only allows the write when the bucket is still in "root" mode:

- **Expression:** "Can write to root" = continuation bin does **not** exist **and** map size &lt; MAX_RECORDS_PER_BUCKET. Use `Exp.and(Exp.not(Exp.binExists(CONTINUATION_BIN)), Exp.lt(MapExp.size(Exp.mapBin(BIN_NAME)), Exp.val(MAX_RECORDS_PER_BUCKET)))`.
- **failOnFilteredOut = false:** If the filter fails (bucket already over threshold or already has continuation), the write does **not** fail the operation; the server may still perform the write (e.g. to a continuation record), or the client retries using the continuation path. So the client checks filter result or uses a separate code path: when root write is not allowed, determine the correct continuation block from the list, then write to that sub-record (or create a new block and append to the list in a transaction if needed).

When at threshold, the bucket is "sealed": root map is moved or cleared, continuation list is created (in a transaction or two-phase process so the bucket switches format atomically). New inserts then target sub-records only.

### Read path

- **Root only (no continuation bin):** Same as Source 8 — operate with time range (and device) expression on the bucket record.
- **With continuation:** Read the bucket record to get the continuation list (and any remaining root data if hybrid). For the requested time range, determine which list entries (blocks) overlap; batch get those sub-records (keys = bucketKey + ":" + eventId for each). Merge results from root (if any) and sub-records by eventId, apply sort and filters. So reads become **multi-record** when the bucket has overflowed.

### Takeaways for our standard example

- **Threshold-triggered overflow:** When events per bucket have high variance, keep the same bucket key and query API but add a **per-bucket threshold**. Below threshold: single record, root map only (Source 8 style). At threshold: create **continuation bin** (list of event IDs), empty root map; each list element = minimum eventId for a **block** stored in a separate record (key = bucketKey:eventId).
- **Block selection for insert:** Continuation list is ordered by minimum eventId per block. Insert new event into the block whose minimum is the **largest value in the list less than** the new event's id. Enables correct placement for range queries and out-of-order arrival if first list element is bucket minimum.
- **Filter expression for root writes:** Use **filterExp** on write so root-map insert is only allowed when no continuation bin exists and map size &lt; threshold. Use **failOnFilteredOut = false** so when the bucket has already overflowed, the write doesn't fail; application then uses continuation path (write to correct sub-record).
- **Append-only continuation list:** Once a block is added, its list entry is immutable. New blocks append to the list; sub-records are written independently so no single record grows without bound.
- **Reads scale with blocks:** Query over a bucket that has overflowed = read bucket record (continuation list) + batch get the relevant sub-records, merge by eventId. Same time-range and device-filter logic as Source 8 applied to the merged stream.
- **See also:** Source 8 (Time Series) for base model, eventId, and filters; [aerospike-expressions.md](aerospike-expressions.md) for filter/expression API.

---

## Source 10: Recent events across DCs — Use Case Cookbook

**Source:** [aerospike-examples/use-case-cookbook](https://github.com/aerospike-examples/use-case-cookbook) — [Time series data across multiple DCs](https://github.com/aerospike-examples/use-case-cookbook/blob/main/UseCases/top-transactions-across-dcs.md)  
**Code:** [TopTransactionsAcrossDcs.java](https://github.com/aerospike-examples/use-case-cookbook/blob/main/source/src/main/java/com/aerospike/examples/transactionprocessing/TopTransactionsAcrossDcs.java)

Use case: e.g. fraud detection — need the **N most recent transactions** per account (e.g. 50) in low single-digit ms. Transactions can be generated in **two different regions** at once; they arrive out of order (remote region ~100ms+). "Most recent" must be by **transaction timestamp**, not arrival order. **Eventual consistency** between DCs (XDR, no stretch cluster); each DC writes locally and bins replicate independently.

### Single-DC model

| Component | Set / record | Content |
|-----------|----------------|---------|
| **Transaction** | Own set; key = txn id | Full transaction (accountId, timestamp, amount, status, etc.). Record TTL as needed. |
| **Account** | One record per account; key = accountId | One bin: **KEY_ORDERED map**. Map key = composite `timestamp-txnId` (e.g. `%013d-%8s`) for uniqueness and sort order; map value = txn id (or more for filtering). |

**Save:** (1) Write transaction record. (2) Operate on account record: **map_put**(compositeKey, txnId), then **map_removeByIndexRange**(bin, -N, MapReturnType.INVERTED) — keep only the last N entries by key order (KEY_ORDERED so highest timestamp at end; -N = last N; INVERTED = remove everything else). So the account holds a bounded "top N" index.

**Read:** Operate with MapExp.getByIndexRange(VALUE, 0, mapBin) to get the N (or all) values; batch get transaction records by those ids; return in descending order (most recent first).

### Multi-DC: one bin per DC

XDR does **bin-level** replication. If both DCs update the **same bin**, convergence is last-writer-wins and updates can be lost. So use **one bin per DC** (e.g. `txns_dc1`, `txns_dc2`). Each DC writes only to its own bin; XDR ships each bin independently → **eventual consistency** without concurrent writes to the same bin.

- **Write:** Unchanged except bin choice: which DC originated the transaction determines the bin. So DC1 always writes to BIN_DC1, DC2 to BIN_DC2. Same operate: put + removeByIndexRange(-N, INVERTED) on that bin.
- **Read:** Merge both bins in a **read** (expression, no disk write). Build merged map: if both bins exist, **MapExp.putItems**(mapPolicy, Exp.mapBin(BIN_DC1), Exp.mapBin(BIN_DC2)) → single key-ordered map; if only one exists use that bin; if neither, empty map. Then **MapExp.getByIndexRange**(VALUE, -countToUse, mergedMap) to get the last `countToUse` values (e.g. N+3 to allow for replication lag — transaction record may not have arrived yet). Batch get transaction records; skip nulls; return up to N, most recent first.

### Composite key and over-fetch

- **Composite key:** Same pattern as time series: timestamp + txn id (e.g. `"%013d-%8s"`) so key is unique and sortable when timestamps collide across regions.
- **Over-fetch on read:** Request slightly more than N (e.g. N+3) from the merged map so that after batch get, if some transaction records are still in flight via XDR, the application can still return N non-null results.

### Takeaways for our standard example

- **Top-N per entity with eventual consistency across DCs:** Store transactions (or events) in their own set; per-entity record holds a **KEY_ORDERED map** (composite key = timestamp+id, value = id) as a "top N" index. Write: put new entry + removeByIndexRange(-N, INVERTED) to cap size. For **multi-DC**, use **one bin per DC** so each DC updates only its bin; XDR replicates bins independently and no last-writer-wins on a single bin. Read: **merge bins in an expression** (MapExp.putItems of both bin maps), then getByIndexRange(-N) on the merged result; batch get full records; over-fetch slightly to tolerate replication lag.
- **"Most recent" by timestamp:** Composite key orders by transaction time; index range keeps last N. Order is by key (timestamp), not by arrival, so out-of-order and cross-DC writes still yield correct recency.
- **Expression merge is read-only:** putItems in a read expression merges the two maps in memory for the read; it does not write back to the record.
- **See also:** Source 8 (Time Series) for eventId/composite key; Time Series with Large Variance if N or volume grows; XDR/bin-level replication in Aerospike docs.

---

## Source 11: Advanced Expressions — Use Case Cookbook

**Source:** [aerospike-examples/use-case-cookbook](https://github.com/aerospike-examples/use-case-cookbook) — [Advanced Expression usage](https://github.com/aerospike-examples/use-case-cookbook/blob/main/UseCases/advanced-expressions.md)  
**Code:** (techniques demonstrated with Car / uccb_car sample data)

This use case is a **collection of expression techniques** rather than a single business scenario. Expressions are used for filter (record ops, XDR, index values), for writing computed/synthetic values, and for combining multiple dependent steps in one operation. The following patterns are directly relevant to data modeling and querying.

### 1. IN: passed value in a list bin (does the record's list contain X?)

**ListExp.getByValue**(ListReturnType.EXISTS, Exp.val(value), Exp.listBin("binName")) — returns true if the list in the bin contains the given value. Use as **filterExp** on get, batch get, or secondary-index query (e.g. "cars that have feature Sunroof").

**List as set:** Aerospike has no Set type; use a list with **ListOrder.UNORDERED** (or ORDERED) and **ListWriteFlags.ADD_UNIQUE | ListWriteFlags.NO_FAIL** so duplicate adds are ignored and do not fail. Append with that policy gives set-like behavior.

### 2. IN (reverse): bin value in a passed list (is the record's scalar in our list?)

"Record's `color` bin — is it one of Red, Green, Blue?" **ListExp.getByValue** takes (returnType, *value*, *list*). The *list* argument can be any list-typed expression, not only a bin. So pass the **bin as the value** and the **application list as the list**: ListExp.getByValue(ListReturnType.EXISTS, Exp.stringBin("color"), Exp.val(List.of("Red", "Green", "Blue"))). Use as filter for "color IN (Red, Green, Blue)".

### 3. Multiple dependent steps in one operation (let/def and write)

**Operate** can run many Operation objects, but they are **independent** — no passing of results from one to the next. When logic requires step B to use the **result** of step A (e.g. conditionally append to a list, then conditionally append again to the **modified** list), use **Exp.let** and **Exp.def** so that each step binds a variable and later steps use **Exp.var("name")**.

- **Immutability:** During expression evaluation, bin reads are immutable. **ListExp.append**(..., Exp.listBin("features")) does not modify the bin; it returns a **copy** of the list with the item appended. So the pattern is: def("a", cond(..., append(..., listBin("features")), listBin("features"))), def("b", cond(..., append(..., var("a")), var("a"))), ... and the final expression is the last variable. Then **ExpOperation.write**("features", Exp.build(Exp.let(..., Exp.var("last"))), ExpWriteFlags.DEFAULT) writes that result back to the bin. All steps see the same record and chain via variables; only one bin is updated, in one operate call.
- **Reuse of expensive sub-results:** If one sub-expression is expensive (e.g. getByValueRange on a large map), compute it once in a **def** and reuse via **Exp.var()** elsewhere in the same expression so it is not recomputed.

### Takeaways for our standard example

- **Filter "value IN list bin":** Use **ListExp.getByValue**(EXISTS, Exp.val(value), Exp.listBin("bin")) in filterExp. For set semantics on that list, use ADD_UNIQUE and NO_FAIL on append.
- **Filter "bin value IN (app list)":** Use **ListExp.getByValue**(EXISTS, Exp.*Bin("field"), Exp.val(appList)) — bin as value, passed list as list argument.
- **Multi-step conditional update on one bin:** Use **Exp.let** and **Exp.def** to chain conditionals; each step returns a modified copy (e.g. list with item appended); pass the result to the next step via **Exp.var()**; final **ExpOperation.write** writes the last variable to the bin. One operate, one bin; avoids multiple round-trips and keeps dependent logic atomic.
- **See also:** [aerospike-expressions.md](aerospike-expressions.md) for expression API; Source 6 (Leaderboard), Source 8 (Time Series), Source 10 (across DCs) for MapExp/list usage in expressions.
