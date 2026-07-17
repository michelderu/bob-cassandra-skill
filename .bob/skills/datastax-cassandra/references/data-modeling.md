# Cassandra Data Modelling Reference

## Golden rule

Design **one table per query pattern**. If a pattern cannot be expressed as *known partition key + optional range on clustering columns*, you likely need another table, a different key, or a different store.

---

## Four-step modelling process

1. **List access patterns** — for each pattern note read/write frequency, latency budget, and whether strong or eventual behaviour is needed (ties to consistency level choice).
2. **Choose the partition key** — must route each query to **one partition** (or a small, bounded set) and distribute load evenly across the ring. Warn if cardinality is very high or very low.
3. **Define clustering columns** — satisfy range predicates and ORDER BY. Sort order is a **write-time** decision fixed at schema creation via `CLUSTERING ORDER BY`; it cannot be changed without recreating the table.
4. **Duplicate data when needed** — if two patterns need different partition keys, use two tables and accept denormalization. The application owns consistency between them.

---

## Partition health

- **Hot partitions**: a key with very high write/read concentration makes its replicas a bottleneck while other nodes idle.
- **Mitigation — bucketing**: add a time or hash component to the partition key (e.g. `(user_id, day)`) to cap partition size and spread load.
- **Wide partitions**: very large partitions increase read amplification, repair scope, heap, and GC pressure — treat unbounded growth as a design bug.
- Diagnose with: `nodetool getendpoints <ks> <table> <key>` (replica placement) and `nodetool tablestats <ks>.<table>` (partition size, tombstone counts).

---

## Key patterns

| Pattern | Partition key | Clustering key | Notes |
|---|---|---|---|
| **Time-series bucketing** | `(device_id, bucket)` | `timestamp DESC` | Caps partition size; bucket = day/hour depending on volume |
| **Fan-out / dual-write** | Different key per access pattern | As needed | App writes to both tables; e.g. `events_by_user` + `events_by_day` |
| **Lookup table** | The lookup column | Row-unique column | One table per query; data duplicated intentionally |
| **Counter table** | Any valid partition key | Any clustering | `COUNTER` type only; cannot mix with non-counter columns |

---

## Tombstones & delete-heavy workloads

- A **delete** is a **write** of a tombstone marker (LSM-tree; no in-place erase on disk).
- Tombstones must survive for `gc_grace_seconds` (default 10 days) so offline replicas can learn about the delete before compaction discards the data.
- Wide partitions with heavy insert/delete churn accumulate tombstones that slow reads — pair partition sizing with a deliberate TTL strategy.
- Watch `tombstone_warn_threshold` in logs; consider `nodetool tablestats` to track live vs tombstone cell counts.

---

## Secondary indexes & SAI

- **Traditional secondary indexes**: fan out to every node for high-cardinality lookups — poor for "find by email across the whole cluster." Use only for low-cardinality, scoped queries.
- **Storage-Attached Indexing (SAI)** (Cassandra 4.0+, Astra DB, HCD): the recommended secondary index approach. Efficient and scalable; avoids fan-out. Syntax:

```sql
CREATE CUSTOM INDEX IF NOT EXISTS ON keyspace_name.table_name (column_name)
    USING 'StorageAttachedIndex';
```

- **Preference order**: denormalised table with a matching partition key → SAI → traditional secondary index.

---

## Materialised views

- Automatically maintain a second copy of data under a different partition key.
- Add write overhead and can introduce consistency lag between base table and view.
- Use sparingly; prefer explicit dual-write tables for full control.

---

## Pre-flight schema checklist

Before shipping a schema to production:

| Diagnostic question | Implication if "no" |
|---|---|
| Is the exact CQL written for every hot path? | Risk of surprise full scans or extra round-trips |
| Is each read partition-scoped with a known key? | Redesign or add a denormalised table |
| Will any partition grow unbounded over time? | Add bucketing, TTL, or archiving strategy |
| Does RF + CL match latency and durability goals? | Re-tune per use case (see `architecture.md`) |
| Are deletes / TTL heavy in any table? | Plan for tombstone amplification and repair scheduling |
| Is LWT used only where strictly necessary? | Evaluate idempotent alternatives to reduce Paxos overhead |
