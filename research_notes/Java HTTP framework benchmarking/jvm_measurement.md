# JVM-specific performance measurement for a Java HTTP framework (memory footprint, GC/JIT/AOT effects, diagnostics, JMH, OS differences): state as of 2026-10-21

Scope note: these notes cover measurement methodology on the JVM only. Load-generator methodology (coordinated omission, wrk2/Hyperfoil and so on), HTTP/2 and HTTP/3 tooling, and CI regression services are covered by other researchers. They appear here only where a JVM source mentions them.

## Q1. How should the memory footprint of a JVM server be measured and reported? (RSS vs heap used/committed, NMT, direct buffers, metaspace, thread stacks, per-connection overhead, memory under load, minimum heap via -Xmx sweeps, containers; how Quarkus/Micronaut/Helidon do it; pitfalls)

### Takeaway
The headline number should be the OS-level resident set size (RSS) of the whole process, sampled with `ps`/`pmap`/`/proc` (never `docker stats`). Measure it at defined lifecycle points: after the first request, under a fixed load, and idle after load. Explain it with the used and committed heap and with Native Memory Tracking (NMT) categories, but take those from separate diagnostic runs, because NMT costs 5–10% in performance. Run memory and GC comparisons across heap sizes stated as multiples of the minimum heap the workload needs. Fix every ergonomic input, because each one moves RSS by tens of percent on its own: `-Xms`/`-Xmx`, CPU count, the container limit, the transparent huge pages (THP) mode, compact headers, the GC, and the allocator settings.

### Cited Findings

#### What RSS contains, and how Quarkus, Micronaut and Helidon measure it
- Quarkus measures RSS, not heap. The heap is "only a small part of the overall problem"; the total also includes class metadata, thread stacks, compiled code and GC structures. Heap is set by `-Xms`/`-Xmx`, and the other native allocations also count toward RSS — [Quarkus "Measuring performance"](https://quarkus.io/guides/performance-measure)
- The Quarkus performance lab records RSS **after the first successful request** with `pmap -x $PID | grep total | awk '{print $4}'` (KB, converted to MiB). Other Linux options are `ps -o pid,rss,command -p <pid>` and `pmap -x <pid>`, whose columns are Kbytes, RSS and Dirty. macOS uses `ps x -o pid,rss,command -p <PID>` (KB). For Docker, use `docker top <id> -o pid,rss,args` or `ps` on the host — [Quarkus](https://quarkus.io/guides/performance-measure)
- Quarkus pitfalls:
  - Do not use `docker stats` or tools derived from it, such as ctop. They measure only "a subset of the in-use resident pages".
  - Do not report only private memory. That figure misleads in Kubernetes/OpenShift, where the full RSS decides whether a container is killed.
  - NMT "will add a 5-10% performance overhead", so do not publish RSS from NMT-enabled runs.

  — [Quarkus](https://quarkus.io/guides/performance-measure)
- Quarkus measures startup as **Time to First Request (TTFR)**, not the framework's self-reported startup time, "because some frameworks use aggressive lazy initialization". The self-reported figure "only measures internal initialization". The script takes a timestamp, then spins on `curl` with no sleep until it gets HTTP 200. macOS needs GNU `gdate` for `%N` — [Quarkus](https://quarkus.io/guides/performance-measure)
- Quarkus benchmark JVM flags are `-XX:ActiveProcessorCount=4 -Xms512m -Xmx512m`. `ActiveProcessorCount` keeps thread-pool sizing consistent, and equal `-Xms`/`-Xmx` "eliminate heap resizing during the measurement". Processes are pinned with `taskset`. The guide does not cover Windows — [Quarkus](https://quarkus.io/guides/performance-measure)
- Quarkus warns that the extra flags it passes to `native-image` can change comparisons: `-H:FallbackThreshold=0`, `-H:-SpawnIsolates` and `-H:-UseServiceLoaderFeature`. Check the full native-image command line in the build log — [Quarkus](https://quarkus.io/guides/performance-measure)
- Micronaut's Docker-image guide includes a time-to-first-request script that polls with curl. It supports Docker, fat-JAR, native and CRaC-checkpoint artifacts. No RSS procedure was found in Micronaut's docs — [Micronaut guide](https://guides.micronaut.io/latest/micronaut-docker-image-gradle-java.html)
- Micronaut's 2020 comparison with Spring Boot used the startup times the frameworks report themselves, best of five sequential starts, plus a separate time-to-first-response script — [Micronaut blog 2020](https://micronaut.io/blog/2020-04-28-performance-comparison-spring-boot-micronaut.html)
- Helidon's site lists a memory footprint of about 70 MB (native), 110 MB (jlink JVM) and 117 MB (plain JVM), and startup of 0.08 s, 1.24 s and 1.8 s. It gives no version, workload or method, so treat these as marketing numbers. They come from a search-result summary, not a full fetch — [helidon.io](https://helidon.io/)
- A third-party comparison of Helidon 4 SE, Quarkus 3 and Micronaut 4 used "RSS after 60s steady-state traffic" at 10,000 concurrent connections as its memory metric. Its methodology was not inspected — [Java Code Geeks](https://www.javacodegeeks.com/?p=142003)
- In March 2026, Oracle's Project Leyden team presented startup comparisons using Helidon, Micronaut, Quarkus, Spring Boot and Spring PetClinic, as elapsed time normalized to the mainline JDK (= 1000) — [JavaOne 2026 Leyden deck](https://jcp.org/aboutJava/communityprocess/ec-public/materials/2026-03-20/JavaOne-2026-Leyden-v1.pdf)

#### Breaking down native memory with NMT
- Enable NMT with `-XX:NativeMemoryTracking=summary|detail`. Read it with `jcmd <pid> VM.native_memory [summary|detail|baseline|summary.diff|detail.diff]`, or print it at exit with `-XX:+PrintNMTStatistics`. Categories include Java Heap, Class (metaspace), Thread, Code, GC, Compiler, Internal, Symbol, Arena Chunk and NMT itself — [Shipilëv, JVM Anatomy Quark #12](https://shipilev.net/jvm/anatomy-quarks/12-native-memory-tracking/); [Quarkus](https://quarkus.io/guides/performance-measure)
- Reserved and committed memory differ greatly. Example: Class is "reserved=1066093KB, committed=14189KB" — [Quark #12](https://shipilev.net/jvm/anatomy-quarks/12-native-memory-tracking/)
- A hello-world with a 16 MB heap commits about 75 MB in total. Java Heap is 16 MB, Class 14.2 MB, GC 11.0 MB, Internal 9.4 MB and Code 2.6 MB. The author calls this "certainly unexpected" — [Quark #12](https://shipilev.net/jvm/anatomy-quarks/12-native-memory-tracking/)
- GC native structures cost "around 3-4% from the configured heap size". A 16 GB heap gives GC "committed=623339KB" (about 609 MB). Native overhead therefore matters most at small heaps — [Quark #12](https://shipilev.net/jvm/anatomy-quarks/12-native-memory-tracking/)
- Footprint levers measured in the same article:
  - `-XX:+UseSerialGC` cut total committed from 75.2 MB to 37.6 MB.
  - `-Xss256k` cut the Thread category from 11.4 MB to 7.5 MB.
  - Fewer JIT threads cut Thread further, to 4.4 MB.
  - Combined tuning reached about 21 MB in total, about 8.5 MB above the heap.

  — [Quark #12](https://shipilev.net/jvm/anatomy-quarks/12-native-memory-tracking/)
- ZGC "has a fixed overhead 3% of the heap size, requiring more native memory than G1" — [Netflix Tech Blog](https://netflixtechblog.com/bending-pause-times-to-your-will-with-generational-zgc-256629c9386b) (the original returned HTTP 403; text read via [mirror](https://noise.getoto.net/2024/03/06/bending-pause-times-to-your-will-with-generational-zgc/))
- glibc keeps memory after `free()`. Because it is freed memory, "it won't appear in NMT; it is just a part of RSS". JDK 18 added `jcmd <pid> System.trim_native_heap`, which calls `malloc_trim`. JDK-8293114 added periodic trimming (`-XX:TrimNativeHeapInterval`, in ms; 0 disables it), logged under the `trimnative` tag, e.g. "RSS+Swap: 1192M->1191M". The final default and the release it shipped in were not confirmed from this review thread — [hotspot-dev RFR JDK-8293114](https://mail.openjdk.org/pipermail/hotspot-dev/2023-July/076443.html)

#### Heap used vs committed vs resident; expansion and uncommit
- Footprint should be measured in each lifecycle phase (startup, load, idle, after a Full GC) because GC heuristics differ between them. Specific observations:
  - Under load, G1 grew to "around 3/4 of the max heap" and gave memory back on explicit or Full GCs.
  - Shenandoah uncommits periodically even without a GC request.
  - Idle uncommit for G1 was expected from JEP 346 periodic GCs (JDK 12).
  - Linux does not commit physical memory until first touch, so memory the JVM has committed may not yet appear in RSS.

  — [Shipilëv, Quark #21 "Heap Uncommit"](https://shipilev.net/jvm/anatomy-quarks/21-heap-uncommit/) (2019 article; ZGC uncommit was "planned" at the time)
- `-XX:+AlwaysPreTouch` moves page-fault cost to startup. On a 4 TB heap, pre-touching took "13m58s" without THP and "2m14s" with THP — [Shipilëv, Quark #2 "Transparent Huge Pages"](https://shipilev.net/jvm/anatomy-quarks/2-transparent-huge-pages/)
- For ZGC, Netflix recommends equal minimum and maximum heap, `-XX:+UseTransparentHugePages` and `-XX:+AlwaysPreTouch`. "Except in a couple of cases, there's been no need to lower the maximum heap size to allow for more headroom" — [Netflix Tech Blog](https://netflixtechblog.com/bending-pause-times-to-your-will-with-generational-zgc-256629c9386b) ([mirror](https://noise.getoto.net/2024/03/06/bending-pause-times-to-your-will-with-generational-zgc/))
- The DaCapo authors' "Rethinking Java Performance Analysis" (ASPLOS 2025) makes these recommendations:
  - H1: "Garbage collectors should be evaluated across a range of heap sizes".
  - H2: "Heap sizes should be expressed in terms of multiples of the minimum heap size in which a baseline collector can run that workload".
  - Choose heap sizes "in a distribution that gives more resolution to small heap sizes", up to about 6× the minimum.

  — [Blackburn et al., ASPLOS 2025 (DaCapo Chopin)](https://www.dacapobench.org/assets/pdf/dacapo-asplos-2025-with-appendix.pdf)
- The same paper warns that a `-Xmx`-controlled minimum heap "reflects the workload's peak memory usage, not its average usage". It suggests that "a metric which reflected the 'area under the memory use curve' might better reflect the net memory footprint". DaCapo's minimum heaps range from 5 MB (avrora) to 681 MB (h2) at default sizes, and up to 20 GB at vlarge size — [DaCapo ASPLOS 2025](https://www.dacapobench.org/assets/pdf/dacapo-asplos-2025-with-appendix.pdf)
- "Because ZGC does not support compressed pointers, care should be taken when comparing it with the other collectors" — [DaCapo ASPLOS 2025](https://www.dacapobench.org/assets/pdf/dacapo-asplos-2025-with-appendix.pdf). In GraalVM Native Image, compressed references are on by default and cap the heap at 32 GB — [GraalVM Memory Management](https://www.graalvm.org/latest/reference-manual/native-image/optimizations-and-performance/MemoryManagement/)

#### Ergonomics, containers and default heap sizes (major sources of distortion)
- Before JDK 27, the JVM chose Serial instead of G1 in "constrained environments": "a single CPU or less than 1792 MB of physical memory". JDK 27 (JEP 523) always selects G1 when no GC is specified, and claims G1's native memory use is now comparable to Serial's — [JEP 523](https://openjdk.org/jeps/523)
- The container-aware JVM's default `MaxRAMPercentage` is 25% of the cgroup memory limit, for example a 1 GiB heap in a 4 GiB container. The CPU count is derived from the cgroup quota divided by the period, rounded up, and can be overridden with `-XX:ActiveProcessorCount`. These come from secondary sources, not official JDK docs — [braindetox (2026)](https://braindetox.kr/en/posts/jvm_container_resource_limits.html). An OpenJDK RFC discusses how the JVM should handle container memory limits — [hotspot-dev RFC 8285277](https://mail.openjdk.org/pipermail/hotspot-dev/2022-April/059647.html)
- GraalVM Native Image with Serial GC (the default) sets the maximum heap to **80% of physical memory**; with G1 it is 25%. Override with `-Xmx` at run time or `-R:MaxHeapSize` at build time. GC needs extra memory while collecting, so RSS can rise temporarily — [GraalVM Memory Management](https://www.graalvm.org/latest/reference-manual/native-image/optimizations-and-performance/MemoryManagement/)
- JDK 28 (in development) targets JEP 546, ZGC adaptive heap sizing:
  - It is opt-in with `-XX:+ZAdaptiveHeapSizing`, and `-XX:ZGCIntensity` defaults to 5.
  - The default maximum heap becomes "100% of physical memory, minus a small reserve", and the heap grows or shrinks to hit a GC CPU target.
  - The JEP warns: "There is thus a risk that, for a given application, ZGC will use more memory than it did in past releases".

  — [JEP 546](https://openjdk.org/jeps/546)
- JDK 28 also targets JEP 545, under which the ZGC initial heap would drop to 2 MB from 1/64 of RAM. Expansion pre-touches memory concurrently, and memory is managed in 2 MB units, with automatic large pages on Linux 6.1+. Results on Spring PetClinic (32 GB machine, ramping to 30k req/s):
  - Startup: 5.84 s, the same as JDK 25 with a small heap (5.86 s) and 38% lower than JDK 25 with a 20 GB pre-touched heap (9.45 s).
  - Warm-up: down from about 80 s (JDK 25, small heap) to about 10 s.
  - Without concurrent pre-touch, warm-up took 34 s; without large pages, 18 s.

  Tuning `-Xms`/`-Xmx` and `AlwaysPreTouch` therefore changes ZGC startup and warm-up results a lot — [JEP 545](https://openjdk.org/jeps/545)

#### Object layout: compact object headers
- JEP 519 made compact object headers (96 → 64 bits) a product feature in JDK 25 (`-XX:+UseCompactObjectHeaders`, no unlock flag needed). JEP 534 makes them the default in JDK 27; opt out with `-XX:-UseCompactObjectHeaders`. Reported results:
  - SPECjbb2015: "22% less heap space and 8% less CPU time" in one setting, and GC count "reduced by 15%" with G1 and Parallel in another.
  - A parallel JSON parser benchmark ran "in 10% less time".

  The JEPs take these numbers from external talks and posts — [JEP 519](https://openjdk.org/jeps/519); [JEP 534](https://openjdk.org/jeps/534)

#### Transparent huge pages (THP)
- `-XX:+UseTransparentHugePages` uses `madvise` for the heap, and `-XX:+UseLargePages` uses hugetlbfs on Linux; both default to false. With sysfs set to `madvise`, a TLB-bound microbenchmark ran about 12% faster (32.5 → 28.7 ns/op at 100M elements). The costs are "memory overheads (because you will allocate an entire large page for something small)" and defrag-related latency spikes — [Quark #2](https://shipilev.net/jvm/anatomy-quarks/2-transparent-huge-pages/)
- With THP, thread stacks can inflate RSS. JDK-8303215, "Make thread stacks not use huge pages", was fixed in JDK 21 and backported to 17.0.9 and 11.0.21. It links JDK-8312182, "THPs cause huge RSS due to thread start timing issue" — [JDK-8303215](https://bugs.openjdk.org/browse/JDK-8303215) (via search results)
- Many Linux distributions set `shmem_enabled` to `never`, which "silently prevents ZGC from using huge pages". Netflix uses `enabled=madvise`, `shmem_enabled=advise`, `defrag=defer` and `khugepaged/defrag=1` — [Netflix Tech Blog](https://netflixtechblog.com/bending-pause-times-to-your-will-with-generational-zgc-256629c9386b) ([mirror](https://noise.getoto.net/2024/03/06/bending-pause-times-to-your-will-with-generational-zgc/))

#### Thread stacks: platform vs virtual threads
- "The stacks of virtual threads are stored in Java's garbage-collected heap as stack chunk objects". They grow and shrink, are not GC roots, and "the G1 GC does not support humongous stack chunk objects". Virtual threads "should never be pooled", and thread locals should be used "only after careful consideration". The heap use and GC activity of thread-per-request versus async code "should be roughly similar" — [JEP 444](https://openjdk.org/jeps/444)
- Platform-thread stacks show up in NMT's Thread category and shrink with `-Xss` — [Quark #12](https://shipilev.net/jvm/anatomy-quarks/12-native-memory-tracking/)

#### Direct and off-heap buffers
- Netty's JMH harness sets `-XX:MaxDirectMemorySize=768m` together with `-Xms768m -Xmx768m`, so the direct-memory cap is explicit rather than left to the default — [Netty AbstractMicrobenchmark (4.2)](https://github.com/netty/netty/blob/4.2/microbench/src/main/java/io/netty/microbench/util/AbstractMicrobenchmark.java)
- Netty 4.2 changed the default ByteBuf allocator to `adaptive`. "Most workloads will observe reduced memory usage", but it "will perform worse for some". Revert with `-Dio.netty.allocator.type=pooled`. For memory regressions, record a JFR flight recording with the "Netty Allocator Events.jfc" profile — [Netty 4.2 migration guide](https://netty.io/wiki/netty-4.2-migration-guide.html)

#### Per-connection overhead
- No primary measurement of JVM per-connection memory at 10k/100k idle connections was found. One low-quality secondary source says idle-connection cost is dominated by per-connection state, file descriptors, socket buffers and heartbeat traffic — [cloudspress (weak source)](https://www.cloudspress.com/?p=774087)

### Inferences
- **Report a footprint vector, not a single number.** For each run, record:
  - (a) RSS, as the median and maximum of 1 s samples of `/proc/<pid>/status` VmRSS, or `pmap` total;
  - (b) heap used after a forced full GC, i.e. the live set;
  - (c) committed heap;
  - (d) the NMT summary by category, from a separate NMT run, since NMT has overhead;
  - (e) direct buffer pool usage (JDK `BufferPoolMXBean` "direct"/"mapped"; a standard API, not separately verified here) plus the framework allocator's own metrics;
  - (f) thread count, platform and virtual;
  - (g) the GC/heap configuration in force (`-XX:+PrintFlagsFinal`).
- **Define phases.** Use these phases:
  - P0: TTFR plus RSS right after the first request (the Quarkus convention).
  - P1: idle after warm-up and an explicit GC.
  - P2: under a fixed, below-saturation request rate, reporting time-averaged RSS (the "area under curve" idea from DaCapo) and peak RSS.
  - P3: at saturation.
  - P4: idle again after load, to show uncommit behaviour and native-heap retention, with and without `System.trim_native_heap`.
- **Per-connection cost.** Hold N idle keep-alive connections, for N in {0, 1k, 10k, 100k}, and report the slope (bytes per connection), separately for heap (live set), native (NMT) and RSS. Kernel socket memory (e.g. `/proc/net/sockstat`) is not part of process RSS and should be listed separately. Repeat for HTTP/2 connections with idle streams and for HTTP/3/QUIC connections. Compare platform-thread-per-connection with virtual-thread-per-connection models: virtual-thread stacks land in the heap, so they show up in the live set, not NMT Thread.
- **Minimum heap that sustains a target load.** Binary-search `-Xmx` at a fixed request rate, keeping the GC and its other settings fixed. Count a heap size as passing when the latency SLO holds (for example p99 and p99.9 below a threshold), with no OutOfMemoryError, ZGC allocation stalls or Full GCs. Then report throughput, latency and RSS at 1.25×, 1.5×, 2× and 3× of that minimum, following DaCapo H1/H2. This turns "small footprint" into a falsifiable claim and exposes frameworks that look fast only because they have a large heap.
- **Pin and record every ergonomic input.** These include: `-Xms`/`-Xmx` (or `MaxRAMPercentage`), `-XX:ActiveProcessorCount`, the container memory and CPU limits, the explicit GC (because JDK 27 changed the default in small containers), `UseCompactObjectHeaders` (because the default flipped in JDK 27), the THP sysfs values, `AlwaysPreTouch`, the native-image `-Xmx` (80% of RAM by default with Serial), and the Netty allocator type for Netty-based competitors.
- Run footprint tests in two modes, so a framework cannot win simply by leaving heap ergonomics at default:
  - "Fixed heap" (`-Xms`=`-Xmx`, no pre-touch) for apples-to-apples comparison.
  - "Default ergonomics in a fixed container", for what users actually get.
- RSS with `AlwaysPreTouch` and `-Xms`=`-Xmx` mostly reflects the configured heap, not the framework. Do not use that configuration for footprint claims.

### Gaps
- No primary, reproducible per-connection memory data for Netty, Vert.x, Jetty or Helidon at 10k/100k idle connections was found. Meleys will need to produce its own.
- Helidon's measurement method behind its published RSS and startup numbers was not found, and no Micronaut RSS procedure was found.
- The default value and first release of `-XX:TrimNativeHeapInterval` were not confirmed from final JDK docs.
- The default `MaxDirectMemorySize` behaviour and the NMT category for direct ByteBuffers were not verified this session.
- Current ZGC and G1 uncommit behaviour (JEP 346, ZGC uncommit) was not re-verified. Shipilëv's article dates from 2019.
- The `MaxRAMPercentage`/cgroup details come from secondary blogs. Official JDK documentation was not fetched.
- No guidance was found on measuring JVM RSS on Windows (Working Set vs Private Bytes), and the Quarkus guide explicitly omits Windows.

## Q2. How do JVM choices change benchmark results, and how should they be standardized for fair comparisons? (GC choice, JIT warm-up/steady state, Leyden AOT cache and CRaC, GraalVM native image, compact object headers, virtual threads incl. JEP 491, FFM for native I/O)

### Takeaway
On current JDKs, defaults changed in JDK 27: G1 is now used in small containers and compact object headers are on. AOT caches (JDK 24–26), and soon AOT code in JDK 28, change startup and warm-up by 2–5×. GC choice moves throughput, CPU cost, latency and footprint in different directions. A fair comparison therefore has to fix the JDK build and every one of these knobs explicitly, report total CPU alongside throughput, and treat cold start, warm-up and steady state as separate results. Native-image and CRaC results go in their own categories.

### Cited Findings

#### JDK release state as of 2026-10-21
- **JDK 27 is the current feature release.** It reached GA on 15 September 2026. Its JEPs:
  - 523 (G1 default in all environments)
  - 534 (compact object headers by default)
  - 536 (JFR in-process data redaction)
  - 527 (post-quantum hybrid key exchange for TLS 1.3)
  - Previews: 531 (Lazy Constants, third preview), 532 (primitive patterns, fifth preview), 533 (Structured Concurrency, seventh preview), 538 (PEM, third preview)
  - Incubator: 537 (Vector API, twelfth incubator)

  — [JDK 27 project page](https://openjdk.org/projects/jdk/27/)
- **JDK 26** reached GA on 17 March 2026. Its JEPs include 516 (AOT object caching with any GC), 517 (HTTP/3 for the HTTP Client API) and 522 (G1 throughput via reduced synchronization). Previews and incubators: Structured Concurrency (sixth), Lazy Constants (second), Vector API (eleventh incubator) — [JDK 26 project page](https://openjdk.org/projects/jdk/26/)
- **JDK 28 is in development.** No schedule was posted yet. JEPs targeted so far:
  - 401 (Value Objects, Preview)
  - 535 (generational Shenandoah by default)
  - 539 (strict field initialization in the JVM, Preview)
  - 540 (Simple JSON API, Incubator)
  - 541 (deprecate the macOS/x64 port)
  - 542 (PEM)
  - 543 (Structured Concurrency, with no preview label)
  - 544 (Ahead-of-Time Code Compilation)
  - 545 (faster ZGC startup and warm-up)
  - 546 (adaptive heap sizing for ZGC)

  — [JDK 28 project page](https://openjdk.org/projects/jdk/28/)
- JDK 25 delivered:
  - JEP 519 (compact headers as a product feature) — [JEP 519](https://openjdk.org/jeps/519)
  - JEP 514/515 (AOT command-line ergonomics and AOT method profiling) — [JEP 514](https://openjdk.org/jeps/514), [JEP 515](https://openjdk.org/jeps/515)
  - JEP 521 (generational Shenandoah as a product feature) — [JEP 521](https://openjdk.org/jeps/521)
  - JEP 509 (JFR CPU-time profiling, experimental, Linux only) — [JEP 509](https://openjdk.org/jeps/509)

#### Garbage collectors: trade-offs and defaults
- Oracle's JDK 25 GC tuning guide gives this selection guidance:
  - Serial: for "small data sets (up to approximately 100 MB)" or single-processor machines.
  - Parallel: when "peak application performance is the first priority" and pauses of one second or more are acceptable.
  - G1: when "response time is more important than overall throughput".
  - ZGC: "max pause times under a millisecond, but at the cost of some throughput", for heaps "from a few hundred megabytes to 16TB".

  — [Oracle GC Tuning Guide, JDK 25](https://docs.oracle.com/en/java/javase/25/gctuning/available-collectors.html)
- JDK 27 makes G1 the default everywhere. Per the JEP, G1 is "competitive with Serial at all heap sizes", its maximum throughput is "close to that of Serial" after JEP 522, and its native memory use is comparable — [JEP 523](https://openjdk.org/jeps/523)
- Since JDK 24, ZGC is generational only. `-XX:+UseZGC` gives generational ZGC, and `ZGenerational` is obsolete — [JEP 490](https://openjdk.org/jeps/490) (via search summary)
- Generational Shenandoah became a product feature in JDK 25 (`-XX:+UseShenandoahGC -XX:ShenandoahGCMode=generational`, no unlock needed); the default stayed single-generation — [JEP 521](https://openjdk.org/jeps/521). JDK 28 targets making it the default. `-XX:ShenandoahGCMode=satb` keeps the old mode, deprecated, and "some workloads that are non-generational by nature could see a slight slowdown" — [JEP 535](https://openjdk.org/jeps/535)
- Epsilon (`-XX:+UseEpsilonGC`, experimental, JDK 11) exists for "differential performance analysis for other, real GCs". It also isolates barrier costs: "the choice of GC means choosing the set of GC barriers that the workload has to use" — [JEP 318](https://openjdk.org/jeps/318)
- On GC cost (DaCapo Chopin, OpenJDK 21, 22 benchmarks, Lower Bound Overhead (LBO) method):
  - "The CPU overhead of garbage collection is 15% in the best case".
  - Total overhead rises with collector age: Serial (1998), Parallel, G1, Shenandoah, ZGC (2018).
  - Recommendations O1/O2: report GC overheads, "both wall clock and total CPU".

  — [DaCapo ASPLOS 2025](https://www.dacapobench.org/assets/pdf/dacapo-asplos-2025-with-appendix.pdf)
- Cai et al. (OpenJDK 17, five production GCs) found "at least 7-82% more wall-clock time and 6-92% more CPU cycles relative to the baseline cost". Concurrency and generous provisioning "can mask these costs". Low-pause collectors are "significantly more expensive" and "sometimes give worse application latency than stop-the-world collectors" — [Cai, Blackburn, Bond, Maas, "Distilling the Real Cost of Production Garbage Collectors"](https://arxiv.org/abs/2112.07880)
- **Netflix's production results conflict with that picture.** "More than half of our critical streaming video services are now running on JDK 21 with Generational ZGC". The pauses "are usually under one millisecond". "ZGC improves both average and P99 latencies with equal or better CPU utilization when compared to G1". The worst-case 36% CPU penalty of non-generational ZGC became "a nearly 10% improvement" with generational ZGC. One large precompute workload saw a 6–8% throughput improvement; the mirror text was ambiguous about whether that came from ZGC or from Parallel GC for batch jobs — [Netflix Tech Blog](https://netflixtechblog.com/bending-pause-times-to-your-will-with-generational-zgc-256629c9386b) ([mirror](https://noise.getoto.net/2024/03/06/bending-pause-times-to-your-will-with-generational-zgc/))

#### JIT warm-up and steady state
- JVMs are "notoriously good at profile-guided optimizations". In a shared process, "different tests can mix their profiles together", so JMH forks each test by default. `@Fork(0)` runs in the same JVM — [JMHSample_12_Forking](https://github.com/openjdk/jmh/blob/master/jmh-samples/src/main/java/org/openjdk/jmh/samples/JMHSample_12_Forking.java)
- A Leyden slide (borrowed from Tobias Hartmann, 2020) notes on warm-up curves: "Plot implies convergence, this isn't always the case". Using training-run profiles too early can cause "more uncommon traps ... resulting in deoptimization and recompilation" and overloaded compile queues — [JavaOne 2026 Leyden deck](https://jcp.org/aboutJava/communityprocess/ec-public/materials/2026-03-20/JavaOne-2026-Leyden-v1.pdf)
- DaCapo's P1 recommendation: base the decision to time the first iteration (JIT and class loading included) or a warmed iteration on the claims being made. DaCapo found the 5th iteration well warmed for default sizes. The paper also cites Mytkowicz et al. showing that "the length of strings in environment variables can confound findings" — [DaCapo ASPLOS 2025](https://www.dacapobench.org/assets/pdf/dacapo-asplos-2025-with-appendix.pdf)

#### Project Leyden AOT cache (JDK 24–26 shipped; AOT code in JDK 28)
- JEP 483 (JDK 24) loads and links classes from the AOT cache. Spring PetClinic started 41% faster in production, with about 21,000 classes already loaded and linked at startup — as cited in [JEP 516](https://openjdk.org/jeps/516)
- JEP 514 (JDK 25) creates the cache in one step with `-XX:AOTCacheOutput=app.aot` and uses it with `-XX:AOTCache=app.aot`. `JDK_AOT_VM_OPTIONS` passes options to the assembly step. The one-step mode needs roughly double the heap: `-Xms4g -Xmx4g` needs about 8 GB — [JEP 514](https://openjdk.org/jeps/514)
- JEP 515 (JDK 25) adds method profiles to the cache. A short Stream program went from 90 ms to 73 ms (19%), and the cache grew by 250 KB (2.5%). Cached profiles "don't lock in behavior": the JIT keeps profiling. The gain assumes the training run is representative — [JEP 515](https://openjdk.org/jeps/515)
- JEP 516 (JDK 26) makes AOT object caching work with any GC, including ZGC, by streaming objects in a background thread instead of mapping them:
  - The cache is streamed when training used ZGC, `-XX:-CompressedOops` or a heap over 32 GB. Streaming suits cold starts but needs a spare CPU core.
  - The JDK ships two baseline caches, which are used when the application provides none.

  — [JEP 516](https://openjdk.org/jeps/516)
- Leyden startup numbers in the JavaOne 2026 deck: elapsed time normalized to mainline = 1000, lower is better, read from the charts.

  | JDK and configuration | Helidon | Micronaut | Quarkus | Spring Boot | PetClinic |
  |---|---|---|---|---|---|
  | JDK 24 AOTCache | 347 | 366 | 380 | 382 | 586 |
  | Leyden EA AOTCache | 255 | 321 | 284 | 287 | 376 |
  | JDK 25 AOTCache | 335 | 398 | 412 | 334 | 554 |
  | Leyden EA2 (with AOT code) | 278 | 351 | 367 | 253 | 308 |

  The deck cites the openjdk/leyden README as its source — [JavaOne 2026 Leyden deck](https://jcp.org/aboutJava/communityprocess/ec-public/materials/2026-03-20/JavaOne-2026-Leyden-v1.pdf)
- Leyden prerequisites and roadmap:
  - The cache "requires deterministic classloaders" and "consistent classpaths (i.e. same jars)".
  - Hot methods get both "AOT Preload" and fully optimized "A" code, and "even the best AOT code for the hottest methods should eventually be replaced by new JIT code".
  - Roadmap items: portability across micro-architectures ("Portability is not just code. It's also heap sizes, garbage collectors, etc."), caching for user-defined classloaders, and iterative training.

  — [JavaOne 2026 Leyden deck](https://jcp.org/aboutJava/communityprocess/ec-public/materials/2026-03-20/JavaOne-2026-Leyden-v1.pdf)
- JEP 544, AOT code compilation, is reported as integrated in JDK 28 (not yet GA).
  - Results (JDK 28 EA, two-core Linux/x64): on five framework benchmarks the cache alone cut startup "around 50% to 70%", and with AOT code "around 65% to 80%". On a javac warm-up benchmark the first iteration improved about 30% from the cache alone and about 75% in total with AOT code.
  - Restrictions: the same CPU architecture and feature set (e.g. AVX-512), and **the same GC** in training and production. Only x64 and AArch64 are supported. On a mismatch, HotSpot warns and skips the AOT code.
  - Flags: `-XX:AOTMode=required` fails fast, and `-XX:+UnlockDiagnosticVMOptions -XX:-AOTCodeCaching` disables AOT code.
  - Code quality: AOT code cannot fold `static final` fields into constants and adds class-initialization checks, so JIT code can replace AOT code that "fails to age well".

  — [JEP 544](https://openjdk.org/jeps/544)

#### CRaC
- CRaC "takes a snapshot of the Java process (checkpoint) when it is fully warmed up". Open files and sockets block a checkpoint ("The checkpoint cannot succeed if the program has an open file handle"). Spring, Micronaut and Quarkus support it — [Azul CRaC docs](https://docs.azul.com/core/crac/crac-introduction)
- CRaC is not part of mainline OpenJDK. The full mechanism relies on Linux CRIU. Azul's comparison lists Linux and marks Windows and macOS "n/a", and it offers a simulated checkpoint/restore engine for development. Ubuntu 26.04 packages `openjdk-25-crac`. A portable mode without CRIU was a proof of concept in 2024, with slower restores — [Azul CRaC comparison](https://docs.azul.com/crac/azul-zulu-zing-crac-compared); [Ubuntu package](https://packages.ubuntu.com/ja/source/resolute/java/openjdk-25-crac); [crac-dev 2024](https://mail.openjdk.org/pipermail/crac-dev/2024-July/001889.html)

#### GraalVM Native Image
- Serial is the default GC ("optimized for low memory footprint and small Java heap sizes"). G1 (`--gc=G1`) is available only on Linux AMD64/AArch64 and "not available in GraalVM Community Edition". Epsilon is also available. The default maximum heap is 80% of RAM with Serial and 25% with G1 — [GraalVM Memory Management](https://www.graalvm.org/latest/reference-manual/native-image/optimizations-and-performance/MemoryManagement/)
- Quarkus's extra native-image flags can affect native-vs-native comparisons — [Quarkus](https://quarkus.io/guides/performance-measure)

#### Virtual threads
- JEP 491 (JDK 24) means `synchronized` blocks and `Object.wait()` no longer pin carriers. Pinning remains for native frames, including FFM upcalls that block, for class loading, and for class initializers. The `jdk.VirtualThreadPinned` JFR event now reports the reason and the carrier. `jdk.tracePinnedThreads` was removed. Raising scheduler parallelism is not a substitute; the scheduler's default cap is 256 platform threads. A listed risk is that releasing a monitor may now queue a virtual thread to the scheduler, which is currently less efficient than unparking a platform thread. That is a reason to benchmark lock-heavy handlers on both thread models — [JEP 491](https://openjdk.org/jeps/491)
- The scheduler is a FIFO work-stealing ForkJoinPool, tunable with `jdk.virtualThreadScheduler.parallelism` — [JEP 444](https://openjdk.org/jeps/444)

#### FFM API and native I/O
- JUring binds io_uring through FFM and requires Java 22+, Linux 5.1+ and liburing. Its own file-I/O benchmarks claim up to 426% faster 4 KB reads than FileChannel with registered files. Socket support was still planned as of January 2025, and no independent verification was found — [JUring](https://github.com/davidtos/juring); [Phoronix](https://www.phoronix.com/news/JUring-IO_uring-Java)
- Netty's JNI io_uring transport graduated from incubator in 4.2.0 (April 2025). It needs Java 9+ and is configured as `MultiThreadIoEventLoopGroup` + `IoUringIoHandler` — [Netty 4.2.0 release](https://netty.io/news/2025/04/03/4-2-0.html); [Netty 4.2 migration guide](https://netty.io/wiki/netty-4.2-migration-guide.html)
- The FFM API has been final since JDK 22 — [JEP 454](https://openjdk.org/jeps/454) (not re-fetched; consistent with JUring's "Java 22 or higher" requirement)

### Inferences
- **Standardize on a "normalized" configuration** and publish it with every result:
  - The same JDK vendor and build string, e.g. JDK 25.0.x LTS and JDK 27.0.x, run as separate matrices.
  - An explicit `-XX:+UseG1GC` (or another GC); never rely on defaults, because of JEP 523.
  - Explicit `-XX:±UseCompactObjectHeaders`, because of JEP 534.
  - Explicit `-Xms`/`-Xmx` and `-XX:ActiveProcessorCount`, with CPU pinning or a cpuset.
  - An explicit THP mode and AOT-cache policy.
  - `-XX:+PrintFlagsFinal -version` output archived for each run.
- Also run each framework's **"vendor-recommended" configuration** as a second series, so readers can see how much of a gap comes from configuration.
- **Run a GC matrix rather than one GC.** Use G1 (the default), generational ZGC (latency), Parallel (throughput) and Serial (footprint), all at the same heap multiple. Report throughput, latency percentiles, total process CPU (task-clock or cgroup `cpu.stat`) and RSS for each. DaCapo and Netflix disagree on whether ZGC costs more CPU, which is exactly why Meleys should measure that CPU cost on its own workload.
- **Separate three results.** Report (1) TTFR/cold start, (2) the warm-up curve (throughput and p99 per 1–5 s bucket from t=0), and (3) steady state after a fixed warm-up with a stability check. A framework that wins steady state but takes 60 s to warm up should not be presented as uniformly faster.
- **Treat AOT as a variable, not a constant.**
  - Report results without an AOT cache (still using the JDK's default CDS archive) and with an AOT cache trained on a defined, scripted training run that is identical for all frameworks.
  - On JDK 28, the training-run GC must match the production GC (JEP 544), and so must the CPU flags. A cache built on a CI machine with AVX-512 silently loses AOT code on older hardware.
  - Peak throughput with AOT code could differ until JIT replacement happens, because `static final` fields are not folded. Measure steady state long enough.
- **Native image and CRaC are separate categories.** Compare native against native, recording the GC (Serial by default), `-Xmx` (default 80% of RAM) and PGO usage. CRaC is Linux-only and needs socket close/reopen, so its startup numbers are not comparable to cold-JVM TTFR.
- **Virtual threads**: if Meleys offers virtual-thread handlers, benchmark on JDK 24 or later only (JEP 491). Enable the `jdk.VirtualThreadPinned` JFR event in diagnostic runs. Any FFM io_uring path that calls back into Java and blocks will pin, so keep upcalls non-blocking.
- **The JDK 28 changes** (JEP 535, 544, 545, 546) will shift ZGC and Shenandoah footprint, startup and warm-up numbers again in about March 2027. That date is inferred from the six-month cadence; no JDK 28 schedule was posted. Plan to re-baseline then.

### Gaps
- The JDK 28 GA date and schedule were not yet published on the project page.
- Whether JEP 545's changes apply by default or only with `-XX:+ZAdaptiveHeapSizing` was unclear from the fetched summary.
- JEP 541 was reported as "Closed / Delivered" for release 28 even though JDK 28 is not GA. The status label may mean "integrated".
- The attribution of Netflix's 6–8% throughput figure (ZGC or Parallel) is ambiguous in the text read.
- JEP 483's exact restrictions (same JDK build, OS/arch, classpath prefix) were not fetched directly. Only the deck's paraphrase and JEP 544 were read.
- JEP 518 (cooperative sampling) and JEP 520 (method timing/tracing) were not fetched.
- GraalVM PGO and Oracle-GraalVM-vs-CE throughput figures were not fetched.
- Azul Platform Prime (Falcon JIT, C4) and OpenJ9 effects were not researched.
- No independent network-socket benchmarks of FFM-based io_uring against NIO or Netty io_uring were found.
- JDK 25's LTS status comes from the assignment brief and was not re-verified.

## Q3. Which diagnostics explain latency and allocation behaviour? (jHiccup, GC and safepoint logs, JFR, async-profiler CPU/alloc/wall, allocation per request as a metric, JMC, perf with JIT symbol maps)

### Takeaway
Pair the latency histogram from the load generator with JVM-side evidence. Use jHiccup with its idle control JVM to separate platform stalls from JVM or application stalls. Use GC and safepoint unified logs with uptime decorators to correlate pauses, including time-to-safepoint. Use a low-overhead JFR recording for runtime events, including virtual-thread pinning and, on Linux, JDK 25's CPU-time sampler. Run async-profiler or perf with perf maps in separate profiling runs. Track allocated bytes per request as a first-class regression metric, using JMH's `gc.alloc.rate.norm` for components and `getTotalThreadAllocatedBytes()` deltas for the server.

### Cited Findings

#### jHiccup
- jHiccup "logs and records platform 'hiccups'", meaning JVM stalls and "any OS or hardware platform noise that may cause the running application to not be continuously runnable". It runs as `java -javaagent:jHiccup.jar="-d 2000 -i 1000 -l hiccuplog -c" ...`, where `-d` is the start delay, `-i` the interval (default 5 s) and `-l` the log name. It can also attach with `jHiccup -p <pid>`, or wrap the launch command — [jHiccup README](https://github.com/giltene/jHiccup)
- `-c` starts a concurrent "control process" that logs "hiccup information of an idle workload running on a separate jvm", written to a `c.`-prefixed log. Output is HdrHistogram `.hlog`, about 200–400 bytes per interval, processed with `jHiccupLogProcessor` or HistogramLogAnalyzer. No quantitative overhead figure is given — [jHiccup README](https://github.com/giltene/jHiccup)

#### GC and safepoint logs
- Time-to-safepoint (TTSP) is separate from pause duration, and "the time spent reaching that safepoint is not charged to the GC pause timer". On JDK 9+ the suggested logging is `-Xlog:safepoint=info:stdout:uptime,level,tags`, because "without uptime, correlating safepoint events with your application logs and metrics becomes nearly impossible". Logging can be changed at runtime with `jcmd <pid> VM.log`. JDK 8 used `-XX:+PrintSafepointStatistics` — [Java Code Geeks safepoint guide](https://www.javacodegeeks.com/?p=143584)
- JDK Mission Control has an automated-analysis rule that compares time to reach a safepoint with time spent in it — [JMC-5509](https://bugs.openjdk.org/browse/JMC-5509)
- Moving to generational ZGC may change log output and the data available from the management APIs — [JEP 490](https://openjdk.org/jeps/490)
- The unified logging framework behind `-Xlog` comes from JEP 158 — [JEP 158](https://openjdk.org/jeps/158) (not re-fetched)
- DaCapo recommendation L1: "report user-experienced latency, not weak proxies such as GC pauses". L2: report distributions or CDFs. DaCapo's "metered latency" spreads request start times over a 100 ms window, so that a pause also delays the requests queued behind it — [DaCapo ASPLOS 2025](https://www.dacapobench.org/assets/pdf/dacapo-asplos-2025-with-appendix.pdf)

#### JFR
- JEP 509 (JDK 25) is experimental and **Linux only**. Its `jdk.CPUTimeSample` event is enabled with `-XX:StartFlightRecording=jdk.CPUTimeSample#enabled=true,filename=profile.jfr`, with a default throttle of 500/s; `jdk.CPUTimeSamplesLost` reports drops. It fixes three problems of the old execution sampler, which sampled only threads running Java code, missed samples silently, and covered only a subset of threads. It depends on JEP 518 (cooperative sampling) and marks samples that may be safepoint-biased — [JEP 509](https://openjdk.org/jeps/509)
- JFR virtual-thread events are `jdk.VirtualThreadStart`/`End`, `jdk.VirtualThreadPinned` and `jdk.VirtualThreadSubmitFailed`. JSON thread dumps come from `jcmd <pid> Thread.dump_to_file -format=json`, which "does not pause the application" — [JEP 444](https://openjdk.org/jeps/444); [JEP 491](https://openjdk.org/jeps/491)
- JDK 27 adds JFR in-process data redaction (JEP 536; only the title was verified), which is relevant before sharing recordings from benchmark runs — [JDK 27](https://openjdk.org/projects/jdk/27/)
- For allocator regressions, Netty recommends a JFR recording with its "Netty Allocator Events.jfc" profile — [Netty 4.2 migration guide](https://netty.io/wiki/netty-4.2-migration-guide.html)

#### async-profiler
- async-profiler 4.6 is the current stable release. Official builds cover Linux x64/arm64 and macOS x64/arm64, with community ports for other Linux architectures. **Windows is not listed.** Usage: `asprof -d 30 -f flamegraph.html <PID>`. Output includes flame graphs, JFR, a converter and heatmaps — [async-profiler README](https://github.com/async-profiler/async-profiler)
- Modes:
  - CPU (perf_events plus AsyncGetCallTrace) shows Java, native, JVM and kernel frames without `-XX:+PreserveFramePointer`.
  - Allocation (`--alloc 500k`) uses TLAB-driven sampling: new-TLAB and slow-path allocations, with no bytecode instrumentation.
  - Wall-clock (`-e wall -t -i 50ms`) samples all threads in any state and is "most useful in per-thread mode".
  - Lock (`-e lock`) covers Java monitors, and `--nativelock` covers pthread locks.
  - nativemem tracks malloc/free leaks via `jfrconv --leak`.
  - `--all` "is not recommended to enable this in production".
  - Attaching at runtime to instrument Java methods may deoptimize compiled code; starting as an agent avoids this.

  — [async-profiler Profiling Modes](https://github.com/async-profiler/async-profiler/blob/master/docs/ProfilingModes.md)

#### perf with JIT symbol maps
- `jcmd <pid> Compiler.perfmap` (JDK 17, JDK-8254723, Linux only) writes `/tmp/perf-<pid>.map` for `perf`. `-XX:+DumpPerfMapAtExit` writes it at exit. `-XX:+PreserveFramePointer` is needed for readable Java call stacks in perf. Before JDK 17, perf-map-agent was needed — [JDK-8254723](https://bugs.openjdk.org/browse/JDK-8254723); [BellSoft perf guide](https://docs.bell-sw.com/liberica-jdk/10/debugging/perf-monitor-java-performance/); [jcmd man page (OpenJDK 27)](https://man.archlinux.org/man/jcmd-openjdk27.1.en.txt)

#### Allocation rate per request
- JMH `-prof gc` reports `gc.alloc.rate` (per second) and `gc.alloc.rate.norm` (per operation). Its sample found about 2048 B/op, which exposed Integer autoboxing. Guidance:
  - "always cross-reference 'gc.alloc' and 'gc.churn' values".
  - "non-normalized counters are dependent on benchmark performance!", so use the normalized ones for per-op behaviour.

  — [JMHSample_35_Profilers](https://github.com/openjdk/jmh/blob/master/jmh-samples/src/main/java/org/openjdk/jmh/samples/JMHSample_35_Profilers.java)
- `com.sun.management.ThreadMXBean.getTotalThreadAllocatedBytes()` (JDK 21, JDK-8304074) returns an approximation of heap bytes allocated by all threads since JVM start, including threads that have exited. JMH's `-prof gc` switched to it because the old per-thread method missed "those threads most of the time" — [JMH RFR CODETOOLS-7903484](https://mail.openjdk.org/pipermail/jmh-dev/2023-June/003682.html); [ThreadMXBean (JDK 25 docs mirror)](https://docs.w3cub.com/openjdk~25/jdk.management/com/sun/management/threadmxbean)
- async-profiler's alloc mode attributes heap pressure to allocation sites — [async-profiler Profiling Modes](https://github.com/async-profiler/async-profiler/blob/master/docs/ProfilingModes.md)

### Inferences
- **Diagnostic bundle for every benchmark run**, cheap enough to keep on:
  - The load generator's HdrHistogram.
  - jHiccup with `-c` (the control JVM); a stall in both logs points to the platform, and one only in the server's log points to the JVM or application.
  - `-Xlog:gc*,safepoint:file=...:uptime,time,level,tags`.
  - A JFR recording with the default profile and the `jdk.VirtualThreadPinned` event.
  - `getTotalThreadAllocatedBytes()` and request counters sampled every second, giving **bytes allocated per request**, tracked over time like throughput.
  - RSS samples.
- **Separate profiling runs** (results not published as performance numbers): async-profiler CPU, alloc, wall (`-t`) and lock on Linux/macOS; JFR CPU-time sampling on Linux; perf with `Compiler.perfmap` (plus `-XX:+PreserveFramePointer`, which itself costs some performance) for kernel and network-stack time.
- Explain p99 and p99.9 outliers by aligning, on the uptime axis: GC pauses plus TTSP from safepoint logs, jHiccup hiccups, JIT deoptimization and compilation bursts (JFR compilation events), and the load generator's latency timeline. If outliers persist with all of these clean, look at the kernel or network (perf, softirq).
- Bytes per request (B/req) is the easiest metric to make deterministic in CI and usually leads throughput regressions. Track it for each HTTP path (H1 plaintext, H1 JSON, H2, H3) and for each handler model (event loop vs virtual threads).

### Gaps
- JFR allocation events (`jdk.ObjectAllocationSample` throttling), JFR overhead figures and JMC usage beyond the TTSP rule were not verified this session.
- The publication date of async-profiler 4.6 was not verified.
- jHiccup's internal mechanism and overhead are not documented in its README.
- The exact current replacement for `PrintSafepointStatistics` (`-Xlog:safepoint+stats`) came only from search snippets.
- No source was found on running perf-style JIT symbol mapping on Windows or macOS.

## Q4. What are JMH best practices and pitfalls for microbenchmarking HTTP parsers, encoders and routers? (DCE, constant folding, Blackhole, forks, warm-up, @State scopes, -prof gc/perfasm/async; how Netty, Jetty and Vert.x structure suites; CI output formats)

### Takeaway
Write each component benchmark to defeat the JIT. Consume every result (return it or use a Blackhole) and read inputs from non-final `@State` fields. Run at least two forks so different benchmarks don't mix JIT profiles. Track `gc.alloc.rate.norm` as a first-class result, and use perfasm only on Linux with one fork. Follow Netty's pattern: a shared abstract base class with fixed JVM arguments, 10×1 s warm-up and 10×1 s measurement, two forks, `Scope.Thread` state, and JSON output per class for CI. JMH 1.37 (2023) is still the current release.

### Cited Findings

#### Core pitfalls (JMH samples)
- **Dead-code elimination**: "The downfall of many benchmarks is Dead-Code Elimination (DCE): compilers are smart enough to deduce some computations are redundant and eliminate them completely". Return the result, which JMH implicitly consumes with a Blackhole. "This is wrong: result is not used and the entire computation is optimized away" — [JMHSample_08_DeadCode](https://github.com/openjdk/jmh/blob/master/jmh-samples/src/main/java/org/openjdk/jmh/samples/JMHSample_08_DeadCode.java)
- **Constant folding**: "The flip side of dead-code elimination is constant-folding". Prevent it "by always reading the inputs from non-final instance fields of @State objects". On IDE hints to make such fields final: "Don't. Trust. Them." — [JMHSample_10_ConstantFold](https://github.com/openjdk/jmh/blob/master/jmh-samples/src/main/java/org/openjdk/jmh/samples/JMHSample_10_ConstantFold.java)
- **Forking and profile pollution**: "different tests can mix their profiles together"; "Forking (running in a separate process) each test can help to evade this issue". JMH forks by default, and `@Fork(0)` runs in the same JVM — [JMHSample_12_Forking](https://github.com/openjdk/jmh/blob/master/jmh-samples/src/main/java/org/openjdk/jmh/samples/JMHSample_12_Forking.java)

#### Profilers
- `-lprof` lists profilers, and `-prof <name>:help` shows their options. Profilers are for quick turnaround and "not a substitute for full external profilers" — [JMHSample_35_Profilers](https://github.com/openjdk/jmh/blob/master/jmh-samples/src/main/java/org/openjdk/jmh/samples/JMHSample_35_Profilers.java)
- `-prof stack` shows whether the code under test runs, but "can fail to notice quickly executing methods" — [JMHSample_35](https://github.com/openjdk/jmh/blob/master/jmh-samples/src/main/java/org/openjdk/jmh/samples/JMHSample_35_Profilers.java)
- `-prof gc` provides `gc.alloc.rate.norm` (B/op) and `gc.churn`; cross-check the two — [JMHSample_35](https://github.com/openjdk/jmh/blob/master/jmh-samples/src/main/java/org/openjdk/jmh/samples/JMHSample_35_Profilers.java). On JDK 21+ it uses `getTotalThreadAllocatedBytes`, which also counts allocations made by other threads — [JMH RFR](https://mail.openjdk.org/pipermail/jmh-dev/2023-June/003682.html)
- The hardware-counter profilers:
  - `-prof perf` and `-prof perfnorm` give per-op CPU counters (cycles, CPI, loads, branches). Run them with multiple forks for better error estimates.
  - `-prof perfasm` maps hot regions to generated assembly. "Since program addresses change from fork to fork, it does not make sense to run perfasm with more than a single fork", and "Hardware counters are not very precise about the instruction addresses".
  - These require Linux perf_events. The equivalents are `xperfasm` on Windows and `dtraceasm` on macOS.

  — [JMHSample_35](https://github.com/openjdk/jmh/blob/master/jmh-samples/src/main/java/org/openjdk/jmh/samples/JMHSample_35_Profilers.java)
- `-prof comp`: "It is expected to have some level of background compilation even at steady state". `-prof cl` reports class loading per operation, which should be zero for non-classloading benchmarks — [JMHSample_35](https://github.com/openjdk/jmh/blob/master/jmh-samples/src/main/java/org/openjdk/jmh/samples/JMHSample_35_Profilers.java)
- A THP microbenchmark used `-prof perfnorm` to show dTLB load misses per op dropping from 1.017 to about 10⁻³ — an example of using counters to explain a speed-up — [Quark #2](https://shipilev.net/jvm/anatomy-quarks/2-transparent-huge-pages/)

#### Versions and output formats for CI
- jmh-core 1.37, released 3 August 2023, is the latest version on Maven Central — [Maven Central metadata](https://repo1.maven.org/maven2/org/openjdk/jmh/jmh-core/maven-metadata.xml). Vert.x core 5.1.8's POM pins JMH 1.37 — [Maven Central vertx-core 5.1.8](https://central.sonatype.com/artifact/io.vertx/vertx-core/5.1.8)
- Result formats selectable with `-rf` are text, csv, scsv, json and latex — [jmh-dev 2015](https://mail.openjdk.org/pipermail/jmh-dev/2015-September/002047.html). JSON was added because tools had been scraping the console output, which is not formally specified — [jmh-dev 2013](https://mail.openjdk.org/pipermail/jmh-dev/2013-October/000195.html). CSV output follows the system locale, a problem raised on the list in 2016 — [jmh-dev 2016](https://mail.openjdk.org/pipermail/jmh-dev/2016-September/002349.html)

#### How Netty structures its suite (4.2 branch)
- `AbstractMicrobenchmarkBase` sets:
  - `@Warmup(iterations = 10, time = 1, timeUnit = SECONDS)`, `@Measurement(iterations = 10, time = 1 s)` and `@State(Scope.Thread)`.
  - `BASE_JVM_ARGS` = `-server -dsa -da -ea:io.netty... -XX:+HeapDumpOnOutOfMemoryError -Dio.netty.leakDetection.level=disabled`, plus `ResourceLeakDetector.setLevel(DISABLED)` in code.
  - Results written with `ResultFormatType.JSON` to `<perfReportDir>/<ClassName>.json`, where `perfReportDir` is a system property.

  — [AbstractMicrobenchmarkBase](https://github.com/netty/netty/blob/4.2/microbench/src/main/java/io/netty/microbench/util/AbstractMicrobenchmarkBase.java)
- `AbstractMicrobenchmark` sets `@Fork(DEFAULT_FORKS = 2)`, which a `-Dforks` system property can override. It adds `-Xms768m -Xmx768m -XX:MaxDirectMemorySize=768m`, and optionally a custom JMH harness executor (`-Djmh.executor=CUSTOM -Djmh.executor.class=...HarnessExecutor`) — [AbstractMicrobenchmark](https://github.com/netty/netty/blob/4.2/microbench/src/main/java/io/netty/microbench/util/AbstractMicrobenchmark.java)
- Netty's wiki page on microbenchmarks is outdated: it still shows the old `@GenerateMicroBenchmark` annotation and `mvn -DskipTests=false test`. It gives the same 10/10/2 defaults, and the current code differs from it, notably in the JSON output and the custom harness executor. The README of the `microbench` module only links to that page — [Netty wiki microbenchmarks](https://netty.io/wiki/microbenchmarks.html); [netty/microbench](https://github.com/netty/netty/tree/4.2/microbench)

#### Jetty and Vert.x
- Jetty 12.1.x has a `tests/jetty-jmh` module (`src/main` plus `pom.xml`); its contents were not enumerated — [jetty.project tests/jetty-jmh](https://github.com/jetty/jetty.project/tree/jetty-12.1.x/tests/jetty-jmh). In Jetty 9, benchmarks lived in packages such as `org.eclipse.jetty.http.jmh`, including `HttpMethodBenchmark` and `MultiPartBenchmark` with `testParser` and `testLargeGenerated` — [Jetty 9 Javadoc](https://javadoc.jetty.org/jetty-9/org/eclipse/jetty/http/jmh/package-summary.html)
- Jetty also publishes end-to-end figures: Jetty 12 EE10 is "5% faster" than Jetty 11 EE9, and the Jetty 12 Core API is "22% faster" — [Webtide](https://webtide.com/jetty-12-performance-figures)
- Vert.x: only the JMH version pinned in the POM was confirmed; the location of the benchmark sources was not — [vertx-core 5.1.8](https://central.sonatype.com/artifact/io.vertx/vertx-core/5.1.8)

### Inferences
- **HTTP parser and encoder benchmarks:**
  - Keep a corpus of realistic request and response byte arrays in `@State(Scope.Thread)`, covering small, typical (10–20 headers) and large-header cases, pipelined requests, and chunked bodies.
  - Index the corpus with a non-final cursor so the JIT cannot fold inputs.
  - Feed both heap and direct `ByteBuffer`s.
  - Return the parsed object or consume every field via `Blackhole`.
  - Use `@Param` for corpus shape.
  - Report throughput and `gc.alloc.rate.norm` together. Zero-allocation parsing is a claim B/op can verify.
- **Router benchmarks:**
  - Size route tables realistically (10, 100, 1,000 routes with parameters and wildcards).
  - Draw lookup keys from a pre-generated random sequence of hits and misses, not a single repeated path, so neither branch prediction nor constant folding flatters the result.
  - Include a megamorphic case (many handler types). Single-type benchmarks overstate the benefit of inlining.
- **Benchmark JVM flags:**
  - Mirror the standardized server flags: explicit GC, explicit compact headers, and the same JDK build.
  - Run the suite on both the LTS (JDK 25) and the current release (JDK 27). JDK 27's default compact headers change object sizes and therefore cache behaviour and B/op.
- **Fork policy:** use at least 2 forks (Netty's default), with more forks for small effects, and a single fork only for `-prof perfasm`.
- **CI:**
  - Emit JSON (`-rf json -rff <file>`), never parse console text.
  - Use CSV only if the locale is pinned.
  - Archive B/op and ops/s per benchmark per commit.
  - Flag regressions on B/op deterministically, and on ops/s against confidence intervals.
- Keep **end-to-end server benchmarks** separate from JMH. JMH isolates component costs but cannot show syscall, scheduling or GC interaction under real concurrency, which is why Jetty publishes both kinds.

### Gaps
- The layout of Vert.x's JMH sources could not be verified: GitHub API access was restricted in this session, and a guessed path returned 404.
- The class list in Jetty 12's `jetty-jmh` was not enumerated.
- Netty's package layout (`microbench/.../http`, `http2`, `buffer`) was not listed.
- JMH's async-profiler integration (`-prof async`) was not verified from a primary source.
- The JMH `@State` scope semantics (Benchmark vs Group vs Thread) and the Blackhole samples (JMHSample_09) were not fetched, beyond the references in JMHSample_08.

## Q5. What differs across operating systems for JVM network I/O (Linux epoll/io_uring, macOS kqueue, Windows IOCP vs NIO), how do Netty's native transports compare with NIO, and what does that imply for benchmarking on each OS?

### Takeaway
- **Linux** is the only OS with the full fast path (Netty epoll and io_uring, FFM-based io_uring experiments, THP and large pages) and the full diagnostic toolchain (perf, async-profiler, JFR CPU-time sampling, perf maps). Headline numbers should come from Linux.
- **macOS** (kqueue) is a developer platform, and its x64 port is being deprecated in JDK 28.
- **Windows** runs NIO on a wepoll-based selector (JDK 17+), with IOCP only behind AsynchronousChannels, and Netty has no native transport there. Windows needs its own smaller regression suite and must never be compared directly with Linux numbers.

### Cited Findings
- Netty's JNI transports are epoll on Linux (since 4.0.16) and kqueue on macOS/BSD (since 4.1.11). They "add features specific to a particular platform, generate less garbage, and generally improve performance when compared to the NIO based transport". They need OS/arch classifiers (e.g. `linux-x86_64`, `osx-x86_64`). "The official Linux builds are all linked against GLIBC", so musl/Alpine needs a self-built library. The page mentions no Windows transport — [Netty native transports](https://netty.io/wiki/native-transports.html)
- Netty 4.2 changes:
  - io_uring graduated from incubator in 4.2.0 into "a fully supported first-class transport module"; the incubator module is unsupported, and io_uring "requires Java9+".
  - Transport-specific event-loop groups such as `NioEventLoopGroup` are deprecated in favour of `MultiThreadIoEventLoopGroup` with `NioIoHandler`, `EpollIoHandler`, `KQueueIoHandler` or `IoUringIoHandler` factories.
  - The pipeline call stack "has been flattened, to reduce stack depths".

  — [Netty 4.2 migration guide](https://netty.io/wiki/netty-4.2-migration-guide.html); [Netty 4.2.0 news](https://netty.io/news/2025/04/03/4-2-0.html)
- io_uring is still being tuned in 4.2.x; for example, 4.2.10 (February 2026) submits `send` rather than `write` for sockets — [Netty 4.2.10 news](https://netty.io/news/2026/02/05/4-2-10.html) (via search summary)
- Windows: JDK 17 replaced the legacy Windows Selector, based on `select`, which "scales horribly". It was O(N) in registered channels and limited to 1024 sockets, forcing a thread per 1024 sockets. The replacement is a wepoll-based `WEPollSelectorImpl`; wepoll gives an epoll-like interface over the Winsock Ancillary Function Driver, so cost scales with ready sockets. The old selector remains available via `-Djava.nio.channel.spi.SelectorProvider=sun.nio.ch.WindowsSelectorProvider` — [JDK-8266369](https://bugs.openjdk.org/browse/JDK-8266369); [nio-dev RFR](https://mail.openjdk.org/pipermail/nio-dev/2021-April/008873.html)
- On Windows, `AsynchronousChannelGroup` is implemented over an I/O completion port (`Iocp`), with socket handles registered to the port and per-channel pending-I/O caches. This is from JDK 7/8 sources, so current code may differ — [JDK 8 WindowsAsynchronousChannelProvider](https://code.yawk.at/java/8/sun/nio/ch/WindowsAsynchronousChannelProvider.java); [JDK 7 Iocp](https://code.yawk.at/java/7/sun/nio/ch/Iocp.java)
- macOS: JEP 541 deprecates the macOS/x64 port for removal in JDK 28. Apple has moved to AArch64, and Oracle will stop maintaining the x64 port — [JEP 541](https://openjdk.org/jeps/541)
- Linux-only or Linux-first measurement features:
  - JFR CPU-time profiling (JEP 509) — [JEP 509](https://openjdk.org/jeps/509)
  - `Compiler.perfmap` — [JDK-8254723](https://bugs.openjdk.org/browse/JDK-8254723)
  - JMH perf/perfnorm/perfasm (Windows has `xperfasm`, macOS `dtraceasm`) — [JMHSample_35](https://github.com/openjdk/jmh/blob/master/jmh-samples/src/main/java/org/openjdk/jmh/samples/JMHSample_35_Profilers.java)
  - async-profiler on Linux and macOS only, with no Windows build — [async-profiler](https://github.com/async-profiler/async-profiler)
  - ZGC automatic large pages on Linux 6.1+ (JDK 28 target) — [JEP 545](https://openjdk.org/jeps/545)
  - CRaC on Linux only — [Azul CRaC comparison](https://docs.azul.com/crac/azul-zulu-zing-crac-compared)
  - Native Image G1 on Linux only — [GraalVM](https://www.graalvm.org/latest/reference-manual/native-image/optimizations-and-performance/MemoryManagement/)
- FFM-based io_uring (JUring) needs Linux 5.1+ and liburing, and its published benchmarks cover file I/O only — [JUring](https://github.com/davidtos/juring). `nio_uring` is a JNI alternative (Linux 5.1+, Java 8+) that describes itself as "truly zero-copy" with direct ByteBuffers — [nio_uring](https://github.com/bbeaupain/nio_uring)
- The Quarkus measurement guide covers Linux, macOS and Docker commands, and not Windows — [Quarkus](https://quarkus.io/guides/performance-measure)

### Inferences
- **Benchmark tiers by OS:**
  - Tier 1, Linux on x64 and AArch64: authoritative throughput, latency and footprint numbers, comparing NIO, epoll-equivalent and io_uring backends on the same kernel. Record the kernel version, because io_uring features and performance depend on it. Also record the glibc version and the THP settings.
  - Tier 2, macOS on AArch64 only, given JEP 541: functional and regression performance with kqueue or NIO, for developer feedback only.
  - Tier 3, Windows: NIO with the wepoll selector, plus Meleys' own IOCP path if it builds one via FFM. Use it to catch Windows-specific regressions such as selector scaling, and report it separately.
- **When comparing with Netty-based frameworks,** record which transport each used, because NIO vs epoll vs io_uring changes garbage generated and throughput. A Netty framework on Linux may silently fall back to NIO if the native classifier is missing or the system uses musl (Alpine images).
- **For Meleys' own FFM-based native I/O:** measure it against JDK NIO on the same JDK, using equal thread counts and the same allocation-per-request metric. Check that FFM upcalls never block on virtual threads, since they would pin under JEP 491. Note that FFM-based io_uring for network sockets has no published, independent benchmark baseline yet.
- **Diagnostic coverage is uneven:**
  - Windows has no async-profiler, perf or JFR CPU-time sampler, so JFR's standard execution sampler, JMH `xperfasm` and Windows tooling (ETW/WPA, not researched) must stand in.
  - On macOS, async-profiler works but perf does not.
  - Plan the diagnostic bundle per OS accordingly.

### Gaps
- No source was found on how the JDK's virtual-thread I/O poller is implemented on each OS (epoll, kqueue, wepoll), or on how it affects benchmark results.
- That JDK NIO selectors use epoll on Linux and kqueue on macOS (`EPollSelectorImpl`/`KQueueSelectorImpl`) was not verified from a primary source this session.
- No benchmark comparing Netty epoll vs NIO vs io_uring on current kernels and JDKs was found in primary sources.
- No data was found on Windows IOCP vs the wepoll selector, or on any Windows-native Java HTTP transport.
- Windows process-memory metrics (Working Set, Private Bytes, Commit) for JVM footprint reporting were not researched.
- ETW/WPA-based JVM profiling on Windows was not researched.
