# Continuous Benchmarking and Performance-Regression Detection in CI (for Meleys: an open-source Java HTTP framework on GitHub)

Research date: 2026-10-21. "As of" statements refer to pages read on that date unless another date is given. Vendor claims are labelled as such. JMH best practices are out of scope (another researcher covers them); this file covers CI integration, infrastructure, noise and regression detection.

## 1. Which tools and services exist as of 2026, and how do they compare?

### Takeaway
For a JMH-based project that publishes results openly, Bencher fits best. It is open source, free for public projects and self-hostable, it has a native `java_jmh` adapter and statistical thresholds, and it offers optional bare-metal runners. Nyrkiö (a hosted service built on Apache Otava) has the strongest track record for change-point detection on nightly time series. CodSpeed added Java only in May 2026: it uses a JMH fork, supports walltime only, is described as early development, and its standard bare-metal runners are ARM64 Linux only. github-action-benchmark costs nothing but is statistically naive: it compares against the previous result with a default 200% threshold. conbench is effectively legacy.

### Cited Findings

**Bencher (bencher.dev)**
- Free plan: $0/month, public projects only. It includes Bare Metal runner access limited to 1 concurrent job, a 5-minute timeout and the standard queue. The rate limit is 65,535 metrics/day, or 255/day for unclaimed projects. Support is through the community (GitHub issues, Discord). — [Bencher pricing](https://bencher.dev/pricing/)
- Pro plan: $100/month for up to 250 active series. A series is a unique benchmark × measure × testbed that reports at least one result in the month. The price rises to $150 for 251–375 series and $200 for 376–500. On-demand bare-metal runners cost $1.00/hour ($0.01666/min), billed separately per active minute. Pro has unlimited concurrency and no timeout. Enterprise adds Dedicated (flat monthly) and Custom runners, with prices not listed. — [Bencher pricing](https://bencher.dev/pricing/)
- The on-demand bare-metal hardware, `intel-v1`, runs Linux x86_64 with 4 CPUs, 48 GiB RAM and a 128 GiB disk, and code runs in a Firecracker microVM sandbox. Linux ARM64 on-demand is "coming soon". macOS and Windows are available only on Cloud Dedicated/Custom (Enterprise) or with self-hosted runners. — [Bencher pricing](https://bencher.dev/pricing/)
- Self-hosted: free to use under the Bencher License, equivalent to the Cloud Free plan. "Plus" features need an Enterprise license, and self-hosting charges nothing for runner minutes. Bencher deploys on bare metal, Docker or Kubernetes. Code outside "plus" directories is dual-licensed Apache-2.0/MIT; "plus" code uses the Bencher Plus License. — [Bencher pricing](https://bencher.dev/pricing/); [Bencher README](https://raw.githubusercontent.com/bencherdev/bencher/main/README.md)
- The `java_jmh` adapter parses JMH JSON output (`-rf json`) and records `latency` (ns) and `throughput` (ops/s). The value is the mean, and the lower and upper values are the confidence interval. Example: `bencher run --adapter java_jmh --file results.json "java -jar benchmarks.jar -rf json -rff results.json"`. Other adapters cover Rust (criterion, iai, gungraun, which are instruction-count/Callgrind based), Go, C++, .NET, Python and JS. — [Bencher adapters](https://bencher.dev/docs/explanation/adapters/)
- Bencher has seven threshold tests: `static`, `percentage`, `z_score`, `t_test`, `log_normal`, `iqr` and `delta_iqr`. Each threshold takes upper and/or lower boundaries, min/max sample sizes and an optional time window (the docs' example uses 4 weeks). An alert fires when a new metric crosses a boundary limit. `--error-on-alert` fails the run, and benchmarks can be excluded with `_bencher_ignore` name suffixes. — [Bencher thresholds](https://bencher.dev/docs/explanation/thresholds/)
- Recommended GitHub Actions setup:
  - Track the base branch on `push` with `--threshold-test t_test --threshold-max-sample-size 64 --threshold-upper-boundary 0.99 --thresholds-reset --error-on-alert`.
  - On PRs, use `--start-point "$GITHUB_BASE_REF"`, `--start-point-hash`, `--start-point-clone-thresholds`, `--start-point-reset` and `--github-actions '${{ secrets.GITHUB_TOKEN }}'`. The last flag posts a GitHub Check and a PR comment.
  - Fork PRs get no secrets, so the docs prescribe two workflows. A `pull_request` workflow runs the benchmarks and uploads the results and event JSON as artifacts. A `workflow_run` workflow on the default branch uploads them with `--ci-number`. The page also states that Bencher Bare Metal supports fork PRs.
  — [Bencher GitHub Actions guide](https://bencher.dev/docs/how-to/github-actions/)
- Bencher's vendor variance claims: the README says "Typical CI runners: >30% variance" versus "Bencher Bare Metal runners: <2% variance". A search snippet of the pricing page claims <1% wall-clock variance. No methodology is published, and the two figures conflict. — [Bencher README](https://raw.githubusercontent.com/bencherdev/bencher/main/README.md)
- Listed users include Microsoft snmalloc, Google Sedpack, GitLab Git, pnpm, conda, Firezone, Servo, Rustls and clap. The project looks active in 2026 (the pricing page is © 2026). — [Bencher README](https://raw.githubusercontent.com/bencherdev/bencher/main/README.md)

**CodSpeed (codspeed.io)**
- Java support was announced in a changelog entry dated 2026-05-06: "For now, only Walltime mode is supported". Maven and Gradle are supported. — [CodSpeed changelog: Java support](https://codspeed.io/changelog/2026-05-06-java-support)
- The Java docs give these details:
  - The integration "works through a fork of JMH" (`io.codspeed.jmh` 0.2.1), which is not yet on Maven Central ("planning to publish to Maven Central in the future").
  - Gradle users add it as a git submodule with a composite build; Maven users publish it locally.
  - It needs JDK 21 or later and is described as "still in early development".
  - `@BenchmarkMode` annotations are ignored because CodSpeed uses a custom mode, and benchmark classes must not be final.
  - The CI example runs on `runs-on: codspeed-macro-arm64-graviton-ubuntu-22-04` with `CodSpeedHQ/action@v5` and `mode: walltime`.
  — [CodSpeed Java docs](https://codspeed.io/docs/benchmarks/java)
- Kotlin and Scala docs describe the same fork-based, walltime-only approach. — [CodSpeed Kotlin docs](https://codspeed.io/docs/benchmarks/kotlin); [CodSpeed Scala docs](https://codspeed.io/docs/benchmarks/scala); [codspeed-jvm repo](https://github.com/CodSpeedHQ/codspeed-jvm)
- CodSpeed has three executor instruments:
  - CPU Simulation "simulates CPU behavior… instructions executed, cache and memory access patterns", and benchmarks "run only once for consistent, hardware-agnostic results". It is the default recommendation.
  - Walltime runs "on bare-metal runners managed by CodSpeed". It is suggested for I/O-heavy benchmarks or benchmarks longer than about 5 s.
  - Memory captures memory usage and heap allocations.
  The listed languages are Python, Rust, C++, Node.js, Go, Java, Kotlin and Scala. — [CodSpeed instruments](https://codspeed.io/docs/instruments)
- Pricing:
  - Free is $0, "forever free". It includes unlimited runs and repositories, up to 5 users ("unlimited for open-source projects"), **3-month history**, PR reports and checks, and the walltime instrument with **600 macro-runner minutes/month**.
  - Pro is $15/user/month with unlimited history; macro-runner time beyond 600 minutes costs **$0.032/min**.
  - Enterprise is custom and includes custom macro runners (GPU, OS, architecture).
  - The standard macro runner is a "dedicated 16-core ARM64 CPU and 32 GB RAM".
  — [CodSpeed pricing](https://codspeed.io/pricing)
- According to the 2025-07-30 blog, macro runners are AArch64 "AWS Graviton Cortex-A72" machines with all 16 CPUs on bare metal, 32 GB RAM, and Ubuntu 24.04 "with additional stability fine-tuning". The same post lists GitHub-hosted runners as an AMD EPYC 7763 with 4 vCPUs. — [CodSpeed blog: Benchmarks in CI: Escaping the Cloud Chaos](https://codspeed.io/blog/benchmarks-in-ci-without-noise)

**github-action-benchmark (benchmark-action/github-action-benchmark)**
- It supports `tool: 'jmh'` (there is an example in `examples/java`), plus cargo, `go test -bench`, benchmark.js, pytest-benchmark, Google Benchmark, Catch2, BenchmarkTools.jl, BenchmarkDotNet and custom JSON (`name`/`unit`/`value`/`range`). — [github-action-benchmark](https://github.com/benchmark-action/github-action-benchmark)
- It commits results to `dev/bench/data.js` on a `gh-pages` branch and renders a Chart.js dashboard; hovering shows the commit and clicking opens it. Alternatively, `external-data-json-path` stores a JSON file that must be persisted with, for example, `actions/cache`. — [github-action-benchmark](https://github.com/benchmark-action/github-action-benchmark)
- Alerting:
  - `alert-threshold` defaults to "200%".
  - `comment-on-alert` posts a commit comment, and `fail-on-alert` (default false) fails the workflow, with `fail-threshold` setting the failure level.
  - `comment-always` and `summary-always` (Job Summary) report on every run.
  - Each result is compared with the **previous stored result**. No significance testing is described; the optional `range` field is only recorded.
  — [github-action-benchmark](https://github.com/benchmark-action/github-action-benchmark)
- Activity on the fetched page: about 1.3k stars, 187 forks, 99 open issues and 32 open PRs. No release or date was visible. A search summary says the README reports roughly ±10–20% fluctuation on GitHub-hosted runners for its examples; I could not verify this on the fetched page. — [github-action-benchmark](https://github.com/benchmark-action/github-action-benchmark)

**Nyrkiö (nyrkio.com)**
- Nyrkiö is an open-source platform (Apache-2.0) and hosted service for change detection. Results go in through a REST API or the GitHub Action `nyrkio/change-detection` (MIT), released 2025-05-30. The action is based on github-action-benchmark, and detection comes from Apache Otava. — [Nyrkiö blog: change-detection action released](https://blog.nyrkio.com/2025/05/30/nyrkio-change-detection-github-action-is-released/); [nyrkio/nyrkio](https://github.com/nyrkio/nyrkio)
- Nyrkiö v2.0.0 with "Nyrkiö Runner for GitHub" was publicly launched on 2026-02-24, after being shown at FOSDEM 2026. Vendor claims:
  - Typical environmental noise is "+/- 20% to 50%".
  - Pilot customers saw a "factor of 10x" noise reduction; in one example, 60 ns swings were contained within 1 ns.
  - Pilots included Turso Database and UnoDB.
  - A third-party price-performance runner was faster than both, but "it did not improve the stability of the results".
  The post gives no hardware specs or pricing. — [Nyrkiö Runner announcement](https://blog.nyrkio.com/2026/02/24/nyrkio-runner-for-github-is-here-continuous-benchmarking-with-nanosecond-precision/)
- The runners are a paid add-on: you choose a subscription, then change `runs-on`. A search summary of Nyrkiö material described the tuning as no hyperthreading, frequency scaling and turbo disabled, no NUMA, and tuned kernel parameters. I could not verify this on a fetched page, and I found no pricing. — [nyrkio/nyrkio](https://github.com/nyrkio/nyrkio)
- In an SSP 2025 study, Nyrkiö was configured with a p-value of 0.001 ("finds fewer change points, decreasing false positives") and a minimum change magnitude of 5%. — [Yang, Reichelt, Ingo, Hasselbring, arXiv 2510.11310](https://arxiv.org/abs/2510.11310)

**Apache Otava (incubating), formerly Hunter**
- Correction to the brief: Otava is not MongoDB's tool itself. MongoDB built E-Divisive change-point detection in 2017 and open-sourced it (paper published 2020). DataStax extended it into the Hunter CLI, adding Grafana annotations and Slack, and donated the codebase to the ASF in 2024. — [Ingo, "8 Years of Optimizing Apache Otava", arXiv 2505.06758](https://arxiv.org/html/2505.06758v1); [Daly et al., ICPE 2020, arXiv 2003.00584](https://arxiv.org/abs/2003.00584) (its footnote links MongoDB's Python library at github.com/mongodb/signal-processing-algorithms)
- Otava entered the Apache Incubator on 2024-11-27 and was renamed from Hunter on 2025-02-12. The incubator status page shows no graduation (still incubating) and lists no releases. — [Apache Incubator: Otava status](https://incubator.apache.org/projects/otava.html)
- Releases: the Nyrkiö blog announced Otava 0.6.1 (post dated 2025-07-18) and Otava 0.8.0 (post dated 2026-06-06, together with an "Otava Playground"). The incubator page's empty release list appears stale. — [Nyrkiö blog index](https://blog.nyrkio.com/)
- Otava is a Python CLI. CSV, JSON and Graphite are supported by default; extras add PostgreSQL, BigQuery, InfluxDB 3 and Slack notifications. The accepted false-positive probability and minimum change magnitude are tunable. It can compare a feature branch with main. The typical deployment is a Jenkins or cron job after scheduled or per-commit tests, with optional Grafana annotations. — [Apache Otava site](https://otava.apache.org/); [apache/otava repo](https://github.com/apache/otava)
- Algorithm and performance:
  - Hunter replaced E-Divisive's Monte-Carlo permutation significance test with a Student's t-test and windowing. The paper mentions W = 50 in one section and 30 points in another.
  - Cumulative speedups versus the naive implementation are 18,000×–300,000×, and an incremental mode costs O(1) per appended point.
  - Reported users include MongoDB, DataStax, Netflix (a similar system, 2022 blog) and Confluent (nightly Kafka performance results, Kafka Summit London 2024).
  — [Ingo, arXiv 2505.06758](https://arxiv.org/html/2505.06758v1)

**conbench**
- conbench offers language-agnostic JSON result submission, a server with API and dashboard, a CLI for CI, and a `benchalerts` package. "The Apache Arrow project is using Conbench for Continuous Benchmarking." — [conbench repo](https://github.com/conbench/conbench)
- Maintenance signals: the PyPI `conbench` package is "deprecated; install the webapp or conbenchlegacy from git instead", and the `legacy` commands "are not guaranteed to work as documented". There are 295 open issues and no GitHub releases. The repository is not archived, but no last-commit date was visible. — [conbench repo](https://github.com/conbench/conbench)

**Other tools found**
- **dotnet/crank** (MIT) is the .NET team's benchmarking infrastructure. `crank-agent` runs jobs and the `crank` CLI enqueues them. It has built-in load tools (wrk/wrk2, Linux only; bombardier, cross-platform), stores results as JSON or in SQL Server, and is used for TechEmpower scenarios. It needs the .NET 10 SDK. — [dotnet/crank](https://github.com/dotnet/crank)
- **Horreum** is a Hyperfoil project, built on Quarkus, described as a "service for storing performance data and regression analysis". The companion **Hyperfoil** is a load driver designed to avoid coordinated omission. — [Hyperfoil GitHub org](https://github.com/hyperfoil)
- **Codespeed** is the Django web app behind speed.python.org and speed.pypy.org, an older generation of dashboards. — [speed.python.org about](https://speed.python.org/about)
- **Mozilla Perfherder** is covered in section 3 as a process reference. It is not packaged for outside adoption.

**Comparison summary (all facts from the sources above)**

| Tool | JMH/Java | Detection | Runners | Price for an OSS public repo | Status (Oct 2026) |
|---|---|---|---|---|---|
| Bencher | Native `java_jmh` adapter | t-test, z-score, IQR, log-normal, percentage and static thresholds per branch; PR start-point comparison | Any CI runner; Bencher Bare Metal (Linux x86_64 on-demand; ARM64 "coming soon"; macOS/Windows on dedicated, Enterprise only); self-hosted | Free (public projects); bare metal on Free capped at 1 concurrent job and 5-min timeout | Active; open source (Apache-2.0/MIT + Plus) |
| CodSpeed | JMH fork, walltime only, early (since 2026-05-06), JDK 21+ | Vendor's own PR regression reports | CodSpeed macro runners (16-core ARM64 Graviton bare metal) | Free incl. 600 macro min/month, 3-month history; $0.032/min extra | Active; SaaS |
| github-action-benchmark | `tool: jmh` | Previous-result ratio, default 200% | Any runner | Free | ~1.3k stars, 99 open issues |
| Nyrkiö | Via JSON/action (generic) | E-Divisive (Otava), e.g. p = 0.001, 5% minimum magnitude | Any runner; paid Nyrkiö Runners (since 2026-02-24) | Runner pricing not found | Active; platform Apache-2.0 |
| Apache Otava | Generic (CSV/JSON/DB) | E-Divisive + t-test | N/A (analysis CLI) | Free (ASF) | Incubating; 0.8.0 (June 2026) |
| conbench | Generic JSON | Lookback/alerts (`benchalerts`) | N/A | Free | Legacy; PyPI deprecated |
| crank | No (.NET-centric), but drives arbitrary load tests | None built in | Own agents | Free (MIT) | Used by .NET team |

### Inferences
- Bencher is the only tool that combines native JMH ingestion, per-branch statistical thresholds, PR comments, fork-PR support, public dashboards at no cost for public projects, and a self-hosting escape hatch. That makes it the most natural system of record for Meleys' microbenchmarks.
- Change-point detection (Otava, Nyrkiö) suits nightly and per-merge time series better than per-PR thresholds, because it accounts for noise and multiple shifts after the fact. It can run alongside Bencher: export the time series to Otava, or use Nyrkiö's action in parallel.
- CodSpeed's distinctive strength in other languages, CPU-simulation instruction counting, is not available for Java. For Meleys it reduces to a 16-core ARM64 bare-metal walltime runner. The "Cortex-A72" label matches the specs of an older first-generation Graviton bare-metal class, so absolute numbers may not represent modern x86 or Neoverse servers. The JMH fork is not on Maven Central, and the free tier keeps only 3 months of history. Treat CodSpeed as optional and experimental for Java in 2026.
- github-action-benchmark's previous-commit comparison with a 200% default threshold only catches gross regressions on noisy runners. It is useful as a free gh-pages chart, not as a regression gate.
- None of the hosted bare-metal offerings provides standard Windows or macOS performance runners. Cross-OS performance numbers therefore need GitHub-hosted VMs (noisy) or self-managed hardware.

### Gaps
- I found no current price for Nyrkiö Runners or the Nyrkiö service, no hardware specs for them, and no confirmation of a free tier for open source. The nyrkio.com pricing page is JavaScript-rendered and returned no content.
- I could not confirm the latest github-action-benchmark release or date, or conbench's last commit date, because the GitHub API was not accessible in this session and the pages hid them.
- Bencher Bare Metal launch date and the hardware behind its <2% / <1% variance claims are unpublished, and no independent measurement was found.
- I could not confirm whether CodSpeed walltime mode for Java works on GitHub-hosted or other runners, nor whether it supports macOS or Windows.
- I found no public evidence that Quarkus or the wider Red Hat ecosystem uses Horreum change detection in CI, and no current Horreum maintenance data.

## 2. How noisy are GitHub-hosted runners, which mitigations work, and what do they cost?

### Takeaway
Published measurements put typical run-to-run variation on GitHub-hosted runners at a coefficient of variation of a few percent. CodSpeed measured an average CV of 2.66%, against 0.56% on bare metal. Tails are much worse: vendors claim ±20–50%, and an A/A study saw ratios between identical commits ranging from 0.51 to 1.36. Hosted runners also change hardware and OS images underneath you. A per-PR gate on hosted runners can therefore only catch large regressions, about 7% or more at a 1% false-positive rate (CodSpeed) and as coarse as about 50% in an older A/B study (Quansight). Interleaved same-machine A/B runs with statistical tests, change-point detection, and dedicated tuned bare metal give large improvements. Bare metal costs roughly €60–260/month per Hetzner box in mid-2026, or $1–2/hour from Bencher or CodSpeed. Instruction counting is largely unavailable on hosted runners (PMU restricted) and does not fit JIT-compiled Java.

### Cited Findings

**Published noise measurements**
- **CodSpeed (2025-07-30, vendor).** Four suites (turbopack, ruff, uv, reflex) were each run 100 times on GitHub-hosted runners and 100 times on CodSpeed Macro Runners, with each run on a different machine.
  - The coefficient of variation averaged **2.66% on GitHub-hosted** runners and **0.56% on bare-metal macro runners**.
  - With a 2% regression gate, the chance of a false positive was about **45% on GitHub-hosted** runners and about 0.04% on macro runners.
  - Holding the false-positive rate at 1% needs a threshold of about **7% on GitHub-hosted** runners, versus about 1.5% on macro runners.
  — [CodSpeed blog](https://codspeed.io/blog/benchmarks-in-ci-without-noise)
- **Quansight Labs (2021-08-18).** The scikit-image asv suite (about 75 benchmarks) compared two identical commits on GitHub Actions every 6 hours for 16 days.
  - With default interleaved A/B/A/B passes in the same job, the false-positive rate was 3.7% (4/108) and ratios ranged from 0.51 to 1.36 (std 0.05).
  - Without interleaving, it rose to 9.99%, and with a single pass to 12.5% (maximum ratio 2.76).
  - Conclusion: relative benchmarking on GitHub Actions is "sensitive enough" only for regressions of about 50% or more. Runs should be on demand via a `run-benchmark` PR label.
  — [Quansight Labs (dev.to mirror)](https://dev.to/quansightlabs/is-github-actions-suitable-for-running-benchmarks-111h); original at [labs.quansight.org](https://labs.quansight.org/blog/github-actions-benchmarks)
- **Reichelt, Jung, van Hoorn (SSP 2024, Java/MooBench, JDK 21).** On GitHub Actions (4 vCPU/16 GB Azure runner), the relative standard deviation was 0.11% for the baseline and 1.97% for the "Binary Writer" configuration. The minimum detectable change was **4.41%** for sequential workloads (n = 10, α and β < 1%). A bare-metal Ryzen 7 5700G reached σ = 0.87% and Δ = 1.94%. For multi-threaded runs, bare metal had lower runtime and standard deviation; GitHub Actions slowed down faster as threads increased, and a 16-thread run was cancelled. The authors judge GitHub Actions unsuitable for measuring parallel-thread overhead. — [arXiv 2411.05491](https://arxiv.org/html/2411.05491v1)
- **Laaber, Scheuner, Leitner (Empirical Software Engineering 2019).** More than 4.5 million data points from Java and Go microbenchmarks on AWS, GCE and Azure, plus IBM bare metal, showed coefficients of variation from 0.03% to over 100%. Running test and control on the **same instances in randomized order** still detected slowdowns of 10% or less with high confidence (Wilcoxon rank-sum, bootstrapped confidence intervals). — [Chalmers research record](https://research.chalmers.se/publication/511491); [PeerJ preprint](https://peerj.com/preprints/3507)
- **Vendor and tool claims (not independently measured).** Nyrkiö: "+/- 20% to 50%" typical environmental noise ([Nyrkiö](https://blog.nyrkio.com/2026/02/24/nyrkio-runner-for-github-is-here-continuous-benchmarking-with-nanosecond-precision/)). Bencher: ">30% variance" on typical CI runners ([Bencher README](https://raw.githubusercontent.com/bencherdev/bencher/main/README.md)).

**Noise sources specific to hosted runners**
- **Different CPUs under the same label.** In 10 `ubuntu-24.04` jobs running the identical binary, 9 landed on AMD EPYC 7763 and 1 on Intel Xeon Platinum 8370C. Cache sizes differed (L2 1280 KiB vs 512 KiB) and so did CPU flags (AVX-512).
  - glibc chose different code paths (hwcaps, cache-size-based non-temporal stores). Even Callgrind instruction counts differed by about 1.6%, and cross-job RSD reached 0.505%, against 0% on a single machine.
  - Mitigations CodSpeed lists: GitHub larger runners (the same machine was observed over 100 runs, but the CPU can't be chosen), dedicated bare metal, `GLIBC_TUNABLES` (described as hacky), and logging the CPU model with every result.
  — [CodSpeed blog, 2026-03-03](https://codspeed.io/blog/unrelated-benchmark-regression)
- **Runner-image and kernel changes.** MooBench on GitHub Actions saw a performance regression in Kieker's AspectJ agent between two commits that contained no software changes. The cause was the `ubuntu-latest` image moving from 22.04 to 24.04, and specifically a Linux kernel version change. Nyrkiö detected it and it was reproduced on self-hosted VMs. — [arXiv 2510.11310](https://arxiv.org/abs/2510.11310)
- **GitHub guarantees nothing about the hardware.** The docs list only vCPU, RAM and disk; "each GitHub-hosted runner is a new virtual machine (VM)" (except `ubuntu-slim`, a 1-vCPU container on a shared VM); macOS arm64 runners cannot do nested virtualization. — [GitHub-hosted runners reference](https://docs.github.com/en/actions/reference/runners/github-hosted-runners)

**Mitigations and evidence**
- **Relative A/B in the same job, interleaved.** Interleaving cut false positives from 9.99–12.5% to 3.7% ([Quansight](https://dev.to/quansightlabs/is-github-actions-suitable-for-running-benchmarks-111h)). Same-instance randomized ordering detected slowdowns of 10% or less in the cloud ([Laaber et al.](https://research.chalmers.se/publication/511491)).
- **Randomized Multiple Interleaved Trials (RMIT).** Abedi & Brecht (ICPE 2017) showed that common comparison methods flagged two identical systems on EC2 as differing by **38% at 95% confidence**, and proposed RMIT as the fix. — [ICPE 2017 paper](https://research.spec.org/icpe_proceedings/2017/proceedings/p287.pdf)
- **Duet benchmarking.** Running both versions concurrently on the same VM and filtering synchronized interference improved accuracy 2.3×–12.5× (about 5× on average) for ScalaBench/DaCapo on the JVM, and 23.8×–82.4× for SPEC CPU 2017. — [Bulej et al., ICPE 2020, arXiv 2001.05811](https://arxiv.org/pdf/2001.05811)
- **Statistical tests and repetitions.**
  - The .NET ResultsComparer uses the Mann-Whitney U test ([dotnet/runtime #41871](https://github.com/dotnet/runtime/issues/41871)).
  - Bencher offers t-test and IQR thresholds ([Bencher thresholds](https://bencher.dev/docs/explanation/thresholds/)).
  - Change-point detection (E-Divisive) replaced fixed 10% thresholds at MongoDB, where the threshold system had produced up to 99% false positives ([Daly et al.](https://arxiv.org/abs/2003.00584)).
- **Instruction counting.**
  - rustc-perf triages mainly on instruction counts ("max rss is much more variable than instruction count") using a dedicated machine ([rustc-perf triage README](https://github.com/rust-lang/rustc-perf/blob/master/triage/README.md)).
  - On GitHub-hosted runners, `perf stat` reports some hardware counters as not available, and `perf_event_paranoid` reads 4. LLVM BOLT skips perf tests on GitHub runners, and Cachegrind is the suggested but much slower alternative ([actions/runner-images #4974](https://github.com/actions/runner-images/issues/4974)).
  - CodSpeed's CPU-simulation mode is not offered for Java; Java is walltime only ([CodSpeed Java docs](https://codspeed.io/docs/benchmarks/java)).
- **OS tuning on dedicated hosts.** MongoDB's EC2 work attributed most of its variability reduction to disabling hyperthreading, using `numactl`, and disabling CPU power-saving states. After tuning, the worst WiredTiger case stayed under a 10% min-max range. — [MongoDB blog, 2019-04-30](https://mongodb.com/blog/post/reducing-variability-performance-tests-ec2-setup-key-results)
- **Faster CPython's experience.** Linux results are "more predictable" because `pyperf tune` is applied there. — [faster-cpython/benchmarking-public](https://github.com/faster-cpython/benchmarking-public)
- **Bare-metal runners (vendor numbers).** CodSpeed measured 0.56% average CV ([CodSpeed](https://codspeed.io/blog/benchmarks-in-ci-without-noise)). Bencher claims <2% ([README](https://raw.githubusercontent.com/bencherdev/bencher/main/README.md)). Nyrkiö claims a 10× reduction ([Nyrkiö](https://blog.nyrkio.com/2026/02/24/nyrkio-runner-for-github-is-here-continuous-benchmarking-with-nanosecond-precision/)).

**Costs (as of Oct 2026)**
- **GitHub-hosted standard runners are "free and unlimited on public repositories".** Public-repo specs:
  - Linux x64 (`ubuntu-22.04/24.04/26.04`): 4 vCPU / 16 GB.
  - Linux arm64 (`ubuntu-24.04-arm` etc.): 4 vCPU / 16 GB.
  - Windows x64 (`windows-2022/2025`): 4 vCPU / 16 GB.
  - Windows arm64 (`windows-11-arm`): 4 vCPU / 16 GB.
  - macOS Intel (`macos-15-intel`, `macos-26-intel`): 4 vCPU / 14 GB.
  - macOS arm64 (`macos-14/15/26`, `macos-latest`): 3 vCPU (M1) / 7 GB.
  — [GitHub-hosted runners reference](https://docs.github.com/en/actions/reference/runners/github-hosted-runners)
- **Arm64 hosted runners** became generally available for public repositories on 2025-08-07 ([GitHub changelog](https://github.blog/changelog/2025-08-07-arm64-hosted-runners-for-public-repositories-are-now-generally-available/)) and for private repositories on 2026-01-29 ([GitHub changelog](https://github.blog/changelog/2026-01-29-arm64-standard-runners-are-now-available-in-private-repositories/)).
- **GitHub pricing changes (private repos).** Hosted-runner prices fell by up to 39% on 2026-01-01; Linux 2-core went from $0.008 to $0.006/min. A $0.002/min "platform" fee for self-hosted runners in private repos, announced for 2026-03-01, was postponed and is reportedly still not in effect. Secondary sources only; I did not read GitHub's changelog directly. — [Socket.dev](https://socket.dev/blog/github-actions-pricing-whiplash); [SAMexpert](https://samexpert.com/github-actions-pricing-backlash-2026/)
- **Hetzner dedicated servers.** Official prices for new orders from 2026-06-15, in EUR/month excluding VAT and IPv4:
  - AX42-1: €97.30 (+€49 setup).
  - AX102-1: €257.30 (+€129).
  - AX162-1: €612.30.
  - EX63-1: €147.30.
  - Limited ("-LTD") variants: AX41-1-LTD €57.30, AX42-1-LTD €77.30, EX44-1-LTD €57.30, AX102-1-LTD €157.30.
  — [Hetzner price adjustment](https://docs.hetzner.com/general/infrastructure-and-availability/price-adjustment/)
- **Earlier 2026 Hetzner increases (secondary sources only).**
  - A 2026-04-01 increase was reportedly 30–37% on cloud servers and 2–21% on dedicated servers ([agentdeals issue #2226](https://github.com/robhunter/agentdeals/issues/2226)).
  - A search summary cited AX42 going from €47.30 to €57.30, but I could not tie that figure to a primary page.
  - Other reports say Hetzner raised prices three times in 2026 ([webhosting.today](https://webhosting.today/2026/05/29/hetzner-has-now-raised-prices-three-times-in-2026-this-one-is-different/); [HostingJournalist](https://hostingjournalist.com/news/hetzner-to-significantly-raise-cloud-and-server-prices-in-2026)).
  - These secondary figures partly conflict. Use Hetzner's official table above for planning.
- **Bencher Bare Metal.** Included on the Free plan with 1 concurrent job and a 5-minute timeout. On-demand runners cost $1.00/hour on Pro, which is $100/month and up. — [Bencher pricing](https://bencher.dev/pricing/)
- **CodSpeed macro runners.** The first 600 minutes/month are free (10 h); after that the price is $0.032/min (= $1.92/h). — [CodSpeed pricing](https://codspeed.io/pricing)

### Inferences
- On hosted runners, a blocking per-PR gate needs thresholds of roughly 7–10% or more to keep false positives near 1% (CodSpeed, Laaber). That is too coarse for a project whose vision is "fastest". Hosted runners should give advisory signals and catch catastrophic regressions, not serve as the source of truth.
- For JVM microbenchmarks, instruction counting is a poor fit. Hosted runners don't expose the PMU, Valgrind/Cachegrind slows the JIT dramatically, and CodSpeed doesn't offer simulation for Java. JIT compilation timing and GC also make instruction counts nondeterministic; this is my inference, not a sourced measurement. Interleaved A/B walltime plus bare metal is the practical route.
- Pin runner labels (e.g. `ubuntu-24.04`, not `ubuntu-latest`). Record the CPU model, kernel, runner-image version and JDK build with every data point. Both CodSpeed's CPU-heterogeneity finding and MooBench's kernel-change finding show that environment drift can look like regressions.
- Multi-threaded and network benchmarks are exactly where hosted runners do worst (Reichelt et al.). HTTP server load tests, which are inherently multi-threaded and use network and loopback, belong on dedicated hardware.
- A single Hetzner AX42-class box (about €97/month after June 2026) running only trusted main-branch jobs gives about 720 machine-hours/month. That is far cheaper per hour than Bencher on-demand ($1/h) or CodSpeed ($1.92/h) once usage exceeds roughly 50–100 h/month, but the project then owns tuning, maintenance and security.

### Gaps
- I found no published noise measurements for GitHub-hosted **Windows** or **macOS** runners, or for `windows-11-arm` / `ubuntu-24.04-arm` specifically. All the numbers above are for Linux x64 hosted runners.
- I could not open the full text of Kozlov (2025), a comparative study of GitHub-hosted, self-hosted and Kubernetes runners cited by the MooBench paper.
- I did not research OVH, Scaleway, AWS EC2 Mac or MacStadium pricing. Bare-metal macOS (Mac mini) and Windows options and costs remain open, as does whether Hetzner currently sells ARM (Ampere) dedicated servers at listed 2026 prices.
- No independent (non-vendor) measurement of Bencher Bare Metal or Nyrkiö Runner noise was found.

## 3. How do established projects track performance continuously?

### Takeaway
Mature projects separate a fast, noisy PR signal from authoritative runs on dedicated hardware. They detect changes statistically: IQR fences at Rust, σ deviation at ASP.NET, E-Divisive change points at MongoDB, t-test and 2% thresholds at Mozilla. Every one of them relies on a human triage rotation that labels or auto-files issues and asks authors to justify or revert. In the Java HTTP world, Jetty runs a dedicated six-machine load test with latency assertions, and Quarkus publishes results from a performance lab. I found no public continuous regression tracking for Netty or OpenJDK.

### Cited Findings

**Rust compiler (perf.rust-lang.org / rustc-perf)**
- Significance is decided with IQR fencing on historical data: a change is significant if `result > Q3 + 3 × IQR`, and the lower fence is ignored. The same doc's intro mentions "1.5 times", which conflicts with the formula. Magnitude buckets run from very small to very large and rate both how far a change exceeds the threshold and the percentage change. Relevance rules (high, medium, low) decide what gets reported: PR try runs report "somewhat relevant" results, while triage reports include only "definitely relevant" ones. — [rustc-perf comparison analysis](https://github.com/rust-lang/rustc-perf/blob/master/docs/comparison-analysis.md)
- Triage process:
  - A named roster triages merged PRs, ideally between Monday evening and Tuesday afternoon (North America) so the results make it into This Week in Rust.
  - Triagers use `/perf/triage?start=` reports, compare pages and binary-size reports, and run `@rust-timer build $SHA` to isolate PRs inside rollups.
  - Authors are asked to respond; for a big regression the triager may request a revert. Authors are reminded to use `@bors rollup=never` for perf-sensitive PRs, and summaries go to Zulip.
  — [rustc-perf triage README](https://github.com/rust-lang/rustc-perf/blob/master/triage/README.md)
- Since July 2021, regressing PRs get the `perf-regression` label. The author justifies the change (adding `perf-regression-triaged`) or fixes it. — [rustc-perf triage 2021-07-06](https://github.com/rust-lang/rustc-perf/blob/master/triage/2021-07-06.md)

**ASP.NET Core / .NET**
- PR runs are triggered by a `/benchmark <benchmark> <profile> <component>` comment, which the "Crank Pull Request Bot" handles; only .NET team members can trigger runs on official infrastructure.
  - Benchmarks include the TechEmpower plaintext, json and fortunes scenarios, plus yarp and mvcjsoninput2k.
  - Profiles: `aspnet-perf-lin`/`-win` (Intel, 12 cores), `aspnet-citrine-lin`/`-win` (Intel, 28 cores), `aspnet-citrine-ampere` (80 cores) and `aspnet-citrine-amd` (48 cores).
  - The bot posts separate result comments for the PR and the base branch.
  - A PowerBI dashboard (aka.ms/aspnet/benchmarks) shows the exact crank command line for each point, so regressions can be confirmed by re-running two points.
  — [aspnetcore docs/Benchmarks.md](https://github.com/dotnet/aspnetcore/blob/main/docs/Benchmarks.md)
- The `pr-benchmarks` bot auto-files "Perf regression: <scenario>" issues labelled `perf-regression`. Each issue shows old and new RPS, the percentage change, the deviation in σ and the dependency version changes. For example, issue #57468 (2024-08-22) reported −1.92%, −3.63σ on "Plaintext Endpoint… Intel Windows" and was closed 3 days later. — [dotnet/aspnetcore #57468](https://github.com/dotnet/aspnetcore/issues/57468)
- In a 2026 example, dotnet/runtime #127484 reports a .NET 11 ARM64 throughput regression at 16+ cores (−14.7% on the JSON Platform benchmark), visible on the KPI dashboard's "Cobalt" environments. — [dotnet/runtime #127484](https://github.com/dotnet/runtime/issues/127484)
- For runtime microbenchmarks, the performance auto-filer bot opens issues in `dotnet/perf-autofiling-issues`, using an "Edge Detector" plus a 5% rule to classify changes ([example #18434](https://github.com/dotnet/perf-autofiling-issues/issues/18434)). The bot was evaluated before .NET 5, and the ResultsComparer uses Mann-Whitney U ([dotnet/runtime #41871](https://github.com/dotnet/runtime/issues/41871)).

**MongoDB**
- The Evergreen CI system runs thousands of benchmarks, most commonly every 2 or every 24 hours. Worst-case run-to-run variation is under 10% for most tests, but some sensitive tests fluctuate 20% or more. — [Daly et al., ICPE 2020](https://arxiv.org/abs/2003.00584)
- The earlier threshold system compared each result with the previous run, the run from a week earlier and the last stable release, using X = 10% by default. It produced up to 99% false positives "depending on how you count", missed small regressions, and flagged the wrong commits. It was replaced by E-Divisive means. — [Daly et al.](https://arxiv.org/abs/2003.00584)
- Triage under the new system:
  - A rotating "Build Baron" works through unprocessed change points, ranked by "hazard level" (the log of the before/after mean ratio).
  - The Build Baron backfills missing commits to isolate the culprit, and acknowledges or hides change points.
  - Each real change gets a Jira ticket. Canary workloads detect noise from the test system itself.
  - Change points typically appear 3–5 days after the change. Recomputation over up to about 500 points takes under 10 s.
  — [Daly et al.](https://arxiv.org/abs/2003.00584)
- The ICPE 2020 slides report that useful notifications rose from about 1% to 67%. This figure comes from a search summary of the slides, which I did not read directly. — [Daly ICPE 2020 slides](https://icpe2020.spec.org/slides/Daly_Use.pdf)
- A follow-up paper describes a "virtuous cycle" of better coverage, faster and more accurate signals, and more investment. — [Daly, ICPE 2021, arXiv 2101.10231](https://arxiv.org/pdf/2101.10231)

**Python (CPython)**
- speed.python.org runs Codespeed with the pyperformance suite on x86 and AArch64. Its main machine, nicknamed "the beast", was donated by HP. — [speed.python.org about](https://speed.python.org/about)
- Faster CPython's `benchmarking-public` covers Linux x86_64, Linux aarch64, Windows amd64 and x86, and macOS arm64 (M1 Mac mini).
  - Runs are triggered by `workflow_dispatch` (`gh workflow run benchmark.yml -f fork=… -f ref=…`) using bench_runner. There are weekly runs of upstream main.
  - Raw pyperformance JSON, comparison tables, violin and memory plots, and longitudinal SVGs are committed to the repo.
  - Headline numbers are geometric means (the experimental HPT score was dropped from plots on 2024-11-26), with 0.1% resolution.
  - The latest results visible were dated 2025-11-01, and coverage was uneven, so current activity is unclear.
  — [faster-cpython/benchmarking-public](https://github.com/faster-cpython/benchmarking-public)

**Java ecosystem**
- **Jetty.** "The Jetty Performance Effort" (Ludovic Orban, 2021, updated 2024) describes the setup:
  - Six dedicated physical machines: a server (16-core i9-7960X), one probe at 100 req/s and four loaders at 60K req/s each (about 240K req/s, keeping the server around 50% CPU).
  - Runs are a 1-minute warmup followed by 3 minutes of recording. The Jetty Load Generator drives the load, and a Cluster Orchestrator runs everything over SSH from a JUnit test.
  - Async-profiler produces flame graphs for every machine on every run.
  - Coverage spans HTTP/1.1 and HTTP/2, each clear-text and TLS. Performance assertions run on the 10.0.x and 11.0.x branches.
  - Result: "99% of its own HTTPS requests were served in less than 1 millisecond".
  — [Webtide: The Jetty Performance Effort](https://webtide.com/?p=4541)
- **Netty.** The `netty-microbench` JMH module is skipped in normal builds (it needs `skipTests=false`), and the wiki page looks dated ([Netty microbenchmarks wiki](https://netty.io/wiki/microbenchmarks.html)). The 4.2 branch's GitHub workflows include none named for benchmarks; `ci-verify-load.yml` exists but its contents are unknown ([netty workflows](https://github.com/netty/netty/tree/4.2/.github/workflows)).
- **Quarkus.** A new benchmark suite (`quarkusio/spring-quarkus-perf-comparison`) feeds results "from the Quarkus performance lab" into `quarkusio/benchmarks`. It charts throughput, boot plus first-response time, RSS, build time, a composite score and density by RSS, published at quarkus.io/benchmarks. Quarkus Insights #244 (2026-04-20) covered patterns and anti-patterns, including an app and load generator sharing CPU cores. — [quarkusio/benchmarks](https://github.com/quarkusio/benchmarks); [Holly Cummins: How we benchmarked Quarkus](https://hollycummins.com/how-we-benchmarked-quarkus/)
- **OpenJDK.**
  - The JMH-based microbenchmark suite (JEP 230) has lived in the main repository since JDK 12 ([InfoQ](https://infoq.com/news/2019/04/jep-230-microbenchmark-suite)).
  - Regressions appear as JBS bugs, e.g. JMH-JavacSteady regressions in 9-b31 ([JDK-8060429](https://bugs.openjdk.org/browse/JDK-8060429)).
  - In a 2016 jmh-dev thread, LMAX said it tried and failed to build a pass/fail JMH regression job because inter-run noise caused frequent false positives even on tuned systems ([jmh-dev](https://mail.openjdk.org/pipermail/jmh-dev/2016-November/002426.html)).
- **Elasticsearch / Lucene.** Elasticsearch nightly Rally benchmarks run against the latest main snapshot, on environments recreated daily on AWS/GCP VMs ([Elasticsearch nightly benchmarks](https://elasticsearch-benchmarks.elastic.co/)). Elastic moved most macro-benchmarks from bare metal to cloud but kept bare metal for workloads that need minimal platform variability ([Elastic blog](https://www.elastic.co/blog/benchmark-driven-optimizations-scalability-elasticsearch-8)). Lucene's nightly benchmarks have been maintained by Mike McCandless on his own time and hardware for about 13 years ([Elastic Search Labs](https://www.elastic.co/search-labs/de/blog/apache-lucene-9-9-search-speedups); [Lucene nightly](https://blunders.io/lucene-bench)).

**Other process references**
- **Mozilla Perfherder** has generated alerts since January 2016. It looks for a sustained shift using the original point plus 12 later points, and shows a t-test-based confidence. — [Mozilla wiki: Performance sheriffing](https://wiki.mozilla.org/Performance_sheriffing)
  - Thresholds: 2% for Talos, Browsertime and build metrics; 0.25% for AWSY; 100 KB for installer size. — [Firefox source docs: Perf sheriffing](https://firefox-source-docs.mozilla.org/testing/perfdocs/perf-sheriffing.html)
  - Sheriffs triage daily, retrigger noisy jobs, find the culprit and file bugs with a needinfo flag for the author. The response target is 3 days. — [Mozilla perf sheriff newsletter, 2020](https://blog.mozilla.org/performance/2020/10/15/performance-sheriff-newsletter-september-2020/)
- **Confluent** runs Otava over nightly Kafka performance results, and Netflix reported a similar change-point system in 2022. — [Ingo, arXiv 2505.06758](https://arxiv.org/html/2505.06758v1)

### Inferences
- Common pattern:
  1. Run authoritative measurements on owned or dedicated hardware, not shared CI.
  2. Use statistical detection: IQR, σ, t-test or change points rather than fixed percentages.
  3. Collect PR signals on demand or with restricted triggers (Rust's `@rust-timer`, ASP.NET's `/benchmark`, Quansight's label).
  4. Have a named human rotation with SLAs (Rust weekly, MongoDB Build Baron, Mozilla 3-day target).
  5. Auto-file issues with enough context to act on: σ, environment and dependency diffs, and the reproduce command.
- ASP.NET runs TechEmpower scenarios (plaintext, json, fortunes) on its own dedicated machines via crank, and gates through its own bot and dashboard. This suggests TechEmpower is valuable as a source of workload definitions, not as the regression-detection loop. The loop must run on project-controlled infrastructure at commit granularity.
- Jetty's design is directly reusable for Meleys: a fixed-rate probe that measures latency under a fixed background load, a separate load generator, flame graphs on every run, and latency assertions. It measures tail latency at a defined load instead of only maximum throughput, which matches Meleys' p99+ goals.
- Change-point detection has a built-in delay (MongoDB: 3–5 days), so it complements per-PR checks rather than replacing them.

### Gaps
- Netty's and OpenJDK's internal or continuous performance processes are not publicly documented, as far as I found; Oracle's internal performance infrastructure was not visible.
- I could not find Quarkus's lab hardware, schedule or regression-detection method (Horreum or otherwise) in the fetched pages. The slides and video weren't read.
- I don't know Jetty's current schedule, threshold values or whether the suite has evolved for Jetty 12. The article was updated in 2024 and gives no thresholds.
- The 1% → 67% useful-notification figure for MongoDB comes from a search summary of the slides; I did not read it in the paper.
- rustc-perf's collector hardware, and how many historical points it uses for the IQR, were not stated in the fetched docs.

## 4. What is a practical design for Linux, macOS and Windows (including ARM64)? What runs per PR versus nightly or per release, and how are results published?

### Takeaway
Use a three-tier design. The tiers are separated by how much noise each can tolerate:
- **Every PR (hosted runners, all six OS/arch combinations):** compile and smoke-run the JMH suite, plus an interleaved base-versus-head A/B on Linux that only flags large regressions.
- **Every merge to main (dedicated tuned bare metal, trusted code only):** the curated JMH suite tracked in Bencher with t-test thresholds, publicly visible.
- **Nightly or weekly, and per release (dedicated multi-machine rigs):** full HTTP/1.1, HTTP/2 and HTTP/3 load tests measuring throughput, p99/p99.9 latency and RSS. Linux x64 and ARM64 run nightly; Windows and macOS run weekly or per release on owned machines. These series are analysed with change-point detection (Otava or Nyrkiö) and published as dashboards plus raw JSON.

### Cited Findings (constraints that shape the design)
- Hosted runner labels and specs for public repos are listed in section 2. Pinned versions exist: `ubuntu-24.04`, `ubuntu-24.04-arm`, `windows-2025`, `windows-11-arm`, `macos-15`/`macos-26` (M1, 3 vCPU, 7 GB) and `macos-15-intel`/`macos-26-intel`. — [GitHub-hosted runners reference](https://docs.github.com/en/actions/reference/runners/github-hosted-runners)
- "Self-hosted runners should almost never be used for public repositories on GitHub", because "any user can open pull requests against the repository and compromise the environment". JIT runners run at most one job, but reusing the hardware can still expose information. — [GitHub Actions secure use](https://docs.github.com/en/actions/reference/security/secure-use)
- Bencher handles fork PRs safely with the `pull_request` + `workflow_run` split, and states that Bencher Bare Metal supports fork PRs. — [Bencher GitHub Actions guide](https://bencher.dev/docs/how-to/github-actions/)
- Bencher's free bare-metal tier allows 1 concurrent job with a 5-minute timeout, on Linux x86_64 only. — [Bencher pricing](https://bencher.dev/pricing/)
- CodSpeed's free tier gives 600 macro minutes/month on ARM64 Linux, keeps only 3 months of history, and supports Java in walltime mode only. — [CodSpeed pricing](https://codspeed.io/pricing); [CodSpeed Java docs](https://codspeed.io/docs/benchmarks/java)
- On hosted runners, a 1% false-positive rate requires about a 7% gate ([CodSpeed](https://codspeed.io/blog/benchmarks-in-ci-without-noise)). Interleaved A/B in the same job beats separate runs ([Quansight](https://dev.to/quansightlabs/is-github-actions-suitable-for-running-benchmarks-111h)). Same-instance randomized comparisons detect slowdowns of 10% or less in the cloud ([Laaber et al.](https://research.chalmers.se/publication/511491)).
- Multi-threaded workloads are the weakest case for GitHub Actions. — [Reichelt et al.](https://arxiv.org/html/2411.05491v1)
- Runner-image updates (Ubuntu 22.04 → 24.04 kernel) and CPU heterogeneity cause false regressions. — [arXiv 2510.11310](https://arxiv.org/abs/2510.11310); [CodSpeed](https://codspeed.io/blog/unrelated-benchmark-regression)
- Publishing precedents:
  - ASP.NET publishes a public PowerBI dashboard with the reproduce command for each point ([aspnetcore Benchmarks.md](https://github.com/dotnet/aspnetcore/blob/main/docs/Benchmarks.md)).
  - Faster CPython commits raw JSON and plots to a public results repo ([benchmarking-public](https://github.com/faster-cpython/benchmarking-public)).
  - Quarkus publishes charts at quarkus.io/benchmarks from a results repo ([quarkusio/benchmarks](https://github.com/quarkusio/benchmarks)).
  - github-action-benchmark builds gh-pages charts ([repo](https://github.com/benchmark-action/github-action-benchmark)).
  - Bencher hosts public project perf pages on its free plan for public projects ([Bencher pricing](https://bencher.dev/pricing/)).
- Change-point settings used in practice: p = 0.001 and a 5% minimum magnitude in Nyrkiö for MooBench ([arXiv 2510.11310](https://arxiv.org/abs/2510.11310)). Bencher's documented default for main is a t-test with max sample size 64 and upper boundary 0.99 ([Bencher](https://bencher.dev/docs/how-to/github-actions/)).

### Inferences (recommended setup for Meleys)

**Tier 0: every PR, GitHub-hosted, all platforms (free)**
- Matrix: `ubuntu-24.04`, `ubuntu-24.04-arm`, `windows-2025`, `windows-11-arm`, `macos-15` (or `macos-26`) and `macos-15-intel`. Pin labels; never use `-latest`. Test the newest GA JDK plus an early-access build.
- Purpose: confirm that benchmarks compile and run on every OS, and catch catastrophic regressions or pathologies such as 2× slowdowns, allocation explosions or OS-specific transport bugs.
- Use one fork and short iterations. The result is a non-blocking job summary.

**Tier 1: every PR, relative A/B on Linux x64 hosted runners (free)**
- Build the base and head JARs in the same job and run them interleaved, alternating JMH forks in randomized order (RMIT or duet style).
- Compare with Mann-Whitney U or bootstrap confidence intervals. Post a PR comment; block only on regressions above roughly 10%, a level the literature suggests is achievable at a low false-positive rate.
- Optionally, add an opt-in, label-triggered run on Bencher Bare Metal (free tier, curated benchmarks under 5 minutes, x86 only) or a CodSpeed macro runner (ARM64) for finer resolution.

**Tier 2: every push to main, dedicated bare metal (about €60–260/month per box)**
- Use one or two Hetzner-class servers (e.g. AX42/AX102 or EX44-LTD) registered as runners. Restrict them to `push` on main, schedules and maintainer-only `workflow_dispatch`, and never to fork `pull_request` code.
- Tune the hosts: disable SMT and turbo, pin the governor, isolate cores and use `numactl` (per MongoDB's experience).
- Run the curated JMH suite and track it in Bencher. Use the testbed name to encode the host, OS, arch and JDK; use a `t_test` threshold; and alert through GitHub issues or Checks.
- Add a Linux ARM64 bare-metal box (Ampere, or Bencher Dedicated or CodSpeed) when budget allows, since GitHub-hosted ARM64 runners are VMs.

**Tier 3: nightly or weekly, and per release (dedicated multi-machine rigs)**
- Run HTTP load tests with a separate load-generator machine; never share cores with the server under test (the Jetty and Quarkus lessons).
- Measure:
  - Fixed-rate probe latency (p50, p99, p99.9 and max) under a fixed background load, using coordinated-omission-safe generators.
  - Maximum throughput.
  - RSS at idle and under load.
- Cover HTTP/1.1, HTTP/2 and HTTP/3 (the latter clear and TLS where applicable) for Linux x64 and Linux ARM64 nightly.
- Windows and macOS need their own owned hardware, for example a bare-metal Windows box and an Apple-silicon Mac mini as self-hosted runners restricted to trusted triggers. Run them weekly and per release, because hosted macOS (3 vCPU, 7 GB VM) and Windows runners have no published noise data and are VMs.
- Feed the nightly time series into Apache Otava (self-run) or Nyrkiö (hosted), with p ≈ 0.001 and a minimum magnitude of about 3–5%. Their change points then open issues.

**Publishing**
- Use Bencher's public project pages for the microbenchmarks.
- Create a `meleys-benchmarks` results repo with raw JSON, full environment metadata (CPU model, microcode/kernel, OS image, JDK vendor and build, JVM flags, load-generator version and command line) and generated charts. Serve it via GitHub Pages and link it from the website, following the Faster CPython and Quarkus patterns.
- Every published point should carry its reproduce command, as ASP.NET does.

**Triage process**
- Run a weekly perf-triage rotation (as Rust does), with labels `perf-regression` and `perf-regression-triaged`.
- Auto-file issues containing the σ deviation, the base and head commits, and environment and dependency diffs (as ASP.NET does).
- Use canary or A/A benchmarks to detect infrastructure noise (as MongoDB does).
- Set a response SLA, for example 3 days (as Mozilla does).

**Rough monthly cost**
- Tiers 0 and 1: $0, since hosted runners are free for public repos.
- Tier 2: about €60–260 for one Hetzner box.
- Tier 3: Linux client/server pairs cost about €115–515 for two boxes; Windows and macOS hardware is a one-off purchase or colocation, not researched.
- Optional extras: Bencher Pro at $100/month for unlimited bare-metal timeouts, and CodSpeed Pro at $15/user/month if more than 600 macro minutes or more than 3 months of history are needed.

### Gaps
- I found no published guidance or measurements on running JVM load tests on GitHub-hosted Windows or macOS runners, or on Apple-silicon bare metal for CI. Mac mini and Windows bare-metal hosting costs were not researched.
- I did not establish whether Bencher Dedicated or Custom macOS and Windows runners are offered to open-source projects, or at what price.
- I found no source quantifying the false-positive rates of interleaved JMH A/B runs specifically on GitHub-hosted runners. The ~10% figure is extrapolated from Laaber et al. (cloud VMs, Java and Go) and CodSpeed (non-Java suites).
- I did not check whether the TechEmpower continuous-run infrastructure could be reused for per-commit tracking. TechEmpower is covered by another researcher.
