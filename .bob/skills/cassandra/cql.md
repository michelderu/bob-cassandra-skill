# Cassandra CQL Reference

## Writing queries

- Always qualify the keyspace or remind the user to `USE <keyspace>` first.
- **SELECT**: filter on the full partition key; optionally bound clustering columns in prefix order. `ALLOW FILTERING` is acceptable only on small datasets — explain the performance cost and that it often signals the schema is wrong for that query.
- **INSERT**: use `IF NOT EXISTS` only when idempotency is required; explain the Paxos overhead (≈4× round-trips vs a normal write).
- **UPDATE / DELETE**: Cassandra uses upsert semantics; deletes write tombstones, not in-place erases.
- **BATCH**:
  - `LOGGED BATCH` across multiple partitions: use only for atomicity when truly needed.
  - `UNLOGGED BATCH` for the same partition: preferred; no coordinator fan-out overhead.
  - Multi-partition batches increase coordinator load — avoid as a performance tool.
- Use **prepared statements** to avoid repeated parsing overhead.
- Use **paging** (`LIMIT` + `PagingState`) for large result sets — never load unbounded result sets into memory.

---

## Optimisation checklist

Run through these before declaring a query production-ready:

1. Is the **full partition key** present in the WHERE clause?
2. Are **clustering columns** filtered in prefix order (no gaps)?
3. Is `ALLOW FILTERING` avoidable by a schema change or denormalised table?
4. Is **paging** used for potentially large result sets?
5. Is the **consistency level** appropriate for the latency/durability trade-off?
6. Are **prepared statements** used to avoid re-parsing on every call?

---

## Anti-patterns — call out explicitly when spotted

| Anti-pattern | Why it's dangerous | Preferred alternative |
|---|---|---|
| `SELECT * FROM table` with no WHERE | Full cluster scan | Add partition key predicate |
| `ALLOW FILTERING` in production on large tables | Coordinator scans every partition | Redesign schema or add SAI index |
| Secondary index on a high-cardinality column | Fan-out to every node | SAI or a denormalised table with matching partition key |
| Large `IN` clauses | Driver serialises many single-partition requests; coordinator fan-out | Token range queries or application-side batching |
| Read-before-write (read-modify-write) | Adds round-trip latency; race condition without LWT | Idempotent writes; LWT only when compare-and-set is truly required |
| Multi-partition `LOGGED BATCH` for performance | Increases coordinator load; not faster | Write each partition independently |

---

## Useful CQL snippets

```sql
-- Keyspace with NetworkTopologyStrategy (production multi-DC)
CREATE KEYSPACE IF NOT EXISTS my_ks
  WITH replication = {'class': 'NetworkTopologyStrategy', 'dc1': 3};

-- Table with composite partition key and descending clustering
CREATE TABLE IF NOT EXISTS my_ks.events_by_user_day (
  user_id  uuid,
  day      text,
  event_ts timestamp,
  payload  text,
  PRIMARY KEY ((user_id, day), event_ts)
) WITH CLUSTERING ORDER BY (event_ts DESC);

-- SAI secondary index
CREATE CUSTOM INDEX IF NOT EXISTS ON my_ks.events_by_user_day (payload)
    USING 'StorageAttachedIndex';

-- LWT conditional insert
INSERT INTO my_ks.inventory (sku, qty) VALUES ('item-1', 10) IF NOT EXISTS;

-- LWT conditional update
UPDATE my_ks.inventory SET qty = 11 WHERE sku = 'item-1' IF qty = 10;
```
