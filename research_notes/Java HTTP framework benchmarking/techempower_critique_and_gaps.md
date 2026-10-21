# TechEmpower Framework Benchmarks (TFB): criticisms, gaming, how teams view it, and what it cannot measure for Meleys

Research date 2026-10-21. Every claim is dated. "Final commit" means TFB `master` at 57d92fb (2026-03-23), the last commit before the repository was archived. I read it through a sparse clone of the toolset, the entry configs and selected entries, plus a clone of the TFB wiki repository, which gave the dates of rule changes from its history. Another researcher covers TFB mechanics, status and rankings, so those appear here only where the critique needs them.

## 1. Main published criticisms of TFB (2018–2026)

### Takeaway
The criticisms from 2018–2026 agree with each other. TFB measures peak HTTP/1.1 keep-alive throughput for tiny GET responses, with plaintext sent 16-deep pipelined. It runs on one Linux/Docker lab and uses closed-loop `wrk` in 15-second runs. As a result, the top ranks hit network, load-generator and database ceilings, and they reward hand-tuned "benchmark code". The composite is a subjectively weighted sum of throughputs. The latency figures (avg/stdev/max at saturation) are not evidence of tail latency.

### Cited Findings

#### Unrealistic workloads
- The JSON test must return `{"message":"Hello, World!"}` (about 28 bytes) and the plaintext body is `Hello, World!`. Every test is "exercised using GET requests", and gzip "is not permitted". — [TFB wiki: Framework Tests Overview](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Framework-Tests-Overview)
- Round 16 (2018-06-06): frameworks allowed to pipeline "can completely saturate ten gigabit per second with ~140-byte response payloads". — [TechEmpower Round 16 blog](https://www.techempower.com/blog/2018/06/06/framework-benchmarks-round-16/)
- Litestar maintainer provinzkraut (2023-05-05): "The TechEmpower benchmarks are useless, because they don't impose any constraints on how the applications" [and environments are set up]. He says what you see "is mostly a difference in performance of servers, not frameworks", that the tests are "very synthetic", and that the gap between 10-byte and 100 kB responses "is something TechEmpower does not test for". — [litestar-org/litestar discussion #1620](https://github.com/litestar-org/litestar/discussions/1620)
- HN, 2020-08-29: the database tests cover only "_simple_ database requests". The commenter proposed a "chaos" test against slow backends. — [HN: Lithium ranked highest](https://news.ycombinator.com/item?id=24315285)
- HN, 2026-03-24: jerf argues that for most teams, measuring framework overhead is "quintessential premature optimization". Teams that really are performance-bound "have to stub out your performance and run benchmarks yourself". — [HN: Sunsetting TFB](https://news.ycombinator.com/item?id=47497763)
- TechEmpower's own FAQ says "nothing beats conducting performance tests yourself for the specific workload of your application". — [TFB wiki: Expected Questions #24](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Expected-Questions)

#### HTTP pipelining in the plaintext test
- The rules say "Requests will be sent using HTTP pipelining" and "Server support for HTTP/1.1 pipelining is assumed", at 256, 1,024, 4,096 and 16,384 connections. — [TFB wiki: Framework Tests Overview](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Framework-Tests-Overview)
- The toolset's `pipeline.lua` concatenates `depth` copies of the request into one write, and the plaintext test passes `'pipeline': 16`. — [toolset/wrk/pipeline.lua](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/wrk/pipeline.lua); [toolset/test_types/plaintext/plaintext.py](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/test_types/plaintext/plaintext.py)
- Outdated wording: the plaintext description still says the test can "saturate the gigabit Ethernet of the test environment". The lab has used 10 GbE since 2018 and 40 GbE since 2025 (see below). — [TFB wiki: Framework Tests Overview](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Framework-Tests-Overview)
- Example of a framework feature aimed at pipelining: the TFB `hyper` entry calls `http.pipeline_flush(true)` (main.rs line 164). hyper documents that option as "Aggregates flushes to better support pipelined responses. Experimental, may have bugs." — [TFB hyper main.rs](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/frameworks/Rust/hyper/src/main.rs); [docs.rs hyper http1::Builder](https://docs.rs/hyper/latest/hyper/server/conn/http1/struct.Builder.html#method.pipeline_flush)
- Andreas Nägeli (GenHTTP maintainer, 2026-03-24) says the maintainers "allowed frameworks to use low-level tricks", so results "reflect more a socket benchmark", and top frameworks were "capped in plaintext at roughly 30 milion requests per second" because of wrk and large request headers. — [DEV: TFB are now archived – what's next?](https://dev.to/kaliumhexacyanoferrat/techempower-framework-benchmarks-are-now-archived-whats-next-3l0a)

#### "Benchmark-special" and stripped implementations
- Whether an entry counts as "Realistic" or "Stripped" is self-declared metadata in each entry's config. — [TFB wiki: Codebase Framework Files](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Codebase-Framework-Files)
- At the final commit, only 29 of 965 test configurations (3%) are declared "Stripped" and 936 are "Realistic". By classification: Platform 233, Micro 431, Fullstack 296, unset 5. These counts come from my analysis of every `frameworks/*/*/benchmark_config.json`. — [TFB frameworks tree](https://github.com/TechEmpower/FrameworkBenchmarks/tree/master/frameworks)
- HN, 2021-02-09: Volundr was "annoyed that they are calling that a 'realistic' approach rather than stripped" about just-js, which is still `Platform/Realistic` at the final commit. — [HN: Round 20](https://news.ycombinator.com/item?id=26068204); [just benchmark_config.json](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/frameworks/JavaScript/just/benchmark_config.json)
- HN, 2021-02-08: "Most of these benchmarks are insanely optimized. The code is rarely idiomatic. There's also a few frameworks in there that just wraps C libraries to get good scores." — [HN: Round 20](https://news.ycombinator.com/item?id=26068204)
- HN, 2026-03-24: idoubtit, who contributed to TFB implementations, "lost most of the trust I had in these benchmarks… Some frameworks were just not production ready, or had shortcuts made just for a benchmark case". — [HN: Sunsetting TFB](https://news.ycombinator.com/item?id=47497763)
- HN, 2018-06-07: one commenter had "seen in some of the implementations that I care about a bit of benchmark gaming". — [HN: Round 16](https://news.ycombinator.com/item?id=17254152)

#### Network ceiling, load generator and database as the bottleneck
- Round 16 (2018) said Plaintext and JSON results "were converging on theoretical maximums for 10-gigabit networking". The Citrine lab was "three homogeneous Dell R440 servers, each equipped with a Xeon Gold 5120 CPU" on a 10-gigabit switch. — [Round 16 blog](https://www.techempower.com/blog/2018/06/06/framework-benchmarks-round-16/); [TFB wiki: Environment](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Environment)
- just-js author, c. Oct 2020: top plaintext entries "all score around the same… because the 10Gbps network is saturated at 7m requests per second". On the database tests, many frameworks were "not maxing out on CPU", probably because the database was maxed out, "but again it is not possible to determine this as we don't get that data from the techempower datasets". He also notes that TFB does not provide numbers "for the client or the Postgres server". — [just-js blog: On JavaScript performance 01](https://just.billywhizz.io/blog/on-javascript-performance-01/)
- TFB FAQ #25 says that on physical hardware "the limiting factor for the JSON test is the client load-generating machine, which is less powerful than the machine running the app server". This is likely outdated: it predates the homogeneous Citrine machines (2018) and the 2025 hardware. — [TFB wiki: Expected Questions](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Expected-Questions)
- Round 23 (2025-03-17) ran on new hardware provided by Microsoft: ProLiant DL360 Gen10 Plus servers, Xeon Gold 6330 (56 cores), 64 GB of memory and ConnectX-6 40 Gbps Ethernet. TechEmpower reported "3x Improvements in Practical Network-Bound Tests" and "4x Improvements in Theoretical Network-Bound Tests". The ceiling moved but did not go away; see the "~30M rps" claim above. — [Round 23 blog](https://www.techempower.com/blog/2025/03/17/framework-benchmarks-round-23/)
- Database: Round 17 (2018-10-30) found Postgres results "stratified", with "groups of test implementations that seem implausibly faster than other test implementations". The effect was "most emphatic on the Multi-query and Updates tests" and came from driver-level "query pipelining". — [Round 17 blog](https://www.techempower.com/blog/2018/10/30/framework-benchmarks-round-17/)
- HN, 2021-02-08: "the biggest beneficiary of these benchmarks is Postgres. Its drivers are often the bottleneck". — [HN: Round 20](https://news.ycombinator.com/item?id=26068204)

#### Run-to-run variance and statistical method
- Each measurement is one `wrk` run of 15 s (default `--duration 15`). Concurrency levels are 16–512 for JSON and DB tests and 256–16,384 for plaintext, with query levels 1/5/10/15/20. Each sequence starts with a 5-s, 8-connection primer and one warm-up run at maximum concurrency. — [toolset/run-tests.py](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/run-tests.py); [toolset/wrk/pipeline.sh](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/wrk/pipeline.sh)
- The results site reports the peak level. For example, "the peak plaintext performance was at 256 concurrency", and errors are counted "during the peak test". — [TFB wiki: Results Website](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Results-Website)
- Round 16 (2018) said that moving to Docker meant "we now see much lower variability between each run of the full suite". — [Round 16 blog](https://www.techempower.com/blog/2018/06/06/framework-benchmarks-round-16/)
- Feedback is slow. A full run took "approximately 111 hours (4.6 days)" for 2,625 tests in 2020. — [Round 19 blog](https://www.techempower.com/blog/2020/05/28/framework-benchmarks-round-19/)
- Anton Kirilov (2023-05-22) tested a libpq patch on TFB's continuous lab. He saw "standard errors were 0.65% or less" and noted "it takes roughly a week to get a new set of results". The PostgreSQL reviewer asked "what would the range of the usual noise" be, and the thread excerpt does not answer. — [pgsql-hackers: Add PQsendSyncMessage() to libpq](https://www.postgresql.org/message-id/fdbf4c41-37ea-9da5-46c5-4c167ac79e8a%40gmail.com)

#### A single hardware environment
- Official rankings come from TechEmpower's own lab: Citrine physical, plus Azure D3v2 cloud in earlier rounds. — [Round 19 blog](https://www.techempower.com/blog/2020/05/28/framework-benchmarks-round-19/); [TFB wiki: Environment](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Environment)
- Rankings depend on the environment. FAQ #25 exists because the JSON and Plaintext "leaderboards… [are] so different on Cloud versus Physical". In 2018 TechEmpower declined to publish a diff between Round 15 and Round 16 "because there are simply too many variables—most importantly this new hardware and Docker". — [TFB wiki: Expected Questions](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Expected-Questions); [Round 16 blog](https://www.techempower.com/blog/2018/06/06/framework-benchmarks-round-16/)
- The lab can produce pathologies specific to itself. In Round 23, spring-mongo collapsed on Citrine. The fixes were a workaround for TCMalloc per-CPU memory pools inside Docker (PR #10514) and connection-pool settings (PR #10565). — [TechEmpower blog 2026-03-16](https://www.techempower.com/blog/2026/03/16/how-our-benchmarks-led-to-a-25x-mongodb-performance-improvement/)

#### What the composite score means
- Composite scores were introduced in Round 19 (2020-05-28). TechEmpower "normalize[s] results for each test type and then apply subjective weights for each", for example weighting Fortunes above Plaintext. Only frameworks that implement every test type get a composite. — [Round 19 blog](https://www.techempower.com/blog/2020/05/28/framework-benchmarks-round-19/)
- The tentative per-test "bias coefficients" are json 1, single query 0.75, 20-query 0.75, fortunes 1.5, updates 1.25, plaintext 0.75. "Each Round will compute new weights for each test type", which are applied and "then summed". — [TFB wiki: TPR](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/TechEmpower-Performance-Rating-(TPR)); [TFB wiki: Composite Scoring](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Composite-Scoring-for-Frameworks)

#### Latency from closed-loop wrk at high concurrency (coordinated omission)
- TFB drives every test with `wrk` at fixed connection counts (`--timeout 8`, threads = `nproc`). wrk is closed-loop: each connection sends its next request only after a response. — [toolset/wrk/pipeline.sh](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/wrk/pipeline.sh)
- TFB's parser keeps only `latencyAvg`, `latencyStdev` and `latencyMax` from wrk's "Latency" line. It stores no percentiles. — [toolset/utils/results.py](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/utils/results.py)
- Before printing latency stats, wrk applies a post-run heuristic `stats_correct(statistics.latency, interval)`. — [wg/wrk src/wrk.c lines 169–176](https://github.com/wg/wrk/blob/master/src/wrk.c)
- API7 (page dated 2018-09-03, but it describes the current wrk source) says that correction "acknowledges coordinated omission, but it is not the same as running a workload with an explicit arrival-rate plan. The expected interval is inferred after the run, while the test itself remains connection-driven." — [API7: wrk vs wrk2](https://api7.ai/blog/analysis-of-inaccurate-distribution-of-wrk-latency)
- Gil Tene's wrk2 README (written against wrk's Nov 2014 model) says wrk's send-to-completion latency model "exhibits a strong Coordinated Omission effect". In his example, the corrected tool reports a 99th percentile "200x (!!!) larger" than the uncorrected one. — [giltene/wrk2 README](https://github.com/giltene/wrk2)
- HN, 2018-06-07: "latency is more important to most users than the peak throughput… production applications almost never hit peak throughput". — [HN: Round 16](https://news.ycombinator.com/item?id=17254152)

### Inferences
- TFB measures at or near saturation with a closed-loop generator, so its average latency is mostly the queueing implied by Little's law. Illustration (my arithmetic): 16,384 connections × 16 pipelined requests is about 262k requests in flight, and at 7M rps that gives roughly 37 ms average latency, whatever the server's service time. TFB latency therefore says nothing about Meleys's p99/p99.9 goal at a realistic offered load.
- The top of plaintext and JSON is capped by the network or the load generator: 10 GbE through Round 22, then 40 GbE and wrk/header limits in Round 23. Order inside the top cluster is therefore close to noise and cannot support a "fastest" claim.
- Four of the six composite tests depend on the database, with Fortunes weighted 1.5 and Updates 1.25. So for a pure HTTP framework the composite largely measures the PostgreSQL driver, the pool and the pipelining strategy, not the HTTP stack.
- Background knowledge, not verified in this session: mainstream browsers do not use HTTP/1.1 pipelining, and HTTP/2 multiplexing replaced it. The plaintext test therefore rewards a code path (batched parse and flush of 16 queued requests) that real clients rarely use.

### Gaps
- I found no TechEmpower documentation of repeat counts, confidence intervals or measured noise ranges. The variance evidence is anecdotal: Round 16's claim and one pgsql-hackers data point.
- The 2024 article "How can Rust be so fast in the TechEmpower Web Framework Benchmarks?" (kerkour.com), reposted on HN as late as 2026-07-06, now returns 404, so I could not use it.

## 2. Known gaming / benchmark-special techniques, and how TechEmpower responded

### Takeaway
Documented techniques include:
- PostgreSQL pipelining and batching, including Sync-less batching later ruled unsafe
- bulk updates, which are explicitly allowed
- pre-built headers and once-per-second Date strings
- hand-built Fortunes HTML
- custom minimal HTTP parsers and database drivers
- allocator and syscall tuning
- dropping production behaviour such as reconnecting to the database

TechEmpower responded with rule text (2019), response validation (Round 9 onward), query-count verification (Round 19, 2020), a Sync-per-query rule (March 2022, enforced for Round 21) and a Realistic/Stripped filter. Enforcement relied on self-labelling, and in 2024 TechEmpower explicitly preferred documenting gaps to forbidding them.

### Cited Findings

#### PostgreSQL query pipelining and batching (Vert.x, Rust, C/C++, just-js, Lithium)
- Round 17 (2018-10-30): TechEmpower identified "query pipelining" in the Postgres Extended Query protocol, noted that "the client application's business logic is unchanged", and "ultimately arrived at permitting this optimization". — [Round 17 blog](https://www.techempower.com/blog/2018/10/30/framework-benchmarks-round-17/)
- Vert.x's TFB entry sets `options.setPipeliningLimit(256); // Large pipelining means less flushing and we use a single connection anyway`. — [TFB vertx App.java line 170](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/frameworks/Java/vertx/src/main/java/vertx/App.java)
- HN, 2021-02-09: "the top C/C++ benchmarks are still using an unofficial fork of libpq client that supports batching. Rust and Java was on the top of the benchmark for a while just because they their own pipeline supported drivers." — [HN: Round 20](https://news.ycombinator.com/item?id=26068204)
- just-js wrote its own JavaScript Postgres driver with "pre-compile[d] the queries into ArrayBuffers" and noticed that other entries used "a patchset for libp[q]". — [just-js blog](https://just.billywhizz.io/blog/on-javascript-performance-01/)
- Rule history from the wiki repository:
  - 2019-08-22: General Requirement #7 added, permitting pipelining "as a network-level optimization" when queries still "execute separately on the database server".
  - 2022-03-22 (commit 33e74ec, Michael Hixson): added "If using PostgreSQL's extended query protocol, each query must be separated by a Sync message", and banned any optimization that "could change the transactionality and/or error handling".
  — [TFB wiki: Framework Tests Overview](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Framework-Tests-Overview); [wiki page history](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Framework-Tests-Overview/_history)
- Lithium's maintainer (2022-06-24) said pipelining was "forbidden after the TFB team stated this as an unsafe optimization, even for read only usecases", and that frameworks were asked to remove it for Round 21 (citing TFB #7289). Lithium's Fortunes throughput fell from 670,000/s to 380,000/s, about 43%. — [matt-42/lithium discussion #116](https://github.com/matt-42/lithium/discussions/116)
- In #8790 (2024-03-08), contributor volyrique said the frameworks that had to be fixed had "violated transactionality, specifically by skipping Sync messages". billywhizz (just-js) replied that nothing verifies a Sync is sent for every query. — [TFB issue #8790](https://github.com/TechEmpower/FrameworkBenchmarks/issues/8790)

#### Bulk / batched updates
- Rule, added 2019-05-14: "Using bulk updates—batches of update statements or individual update statements that affect multiple rows—is acceptable but not required"; bulk reads are not. — [TFB wiki: Framework Tests Overview](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Framework-Tests-Overview); [wiki history](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Framework-Tests-Overview/_history)
- The verifier detects bulk updates, where the query count is close to the expected count, and allows a 5% margin on updated rows. — [toolset/test_types/verifications.py](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/test_types/verifications.py)
- HN, 2026-03-24: an implementation "converted the 'multiple sql updates' requirements into a single complex update using CASE". This is legal under the rule above. In 2020, another commenter saw "presorting and batching inserts". — [HN: Sunsetting TFB](https://news.ycombinator.com/item?id=47497763); [HN: Lithium thread](https://news.ycombinator.com/item?id=24315285)

#### Pre-computed or cached responses, headers and Date
- Rules:
  - The JSON serialization "must not be cached" (added 2019-05-14).
  - Plaintext may reuse the body buffer, but "It is not acceptable to store the entire payload of the response, headers inclusive, as a pre-rendered buffer" (2019-03-06).
  - The Date header may be re-rendered "once per second", which is "an acceptable tactical optimization" (2019-08-22).
  - A `Server` header is mandatory because it "roughly normalizes network load".
  — [TFB wiki: Framework Tests Overview](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Framework-Tests-Overview); [wiki history](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Framework-Tests-Overview/_history)
- Dusted Codes (2022-11-14) analysed ASP.NET Core's Round 21 "Platform" entry `aspcore-ado-pg`. It uses:
  - custom `BenchmarkApplication`/`HttpApplication` types, with Kestrel the only recognisable ASP.NET Core part
  - full HTTP headers pre-built as byte arrays (`AsciiString`)
  - no `HttpContext` or standard routing
  - no real HTML template for Fortunes
  - a static `DateHeader` refreshed by a timer

  The article also cites a 2019 ruling that "concatenating hardcoded strings is cheating". — [Dusted Codes: How fast is ASP.NET Core?](https://dusted.codes/how-fast-is-really-aspnet-core)
- just-js author: "the html is being built manually which is again a little bit of a cheat". On HTML-escaping at the driver level: "I admit escaping the string at the database driver level is a little bit of a cheat". — [just-js blog](https://just.billywhizz.io/blog/on-javascript-performance-01/)
- HN, 2026-03-24: "It was definitely possible to catch not having sent date headers (or caching them) etc." and "There were suggestions but it didn't happen." — [HN: Sunsetting TFB](https://news.ycombinator.com/item?id=47497763)

#### Minimal HTTP stacks and platform-level entries
- Rule #2: implementations "not based on a realistic HTTP implementation will be marked as Stripped". Instead of an HTTP specification, the rule defers "to common sense". — [TFB wiki: Framework Tests Overview](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Framework-Tests-Overview)
- Entries labelled Stripped at the final commit include C# `aspnetcore` default/aot/mysql (Platform), Go `gnet` and `goev`, Rust `actix` "server" variant, Rust `faf`, `ntex` "plt" variants and `water-http`, Python `wsgi`/`uwsgi`/`mrhttp`, and Java `aio-socket`, the only Stripped Java entry. Minimal stacks such as Rust `may-minihttp`, `xitca-web` "barebone", JavaScript `just` and C++ `lithium` are labelled Realistic. Source: my config analysis. — [TFB frameworks tree](https://github.com/TechEmpower/FrameworkBenchmarks/tree/master/frameworks); [aspnetcore benchmark_config.json](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/frameworks/CSharp/aspnetcore/benchmark_config.json)
- Of 68 Java `default` entries, 20 are classified "Platform" and all 20 are "Realistic", for example netty, undertow, vertx, jetty, httpserver, microhttp and rapidoid. — [TFB frameworks/Java](https://github.com/TechEmpower/FrameworkBenchmarks/tree/master/frameworks/Java)
- just-js used "picohttpparser which takes advantage of SSE4 instructions". — [just-js blog](https://just.billywhizz.io/blog/on-javascript-performance-01/)
- Nägeli (2026): many listed "frameworks" "would probably even fail basic compliance tests". — [DEV](https://dev.to/kaliumhexacyanoferrat/techempower-framework-benchmarks-are-now-archived-whats-next-3l0a)

#### Single-purpose frameworks built for TFB
- just-js (2020–21) was a V8 runtime side project tuned for TFB. Its author describes a custom DB driver, picohttpparser and the hand-built HTML noted above, and reports placing 5th at first. HN titles in 2022 call it "now #1". — [just-js blog](https://just.billywhizz.io/blog/on-javascript-performance-01/); [HN listing (Algolia)](https://news.ycombinator.com/item?id=32923084)
- Lithium (C++, 2020): the HN post "Lithium ranked the highest performance web application platform" got 99 points and 127 comments on 2020-08-29. — [HN](https://news.ycombinator.com/item?id=24315285)
- httpbeast (Nim): its author "had so much fun implementing a http server (called httpbeast) from scratch to get as far up these benchmarks as possible". — [HN: Sunsetting TFB](https://news.ycombinator.com/item?id=47497763)

#### Low-level tuning for the lab
- Round 22 (2023-11-15) contributor notes:
  - mORMot "reduced the syscalls as much as possible… (a /plaintext request requires no memory allocation)".
  - A Rust contributor recommends that entries "Specify memory allocators: mimalloc, snmalloc-rs, jemallocator".
  — [Round 22 blog](https://www.techempower.com/blog/2023/11/15/framework-benchmarks-round-22/)
- TFB passes `BENCHMARK_ENV` (Citrine or Azure) into containers, documented as "Useful for tuning your application". — [TFB wiki: Codebase Framework Files](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Codebase-Framework-Files)
- HN, 2021-02-08: "truly high performance requires tuning your code to linux, to the network interface cards". — [HN: Round 20](https://news.ycombinator.com/item?id=26068204)

#### Dropping production robustness (2024)
- Issue #8790, "Most of the best-performing frameworks don't survive temporary db connectivity loss" (itrofimow, 2024-03-05), tested the top 20 entries by restarting the database mid-run:
  - 8 recovered: xitca-web after a fix, h2o, redkale, drogon, mormot, wizzardo-http, aspnetcore, lithium.
  - 6 hung: may-minihttp, just, officefloor, jooby, ntex, inverno.
  - 5 stayed stuck returning 500/502: vertx, viz, axum, actix, vertx-web.
  - salvo crashed.

  The author says Axum fell from about 550K to about 90K rps in a configuration that recovers. — [TFB issue #8790](https://github.com/TechEmpower/FrameworkBenchmarks/issues/8790)

#### The actix-web unsafe-code episode (2018–2020)
- Steve Klabnik (January 2020) wrote that actix-web's "score on Techempower in particular was impressive, and got more impressive over time". He added that "a lot of that unsafe code is not actually needed", that patches "weren't welcome", and that "actix-web kept climbing the Techempower benchmarks, reaching the top of most of them by a lot". Then "the maintainer has quit". — [Steve Klabnik: A sad day for Rust](https://steveklabnik.com/writing/a-sad-day-for-rust)
- HN, 2020-05-29: criticism of "'cheating' and unnatural tricks… [is] more a fundamental criticism about this kind of benchmark games than a specific criticism of Actix". — [HN: Round 19](https://news.ycombinator.com/item?id=23342052)

#### TechEmpower's responses
- Round 9 onward: validation checks; non-conforming entries show "Did not complete". — [TFB wiki: Expected Questions #27](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Expected-Questions)
- 2019 rules overhaul, dated from the wiki history:
  - "should be production-grade", plus bans on in-memory caching of World and Fortune rows: 2019-03-06
  - JSON must not be cached; bulk updates allowed: 2019-05-14
  - the Stripped marking, the database-pipelining permission and the once-per-second Date rule: 2019-08-22

  Caveat: the 2019-03-06 commit is titled "wiki work" and looks like the migration of the docs into the wiki. Rules that first appear on that date may be older. — [wiki history](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Framework-Tests-Overview/_history)
- Round 19 (2020-05-28) added "Additional database test verifications to ensure implementations are valid". An HN commenter noted that the new "realistic" vs "synthetic" filter used "realistic" as the default. — [Round 19 blog](https://www.techempower.com/blog/2020/05/28/framework-benchmarks-round-19/); [HN: Round 19](https://news.ycombinator.com/item?id=23342052)
- `verify_queries_count` reads the database's statistics. On official runs it accepts "No margin… on the number of queries", 1% on rows read, and 5% on bulk-updated rows. — [toolset/test_types/verifications.py](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/test_types/verifications.py)
- 2022: the Sync-per-query rule and its Round 21 enforcement, covered above. — [Lithium #116](https://github.com/matt-42/lithium/discussions/116)
- 2024 (#8790): NateBrady23 (TechEmpower) agreed that production-grade frameworks should recover but said this "probably wouldn't alter results". bhauer (TechEmpower) said rule 1 is intentionally vague and that he prefers to document risky optimizations rather than forbid them. TechEmpower closed the issue without a rule change. — [TFB issue #8790](https://github.com/TechEmpower/FrameworkBenchmarks/issues/8790)
- 2018: bhauer said "we reserve the right to reject code that is too experimental". — [HN: Round 16](https://news.ycombinator.com/item?id=17254152)

### Inferences
- The rules were enforced only where the verifier could observe something: response shape, required headers, and DB query and row counts. HTTP compliance, Sync per query, Date and header caching, and resilience went unchecked. Since "Realistic" is self-declared, it does not mean "idiomatic". It only means nobody successfully challenged the label.
- A plain, idiomatic Meleys entry would compete against Platform entries, custom drivers and allocator- and syscall-tuned stacks. Its rank would partly reflect things that have nothing to do with the quality of its HTTP/1.1/2/3 stack. If Meleys ever submits TFB-style code, it needs an explicit policy against benchmark-special code, published alongside the results.

### Gaps
- I did not read TFB issue #7289 (the 2022 pipelining ruling) or the 2019 ASP.NET "hardcoded strings" issue directly. GitHub's API and plain curl were blocked for that repository from this session, and I did not fetch these two pages within the budget. Both are cited through secondary sources: Lithium #116 and Dusted Codes. The issue number of the 2019 ruling is still unknown.
- I found no evidence that actix-web's unnecessary `unsafe` was written specifically to win TFB. The documented link is only that the controversy hit the top-ranked TFB framework.

## 3. How mainstream framework teams talk about TFB

### Takeaway
Teams value TFB for visibility, as a free shared multi-machine lab, and as a way to find real performance bugs. Microsoft is the heaviest user and also sponsored TFB hardware. Other cases are the Rust frameworks, Postgres driver authors, and MongoDB with Spring in 2026. Many maintainers, including some of those same teams, warn that entries are unidiomatic or tuned and that rankings mislead. I found no official statements from the Spring, Netty, Fastify or Red Hat teams.

### Cited Findings

#### Microsoft / ASP.NET Core
- Microsoft sponsored the hardware for Round 23: "our sponsor Microsoft has generously provided new hardware". An HN commenter in 2018 also said Citrine was sponsored by Microsoft. — [Round 23 blog](https://www.techempower.com/blog/2025/03/17/framework-benchmarks-round-23/); [HN: Round 16](https://news.ycombinator.com/item?id=17254152)
- The ASP.NET team's own repository holds "Benchmarks for ASP.NET Core including (but not limited to) scenarios from the TechEmpower Web Framework Benchmarks". The team tracks results at aka.ms/aspnet/benchmarks and runs them with Microsoft Crank. — [aspnet/Benchmarks README](https://github.com/aspnet/Benchmarks)
- Marketing, as quoted by Dusted Codes (2022-11-14):
  - A BUILD 2021 slide and Scott Hunter claimed .NET is "more than 10 times faster" than Node.js.
  - Sébastien Ros: "even with the fastest Go web framework, .NET is still faster when using a high level stack".
  - David Fowler said the team would be "more mindful" (the article's paraphrase).

  The article's verdict is that ASP.NET Core is fast but "evidently not faster than Java, Go or C++". It calls labelling the Platform entry "ASP.NET Core" "slightly disingenuous", and notes the MVC entry ranked 109th at 184k rps in Round 21 Fortunes. — [Dusted Codes](https://dusted.codes/how-fast-is-really-aspnet-core)
- At the final commit, `aspnetcore` default/aot/mysql are `Platform/Stripped`, while `minimal` is `Micro/Realistic` and `mvc` is `Fullstack/Realistic`. — [aspnetcore benchmark_config.json](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/frameworks/CSharp/aspnetcore/benchmark_config.json)
- HN, 2026-03-24: TFB lets teams "easily justify the use of out of the box ASP.NET Core… Are the benchmarks gamed? Yes of course". Another commenter notes that in Round 23 "aspnetcore" fell to 35th on Fortunes and its code "really just uses kestrel". — [HN: Sunsetting TFB](https://news.ycombinator.com/item?id=47497763)

#### Vert.x / Red Hat
- The Vert.x entries depend on reactive-client DB pipelining (`setPipeliningLimit(256)`, see above). In the 2024 resilience test, `vertx` and `vertx-web` stayed stuck returning 500/502 after a database restart. — [TFB vertx App.java](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/frameworks/Java/vertx/src/main/java/vertx/App.java); [issue #8790](https://github.com/TechEmpower/FrameworkBenchmarks/issues/8790)
- Quarkus, Helidon (including `nima`), Micronaut, Jooby and Spring entries are all labelled Realistic. — [TFB frameworks/Java](https://github.com/TechEmpower/FrameworkBenchmarks/tree/master/frameworks/Java)

#### Spring
- In Round 23, `spring-mongo` "had fallen off a cliff". MongoDB's lead performance engineer, Ger Hartnett, "treated the weird benchmark as a bug report". After PRs #10514 (the TCMalloc-in-Docker workaround) and #10565 (pool size), the query test went "from 583 to 14.2k", about 25×. — [TechEmpower blog 2026-03-16](https://www.techempower.com/blog/2026/03/16/how-our-benchmarks-led-to-a-25x-mongodb-performance-improvement/)
- HN, 2018-06-07: "always shocked that Spring does so poorly (it's 7% here). I haven't had any performance issues with Spring in production". — [HN: Round 16](https://news.ycombinator.com/item?id=17254152)

#### Netty
- The Netty entry is `Platform/Realistic`. An HN commenter (2020) notes that "Netty is network IO framework and is almost never used as is to develop web applications". — [TFB frameworks/Java](https://github.com/TechEmpower/FrameworkBenchmarks/tree/master/frameworks/Java); [HN: Round 19](https://news.ycombinator.com/item?id=23342052)

#### Go (fasthttp)
- The fasthttp README links "TechEmpower Benchmark round 23 results" but warns: "fasthttp might not be for you! … For most cases `net/http` is much better… For most cases you won't even notice the performance difference". HTTP/2 support is still "in progress". — [valyala/fasthttp README](https://github.com/valyala/fasthttp)

#### Rust (axum, actix/ntex, xitca-web, hyper)
- jplatte (axum maintainer) in #8790: not recovering from database outages is "pretty weird", and he would not mind removing axum's `pg` implementations. fafhrd91 (actix/ntex author): recovery is easy to add, and he doubted it would change results measurably. fakeshadow (xitca-web): switching to a pool cost an estimated 1–2%, and "most top Rust frameworks would see larger regressions". — [TFB issue #8790](https://github.com/TechEmpower/FrameworkBenchmarks/issues/8790)

#### Node (Fastify)
- I found no statement from the Fastify maintainers.

#### Others who talk about TFB's value
- just-js author (2020): TFB is "a very useful tool… highlighting weaknesses and driving efforts in many of these frameworks to improve performance… all such competitions should be taken with a large grain of salt". — [just-js blog](https://just.billywhizz.io/blog/on-javascript-performance-01/)
- GenHTTP maintainer (2026): TFB "did an amazing job in making performance of web frameworks visible", but its results are "highly missleading". — [DEV](https://dev.to/kaliumhexacyanoferrat/techempower-framework-benchmarks-are-now-archived-whats-next-3l0a)
- Litestar maintainer (2023): "people really ought to stop referring to the TechEmpower benchmarks and frameworks should stop participating". — [litestar #1620](https://github.com/litestar-org/litestar/discussions/1620)
- HN, 2021-02-08: "the biggest benefit of these benchmarks has been to trigger improvements to all the low-hanging fruit in various frameworks". — [HN: Round 20](https://news.ycombinator.com/item?id=26068204)
- PostgreSQL contributors used TFB's continuous lab to evaluate libpq changes, measuring +8.94% on single query and +9.54% on Fortunes in 2023. — [pgsql-hackers](https://www.postgresql.org/message-id/fdbf4c41-37ea-9da5-46c5-4c167ac79e8a%40gmail.com)
- TechEmpower's own framing: TFB measures "the high-water mark performance" (Round 19), and "performance data should be one part of your decision-making process" (FAQ). — [Round 19 blog](https://www.techempower.com/blog/2020/05/28/framework-benchmarks-round-19/); [Expected Questions #24](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Expected-Questions)

### Inferences
- The concrete wins in the record (Spring/Mongo in 2026, Postgres driver work, low-hanging fruit) came from large outliers that TFB made visible, not from fine-grained rank ordering. For Meleys, that value can be reproduced in-house with a regression suite.
- In Java, the flagship frameworks (Spring, Quarkus, Micronaut, Helidon) run idiomatic "Realistic" entries and rarely headline TFB. The Java names at the top (vertx, platform-level servers) lean on DB pipelining or bare-bones stacks. A TFB rank therefore tends to persuade a performance-enthusiast audience rather than the enterprise Java decision-makers Meleys may need.

### Gaps
- Within the search budget I found no official statements from Spring, Netty, Fastify or Red Hat (Vert.x or Quarkus) maintainers about TFB.
- A search summary said Microsoft's dotnet.microsoft.com ASP.NET pages cite Round 23 numbers (minimal APIs at 2.05M JSON rps against Node.js 224K and Java Servlet 328K). I did not fetch the page directly, so this is unverified.
- A search snippet attributed to the vertx-pg-client README a claim of "#1 in TechEmpower Round 15 single query". The README fetch failed, so this is unverified.

## 4. Coverage gaps against Meleys's goals

### Takeaway
TFB covers only part of one Meleys goal: peak HTTP/1.1 keep-alive throughput (including pipelining) for tiny GET responses on Linux. It does not test HTTP/2, HTTP/3 or TLS. Its latency is not trustworthy for tails: closed-loop, avg/stdev/max only, measured at saturation. It does not report memory or CPU per request. It runs no macOS or Windows targets. It has no tests for connection churn, large or streamed bodies, uploads, slow clients, large routing tables, middleware, startup time or fault recovery.

### Cited Findings
- **Protocols and TLS:** the toolset builds plain `http://` URLs (`framework_test.py`) and drives load with Ubuntu 24.04's packaged `wrk`. A grep of the final toolset finds no h2load, TLS or QUIC configuration (my check). — [toolset/benchmark/framework_test.py](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/benchmark/framework_test.py); [toolset/wrk/wrk.dockerfile](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/wrk/wrk.dockerfile)
- **No HTTP/2 needed to rank:** fasthttp lacks HTTP/2 ("in progress") yet showcases its TFB Round 23 plaintext result. — [fasthttp README](https://github.com/valyala/fasthttp)
- **Evidence the gap is fillable:** HttpArena (2026, "Alpha Round") separates out HTTP/2, HTTP/3, gRPC, WebSocket and Gateway families. Its HTTP/1.1 profiles include TLS, compression, uploads, static files and mixed endpoints, and it reports memory efficiency. — [HttpArena](https://www.http-arena.com/)
- **Latency:**
  - TFB stores only avg, stdev and max from closed-loop wrk. — [toolset/utils/results.py](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/utils/results.py)
  - wrk's model "exhibits a strong Coordinated Omission effect". — [wrk2 README](https://github.com/giltene/wrk2)
  - Each measurement is 15 seconds. — [run-tests.py](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/run-tests.py)
- **Memory and CPU:**
  - TFB runs system-wide `dool -Tafilmprs …` during tests and summarises only "Average CPU, Average Memory, Total network use, Total disk use". — [toolset/benchmark/benchmarker.py](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/benchmark/benchmarker.py); [toolset/utils/results.py](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/utils/results.py)
  - bhauer (TechEmpower, 2018): "We do in fact capture dstat data while executing the tests but as of today do not render these in any way". — [HN: Round 16](https://news.ycombinator.com/item?id=17254152)
  - A third party built "tfbvis" (2020) to visualise that dstat data for continuous runs from 2020 to 2023-09. — [tfbvis](https://ajdust.github.io/tfbvis/)
  - A user reports asking "for inclusion of memory usage stats… and start-up time stats… But nothing ever happened" (2021). In 2026 the same user said TFB tested "all the metrics the JVM is good at, and non the JVM is bad at (mainly: memory usage, start-up time, container size)". — [HN: Round 20](https://news.ycombinator.com/item?id=26068204); [HN: Sunsetting TFB](https://news.ycombinator.com/item?id=47497763)
  - TFB provides no numbers for the client or the database server, so you cannot tell what saturated. — [just-js blog](https://just.billywhizz.io/blog/on-javascript-performance-01/)
- **Operating systems:**
  - Since Round 16 (2018), "all test implementations are required to be Dockered". — [Round 16 blog](https://www.techempower.com/blog/2018/06/06/framework-benchmarks-round-16/)
  - Citrine moved "from CentOS to Ubuntu 18.04" in Round 19. — [Round 19 blog](https://www.techempower.com/blog/2020/05/28/framework-benchmarks-round-19/)
  - Earlier rounds had community-contributed Windows tests, per a historical FAQ entry, and the config schema still lists `os` as "Linux or Windows". — [Expected Questions #10](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Expected-Questions); [Codebase Framework Files](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Codebase-Framework-Files)
- **Connection churn:** every wrk script sends `Connection: keep-alive` over a fixed pool of connections, and keep-alive is "strongly encouraged". — [toolset/wrk/concurrency.sh](https://github.com/TechEmpower/FrameworkBenchmarks/blob/master/toolset/wrk/concurrency.sh); [Framework Tests Overview](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Framework-Tests-Overview)
- **Bodies, routing and middleware:**
  - All requests are GETs with tiny responses and no gzip.
  - Routing may be "hand-crafted minimalist routing using control structures such as if/else branching" where that is canonical for the framework.
  - Disk logging must be disabled.
  — [Framework Tests Overview](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Framework-Tests-Overview)
  - There are no large-payload tests. — [litestar #1620](https://github.com/litestar-org/litestar/discussions/1620)
- **Robustness:** TFB has no fault-injection test. Most top entries failed an informal database-restart test in 2024. — [issue #8790](https://github.com/TechEmpower/FrameworkBenchmarks/issues/8790)
- **Status:** the repository has been archived since 2026-03-24, so no new official runs will happen (see Q5). — [issue #10932](https://github.com/TechEmpower/FrameworkBenchmarks/issues/10932)

### Inferences
Synthesis, Meleys goal by goal (my analysis, based on the findings above):

| Meleys goal | What TFB covers | What it implies / what must supplement |
|---|---|---|
| Highest throughput | Partly. HTTP/1.1 keep-alive and pipelined plaintext, tiny GETs, peak of 15-s runs, Linux only | HTTP/1.1 small-response efficiency only. Add HTTP/2 and HTTP/3 throughput tests (h2load or similar), TLS-on runs, and realistic payloads. |
| Lowest latency incl. p99+ | No. Closed loop, measured at saturation, avg/stdev/max only, coordinated omission | Needs open-model, constant-arrival-rate tests with HdrHistogram (wrk2-style), latency-versus-load curves, and runs of minutes, not 15 s, to capture JIT and GC behaviour. Report p99/p99.9/p99.99. |
| Smallest memory (idle and under load) | No. Only system-wide dool averages are collected, and never shown | Measure per-process RSS at idle and under load, per-connection memory, and JVM heap, off-heap and direct buffers. Fix JDK flags and heap sizes across competitors, because TFB lets each entry choose its own. |
| CPU efficiency | No | Measure CPU-seconds per request at a fixed offered load, below saturation. |
| HTTP/2 and HTTP/3 first-class | No (wrk is HTTP/1.1, plain http) | Needs separate suites. TFB's pipelined plaintext exercises an HTTP/1.1-only optimisation path. |
| Newest JDKs | No (archived snapshot; each entry pins its own image) | Run a continuous in-house matrix across JDK versions. |
| Linux, macOS, Windows | Linux/Docker only | Run cross-OS (macOS kqueue, Windows) in-house. TFB cannot speak to this goal at all. |
| Real-world behaviour | No churn, TLS handshakes, large or streamed bodies, uploads, slow clients, many routes or middleware, fault recovery | All of this needs dedicated scenarios. |

- TFB is still useful as a familiar sanity check (HTTP/1.1 JSON and plaintext efficiency, Fortunes-style DB+template work) and as a public corpus of competitors' highly tuned code to study. It cannot validate the Meleys vision as a whole.

### Gaps
- Not verified: whether TFB's raw per-run logs on the TechEmpower logs server keep wrk's printed `--latency` percentile distribution. The scripts pass `--latency`, so the logs may hold p50–p99 even though `results.json` and the website do not.
- I found no explicit TechEmpower announcement that ended Windows tests. The "Linux-only since the 2018 Docker era" conclusion is inferred from the Round 16 and Round 19 posts and the toolset.

## 5. Is a TFB ranking still meaningful and credible in 2025–2026?

### Takeaway
As of 2026-10-21, TFB is a frozen snapshot, not a live benchmark. Round 23 (2025-03-17) is the last official round. TechEmpower sunset the project and archived the repository on 2026-03-24 without giving a reason. Community reaction ranged from "good riddance" to "a rough guide to performance brackets". Successors such as HttpArena (April 2026) are emerging. Meleys cannot obtain an official TFB ranking.

### Cited Findings
- Round 23 was published 2025-03-17 by Mike Smith (TechEmpower), on new hardware. — [Round 23 blog](https://www.techempower.com/blog/2025/03/17/framework-benchmarks-round-23/)
- TechEmpower's last framework-benchmarks post before the sunset (2026-03-16) told the Spring/MongoDB 25× story. — [TechEmpower blog 2026-03-16](https://www.techempower.com/blog/2026/03/16/how-our-benchmarks-led-to-a-25x-mongodb-performance-improvement/)
- Issue #10932, "Sunsetting the TechEmpower Framework Benchmarks" (NateBrady23, 2026-03-24, pinned): the repository was archived the same day and is read-only. The issue says "The repository, history, and past results remain as a snapshot". It gives no reason and mentions no final round. — [TFB issue #10932](https://github.com/TechEmpower/FrameworkBenchmarks/issues/10932)
- The last commit on master is 57d92fb (2026-03-23), a merge of a dependabot PR (#10927). — [TFB commits](https://github.com/TechEmpower/FrameworkBenchmarks/commits/master)
- The HN sunset thread (74 points, 22 comments as indexed) included:
  - "This text lacks information about why it is being sunset."
  - "too many entries, most of which seem tuned to cheating benchmarks"
  - "it still gave a good indication of the performance bracket a language was in"
  - "good riddance"
  - "Hopefully an active fork emerges."
  - one commenter defending its value for cutting cloud costs
  - a pointer to HttpArena

  — [HN: Sunsetting TFB](https://news.ycombinator.com/item?id=47497763)
- Nägeli (2026-03-24): "On 24th of March 2026, the TechEmpower Framework Bechmarks have been discontinued". He describes the current results as "highly missleading and do not reflect typical use cases such as providing a high performance REST API", and proposes HttpArena as a successor. — [DEV](https://dev.to/kaliumhexacyanoferrat/techempower-framework-benchmarks-are-now-archived-whats-next-3l0a)
- HttpArena "Alpha Round", 100 entries:
  - Setup: everything runs on one machine (Threadripper PRO 3995WX) with pinned cores, using "best of three runs".
  - HN post "New HTTP framework benchmarks platform", 2026-04-20.
  - Java placings on HTTP/1.1: vertx 15th, helidon-tuned 23rd, micronaut 28th, quarkus-jvm 33rd, spring-boot 70th.

  — [HttpArena](https://www.http-arena.com/); [HN](https://news.ycombinator.com/item?id=47837890)
- Late-2025 marketing still uses TFB: "Elide hit 800k RPS on TechEmpower…" (HN, 2025-12-15). — [HN](https://news.ycombinator.com/item?id=46282585)

### Inferences
- Any "TFB ranking" cited after March 2026 refers to early-2025 code, JDKs and hardware. It cannot show Meleys's "newest JDK" positioning, and a new framework cannot enter it.
- Meleys could still run the archived open-source toolset on its own hardware to produce TFB-comparable numbers. Those would not be an official ranking, and comparing them with Round 23 data would be confounded by hardware. The honest use is to borrow TFB's well-known test types (JSON, plaintext, Fortunes, perhaps DB) as one familiar module of a broader in-house suite. Round 23 should be cited only as historical context.

### Gaps
- I found no statement on how long the TFB results website and logs server will stay online, and no evidence yet of an active community fork of TFB itself.
- I found no 2026 statement from TechEmpower explaining the sunset.
