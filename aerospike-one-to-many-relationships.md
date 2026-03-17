# Aerospike Data Modeling: One-to-Many Relationships

**Summary:** Patterns for modeling one-to-many relationships in Aerospike: when to keep a list on the parent, when to consolidate children into one record per parent, and when to put the link on the child side and use a secondary index. Choice is driven by **cardinality and who drives the read**. Record size should follow the **Goldilocks Principle** (1–128 KiB ideal; avoid too small for index-to-data ratio, too large for I/O and defrag). Informed by the SubMilliPost data model and project research.

**Status:** Research. Complements [aerospike-data-modeling.md](aerospike-data-modeling.md) (Goldilocks Principle, primary index cost).

---

## 1. Parent-held list of child keys

The parent record stores a list of child IDs (e.g. user has `post_ids` / `note_ids`; post has `repost_note_ids`).

**When it fits:** The "many" side has **modest cardinality** and you typically load the parent when you need the children (e.g. "list my posts", "list notes that repost this post").

**Pros:** One get on the parent gives you the keys; then batch get children. No secondary index. Simple.

**Cons:** The parent record grows with the number of children. If that list gets very large, the parent record is dominated by it and you may hit record-size or write-amplification concerns.

**Rule of thumb:** Use when you're confident the list stays small (e.g. reposts per post; posts/notes per user within your scale).

---

## 2. Consolidation: one record per parent that contains all children

Instead of N child records, you have **one record per parent** keyed by parent id (e.g. `content_comments` keyed by `post:{id}` / `note:{id}`). All "children" live inside that record as a nested map/list (e.g. comment tree plus `comment_paths`, `comment_order`, etc.).

**When it fits:** **High cardinality** and **small children** (e.g. hundreds of comments per post, each comment small).

**Pros:** One primary index entry per parent (not per child); one get returns the whole set; no N keys to maintain on the parent; avoids blowing up the parent record.

**Cons:** That record can get large (must stay under limits); updates to one child are in-place operates with path/context; you need a way to target a single child (e.g. `comment_paths` for comments).

**Rule of thumb:** Prefer when the "many" would otherwise be a long list on the parent or many small records, and you're happy to read/update in one record.

---

## 3. Child-held parent reference + secondary index (inverse lookup)

Children (or a record that aggregates them) hold the parent id or a "who's here" list (e.g. `commenters` on `content_comments`). You create a secondary index on that bin and **query by the "many" side** (e.g. "all content where this user commented").

**When it fits:** You need **inverse access** ("find all X that reference this Y") and you don't want the referenced entity to hold a growing list (e.g. user must not hold every post/note they commented on).

**Pros:** The referenced entity (e.g. user) stays small; you get a bounded set of records to touch (e.g. for user-delete cascade).

**Cons:** Inverse lookups are queries (secondary index) plus batch get/operate, not a single get on the parent.

**Rule of thumb:** Use when cardinality of the "many" is high and you need to find "all parents (or containers) that reference this one thing" (e.g. all content_comments where this user commented).

---

## 4. How to choose: cardinality and who drives the read

| Situation | Pattern |
|-----------|---------|
| **Low cardinality, parent-driven reads** | List on parent (e.g. `repost_note_ids`). |
| **High cardinality, parent-driven reads** | Consolidate into one record per parent (e.g. comments in `content_comments`). |
| **High cardinality, child-driven or inverse reads** | Don't put the list on the "one" side; put the link on the "many" (or its container) and use a secondary index to query (e.g. `commenters` for user-delete). |

**Record size (Goldilocks Principle):** Aim for records in the **1–128 KiB** band. Too small and the 64-byte primary index per record dominates (bad when PI is in memory); too large and read/write plus defrag hurt performance. See [aerospike-data-modeling.md](aerospike-data-modeling.md) § Foundational concepts.

---

## References

- [Aerospike Data Modeling — Project Notes](aerospike-data-modeling.md) — Foundation, indexes, and applied patterns.
- [SubMilliPost Aerospike Data Model Spec](../../../projects/SubMilliPost/architecture/designs/submillipost-aerospike-data-model-spec.md) — Application of these patterns (comments consolidated; reposts as list on post/note; user-delete via SI on commenters).
- [Architect–DocWriter data modeling notes](../../../projects/SubMilliPost/plans/architect-docwriter-data-modeling-notes.md) — Rationale for consolidation, path index, inverse index, and list-on-parent vs consolidate.
