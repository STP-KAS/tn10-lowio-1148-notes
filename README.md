# tn10-lowio-1148-notes

| Folder / link | What is there |
|---|---|
| [README.md](README.md) | Intro and the What / Why / How briefing |
| [test/AB-TEST.md](test/AB-TEST.md) | Proposed A/B test, lowio vs default (not run yet) |
| [test/ab-template.csv](test/ab-template.csv) | Empty CSV template for that test |
| [rusty-kaspa PR #1148](https://github.com/kaspanet/rusty-kaspa/pull/1148) | supertypo's "Add lowio RocksDB preset" (open) |
| [KaspaScopio on X](https://x.com/kaspascopio/status/2109623688924389547) | The post that pointed us to it |
| [tn-master](https://github.com/STP-KAS/tn-master) | Our TN10 findings and lessons |
| [tn10-storm-2026-10-11](https://github.com/STP-KAS/tn10-storm-2026-10-11) / [tn10-storm-2026-10-10](https://github.com/STP-KAS/tn10-storm-2026-10-10) | Storm logs this note draws on |

## Intro

Notes from TN10 operators who run storm tests on a single 126 GB box disk. supertypo's PR #1148 adds a `lowio` RocksDB preset for pruned nodes. This repo is our read of the diff, why it matters for what we saw in our storms, what we would want checked, and a test we could run later. It is commentary only: nothing here was posted to the PR or to kaspanet. Labels: **PR claim** (from the PR text), **Logged** (our own logs), **Estimate** (our reasoning, not measured).

## Briefing

### What: what #1148 changes (from the diff at head `e3773118`, 1 commit, +115/-10, 5 files)

- `database/src/db/rocksdb_preset.rs`: new `RocksDbPreset::LowIo` ("Recommended for pruned nodes"), parsed from `lowio` (any case). `apply_lowio` sets:
  - write buffers = `mem_budget / 4`, `max_write_buffer_number(4)`; target file 256 MB, level base 1 GB, dynamic level bytes; L0 trigger 4, slowdown 40, stop 64.
  - ZSTD (`compression_options(-14, 1, 0, 0)`, level 1) on all levels.
  - Block table: Bloom 16 bits, partitioned filters, two-level index, index/filter blocks in cache, 16 KB blocks, LRU block cache = `cache_budget` or 1 GB.
  - BlobDB: values ≥ 512 bytes, 256 MB blob files, LZ4, blob GC on with age cutoff 0.1 and force threshold 0.75 ("Pruning deletes the oldest blobs first").
  - No rate limiter (the HDD preset has one).
  - New `mem_budget(ram_scale)`: lowio = 1 GB × ram-scale, clamped to 256 MB–1 GB (scales **down only**); default/hdd stay at 64 MB. Unit test `lowio_mem_budget_scales_down_only`.
- `kaspad/src/daemon.rs`: `configure_rocksdb` now returns the mem budget; lowio gets a cache of 1 GB × ram-scale (min 64 MB) unless `--rocksdb-cache-size` is set; the budget is passed with `.with_mem_budget(...)` to the meta DB, consensus DBs, staging consensus and the UTXO index DB.
- `consensus/src/consensus/factory.rs`: `Factory` carries `rocksdb_mem_budget` into consensus DB builds.
- `kaspad/src/args.rs`: help text lists `lowio`. Usage: `kaspad --rocksdb-preset=lowio` (or `KASPAD_ROCKSDB_PRESET`).
- `testing/integration/src/consensus_integration_tests.rs`: passes the new argument.
- **PR claims** (mainnet IBD from scratch, Ryzen 9 5900X, NVMe, LAN peer): about 50% fewer bytes written, about 99% fewer compactions, about a third less compaction CPU, about the same DB size, about 10% faster IBD; per unit of work block bodies -31%, UTXO download -7%, proof -4%, UTXO import -3%, headers -13%.

### Why: why it matters for TN10 operators and our storms

- **Disk was our main failure mode.** Logged: pruning bursts took up to about 16 GB in 30 min; a 126 GB disk could not hold an ~86-88 GB node plus 14-18 GB dips ([tn-master LESSONS](https://github.com/STP-KAS/tn-master/blob/main/docs/LESSONS.md)). n0 filled its disk while pruning and was retired ([10-10 LESSONS](https://github.com/STP-KAS/tn10-storm-2026-10-10/blob/main/docs/LESSONS.md)). On 10 Oct 21:08-21:29 CEST a disk/freeze event zeroed a step ([RESULTS](https://github.com/STP-KAS/tn10-storm-2026-10-10/blob/main/docs/RESULTS.md)). On 11 Oct the box hit 18.8 GB free at 07:31Z and ran with guards (senders pause 8 GB, node STOP 5 GB, halt 3 GB, public fallback below 10 GB); the guard stopped n2 at 21:37 CEST and its 88G data folder was later deleted ([CHANGES-DURING-RUN](https://github.com/STP-KAS/tn10-storm-2026-10-11/blob/main/logs/bot/CHANGES-DURING-RUN.md)).
- **Freezes.** Logged: n2 froze 11:08-11:25Z on 11 Oct (DAA still, sink lag ~16 min) and was restarted on the same flags; the box also paused at 07:38, 08:45, 09:56Z and ~11:56 CEST ([WRITE-UP](https://github.com/STP-KAS/tn10-storm-2026-10-11/blob/main/logs/WRITE-UP.md)). We have **not** shown these were caused by write I/O; it is an open question whether fewer writes and compactions would have helped. Note: lowio's blob GC and compressed levels do not obviously shrink the pruning *size* burst (PR claim: "a database of about the same size"); the gain claimed is in bytes written and compactions.
- **#1147 link.** A node that catches up by IBD can keep already-accepted txs in its mempool and skew fee estimates ([tn-master FINDINGS 11](https://github.com/STP-KAS/tn-master/blob/main/FINDINGS.md); [PR #1147](https://github.com/kaspanet/rusty-kaspa/pull/1147)). supertypo's [review on #1147](https://github.com/kaspanet/rusty-kaspa/pull/1147#issuecomment-6124362740) worries about the cost of re-checking the whole mempool after catch-up. Estimate: a faster IBD (PR claim ~10%) shortens the window in which stale entries build up, but it does not remove them; the two PRs are complementary.

### How: risks, open questions, and what we would test

Risks and open questions (Estimates unless marked):
1. **RAM on small boxes.** From the diff, each DB opened with lowio (meta, consensus, staging consensus, UTXO index) gets its own `mem_budget` (up to 4 × 256 MB write buffers) and its own new 1 GB LRU cache (`Cache::new_lru_cache` inside `apply_lowio`). Our reading is that peak RAM can be several GB above default at `--ram-scale=1.0`, not "a bit more"; this is not measured. Our box already capped the bot at about 18 senders on RAM (Logged).
2. **Lowering ram-scale has a known trap.** `--ram-scale=0.1` gave a 100k mempool cap and 2.1.0 panicked at `100001 > 100000` (Logged, [FINDINGS 4](https://github.com/STP-KAS/tn-master/blob/main/FINDINGS.md)). Operators trimming lowio's RAM by scale should stay well above that.
3. **Archival vs pruned.** The PR targets pruned nodes; blob GC tuned for "oldest first" pruning. Behaviour on archival nodes (which keep everything) is not claimed; the HDD preset remains the archival recommendation.
4. **Write amplification under high tx/s.** The PR's numbers are mainnet IBD. Our storms ran 3,000+ tx/s for 163 min at ~10 blocks/s (Logged). Steady-state writes under that load, and L0 stalls with the 40/64 triggers and no rate limiter, are untested.
5. **Mempool interaction.** None in the diff: the mempool is in memory and not stored in RocksDB. Indirect effects only (IBD speed, #1147 window above).
6. **UTXO index reads.** We read n2's UTXO index through a read-only secondary RocksDB during storms (Logged, safe on default). Whether that works the same with BlobDB on is untested.

Proposed test: [test/AB-TEST.md](test/AB-TEST.md) with [test/ab-template.csv](test/ab-template.csv): two pruned TN10 nodes, lowio vs default, IBD + idle + 60 min at about 600 tx/s + a resync; metrics disk write bytes, compactions, IBD time, RSS, DB size, freezes, fee estimate.

## Sources
- PR: https://github.com/kaspanet/rusty-kaspa/pull/1148 (supertypo, opened 12 Oct 2026, 13:34 CEST; head `e3773118`, base `01b532e8` = v2.1.0)
- X post: https://x.com/kaspascopio/status/2109623688924389547
- PR #1147 and supertypo's review: https://github.com/kaspanet/rusty-kaspa/pull/1147, https://github.com/kaspanet/rusty-kaspa/pull/1147#issuecomment-6124362740
- Master file: [kaspa-master-file README](https://github.com/STP-KAS/kaspa-master-file) (Live row: v2.1.0 tag `01b532e8`), [TN10-INCIDENT-2026-09-29](https://github.com/STP-KAS/kaspa-master-file/blob/main/TN10-INCIDENT-2026-09-29.md) (n0 IBD from an empty data dir)
- Our logs: [tn-master FINDINGS](https://github.com/STP-KAS/tn-master/blob/main/FINDINGS.md), [LESSONS](https://github.com/STP-KAS/tn-master/blob/main/docs/LESSONS.md), [ADVICE-TO-DEVELOPERS 14](https://github.com/STP-KAS/tn-master/blob/main/ADVICE-TO-DEVELOPERS.md), [tn10-storm-2026-10-11 WRITE-UP](https://github.com/STP-KAS/tn10-storm-2026-10-11/blob/main/logs/WRITE-UP.md), [tn10-storm-2026-10-10 RESULTS](https://github.com/STP-KAS/tn10-storm-2026-10-10/blob/main/docs/RESULTS.md)
- Kaspa Pulse (@gokugalax, kaspapulse.com) numbers are not used here.
