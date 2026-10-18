# Rust / Tokio

Front: direct

Rust 1.99.0, Tokio 1.53.2, tokio-postgres 0.7.18, PostgreSQL 18 and mimalloc 0.1.52. Dependencies are pinned in Cargo.lock.

- Direct HTTP/1.1 using httparse, with fixed-capacity buffer free lists, separate 1 KiB request buffers, stack response headers and vectored writes. Buffers are released while sockets are idle. Active request futures are boxed so idle connections retain a small task. Read completion is bounded after the first request byte; idle keep-alive remains unlimited.
- Prepared generic plans, separate read/write connections, and one statement per mutation. Failed connections and their statements are replaced together, without replaying requests. Global and tag pages compute exact favorite counts and viewer membership in one index scan. Article metadata is computed after pagination; API-bounded pages improve generic-plan cardinality, capped tag counts avoid aggregation when the cap is proven, and favorited-page membership is computed once per request. Existing tags are read without unnecessary updates or locks.
- Read actors pack parameters once and send queued Bind/Execute messages with one Sync, up to 32 reads with no collection delay. Each statement takes its own READ COMMITTED snapshot; results wait for ReadyForQuery. The vendored tokio-postgres changes drain errors before reuse and preserve upstream licenses.
- Durable group commit: connection-owning actors collect concurrent writes, commit, then release responses. Failed batches roll back and retry requests individually; uncertain commits are never retried. Favorites and article updates read a fresh snapshot after commit.
- Borrowed PostgreSQL text arrays, direct JSON serialization and SSE2 string escaping on x86-64. Fat LTO and one codegen unit.
- Bounded bcrypt work using the Openwall/libxcrypt 4.4.36 core, with fresh random salts and cost 10. Up to four independent hashes are interleaved to hide dependent table-load latency, retaining every bcrypt round. Startup checks compare the interleaved paths against the original core. Embedded NULs use the portable Rust backend.

Every request reads PostgreSQL. Authentication checks signatures and expiry on every use; there are no result, token-verification or password caches. No schema/config changes, CPU affinity or host tuning.

On eight CPUs the defaults are two Tokio workers, six read connections, two grouped write connections and one bcrypt worker. A write group waits up to 1 ms and contains at most 128 requests. Bcrypt collects up to four jobs with a 160 ms collection window. Optional process settings: `APP_THREADS`, `DB_CONNS`, `DB_WRITE_CONNS`, `BCRYPT_THREADS`, `WRITE_BATCH_US`, `BCRYPT_BATCH_MS`, `READ_BATCH_SIZE` (default 32; 0 disables read batching), `READ_BATCH_US` (default 0).

```sh
sudo bash install.sh
bash build.sh
DATABASE_URL=postgres://conduit:conduit@127.0.0.1:5432/conduit JWT_SECRET=... HOST=0.0.0.0 PORT=80 bash start.sh
```

The API and initial SQL build on [cknutson12's MIT-licensed rust-hyper submission](https://github.com/arjaythedev/multicore-server-challenge/pull/1), with attribution retained in LICENSE. The native bcrypt source and its license are documented in native/README.md.

Local full-seed tests on MEGASERVER (Xeon E5-2690 v3), with the app and PostgreSQL sharing an 8-CPU quota and 16 GiB, no affinity or host tuning. The unchanged load generator confirmed 76,000 users over five minutes: 19,964 req/s, p95 92 ms, p99 367 ms, zero errors, 105 MiB peak app RSS. In a separate aged-database 80,000-user control/candidate/control comparison, combining favorite count and viewer membership delivered 20,679 req/s versus 19,931 and 20,055: 3.11% above the better control, with p95 201 ms versus 308 ms. Generators used separate cgroups on the same physical host, with loopback networking and CPU/port validation; these are local comparisons, not official challenge scores.

Validation: all 197 unchanged conformance checks on both dev and full seeds, HTTP framing/pipelining and request-deadline checks, database-backend and full-cluster restart recovery, concurrent tags/favorites, grouped-write rollback isolation and immediate visibility, release unit tests, Clippy, and native bcrypt ASan/UBSan compatibility checks.
