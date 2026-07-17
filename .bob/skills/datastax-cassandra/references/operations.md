# Cassandra Operations Reference

## Storage engine

### Write path (LSM)
- Writes are **append-only**: commit log (crash durability) → in-memory **memtable** (sorted) → periodic flush to immutable **SSTables** on disk.
- No random in-place updates — favours sequential I/O and high ingest throughput.

### Read path
- Read checks **memtable** first, then SSTables.
- **Bloom filters** cheaply skip SSTables that cannot contain the target partition.
- **Key cache** speeds index lookups into SSTables.
- Multiple SSTable versions are merged using **last-write-wins (LWW)** by cell timestamp.
- Minimise SSTable reads by choosing the right compaction strategy and keeping caches warm.

### Compaction strategies

| Strategy | Best for | Trade-off |
|---|---|---|
| **STCS** (SizeTieredCompactionStrategy) | Write-heavy workloads | Higher read amplification over time |
| **LCS** (LeveledCompactionStrategy) | Read-heavy, predictable latency | Higher write amplification |
| **TWCS** (TimeWindowCompactionStrategy) | Time-series; old data rarely updated | Inefficient if data is updated after window closes |
| **UCS** (UnifiedCompactionStrategy) | Adaptive; newer releases | Less predictable than explicit strategy choice |

Mismatched strategy is a common performance culprit — match to the table's access pattern.

---

## Self-healing mechanisms

- **Hinted handoff**: if a replica is temporarily down, the coordinator stores a **hint** and delivers it when the peer returns. Hints expire after `max_hint_window` (default 3 hours).
- **Read repair**: on read, if replicas disagree, the coordinator writes the latest version back to stale replicas (policy-dependent; adds read latency).
- **Anti-entropy repair** (`nodetool repair`): compares data between nodes via **Merkle trees** and fixes divergence without relying on a client read. This is the primary mechanism for long-term replica convergence.

---

## Cluster setup checklist

1. Set snitch to `GossipingPropertyFileSnitch` for production.
2. Set replication factor per keyspace — `3` is the minimum for production.
3. Configure `num_tokens` (vnodes) — `16` is a modern default.
4. Tune JVM heap: half of RAM up to 8 GB; use G1GC for heaps > 4 GB.
5. Disable swap: `vm.swappiness=1`.
6. Default consistency level: `LOCAL_QUORUM` for single-DC and most multi-DC deployments.

---

## Repair scheduling

- Run `nodetool repair` at least once every `gc_grace_seconds` (default 10 days) to prevent deleted data from resurrecting on rejoining nodes.
- Prefer **incremental repair** (`-inc`) for large clusters — full repair is I/O-intensive.
- Use **Cassandra Reaper** for automated, sub-range repair scheduling in production.

```bash
nodetool repair -inc my_keyspace          # incremental repair
nodetool repair --full my_keyspace        # full repair (expensive on large clusters)
```

---

## Quick triage order (during an incident)

1. **Availability** — `nodetool status` (UN/DN), client timeouts, dropped messages, unavailables.
2. **Latency** — Read/Write p99, GC pauses, pending compactions, SSTables per read.
3. **Write issues** — MemtableFlushWriter/MutationStage pending, disk iowait, compaction backlog.
4. **Read timeouts** — Tombstones scanned, SSTables per read, GC, iowait.
5. **Inconsistency risk** — Repair progress, hints backlog and delivery failures.

See `health-check.md` for the full five-step health snapshot, metric thresholds, and hints diagnosis.

---

## Common failure diagnoses

| Symptom | Likely cause | Remediation |
|---|---|---|
| Read timeout | Slow replica, GC pause, large partition | Check `nodetool tpstats`, GC logs, partition size |
| Write timeout | Overloaded coordinator or replica | Check write path, compaction backlog, disk I/O |
| Node down / gossip failure | Network partition or OOM | Check system logs, heap usage, network connectivity |
| Tombstone warnings | Excessive deletes | Review TTL strategy, compaction, `tombstone_warn_threshold` |
| `ALLOW FILTERING` in prod | Schema mismatch with query | Redesign table or add a denormalised lookup table |
| Hot partition | Bad partition key cardinality | Add a bucketing dimension to the partition key |

---

## Nodetool command reference

```bash
nodetool status                          # ring health (UN = Up/Normal)
nodetool tpstats                         # thread pool statistics (dropped messages, pending tasks)
nodetool compactionstats                 # in-progress compactions
nodetool tablestats <ks>.<table>         # partition size, SSTable count, tombstone counts
nodetool getendpoints <ks> <tbl> <key>   # which nodes hold replicas for a partition key
nodetool repair -inc <keyspace>          # incremental repair
nodetool flush                           # flush memtables to SSTables
nodetool drain                           # drain node before graceful stop
nodetool describecluster                 # cluster name, partitioner, snitch
nodetool gossipinfo                      # gossip state of all peers
nodetool failuredetector                 # phi-accrual suspicion scores per peer
nodetool netstats                        # streaming / hint delivery progress
```
