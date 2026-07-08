# Cassandra Health Check Reference

Goal: answer **"is the cluster healthy right now, and where is the pain?"** in under five minutes using only what is on the node or reachable over SSH.

Work top-down: **cluster → node → disk → logs.**

---

## Health snapshot — five-step procedure

### 1) Cluster membership

```bash
nodetool status
```

| State | Meaning | Action |
|---|---|---|
| **UN** | Up, Normal | Healthy member |
| **UJ** | Up, Joining | Bootstrap in progress |
| **UL** | Up, Leaving | Decommission in progress |
| **DN** | Down, Normal | Unreachable — investigate immediately |
| **DJ** | Down, Joining | Failed or stuck join |

Also check: **Load** skew per node (indicates hot node or uneven partitioning); **Owns** alignment with RF; correct Rack/DC topology.

```bash
nodetool describering <keyspace>   # quick ring view
```

### 2) Thread pools and compaction

```bash
nodetool tpstats
nodetool compactionstats
```

In `tpstats`, watch **Pending** and **Blocked** — not just Active:

| Pool | Signals |
|---|---|
| `Native-Transport-Requests` | Client read/write pressure |
| `ReadStage` / `MutationStage` | Coordinator and replica work |
| `CompactionExecutor` | Compactions falling behind |
| `MemtableFlushWriter` | Memtable flush backlog |
| `HintsDispatcher` | Hint delivery backlog |

In `compactionstats`: many pending tasks or a single huge task can explain latency spikes.

```bash
nodetool tablestats <keyspace>.<table>
nodetool getcompactionthroughput
```

### 3) Schema and gossip

```bash
nodetool describecluster
nodetool gossipinfo | head -40
```

Confirm: one cluster name, one partitioner, consistent release across nodes. Stale gossip versions on a node suggest it has been partitioned.

### 4) Storage and OS

```bash
df -h                                         # disk usage — alert > 85% on data/commitlog volumes
du -sh /var/lib/cassandra/data/* | sort -h | tail
iostat -xm 1 3                                # sustained high iowait = disk bottleneck
free -h                                       # any swap use is a problem
uptime                                        # sustained load >> CPU count = saturation
```

| Signal | Concern |
|---|---|
| Data volume **> 85%** | Compaction and writes may fail |
| Commitlog volume full | Node refuses writes |
| High iowait sustained | Disk bottleneck |
| Swap in use | GC and latency suffer |
| Load >> CPU count | CPU saturation |

### 5) Logs — last 15 minutes

```bash
tail -200 /var/log/cassandra/system.log
grep -E 'ERROR|WARN|OutOfMemory|GC overhead|Compaction|hints' \
    /var/log/cassandra/system.log | tail -50
```

Patterns worth stopping for: repeated GC overhead / long pauses; compaction failures or disk full; hints not replaying; streaming errors during repair or bootstrap.

---

## Five-line health summary template

After the checks above, write:

1. **Membership:** (all UN / N nodes down / bootstrap in flight)
2. **Backpressure:** (tpstats/compaction — clear or which pool)
3. **Disk:** (free % on data and commitlog)
4. **Logs:** (clean / top error theme)
5. **Next step:** (e.g. add capacity, repair node X, collect diagnostics)

---

## Key metrics — quick-triage flow

Use this order during an incident:

1. **Availability** — nodes UN/DN, Dropped Messages, Client Timeouts, Unavailables → target all UN, error rates ~0.
2. **Latency** — Read/Write p99, GC pauses, pending compactions, SSTables per read.
3. **Write issues** — MemtableFlushWriter / MutationStage pending, disk iowait, compaction backlog.
4. **Read timeouts** — Tombstones scanned, SSTables per read, GC, iowait.
5. **Inconsistency risk** — Repair progress, hints backlog and delivery failures.

Always correlate latency with GC, compaction, disk IO, and network before tuning CQL or JVM.

---

## Metric thresholds

| Signal | Steady state (planning) | Investigate (incident) |
|---|---|---|
| Nodes up | 100% | Any unexpected down node |
| Client timeouts / unavailables / dropped messages | ~0 sustained | Any sustained non-zero under normal load |
| Read/Write p99 latency | Within SLA | Climbs while load is flat |
| GC pause (young/old gen) | < 200 ms | Frequent pauses > 200–500 ms |
| JVM Old Gen / heap | < ~70% of max sustained | Rising Old Gen + long GC pauses |
| CPU busy | < 70–80% sustained | Pegged with rising latency |
| CPU iowait | < ~5% sustained | High with compaction/disk backlog |
| Disk utilization (data + compaction headroom) | < 60–70% | > ~85% on data or commitlog mount |
| Pending compactions | Low, drains quickly | Grows linearly or never drains |
| SSTables per read / tombstones scanned | Low baseline | Sudden or sustained spike |
| Hints on disk / total hints | Drains after outage | Grows while all nodes UN |
| Full cluster repair | Within RPO (e.g. every 7–14 days) | Stalled or failing segments |

---

## Key metrics cheat sheet

### Client request latency (most important)

| Metric | JMX path | Investigate when |
|---|---|---|
| Read p99 | `ClientRequest` → scope `Read` → `Latency` | p99 climbs while load is flat |
| Write p99 | `ClientRequest` → scope `Write` → `Latency` | Same; often coupled with compaction or disk |
| Range read p99 | `ClientRequest` → scope `RangeSlice` → `Latency` | High on wide-partition scans or large IN queries |
| LWT (CAS) latency | `CASRead`, `CASWrite`, `CASPrepare` | Elevated during lightweight-transaction contention |

JMX latency units are **microseconds** (µs) — divide by 1000 for milliseconds.

```bash
nodetool tpstats           # backpressure shows before latency appears in JMX
nodetool tablestats ks.t   # per-table latency
```

### Client errors

| Metric | JMX scope/name | Meaning |
|---|---|---|
| Timeouts | `ClientRequest` → `Timeouts` | Coordinator waited too long |
| Unavailables | `ClientRequest` → `Unavailables` | Not enough replicas alive for CL |
| Failures | `ClientRequest` → `Failures` | Unexpected errors on request path |
| Dropped messages | `DroppedMessage` (by verb) | Overload or inter-node message timeouts |

Track both **Count** (total since restart) and **OneMinuteRate** (recent trend).

### Thread pools

Watch **Pending** and **Blocked** per pool — see table in §2 above.

JMX: `ThreadPools` → `PendingTasks`, `CurrentlyBlockedTasks`, `ActiveTasks` per scope.

### Compaction and disk

```bash
nodetool compactionstats
nodetool getcompactionthroughput
nodetool tablestats <ks>.<table>    # SSTable count, live data size, tombstones
df -h /var/lib/cassandra
```

JMX: `Compaction` → `PendingBytes` (growing without clearing = problem), `Completed`, `BytesCompacted`.

### JVM and GC

```bash
grep -E 'GC|Pause' /var/log/cassandra/gc.log | tail -20
grep -E 'OutOfMemory|GC overhead' /var/log/cassandra/system.log
```

JMX: `memory` → heap usage; `GarbageCollector` → pause time. Target Old Gen < ~70% of max; pauses < 200 ms.

### Hints (hinted handoff)

A small backlog during a brief outage is normal. A **large or growing** backlog means data may be stale on a replica until hints drain or repair catches up.

| Signal | Investigate when |
|---|---|
| `TotalHints` rising while all nodes UN | Replay stuck — check disk, `HintsDispatcher`, network |
| `TotalHintsInProgress` > 0 long after node recovered | Replay not finishing |
| `HintsDispatcher` Pending/Blocked sustained | Thread pool saturation |
| Hints on disk growing | Check `max_hint_window_in_ms` and hints directory disk space |

```bash
nodetool tpstats                          # HintsDispatcher row
du -sh /var/lib/cassandra/hints
grep -i hint /var/log/cassandra/system.log | tail -30
```

**Hint lifecycle patterns:**

| Pattern | Meaning |
|---|---|
| Spike during outage, drains after UN | Healthy hinted handoff |
| Backlog grows while all nodes UN | Replay stuck — check logs, disk, network |
| Hints after decommission/move | May need repair; check `Hints_for_unowned_ranges` |
| Hint window expired in logs | Some writes lost for that CL — confirm with repair |

**Config to verify** in `cassandra.yaml`:
- `hinted_handoff_enabled` — should be `true`
- `max_hint_window_in_ms` — default 3 hours
- `hints_directory` — must have disk headroom

### Cluster topology and repair

```bash
nodetool status              # UN vs DN
nodetool netstats            # streaming / repair in flight
nodetool describering <ks>   # token balance
```

JMX: `Repair` / `Validation` for repair progress; `ReadRepair` surges suggest replica drift.

Run a full cluster repair within your RPO window (typically every 7–14 days).

### Per-table hot spots

```bash
nodetool tablestats <ks>.<table>     # SSTables per read, tombstones per read, partition size
nodetool tablehistograms <ks>.<table>
```

JMX (`ColumnFamily` per table): read/write rates, SSTables per read, tombstones scanned, live disk space.

---

## Minimum Grafana/dashboard panel set

If you can only keep a few panels:

1. **Nodes Up / Nodes Down**
2. **Coordinator Read / Write Latency** p99
3. **Client Timeouts** + **Dropped Messages** rate
4. **Requests Served** rate (throughput dip detector)
5. **Pending Compactions** + **Compacted Bytes** rate
6. **JVM Old Gen** used + **GC Old Gen** pause rate
7. **Live Data Size** + **Disk Used** + **CPU IOWait**
8. **Total Hints** + **Hints Failed** rate
9. **SSTables Per Read** + **Tombstones Scanned** (when read latency is the symptom)
