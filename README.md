# IBM Bob skill: Cassandra

![IBM](https://img.shields.io/badge/IBM-052FAD?style=flat-square&logo=ibm&logoColor=white)
![Bob skill](https://img.shields.io/badge/Bob-skill-7c5cd8?style=flat-square)
![Apache Cassandra](https://img.shields.io/badge/Apache%20Cassandra-compatible-3b82d4?style=flat-square&logo=apachecassandra&logoColor=white)

A Bob skill that turns any conversation about Apache Cassandra into an expert-guided session.
It covers the three core areas of Cassandra work: schema design, CQL queries, and cluster operations.

---

## About Bob

[Bob](https://bob.ibm.com/) is an AI-powered developer assistant by IBM for the full **SDLC**. Skills extend Bob with domain-specific expertise —
this one adds deep **Cassandra** knowledge to every conversation.

![why-ibm-bob](./assets/why-ibm-bob.png)

---

## About the Skill

| Area | What you get |
|---|---|
| **Deployment & Architecture** | Compares OSS Cassandra, DataStax HCD, and Astra DB; explains masterless topology, consistent hashing, vnodes, gossip, and phi-accrual failure detection. |
| **CAP & Tunable Consistency** | Explains AP-leaning design, the R + W > RF quorum rule, and when to use ONE / LOCAL_QUORUM / QUORUM / ALL. |
| **Schema & Data Modelling** | Query-driven four-step workflow — elicits access patterns, recommends partition keys, clustering columns, bucketing, and SAI; validates against common pitfalls and outputs annotated `CREATE TABLE` CQL. |
| **CQL Query Writing & Optimisation** | Writes and optimises SELECT / INSERT / UPDATE / DELETE / BATCH statements, runs a 6-point optimisation checklist, and calls out anti-patterns like full-table scans, `ALLOW FILTERING`, and large `IN` clauses. |
| **Storage Engine** | Explains the LSM write path, memtable flush, SSTable read path (Bloom filters, LWW merge), and all four compaction strategies (STCS / LCS / TWCS / UCS). |
| **Self-healing & Repairs** | Hinted handoff, read repair, anti-entropy repair with Merkle trees, incremental repair, and Cassandra Reaper. |
| **LWT & Transactions** | LWT cost (~4× round-trips, Paxos), hot-partition contention traps, and Cassandra 6 Accord full ACID transactions. |
| **Operations & Administration** | Cluster setup checklist, compaction strategy selection, repair scheduling, failure-diagnosis table, pre-flight schema checklist, and extended nodetool command reference. |


---

## Activation

The skill activates automatically whenever the conversation is about Cassandra — no explicit
command needed. You can also invoke it explicitly with:

```
/cassandra
```

---

## Files

```
.bob/skills/cassandra/
  SKILL.md      ← skill instructions loaded by Bob at activation
```

---

## Skill highlights

### Deployment & Architecture
- Compares Apache Cassandra (OSS), DataStax HCD, and Astra DB side-by-side.
- Explains the masterless peer-to-peer ring, consistent hashing, and vnode-based rebalancing.
- Covers gossip (phi-accrual failure detection) and multi-DC topology with `NetworkTopologyStrategy`.
- Frames CAP as AP-leaning with the R + W > RF quorum overlap rule and `LOCAL_QUORUM` guidance.

### Schema & Data Modelling
- Starts from **access patterns**, not relational entities (four-step process).
- Warns on unbounded partitions, hot keys, high/low cardinality, and tombstone-heavy designs with bucketing mitigations.
- Recognises common patterns: time-series bucketing, fan-out dual-write tables, lookup tables, counter tables.
- Covers **SAI (Storage-Attached Indexing)** as the recommended secondary index path and explains when traditional indexes fan out.

### CQL
- Applies keyspace qualification, paging, consistency-level advice, and prepared-statement reminders.
- Flags: full-table scans, large `IN` clauses, read-before-write, secondary indexes on high-cardinality columns, `ALLOW FILTERING` in production.
- Explains `LOGGED` vs `UNLOGGED BATCH` trade-offs and LWT Paxos overhead (~4× round-trips).

### Storage Engine
- Full LSM write path: commit log → memtable → SSTable flush.
- Read path: memtable check → Bloom filter → key cache → SSTable merge (LWW by timestamp).
- Compaction strategy guide: STCS (write-heavy) · LCS (read-heavy) · TWCS (time-series) · UCS (adaptive, newer releases).
- Tombstone lifecycle and `gc_grace_seconds`.

### Self-healing & Repairs
- Hinted handoff, read repair, and anti-entropy repair (Merkle trees).
- Incremental vs full repair, `gc_grace_seconds` cadence, Cassandra Reaper recommendation.

### LWT & Transactions
- LWT cost model (~4× Paxos round-trips) and hot-partition contention trap.
- **Cassandra 6 Accord** full ACID distributed transactions.

### Operations
- Production setup defaults: `GossipingPropertyFileSnitch`, RF=3, 16 vnodes, G1GC, `vm.swappiness=1`.
- Failure diagnosis table covering read/write timeouts, node-down, tombstone warnings, hot partitions, and `ALLOW FILTERING` in production.
- Pre-flight schema checklist (partition scope, unbounded growth, RF/CL alignment, tombstone/LWT risk).

### References
- Content informed by [cassandra-fundamentals](https://github.com/michelderu/cassandra-fundamentals) — an open-source Cassandra architecture and data-modelling course.

---

## Requirements

- Bob (any version that supports `.bob/skills/`)
- No external tools or scripts required — the skill is purely instructional.

---

## Scope

This skill is **workspace-scoped** when stored  in your project directory [`.bob/skills/cassandra/`](.bob/skills/cassandra/).
To make it global, move the directory to `~/.bob/skills/cassandra/`.
