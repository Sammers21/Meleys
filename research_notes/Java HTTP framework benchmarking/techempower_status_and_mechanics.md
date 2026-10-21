# TechEmpower Framework Benchmarks (TFB): status, mechanics, submission process and Java rankings (as of 2026-10-21)

Research date: 2026-10-21. Every fact is dated where it matters. "R22/R23" = Round 22/23. Numbers marked "computed" were calculated by me from the official Round 22/23 results JSON files that techempower.com still serves. I used the same formula as the results website's JavaScript bundle (see Q3/Q4). Ranks are among "Realistic" permutations only, because the site hides "Stripped" entries by default. They are ranks of permutations (one framework can have several), not of frameworks.

## Q1. What is TFB's status as of October 2026: latest round, continuous runs, and announcements about its future?

### Takeaway
TFB is over as a living benchmark. TechEmpower announced the project's sunset on 2026-03-24 and archived the GitHub repository (read-only) the same day. Round 23 is the final official round: results dated 2025-02-24, blog post 2025-03-17. No Round 24 was ever published. On 2026-10-21 the continuous-run dashboard (tfb-status) did not respond (HTTP 503), but the results website and the Round 23 raw data are still online. Meleys cannot be listed. TFB is now only a frozen reference dataset plus an open-source (BSD-style) toolset and spec that can be run privately.

### Cited Findings
- **Sunset announcement.** Issue #10932, "Sunsetting the TechEmpower Framework Benchmarks", was opened and pinned by NateBrady23 on 2026-03-24. It says "the repository, history, and past results remain as a snapshot" and gives tfb@techempower.com as the contact. It gives no specific reason, and does not mention the results site, tfb-status, the hardware, or any successor or fork. — [GitHub issue #10932](https://github.com/TechEmpower/FrameworkBenchmarks/issues/10932)
- **Repository archived.** The repo shows "archived by the owner on Mar 24, 2026. It is now read-only." — [issue #10932](https://github.com/TechEmpower/FrameworkBenchmarks/issues/10932); [Discussions](https://github.com/TechEmpower/FrameworkBenchmarks/discussions)
- **Last commits.** The newest commits on master are Dependabot merges dated 2026-03-23 (e.g., PR #10927). — [master commits](https://github.com/TechEmpower/FrameworkBenchmarks/commits/master)
- **No open PRs.** A GitHub search on 2026-10-21 returns 0 open PRs for the repo. — [PR search](https://github.com/TechEmpower/FrameworkBenchmarks/pulls?q=is%3Apr+is%3Aopen)
- **Active until the end.** New frameworks were merged until two weeks before the sunset:
  - Rust "vortex" (#10833) and Zig "oxelot-http" (#10816), both merged 2026-03-10.
  - Java "loveqq-framework" (#10342), merged 2025-12-12.
  - Spring jOOQ variant (#10555), merged 2026-01-14.
  - [PR search](https://github.com/TechEmpower/FrameworkBenchmarks/pulls?q=is%3Apr+is%3Amerged+add+in%3Atitle+created%3A%3E2025-03-01); [#10833](https://github.com/TechEmpower/FrameworkBenchmarks/pull/10833); [#10342](https://github.com/TechEmpower/FrameworkBenchmarks/pull/10342)
- **Toolset still maintained in late 2025.** msmith-techempower merged PR #10375 "Add operating system info to results.json" on 2025-12-08. PR #10721 added "maintainers and tags fields to benchmark config" (merged 2026-02-10). — [PR search](https://github.com/TechEmpower/FrameworkBenchmarks/pulls?q=is%3Apr+is%3Amerged+add+in%3Atitle+created%3A%3E2025-03-01)
- **No reason given anywhere public.** The Hacker News thread "Sunsetting the Techempower Framework Benchmarks" (submitted by nbrady; 74 points, 22 comments) has no TechEmpower explanation. Commenter mseepgood: "This text lacks information about why it is being sunset." One commenter pointed to HttpArena as an alternative. Another hoped "an active fork emerges". — [HN](https://news.ycombinator.com/item?id=47497763)
- **Third-party view (dated 2026-03-24, edited June 2026).** Andreas Nägeli, GenHTTP maintainer, wrote that TFB was discontinued 2026-03-24. In his view the platform had "virtually no changes" in recent years, and plaintext plateaued at about 30M req/s because of wrk and the request headers. He proposed HttpArena (https://www.http-arena.com/), which runs benchmarks on PRs and covers HTTP/2, WebSockets and gRPC. Other researchers cover alternatives and criticism. — [DEV Community post](https://dev.to/kaliumhexacyanoferrat/techempower-framework-benchmarks-are-now-archived-whats-next-3l0a)
- **Round dates (results site metadata).**
  - R20: 2021-02-08
  - R21: 2022-07-19
  - R22: 2023-10-17
  - R23: 2025-02-24, titled "Benchmarking at the speed of light!", with the note "TechEmpower's Framework Benchmarks have new fiber optic networking and 40-gigabit network cards."
  - No Round 24 entry exists.
  - [techempower.com/benchmarks](https://www.techempower.com/benchmarks/); [site JS bundle](https://www.techempower.com/benchmarks/assets/index-rdjV3uRt.js)
- **How Round 23 was published.**
  - 2025-02-11: msmith-techempower opened issue #9589 "Round 23 Complete", linking the tfb-status run and asking the community to check it.
  - 2025-03-03: issue closed as published.
  - 2025-04-23: he said Actix and all its variants "failed to build in the round run".
  - [Issue #9589](https://github.com/TechEmpower/FrameworkBenchmarks/issues/9589)
- **Blog posts (TFB tag).**
  - "Framework Benchmarks Round 23": 2025-03-17
  - "Framework Benchmarks Round 22": 2023-11-15
  - Last TFB post: "How Our Benchmarks Led to a 25x MongoDB Performance Improvement", 2026-03-16, by Mike Smith. It does not mention Round 24 or the sunset.
  - [TechEmpower blog, tfb tag](https://www.techempower.com/blog/tag/tfb/); [MongoDB post](https://www.techempower.com/blog/2026/03/16/how-our-benchmarks-led-to-a-25x-mongodb-performance-improvement/)
- **Round 23 is a continuous run.** The official data is the run named "Continuous Benchmarking Run 2025-01-30 18:47:41" (uuid 91a66052-9d86-446c-b31a-eadbd669ed08, environment label "Citrine"). It ran master at commit 523534b (2025-01-23), from 2025-01-30 18:47 UTC to 2025-02-06 06:39 UTC, i.e. 155.9 hours (computed). — [Round 23 results JSON](https://www.techempower.com/benchmarks/results/round23/ph.json)
- **Round 22 is also a continuous run.** It is "Continuous Benchmarking Run 2023-10-11 08:00:48" (uuid 66d86090-…), which took 158.8 hours (computed). — [Round 22 results JSON](https://www.techempower.com/benchmarks/results/round22/ph.json)
- **Round 24 question unanswered.** A Q&A discussion, "Also, when do you expect the next benchmark round to run (Round 24)?" (HassanSharara, 2025-11-09), is marked Unanswered with no maintainer reply visible. — [Discussions](https://github.com/TechEmpower/FrameworkBenchmarks/discussions)
- **Continuous runs were still running in 2025–2026.**
  - 2025-04-25: a maintainer said Actix results had reappeared "in the latest continuous runs". — [#9589](https://github.com/TechEmpower/FrameworkBenchmarks/issues/9589)
  - 2026-03-13: a discussion titled "Potential Stall in Continuous Run: Estimated Remaining Time Increasing (Run e297014e)" was opened (no visible replies). — [Discussions](https://github.com/TechEmpower/FrameworkBenchmarks/discussions)
- **tfb-status is unreachable (own check, 2026-10-21).** WebFetch of tfb-status.techempower.com returned HTTP 503 Service Unavailable. curl through the proxy failed for both tfb-status and tfb-logs.techempower.com (connection reset / no response). By contrast, https://www.techempower.com/benchmarks/results/round23/ph.json returned HTTP 200 (5.0 MB). — [tfb-status](https://tfb-status.techempower.com/); [R23 JSON](https://www.techempower.com/benchmarks/results/round23/ph.json)
- **Scale of Round 23.** 841 test permutations covering 331 distinct frameworks (computed). This matches the DEV post's "more than 330 framework implementations". — [R23 JSON](https://www.techempower.com/benchmarks/results/round23/ph.json); [DEV](https://dev.to/kaliumhexacyanoferrat/techempower-framework-benchmarks-are-now-archived-whats-next-3l0a)
- **Funding and hardware.** Microsoft provided the new Round 23 hardware. — [TechEmpower blog, Round 23 post via tfb tag](https://www.techempower.com/blog/tag/tfb/). Round 13 (2016-11-16) was the first round with Microsoft (Azure) and ServerCentral environments, and the first with continuous benchmarking. — [site JS bundle](https://www.techempower.com/benchmarks/assets/index-rdjV3uRt.js)
- **License.** The repository is under a BSD-style license ("Copyright (c) 2021, TechEmpower, Inc. … Redistribution and use in source and binary forms, with or without modification, are permitted…"). — [LICENSE](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/LICENSE)

### Inferences
- **Meleys cannot get an official TFB listing.** The repo is read-only: no PRs, no continuous runs, no Round 24. Any "TFB score" for Meleys would be self-run and unofficial. It would only be comparable to Round 23 numbers on equivalent hardware with the same toolset and competitor images.
- **The final dataset is aging.** Round 23 reflects framework code as of 2025-01-23 on JDKs 8–22 (see Q6). By October 2026 it is about 21 months old. Many Java entries were updated to JDK 24/25 afterwards, but those runs were only ever on tfb-status, which is now offline.
- **Merge timing tracked run starts.** Merges were batched roughly weekly (e.g., six PRs merged on 2026-03-10, several on 2026-02-10 and 2025-11-11). This fits full runs of about 6.5 days that clone master at each start (see Q5). That points to a cadence of roughly one continuous run per week in 2025–2026. This is inference only; tfb-status is not available to confirm it.
- **Official rounds were already infrequent.** The gaps between the last rounds were about 15 months (R21 to R22) and about 16 months (R22 to R23).

### Gaps
- The reason for the sunset (cost, sponsorship, staffing) is not stated in any primary source found.
- I could not determine the date of the last continuous run, or whether tfb-status is permanently down. tfb-status returned 503 and the Wayback Machine was rate-limited/blocked from this environment.
- No community fork that continues TFB itself was found. Only alternative projects (HttpArena, web-frameworks-benchmark) are mentioned.

## Q2. What hardware and environment produced the official results, and has it changed from "Citrine" (Dell R440, Xeon Gold 5120, 10 GbE)?

### Takeaway
Yes, it changed for Round 23, though the environment kept the name "Citrine". Rounds 16–22 ran on three Dell R440 servers (Xeon Gold 5120, 14 cores/28 threads, 32 GB, 10 GbE). Round 23 ran on three homogeneous HPE ProLiant DL360 Gen10 Plus servers: Intel Xeon Gold 6330 @ 2.00 GHz (listed as "56 cores"; the chip is 28 cores/56 threads), 64 GB RAM, enterprise SSD, Mellanox ConnectX-6 40 Gbps Ethernet. The hardware was provided by Microsoft. The host OS was Ubuntu 24.04 and all components ran in Docker. The project wiki was never updated and still describes the old environment.

### Cited Findings
- **Round 23 hardware (results site).** "Citrine (rounds 23 onward): Three homogeneous ProLiant DL360 Gen10 Plus equipped with Intel Xeon Gold 6330 CPU @ 2.00GHz (56 cores), 64 GB of memory, an enterprise SSD, and Mellanox Technologies MT28908 Family [ConnectX-6] 40Gbps Ethernet. Provided by Microsoft." Environment codes: `citrine` = "Dell R440 Xeon Gold + 10 GbE"; `citrine40` = "ProLiant DL360 Gen10 Plus + 40 GbE". Round 23 uses `physical:"citrine40"`. — [site JS bundle](https://www.techempower.com/benchmarks/assets/index-rdjV3uRt.js)
- **Previous hardware (results site).** "Citrine (rounds 16 onward): Three homogeneous Dell R440 servers each equipped with an Intel Xeon Gold 5120…". — [site JS bundle](https://www.techempower.com/benchmarks/assets/index-rdjV3uRt.js)
- **Round 23 blog post (2025-03-17).** Same hardware description. It reports "3x improvement in practical network-bound tests among top frameworks and up to 4x in theoretical network-bound tests", and says TechEmpower is committed to continuing improvements. — [TechEmpower blog, tfb tag](https://www.techempower.com/blog/tag/tfb/)
- **Wiki environment page (last edited 2024-04-04; outdated).**
  - `BENCHMARK_ENV=Citrine`
  - All 3 machines (tfb-server, tfb-database, tfb-client): "28 HT Cores Intel(R) Xeon(R) Gold 5120 CPU @ 2.20GHz, 32GB RAM, Ubuntu 18.04.3 4.15.0-88-generic", switched 10-gigabit Ethernet.
  - sysctl: `net.ipv4.tcp_max_syn_backlog = 65535`, `net.core.somaxconn = 65535`, `net.ipv4.tcp_tw_reuse = 1`, `vm.max_map_count = 1048576`.
  - Azure environment: 3× Standard_D3_v2 (4 vCPU, 14 GB).
  - Load simulator: wrk.
  - Earlier environments: AWS c3.large/m1.large (R1–12), i7-2600K (R1–8), Peak R720xd (R9–12), ServerCentral R910/R710 (R13–15).
  - [Wiki: Environment](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Environment)
- **Environment label unchanged in the data.** The Round 23 results file still labels the environment "Citrine". — [R23 JSON](https://www.techempower.com/benchmarks/results/round23/ph.json)
- **CPU spec.** The Xeon Gold 6330 has 28 cores and 56 threads, 2.0 GHz base / 3.1 GHz turbo, 42 MB cache, 205 W TDP, Ice Lake-SP. — [PB Tech spec listing](https://www.pbtech.com/product/CPUITXBG6330/Intel-Xeon-Gold-6330-CPU-28-Core--56-Thread---20GH)
- **TechEmpower's own wording and OS versions (2026-03-16).** The MongoDB post describes the Round 23 environment as "a single Xeon 6330 box with 56 hyper-threaded cores". Round 22 ran on Ubuntu 22.04 and Round 23 on Ubuntu 24.04; the host OS "moved forward" without being recorded in the results. MongoDB ran in Docker, and a TCMalloc per-CPU-cache issue in Docker cut spring-mongo performance (query test: 5.9k req/s in R22, 583 in the R23 baseline, 14.2k after fixes in PRs #10514 and #10565). — [TechEmpower MongoDB blog post](https://www.techempower.com/blog/2026/03/16/how-our-benchmarks-led-to-a-25x-mongodb-performance-improvement/)
- **Official benchmarking setup (wiki).** Three machines (app server, database server, client), each running Ubuntu Server 22.04 LTS with Docker. The Docker daemon is exposed on `-H <ip>:2375`. The suite runs as `docker run … techempower/tfb --server-host … --database-host … --client-host … --network-mode host`. — [Wiki: Benchmarking Getting Started](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Benchmarking-Getting-Started)
- **Load-generator container.** The wrk image is `FROM ubuntu:24.04` plus the distribution's `wrk` package (as of 2026-03). — [wrk.dockerfile](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/wrk/wrk.dockerfile)
- **No cloud results recently.** The Round 22 and Round 23 entries in the site metadata list only a physical environment (no cloud/Azure run-details). — [site JS bundle](https://www.techempower.com/benchmarks/assets/index-rdjV3uRt.js)
- **Throughput jump from the hardware change (computed).** Top results, Round 22 (10 GbE) vs Round 23 (40 GbE):
  - JSON: 1,211,777 → 3,118,330 req/s
  - Plaintext: about 7.01M → 28.38M req/s
  - Single query: 628,505 → 1,448,273 req/s
  - Fortunes: 585,122 → 1,327,378 req/s
  - [R22 JSON](https://www.techempower.com/benchmarks/results/round22/ph.json); [R23 JSON](https://www.techempower.com/benchmarks/results/round23/ph.json)

### Inferences
- **"56 cores" most likely means 56 hardware threads per machine** (one 28-core Gold 6330). The old Citrine machines were described the same way ("28 HT cores" for a 14-core Gold 5120), and TechEmpower's 2026 post says "56 hyper-threaded cores". A dual-socket configuration is not ruled out by the sources.
- **The plaintext top tier is limited by the network card, not by the frameworks.**
  - The toolset sends three headers: Host, a roughly 86-character Accept header, and Connection: keep-alive. That makes each request about 165 bytes (my estimate from the wrk scripts).
  - At 28.4M req/s that is about 37 Gbit/s of request payload, which is close to the 40 GbE line rate once framing is added.
  - Round 22's plaintext ceiling of about 7.0M req/s works out to about 9.2 Gbit/s, i.e. the 10 GbE limit.
  - In Round 23, 11 realistic permutations are within 3% of the plaintext leader (computed, Q6). So plaintext cannot separate the top frameworks.
- **Numbers are not comparable across rounds, or with Meleys' own hardware.** The ratios between frameworks are more portable than the absolute values.

### Gaps
- Not documented in the sources: socket count per machine, kernel version, BIOS power settings, NUMA layout, switch model, and database versions (PostgreSQL/MySQL/MongoDB) for Round 23.

## Q3. What exactly does TFB test: test types and requirements, load-generator settings, required headers, what is forbidden, classification, scoring, and rule changes 2022–2026?

### Takeaway
There are seven HTTP/1.1 GET test types: JSON, single query, multiple queries, Fortunes, updates, plaintext (pipelined) and cached queries. They are driven by wrk in a closed loop. Each measurement lasts 15 seconds, after a 5-second primer and a 15-second warmup. Connection counts are 16–512 for most tests, 512 for the database query tests, and 256–16,384 with pipeline depth 16 for plaintext. Responses must include Server and Date headers plus Content-Length or Transfer-Encoding. Response caching, gzip, and coalescing or batching database reads are forbidden. Update batching is allowed. Database network pipelining is allowed only with separate-roundtrip semantics, and since March 2022 it requires one Sync per query on PostgreSQL. The site ranks peak requests per second minus errors. Entries are tagged Realistic or Stripped and Fullstack, Micro or Platform.

### Cited Findings
**General requirements** (wiki spec; substantive text unchanged since 2022-03; page last edited 2024-04-04) — [Wiki: Framework Tests Overview](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Framework-Tests-Overview)
- Implementations "should be production-grade". Implementations not based on a realistic HTTP implementation "will be marked as Stripped".
- All disk logging must be disabled.
- JSON keys are case-sensitive.
- `Server` and `Date` response headers are required for every test. A Server header must be added even if the framework doesn't normally send one. Re-rendering Date once per second is acceptable.
- Some form of request routing is required.
- Database queries must not be "coalesced or deduplicated at the database driver". Network-level pipelining of separate queries is permitted.
- Requests use a 4-digit port (e.g., 8080).

**JSON serialization**
- A `{"message":"Hello, World!"}` object must be instantiated per request and serialized with a JSON serializer. Serialization "must not be cached".
- About 28 bytes; `Content-Type: application/json`; Content-Length or Transfer-Encoding; Server and Date; no gzip.
- 16–512 concurrency, GET.
- [Wiki spec](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Framework-Tests-Overview)

**Single query**
- One random row (id 1–10,000) from the World table (10,000 rows), fetched as the whole row, not only randomNumber.
- Use an ORM, otherwise the entry is classified "raw".
- About 32 bytes. No in-memory caching of World rows. Prepared statements encouraged.
- 16–512 concurrency.
- [Wiki spec](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Framework-Tests-Overview)

**Multiple queries**
- `queries` parameter clamped to 1–500 (missing or invalid means 1).
- Each row is selected individually. `WHERE id IN (...)` and multiple SELECTs in one complex query are forbidden; network pipelining is allowed.
- Run at 512 concurrency only, with 1, 5, 10, 15 and 20 queries.
- [Wiki spec](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Framework-Tests-Overview)

**Fortunes**
- Fetch all rows of the Fortune table (a dozen rows) into a dynamic list. Add "Additional fortune added at request time." at request time.
- Sort by message in the application (no `ORDER BY`). Render with a server-side template that escapes HTML (one row contains a `<script>` tag). Output UTF-8, including a Japanese message.
- `text/html; charset=utf-8`; no caching; 16–512 concurrency.
- [Wiki spec](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Framework-Tests-Overview)

**Database updates**
- Read each row with its own query (no `IN`, batch or multi-SELECT reads). Set a new random randomNumber (1–10,000) and persist it.
- "Bulk updates—batches of update statements or individual update statements that affect multiple rows—is acceptable". Transactions are acceptable.
- Raw tests must give each row a unique new value.
- 512 concurrency, 1–20 updates.
- [Wiki spec](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Framework-Tests-Overview)

**Plaintext**
- Body "Hello, World!", `text/plain`. Reusing one buffer for the body is acceptable, but "the rest of the response must be fully composed on the spot"; a pre-rendered whole response is not allowed.
- HTTP/1.1 pipelining "is assumed".
- 256, 1,024, 4,096 and 16,384 concurrency.
- [Wiki spec](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Framework-Tests-Overview)

**Cached queries**
- `count` parameter 1–500, served from a CachedWorld table.
- Must use a bona fide cache library or server ("Guava CacheBuilder" in-VM or "Redis" as references), not a plain map. No reverse-proxy caching. The cache runs in-process or on localhost, and pre-priming is allowed.
- The spec says 256 concurrency, with counts 1, 10, 20, 50 and 100.
- [Wiki spec](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Framework-Tests-Overview)

**Load generator settings** (toolset at archive, 2026-03)
- Each script runs three phases, all wrk with `--latency` and `--timeout 8`:
  - Primer: `-d 5 -c 8 -t 8`.
  - Warmup: `-d $duration -c $max_concurrency -t $(nproc)`.
  - Measurement: one run per level with `-d $duration -c $c -t min(c, nproc)`, with 2-second sleeps between levels.
- Headers sent: `Host`, a type-specific `Accept`, and `Connection: keep-alive`.
- Plaintext adds `-s pipeline.lua -- 16`. The Lua script concatenates N copies of the request, so pipeline depth is 16.
- [concurrency.sh](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/wrk/concurrency.sh); [pipeline.sh](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/wrk/pipeline.sh); [query.sh](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/wrk/query.sh); [pipeline.lua](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/wrk/pipeline.lua); [plaintext.py](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/test_types/plaintext/plaintext.py)
- Accept header values:
  - JSON: `application/json,text/html;q=0.9,application/xhtml+xml;q=0.9,application/xml;q=0.8,*/*;q=0.7`
  - HTML: `text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8`
  - Plaintext: `text/plain,text/html;q=0.9,…`
  - [abstract_test_type.py](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/test_types/abstract_test_type.py)
- Default settings:
  - `--duration 15`
  - `--concurrency-levels [16, 32, 64, 128, 256, 512]`
  - `--pipeline-concurrency-levels [256, 1024, 4096, 16384]`
  - `--query-levels [1, 5, 10, 15, 20]`
  - `--cached-query-levels [1, 10, 20, 50, 100]`
  - [run-tests.py](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/run-tests.py)
- Warmup (all types) and the query-type tests (multiple queries, updates, cached) use `max_concurrency = max(concurrency_levels)`, which is 512. — [json.py](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/test_types/json/json.py); [cached-query.py](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/test_types/cached-query/cached-query.py); [plaintext.py](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/test_types/plaintext/plaintext.py)
- **The site FAQ disagrees with the scripts on warmup.** It says:
  - Restart the database servers.
  - Start the framework.
  - Run a "5-second primer at 8 client-concurrency".
  - Run a "15-second warmup at 256 client-concurrency".
  - Run a 15-second captured test per level.
  - Stop the framework.
  It also says the JIT "has even completed its work already after just the primer" in their JVM spot tests, and that durations were cut from 60 to 15 seconds to fit the growing suite. — [site JS bundle (FAQ text)](https://www.techempower.com/benchmarks/assets/index-rdjV3uRt.js)

**Verification enforced by the toolset**
- Fails if `Server`, `Date` or `Content-Type` is missing, or if neither Content-Length nor Transfer-Encoding is present.
- Warns on a malformed Date. Fails if Date is identical on two separate requests ("Invalid Cached Date").
- Checks Content-Type, JSON keys, and edge cases of the queries parameter (`0`, `foo`, `501`, empty).
- [verifications.py](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/test_types/verifications.py)
- Database tests are verified by counting executed statements through PostgreSQL `pg_stat_statements`:
  - "No margin is accepted on the number of queries" in real runs (Citrine/Azure); 1.5% margin in CI.
  - 1% margin on rows read.
  - 5% `bulk_margin` on updated rows when bulk updates are detected.
  - Transaction failures are flagged.
  - [verifications.py](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/test_types/verifications.py); [postgres.py](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/databases/postgres/postgres.py)

**Classification metadata** (set per test in benchmark_config.json) — [Wiki: Framework Files](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Codebase-Framework-Files)
- `approach`: Realistic or Stripped.
- `classification`: Fullstack, Micro or Platform.
- `orm`: Full, Micro or Raw.
- `database`: MySQL, Postgres, MongoDB, … or None.
- `platform`, `webserver`, `os` (Linux or Windows), `database_os`, `display_name`.
- `versus`: the underlying platform, used for the "framework overhead" comparison.
- "Framework" is used loosely for "any HTTP stack—a full-stack framework, a micro-framework, or even a web platform such as Rack, Servlet, or plain PHP." — [Wiki: Basic Concepts](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Basic-Concepts)

**Realistic vs Stripped (site FAQ)**
- Realistic "uses the framework with most out-of-the-box functionality enabled".
- Stripped "removes or outright avoids implementing features that are unnecessary for the particulars of this benchmark exercise". The definition was broadened by issue #2783.
- Stripped entries are hidden by default because they are "specially crafted to excel at our benchmark".
- Round 23 had 15 Stripped permutations out of 841 (computed).
- [site JS bundle](https://www.techempower.com/benchmarks/assets/index-rdjV3uRt.js); [R23 JSON](https://www.techempower.com/benchmarks/results/round23/ph.json)

**How the site ranks results** (current JS bundle)
- successful = totalRequests − non-2xx/3xx ("5xx") − socket read/write/connect errors (timeouts not subtracted). RPS = floor(successful / 15 s).
- "Best" is the peak across concurrency levels for JSON, single query, Fortunes and plaintext.
- Multiple queries and updates are charted at 20 queries per request ("Responses per second at 20 queries per request"; "Latency of 20-update responses").
- [site JS bundle](https://www.techempower.com/benchmarks/assets/index-rdjV3uRt.js); [Wiki: Results Website FAQ](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Results-Website)

**Composite score**
- Introduced in Round 19 (2020-05-28). — [site JS bundle](https://www.techempower.com/benchmarks/assets/index-rdjV3uRt.js)
- Algorithm (wiki page still marked work-in-progress):
  - Only frameworks implementing all test types are scored.
  - Per round, each test type's average RPS over reference "TPR-tagged" frameworks is normalized to the JSON average.
  - Biases are applied: JSON 1, single query 0.75, 20-query 0.75, Fortunes 1.5, updates 1.25, plaintext 0.75.
  - Weighted results are summed and divided by 1,000.
  - [Wiki: Composite Scoring](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Composite-Scoring-for-Frameworks); [Wiki: TPR](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/TechEmpower-Performance-Rating-(TPR))
- The current bundle marks JSON, 1-query, 20-query, Fortunes, Updates and Plaintext as "composable"; cached queries are not. — [site JS bundle](https://www.techempower.com/benchmarks/assets/index-rdjV3uRt.js)
- A user asked on 2025-08-29 "Why I cannot see composite scores tab?". — [Discussions](https://github.com/TechEmpower/FrameworkBenchmarks/discussions)

**Rule changes 2022–2026** (wiki revision history) — [Wiki history](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Framework-Tests-Overview/_history)
- **2022-03-08, rev a6b3692 (Nate).** Added to the database rule: "If any pipelining or network optimization is performed, then the execution behavior and semantics must match what they would be if the multiple SQL statements were executed without the optimization, i.e. in separate roundtrips… it is forbidden to perform any optimization that could change the transactionality and/or error handling… even if they wouldn't produce the behavioral change with the specific SQL statements of the benchmarks."
- **2022-03-22, rev 33e74ec (Michael Hixson).** "single complex query" became "single complex query or transaction". Added: "If using PostgreSQL's extended query protocol, each query must be separated by a Sync message."
- **2022-03-31.** Cached test URI renamed from /cached-worlds to /cached-queries.
- **2024-04-04.** Minor wording edit to the caching description.
- No further spec changes before the archive.
- Earlier context: the pipelining-permitted clause and the "Stripped" sentence date from 2019-08-22.
- Round 18 added clarifications on how often Date must be recomputed and stricter validation. — [site JS bundle](https://www.techempower.com/benchmarks/assets/index-rdjV3uRt.js)

### Inferences
- **TFB measures closed-loop throughput at fixed connection counts, over short 15-second windows.** It is not a latency-SLO or open-loop test. The headline number is the best level, so a framework is credited with its peak and degradation at other levels is not penalised in the bar chart.
- **The 2022 Sync-per-query rule is aimed at the reactive PostgreSQL drivers' batching.** It is the most important database rule change of the period. Update batching remains explicitly allowed, which favours drivers and frameworks that batch UPDATEs.
- **The spec and the scripts disagree on two details** (warmup and cached-query concurrency, 256 vs 512). The toolset is authoritative for how results were actually produced.

### Gaps
- I could not confirm whether the current site still renders composite scores for Round 23, or what the Round 23 per-round weights were. The bundle has no weight constants.

## Q4. How does the toolset work in practice, what raw data does it produce, and which metrics are visible to readers?

### Takeaway
The `./tfb` script runs a Python toolset inside a `techempower/tfb` Docker container. It builds each test's Dockerfile, verifies the endpoints, runs wrk from the client machine, and writes `results.json` plus per-test logs, raw wrk output, host statistics (dool) and Docker container statistics (memory and CPU). The website only shows requests per second, error counts, framework overhead, and wrk's average, standard deviation and maximum latency. It shows no percentiles (no p99), no CPU and no memory. The raw files that held those were linked from tfb-status, which is down.

### Cited Findings
- **Quick start.** `git clone …`; `./tfb --mode verify --test gemini`. Docker or Vagrant is required. — [README](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/README.md)
- **Local commands.**
  - `tfb --list-tests`
  - `tfb --test X --mode verify` (verification only)
  - `--mode debug` (skips verification, leaves the server up)
  - `tfb --test X` (benchmark)
  - `--type json|db|fortune…`
  - `--test-container-memory 1G --extra-docker-runtime-args cpu_period:10000 cpu_quota:50000`
  - Logs go to `results/<run name>/<timestamp>/logs/<test name>/…`.
  - [Wiki: Testing and Debugging](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Development-Testing-and-Debugging)
- **All CLI options.** `--cpuset-cpus`, `--audit`, `--new` (scaffolding wizard), `--results-name`, `--results-environment`, `--results-upload-uri`, `--parse`, `--test`, `--test-dir`, `--test-lang`, `--tag`, `--exclude`, `--type`, `--mode` (default `benchmark`), `--list-tests`, `--list-tag`, `--duration`, `--server-host` (default `tfb-server`), `--database-host` (`tfb-database`), `--client-host`, the level lists, `--test-container-memory`, `--extra-docker-runtime-args`, `--force-rm`, `--network-mode`. — [run-tests.py](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/run-tests.py)
- **Three roles.** App server, load generator and database server. They can be one machine; the toolset is run on the app server. — [Wiki: Basic Concepts](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Basic-Concepts). For development "it is usually adequate to only use one computer" (all hosts 127.0.0.1). — [Wiki: Benchmarking Getting Started](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Benchmarking-Getting-Started)
- **Continuous runner loop.**
  - Shallow-clone the branch (`--depth 1`).
  - Alternate normal and `--reverse-order` on every other run.
  - Build `techempower/tfb` and run the full suite with `--network-mode host --results-upload-uri …`.
  - Zip `results/` and POST it to the upload URI (tfb-status).
  - [tfb-startup.sh](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/continuous/tfb-startup.sh)
- **Fields parsed from wrk per level:** `latencyAvg`, `latencyStdev`, `latencyMax`, `totalRequests`, socket `connect`/`read`/`write`/`timeout` errors, `5xx` (non-2xx/3xx), `startTime`, `endTime`. wrk's `--latency` percentile lines are not parsed. — [results.py](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/utils/results.py); [concurrency.sh](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/wrk/concurrency.sh)
- **results.json top-level keys:** uuid, name, environmentDescription, git (commitId, repositoryUrl, branchName), startTime, completionTime, concurrencyLevels, pipelineConcurrencyLevels, queryIntervals, cachedQueryIntervals, frameworks, duration, rawData (per test type, plus commitCounts and slocCounts), completed, succeeded, failed, verify, testMetadata. — [R23 JSON](https://www.techempower.com/benchmarks/results/round23/ph.json)
- **Host statistics.** During each test the toolset runs `dool -Tafilmprs --aio --fs --ipc --lock --socket --tcp --raw --udp --unix --vm --disk-util --rpc --rpcd --output stats.txt`. results.py parses the CPU, "memory usage" and disk read/write columns. — [benchmarker.py](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/benchmark/benchmarker.py); [results.py](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/utils/results.py)
- **Container statistics.** The toolset streams `container.stats(stream=True)` from the framework container into `docker_stats.json`, and container logs into a raw file, per test type. — [benchmarker.py](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/benchmark/benchmarker.py)
- **Website views per test type:** "Best (bar chart)", "Data table" (RPS per concurrency/query level), "Latency" (a table of avg/max/stdev only), and "Framework overhead" (vs the `versus` platform). There is also an errors count. — [site JS bundle](https://www.techempower.com/benchmarks/assets/index-rdjV3uRt.js); [Wiki: Results Website](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Results-Website)
- **No CPU or memory on the site (own check).** The current site bundle contains no "dool", "CPU usage", "memory usage" or "Memory" strings, i.e. no CPU or memory display. — [site JS bundle](https://www.techempower.com/benchmarks/assets/index-rdjV3uRt.js). A 2024-03-02 discussion asked to "Add websocket benchmark + memory used + cpu %". — [Discussions](https://github.com/TechEmpower/FrameworkBenchmarks/discussions)
- **Raw-log links point to tfb-status.** Round 23 logs were linked at `tfb-status.techempower.com/unzip/results.2025-02-05-22-40-37-331.zip/results/20250130184741`, and run details at `tfb-status.techempower.com/results/91a66052-…`. — [site JS bundle](https://www.techempower.com/benchmarks/assets/index-rdjV3uRt.js). Both hosts were unreachable on 2026-10-21 (own check).
- **What the published latency looks like (computed, Round 23, JSON at 512 connections):**
  - edap-http-fast: avg 164.6 µs, max 47.09 ms
  - smart-socket: avg 162.3 µs, max 1.87 ms
  - vertx: avg 166.9 µs, max 4.02 ms
  - [R23 JSON](https://www.techempower.com/benchmarks/results/round23/ph.json)

### Inferences
- **For Meleys' latency and memory goals, TFB's public output is inadequate.** It has no p99 or higher percentiles, wrk's closed-loop latency suffers from coordinated omission, and memory is unpublished. The toolset does capture container memory per test (`docker_stats.json`). Running the toolset privately would therefore give Meleys memory data for itself and for competitor images under identical load, and wrk's `--latency` percentiles are in the raw logs.
- **Reproducing official conditions** needs three Linux hosts with Docker and the TFB repo at a fixed commit. Round 23 used commit 523534b.

### Gaps
- I could not inspect an actual run archive (tfb-status down) to confirm which parsed statistics files (e.g., stats.json) were shipped in the zips.

## Q5. What did a new framework submission require, how long did review and listing take, and is submission still possible?

### Takeaway
Historically, a submission was a PR to TechEmpower/FrameworkBenchmarks adding `frameworks/Java/<name>/` with:
- `benchmark_config.json`
- one `<test>.dockerfile` per test (a default test is mandatory)
- README.md
- only the source needed for the benchmark

GitHub Actions verified each changed language directory in Docker. Review was fast: median 2.5 days for "add" PRs between August 2025 and March 2026. Merged code was picked up by the next continuous run (about 6.5 days long). Official rounds came roughly every 15–16 months. Since 2026-03-24 the repository is archived and accepts no PRs, so a new submission is impossible.

### Cited Findings
- **Required files.**
  - `benchmark_config.json` at the framework root: `framework`, `maintainers` (GitHub users pinged on PRs), and `tests` with URLs (`json_url`, `db_url`, `query_url`, `fortune_url`, `update_url`, `plaintext_url`, `cached_query_url`), `port`, optional `dockerfile`/`docker_cmd`, plus the metadata listed in Q3.
  - "All frameworks must have a default test with a corresponding dockerfile" named `<test-name>.dockerfile`.
  - The build receives Docker ARGs `BENCHMARK_ENV` (e.g., Citrine/Azure), `TFB_TEST_NAME` and `TFB_TEST_DATABASE`.
  - [Wiki: Framework Files](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Codebase-Framework-Files)
- **Process.**
  - `tfb --new` runs a scaffolding wizard.
  - Implement at least one test ("we'd be even happier with six").
  - Add framework and language READMEs.
  - Lock all dependency versions.
  - Pass verification locally and in your fork's GitHub Actions before opening the PR.
  - `$BENCHMARK_ENV` may be used for environment-specific tuning.
  - [Wiki: How to Add Frameworks](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Development-How-To-Add-Frameworks-or-Tests); [Wiki: Contributing Guide](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Development-Contributing-Guide)
- **PR template.** New frameworks need a README.md linking the homepage and docs. Do not include functional tests ("JUnit tests in a src/test directory"), standalone startup scripts, or local dev configs. — [PULL_REQUEST_TEMPLATE.md](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/.github/PULL_REQUEST_TEMPLATE.md)
- **CI.**
  - `build.yml`: a setup job diffs against master. A verify matrix then runs one job per language directory on ubuntu-22.04. Each job builds `techempower/tfb` and runs `--mode verify --force-rm --test-dir $RUN_TESTS --results-environment Github-Actions`.
  - Separate workflows label failing PRs (`label-failing-pr.yml`) and ping maintainers (`get-maintainers.yml`, `ping-maintainers.yml`).
  - [build.yml](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/.github/workflows/build.yml); [label-failing-pr.yml](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/.github/workflows/label-failing-pr.yml); [ping-maintainers.yml](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/.github/workflows/ping-maintainers.yml)
  - Commit-message hooks: `[ci skip]`, `[ci run-all]`, `[ci fw <dirs>]`, `[ci fw-only <dirs>]`, `[ci lang <langs>]`, `[ci lang-only <langs>]`.
  - A partial verification takes 10–15 minutes; a full verification takes 3–5 hours. Using your own fork's Actions queue is strongly recommended.
  - [Wiki: GitHub Actions](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-GitHub-Actions)
- **Review times (computed from GitHub search, 60 merged PRs with "add" in the title, created 2025-08-17 to 2026-03-09).** Open-to-merge median 2.5 days, p75 5.5 days, maximum 21.2 days. Examples of new-framework PRs:
  - loveqq (Java): 14.7 days (#10342)
  - vortex (Rust): 8.6 days
  - oxelot-http (Zig): 12.3 days
  - Hotaru (Rust): 7.4 days
  - Piko: 5.3 days
  - Effinitive (C#): 0.5 days
  - simplew: 0.9 days
  - [PR search](https://github.com/TechEmpower/FrameworkBenchmarks/pulls?q=is%3Apr+is%3Amerged+add+in%3Atitle+created%3A%3E2025-03-01)
- **When merged code got benchmarked.** Continuous runs clone master at start. — [tfb-startup.sh](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/continuous/tfb-startup.sh). Recent full runs took 155.9 h (R23) and 158.8 h (R22) (computed). — [R23 JSON](https://www.techempower.com/benchmarks/results/round23/ph.json); [R22 JSON](https://www.techempower.com/benchmarks/results/round22/ph.json). Official rounds are selected continuous runs; hand-made previews were dropped in favour of continuous benchmarking. — [Wiki: Basic Concepts](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Basic-Concepts)
- **Maintenance expectations.**
  - Maintainers are listed in `benchmark_config.json` and pinged automatically. — [Wiki: Framework Files](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Codebase-Framework-Files)
  - A "Verified" tag lets a framework maintainer vouch for an implementation (work in progress). — [Wiki: Verified Tag](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Verified-Tag). Maintainers and tags fields were added to configs in PR #10721 (2026-02). — [PR search](https://github.com/TechEmpower/FrameworkBenchmarks/pulls?q=is%3Apr+is%3Amerged+add+in%3Atitle+created%3A%3E2025-03-01)
  - Dependabot bumps were merged up to 2026-03-23. — [master commits](https://github.com/TechEmpower/FrameworkBenchmarks/commits/master)
  - Implementations that break drop out of a round. Actix and all its variants "failed to build in the round run" for Round 23 and returned in later continuous runs after fixes. — [#9589](https://github.com/TechEmpower/FrameworkBenchmarks/issues/9589)
- **Example of a dropped Java entry.** Redkale (a top Java entry in Round 22: JSON 1,210,086 req/s, 99.9% of the leader) is listed in Round 23 metadata but has no verification or results entries (computed). — [R23 JSON](https://www.techempower.com/benchmarks/results/round23/ph.json); [R22 JSON](https://www.techempower.com/benchmarks/results/round22/ph.json)
- **Submission is no longer possible.** The repository is archived and read-only since 2026-03-24. — [issue #10932](https://github.com/TechEmpower/FrameworkBenchmarks/issues/10932)

### Inferences
- **Typical historical latency.** From PR open to first appearance in continuous results was roughly 1–3 weeks: a few days of review, up to about 7 days waiting for the next run to start, and up to about 6.5 days until the framework's slot in the run. Inclusion in an official round could take over a year.
- **Maintenance burden was real.** Pinned JDK/base-image versions rot (several Java entries still used JDK 8/11 images in 2025). Failing to build silently removes an entry from a round.
- **Options for Meleys now.** Either reuse the TFB spec and toolset privately (BSD-style license; fork plus own hardware), or target an active successor. Successors are covered by another researcher.

### Gaps
- No direct measurement of time from merge to first continuous result for a new Java framework, because tfb-status is down.

## Q6. How do Java frameworks rank in the latest round (Round 23), per test type, with stacks, JDK versions, flags and classification?

### Takeaway
On Round 23 hardware (data from 2025-01-30 to 2025-02-06), the best Java entries are near the top in database-bound tests:
- updates: inverno-postgres #2, 99.6% of the leader
- multiple queries: quarkus-vertx #4 (99.4%) and vertx-postgres #5 (99.1%)
- single query: wizzardo-http #3 (94.7%)

Java is also close in JSON (edap-http-fast #6, 96.8%). It trails in Fortunes (best vertx-postgres #13, 78.4%), and only one Java entry (firenio-http-lite #11, 97.8%) reaches plaintext's saturated top tier. The leaders overall are C, Rust and JS/Python platform entries. Most top Java entries are "Platform" or "Micro" with raw database access, built on Vert.x/Netty or custom NIO servers. They ran JDK 8 to 22 at the time, with `-XX:+UseParallelGC -XX:+UseNUMA` and fixed 2 GB heaps. Among mainstream frameworks, Quarkus (reactive) is strong, Micronaut is mid-pack, and Spring, Javalin and Dropwizard sit far down.

### Cited Findings
All numbers below are computed from the [Round 23 results JSON](https://www.techempower.com/benchmarks/results/round23/ph.json) with the site's formula ([site JS bundle](https://www.techempower.com/benchmarks/assets/index-rdjV3uRt.js)). Values are req/s, rank is among Realistic permutations, and % is relative to the Realistic leader. Classification and platform come from Round 23 `testMetadata`.

**Scale of Java in Round 23**
- 145 of 841 permutations are Java, covering 51 of 331 distinct frameworks. Round 22 had 142 of 791 and 48 of 301.
- Java entries with results per test: JSON 88, single query 85, Fortunes 90, plaintext 88, multiple queries 83, updates 77, cached 14.

**JSON** (leader libreactor-server, C: 3,118,330)

| Rank | Entry | Classification | Platform | Req/s | % of leader |
|---|---|---|---|---|---|
| #6 | edap-http-fast | Platform | edap | 3,019,148 | 96.8% |
| #7 | edap-http | Platform | edap | 3,002,791 | 96.3% |
| #8 | wizzardo-http | Micro | none | 2,972,955 | 95.3% |
| #9 | firenio-http-lite | Platform | firenio | 2,972,507 | 95.3% |
| #11 | smart-socket | Platform | smartboot | 2,948,084 | 94.5% |
| #14 | vertx | Platform | vert.x | 2,889,592 | 92.7% |
| #17 | firenio | Platform | firenio | 2,866,762 | 91.9% |
| #27 | quarkus-vertx | Micro | vert.x | 2,756,818 | 88.4% |
| #29 | rapidoid-http-fast | Platform | rapidoid | 2,747,166 | 88.1% |

Clustering: 3 realistic permutations within 1% of the leader, 5 within 3%, 21 within 10%.

**Single query** (leader may-minihttp, Rust: 1,448,273)

| Rank | Entry | Classification | Platform | Req/s | % of leader |
|---|---|---|---|---|---|
| #3 | wizzardo-http | Micro | none | 1,372,100 | 94.7% |
| #6 | vertx-postgres | Platform | vert.x | 1,315,264 | 90.8% |
| #11 | vertx-web-postgres | Micro | vert.x | 1,262,001 | 87.1% |
| #13 | quarkus-vertx | Micro | vert.x | 1,258,371 | 86.9% |
| #17 | quarkus-reactive-routes-pgclient | Fullstack | vert.x | 1,181,497 | 81.6% |
| #19 | inverno-postgres | Micro | inverno | 1,134,919 | 78.4% |
| #20 | jooby-pgclient | Fullstack | netty | 1,105,323 | 76.3% |
| #22 | officefloor-rawsqlclient | Platform | officefloor | 1,076,847 | 74.4% |
| #39 | revenj-jvm | Fullstack | Servlet/Resin | 957,604 | 66.1% |

**Fortunes** (leader may-minihttp, Rust: 1,327,378)

| Rank | Entry | Classification | Platform | Req/s | % of leader |
|---|---|---|---|---|---|
| #13 | vertx-postgres | Platform | vert.x | 1,040,599 | 78.4% |
| #14 | quarkus-vertx | Micro | vert.x | 1,028,408 | 77.5% |
| #21 | jooby-pgclient | Fullstack | netty | 917,719 | 69.1% |
| #22 | quarkus-reactive-routes-pgclient | Fullstack | vert.x | 916,813 | 69.1% |
| #23 | officefloor-rawsqlclient | Platform | officefloor | 911,341 | 68.7% |
| #27 | vertx-web-postgres | Micro | vert.x | 856,756 | 64.5% |
| #30 | inverno-postgres | Micro | inverno | 798,618 | 60.2% |
| #40 | helidon-nima | Micro | none | 696,144 | 52.4% |
| #45 | micronaut | Micro | netty | 660,238 | 49.7% |

Only 1 realistic permutation within 3% of the leader, 4 within 10%.

**Multiple queries, 20 per request** (leader "just", JavaScript: 89,470; the Stripped xitca-web-unrealistic at 89,666 is hidden by default)

| Rank | Entry | Classification | Req/s | % of leader |
|---|---|---|---|---|
| #4 | quarkus-vertx | Micro | 88,902 | 99.4% |
| #5 | vertx-postgres | Platform | 88,708 | 99.1% |
| #12 | inverno-postgres | Micro | 87,919 | 98.3% |
| #15 | wizzardo-http | Micro | 86,504 | 96.7% |
| #17 | quarkus-reactive-routes-pgclient | Fullstack | 86,323 | 96.5% |
| #18 | jooby-pgclient | Fullstack | 86,183 | 96.3% |
| #19 | officefloor-rawsqlclient | Platform | 85,997 | 96.1% |
| #21 | micronaut | Micro | 85,904 | 96.0% |
| #22 | helidon | Micro | 85,818 | 95.9% |

The JVM/Kotlin entry vertx-web-kotlin-dsljson-postgresql is #2 at 89,289. Clustering: 14 permutations within 3%, 32 within 10%.

**Updates, 20 per request** (leader viz-pg, Rust: 59,374; the Stripped xitca at 63,028 is hidden)

| Rank | Entry | Classification | Req/s | % of leader |
|---|---|---|---|---|
| #2 | inverno-postgres | Micro | 59,146 | 99.6% |
| #7 | vertx-postgres | Platform | 58,300 | 98.2% |
| #13 | quarkus-vertx | Micro | 57,616 | 97.0% |
| #15 | officefloor-rawsqlclient | Platform | 56,791 | 95.6% |
| #21 | jooby-pgclient | Fullstack | 52,019 | 87.6% |
| #26 | officefloor-sqlclient | Platform | 46,050 | 77.6% |
| #27 | vertx-web-postgres | Micro | 45,893 | 77.3% |
| #30 | wizzardo-http | Micro | 45,641 | 76.9% |
| #35 | micronaut | Micro | 43,322 | 73.0% |

Example latency at 20 updates: inverno-postgres avg 8.56 ms, max 20.3 ms.

**Plaintext** (leader mrhttp, Python: 28,384,305)

| Rank | Entry | Classification | Req/s | % of leader | Latency note |
|---|---|---|---|---|---|
| #11 | firenio-http-lite | Platform | 27,755,180 | 97.8% | |
| #23 | wizzardo-http | Micro | 22,212,355 | 78.3% | |
| #24 | rapidoid-http-fast | Platform | 20,054,554 | 70.7% | |
| #25 | rapidoid | Platform | 19,319,963 | 68.1% | |
| #26 | activej | Micro | 19,171,625 | 67.5% | |
| #32 | edap-http | Platform | 17,832,353 | 62.8% | max latency 2.76 s |
| #36 | firenio | Platform | 16,991,116 | 59.9% | |
| #44 | vertx | Platform | 14,340,940 | 50.5% | |
| #45 | officefloor-sqlclient | Platform | 13,994,309 | 49.3% | |
| #47 | jooby-netty | Fullstack | 13,795,851 | 48.6% | |

Clustering: 11 realistic permutations within 3% of the leader.

**Cached queries, count 100** (leader mormot-postgres-raw, Pascal: 1,411,733; only 128 permutations implement this test)

| Rank | Entry | Classification | Platform | Req/s | % of leader |
|---|---|---|---|---|---|
| #11 | servlet-postgresql | Platform | Servlet/Resin | 993,474 | 70.4% |
| #13 | officefloor-r2dbc | Platform | officefloor | 976,603 | 69.2% |
| #16 | vertx-postgres | Platform | vert.x | 836,816 | 59.3% |
| #19 | gemini-postgres | Fullstack | Servlet/Resin | 633,410 | 44.9% |

Gemini is TechEmpower's in-house framework.

**Mainstream Java frameworks** (best permutation per framework; rank / req/s / % of leader)

| Framework | JSON | Single query | Fortunes | 20 queries | 20 updates | Plaintext |
|---|---|---|---|---|---|---|
| Quarkus (quarkus-vertx) | #27 / 2.76M / 88% | #13 / 1.26M / 87% | #14 / 1.03M / 77% | #4 / 88.9K / 99% | #13 / 57.6K / 97% | #53 / 13.1M / 46% |
| Vert.x (vertx; vertx-postgres for DB tests) | #14 / 2.89M / 93% | #6 / 1.32M / 91% | #13 / 1.04M / 78% | #5 / 88.7K / 99% | #7 / 58.3K / 98% | #44 / 14.3M / 51% |
| Micronaut | #155 / 1.47M / 47% | #50 / 837K / 58% | #45 / 660K / 50% | #21 / 85.9K / 96% | #35 / 43.3K / 73% | #156 / 2.88M / 10% |
| Helidon (helidon-nima for JSON/Fortunes/plaintext, helidon otherwise) | #86 / 2.23M / 72% | #96 / 594K / 41% | #40 / 696K / 52% | #22 / 85.8K / 96% | #135 / 17.0K / 29% | #86 / 8.42M / 30% |
| Spring (WebFlux/MVC variants) | #171 / 1.29M / 41% (spring-webflux-mongo) | #193 / 323K / 22% (spring-webflux) | #177 / 246K / 19% | #236 / 31.6K / 35% (spring) | #74 / 29.3K / 49% | #170 / 2.51M / 9% |
| Javalin | #176 / 1.27M / 41% | #134 / 463K / 32% | #112 / 347K / 26% | #152 / 39.5K / 44% | #162 / 14.6K / 25% | #195 / 1.82M / 6% |
| Dropwizard | #320 / 431K / 14% | #279 / 183K / 13% | #277 / 120K / 9% | #127 / 45.5K / 51% | #73 / 29.8K / 50% | #331 / 487K / 2% |
| Netty (raw; JSON and plaintext only) | #37 / 2.71M / 87% | | | | | #60 / 12.2M / 43% |
| ActiveJ | #51 / 2.63M / 84% | | | | | #26 / 19.2M / 68% |
| Jooby (Fullstack: jooby-netty; jooby-pgclient for DB tests) | #38 / 2.71M / 87% | #20 / 1.11M / 76% | #21 / 918K / 69% | #18 / 86.2K / 96% | #21 / 52.0K / 88% | #47 / 13.8M / 49% |

**Round 22 for context** (Dell R440, 10 GbE; data from 2023-10; % of overall leader including Stripped) — [R22 JSON](https://www.techempower.com/benchmarks/results/round22/ph.json)
- JSON: redkale-graalvm 1,210,086 (99.9%)
- Single query: vertx-postgres 581,542 (92.5%)
- Fortunes: vertx-postgres 453,406 (77.5%)
- Plaintext: firenio-http-lite 7,004,779 (99.9%)
- 20 queries: vertx-postgres 32,505 (94.0%)
- Updates: redkale-graalvm 22,900 (96.2%)
- Cached: redkale-graalvm 352,969 (99.9%)

**Base images and JVM flags at the Round 23 commit** (523534b, 2025-01-23) — [R23-commit Java tree](https://github.com/TechEmpower/FrameworkBenchmarks/tree/523534bb61450e3522d775a749ad060753e26e3a/frameworks/Java)

| Entry | JDK / base image | Notable flags |
|---|---|---|
| edap-http-fast | openjdk:8u275-jdk-slim | `-server -XX:+UseNUMA -XX:+UseParallelGC -XX:+AggressiveOpts -Dedap.http.decoder.type=fast` |
| wizzardo-http | openjdk:11.0.3-jdk-slim | `-Xmx2G -Xms2G -server -XX:+UseNUMA -XX:+UseParallelGC -XX:+AggressiveOpts` |
| firenio-http-lite | JDK 11 (maven:3.6.1-jdk-11-slim) | `-XX:+UseNUMA -XX:+UseParallelGC -XX:+AggressiveOpts` |
| smart-socket | openjdk:21-jdk-slim | `-server -XX:+UseNUMA -XX:+UseParallelGC` |
| vertx / vertx-postgres | Temurin 17 | `-Xms2G -Xmx2G -server -XX:+UseNUMA -XX:+UseParallelGC -Djava.lang.Integer.IntegerCache.high=10000 -Dvertx.disableMetrics=true -Dvertx.disableH2c=true -Dvertx.disableWebsockets=true -Dvertx.flashPolicyHandler=false …` |
| quarkus-vertx | Red Hat UBI8 OpenJDK 17 runtime 1.15 | launched by `run_quarkus.sh` |
| inverno-postgres | Temurin 21 | |
| jooby-pgclient | Temurin 22 | `-Xms2g -Xmx2g -XX:+UseNUMA -XX:+UseParallelGC -Dio.netty.disableHttpHeadersValidation=true -Dio.netty.buffer.checkBounds=false -Dio.netty.buffer.checkAccessible=false` |
| officefloor-rawsqlclient | openjdk:15-slim | `-Xms2g -Xmx2g -XX:+UseNUMA -XX:+UseParallelGC` plus Vert.x disable flags |
| helidon-nima | openjdk:22-rc-jdk-slim | `-XX:+UseNUMA -XX:+UseParallelGC` |
| micronaut | openjdk:21 | `run_benchmark.sh` |
| netty | openjdk:11.0.3-jdk-slim | `-XX:+UseNUMA -XX:+UseParallelGC -XX:+AggressiveOpts -Dio.netty.buffer.checkBounds=false -Dio.netty.buffer.checkAccessible=false -Dio.netty.iouring.iosqeAsyncThreshold=32000` |
| spring | Liberica OpenJRE 23 | `-XX:+DisableExplicitGC -XX:+UseStringDeduplication -Dlogging.level.root=OFF` |

**State at archive** (master, March 2026) — [frameworks/Java](https://github.com/TechEmpower/FrameworkBenchmarks/tree/master/frameworks/Java)
- 68 Java framework directories.
- Several entries moved to newer JDKs after Round 23:
  - vertx: Temurin 24, adding `--enable-native-access=ALL-UNNAMED --sun-misc-unsafe-memory-access=allow -XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints`
  - vertx-web and jooby: Temurin 25
  - smart-socket: Corretto 25.0.1-alpine
  - edap-http: Corretto 21.0.8
  - helidon-nima: openjdk:25-ea
  - micronaut: Oracle OpenJDK 25.0.2 and GraalVM native-image 25 variants
  - quarkus: UBI9 OpenJDK 21
  - netty: Temurin 24
- Others stayed on JDK 11: wizzardo-http, firenio, rapidoid, servlet/Resin, gemini/Resin, hserver. OfficeFloor stayed on openjdk:15.
- Directories that reference virtual threads or Loom: netty (incl. netty-loom), micronaut, smart-socket, helidon, tio-boot, avaje-jex (own grep).

### Inferences
- **Benchmark targets for "fastest Java" on TFB hardware:**

  | Test | Best Java entry | Req/s |
  |---|---|---|
  | JSON | edap-http-fast | ≥3.02M |
  | Single query | wizzardo-http | ≥1.37M |
  | Fortunes | vertx-postgres | ≥1.04M |
  | 20 queries | quarkus-vertx | ≥88.9K |
  | 20 updates | inverno-postgres | ≥59.1K |
  | Plaintext | firenio-http-lite | ≥27.8M (network-bound, see Q2) |

- **The database tests barely separate the top entries.** Multiple queries and updates cluster tightly (14 entries within 3% for queries), so they mostly reward the PostgreSQL driver: the reactive Vert.x PG client used by quarkus-vertx, vertx-postgres, jooby-pgclient and officefloor-*sqlclient (inferred from entry names and platforms). Fortunes (best Java at 78% of the leader) and single query are where Java's per-request overhead shows most.
- **Many top Java entries are custom NIO platforms, not mainstream frameworks.** These include edap, firenio, smart-socket, wizzardo, rapidoid and voovan. Several ran very old JDKs (8/11) with obsolete flags such as `-XX:+AggressiveOpts`. A new framework on a current JDK would be compared against these "Platform"-class entries if Meleys is classified as a platform or micro framework.
- **Virtual threads did not top any Round 23 chart** among Java entries. helidon-nima ranked #40 in Fortunes and #86 in JSON.

### Gaps
- I could not obtain Java rankings from post-Round-23 continuous runs (2025–2026), because tfb-status is down. Those would reflect the JDK 24/25 updates.
- Round 23 composite-score ranks for Java are not available (see Q3).
- Exact JVM flags inside `run_quarkus.sh` and `run_benchmark.sh` (Micronaut) were not inspected.

## Q7. What is TFB's protocol and platform scope: HTTP/1.1 only, any TLS, Linux only?

### Takeaway
TFB covers cleartext HTTP/1.1 only: GET requests over keep-alive connections, with HTTP/1.1 pipelining in plaintext, on 4-digit ports. There is no TLS, no HTTP/2, no HTTP/3, and no WebSocket test. In practice it is Linux-only: every Round 23 permutation ran Linux in Docker, although the metadata allows Windows. None of Meleys' HTTP/2, HTTP/3, TLS, macOS or Windows goals can be assessed with TFB.

### Cited Findings
- **Protocol.** All test types use HTTP/1.1 GET example requests. Plaintext requires HTTP/1.1 pipelining ("is assumed"). Requests go to a 4-digit port. The spec mentions no TLS or HTTP/2. — [Wiki spec](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Framework-Tests-Overview)
- **Load generator.** wrk is invoked with plain URLs and `Connection: keep-alive`; plaintext uses Lua-scripted pipelining (depth 16). — [concurrency.sh](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/wrk/concurrency.sh); [pipeline.lua](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/wrk/pipeline.lua)
- **HTTP/2 explicitly switched off by a top entry.** The Round 23 Vert.x entry disables HTTP/2 cleartext and WebSockets via flags (`-Dvertx.disableH2c=true -Dvertx.disableWebsockets=true`). — [R23 vertx.dockerfile](https://github.com/TechEmpower/FrameworkBenchmarks/blob/523534bb61450e3522d775a749ad060753e26e3a/frameworks/Java/vertx/vertx.dockerfile)
- **OS.** Metadata allows `os` to be "Linux or Windows". — [Wiki: Framework Files](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Codebase-Framework-Files). In Round 23 all 841 permutations have `os: linux` (computed). — [R23 JSON](https://www.techempower.com/benchmarks/results/round23/ph.json). Hosts were Ubuntu (22.04 for R22, 24.04 for R23) running Docker. — [MongoDB blog post](https://www.techempower.com/blog/2026/03/16/how-our-benchmarks-led-to-a-25x-mongodb-performance-improvement/); [Wiki: Benchmarking Getting Started](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Benchmarking-Getting-Started)
- **Requests for broader coverage were never met.** Users asked for WebSocket, memory and CPU metrics (discussion of 2024-03-02). — [Discussions](https://github.com/TechEmpower/FrameworkBenchmarks/discussions). HttpArena, proposed as a successor, reportedly covers HTTP/2, WebSockets and gRPC (TLS and h2c). — [DEV post](https://dev.to/kaliumhexacyanoferrat/techempower-framework-benchmarks-are-now-archived-whats-next-3l0a)
- **Brief criticism note** (covered in depth by another researcher). The DEV post argues many listed "frameworks" would fail basic HTTP compliance tests and rely on low-level tricks. HN commenters describe gaming, for example a SQL requirement rewritten as a single CASE update. — [DEV](https://dev.to/kaliumhexacyanoferrat/techempower-framework-benchmarks-are-now-archived-whats-next-3l0a); [HN](https://news.ycombinator.com/item?id=47497763)

### Inferences
- **TFB's methodology could only serve as Meleys' HTTP/1.1 plaintext/JSON baseline harness on Linux.** HTTP/2, HTTP/3, TLS, cross-OS behaviour, tail latency and memory all need other tools, which other researchers are covering.

### Gaps
- None material. Absence of TLS and HTTP/2 is inferred from the spec and scripts, which never mention them, rather than from an explicit "not supported" statement.
