<<<<<<< HEAD
# lsm3 — an LSM tree index for PostgreSQL

`lsm3` is a PostgreSQL extension that adds an **LSM tree index access method**, built
entirely out of ordinary Postgres B-Tree indexes underneath. The goal is **fast inserts**:
instead of scattering random writes across one huge index, new rows go into a small index
that fits in memory, and a background worker gradually merges that data downward.

## How it works

Every `lsm3` index is really a collection of plain B-Trees, arranged in layers:

```text
inserts
   |
   v
top0 / top1        <- two small mutable "L0" buffers (one active, one being merged)
   |
   v
level0: run0 run1 run2 run3    <- immutable runs
level1: run0 run1 run2 run3
level2: run0 run1 run2 run3
   |
   v
base               <- one big B-Tree holding the bulk of the data
```

- **Inserts** only ever touch the **active top index**. It's small, so it stays hot in
  memory and writes are cheap.
- When the active top index grows past `top_index_size`, the two tops **swap roles**. The
  now-inactive one is merged in the background while inserts keep flowing into the other,
  so merging never blocks writers.
- The inactive top is merged into a free **run** of level0. Runs are immutable once written.
- When a level runs out of free run slots — or its total size crosses its threshold — all
  its runs are compacted together into a single run of the next level (**tiered
  compaction**). The last level compacts into `base`.
- A level's size threshold grows geometrically: `top_index_size * ratio^(level+1)`.

**Reads** have to scan every component (both tops, all occupied runs, and base) and merge
the results into one sorted stream. That's the trade-off: writes get much faster, reads do
more work.

### Bloom filters

To claw back some read cost, `lsm3` keeps **backend-local Bloom filters** for the immutable
level runs. On an **equality lookup** on the first index key, a run whose Bloom filter says
"definitely not here" is skipped entirely — no B-Tree descent at all.

Details worth knowing:

- Bloom filters apply only to **immutable level runs** — never the active tops (they keep
  changing) and never `base`.
- They apply only to **equality predicates**, not range scans.
- They're cached per backend and invalidated via shared generation counters whenever a run
  is rewritten or truncated.
- A "maybe present" answer is always safe; only a definite "absent" causes a skip, so
  results are never wrong.

Controlled by the `bloom_enabled` reloption (on by default).

## Installation

`lsm3` allocates shared memory, so it **must** be listed in `shared_preload_libraries` in
`postgresql.conf` — installing it at runtime alone is not enough:

```ini
shared_preload_libraries = 'lsm3'
```

Build and install with PGXS, then restart the server:

```sh
make
sudo make install
sudo service postgresql restart
```

Then, in your database:

```sql
create extension lsm3;
create table t(id integer, val text);
create index idx on t using lsm3(id);
```

`lsm3` supports the same data types and operators as a standard B-Tree.

## Restrictions

- Parallel index scan is not supported.
- Array keys are not supported.
- An `lsm3` index **cannot be declared as a unique constraint** — uniqueness cannot be
  enforced across the components.

## The `unique` option

Even though uniqueness can't be *enforced*, you can still mark an index unique as a
**search optimization** — a promise from you, not a guarantee from the index:

```sql
create index idx on t using lsm3(id) with (unique=true);
```

This tells `lsm3`: if the key is found in the active top index, stop — don't look in the
other components. Applications usually search for recently inserted data, so this often
turns a many-component scan into a single lookup.

If your data actually contains duplicates, this option will silently hide the older ones.

## Configuration

### Server settings (GUCs)

| Setting | Meaning | Default |
| --- | --- | --- |
| `lsm3.top_index_size` | Size (KB) the active top index may reach before a merge is triggered | 64 MB |
| `lsm3.max_indexes` | Maximum number of `lsm3` indexes (sizes the shared-memory array; requires restart) | 1024 |

### Index options (reloptions)

| Option | Meaning | Default | Range |
| --- | --- | --- | --- |
| `top_index_size` | Per-index override of the `lsm3.top_index_size` GUC | GUC value | — |
| `num_levels` | Number of intermediate levels between the tops and `base` | 3 | 1–8 |
| `runs_per_level` | Immutable runs a level holds before it compacts | 4 | 1–8 |
| `level_size_ratio` | Size multiplier between consecutive levels | 4 | ≥ 2 |
| `bloom_enabled` | Use Bloom filters to skip level runs on equality lookups | `true` | — |
| `unique` | Stop the search after the first hit in the active top index | `false` | — |

Standard B-Tree options `fillfactor`, `deduplicate_items`, and
`vacuum_cleanup_index_scale_factor` are also accepted.

Example:

```sql
create index idx on t using lsm3(id)
  with (top_index_size=1024, num_levels=2, runs_per_level=8, bloom_enabled=true);
```

Fewer levels and larger runs mean less read amplification but more work per compaction;
more levels spread the compaction cost out but make scans touch more components.

## Helper functions

| Function | Purpose |
| --- | --- |
| `lsm3_get_merge_count(index regclass)` | Number of merges performed since the database started |
| `lsm3_start_merge(index regclass)` | Trigger a merge manually |
| `lsm3_wait_merge_completion(index regclass)` | Block until the current merge finishes |
| `lsm3_top_index_size(index regclass)` | Current size of the active top index |

## Background workers

**Each `lsm3` index starts its own background merge worker.** If you create several
indexes, raise `max_worker_processes` in `postgresql.conf` accordingly — otherwise index
creation will fail once the worker slots run out.

## Tests

The regression test runs through PGXS:

```sh
make installcheck
```

The repo also contains standalone SQL scripts used for benchmarking and experiments
(`bench_test.sql`, `bloom_test.sql`, `step_test.sql`, `lsm3vsourlsm.sql`, and others).

## Credits

Based on [postgrespro/lsm3](https://github.com/postgrespro/lsm3) by Konstantin Knizhnik,
extended here with multi-run tiered levels and per-run Bloom filters.
=======
# LSM-Trees-in-PostgreSQL
>>>>>>> 964ea2b0abdf9f62383db1eab3eb16f0fe3c7e40
