# sysbench-compressible

sysbench OLTP workloads with **compressible data**: one new option, `--data-random-pct`, for the stock sysbench Lua
scripts.

Stock sysbench fills the `c` (120 characters) and `pad` (60 characters) columns with random digits, so its InnoDB pages
hardly compress: about **1.14x with lz4**. Real data usually compresses better than that. This matters whenever you
benchmark anything that compresses: zram / zswap, InnoDB page or table compression, compressed file systems or storage
(ZFS, Btrfs, compressing SSDs and cloud volumes), backups, replication or network compression. With stock sysbench, such
features look useless.

`--data-random-pct=N` keeps N% of the characters in each 11-digit group random and turns the rest into fixed text, the
same in every row. The default, 100, is stock sysbench. The same templates are used by the UPDATE and INSERT statements
during `run`, so the data stays equally compressible while the workload runs.

```
stock (100):  68487932199-96439406143-93774651418-41631865787-96406072701-...
10:           31e quick b-56n fox jum-80over the -64y dog whi-03the cat s-...
```

## Measured compressibility

1M rows on Percona Server 8.4, InnoDB 16 KiB pages. "4 KiB" = each 4 KiB piece of the `.ibd` file compressed on its own,
the unit Linux zram and zswap compress (`tools/pagecomp.py`):

| `--data-random-pct` | lz4, 4 KiB | zstd -1, 4 KiB | lz4, 16 KiB |
|---|---|---|---|
| 100 (stock) | 1.14 | 2.12 | 1.16 |
| 30 | 2.19 | 4.08 | 2.42 |
| 20 | 2.51 | 4.94 | 2.83 |
| 10 | 2.96 | 5.90 | 3.30 |
| 0 | 6.35 | 8.23 | 7.62 |

For comparison, other benchmarks' data measured the same way (lz4, 4 KiB): HammerDB TPROC-C 1.33, BenchBase TPC-C
1.42, IMDb (real data, Join Order Benchmark) 1.73, TPC-H 2.38, BenchBase AuctionMark 2.79.

## Use

Works with stock sysbench 1.0.20 (the version in most Linux distributions); no rebuild needed. Put the scripts in a
directory and point `LUA_PATH` at it, so the patched `oltp_common.lua` is loaded instead of the system one:

```bash
git clone https://github.com/Percona-Lab/sysbench-compressible.git
cd sysbench-compressible
export LUA_PATH="$PWD/?.lua;;"
sysbench ./oltp_read_write.lua --mysql-user=sbtest --mysql-password=sbtest --mysql-db=sbtest \
  --tables=16 --table-size=11000000 --data-random-pct=10 --threads=16 prepare
sysbench ./oltp_read_write.lua --mysql-user=sbtest --mysql-password=sbtest --mysql-db=sbtest \
  --tables=16 --table-size=11000000 --data-random-pct=10 --rand-type=uniform --threads=64 --time=600 run
```

Use the same `--data-random-pct` for `prepare` and `run`.

All `oltp_*.lua` scripts here are the stock sysbench 1.0.20 scripts; only `oltp_common.lua` is changed. For packagers,
`oltp_common-data-random-pct.patch` is the same change as a patch against sysbench's `src/lua/oltp_common.lua`; it
applies to 1.0.20 and to current master.

Measure your own data with `tools/pagecomp.py <name> /var/lib/mysql/<schema>` (needs `python3-lz4` and
`python3-zstandard`; run it when the buffer pool has no dirty pages for that schema).

## Background

Built for a study of zram and zswap with an InnoDB buffer pool larger than RAM, where stock sysbench and HammerDB data
compressed too little to test the idea: <https://github.com/Percona-Lab/zram-zswap-db-benchmark>.

## License and disclosure

GPL v2 or later, like sysbench (Copyright (C) 2006-2018 Alexey Kopytov); see `LICENSE`.

This change, the measurements and this README were produced with an AI agent (Claude) and spot-checked only; they may
contain mistakes.
