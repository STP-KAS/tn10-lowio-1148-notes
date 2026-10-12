# Proposed A/B test: lowio vs default on TN10 (not run yet)

**Status: proposal only. No numbers in this repo come from this test yet.**

## Setup
- Two pruned TN10 nodes, same kaspad build (2.1.0 plus PR #1148 head `e3773118`), same machine class, same disk type, same `--ram-scale`, same `--utxoindex` setting.
- Node A: `--rocksdb-preset=default`. Node B: `--rocksdb-preset=lowio`.
- Each node on its own machine (our lesson: one shared disk mixes the two signals).
- Disk guard as in our storms: senders pause at 8 GB free, node STOP at 5 GB, halt at 3 GB.

## Phases
1. **IBD from scratch** on both nodes, same peers. Log per-phase time (proof, headers, UTXO download, UTXO import, block bodies), normalised per unit of work the way the PR does.
2. **Synced, idle**, 30 min baseline.
3. **Modest load**, about 600 tx/s for 60 min at fixed fees (no climbs), announced a day ahead with a UTC window.
4. **Resync**: stop each node 15 min, restart, measure catch-up time and the mempool/fee estimate after catch-up (the #1147 effect).

## Metrics (every 10 s, one row per node, `ab-template.csv`)
- `disk_write_bytes_total`: `/proc/<pid>/io` `write_bytes`
- `compactions_total`, `compaction_cpu_s`: RocksDB stats in the kaspad log / LOG files
- `rss_bytes`: `/proc/<pid>/status` VmRSS
- `db_size_bytes`: `du -sb` of the datadir every 5 min
- `ibd_phase`, `ibd_elapsed_s`, `virtual_daa_score`, `sink_lag_s`, `mempool_size`, `fee_estimate_priority`: RPC

## Pass questions
- Does B write clearly fewer bytes than A under TN10 load (the PR claims ~50% on mainnet IBD)?
- Peak RSS of B minus A, at our `--ram-scale`.
- Do B's pruning bursts still reach the ~16 GB in 30 min we logged on default?
- Any freeze (DAA still >3 min, sink lag >3 min) on either node.
