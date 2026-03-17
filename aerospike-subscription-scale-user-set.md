# Subscription Scale: User Set and Very Large Subscriber Counts

**Summary:** The current SubMilliPost data model stores `subscribers` and `subscribed_to` as lists on the user record. That design does not scale when users have hundreds of thousands of subscribers (or follow many accounts). This note describes the problem, why a one-record-per-subscription set is a bad fit (primary index ratio), and a **consolidated** scalable alternative that mirrors how we model comments (one record per parent that holds the list).

**Status:** Research. Informs design if product expects very popular users (e.g. hundreds of thousands of subscribers). Current spec assumes dual lists on user; adopt the consolidated alternative if scale requires it.

**See also:** [aerospike-data-modeling.md](aerospike-data-modeling.md) (record sizing, 64-byte PI cost, consolidate into records of a few KiB); [aerospike-one-to-many-relationships.md](aerospike-one-to-many-relationships.md); [SubMilliPost data model spec](../../../projects/SubMilliPost/architecture/designs/submillipost-aerospike-data-model-spec.md) section 4.6 (content_comments — one record per post/note holding all comments).

---

## Problem with lists on user

Today the user record has:

- **`subscribers`** — list of handles of users who subscribe to this user.
- **`subscribed_to`** — list of handles this user subscribes to.

Access patterns: feed (read my `subscribed_to`, batch get those users' post_ids/note_ids); subscribe/unsubscribe (append/remove from both users' lists); user delete (get my subscribers, remove me from each of their `subscribed_to`, clear my lists).

**At scale:** A very popular user might have hundreds of thousands of subscribers (200k × ~15 bytes ≈ 3 MB in one list). The user record would dominate or exceed practical record size. So subscription data must move off the user record.

---

## Why not one record per subscription?

A naive move is a **subscription set**: one record per (subscriber, subscribed_to) pair, key `sub:A:B`, bins subscriber + subscribed_to, with secondary indexes for "subscribers of B" and "who does A follow."

**Primary index ratio:** Aerospike incurs **~64 bytes of primary index metadata per record** (see [aerospike-data-modeling.md](aerospike-data-modeling.md)). A subscription record is tiny: key ~40–60 bytes, two short string bins ~25–35 bytes → **~50–150 bytes per record**. So we’d pay **64 bytes of index in RAM for a ~100-byte record** — a terrible ratio. At millions of subscriptions, index memory would dominate. The research guidance is to **consolidate data into records that are a few KiB in size** and avoid many tiny records that inflate primary index cost.

---

## Consolidated alternative (like content_comments)

Model the relationship like **comments and a post/note**: one record per **parent** that holds the list of **children**, not one record per child.

- **Comments:** One record per post/note (`content_comments` keyed by `post:{id}` / `note:{id}`) holds the whole comment tree. One 64-byte PI entry per post/note; the record can be large (hundreds of comments, hundreds of KiB). Good index-to-data ratio.
- **Subscriptions:** Do the same in **two directions** (we need both "subscribers of B" and "who does A follow" by key):

  1. **Subscribers of a user** — One record per user who has ≥1 subscriber. Key = that user’s handle (e.g. set `user_subscribers`, key = B). Bin = **handles** (list of handles). "List subscribers of B" = one **get by key**. One 64-byte PI per user who has subscribers; the record can be large (e.g. 200k handles ≈ 3 MB). Ratio: 64 bytes index per 3 MB record.

  2. **Who a user follows** — One record per user who follows ≥1. Key = that user’s handle (e.g. set `user_following`, key = A). Bin = **handles** (list of handles). "Who does A follow?" / feed = one **get by key**. One 64-byte PI per user who follows someone; the record can be large (e.g. 10k handles ≈ 150 KB). Ratio: 64 bytes index per record.

So we **denormalize** the relationship into two lists (as today), but store those lists in **dedicated records keyed by one side**, not on the user record and not as one record per subscription. Record count and PI count are **O(users with subscription activity)**, not O(subscriptions). No secondary index is required for "list subscribers of B" or "feed" — both are get by key.

### Sets and keys (consolidated)

| Set | Record key | Bin | Purpose |
|-----|------------|-----|---------|
| `user_subscribers` | subscribed_to_handle (B) | `handles` (list) | One record per user who has ≥1 subscriber. Get by key = list subscribers of B. |
| `user_following` | subscriber_handle (A) | `handles` (list) | One record per user who follows ≥1. Get by key = who does A follow; feed. |

List policies: ORDERED, ADD_UNIQUE, NO_FAIL (as in spec). User record: remove `subscribers` and `subscribed_to`; keep `subscriber_cnt` (and optionally `following_cnt`). Create each record on first add; delete when list becomes empty if desired.

### Access pattern drives the design

**What the UI needs:** (1) **Counts** — "How many subscribers does this user have?" and "How many does this user follow?" (2) **Paginated lists** — "Show this user's subscribers" and "Show who this user follows," in pages (e.g. 20 or 50 per page). We do **not** need to return a full list in one query.

So the data model can rely on:
- **Aggregated counts on the user record** (`subscriber_cnt`, optionally `following_cnt`) for the count display. One read of the user gives both numbers.
- **Consolidated list records** (user_subscribers, user_following) that can grow large, with **list API pagination** (e.g. `list_get_by_index_range` or equivalent) to fetch a slice of subscribers or following at a time. The UI never needs the full list in one response.
- **Infrequent writes** — Subscribe/unsubscribe are list append or list_remove with ORDERED, ADD_UNIQUE, NO_FAIL. These records get big but are not written on every request; the write pattern is manageable.
- **User delete** — The lists are **ordered**; we pop groups of elements off the **front** using **list_remove_by_index_range**(0, batch_size−1) repeatedly. For each batch: (1) **list_get_by_index_range**(0, batch_size−1) to read the handles in that slice; (2) **list_remove_by_index_range**(0, batch_size−1) to remove that batch from the list (front is always index 0 after each pop); (3) for each handle in the batch, operate list_remove_by_value(me) on that user's opposite record (subscribers → their `user_following`; following → their `user_subscribers`) and decrement counts. No need to load an entire multi‑MB list; we process in bounded batches. Same for both my `user_subscribers` and my `user_following` lists.

Documenting this in the PRD (see TPM handoff) keeps product, UI, and data model aligned: counts from user; lists paginated; delete uses paginated list operations.

### Access patterns

| Operation | How |
|-----------|-----|
| **Subscribe (A → B)** | Operate list_append on B's `user_subscribers` record (create with empty list if missing). Operate list_append on A's `user_following` record (create if missing). Operate increment subscriber_cnt on B; optionally increment following_cnt on A. |
| **Unsubscribe** | Operate list_remove_by_value on both records; operate decrement counts. |
| **Feed** | Get `user_following` by key (my_handle). Read handles. Batch get user records (post_ids, note_ids) for those handles. Build feed as today. |
| **List subscribers of B** | Get `user_subscribers` by key (B). Use list API (e.g. list_get_by_index_range) to paginate over handles; return one page at a time to the UI. |
| **List who A follows** | Get `user_following` by key (A). Use list API to paginate over handles; return one page at a time. |
| **User delete** | **Subscribers:** While my `user_subscribers` list is non-empty: list_get_by_index_range(0, batch_size−1) to get the next batch of handles; list_remove_by_index_range(0, batch_size−1) to pop that batch off the front (ordered list, so front stays at index 0); for each handle in the batch, operate list_remove_by_value(me) on that user's `user_following` and decrement their following_cnt. **Following:** Same on my `user_following` list — pop front in batches with list_remove_by_index_range(0, batch_size−1), then remove me from each of those users' `user_subscribers` and decrement subscriber_cnt. Delete my two records. Proceed with rest of user delete. |

User delete is still O(N+M) operates (N = subscribers, M = following), but we are not allocating 64 bytes of index per subscription; we are updating existing consolidated records.

---

## Trade-offs

| One record per subscription | Consolidated (two lists keyed by user) |
|-----------------------------|----------------------------------------|
| 64 bytes PI per ~100-byte record; index memory dominates at scale | 64 bytes PI per user-with-activity; record holds list (few KiB to few MiB); good ratio |
| Two SIs for "subscribers of B" and "who A follows" | No SI; both paths are get by key |
| Put/delete one record per subscribe/unsubscribe | Two operates (append/remove on two lists) per subscribe/unsubscribe |
| User delete = two SI queries + delete records | User delete = two gets + N+M operates to remove me from others' lists |

---

## When to adopt

- **Keep current design** (lists on user) if subscriber and following counts are moderate (e.g. typical Substack-style).
- **Adopt consolidated design** (user_subscribers + user_following, lists keyed by user) if the product expects very popular users (e.g. hundreds of thousands of subscribers) or very large follow lists, and you want a single design that scales without a terrible primary-index ratio.

If adopted, update the data model spec: add `user_subscribers` and `user_following` sets (or equivalent key convention); remove `subscribers` and `subscribed_to` from user; adjust section 7 (access patterns) and section 8 (user delete cascade) accordingly.
