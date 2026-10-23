# Alternative benchmark suites (beyond TechEmpower) and in-house performance labs of Java HTTP servers and best-in-class projects (state as of 2026-10-21)

Scope note: TFB itself, load-generator internals and latency methodology, HTTP/2/3-specific tooling, JVM tuning and memory measurement, and CI regression-tracking services are covered by other researchers. Dates below are the dates shown on the pages fetched on 2026-10-21; "not stated" means the fetched page did not say.

## 1. Which public cross-framework suites exist and are maintained as of 2026, and how do they work?

### Takeaway
Three multi-language HTTP suites other than TFB were maintained and refreshed in October 2026, and all three accept Java entries:
- **HttpArena** (http-arena.com, MDA2AV/HttpArena) is the broadest. It runs 27 profiles, including HTTP/1.1, HTTP/2, HTTP/3, gRPC, WebSocket, TLS, connection churn and fixed-rate efficiency runs. It reports req/s, p99, CPU and memory on a 64-core Threadripper. It is young ("Alpha Round") and run from a single GitHub account.
- **the-benchmarker/web-frameworks** is a large catalog (21 Java directories) of minimal HTTP/1.1 routing tests. It reports req/s, p50 to p99 and CPU saturation, but not memory, and its method changed three times in 2026.
- **Sharkbench** caps each entry at one CPU core and simulates I/O plus JSON. It reports req/s, median and P99 latency, a "stability" ratio, RAM and build time. Entries must show production usage.

The other suites are narrower:
- fastify/benchmarks is Node-only.
- smallnest/go-web-framework-benchmark is Go-only and stale.
- LesnyRumcajs/grpc_bench is gRPC-only but has many JVM entries.

### Cited Findings

#### HttpArena (http-arena.com / github.com/MDA2AV/HttpArena) — newest and broadest
- **Maintainer:** The site links to the GitHub repo `MDA2AV/HttpArena` and a Discord server, and names no individual maintainer. Each entry lists its own maintainers in a `meta.json` file — [http-arena.com](https://www.http-arena.com/); [GitHub repo](https://github.com/MDA2AV/HttpArena)
- **Activity and license:** MIT license, 2,609 commits, 133 stars, 90 forks, 29 open issues and 26 open PRs. The repo has a CHANGELOG and a Discord, and the footer reads © 2026 — [GitHub repo](https://github.com/MDA2AV/HttpArena)
- **Freshness:** Results were "Updated 2026-10-20" in the "Alpha Round" — [http-arena.com](https://www.http-arena.com/)
- **Hardware:** AMD Ryzen Threadripper PRO 3995WX with 64 cores. Each container is pinned to dedicated cores, and req/s is the best of three runs — [http-arena.com](https://www.http-arena.com/). A third-party listing adds 251 GB RAM, Ubuntu 24.04, the performance governor, and that the project runs every entry itself, so numbers are not self-reported. That listing also calls HttpArena a TechEmpower successor (the site itself does not say this) — [LibHunt listing](https://www.libhunt.com/r/HttpArena) (secondary source, not verified on the site)
- **Profiles, Connection family:**
  - `baseline`: mixed GET/POST with query parsing, at 512 and 4K connections.
  - `pipelined`: 16x pipelining, reference-only.
  - `limited-conn`: the connection closes after 10 requests, which tests connection churn.
  — [GitHub README](https://github.com/MDA2AV/HttpArena)
- **Profiles, Concurrency family:** `async` sends `GET /delay/{ms}` with a flat 10 ms delay over 32K held connections — [GitHub README](https://github.com/MDA2AV/HttpArena)
- **Profiles, Efficiency family:**
  - `latency-1m`: 1M req/s offered at a fixed rate with zrk, measuring CPU per request.
  - `latency-10k`: the same at a near-idle 10K req/s.
  - `latency-500k-8cpu`: the server runs on 4 cores plus their SMT siblings. Reference-only.
  — [GitHub README](https://github.com/MDA2AV/HttpArena)
- **Profiles, Workload family:**
  - `json-comp`: JSON with gzip or brotli.
  - `json-tls`: JSON over TLS.
  - `8gbit`: not described.
  - `static-tls`: 20 static files over TLS.
  - A 10 KB body echoed over TLS at a pinned 50K req/s.
  — [GitHub README](https://github.com/MDA2AV/HttpArena)
- **Profiles, Database and templates:** `async-db` (an async Postgres scan) and `fortunes` (a TechEmpower-style query plus HTML render), both reference-only — [GitHub README](https://github.com/MDA2AV/HttpArena)
- **Profiles, protocol families:**
  - HTTP/2: `baseline-h2`, `static-h2`, `baseline-h2c`, `json-h2c`.
  - HTTP/3: `baseline-h3`, `static-h3`, over QUIC with TLS 1.3.
  - gRPC: `unary-grpc`, `unary-grpc-tls`.
  — [GitHub README](https://github.com/MDA2AV/HttpArena)
- **Profiles, Gateway, Production stack and WebSocket:**
  - Gateway: `gateway-64` and `gateway-h3`, a reverse proxy plus server under a mixed workload.
  - Production stack: edge, Redis, a JWT auth sidecar and the server, with a 10K-item cache-aside pattern and concurrent reads and writes.
  - WebSocket: `echo-ws`, `echo-ws-pipeline`, `echo-ws-limited`.
  — [GitHub README](https://github.com/MDA2AV/HttpArena)
- **Metrics:**
  - Each profile reports RPS, p99, CPU and memory.
  - For efficiency profiles, CPU is "read exactly from the container's cgroup rather than sampled."
  - The efficiency score weights CPU at 0.50, p99 at 0.25 and mean latency at 0.25.
  - Each family (h1, h2, h3, gw, grpc, ws) gets its own sub-composite badge.
  — [GitHub README](https://github.com/MDA2AV/HttpArena); [raw README](https://raw.githubusercontent.com/MDA2AV/HttpArena/main/README.md)
- **Categories:** Flagship and emerging entries are ranked together. Experimental entries, engines and infrastructure (proxies, static servers) are each ranked separately. "Tuned" entries are excluded from default rankings unless you add `-with-tuned` — [GitHub README](https://github.com/MDA2AV/HttpArena)
- **Submission:**
  1. Add `frameworks/<name>/` with a `meta.json` that includes a `maintainers` array.
  2. Run `./scripts/validate.sh <framework>` to check correctness.
  3. Run `./scripts/benchmark.sh <framework>`.
  4. Use PR commands such as `/benchmark -f <framework>`. A PR entry is benchmarked on the same machine as the others.
  — [GitHub README](https://github.com/MDA2AV/HttpArena); [http-arena.com](https://www.http-arena.com/)
- **Example entry layout:** A third-party framework (mq-bridge) ships an HttpArena entry that lists the test profiles it supports — [mq-bridge HttpArena README on docs.rs](https://docs.rs/crate/mq-bridge/0.3.8/source/scripts/httparena/README.md)
- **Java ranks on the HTTP/1.1 composite:**

  | Rank | Entry | Score |
  |---|---|---|
  | 15th | Vert.x | 5308 |
  | 23rd | Helidon (tuned) | 4372 |
  | 28th | Micronaut | 4233 |
  | 29th | Helidon (production) | 4177 |
  | 33rd | Quarkus (JVM) | 3643 |
  | 52nd | Jooby | 2399 |
  | 70th | Spring Boot | 1745 |

  Netty, Jetty, Undertow and ActiveJ were not in the top 100 shown — [http-arena.com](https://www.http-arena.com/)
- **JVM directories seen in the repo** (in the alphabetical part of the listing that rendered, a–k): `helidon-production`, `helidon-tuned`, `jooby`, plus JVM-language entries `http-kit`, `http4k`, `http4s` — [frameworks directory](https://github.com/MDA2AV/HttpArena/tree/main/frameworks)

#### the-benchmarker/web-frameworks (dashboard: web-frameworks-benchmark.vercel.app / .netlify.app)
- **Maintainer:** The GitHub org `the-benchmarker`. No lead is named, and `@waghanza` is the fallback reviewer for dependency PRs — [GitHub repo](https://github.com/the-benchmarker/web-frameworks)
- **Activity:** About 9,481 commits on `develop`, 7.1k stars, 736 forks, 221 open issues and 15 open PRs — [GitHub repo](https://github.com/the-benchmarker/web-frameworks)
- **Workload:** Each implementation listens on port 3000 and serves three routes, which a shared RSpec contract (`.spec/`) checks before benchmarking:
  - `GET /` returns an empty 200.
  - `GET /user/:id` echoes the id.
  - `POST /user` returns an empty 200.

  The default run is 15 s of `GET /`. Concurrency is configurable, for example `CONCURRENCIES=64,256,512` — [README](https://raw.githubusercontent.com/the-benchmarker/web-frameworks/develop/README.md)
- **Load generator history:**
  - Current: `zrk --closed`, closed-loop over keep-alive connections.
  - Until 2026-08: `oha`, with a new connection per request.
  - A 2026-09 revision used a linear ramp to 500k req/s.
  - The README says "The method changed in 2026 and results from different revisions are not comparable."
  — [README](https://raw.githubusercontent.com/the-benchmarker/web-frameworks/develop/README.md); [GitHub repo](https://github.com/the-benchmarker/web-frameworks)
- **Isolation and latency pass:**
  - `SERVER_CPUS` and `LOAD_CPUS` take cpuset specs so the server and the load generator run on separate cores.
  - An optional `LATENCY_RATE` pass runs open-loop at a fixed rate with coordinated-omission correction. It is off by default and reported separately.
  - Runs where the server used under 50% of its allotted CPU are flagged as likely generator-bound.
  — [GitHub repo](https://github.com/the-benchmarker/web-frameworks)
- **Metrics:**
  - Reported: req/s, data received, duration, latency p50, p75, p90 and p99.
  - `saturation.json` records the server's share of allotted CPU and how many cores it kept busy.
  - No memory or RSS metric and no p99.9 is mentioned. No hardware is specified; the README says to read results alongside the machine they came from.
  — [GitHub repo](https://github.com/the-benchmarker/web-frameworks)
- **HTTP versions:** Only keep-alive HTTP/1.1 is implied. HTTP/2 and HTTP/3 are not mentioned — [GitHub repo](https://github.com/the-benchmarker/web-frameworks)
- **Submission:**
  1. Add `<language>/<framework>/` with a `config.yaml` (version, website, files, start command, engine variants), the app and lockfiles.
  2. Run `bundle exec rake config`, then `make -f <path>/.Makefile build test`. Dockerfiles and Makefiles are generated from the config.
  3. Read CONTRIBUTING.md, and say in the PR if you maintain the framework.
  — [GitHub repo](https://github.com/the-benchmarker/web-frameworks)
- **Java entries (21 directories):** activej, armeria, aspectran, blade, helidon-se, javalin, jersey-grizzly2, jersey3-grizzly2, jooby, light-4j, micronaut, quarkus, rapidoid, restheart, spark, spring-webflux, spring, struts2, undertow, vertx, vertx4web — [java directory](https://github.com/the-benchmarker/web-frameworks/tree/develop/java)
- **Disclaimers:**
  - The README calls these "minimal HTTP throughput and latency tests, not a substitute for profiling a production application."
  - Small differences are noise until a run is repeated.
  - Results should not be extrapolated to database-heavy apps.
  — [GitHub repo](https://github.com/the-benchmarker/web-frameworks)

#### Sharkbench (sharkbench.dev / github.com/sharkbench/sharkbench)
- **Maintainer and freshness:** The site credits Tien Nam Do (© 2026) and shows "Last updated: 2026-10-14" — [sharkbench.dev](https://sharkbench.dev/)
- **Repo activity:** 256 commits, 94 stars, 24 forks, 6 issues — [GitHub repo](https://github.com/sharkbench/sharkbench)
- **Hardware:** "Linux on an Intel Core Ultra 5 325." Core count and RAM are not listed — [sharkbench.dev/web](https://sharkbench.dev/web)
- **Workload:**
  - Each app serves two routes, `/api/v1/periodic-table/element` and `/api/v1/periodic-table/shells`, with a `symbol` query parameter. Requests to the two routes are random.
  - The app fetches JSON from a data server on `127.0.0.1:5002`, which "is used to simulate I/O (similar to database queries)."
  - Web benchmarks run in Docker with CPU "limited to 1 core equivalent" and host networking.
  — [GitHub repo](https://github.com/sharkbench/sharkbench)
- **Not stated:** the load generator, default concurrency, duration and how memory is measured — [GitHub repo](https://github.com/sharkbench/sharkbench); [sharkbench.dev/web](https://sharkbench.dev/web)
- **Metrics:**
  - Requests per second.
  - Latency, with a Median/P99 toggle.
  - "Stability", defined as median/P99 latency and shown as a percentage, where higher is better.
  - RAM.
  - Build time, which covers dependency fetch plus compile and excludes SDK setup.
  - A separate computation benchmark (Leibniz π) measures time and memory.
  — [sharkbench.dev/web](https://sharkbench.dev/web); [sharkbench.dev](https://sharkbench.dev/)
- **HTTP versions:** Not stated — [sharkbench.dev/web](https://sharkbench.dev/web)
- **Java ranks among 55 web entries:**

  | Rank | Entry | req/s | Latency | Stability | RAM |
  |---|---|---|---|---|---|
  | 1 | Vert.x (Temurin) | 34,697 | 0.9 ms | 43.00% | 323.2 MB |
  | 8 | Vert.x (Semeru) | 20,650 | 1.4 ms | — | 224.6 MB |
  | 26 | Quarkus Reactive | 11,712 | — | 1.98% | 234.2 MB |
  | 34 | Spring Boot WebFlux (Temurin) | 4,447 | — | — | 277.8 MB |
  | 36 | Quarkus Imperative | 3,789 | — | — | 247.2 MB |
  | 37 | Spring Boot MVC (Temurin) | 2,962 | — | — | 286.4 MB |
  | 40 | WebFlux (Semeru) | 1,908 | — | — | — |
  | 41 | MVC (Semeru) | 1,800 | — | — | — |
  | 42 | Ktor on Netty | 1,743 | — | — | — |

  For contrast, Rust Hyper is #2 at 27,307 req/s with 8.3 MB RAM, Axum is #3 with 16.5 MB, and ASP.NET Core is #13 at 17,041 req/s with 101.8 MB. There are no Javalin, Micronaut, Helidon or raw Netty entries — [sharkbench.dev/web](https://sharkbench.dev/web)
- **Submission:**
  - Directory: `benchmark/web/<language>/<framework>-<min-framework-version>-<mode>-<min-version>/` with a `benchmark.yaml`.
  - Listen on 0.0.0.0:5001.
  - The guidelines require the project to be "production-ready and actively maintained", at least 3 months old, and with evidence of production usage.
  — [GitHub repo](https://github.com/sharkbench/sharkbench)

#### fastify/benchmarks (Node.js only)
- **Maintainer:** The `fastify` GitHub org — [GitHub repo](https://github.com/fastify/benchmarks)
- **Workload:** `GET /` returns `{hello:'world'}`, and `POST /` echoes a JSON body — [GitHub repo](https://github.com/fastify/benchmarks)
- **Load:** `autocannon -c 100 -d 40 -p 10`: 100 connections, 10 pipelined requests, 40 s, with a warm-up round followed by a measured round — [GitHub repo](https://github.com/fastify/benchmarks)
- **Hardware:** GitHub Actions Linux x64 with 4 vCPU and 15.6 GB, running Node v24.22.0. The README warns about "noisy neighbor" VM variance — [GitHub repo](https://github.com/fastify/benchmarks)
- **Metrics:** req/s, latency (ms) and throughput (Mb), with cold-start metrics in METRICS.md. Result tables are generated automatically into the README, and the latest GET run was dated Mon Oct 19 2026 — [GitHub repo](https://github.com/fastify/benchmarks)
- **Inclusion criteria:** At least 500 downloads per week, repo activity within the last month, and use of the Node.js HTTP module. This excludes Java — [GitHub repo](https://github.com/fastify/benchmarks)

#### smallnest/go-web-framework-benchmark (Go only; stale)
- **Maintainer:** smallnest. Apache-2.0, 309 commits, 2.1k stars — [GitHub repo](https://github.com/smallnest/go-web-framework-benchmark)
- **Scenarios:**
  - The handler sleeps 0, 10, 100 or 500 ms to mimic I/O.
  - A concurrency test runs with a 30 ms handler at 100, 1,000 and 5,000 clients.
  - A CPU-bound test runs at 5,000 concurrency.
  - Keep-alive, non-keep-alive and pipelining variants.
  — [GitHub repo](https://github.com/smallnest/go-web-framework-benchmark)
- **Tooling and hardware:** wrk on a 4-core 2 GHz KVM VM with 16 GB, go1.18.5 and Ubuntu 22.04.1 — [GitHub repo](https://github.com/smallnest/go-web-framework-benchmark)
- **Metrics:** Throughput, latency, and heap allocations in MB — [GitHub repo](https://github.com/smallnest/go-web-framework-benchmark)
- **Staleness:** The README says "Last Test Updated: 2020-05", yet the toolchain it lists is from 2022. Results are effectively unmaintained — [GitHub repo](https://github.com/smallnest/go-web-framework-benchmark)
- **Submission:** Add the framework to the README, `server.go` and `libs.sh`, then open a PR — [GitHub repo](https://github.com/smallnest/go-web-framework-benchmark)

#### LesnyRumcajs/grpc_bench (gRPC only, cross-language, many JVM entries)
- **Load and limits:**
  - Load generator: `ghz`.
  - Server: 1 CPU and 512 MB RAM. Client: 1 CPU and 50 connections.
  - 1,000 concurrent requests for 20 s after a 5 s warm-up. The README recommends about 300 s for long-running services.
  — [GitHub repo](https://github.com/LesnyRumcajs/grpc_bench)
- **Metrics:** req/s, average latency with p90, p95 and p99, and average CPU and memory taken from `docker stats` — [GitHub repo](https://github.com/LesnyRumcajs/grpc_bench)
- **JVM entries:**
  - grpc-java under G1, Parallel, Serial, Shenandoah and ZGC, plus OpenJ9.
  - Armeria, Vert.x, Micronaut, Quarkus (JVM and native), and an AOT build.
  - Kotlin and Scala entries.
  — [GitHub repo](https://github.com/LesnyRumcajs/grpc_bench)
- **Freshness and contributions:** No current results date is shown. Results are posted in GitHub Discussions and the wiki — [GitHub repo](https://github.com/LesnyRumcajs/grpc_bench)

#### One-off 2025–2026 comparisons (articles, not suites you can join)
- **Okami101 "A 2026 benchmark of main Web API frameworks"** (Adrien Beaudouin, Dec 30, 2025):
  - Workload: a RealWorld/Conduit REST API with PostgreSQL and ORMs.
  - Load: k6 with a constant-arrival-rate executor, with the rate tuned per framework.
  - Hardware: Docker Swarm on Hetzner `cpx32` nodes (4 vCPU, 8 GB), Talos Linux.
  - Metrics: req/s and avg, median, p90 and p95 latency. No p99 or memory.
  - Spring Boot 4.0 on Java 25 led scenario 1 at about 2,978 req/s with p95 50.77 ms.
  - Code is public.
  — [okami101.io](https://www.okami101.io/blog/web-api-benchmarks-2026/)
- **ITNEXT series** (Spring Boot vs Quarkus vs Micronaut, including an April 2026 Java 21 vs 25 edition with Boot 4.0.2, Quarkus 3.31.1 and Micronaut 4.10.7): uses ApacheBench-style tests and repeated runs. This is from search snippets only, since the article is member-only — [ITNEXT](https://itnext.io/performance-benchmark-java-21-vs-25-spring-boot-4-0-2-vs-quarkus-3-31-1-vs-micronaut-4-10-7-b08329dc9917)
- **JCSI 2025 peer-reviewed study** (Lublin University of Technology, Spring Boot vs Quarkus): measured compile time, startup, file size, CPU and RAM, latency, throughput and DB efficiency, in both JAR and native modes — [JCSI](https://ph.pollub.pl/index.php/jcsi/article/view/8047)

#### Vendor labs that also benchmark other frameworks
- **aspnet/Benchmarks:** Its `te` scenarios measure non-.NET frameworks on the same machines as ASP.NET Core: Node.js, Bun, Actix, Xitca, FastHttp, **Netty, Vert.x, Wizzardo, Spring**, Gin and Express. Scenario names follow the pattern `plaintext_<fw>`, `json_<fw>`, `fortunes_<fw>` — [aspnet/Benchmarks scenarios README](https://github.com/aspnet/Benchmarks/blob/main/scenarios/README.md)
- **quarkusio/spring-quarkus-perf-comparison:** An open Apache-2.0 benchmark repo (432 commits, 37 open issues, 32 open PRs). Results are published to `quarkusio/benchmarks`. Community issues (#55, #108) changed both the method and the published results — [repo](https://github.com/quarkusio/spring-quarkus-perf-comparison); [Quarkus blog](https://quarkus.io/blog/new-benchmarks/)

### Inferences
- **HttpArena first.** It is the only public suite found that matches the full Meleys vision: HTTP/1.1, HTTP/2 (h2 and h2c), HTTP/3, TLS, connection churn, WebSocket, fixed-rate CPU-per-request efficiency, p99 and memory. That makes it the highest-value listing besides TFB. Two cautions:
  - Credibility is still forming: it is labeled "Alpha Round", run from a single account, with 133 stars.
  - A new framework would likely enter as "emerging" or "experimental", and experimental entries are ranked separately from flagship ones.
- **Two HttpArena entries.** Helidon ships both `helidon-production` and `helidon-tuned`. Meleys could do the same so that its default configuration appears in the headline rankings.
- **the-benchmarker is cheap to join** (one `config.yaml` plus generated Docker files) and has the largest Java peer set (21 entries). Its limits: HTTP/1.1 only, no memory metric, three method changes in 2026, and an unpublished machine. Treat it as a "routing overhead" signal, not proof of the vision.
- **Sharkbench needs patience.** It requires at least 3 months of age and evidence of production usage, so Meleys cannot enter on day one.
- **Sharkbench's one-core cap fits a footprint story but penalizes the JVM.** Each entry gets one core and the site shows RAM, so it rewards per-core efficiency and a small footprint, which matches the Meleys goals. But the JVM's JIT and GC threads share that one core, so the setup is structurally hard for Java. Every Java entry on the board uses 170–323 MB against 8–20 MB for Rust.
- **Sharkbench's stability metric is a usable tail-latency proxy.** It is median/P99, and Java reactive stacks sit around 2%, which means P99 is about 50x the median.
- **Closed suites are still worth copying.** fastify/benchmarks and smallnest do not accept Java, but two designs carry over: auto-regenerated README tables, and handlers that sleep to simulate I/O.
- **grpc_bench matters only if Meleys ships gRPC.** It is the established cross-language gRPC venue with JVM peers.
- **crank could run Meleys alongside Netty and Vert.x.** ASP.NET's lab already runs Netty, Vert.x and Spring through crank's `te` scenarios, so Meleys could reuse those Docker-based scenario definitions in its own lab.

### Gaps
- **zrk is unidentified.** It is the load generator now used by the-benchmarker and named by HttpArena, but searches did not identify its author or repository.
- **HttpArena method details are missing.** The fetched pages do not say which generators drive the HTTP/2, HTTP/3, gRPC and WebSocket profiles. Durations, how memory is measured, and the full contents of the "add-framework" guide are also missing (http-arena.com/add-framework returned 404).
- **the-benchmarker Java rankings are unavailable.** The dashboard is JavaScript-rendered and returned no data to the fetcher, and `data.min.json` keys results by numeric `framework_id` with no mapping in the first 100 KB, so the Java ranks and latest run date could not be read. Its hardware is not published.
- **Sharkbench method details are missing:** its load generator, concurrency level and memory-measurement method.
- **TFB status is unverified.** A search result on an unusual subdomain (send-now.techempower.com) claimed TFB's repo was archived in March 2026. This was not verified here, and TFB status is covered by another researcher.
- **The 2026 search found little else.** Apart from HttpArena, no new cross-framework suite that accepts Java turned up. The rest were blog-style one-off comparisons.

## 2. How do Java HTTP servers and frameworks run performance testing in-house?

### Takeaway
Only two Java projects publish a real, repeatable macro-benchmark methodology:
- **Jetty (Webtide)** runs open-loop multi-loader tests with jetty-load-generator and HdrHistogram server-side latency, publishes P99 and max per sample period, and runs a daily Jenkins job.
- **Quarkus (Red Hat perf lab)** uses Hyperfoil, qDup, cgroup/cpuset pinning, time-to-first-request, RSS after the first request and under load, raw results in a public repo, and Horreum.

The rest are thinner:
- **Micronaut** has an archived 2020 Gatling comparison.
- **Spring** publishes startup and memory studies (CDS/AOT on Petclinic).
- **Helidon** cites "local runs" and TFB, with no published method.
- **Netty** has a JMH microbenchmark module.
- **Vert.x, Undertow, Tomcat and ActiveJ** rely mainly on TFB or unpublished claims.

### Cited Findings

#### Jetty (Webtide) — perf lab, jetty-load-generator, HdrHistogram
- **"Jetty 12 performance figures"** (Ludovic Orban, published 05/09/2023, updated 21/02/2024 for Jetty 12.0.2):
  - **Server:** one 16-core Intel Core i9-7960X running one Jetty instance with the default 200 threads, a 32 GB heap and ZGC, on a 10GbE link to a dedicated switch.
  - **Load:** 4 client machines, each with 1GbE, each running one jetty-load-generator (8 GB ZGC heap) at 60,000 req/s, for 240,000 req/s total.
  - **Saturation check:** CPU, network, RAM, heap and HTTP statuses were monitored. Server CPU stayed around 40%.
  — [Webtide](https://webtide.com/jetty-12-performance-figures)
- **Latency method:**
  - The metric is server-side processing time, from reading the first request byte to writing the last response byte. It uses the new `Request.getBeginNanoTime()`, which was also backported to Jetty 11.
  - Values are collected in histograms over 180 sample periods, and min, P99 and max are graphed.
  - Headline percentages come from "P99 integrals", the sum of latencies excluding the slowest 1% of each period.
  — [Webtide](https://webtide.com/jetty-12-performance-figures)
- **Results:**

  | Configuration | Min | P99 | Avg sample peak |
  |---|---|---|---|
  | Jetty 12 EE10 | 7 µs | 34.8 µs | 434 µs |
  | Jetty 12 Core | 6 µs | 30.0 µs | 335 µs |
  | Jetty 11 EE9 | 8 µs | 36.6 µs | 487 µs |

  EE10 is about 5% faster and Core about 22% faster on P99 integrals — [Webtide](https://webtide.com/jetty-12-performance-figures)
- **jetty-perf repo:** "a collection of tests used to benchmark performance of the Jetty server", "currently in the experimental stage". Modules are `common`, `core`, `ee10` and `ee9`. It has 719 commits on `main-12.0.x` — [GitHub](https://github.com/jetty-project/jetty-perf)
- **jetty-perf Jenkinsfile:**
  - Runs `@daily` and triggers the downstream job `/load_testing/jetty-perf-main`, with Jetty version and branch as parameters (default `12.0.33-SNAPSHOT`) and JDK label `load-jdk17`.
  - Archives `**/target/reports/**` and keeps 100 builds.
  - The machine topology lives in the downstream job, which is not public.
  — [Jenkinsfile](https://raw.githubusercontent.com/jetty-project/jetty-perf/main-12.0.x/Jenkinsfile)
- **jetty-load-generator:**
  - Java API and CLI on top of Jetty HttpClient, generating requests asynchronously at a constant rate independent of responses.
  - Resource trees mimic browser page loads. Users and channels map to HTTP/1.1 connections or HTTP/2 streams.
  - Warm-up iterations are not recorded. HTTP/1.1 and HTTP/2 are supported (`--transport h2`).
  - The README example records latency with HdrHistogram's `Recorder`.
  - It advises using several "loaders" plus a lightweight "probe" to record response times, and applying Brendan Gregg's USE method.
  - Branch 4.1.x targets Jetty 12.1.x on Java 17+.
  — [GitHub](https://github.com/jetty-project/jetty-load-generator)
- **HdrHistogram history:** Jetty's perf helper adopted HdrHistogram ("record values without allocation, maintaining a fixed memory cost") to replace an in-house MeasureRecorder — [jetty-perf-helper 1.0.5 tag](https://eclipse.googlesource.com/jetty/org.eclipse.jetty.toolchain/+/jetty-perf-helper-1.0.5)
- **Webtide's critique of pipelined benchmarks:** An early (2012) Vert.x benchmark with deep pipelining was "not really testing any IO performance, but simply how fast they can iterate over a buffer and parse simple requests" — [Webtide](https://webtide.com/?p=937)

#### Quarkus (Red Hat performance lab, Hyperfoil, qDup, Horreum)
- **"Measuring performance" guide (version 3.40):**
  - Measure total RSS, not heap, with `ps -o pid,rss` or `pmap -x`. Do not use `docker stats`; use `docker top <id> -o pid,rss,args` instead.
  - The perf lab captures RSS after the first successful request using `pmap`.
  - Native Memory Tracking costs 5–10%.
  — [Quarkus guide](https://quarkus.io/guides/performance-measure)
- **Startup:** Measured as Time to First Request (TTFR). Take a timestamp just before launch, poll until HTTP 200, and take the difference (`time-to-1st-request.sh`, `date +%s%N`, `gdate` on macOS). Framework-reported startup times ignore JVM bootstrap — [Quarkus guide](https://quarkus.io/guides/performance-measure)
- **Throughput and isolation:**
  - Hyperfoil runs through JBang: 2 min warm-up at 100 connections, 30 s cooldown, then a 30 s measured run.
  - JVM flags: `-XX:ActiveProcessorCount=4 -Xms512m -Xmx512m`.
  - `taskset` pins processes to cores.
  - JMeter and wrk are named as coordinated-omission-affected; wrk2 and Hyperfoil as unaffected.
  — [Quarkus guide](https://quarkus.io/guides/performance-measure)
- **"Quarkus has great performance – and we have new evidence"** (Holly Cummins, March 2, 2026; benchmark built by Eric Deandrea, reviewed by Francesco Nigro):
  - Runs in Red Hat's perf lab with cgroup/cpuset pinning.
  - Hyperfoil uses 16 threads and qDup orchestrates the runs.
  - A "simple" script variant runs on macOS, Linux and Windows.
  - Principles: parity, "normal-ness", high-quality code, and "testing the framework, not the infrastructure."
  — [Quarkus blog](https://quarkus.io/blog/new-benchmarks/)
- **Headline results:**
  - Throughput: 19,255 vs 7,238 tps.
  - Startup: 2.919 s vs 6.569 s.
  - Memory: 269 MB vs 583 MB.
  - Virtual threads added about 6k tps for all frameworks. Native mode roughly halves throughput.
  — [Quarkus blog](https://quarkus.io/blog/new-benchmarks/)
- **Corrections after review:** The app and load generator had been sharing cgroup cores (issue #55), which made slower frameworks look competitive. Spring configuration differences (issue #108) led to a "tuned" variant. The DB still shares cores, and a host-wide cache-drop step remains — [Quarkus blog](https://quarkus.io/blog/new-benchmarks/)
- **The repo (spring-quarkus-perf-comparison):**
  - Modules: springboot3, springboot4, quarkus3, quarkus3-virtual, quarkus3-spring-compatibility. Versions: Quarkus 3.32.2/3.34.1, Boot 3.5.13/4.0.5, Java 25.0.2.
  - Three rigor tiers:
    - laptop scripts (`stress.sh` using Hyperfoil with wrk2 bindings, `1strequest.sh`);
    - a single Linux machine with 12+ cores, via qDup;
    - lab runs "on a regular schedule" under Jenkins, with bare metal needing at least 14 physical cores.
  - Metrics: TTFR, RSS after the first request, RSS under load, throughput, and latency avg/stdev/max.
  - Per-component CPU pinning flags: `--cpus-app`, `--cpus-db`, `--cpus-load-gen`, and others.
  - Results go to `quarkusio/benchmarks` and an internal Horreum. Branches `ootb` and `main` hold the untuned and tuned variants.
  — [GitHub](https://github.com/quarkusio/spring-quarkus-perf-comparison)
- **Hyperfoil:** A "microservice-oriented distributed benchmark framework" that addresses coordinated omission. Apache-2.0, 1,733 commits, builds on Java 21, with modules for controller-api, clustering, k8s-deployer, http and hotrod — [GitHub](https://github.com/Hyperfoil/Hyperfoil)
- **Follow-up Quarkus posts** (search snippets only, not fetched in full):
  - A local Fedora run showed a much smaller gap than the lab's 2.08x. The team traced this to firewalld loading nearly 1000 nftables rules — [Quarkus blog (mirror)](https://es.quarkus.io/blog/hidden-cost-rootless-container-networking/)
  - On time to peak, Quarkus reached about 14,700 req/s within 60 s while Spring plateaued around 3,300 req/s during warm-up — [Quarkus blog (mirror)](https://es.quarkus.io/blog/when-the-jit-cant-keep-up/)

#### Helidon (Níma / Helidon 4 on virtual threads)
- **InfoQ** (Dmitry Alexandrov, Dec 7, 2023):
  - Claims "Our local runs show that the performance is even better compared to pure asynchronous Netty" and cites TFB Round 22 as preliminary. Numbers appear only in images.
  - Tuning findings: discarding buffers beat reusing them, blocking sockets performed best, and async socket writes gave up to 3x with HTTP/1.1 pipelining but no benefit without it.
  - Protocols: HTTP/1.1, HTTP/2, gRPC, WebSocket and TLS/mTLS with ALPN. HTTP/3 is not mentioned.
  — [InfoQ](https://www.infoq.com/articles/helidon-4-adopts-virtual-threads)
- **helidon.io/nima:** Qualitative claims only. The server "replaces Netty in the Helidon ecosystem" and supports HTTP/1, HTTP/2 and gRPC — [helidon.io](https://helidon.io/nima)
- **Counterpoint:** On loom-dev (Oct 2023), a user reported that in the Devoxx plaintext benchmark Níma "failed to outperform even the blocking Undertow." This is community-reported, not a Helidon measurement — [loom-dev](https://mail.openjdk.org/pipermail/loom-dev/2023-October/006269.html)
- **Inside.java (Joe Di Pol, Dec 21, 2025, JavaOne 2025 talk):** Calls the results positive, with no figures on the page — [Inside.java](https://inside.java/2025/12/21/virtual-threads-robust-java-microservices/)
- **HttpArena:** Helidon is listed as both `helidon-production` and `helidon-tuned` — [HttpArena frameworks](https://github.com/MDA2AV/HttpArena/tree/main/frameworks)

#### Micronaut
- **micronaut-comparisons:**
  - Spring Boot 2.2 vs Micronaut 1.3.4 and 2.0.0 M2, using Gatling with 1000 users for 60 s on JHipster sample apps.
  - Startup is taken from each framework's log. Time to first response uses a Node script that polls `/hello-world`. Memory is RSS after the load test (`ps x -o rss,vsz`, or `tasklist` on Windows).
  - Runs use `-Xmx128m` on an HP ZBook laptop (i7-6700HQ, Fedora 31, JDK 8). bcrypt was replaced with a no-op so it would not dominate.
  - The repo was archived on May 6, 2026.
  — [GitHub](https://github.com/micronaut-projects/micronaut-comparisons)
- **2020 blog:** The team revised its configurations after a community member showed that Spring Boot had been disadvantaged — [Micronaut blog](https://micronaut.io/blog/2020-04-28-performance-comparison-spring-boot-micronaut.html)

#### Spring Boot / Spring Framework / WebFlux
- **Spring blog, Aug 29, 2024** (minimal Spring MVC app on a MacBook M2): CDS roughly halved startup and cut memory about 16%. Adding Spring AOT brought about 2x faster startup and about 27% less memory, and Petclinic showed similar gains — [spring.io](https://spring.io/blog/2024/08/29/spring-boot-cds-support-and-project-leyden-anticipation)
- **Spring blog, Dec 4, 2023** (Petclinic on Java 21): CDS cut startup 30–35%, and CDS plus AOT cut it 36–42% — [spring.io](https://spring.io/blog/2023/12/04/cds-with-spring-framework-6-1/)
- **Spring I/O 2024 slides:** Petclinic JDBC startup fell 42% from Boot 2.7 to 3.3, with memory reductions of 6–35% — [slides](https://2024.springio.net/slides/efficient-containers-with-spring-boot-3-java-21-and-cds-springio24.pdf)

#### Netty
- **netty-microbench:** A JMH module, skipped by default and run with `mvn -DskipTests=false test`. The base class defaults to 10 warm-up and 10 measurement iterations of 1 s each, with leak detection disabled. A `netty5-microbench` artifact also exists — [netty.io wiki](https://netty.io/wiki/microbenchmarks.html); [Maven Central](https://central.sonatype.com/artifact/io.netty/netty5-microbench/)
- **External macro runs:** Netty appears in ASP.NET's crank `te` scenarios — [aspnet/Benchmarks scenarios README](https://github.com/aspnet/Benchmarks/blob/main/scenarios/README.md)

#### Vert.x
- **Julien Viet's HTTP/2 talk:** Uses a constant-throughput benchmark: requests are paced at set rates, the ratio of performed to planned requests is logged, and response-time percentiles are recorded. The harness was published as vietj/http2-bench — [Speaker Deck](https://speakerdeck.com/vietj/2)
- **Reproducibility (2017 thread):** Viet wrote that any benchmark should "provide a way to reproduce it for others, otherwise it's just useless" — [vertx Google Group](https://groups.google.com/g/vertx/c/vb0logBJbpo)

#### ActiveJ
- **Claims without numbers:** The docs say ActiveJ HTTP "significantly outperforms" Spring, Jetty and Vert.x, and is "Competitively fast as proven by benchmarks." No numbers appear in the docs, and the docs cover v6.0-beta2 with a © 2024 footer — [activej.io/http](https://activej.io/http)
- **Benchmarks page:** `activej.io/benchmarks` returned 404 on 2026-10-21 (fetch attempt). ActiveJ does have a the-benchmarker entry — [java directory](https://github.com/the-benchmarker/web-frameworks/tree/develop/java)

#### Smaller Java servers
- **robaho/httpserver** (README, self-reported): about 3x Jetty on HTTP/2 and 5x on HTTP/1, and more than 10x the JDK HttpServer on TechEmpower with JDK 23 on identical hardware. The author has submitted it to TFB for third-party confirmation. Sample wrk output (64 connections, Mac mini, pipelined) shows about 694k req/s — [README mirror](https://download.plaud.ai/robaho/httpserver)

### Inferences
- **Copy Jetty's server-side histogram.** Record from first byte read to last byte written with an allocation-free HdrHistogram, so server latency can be told apart from client and network latency. Publish P99 and max per sample period over many periods, not one end-of-run percentile.
- **Copy Jetty's load shape.** Run open-loop at a constant rate from several loader machines and keep a separate low-rate probe for latency, then check that nothing saturates: CPU, NIC, heap and error statuses.
- **Copy Quarkus's footprint method.** Use TTFR from an external timestamp, RSS via `pmap`/`ps` after the first request and under load, no `docker stats`, a fixed heap, pinned cores per component, and tiered scripts for laptop, single Linux box and lab. These fit Meleys' startup and memory claims, and the simple tier already runs on macOS and Windows.
- **Copy Quarkus's openness.** Publish raw results in a separate repo, keep "out-of-the-box" and "tuned" variants, and invite review. Quarkus' own results changed materially after issues #55 and #108, which is evidence that public review catches real errors such as shared cores between the app and the load generator.
- **Nobody publishes HTTP/3 in-house numbers.** No surveyed Java project publishes in-house HTTP/3 benchmarks, and HTTP/2 coverage is limited to Jetty's generator and Viet's old harness. Meleys would be differentiated by publishing its own HTTP/1.1, HTTP/2 and HTTP/3 lab results.
- **Treat self-published speed claims with caution.** ActiveJ, robaho and Helidon's "faster than Netty" all lack a published method. Independent venues (TFB, HttpArena, the-benchmarker) or a reproducible public harness are what create credibility.

### Gaps
- **Not found: public in-house macro-benchmark labs** for Netty (beyond JMH), Vert.x (current), Undertow, Tomcat, Spring WebFlux/Reactor Netty throughput, and Micronaut 4.x. No dashboards or regular reports turned up.
- **Jetty's downstream job is not public.** The downstream `jetty-perf-main` job (machine names, loaders vs probe, protocols, durations) could not be inspected, and the jetty-perf README contains only an introduction.
- **Quarkus lab hardware** (CPU model, cores) and the exact Hyperfoil rates and durations are not published in the blog or README.
- **Helidon's numbers are image-only** in InfoQ, and the full JavaOne 2025 talk was not reviewed.
- **The Hyperfoil website (hyperfoil.io) returned 503**, so protocol support (HTTP/2, HTTP/3) and report formats could not be confirmed from primary docs.

## 3. Which practices from other ecosystems are worth copying (ASP.NET crank and dashboard, hyper/tokio, nginx, h2o, uWebSockets)?

### Takeaway
ASP.NET Core's setup is the model to copy:
- **Tooling:** crank (a controller CLI plus HTTP-API agents with job queues, Docker or .NET jobs, and wrk, wrk2 or bombardier load jobs).
- **Scenarios:** dozens of YAML scenario files covering plaintext, JSON, DB, TLS, connection-close, proxies, gRPC, WebSockets, SignalR, static files, HTTP/3 and other frameworks.
- **Machines:** named Linux and Windows machine profiles.
- **Cadence:** a daily sweep generated from JSON configs, plus an on-demand PR-benchmark bot.
- **Publishing:** a public Power BI dashboard whose pages list the exact crank command lines used.

The other projects contribute single practices:
- **nginx:** RPS, SSL-TPS and throughput tests per core count, with pinned wrk.
- **h2o:** browser-like download timings on an emulated slow link.
- **uWebSockets:** fast native clients and a rule that the server must reach 100% CPU.
- **hyper:** in-repo end-to-end benches.

### Cited Findings

#### ASP.NET Core — crank architecture
- **Components:**
  - "crank-agent is a service that executes the jobs"; "crank is a command-line controller that enqueues jobs and records results."
  - Built-in load jobs: wrk, wrk2 and bombardier.
  - It deploys multi-tier apps as .NET projects or Docker containers.
  - Results are stored as JSON or in SQL Server for charting.
  - Requires the .NET 10 SDK and installs with `dotnet tool update Microsoft.Crank.Controller --version "0.2.0-*"`. The repo has 886 commits and 1.1k stars.
  — [dotnet/crank](https://github.com/dotnet/crank)
- **Architecture doc:**
  - The controller orchestrates jobs across one or more agents and polls their status.
  - Each agent exposes an HTTP API (CRUD on a job queue) and runs jobs one at a time.
  - A job is a Dockerfile or a .NET project, and a scenario is a list of jobs, for example an `application` job plus a `load` job that receives `serverPort` and `path`.
  — [crank architecture](https://github.com/dotnet/crank/blob/main/docs/reference/architecture.md)
- **Metrics in sample output:**
  - Application job: CPU usage %, raw CPU usage %, working set (MB), build time (ms), published size (KB).
  - Load job: requests, bad responses, mean and max latency (µs), max RPS.
  — [crank getting started](https://github.com/dotnet/crank/blob/main/docs/getting_started.md)
- **Other documented features:** benchmarking local source or Docker containers, BenchmarkDotNet microbenchmarks, event counters, post-processing scripts, pre-commands, custom measurements pushed from a job, and SQL Server result storage — [crank docs index](https://github.com/dotnet/crank/blob/main/docs/README.md)

#### ASP.NET Core — scenarios and machines
- **Scenario files (`*.benchmarks.yml`):** antiforgery, blazor, containers, database, dotnet, efcore, goldilocks (Native AOT), grpc, httpclient, json, mono, networkstream, orchard, plaintext, platform, proxy, proxy.grpc, quic, redis, rejection, signalr, sslstream, staticfiles, te, tls, websocket. Profile files include `steadystate.profile.yml` — [scenarios folder](https://github.com/aspnet/Benchmarks/tree/main/scenarios)
- **Scenario details:**
  - Plaintext runs with 16-request pipelining, with `https`, `mvc`, `mapaction`, `connectionclose` and `connectionclosehttps` variants.
  - Proxy scenarios cover YARP, HttpClient, nginx, HAProxy and Envoy over http, https or h2, with configurable response size.
  - gRPC covers unary, server-streaming and ping-pong, using h2load and custom clients.
  - SignalR runs over WebSockets, SSE or long polling. There is also a WebSocket echo and a static-file scenario sized to approach network bandwidth.
  - The httpclient scenarios take `httpVersion` 1.1, 2.0 or 3.0, and 3.0 requires HTTPS.
  - Warm-up and duration default to 15 s each.
  - `collectRequestTimings` records time to headers, to first byte and to last byte (not supported with wrk).
  — [scenarios README](https://github.com/aspnet/Benchmarks/blob/main/scenarios/README.md); [scenarios folder](https://github.com/aspnet/Benchmarks/tree/main/scenarios)
- **Machine profiles:**

  | Profile | CPU | Logical cores | RAM | OS |
  |---|---|---|---|---|
  | `aspnet-perf-lin` | Xeon E-2336 | 12 | 32 GB | Ubuntu 22.04 |
  | `aspnet-perf-win` | Xeon E-2336 | 12 | 32 GB | Windows Server 2022 |
  | `aspnet-gold-lin` | Xeon Gold 6330 | 56 | 64 GB | Ubuntu 22.04 |
  | `aspnet-gold-win` | Xeon Gold 6330 | 56 | 64 GB | Windows Server 2022 |

  The older Citrine and Siryn profiles are retired — [scenarios folder](https://github.com/aspnet/Benchmarks/tree/main/scenarios)
- **Example command:** `crank --config https://raw.githubusercontent.com/aspnet/Benchmarks/main/scenarios/plaintext.benchmarks.yml --scenario plaintext --profile aspnet-perf-lin` — [scenarios README](https://github.com/aspnet/Benchmarks/blob/main/scenarios/README.md)

#### ASP.NET Core — continuous runs and public dashboard
- **Continuous benchmarking:** The ASP.NET team runs all scenarios in the repo continuously. Results are at aka.ms/aspnet/benchmarks. The apps and images "work best when used with Microsoft Crank" — [aspnet/Benchmarks](https://github.com/aspnet/Benchmarks)
- **Daily sweep pipelines:**
  - "The CI pipelines that run the daily benchmark sweep are generated by the pod-scheduler" from JSON configs.
  - The configs cover the on-prem Gold fleet (split across two YAML files), Azure cloud pods and Cobalt-hosted pods. The schedule is a cron in `metadata.schedule`.
  - Snapshot tests fail CI if the JSON and the generated YAML drift.
  — [build/README.md](https://github.com/aspnet/Benchmarks/blob/main/build/README.md)
- **PR-benchmark bot:** `prbenchmarks.yml` polls benchmark requests on dotnet/aspnetcore, dotnet/efcore and dotnet/runtime every 30 minutes. Supported x64 machines are the 56-core gold-lin and gold-win, and ARM64 runtime jobs were retired — [build/README.md](https://github.com/aspnet/Benchmarks/blob/main/build/README.md)
- **Cross-language template:** A `minimal-todo` template runs on gold-lin and on Azure `azure-arm64`/`cobalt-cloud-lin` pods. C# runs in every slot, while Go and Rust run in a shared weekly slot. Generation, admission and calibration tooling lives in a separate `dotnet-performance-tools` repo — [build/README.md](https://github.com/aspnet/Benchmarks/blob/main/build/README.md)
- **Dashboard hosting:** `aka.ms/aspnet/benchmarks` returns a 301 to a public `msit.powerbi.com/view?r=...` Power BI report — [aka.ms redirect](https://aka.ms/aspnet/benchmarks)
- **Dashboard contents:** "Continuous benchmarking results are available on this PowerBI dashboard", "The dashboard's navigation is at the bottom of the page", and "Most pages in the Power BI dashboard list the crank command lines that were used" — [scenarios README](https://github.com/aspnet/Benchmarks/blob/main/scenarios/README.md)

#### nginx (F5) — published sizing methodology
- **RPS test:** One wrk per CPU, pinned with `taskset`, each with 50 connections requesting a 1 KB file for 180 s — [F5 "NGINX Plus sizing guide: how we tested"](https://www.f5.com/company/blog/nginx/nginx-plus-sizing-guide-how-we-tested)
- **SSL/TLS TPS test:** A new connection per request (`Connection: close`, 0-byte file) — [F5 "NGINX Plus sizing guide: how we tested"](https://www.f5.com/company/blog/nginx/nginx-plus-sizing-guide-how-we-tested)
- **Throughput test:** A 1 MB file — [F5 "NGINX Plus sizing guide: how we tested"](https://www.f5.com/company/blog/nginx/nginx-plus-sizing-guide-how-we-tested)
- **Core scaling:** Worker processes are varied by CPU count — [F5 "NGINX Plus sizing guide: how we tested"](https://www.f5.com/company/blog/nginx/nginx-plus-sizing-guide-how-we-tested)
- **Ingress Controller test:** 1,000 connections and 44 wrk threads. Pinning wrk with taskset gives more consistent results than adding threads — [NGINX blog](https://blog.nginx.org/blog/testing-performance-nginx-ingress-controller-kubernetes)
- **CPS:** The NGINX web-server performance post reports CPS (connections per second), and the HTTPS variant is often called SSL TPS — [NGINX blog](https://blog.nginx.org/blog/testing-the-performance-of-nginx-and-nginx-plus-web-servers)
- **Age:** These posts use older NGINX versions and hardware. The method is reusable but the numbers are historical — [NGINX blog](https://blog.nginx.org/blog/testing-the-performance-of-nginx-and-nginx-plus-web-servers)

#### h2o — browser-realistic HTTP/2 timings (outdated data)
- **Download timings:** First-paint and download-completion times on an emulated 8 Mbps / 100 ms link (Linux qdisc `tbf` plus `netem`) — [h2o benchmarks](https://h2o.examp1e.net/benchmarks.html)
- **Static file:** A 612-byte file with 250 concurrent clients — [h2o benchmarks](https://h2o.examp1e.net/benchmarks.html)
- **Load tools:** wrk for HTTP/1 and h2load for HTTP/2 — [h2o benchmarks](https://h2o.examp1e.net/benchmarks.html)
- **Hardware:** Two EC2 c3.8xlarge instances — [h2o benchmarks](https://h2o.examp1e.net/benchmarks.html)
- **Age:** The data dates from about 2015–2016 (h2o 1.5–1.7 vs nginx 1.9.x) and is outdated — [h2o benchmarks](https://h2o.examp1e.net/benchmarks.html)

#### uWebSockets — native clients, CPU-saturation rule (self-admittedly stale)
- **Clients:** The benchmarks folder has C clients (`load_test.c`, `broadcast_test.c`, `scale_test.c`) and a parser benchmark. Tests are hello-world plus echo with 512 B–16 kB messages — [uWebSockets benchmarks](https://github.com/uNetworking/uWebSockets/tree/master/benchmarks)
- **Validity rule:** The server must reach 100% CPU for a result to count — [uWebSockets benchmarks](https://github.com/uNetworking/uWebSockets/tree/master/benchmarks)
- **Rejected clients:** It rejects Autocannon and the Node `ws` package as clients — [uWebSockets benchmarks](https://github.com/uNetworking/uWebSockets/tree/master/benchmarks)
- **TLS cost:** Reports about 60% of performance retained with TLS 1.3 — [uWebSockets benchmarks](https://github.com/uNetworking/uWebSockets/tree/master/benchmarks)
- **Age:** The README says "It was a while ago this page was written (and it needs updating)" — [uWebSockets benchmarks](https://github.com/uNetworking/uWebSockets/tree/master/benchmarks)

#### hyper / tokio
- **hyper:** In-repo bench files `body.rs`, `connect.rs`, `end_to_end.rs`, `pipeline.rs` and `server.rs`, plus a `support/` folder. The page does not say how they are run — [hyper benches](https://github.com/hyperium/hyper/tree/master/benches)

### Inferences
- **crank fits a Java project.** It is .NET-hosted but runs any Dockerfile job. Meleys could run crank agents on its own machines and reuse the aspnet/Benchmarks `te` scenarios, which already define Netty, Vert.x and Spring jobs, to get same-machine comparisons. The alternative is to build an equivalent controller/agent in Java around Hyperfoil or jetty-load-generator.
- **Most worth copying from ASP.NET:**
  1. Named machine profiles, including Windows machines. This matches Meleys' Linux, macOS and Windows goal; no Java project surveyed publishes Windows numbers.
  2. One YAML scenario catalog per feature family.
  3. A scheduled daily sweep generated from config, plus PR-triggered runs.
  4. A public dashboard where every chart cites the exact command line.
- **nginx's split** into RPS (keep-alive), TLS handshakes per second (new connection per request) and bulk throughput (1 MB), each run per core count, gives a clean "per-core scaling" story that fits a requests-per-second-per-core metric.
- **h2o's emulated-latency download test** is the only practice found that evaluates HTTP/2 or HTTP/3 the way users feel it, through page-load timing on a slow, lossy link, rather than raw RPS. It is a candidate for Meleys' HTTP/3 story.
- **uWebSockets' rule** (a result counts only if the server is at 100% CPU and the client is native) parallels the-benchmarker's under-50%-CPU flag. Both guard against generator-bound results.

### Gaps
- **The Power BI dashboard was not readable.** It is a JavaScript-rendered Power BI report, so its pages, metrics (for example whether it shows latency percentiles, memory or startup) and history depth could not be listed. Only the README statements are available.
- **crank reference gaps:** the command-line reference page (`docs/reference/command_line.md`) returned 404, so the options for iterations, outlier exclusion and comparison were not confirmed.
- **tokio's benchmarking practice was not researched.** No source was retrieved.
- **hyper's bench harness is unconfirmed:** whether it uses Criterion or nightly `#[bench]`, and whether it has a perf dashboard.
- **nginx and h2o numbers are historical.** No current (2025–2026) benchmark publications were found for either.

## 4. Which scenarios beyond TFB's do these suites test, and which metrics do they report?

### Takeaway
Beyond TFB's plaintext, JSON, DB and fortunes, the suites cover:
- **Broad coverage:** HttpArena and ASP.NET's crank scenarios cover protocol variety (h2, h2c, h3, gRPC), TLS, connection churn (close after N requests, connection-close), held idle connections (32K), WebSockets and SignalR, static files, compression, large or echo bodies, proxies or gateways, and fixed-rate efficiency.
- **Narrow coverage:** the-benchmarker adds path-parameter routing and POST. Sharkbench and smallnest simulate downstream I/O.

On metrics, p99 is now common, but few go further:
- **p99.9:** No suite publishes it.
- **Memory:** Only Sharkbench, HttpArena, grpc_bench and the vendor labs (Quarkus, crank) report it.
- **CPU efficiency:** HttpArena (CPU per request, from cgroups) and the-benchmarker (CPU saturation).
- **Startup:** Measured only in vendor or in-house setups (Quarkus TTFR, Micronaut time-to-first-response, Spring CDS studies), never in the public cross-framework suites.

### Cited Findings

| Suite or lab | Scenarios beyond TFB | Throughput | Latency percentiles | Memory | CPU | Startup | HTTP versions |
|---|---|---|---|---|---|---|---|
| HttpArena | 512/4K-conn baseline; close after 10 req; 32K held conns (`/delay`); 1M and 10K req/s fixed-rate efficiency; gzip/brotli JSON; JSON and static over TLS; 10 KB echo; gateway and "production stack" (edge + Redis + JWT sidecar); WebSocket echo/pipeline/limited ([README](https://github.com/MDA2AV/HttpArena)) | RPS, best of 3 ([site](https://www.http-arena.com/)) | p99 plus mean ([README](https://github.com/MDA2AV/HttpArena)) | Yes ([README](https://github.com/MDA2AV/HttpArena)) | Yes, CPU per request from cgroup ([README](https://github.com/MDA2AV/HttpArena)) | Not stated | h1, h2, h2c, h3, gRPC, WebSocket ([README](https://github.com/MDA2AV/HttpArena)) |
| the-benchmarker | Path-param route `/user/:id`, `POST /user`, configurable concurrency, optional fixed-rate latency pass ([README](https://raw.githubusercontent.com/the-benchmarker/web-frameworks/develop/README.md)) | RPS | p50, p75, p90, p99 | Not mentioned | Saturation share and cores busy ([repo](https://github.com/the-benchmarker/web-frameworks)) | No | HTTP/1.1 keep-alive (implied) |
| Sharkbench | Random mix of 2 routes with upstream JSON fetch (simulated I/O), 1-core cap ([repo](https://github.com/sharkbench/sharkbench)) | RPS | Median and P99 toggle; "stability" = median/P99 ([site](https://sharkbench.dev/web)) | RAM | No | No (build time instead) | Not stated |
| fastify/benchmarks | JSON echo POST ([repo](https://github.com/fastify/benchmarks)) | req/s, Mb/s | Latency (ms) | No | No | Cold-start metrics in METRICS.md | Not stated |
| smallnest (Go) | Handler sleep 0–500 ms; 100–5,000 clients; CPU-bound; non-keep-alive; pipelining ([repo](https://github.com/smallnest/go-web-framework-benchmark)) | Throughput | Latency | Heap allocations (MB) | No | No | HTTP/1.1 (implied) |
| grpc_bench | Configurable proto payload scenarios, 1-CPU server ([repo](https://github.com/LesnyRumcajs/grpc_bench)) | req/s | avg, p90, p95, p99 | `docker stats` | `docker stats` | No | gRPC (HTTP/2) |
| ASP.NET crank scenarios | HTTPS, connection-close (plain and HTTPS), MVC vs minimal endpoints, proxies (YARP, nginx, HAProxy, Envoy), gRPC streaming, SignalR (WS/SSE/long polling), WebSocket echo, static files near bandwidth, Redis cache, Native AOT, antiforgery, Blazor, Orchard CMS, non-.NET frameworks ([scenarios](https://github.com/aspnet/Benchmarks/tree/main/scenarios)) | RPS ([crank](https://github.com/dotnet/crank/blob/main/docs/getting_started.md)) | Mean and max in sample output; time-to-headers, first and last byte via `collectRequestTimings` ([README](https://github.com/aspnet/Benchmarks/blob/main/scenarios/README.md)) | Working set ([crank](https://github.com/dotnet/crank/blob/main/docs/getting_started.md)) | CPU % ([crank](https://github.com/dotnet/crank/blob/main/docs/getting_started.md)) | Build time and published size shown ([crank](https://github.com/dotnet/crank/blob/main/docs/getting_started.md)) | h1, https, h2 (proxy), HttpClient 1.1/2.0/3.0, gRPC, QUIC scenario file ([README](https://github.com/aspnet/Benchmarks/blob/main/scenarios/README.md)) |
| Jetty perf | Constant-rate 240k req/s from 4 loaders; servlet (EE9/EE10) vs core handler ([Webtide](https://webtide.com/jetty-12-performance-figures)) | Fixed offered rate | Server-side min, P99, max per period; P99 integrals ([Webtide](https://webtide.com/jetty-12-performance-figures)) | Heap fixed (32 GB) | Monitored (~40%) | No | Generator supports h1 and h2 ([jetty-load-generator](https://github.com/jetty-project/jetty-load-generator)) |
| Quarkus lab | DB-backed CRUD app, virtual threads on/off, JVM vs native, out-of-the-box vs tuned ([blog](https://quarkus.io/blog/new-benchmarks/)) | tps | avg, stdev, max ([repo](https://github.com/quarkusio/spring-quarkus-perf-comparison)) | RSS after first request and under load ([repo](https://github.com/quarkusio/spring-quarkus-perf-comparison)) | Pinned cores | TTFR ([guide](https://quarkus.io/guides/performance-measure)) | Not stated |
| nginx (F5) | Keep-alive RPS, TLS TPS (new connection per request), 1 MB throughput, scaling by core count ([F5](https://www.f5.com/company/blog/nginx/nginx-plus-sizing-guide-how-we-tested)) | RPS, TPS, Gbps | Not stated | No | Per-core scaling | No | h1/https |
| h2o | Emulated 8 Mbps / 100 ms page download: first paint and completion ([h2o](https://h2o.examp1e.net/benchmarks.html)) | "Scores" | No | No | No | No | h1, h2 |

- **Per-core and footprint framing:**
  - Sharkbench caps each framework at 1 core ([repo](https://github.com/sharkbench/sharkbench)).
  - grpc_bench caps the server at 1 CPU and 512 MB ([repo](https://github.com/LesnyRumcajs/grpc_bench)).
  - HttpArena includes an 8-logical-CPU 500K req/s reference profile ([README](https://github.com/MDA2AV/HttpArena)).
  - Quarkus pins `-XX:ActiveProcessorCount=4` with a fixed 512 MB heap ([guide](https://quarkus.io/guides/performance-measure)).

  These are the closest existing analogs to a "requests/sec per core" metric.
- **p99.9:** None of the fetched suite pages lists p99.9. HttpArena lists p99 ([README](https://github.com/MDA2AV/HttpArena)), the-benchmarker lists up to p99 ([repo](https://github.com/the-benchmarker/web-frameworks)), and grpc_bench lists up to p99 ([repo](https://github.com/LesnyRumcajs/grpc_bench)).
- **Many routes and middleware chains:** No public suite found has a dedicated "many routes" test. the-benchmarker has only 3 routes ([repo](https://github.com/the-benchmarker/web-frameworks)). The middleware dimension appears only as ASP.NET's MVC vs endpoint vs map-action variants and antiforgery scenarios ([scenarios](https://github.com/aspnet/Benchmarks/tree/main/scenarios)), and as HttpArena's JWT-sidecar production stack ([README](https://github.com/MDA2AV/HttpArena)).
- **Streaming:** Appears as gRPC server-streaming and ping-pong in ASP.NET ([scenarios README](https://github.com/aspnet/Benchmarks/blob/main/scenarios/README.md)) and as SignalR SSE ([scenarios README](https://github.com/aspnet/Benchmarks/blob/main/scenarios/README.md)).
- **Large payloads:** Appear as ASP.NET's static-file scenario designed to approach network bandwidth ([scenarios README](https://github.com/aspnet/Benchmarks/blob/main/scenarios/README.md)), HttpArena's `8gbit`, 10 KB echo and static-TLS profiles ([README](https://github.com/MDA2AV/HttpArena)), and nginx's 1 MB throughput test ([F5](https://www.f5.com/company/blog/nginx/nginx-plus-sizing-guide-how-we-tested)).

### Inferences
- **The two scorecards to align with.** For Meleys' vision (throughput, p99 and above, idle and loaded memory, h1/h2/h3, cross-OS), HttpArena publicly and an ASP.NET-style crank lab privately cover the most scenario dimensions.
- **What Meleys would add on top.** No public suite reports p99.9 or p99.99, idle RSS or per-core throughput, and none runs on Windows or macOS. Meleys would have to measure and publish these in-house:
  - p99.9 and p99.99 from fixed-rate runs (as Jetty and HttpArena's efficiency profiles do);
  - idle RSS and RSS under load (Quarkus method);
  - RPS per pinned core (Sharkbench and grpc_bench style caps);
  - TTFR;
  - Windows and macOS runs (ASP.NET-style -win profiles).
- **A scenario set that spans all suites:**
  - TFB-style plaintext and JSON;
  - the-benchmarker routing;
  - HttpArena's limited-conn and connection-close churn, plus TLS handshakes per second (nginx);
  - 32K idle held connections, which test memory per connection;
  - h2 multiplexing and h3;
  - WebSocket echo;
  - large static or streaming bodies;
  - a many-route or middleware-chain test that no public suite covers.

### Gaps
- **Unlisted percentiles.** It is unknown whether HttpArena stores p99.9 or full HdrHistogram data behind its tables, and whether the ASP.NET dashboard shows latency percentiles. Neither was visible in the fetched pages.
- **Unknown memory method.** Sharkbench's and HttpArena's methods (peak vs average, RSS vs cgroup memory) could not be confirmed.
- **No "many routes" or deep-middleware suite** was found in any ecosystem during this research.
