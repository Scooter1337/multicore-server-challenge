# Rust / epoll

Front: direct

Based on [WorriedArrow’s MIT-licensed implementation](https://github.com/arjaythedev/multicore-server-challenge/pull/47); see NOTICE and LICENSE.

Rust 1.99.0, raw epoll, a PostgreSQL v3 wire-protocol client and mimalloc. Dependencies are pinned in Cargo.lock.

Reads share distinct lookups and metadata within one closed batch. Writes wait for durable commit. Passwords use bcrypt cost 10; each authenticated request verifies its JWT signature and expiry. No database data survives its batch.

Changes from that implementation:

- Serialize viewer-independent article, comment and tag fields once per closed batch; insert each viewer’s flags separately.
- Scan JSON strings with SSE2 before escaping.
- Store PostgreSQL column descriptors inline and copy each row payload once.

197/197 conformance checks pass on both seeds, plus concurrent differential reads and mixed-viewer checks.

Local MEGASERVER comparison: app + PostgreSQL share 8 CPUs / 16 GiB, with generators on the same host. Using the unmodified repository workload, a 60-second ramp and five-minute measured hold, this version passes at **128,000 users**: **32,033 req/s**, p95 **464 ms**, p99 **652 ms**, zero errors. The unchanged #47 implementation fails the matching 128,000-user hold (p95 **513 ms**); its complete repository search scores **120,000 users**.

These are local measurements. The candidate's initial search stopped at a generator-limited 200,000-user probe; 128,000 is a separately confirmed hold. A 130,000-user hold failed. All reported comparisons passed generator validity checks.

```sh
sudo bash install.sh
bash build.sh
DATABASE_URL=postgres://conduit:conduit@127.0.0.1:5432/conduit JWT_SECRET=... HOST=0.0.0.0 PORT=80 bash start.sh
```
