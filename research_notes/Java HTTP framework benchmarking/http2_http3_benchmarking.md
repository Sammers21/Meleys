# Benchmarking HTTP/2 and HTTP/3 (QUIC) servers for a Java HTTP framework (Meleys), as of 2026-10-21

Scope: server-side throughput, latency, CPU and memory for HTTP/2 (h2 over TLS, h2c) and HTTP/3 over QUIC. HTTP/1.1 tools and general latency methodology (coordinated omission, open vs closed models) belong to another researcher and appear here only for comparison. "Observed 2026-10-21" means I checked the repository or page directly on that date (git ls-remote, sparse clone, or curl).

## 1. Which load generators support HTTP/2 and HTTP/3 as of 2026, and how well?

### Takeaway
h2load (nghttp2) is the only mainstream load generator that covers h2, h2c and h3 and controls everything that matters: connections (`-c`), streams per connection (`-m`), flow-control windows, header-table sizes, TLS groups and resumption, and UDP GSO. It now reports median, p95 and p99. Its HTTP/3 mode is still labelled experimental and needs a custom build against ngtcp2/nghttp3. oha supports HTTP/2, and HTTP/3 as an experimental compile-time feature, but its HTTP/3 client hit a ceiling of about 85k req/s in one public suite. Hyperfoil, Gatling, Vegeta and k6 support HTTP/2, mostly over TLS only. None of them has built-in HTTP/3; k6 can get it only through a third-party xk6 extension. The QUIC-level tools (secnetperf, quinn-perf, the quic-interop-runner measurements) measure transports, not HTTP frameworks.

### Cited Findings

#### h2load (nghttp2)
- The current manual page is labelled "nghttp2 1.71.0". The latest Git release tag I observed on 2026-10-21 was v1.70.0, so the docs page may be built from the development branch. — [h2load(1) manual](https://nghttp2.org/documentation/h2load.1.html); [nghttp2 repo](https://github.com/nghttp2/nghttp2)
- Concurrency controls:
  - `-c/--clients` sets the number of concurrent clients (connections).
  - `-m/--max-concurrent-streams` sets the maximum concurrent streams per session. For HTTP/1.1 it sets the number of pipelined in-flight requests instead.
  - `-t/--threads` sets the number of native threads.
  - `-D/--duration` sets a timed run, and `--warm-up-time` adds a warm-up period before it. — [h2load(1) manual](https://nghttp2.org/documentation/h2load.1.html)
- Pacing controls:
  - `-r/--rate` opens a fixed number of connections per `--rate-period` and cannot be combined with `-D`.
  - `--rps=<N>` means "Specify request per second for each client".
  - `--timing-script-file` replays requests at scripted millisecond offsets. — [h2load(1) manual](https://nghttp2.org/documentation/h2load.1.html)
- Flow control:
  - `-w/--window-bits` sets the stream window to (2^N)-1. The default is 30 for TCP and 24 for QUIC (capped at 26).
  - `-W` sets the connection window, default 30.
  - The manual says h2load "effectively disables flow control" by default, and that `-w16 -W16` matches the HTTP/2 spec defaults. — [h2load(1) manual](https://nghttp2.org/documentation/h2load.1.html)
- Header compression: `--header-table-size` (decoder) and `--encoder-header-table-size` both default to 4K. The output reports header bytes and a "space savings" ratio. — [h2load(1) manual](https://nghttp2.org/documentation/h2load.1.html)
- Protocol selection:
  - `--alpn-list` defaults to `h2,http/1.1`.
  - `--h1` forces HTTP/1.1, and `--h3` is shorthand for `--alpn-list=h3`.
  - `-p/--no-tls-proto` defaults to `h2c` for plain `http://` URIs, so cleartext defaults to prior-knowledge HTTP/2. — [h2load(1) manual](https://nghttp2.org/documentation/h2load.1.html); [h2load HOW-TO](https://nghttp2.org/documentation/h2load-howto.html)
- TLS controls:
  - `--ciphers` and `--tls13-ciphers` select cipher suites.
  - `--groups` defaults to `X25519:P-256:P-384:P-521`.
  - `--ktls` enables kernel TLS.
  - `--tls-session-file` reads a TLS session from a file "and set it to all TLS connections to perform the session resumption". — [h2load(1) manual](https://nghttp2.org/documentation/h2load.1.html)
- QUIC-only options are `--qlog-file-base`, `--max-udp-payload-size` and `--no-udp-gso`. QUIC output adds min RTT, smoothed RTT, packets sent/received/lost, and "GRO packets" (the number of packets received in a single recvmsg call). — [h2load(1) manual](https://nghttp2.org/documentation/h2load.1.html)
- Output metrics:
  - "request" is the time from sending the first byte of a request to receiving the last byte of the response.
  - "connect" includes the TLS handshake. TTFB and req/s are also reported.
  - Each distribution reports min, max, median, p95, p99, mean, sd and "+/- sd". — [h2load(1) manual](https://nghttp2.org/documentation/h2load.1.html)
- HTTP/3 build requirements: HTTP/3 support "is experimental", and it needs the `--enable-http3` configure option plus three libraries:
  - A QUIC-capable TLS library: quictls, wolfSSL, LibreSSL (no 0-RTT), aws-lc, BoringSSL (a pinned commit), or OpenSSL >= 3.5.0.
  - ngtcp2 >= 1.23.0.
  - nghttp3 >= 1.17.0. — [nghttp2 README](https://github.com/nghttp2/nghttp2/blob/master/README.rst); [h2load HOW-TO](https://nghttp2.org/documentation/h2load-howto.html)
- The HOW-TO notes that the client itself can become the bottleneck and suggests `-t` as the fix, without guidance on how many threads to use. — [h2load HOW-TO](https://nghttp2.org/documentation/h2load-howto.html)
- HttpArena's HTTP/3 driver ("h2load-h3", h2load linked against ngtcp2, nghttp3 and quictls) uses multi-threaded workers with sendmmsg/recvmmsg batching. Against Kestrel+msquic on the same hardware, it reached about 580k req/s, where oha reached about 85k req/s. At 580k, both client and server were CPU-saturated. — [HttpArena h2load-h3 doc](https://www.http-arena.com/docs/load-generators/h3/h2load-h3/)
- HttpArena's command lines:
  - HTTP/2: `h2load https://localhost:8443/baseline2?a=1&b=1 -c 256 -m 100 -t 128 -D 5s` (also with `-c 1024`).
  - HTTP/3: `h2load-h3 --alpn-list=h3 ... -c 64 -m 64 -t 64 -D 5s`. — [HttpArena h2load doc](https://github.com/MDA2AV/HttpArena/blob/main/site/content/docs/load-generators/h2/h2load.md); [HttpArena h2load-h3 doc](https://www.http-arena.com/docs/load-generators/h3/h2load-h3/)
- Cloudflare's 2020 work on QUIC UDP transmission also used h2load "built with QUIC and HTTP/3 support" as the client. — [Cloudflare blog, 2020-01-08](https://blog.cloudflare.com/accelerating-udp-packet-transmission-for-quic/)

#### oha
- Protocol and concurrency flags:
  - `--http-version` accepts 0.9, 1.0, 1.1, 2 and 3; `--http2` is shorthand for version 2.
  - `-p` sets the "Number of parallel requests to send on HTTP/2", so oha runs c×p workers.
  - `-c` sets connections and defaults to 50. — [oha README](https://github.com/hatoo/oha)
- HTTP/3 must be enabled with the `http3` cargo feature. It uses hyperium's H3 library, requires rustls, and "will remain experimental as long as H3 is experimental". — [oha README](https://github.com/hatoo/oha)
- Latency correction: `--latency-correction` (to "avoid coordinated omission") only works together with `-q`, which sets total QPS. Output formats are text, json and csv. On HTTP/2, redirects aren't supported and `--disable-keepalive` doesn't apply. The latest tag on 2026-10-21 was v1.16.0. — [oha README](https://github.com/hatoo/oha)
- In HttpArena's tests, oha's single-threaded quinn-based HTTP/3 client "became the bottleneck before the server did" at about 85k req/s and "couldn't saturate any framework". — [HttpArena h2load-h3 doc](https://www.http-arena.com/docs/load-generators/h3/h2load-h3/)

#### Hyperfoil (Java, Vert.x/Netty-based)
- Protocol defaults:
  - `allowHttp1x` and `allowHttp2x` are both true; HTTP/2 is negotiated via ALPN.
  - If both versions are allowed and https isn't used, Hyperfoil uses HTTP/1.1. If only 2.0 is allowed, it starts with HTTP/1.1 and upgrades.
  - `directHttp2` (h2c with prior knowledge) is documented as "Currently not implemented". — [Hyperfoil HTTP config docs](https://github.com/Hyperfoil/Hyperfoil/blob/master/docs/site/content/en/docs/user-guide/benchmark/http.md)
- Connection and stream defaults: `maxHttp2Streams` is 100, `sharedConnections` is 1 (the docs say "It is recommended to set this property to a non-default value"), and `pipeliningLimit` is 1. — [Hyperfoil HTTP config docs](https://github.com/Hyperfoil/Hyperfoil/blob/master/docs/site/content/en/docs/user-guide/benchmark/http.md)
- Hyperfoil's wrk/wrk2-compatible CLI disables HTTP/2 by default; `--enable-http2` turns it on. — [Hyperfoil wrk migration doc](https://github.com/Hyperfoil/Hyperfoil/blob/master/docs/site/content/en/docs/migration/wrk.md)
- No HTTP/3 or QUIC support: I grepped a sparse clone of the master branch (commit dated 2026-10-15) and found no HTTP/3 or QUIC references in the `http` module source or the docs. The latest tag was release-0.23. — [Hyperfoil repo](https://github.com/Hyperfoil/Hyperfoil)
- Hyperfoil describes itself as a distributed benchmark framework and load driver that avoids the coordinated-omission fallacy. — [Hyperfoil GitHub org](https://github.com/hyperfoil)

#### k6
- The k6 v2.3.x docs list HTTP/1.1, HTTP/2, WebSockets and gRPC as built-in. Other protocols come through xk6 extensions. — [k6 protocols, v2.3.x](https://grafana.com/docs/k6/v2.3.x/using-k6/protocols/)
- On HTTP/2, k6 "automatically upgrades the connection to HTTP/2.0 if the server supports it", and you check the result with `res.proto`. The docs describe no stream-per-connection or connection-count controls and don't mention h2c. — [k6 HTTP/2 docs](https://grafana.com/docs/k6/latest/using-k6/protocols/http-2/)
- HTTP/3 exists only in a third-party extension, `github.com/dnsoa/net/xk6-http3` (Apache-2.0, dated April 12, 2026). — [pkg.go.dev xk6-http3](https://pkg.go.dev/github.com/dnsoa/net/xk6-http3)
- An older community thread (around 2022) says earlier HTTP/3 attempts stalled because k6 must time requests accurately, and the Go QUIC library lacked httptrace support. — [Grafana community thread](https://community.grafana.com/t/k6-http-3-quic-support/98449)

#### Gatling
- HTTP/2 behaviour:
  - `enableHttp2()` negotiates HTTP/2 via ALPN and falls back to HTTP/1.
  - Once Gatling knows a remote speaks HTTP/2, each virtual user keeps one multiplexed connection to it.
  - `http2PriorKnowledge(Map)` skips negotiation for listed hosts.
  - HTTP/2 Push is unsupported, and `maxConnectionsPerHost` has no effect with HTTP/2.
- The protocol docs don't mention HTTP/3 or QUIC. The newest "What's new" entry is 3.16. — [Gatling HTTP protocol docs](https://docs.gatling.io/reference/script/http/protocol/)
- A Gatling maintainer wrote in 2020 (Gatling 3.3.1): "Gatling doesn't support h2c atm, only h2". This is dated. — [Gatling community](https://community.gatling.io/t/cant-send-request-using-http-2-and-prior-knowledge/5542)

#### Vegeta
- HTTP/2 and connection flags: `-http2` (default true), `-h2c` ("Send HTTP/2 requests without TLS encryption"), `-session-tickets` (TLS resumption), `-connections` (default 10000 idle connections per host) and `-max-connections`.
- Rate flags: `-rate` defaults to 50/1s, and 0 means unlimited. `-max-workers` caps the worker pool.
- Reporting: p50, p90, p95 and p99 plus min/mean/max, and an `hdrplot` output. The README says Vegeta "avoids" coordinated omission.
- The README doesn't mention HTTP/3 or QUIC. The latest tag was v12.13.0 on 2026-10-21. — [Vegeta README](https://github.com/tsenart/vegeta)

#### QUIC-level tools (transport, not HTTP framework)
- MsQuic SecNetPerf:
  - Implements the `draft-banks-quic-performance` protocol.
  - Flags: `-target`, `-exec:<lowlat,maxtput,scavenger,realtime>`, `-up`, `-down`, `-streams`, `-conns`, `-rconn`, `-rstream` and `-runtime`. `-tcp:1` switches the test to TCP/TLS for comparison.
  - Prints throughput, RPS, and latency percentiles up to the 99.9999th.
  - The docs' localhost example shows 21,566 RPS for QUIC against 30,555 RPS for TCP/TLS with identical options. That is one illustrative run. — [MsQuic SecNetPerf docs](https://microsoft.github.io/msquic/msquicdocs/src/perf/readme.html)
- Microsoft's netperf project produces a dashboard that measures "the secure variants of the protocols (e.g. TCP/TLS, QUIC)". — [microsoft/netperf FAQ](https://github.com/microsoft/netperf/blob/main/docs/questions.md)
- quinn ships a `perf` crate with a `quinn-perf` binary that supports JSON output and qlog. — [quinn perf/Cargo.toml](https://github.com/quinn-rs/quinn/blob/main/perf/Cargo.toml)
- The quic-interop-runner has two measurements, both over a simulated network:
  - `goodput`: a 10 MB file over a 10 Mbps link, 5 repetitions, reported in kbps.
  - `crosstraffic`: a 25 MB transfer competing with a TCP cubic iperf flow. — [quic-interop-runner testcases_quic.py](https://github.com/quic-interop/quic-interop-runner/blob/master/testcases_quic.py)
- König et al. (IFIP Networking 2025) found that HTTP/3 instead of HTTP/0.9 "can reduce goodput by as much as 27%". Dedicated traffic generators can more than double the throughput obtained via HTTP. So transport-level perf tools overstate what an HTTP/3 server delivers. — [König et al., 2025](https://doc.tm.kit.edu/2025-Examining-the-Heterogeneous-Throughput-Performance-Landscape-of-QUIC-Implementations-Koenig-et-al.pdf)

#### Client-side bottlenecks
- On the receive side, QUIC is CPU-heavy for clients too. The WWW '24 study measured these CPU loads:

  | Client | HTTP/2 | HTTP/3 |
  |---|---|---|
  | Chrome | 77.1% | 97.4% |
  | Edge | 28.4% | 81.1% |
  | quic_client | n/a | nearly 100% |

  At 1.60 GHz, Chrome's QUIC throughput fell to about 200 Mbps while HTTP/2 stayed above 900 Mbps. — [Zhang et al., "QUIC is not Quick Enough over Fast Internet", WWW '24](https://arxiv.org/html/2310.09423v2)

### Inferences
- **h2load as the primary driver.** For Meleys, h2load (one pinned build with `--enable-http3`) should be the primary HTTP/2, h2c and HTTP/3 throughput driver. It is the only tool with matched semantics across all three protocols, which makes like-for-like protocol comparisons possible. HttpArena's `docker/h2load-h3.Dockerfile` is a usable template.
- **No open-model HTTP/3 latency tool.** For HTTP/2, Vegeta (h2 and h2c, constant rate, HDR output) or Hyperfoil (h2 over TLS only) can supply open-model latency next to h2load. For HTTP/3 there is no mature open-model, coordinated-omission-safe generator: oha's H3 mode tops out too low, and h2load `--rps` is a per-client rate in a multiplexed client. That is my inference: the manual doesn't say whether `--rps` corrects for coordinated omission. HTTP/3 tail-latency claims will therefore need either a custom harness or careful runs at moderate load with spare client CPU.
- **Size the client for HTTP/3.** The client must be provisioned and monitored like a system under test, because QUIC receive processing is expensive. A rule of thumb from the cited data is to give the HTTP/3 client at least as many cores as the server and to check that client CPU stays below saturation.
- **Use transport tools only for attribution.** secnetperf, quinn-perf and similar tools help separate QUIC-stack cost from framework cost, for example quiche via Netty compared with Meleys' HTTP/3 layer. They are not substitutes for HTTP-level tests.

### Gaps
- No independent scaling data for h2load-h3 (rps against thread count and cores) beyond HttpArena's single 580k/85k comparison.
- Unverified: whether k6 2.x supports h2c, the h2c status in Gatling 3.16, and oha's cleartext HTTP/2 behaviour (prior knowledge or upgrade).
- h2load's handling of coordinated omission under `--rps` is undocumented.
- I didn't find docs for a dedicated quiche perf client. Cloudflare's tooling found is h3i, a testing client covered in section 6.

## 2. Which methodology choices matter (streams vs connections, TLS, resumption/0-RTT, HPACK/QPACK, flow control, h2c vs h2, UDP tuning, netem, fair comparison with HTTP/1.1)?

### Takeaway
Results swing most on a few choices:
- In-flight concurrency (connections × streams).
- TLS on or off, and whether handshakes are amortized or repeated.
- Flow-control windows: h2load turns them effectively off by default.
- Header-compression state: HPACK's 4 KiB table versus QPACK's dynamic table, which is off by default.
- For QUIC, the UDP path: GSO/GRO, socket buffer limits, pacing qdisc, packet size, and loopback versus a real NIC.

A fair comparison with HTTP/1.1 equalizes in-flight requests, TLS, payloads and client CPU. Public suites usually don't.

### Cited Findings

#### Streams vs connections
- In h2load, `-m` is the maximum number of concurrent streams per session, and for HTTP/1.1 it becomes the pipelining depth. — [h2load(1) manual](https://nghttp2.org/documentation/h2load.1.html)
- RFC 9113 sets no initial limit on SETTINGS_MAX_CONCURRENT_STREAMS and recommends a value "no smaller than 100, so as to not unnecessarily limit parallelism". — [RFC 9113 §6.5.2](https://www.rfc-editor.org/rfc/rfc9113)
- HttpArena uses different shapes per protocol:
  - HTTP/1.1 baseline: 512 to 4,096 connections, one request at a time.
  - HTTP/2 baseline: 256 or 1,024 connections × 100 streams. The docs say "HTTP/2 uses far fewer connections because each connection multiplexes many streams."
  - HTTP/3 baseline: 64 connections × 64 streams. — [HttpArena baseline-h2](https://github.com/MDA2AV/HttpArena/blob/main/site/content/docs/test-profiles/h2/baseline-h2/implementation.md); [HttpArena baseline-h3](https://github.com/MDA2AV/HttpArena/blob/main/site/content/docs/test-profiles/h3/baseline-h3/implementation.md)
- König et al. found that more QUIC streams "may potentially double the throughput of multi-segment data transfers, depending on the implementation". — [König et al., 2025](https://doc.tm.kit.edu/2025-Examining-the-Heterogeneous-Throughput-Performance-Landscape-of-QUIC-Implementations-Koenig-et-al.pdf)
- Gatling gives each virtual user a single multiplexed HTTP/2 connection. Concurrency per connection therefore depends on the scenario, not on an explicit stream count. — [Gatling HTTP protocol docs](https://docs.gatling.io/reference/script/http/protocol/)

#### TLS cost; h2c vs h2
- I computed h2c-to-TLS throughput ratios from HttpArena's results (repo commit 3717e256, 2026-10-20) at 256 connections × 100 streams. The median h2c/h2 ratio across 43 entries is 1.11, ranging from 0.72 (nginx) to 3.13. Selected entries:

  | Entry | h2c (req/s) | h2 TLS (req/s) | h2c/h2 |
  |---|---|---|---|
  | quarkus-jvm | 6.19M | 4.67M | 1.33 |
  | helidon-production | 4.21M | 3.50M | 1.20 |
  | kestrel | 2.88M | 2.36M | 1.22 |
  | cardigan (Java io_uring/Panama) | 15.1M | 14.5M | 1.05 |
  | ktor | 816k | 794k | 1.03 |

  — [HttpArena results data](https://github.com/MDA2AV/HttpArena/tree/main/site/data/results)
- HttpArena treats TLS profiles as modelling "edge-facing deployments" and h2c profiles as modelling "reverse-proxy-to-origin and service-to-service deployments inside a trust boundary". — [HttpArena H/2 profiles](https://github.com/MDA2AV/HttpArena/blob/main/site/content/docs/test-profiles/h2/_index.md)
- HttpArena's TLS validation requires the harness-mounted certificate, TLS 1.3 with an AEAD cipher, and no obsolete protocols or weak ciphers. — [HttpArena baseline-h2 validation](https://github.com/MDA2AV/HttpArena/blob/main/site/content/docs/test-profiles/h2/baseline-h2/validation.md)
- JEP 527 (JDK 27, GA 2026-09-15) changes the TLS 1.3 key-exchange defaults:
  - X25519MLKEM768 is placed first, and the full default list is X25519MLKEM768, x25519, secp256r1, secp384r1, secp521r1, x448, ffdhe2048/3072/4096.
  - JDK TLS 1.3 clients offer both X25519MLKEM768 and x25519 key shares.
  - The default can be changed with `jdk.tls.namedGroups`.
  - h2load's default `--groups` (X25519:P-256:P-384:P-521) includes no hybrid group. — [JEP 527](https://openjdk.org/jeps/527); [JDK 27 project page](https://openjdk.org/projects/jdk/27/); [h2load(1) manual](https://nghttp2.org/documentation/h2load.1.html)
- Cipher choice behaves differently for QUIC and TLS/TCP on a 10G testbed:
  - For QUIC, ChaCha20 matched hardware-accelerated AES.
  - Without AES-NI, LSQUIC lost about 11% and quiche about 51%.
  - For TLS/TCP, AES-NI was about 3× faster than ChaCha20. — [Jaeger et al., "QUIC on the Highway", IFIP Networking 2023](https://ar5iv.labs.arxiv.org/html/2309.16395)

#### Session resumption and 0-RTT
- Resumption support in the tools:
  - h2load `--tls-session-file` reuses one session for all TLS connections.
  - Vegeta has `-session-tickets`.
  - LibreSSL builds of h2load don't support 0-RTT. — [h2load(1) manual](https://nghttp2.org/documentation/h2load.1.html); [Vegeta README](https://github.com/tsenart/vegeta); [nghttp2 README](https://github.com/nghttp2/nghttp2/blob/master/README.rst)
- The quic-interop-runner has separate `resumption` (no 0-RTT) and `zerortt` test cases. — [quic-interop-runner quic.md](https://github.com/quic-interop/quic-interop-runner/blob/master/quic.md)
- RFC 8470 covers the replay risk of early data and defines the 425 (Too Early) status code. A server benchmarked with 0-RTT must apply its replay policy during the run. — [RFC 8470](https://www.rfc-editor.org/rfc/rfc8470)

#### Header compression (HPACK/QPACK)
- In HTTP/2, SETTINGS_HEADER_TABLE_SIZE starts at 4,096 octets. — [RFC 9113 §6.5.2](https://www.rfc-editor.org/rfc/rfc9113)
- In QPACK, SETTINGS_QPACK_MAX_TABLE_CAPACITY and SETTINGS_QPACK_BLOCKED_STREAMS both default to zero. The dynamic table is therefore off unless the peer enables it. — [RFC 9204 §5](https://www.rfc-editor.org/rfc/rfc9204)
- h2load exposes the decoder and encoder table sizes (4K default each) and reports header "space savings". — [h2load(1) manual](https://nghttp2.org/documentation/h2load.1.html)
- HttpArena's HTTP/2 baseline lists "HPACK header compression performance" among the things it measures. — [HttpArena baseline-h2](https://github.com/MDA2AV/HttpArena/blob/main/site/content/docs/test-profiles/h2/baseline-h2/implementation.md)

#### Flow-control windows
- The RFC 9113 initial stream window is 65,535 octets. h2load's defaults (2^30-1 for TCP, 2^24-1 for QUIC) effectively disable flow control, and `-w16 -W16` restores the spec defaults. — [RFC 9113 §6.5.2](https://www.rfc-editor.org/rfc/rfc9113); [h2load(1) manual](https://nghttp2.org/documentation/h2load.1.html)
- The interop runner's `transfer` test requires small initial stream and connection windows so that windows must be raised during roughly 1 MB transfers. — [quic-interop-runner quic.md](https://github.com/quic-interop/quic-interop-runner/blob/master/quic.md)
- Jaeger et al. found that flow-control interaction between client and server limited goodput in some pairings, with CPU around 70% and no loss. — [Jaeger et al., 2023](https://ar5iv.labs.arxiv.org/html/2309.16395)

#### UDP path tuning for QUIC (GSO/GRO, buffers, pacing, packet size, ECN)
- Linux UDP GSO:
  - Lets an application pass one buffer of up to 64 KB, which the kernel splits. All segments must share a destination and all but the last must have equal size.
  - Available only on Linux kernels 4.18 and later.
  - quic-go lists Windows GSO as future work (#4325). — [quic-go optimizations docs](https://quic-go.net/docs/quic/optimizations/)
- Cloudflare's 2020 sending-path work (NGINX + quiche, h2load client, same Linux 5.3 laptop, 10 sequential requests for a 100 MB resource):
  - The plain sendmsg() path "tops out at around 80-90 MB/s".
  - sendmmsg() cut system calls dramatically.
  - UDP_SEGMENT batches up to 64 segments, and GSO combined with sendmmsg() gave a further "fairly significant" gain.
  - Pacing and batching conflict. SO_MAX_PACING_RATE with the fq scheduler is an option, but SO_TXTIME can't give separate timestamps to segments in a GSO buffer. — [Cloudflare blog, 2020-01-08](https://blog.cloudflare.com/accelerating-udp-packet-transmission-for-quic/)
- Socket buffers:
  - quic-go recommends `sysctl -w net.core.rmem_max=7500000` and `net.core.wmem_max=7500000`, and notes that Linux's maximum is otherwise too small for high-bandwidth QUIC.
  - On BSD/Darwin the kernel limit needs about 15% padding above the target; for about 7.5 MB the wiki gives `sysctl -w kern.ipc.maxsockbuf=8441037`. — [quic-go wiki: UDP Buffer Sizes](https://github.com/quic-go/quic-go/wiki/UDP-Buffer-Sizes)
- Jaeger et al. on buffers and offloads:
  - The default 208 KiB receive buffer caused client drops and retransmissions. Goodput stabilized after about a 16× increase, and the authors recommend raising the default by at least an order of magnitude.
  - QUIC gained little from GSO/GRO in their setup, which only raised client CPU, while TCP gained substantially from TSO.
  - Logging qlog or packet captures "heavily" reduces performance. — [Jaeger et al., 2023](https://ar5iv.labs.arxiv.org/html/2309.16395)
- The WWW '24 study found that the QUIC implementations it tested didn't use UDP GRO. Disabling offloads barely changed QUIC (18.60 s vs 18.82 s) but slowed HTTP/2 (9.36 s to 10.84 s). The authors recommend receiver GRO, recvmmsg, delayed ACKs (the ACK-frequency draft), and pacing compatible with GSO. — [Zhang et al., WWW '24](https://arxiv.org/html/2310.09423v2)
- Fastly's 2020 single-core microbenchmark of quicly showed the effect of each step on throughput:

  | Step | Throughput |
  |---|---|
  | Off-the-shelf quicly | 196 Mbps |
  | 1 ACK per 10 packets | 240 Mbps |
  | GSO, 10-packet bursts | 348 Mbps |
  | GSO, 20-packet bursts | 431 Mbps |
  | 1460-byte packets (vs 1280) | 466 Mbps |
  | TLS 1.3 over TCP (reference) | 466 Mbps |

  With 1350-byte packets (Chrome's default) the result was 8% below TLS over TCP. — [Fastly blog, 2020-04-30](https://www.fastly.com/blog/measuring-quic-vs-tcp-computational-efficiency)
- "QUIC Steps" (Kempf et al., 2025) compared pacing in quiche, picoquic and ngtcp2:
  - User-space timers alone can pace well, as picoquic with BBR shows.
  - For quiche, the FQ qdisc is a good fit.
  - The paper tests ETF hardware offload and a kernel patch that paces individual packets within GSO buffers. — [Kempf et al., arXiv 2505.09222](https://arxiv.org/abs/2505.09222)
- h2load can disable GSO (`--no-udp-gso`), cap the UDP payload (`--max-udp-payload-size`), and report GRO packets per recvmsg. — [h2load(1) manual](https://nghttp2.org/documentation/h2load.1.html)
- The JDK's UDP API:
  - `DatagramChannel` documents only SO_SNDBUF, SO_RCVBUF, SO_REUSEADDR, SO_BROADCAST, IP_TOS and the IP_MULTICAST_* options.
  - JDK 26's `jdk.net.ExtendedSocketOptions` adds TCP_QUICKACK, TCP_KEEPIDLE/INTERVAL/COUNT, SO_INCOMING_NAPI_ID, SO_PEERCRED and IP_DONTFRAGMENT. It has no UDP_SEGMENT or GRO option. — [JDK 26 DatagramChannel](https://docs.oracle.com/en/java/javase/26/docs/api/java.base/java/nio/channels/DatagramChannel.html); [JDK 26 ExtendedSocketOptions](https://docs.oracle.com/en/java/javase/26/docs/api/jdk.net/jdk/net/ExtendedSocketOptions.html)
- An earlier JDK enhancement request, JDK-8238725, noted that Java datagram sockets couldn't set the Don't Fragment bit, which "is problematic for implementing the QUIC protocol in Java". The issue was closed as a duplicate. — [JDK-8238725](https://bugs.openjdk.org/browse/JDK-8238725)

#### Loss/latency emulation
- The quic-interop-runner's network-simulator scenarios are reusable templates:
  - Base: `--delay=15ms --bandwidth=10Mbps --queue=25`.
  - Variants: `drop-rate` at 2% and 30% (bursts of 3), `corrupt-rate`, a `blackhole` with 5 s on and 2 s off, NAT `rebind`, and a 750 ms delay case. — [quic-interop-runner testcases_quic.py](https://github.com/quic-interop/quic-interop-runner/blob/master/testcases_quic.py)
- The WWW '24 study shaped bandwidth from 50 to 1,000 Mbps with Linux `tc`. — [Zhang et al., WWW '24](https://arxiv.org/html/2310.09423v2)
- netem is the standard Linux qdisc for adding delay, jitter, loss, duplication, corruption and reordering. — [tc-netem(8)](https://man7.org/linux/man-pages/man8/tc-netem.8.html)
- A WashU study found HTTP/3 excels under high loss and latency: up to 88.36% improvement in migration scenarios and 81.5% in extreme loss cases. It also found that optimized HTTP/2 can match HTTP/3 in some settings. This is client-perceived performance, not server CPU. — [arXiv 2409.16267](https://arxiv.org/html/2409.16267v3)

#### Reproducibility practices from QUIC benchmarking research
- Jaeger et al. recommend:
  - Dedicated hardware with separate client and server hosts.
  - Fixed configuration files.
  - Large transfers (8 GB) to get multi-second connections.
  - 50 repetitions per measurement.
  - Cubic or Reno for fair comparison (LSQUIC with BBR reached only 30 to 70% of its Cubic goodput).
  - RAM-disk live boots to remove storage I/O.
  - Recording all versions and publishing code and data.
- They also found that compile flags for the target CPU improved LSQUIC by 14 to 20%, and that QUIC on an Intel host was about 27% faster than on an AMD host where TCP was about 3% slower. — [Jaeger et al., 2023](https://ar5iv.labs.arxiv.org/html/2309.16395)
- HttpArena's setup, by contrast:
  - Server and load generator share one 64-core Threadripper PRO 3995WX. The server gets cpuset `0-31,64-95` and the load generator `32-63,96-127`.
  - Runs last 5 s, and the best of 3 is kept.
  - Its HTTP/3 docs attribute part of the HTTP/3 gap to "Loopback at MTU 1500" with no help from a real NIC. — [HttpArena hardware doc](https://github.com/MDA2AV/HttpArena/blob/main/site/content/docs/hardware/_index.md); [HttpArena baseline-h3](https://github.com/MDA2AV/HttpArena/blob/main/site/content/docs/test-profiles/h3/baseline-h3/implementation.md); [HttpArena h2load-h3 doc](https://www.http-arena.com/docs/load-generators/h3/h2load-h3/)

#### Comparing fairly against HTTP/1.1
- The TechEmpower plaintext test is HTTP/1.1 with pipelining, "NOT http 2". — [loom-dev, Nov 2023](https://mail.openjdk.org/pipermail/loom-dev/2023-November/006271.html)
- HttpArena keeps its 16× pipelined profile as reference-only and excludes it from the composite score. — [HttpArena README](https://github.com/MDA2AV/HttpArena)
- HttpArena's HTTP/1.1 and HTTP/2 numbers come from different concurrency. For example, quarkus-jvm does 1.82M req/s on HTTP/1.1 at 4,096 connections × 1 request, and 4.67M req/s on HTTP/2 over TLS at 256 connections × 100 streams. The HTTP/1.1 baseline is also plaintext. — [HttpArena results data](https://github.com/MDA2AV/HttpArena/tree/main/site/data/results); [HttpArena baseline-h2](https://github.com/MDA2AV/HttpArena/blob/main/site/content/docs/test-profiles/h2/baseline-h2/implementation.md)

### Inferences
- **Equalize the protocol matrix.** Define a matrix that matches in-flight requests and TLS:
  - HTTP/1.1 over TLS, N connections with 1 request each.
  - h2 over TLS, N/k connections × k streams.
  - h3, the same N/k × k.
  - h2c and plaintext HTTP/1.1 as separate internal-network profiles.

  Publish the connection and stream shape with every number. Cross-protocol RPS ratios mean little otherwise; HttpArena's 256×100 for h2 against 64×64 for h3 is a confound.
- **Run two flow-control modes.** Use h2load's default (flow control off, for peak capacity) and `-w16 -W16` (spec-default windows). The second shows the cost of window updates and buffering in the framework, which real clients trigger. The same applies to QUIC transport parameters.
- **Benchmark handshakes separately.** Measure new connections per second as full handshakes, resumed handshakes (`--tls-session-file`) and 0-RTT. Keep these apart from steady-state request throughput. Pin TLS key-exchange groups explicitly, X25519 against X25519MLKEM768. JEP 527 makes hybrid the JDK 27 default, so a Meleys server on JDK 27 will pay ML-KEM costs only when clients offer it, and h2load doesn't by default.
- **Pin and record the QUIC UDP environment on Linux:**
  - rmem_max and wmem_max raised (about 7.5 MB or more) and the socket buffers actually applied.
  - fq qdisc.
  - GSO/GRO confirmed in use (h2load's "GRO packets" metric and `--no-udp-gso` A/B runs).
  - MTU and packet size.
  - Kernel version and NIC offload settings.
  - qlog off.

  Avoid headline HTTP/3 numbers from loopback.
- **Treat Java's UDP limits as a measured constraint.** A pure-Java QUIC stack on `DatagramChannel` has no GSO/GRO and no documented way to read received ECN codepoints: the API lists IP_TOS only as a sending option, and I found no ancillary-data receive API. That puts it at a structural disadvantage compared with stacks using native batching. Meleys should benchmark its UDP layer separately (packets per second per core) before blaming the HTTP/3 code.

### Gaps
- No quantitative public data on HPACK versus QPACK CPU cost, or on QPACK dynamic table on versus off for server throughput.
- No public numbers on JVM server handshake rates (full, resumed, 0-RTT) for TLS 1.3 or QUIC, or on the server-side CPU cost of X25519MLKEM768.
- ECN's effect on QUIC server CPU or latency wasn't found. Windows USO/URO (UDP segmentation and receive offloads) and whether a JVM can reach them weren't researched.

## 3. What do published HTTP/2 and HTTP/3 server benchmarks show, and how much CPU does HTTP/3 add (and why)?

### Takeaway
Every independent study since 2020 finds that QUIC/HTTP/3 costs more CPU per byte or per request than TCP+TLS+HTTP/2, unless the stack is heavily optimized with GSO/GRO, ACK reduction, batched syscalls or kernel bypass:
- On 10 Gbit/s+ links, open-source QUIC stacks generally stay below TCP+TLS. Kernel stacks reach roughly 2 to 8 Gbit/s; DPDK pushes past 10.
- For small request/response loads, HttpArena's October 2026 data show a median HTTP/2-to-HTTP/3 throughput ratio of about 2.65× (range 0.70× to 4.66×). The JVM entries tested (Ktor on Netty+quiche) sit at about 2.3× to 3×.

No published benchmark exists for Jetty 12 HTTP/3, Kwik/Flupke, or Netty HTTP/3 compared with Netty HTTP/2. LiteSpeed and lsquic comparisons are old and vendor-adjacent.

### Cited Findings

#### Academic and engineering studies (chronological)
- **Fastly, April 2020 (Oku and Iyengar):**
  - Single-core sender at 400 MHz, synthetic bulk transfer. Raw TCP did 708 Mbps, TLS 1.3 over TCP 466 Mbps, and off-the-shelf quicly 196 Mbps.
  - Reducing ACK frequency, using GSO bursts and enlarging packets brought QUIC to parity (466 Mbps).
  - Named cost sources: ACK processing runs in user space with extra copies and context switches, and QUIC ACKs are encrypted while TCP ACKs are plaintext.
  - The authors caution that this is one microbenchmark. — [Fastly blog](https://www.fastly.com/blog/measuring-quic-vs-tcp-computational-efficiency)
- **Cloudflare, January 2020 (NGINX + quiche):** Baseline about 80 to 90 MB/s with sendmsg(). sendmmsg, GSO, and GSO plus sendmmsg each improved throughput. Results came from a single laptop over loopback, so they may not match production over a real network. — [Cloudflare blog](https://blog.cloudflare.com/accelerating-udp-packet-transmission-for-quic/)
- **Jaeger et al., IFIP Networking 2023 (TU Munich), 10G links:**
  - Throughput varied widely across implementations. The abstract gives about 90 to 4,900 Mbit/s; the body reports pairs from about 52 to 3,004 Mbit/s at baseline. TCP/TLS (nginx 1.18 + wget) reached about 8,010 Mbit/s.
  - LSQUIC and quiche were best. Tuned LSQUIC reached about 5 Gbit/s.
  - Packet I/O (sendmsg/recvmsg) was the largest CPU consumer. Crypto was visible but secondary.
  - The journal follow-up "QUIC on the Fast Lane" appeared in Computer Communications in 2024. — [Jaeger et al.](https://ar5iv.labs.arxiv.org/html/2309.16395); [TUM QUIC project page](https://net.in.tum.de/projects/quic)
- **Zhang et al., WWW '24 ("QUIC is not Quick Enough over Fast Internet"):**
  - QUIC's data rate is "up to 45.2%" lower than TCP+TLS+HTTP/2, and the gap grows with bandwidth. Below about 600 Mbps the two are similar; above it, curl over QUIC is 7 to 16% lower.
  - Root cause is receiver-side processing. The UDP stack received about 744K packets against 58K for TCP, `netif_receive_skb` ran 231K against 15K times, and syscalls were 17K against 4K. User-space ACK generation cost 2.97 s.
  - Server software: OpenLiteSpeed v1.7.15 (lsquic). nginx-quic was 18% slower than OpenLiteSpeed for QUIC, with similar HTTP/2.
  - Video bitrate dropped up to 9.8%, and Alexa-top-100 page load times were on average 3.0% longer.
  - The bottleneck was the client, not the more powerful server. — [Zhang et al., arXiv 2310.09423](https://arxiv.org/html/2310.09423v2)
- **König et al., IFIP Networking 2025 (KIT/TU Darmstadt), 10G:**
  - "None of the open-source QUIC implementations match the throughput achieved by conventional TCP+TLS stacks" at 10 Gbit/s and above.
  - HTTP/3 instead of HTTP/0.9 can cut goodput up to 27%.
  - Larger packets let some implementations saturate 10G, which points to packet processing as the bottleneck.
  - They tested lsquic, picoquic, ngtcp2, quiche and quic-go, among others. — [König et al.](https://doc.tm.kit.edu/2025-Examining-the-Heterogeneous-Throughput-Performance-Landscape-of-QUIC-Implementations-Koenig-et-al.pdf)
- **Mukherjee et al., ANRW '25 (Georgia Tech), "Understanding QUIC's Throughput Speedbumps":**
  - Five stacks on 25 Gbps NICs with GSO. picoquic-dpdk and MsQuic, the two best, reached only 58% and 53% of TCPLS throughput.
  - "Linux's UDP stack is seldom a throughput bottleneck". Crypto "represents a fundamental limitation on the performance of all QUIC implementations", and the choice of crypto library matters.
  - Pipeline parallelism improved quicly and mvfst by 1.36× and 1.88×. — [Mukherjee et al., ANRW '25](https://par.nsf.gov//servlets/purl/10637806)
  - This conflicts with the packet-I/O-dominant findings of Jaeger 2023, Zhang 2024 and Späth 2026. The ANRW study ran with GSO enabled on 25G NICs, so the bottleneck shifts depending on how well I/O is batched.
- **Späth et al., NOMS 2026 (TU Munich/MPI), "Kernel Bypass Surgery":**
  - Integrating DPDK into MsQuic, LSQUIC and quiche gave speedups up to 3× and cut packet-I/O overhead by up to 90%. LSQUIC-DPDK on both sides reached 10.8 Gbit/s.
  - On the kernel stack, MsQuic with GSO and GRO reached 8.3 Gbit/s, comparable to DPDK.
  - quiche was the slowest server at 2.2 Gbit/s (quiche 0.23.2, MsQuic 2.4.8).
  - Once packet I/O is fixed, crypto and file I/O become the new bottlenecks. — [Späth et al., NOMS 2026](https://www.net.in.tum.de/fileadmin/bibtex/publications/papers/spaethj-noms2026.pdf)
- **Kempf et al. ("QUIC Steps", 2025):** User-space pacing faces coarse timers, syscall overhead and OS scheduling delays, and GSO complicates precise pacing. — [arXiv 2505.09222](https://arxiv.org/abs/2505.09222)

#### Request/response multi-framework data (HttpArena, results snapshot at repo commit 3717e256, 2026-10-20)
- HttpArena's documentation says HTTP/3 throughput is "typically 4–6× lower than HTTP/2 on the same framework". It attributes this to per-packet AEAD with no kernel TLS offload, user-space packet processing, more syscalls per request than TCP, and loopback at MTU 1500. It calls this "the state of the QUIC ecosystem in 2026, not a configuration issue". — [HttpArena baseline-h3](https://github.com/MDA2AV/HttpArena/blob/main/site/content/docs/test-profiles/h3/baseline-h3/implementation.md); [HttpArena h2load-h3 doc](https://www.http-arena.com/docs/load-generators/h3/h2load-h3/)
- The current data are milder than the documentation. I compared baseline-h2 (256 connections × 100 streams, TLS) with baseline-h3 (64 × 64) for the 30 entries with valid results in both. The median h2/h3 RPS ratio is 2.65, ranging from 0.70 (nginx, where HTTP/3 beats HTTP/2) to 4.66. — [HttpArena results data](https://github.com/MDA2AV/HttpArena/tree/main/site/data/results)

  | Entry | Stack | h2 (req/s) | h3 (req/s) | h2/h3 |
  |---|---|---|---|---|
  | nginx | | 3.41M | 4.85M | 0.70 |
  | fib | Go | 5.63M | 6.07M | 0.93 |
  | caddy | quic-go | 375k | 361k | 1.04 |
  | h2o-mruby | | 3.96M | 3.04M | 1.30 |
  | zix | Zig | 14.08M | 9.09M | 1.55 |
  | agensio | C++ | 14.74M | 8.63M | 1.71 |
  | bun | | 7.06M | 3.44M | 2.06 |
  | ktor | Kotlin, Netty+quiche | 794k | 341k | 2.33 |
  | proxygen | | 5.85M | 1.83M | 3.20 |
  | ktor-tuned | | 1.14M | 378k | 3.01 |
  | kestrel | msquic | 2.36M | 669k | 3.53 |
  | aspnet-minimal | | 2.22M | 650k | 3.42 |
  | genhttp-ioxide | | 9.01M | 1.93M | 4.66 |

- baseline-h3 leaders were zix (Zig) at 9.09M req/s (p99 796 µs, 1,282% CPU, 412 MiB), agensio (C++) 8.63M, fib (Go) 6.07M, nginx 4.85M, ioxide (C#) 3.71M, and h2o-mruby 3.04M. — [HttpArena results data](https://github.com/MDA2AV/HttpArena/tree/main/site/data/results)
- baseline-h2 leaders at 256 connections were h2o 16.1M req/s (p99 15.8 ms, 3,033% CPU, 117 MiB), agensio 14.7M, and cardigan (Java: io_uring, Panama FFI, virtual threads) 14.5M (p99 51.6 ms, 1,474% CPU, 1.2 GiB). — [HttpArena results data](https://github.com/MDA2AV/HttpArena/tree/main/site/data/results); [cardigan meta.json](https://github.com/MDA2AV/HttpArena/blob/main/frameworks/cardigan/meta.json)
- Other JVM HTTP/2 results over TLS at 256 connections:

  | Entry | req/s | p99 | Memory |
  |---|---|---|---|
  | quarkus-jvm | 4.67M | 32 ms | 10.4 GiB |
  | helidon-production (Helidon SE 27.0.0 on Níma, Java 27) | 3.50M | 54 ms | 11.4 GiB |
  | ktor | 794k | | |
  | spring-boot (Tomcat, JDK 25, virtual threads) | 363k | 68 ms | |

  Vert.x has no HTTP/2 or HTTP/3 entry; its entry runs HTTP/1.1 profiles only. — [HttpArena results data](https://github.com/MDA2AV/HttpArena/tree/main/site/data/results); [HttpArena framework metadata](https://github.com/MDA2AV/HttpArena/tree/main/frameworks)

#### Specific stacks named in the question
- **Cloudflare quiche:** Engineering results are in the 2020 post above. In NOMS 2026, quiche was the slowest server (2.2 Gbit/s single-connection bulk goodput). quiche is what Netty's QUIC codec, Jetty and Vert.x 5.1 use (section 4). — [Cloudflare blog](https://blog.cloudflare.com/accelerating-udp-packet-transmission-for-quic/); [Späth et al., NOMS 2026](https://www.net.in.tum.de/fileadmin/bibtex/publications/papers/spaethj-noms2026.pdf)
- **LiteSpeed/lsquic:** I found only a dated Plesk comparison: OpenLiteSpeed 1.6.4 against nginx 1.16.1 with Cloudflare's quiche patch on one shared VM. LiteSpeed's HTTP/3 was faster and used less CPU and memory. I didn't fetch this page in full; the summary came via search. Independently, WWW '24 found OpenLiteSpeed's QUIC 18% faster than nginx-quic, and Jaeger 2023 ranked LSQUIC among the fastest. — [Plesk blog](https://www.plesk.com/blog/various/litespeed-vs-nginx/); [Zhang et al., WWW '24](https://arxiv.org/html/2310.09423v2); [Jaeger et al., 2023](https://ar5iv.labs.arxiv.org/html/2309.16395)
- **nginx QUIC:** nginx 1.25.0 mainline, released 2023-05-23, was the first with "experimental HTTP/3 support". In HttpArena, nginx is the one entry whose HTTP/3 beats its HTTP/2 (4.85M against 3.41M). — [nginx.org 2023 news](https://nginx.org/2023.html); [HttpArena results data](https://github.com/MDA2AV/HttpArena/tree/main/site/data/results)
- **h2o:** It leads HttpArena's HTTP/2 TLS baseline at 16.1M req/s, and h2o-mruby reaches 3.04M req/s on HTTP/3. h2o's own benchmark page compares web servers using h2load for HTTP/2 and wrk for HTTP/1.1; it is old, and I didn't fetch it in full. — [HttpArena results data](https://github.com/MDA2AV/HttpArena/tree/main/site/data/results); [H2O benchmarks](https://h2o.examp1e.net/benchmarks.html)
- **Caddy (quic-go):** HttpArena shows 375k req/s on HTTP/2 and 361k on HTTP/3; Caddy is listed as an "infrastructure" entry. A 2020 community h2load test compared Caddy with nginx over HTTP/2 and HTTP/3; I didn't fetch its numbers. — [HttpArena results data](https://github.com/MDA2AV/HttpArena/tree/main/site/data/results); [Caddy community, 2020](https://caddy.community/t/caddy-v2-http-2-http-3-benchmarks/8052)
- **msquic:**
  - The README lists XDP kernel bypass, UDP send/receive coalescing and RSS support. SecNetPerf compares QUIC against TCP/TLS directly.
  - ANRW '25 measured MsQuic at 53% of TCPLS at 25G, and NOMS 2026 measured 8.3 Gbit/s with GSO+GRO.
  - Kestrel on msquic shows a 3.53× HTTP/2-to-HTTP/3 ratio in HttpArena. — [MsQuic README](https://cdn.jsdelivr.net/gh/microsoft/msquic@main/README.md); [SecNetPerf docs](https://microsoft.github.io/msquic/msquicdocs/src/perf/readme.html); [Mukherjee et al., ANRW '25](https://par.nsf.gov//servlets/purl/10637806); [Späth et al., NOMS 2026](https://www.net.in.tum.de/fileadmin/bibtex/publications/papers/spaethj-noms2026.pdf)
- **Netty QUIC/HTTP/3 codecs:**
  - The incubator QUIC 0.0.8.Final release (2021) said GSO on `EpollDatagramChannel` improved throughput "quite a bit" in h2load benchmarks against Netty's HTTP/3 implementation, with no numbers.
  - The 0.0.9.Final release made GSO pluggable for any channel that supports it.
  - The only current public numbers are Ktor-on-Netty's in HttpArena (see above). — [Netty QUIC 0.0.8.Final](https://netty.io/news/2021/03/09/quic-0-0-8-Final.html); [Netty QUIC 0.0.9.Final](https://netty.io/news/2021/03/25/quic-0-0-9-Final.html)
- **Jetty 12 HTTP/3:** I found no benchmarks. Webtide said the team chose a working implementation over performance and called HTTP/3 "experimental and not recommended for production" (post from 2022, updated 2024-02-21). — [Webtide](https://webtide.com/?p=4270)
- **Kwik/Flupke (pure Java):** No benchmark numbers. The README says "performance is not optimal yet" because the focus has been correctness and features. — [Kwik README](https://github.com/ptrd/kwik)

### Inferences
- **Where HTTP/3's extra CPU comes from.** These sources compound, which explains why a 2× to 3× request/response gap is typical in 2026 and why stacks that batch aggressively (nginx, zix, agensio, fib) close it:
  - Per-packet AEAD and header protection, against TLS records amortized over TCP segments, often with kTLS.
  - User-space ACK generation and processing, with ACKs encrypted, against kernel-handled TCP ACKs.
  - One syscall and one skb per roughly 1.2 to 1.5 KB datagram unless GSO/GRO and mmsg batching are used, against TCP's TSO/GRO.
  - Weaker NIC offload for UDP.
  - On the JVM, JNI or FFM crossings and copies between Java buffers and native quiche. This last point is my inference; no source measured it.
- **Probable load-generator ceiling.** HttpArena's top three HTTP/2 results (h2o, agensio, cardigan, 14.5 to 16.1M req/s) used only about 15 to 30 of the server's 64 logical CPUs. The top two HTTP/3 results (zix, agensio, about 8.6 to 9.1M req/s) used about 13 to 15. The leaderboard top is therefore probably limited by the load generator, not the server. Meleys should not read the top of that board as server capacity.
- **A realistic HTTP/3 goal.** "HTTP/3 within about 2× of our own HTTP/2 at equal CPU, and better than other JVM HTTP/3 entries" is a realistic, measurable target. Parity with HTTP/2 on the JVM would require native batching (GSO/GRO, sendmmsg/recvmmsg) and likely an optimized native QUIC core.
- **Memory numbers need a fixed heap.** JVM memory figures in shared-machine suites (10 to 12 GiB for Helidon and Quarkus in HttpArena) mostly reflect default heap sizing on a 256 GB host, not intrinsic footprint. Memory comparisons need explicit `-Xmx` and RSS accounting. The JVM-memory researcher covers this.

### Gaps
- I found no published Netty HTTP/3 vs Netty HTTP/2 benchmark, no Jetty 12.1 HTTP/3 benchmark, no Kwik/Flupke benchmark, and no recent independent LiteSpeed/lsquic HTTP/3 server benchmark.
- I didn't retrieve the Microsoft netperf dashboard's actual numbers.
- I couldn't verify the often-quoted Google SIGCOMM 2017 server CPU ratio of QUIC to TCP+TLS.
- HttpArena gives no error bars (best of 3 runs of 5 s), and its HTTP/2 and HTTP/3 profiles use different connection and stream shapes, so its ratios are indicative only.

## 4. Which QUIC/HTTP/3 server implementations are available on the JVM in 2026, what do they mean for Linux/macOS/Windows benchmarking, and can the JDK HttpClient (JEP 517) serve as a test client?

### Takeaway
Server-side HTTP/3 on the JVM in October 2026 comes in three families:
1. Netty 4.2's QUIC codec (JNI to Cloudflare quiche) plus its HTTP/3 codec, which left incubation in 4.2.2 (June 2025). Vert.x 5.1, Quarkus (experimental), Reactor Netty/Spring (experimental) and Ktor (experimental) build on it.
2. Jetty 12.1, which wraps quiche through FFM on JDK 22+ or JNA.
3. Pure-Java Kwik plus Flupke, dual-licensed GPL-3.0/LGPL-3.0, where performance hasn't been the focus.

OpenJDK has no server or public QUIC API. JEP 517 delivered an HTTP/3 client in JDK 26 (GA 2026-03-17), on an internal QUIC stack over `DatagramChannel`, and JDK 27 (GA 2026-09-15) added no QUIC JEP. Native quiche builds exist for Linux x64/aarch64, macOS x64/aarch64 and Windows x64. However, GSO-accelerated UDP sending is Linux/epoll-only in Netty, and Java's own UDP API has no GSO/GRO. Cross-OS HTTP/3 results will therefore differ for structural reasons. The JDK HttpClient works as a functional and interop HTTP/3 client, not as a load generator.

### Cited Findings

#### Netty (QUIC codec + HTTP/3 codec)
- Netty 4.2.2.Final (2025-06-05) graduated HTTP/3 from the incubator (#15048). The latest listed release was 4.2.19.Final, dated 2026-10-04. — [Netty 4.2.2 release notes](https://netty.io/news/2025/06/05/4-2-2.html)
- Netty 4.2's `Http3` helper class provides `newQuicServerCodecBuilder()`, `newQuicClientCodecBuilder()` and `newRequestStream`. — [Netty 4.2 Http3 Javadoc](https://netty.io/4.2/api/io/netty/handler/codec/http3/Http3.html)
- The native QUIC artifact `io.netty:netty-codec-native-quic` is published with classifiers `linux-x86_64`, `linux-aarch_64`, `osx-x86_64`, `osx-aarch_64` and `windows-x86_64` (Maven Central listing for 4.2.1.Final; 4.2.18.Final also exists). — [Maven Central search](https://search.maven.org/solrsearch/select?q=g:io.netty+AND+a:netty-codec-native-quic&core=gav&rows=3&wt=json); [Maven Central artifact](https://central.sonatype.com/artifact/io.netty/netty-codec-native-quic/4.2.18.Final)
- GSO in Netty QUIC:
  - Enabled through `QuicChannelOption.SEGMENTED_DATAGRAM_PACKET_ALLOCATOR`.
  - `EpollQuicUtils.newSegmentedAllocator` builds the allocator for `EpollDatagramChannel` (max segments 1 to 64, default 10). It returns `NONE`, meaning no GSO, when `SegmentedDatagramPacket.isSupported()` is false.
  - GSO therefore applies only with the Linux epoll transport. — [EpollQuicUtils](https://netty.io/4.2/api/io/netty/handler/codec/quic/EpollQuicUtils.html); [SegmentedDatagramPacketAllocator](https://netty.io/4.2/xref/io/netty/handler/codec/quic/SegmentedDatagramPacketAllocator.html)
- Quarkus HTTP/3 (`quarkus-http3`):
  - Explicitly "experimental", using quiche "provided through Netty".
  - Supports the five classifiers above. Maven selects the matching native library automatically; Gradle needs the classifier declared manually.
  - Configuration: `quarkus.http3.enabled`, `quarkus.http3.alt-svc` and `quarkus.http3.alt-svc-max-age`.
  - Shares the HTTPS port over UDP. Limitations include UDP forwarding in containers, firewalls that block UDP, and no WebSocket over HTTP/3. — [Quarkus HTTP/3 reference (main)](https://quarkus.io/version/main/guides/http3-reference)
- Vert.x 5.1:
  - The "What's new" post (dated 2026-05-26, Julien Viet) adds HTTP/3 and a standalone `QuicServer`/`QuicClient`, built on "Netty and QUICHE".
  - A hybrid server can bind TCP for HTTP/1.x and HTTP/2 plus UDP for HTTP/3. The client can follow Alt-Svc, and the QUIC datagram extension (RFC 9221) is supported.
  - The post doesn't label HTTP/3 experimental and makes no performance claims.
  - A search summary dated the 5.1.0 release announcement to 27 June 2026; I didn't verify that. — [Vert.x 5.1 blog](https://vertx.io/blog/whats-new-in-vert-x-5-1/)
- Reactor Netty 1.2 (Reactor 2024.0 release train, 2024-11-26) added experimental HTTP/3 usable from Spring Boot 3.4 and Spring Cloud Gateway. At the time it required `io.netty.incubator:netty-incubator-codec-http3` 0.0.28.Final, which is outdated now that HTTP/3 has moved into Netty 4.2. — [Spring blog](https://spring.io/blog/2024/11/26/http3-in-reactor-2024)
- Ktor 3.x on Netty enables HTTP/3 with `enableHttp3()` under `@OptIn(ExperimentalKtorApi::class)`, as used in HttpArena's Ktor entry. That entry runs JDK 25 and pins `netty-transport-native-epoll` 4.2.16.Final. — [HttpArena Ktor Application.kt](https://github.com/MDA2AV/HttpArena/blob/main/frameworks/ktor/src/main/kotlin/com/httparena/Application.kt); [HttpArena Ktor versions](https://github.com/MDA2AV/HttpArena/blob/main/frameworks/ktor/gradle/libs.versions.toml)

#### Jetty 12.1
- Jetty's QUIC support comes "by default via Jetty classes that wrap the Quiche native library". The foreign-function binding is used on JDK 22+, older JDKs fall back to JNA, and native access must be granted with `--enable-native-access=ALL-UNNAMED` or `org.eclipse.jetty.quic.quiche.foreign`. — [Jetty 12.1 HTTP/3 client docs](https://jetty.org/docs/jetty/12.1/programming-guide/client/http3.html)
- `jetty-http3-server` is published at 12.1.12. — [Maven Central jetty-http3-server](https://central.sonatype.com/artifact/org.eclipse.jetty.http3/jetty-http3-server/)
- Jetty's native quiche builds use quiche 0.30.0 and target Linux x86-64, Linux aarch64, macOS x86-64, macOS aarch64 and Windows x86-64. The documented build command is `cargo build --features ffi,qlog`, and the documented output path is `target/debug/...`. — [jetty-project/jetty-quiche-native](https://github.com/jetty-project/jetty-quiche-native)
- Webtide's reason for using native code: QUIC's TLS use "requires new APIs that OpenJDK doesn't provide", so QUIC couldn't be done in pure Java. HTTP/3 has existed since Jetty 10.0.8/11.0.8, and the post called it "experimental and not recommended for production use" (2022, updated 2024). — [Webtide](https://webtide.com/?p=4270)

#### Pure Java and other JVM stacks
- Kwik:
  - "An implementation of the QUIC protocol in (100%) Java", client and server (server since May 2021).
  - Supports RFC 9000, QUIC v2 (RFC 9369), version negotiation (RFC 9368), Datagram (RFC 9221) and a post-quantum hybrid key agreement (listed in the README as RFC 10024).
  - The server doesn't support connection migration yet. HTTP/3 comes from Flupke.
  - "performance is not optimal yet"; not tested in extreme network conditions; its TLS library hasn't been security-reviewed.
  - Dual-licensed GPL-3.0/LGPL-3.0. Latest tags observed on 2026-10-21: Kwik v0.12.1, Flupke v0.9.4. — [Kwik README](https://github.com/ptrd/kwik); [Flupke repo](https://github.com/ptrd/flupke)
- Kwik is the only JVM implementation registered in the quic-interop-runner. The list has 17 entries: aioquic, chrome, go-x-net, haproxy, kwik, lsquic, msquic, mvfst, neqo, nginx, ngtcp2, picoquic, quic-go, quiche, quinn, s2n-quic and xquic. — [implementations_quic.json](https://github.com/quic-interop/quic-interop-runner/blob/master/implementations_quic.json)
- Ensō (Clojure/Java) runs "libquiche 0.29.3 + our JNI shim + our pure-Java HTTP/3 layer" and reports 39/49 h3spec passes. — [Ensō HTTP/3 conformance doc](https://cljdoc.org/d/com.s-exp/enso/1.0.0-alpha56/doc/http-3-conformance)
- HttpArena's "busker" (Clojure) uses a native shim for HTTP/3. Its recorded HTTP/3 result is 127 req/s, which looks like a broken run. — [busker meta.json](https://github.com/MDA2AV/HttpArena/blob/main/frameworks/busker/meta.json); [HttpArena results data](https://github.com/MDA2AV/HttpArena/tree/main/site/data/results)

#### OpenJDK: JEP 517 and the JDK's internal QUIC
- JEP 517, "HTTP/3 for the HTTP Client API", is Closed/Delivered in JDK 26 (owner Daniel Fuchs; updated 2026-01-21).
  - Non-goals include "a public API for QUIC", third-party security providers, and "a server-side HTTP/3 implementation".
  - HTTP/2 stays the default; HTTP/3 is opt-in through `HttpClient.Version.HTTP_3` or `H3_DISCOVERY` with `Http3DiscoveryMode.ALT_SVC` or `HTTP_3_URI_ONLY`.
  - Path MTU Discovery isn't implemented, and only the default SunJSSE provider is supported.
  - It was interop-tested against Netty, quic-go, quiche, Neqo and nghttp3, and supports RFC 9368/9369. — [JEP 517](https://openjdk.org/jeps/517)
- JDK 26 reached General Availability on 17 March 2026 with JEP 517. JDK 27 reached GA on 15 September 2026 with JEPs 523, 527, 531 to 534, and 536 to 538, none of them QUIC-related. — [JDK 26](https://openjdk.org/projects/jdk/26/); [JDK 27](https://openjdk.org/projects/jdk/27/)
- inside.java (Jaikiran Pai, 2025-10-22) said the implementation "has not seen much usage outside of the JDK development team" and asked for early-access feedback. For the first request, the client tries TCP (HTTP/2) and UDP (HTTP/3) and uses whichever succeeds first. — [inside.java HTTP/3 article](https://inside.java/2025/10/22/http3-support/); [inside.java quality heads-up](https://inside.java/2025/10/30/quality-heads-up/)
- The JDK's internal QUIC endpoint, `jdk.internal.net.http.quic.QuicEndpoint`, "encapsulate[s] a DatagramChannel". The JDK test library contains `QuicStandaloneServer` and `Http3TestServer`, built on internal APIs such as `jdk.internal.net.quic.QuicTLSContext`. These are test-only, not supported server APIs. — [QuicEndpoint.java](https://github.com/openjdk/jdk/blob/master/src/java.net.http/share/classes/jdk/internal/net/http/quic/QuicEndpoint.java); [QuicStandaloneServer.java](https://github.com/openjdk/jdk/blob/master/test/jdk/java/net/httpclient/lib/jdk/httpclient/test/lib/quic/QuicStandaloneServer.java); [Http3TestServer.java](https://github.com/openjdk/jdk/blob/master/test/jdk/java/net/httpclient/lib/jdk/httpclient/test/lib/http3/Http3TestServer.java)
- Java's UDP API offers no GSO/GRO option; see the DatagramChannel and ExtendedSocketOptions entries in section 2. quic-go's GSO is Linux-only (kernel 4.18+), with Windows support still future work. — [JDK 26 DatagramChannel](https://docs.oracle.com/en/java/javase/26/docs/api/java.base/java/nio/channels/DatagramChannel.html); [JDK 26 ExtendedSocketOptions](https://docs.oracle.com/en/java/javase/26/docs/api/jdk.net/jdk/net/ExtendedSocketOptions.html); [quic-go optimizations](https://quic-go.net/docs/quic/optimizations/)

### Inferences
- **The realistic paths for Meleys:**
  - (a) Build on Netty's QUIC codec, with quiche over JNI and BoringSSL inside the native library. This is the most widely used JVM path, with five-platform natives and Linux GSO.
  - (b) Bind quiche or another native QUIC library through FFM, as Jetty does. That gives full control over batching and buffers.
  - (c) A pure-Java QUIC on `DatagramChannel`. It is portable, but has no GSO/GRO, needs its own QUIC-TLS integration because the JDK's is internal, and Kwik's license (GPL/LGPL) is hard to fit an Apache-2.0 project. That last point is my reading, not legal advice.

  Benchmark (a) and (b) as baselines before deciding on (c).
- **Check Jetty's quiche build profile first.** The jetty-quiche-native README documents a plain `cargo build` with output in `target/debug`, which would be an unoptimized debug build. If the shipped binaries really are debug builds, Jetty HTTP/3 numbers would understate quiche badly. Inspect the shipped library before using Jetty as a baseline. This is an inference from build notes, not verified against the shipped JAR.
- **Cross-OS benchmarking:**
  - Linux (epoll or io_uring, GSO, fq) is the only platform where JVM HTTP/3 numbers mean anything for "fastest framework" claims.
  - macOS (kqueue) and Windows (NIO) lack the GSO path in Netty, and Java exposes neither Windows USO/URO nor macOS batching. Treat them as correctness and regression tracks with their own baselines, and never compare them directly with Linux.
  - Per-OS natives (five classifiers) must be part of the CI matrix.
- **Using the JDK 26+ HttpClient.** It is useful as an independent HTTP/3 interop client for functional tests, since it is a third QUIC implementation besides quiche and ngtcp2. Use `HTTP_3_URI_ONLY` to prevent silent fallback to HTTP/2. It isn't suitable as a load generator: it has no rate control or latency histograms, its CPU efficiency is unknown, PMTUD is missing, and public usage was limited as of late 2025. Pre-JDK-26 runtimes can't be used.
- **No usable JDK server today.** OpenJDK's test-only QUIC server shows that server-side QUIC code exists internally, but it isn't usable or supported. Nothing in JDK 26 or 27 suggests a public QUIC API soon.

### Gaps
- I didn't verify Helidon (4.x/27.x) or Micronaut HTTP/3 status. HttpArena's Helidon entries subscribe only to HTTP/1.1 and HTTP/2, which isn't proof of absence.
- I didn't verify the quiche license, the exact quiche and BoringSSL versions bundled in Netty 4.2.19, or whether Jetty's shipped natives are release builds.
- I found no published per-OS (macOS or Windows) QUIC throughput numbers for any JVM stack.
- I didn't confirm whether the JDK's internal QUIC does any send batching (it sits on DatagramChannel, which suggests not).

## 5. Does any public multi-framework suite benchmark HTTP/2 or HTTP/3? (And confirm TFB does not.)

### Takeaway
Yes. HttpArena (http-arena.com; github.com/MDA2AV/HttpArena) is an active multi-framework suite, last updated 2026-10-20. It has HTTP/2 profiles over TLS and h2c, HTTP/3 profiles, gRPC, and gateway profiles including gateway-h3. About 59 enabled entries run HTTP/2 over TLS and about 40 run HTTP/3. The JVM is thinly represented in HTTP/3: only Ktor (Netty) and one Clojure engine. TechEmpower Framework Benchmarks runs only wrk over HTTP/1.1 and has no HTTP/2 or HTTP/3 test type. Other public HTTP/2 and HTTP/3 comparisons cover web servers or QUIC stacks, not frameworks, and are mostly old.

### Cited Findings
- **HttpArena basics:**
  - Describes itself as an "HTTP framework benchmark platform" with "30 test profiles. 64-core dedicated hardware. Same conditions for every framework".
  - H/2 profiles: `baseline-h2`, `static-h2`, `baseline-h2c`, `json-h2c`. H/3 profiles: `baseline-h3`, `static-h3`. Gateway: `gateway-64`, `gateway-h3`. Also `unary-grpc` and `unary-grpc-tls`.
  - PR commands (`/benchmark -f <framework>`) return per-profile RPS, p99, CPU and memory, and per-family badges (h1, h2, h3, gw, grpc, ws) are available. — [HttpArena README](https://github.com/MDA2AV/HttpArena)
- **Leaderboard counts:** The homepage lists 175 entries across 27 profiles, updated 2026-10-20, which differs from the README's "30 test profiles". It runs on an AMD Threadripper PRO 3995WX (64 cores) with containers pinned to dedicated cores. RPS is the best of three runs, and latency, CPU and memory come from that run. — [HttpArena homepage](https://www.http-arena.com/)
- **My analysis of repo metadata (commit 3717e256, 2026-10-20):** 235 framework directories, 198 enabled. 59 enabled entries subscribe to TLS HTTP/2 profiles and 40 to HTTP/3 profiles. Results exist for 57 entries on baseline-h2-256, 46 on baseline-h2c-256 and 32 on baseline-h3-64. — [HttpArena frameworks](https://github.com/MDA2AV/HttpArena/tree/main/frameworks); [HttpArena results](https://github.com/MDA2AV/HttpArena/tree/main/site/data/results)
- **JVM entries on HTTP/2:** helidon-production and helidon-tuned, quarkus-jvm (plus quarkus-jvm-h2c), spring-boot, cardigan, ktor, neton (Kotlin), and several Clojure engines. JVM entries on HTTP/3: ktor and ktor-tuned (Netty), and busker (Clojure). The vertx entry runs HTTP/1.1 profiles only. — [HttpArena framework metadata](https://github.com/MDA2AV/HttpArena/tree/main/frameworks)
- **HttpArena rules:**
  - HTTP/3 "standard" entries "must use the framework native QUIC/HTTP3 support with default configuration". "Tuned" entries may tune QUIC parameters, congestion control and UDP buffer sizes.
  - HTTP/2 standard entries must use the framework's standard HTTP/2 + TLS configuration, while tuned entries may change TLS settings, stream limits and window sizes. — [HttpArena baseline-h3](https://github.com/MDA2AV/HttpArena/blob/main/site/content/docs/test-profiles/h3/baseline-h3/implementation.md); [HttpArena baseline-h2](https://github.com/MDA2AV/HttpArena/blob/main/site/content/docs/test-profiles/h2/baseline-h2/implementation.md)
- **HTTP/3 validation:** Uses the ngtcp2-built h2load because "OpenSSL cannot speak QUIC". Before this was added, an HTTP/3-only entry "ran no assertions at all and still exited 0". — [HttpArena baseline-h3 validation](https://github.com/MDA2AV/HttpArena/blob/main/site/content/docs/test-profiles/h3/baseline-h3/validation.md)
- **Positioning:** One framework's integration README calls HttpArena "the spiritual successor to TechEmpower". This is secondhand. — [mq-bridge HttpArena README](https://docs.rs/crate/mq-bridge/0.3.9/source/scripts/httparena/README.md)
- **TFB:** The toolset in TechEmpower/FrameworkBenchmarks master (latest commit observed 2026-03-23) drives every test with `wrk` scripts (`concurrency.sh`, `pipeline.sh`, `query.sh`). My grep of `toolset/` found no h2load, HTTP/2, h2c, QUIC or HTTP/3 references. — [TFB toolset/wrk](https://github.com/TechEmpower/FrameworkBenchmarks/tree/master/toolset/wrk)
- **TFB (independent confirmation):** "the TechEmpower plaintext test is http 1.1 and NOT http 2". — [loom-dev, Nov 2023](https://mail.openjdk.org/pipermail/loom-dev/2023-November/006271.html)
- **Other public comparisons:** These compare web servers or protocol stacks, not frameworks:
  - H2O's benchmark page and http2benchmark.org (h2load for HTTP/2, wrk for HTTP/1.1).
  - The 2020 Caddy community test.
  - The quic-interop-runner's goodput and cross-traffic measurements across QUIC implementations. — [H2O benchmarks](https://h2o.examp1e.net/benchmarks.html); [http2benchmark.org](https://http2benchmark.org/); [Caddy community](https://caddy.community/t/caddy-v2-http-2-http-3-benchmarks/8052); [interop runner](https://github.com/quic-interop/quic-interop-runner)

### Inferences
- **What HttpArena is good for.** It is the only public venue where Meleys can be ranked against other frameworks on HTTP/2 and HTTP/3, and JVM HTTP/3 entries are scarce. Its methodology doesn't support Meleys' tail-latency and memory claims: 5-second runs, best of 3, closed-loop h2load at saturation, loopback on one host, a probable load-generator ceiling at the top, and different stream shapes for HTTP/2 and HTTP/3. Use it for external visibility and keep an in-house, two-host, open-model harness for engineering decisions.
- **Benchmark config is part of the result.** HttpArena has "standard" and "tuned" tiers, and Meleys' defaults (window sizes, stream limits, UDP buffers) will be judged as shipped. Default configuration is therefore a performance feature.

### Gaps
- I didn't find HttpArena's operators, funding, history (start date) or result-review process.
- I didn't verify whether Microsoft's crank-based ASP.NET benchmarks publish multi-framework HTTP/2 or HTTP/3 results, or whether "the-benchmarker/web-frameworks" has any HTTP/2 mode.

## 6. Which conformance and interop suites should run alongside performance tests?

### Takeaway
Run conformance gates before any performance run:
- h2spec for HTTP/2 and HPACK. It is old (written against RFC 7540) but still standard, and emits JUnit reports.
- h3spec for QUIC and HTTP/3 error cases.
- Cloudflare's h3i for scripted, rule-bending HTTP/3 server tests.
- The quic-interop-runner, if Meleys owns a QUIC stack: transport interop matrix plus goodput and cross-traffic measurements.
- A "validate before benchmark" step like HttpArena's: protocol negotiation and anti-cheat checks.

Also cross-test against independent clients: the JDK 26+ HttpClient, curl, and h2load itself.

### Cited Findings
- **h2spec:**
  - "a conformance testing tool for HTTP/2 implementation" compliant with RFC 7540 and RFC 7541.
  - Flags include `-t` (TLS), `-k`, `-S` (strict: run all cases including strict ones), `-j` (JUnit report), `--max-header-length` and `-o` (timeout). Individual spec sections can be selected, such as `h2spec http2/6.3`.
  - Available as binary releases and a Docker image. The latest tag observed on 2026-10-21 was v2.6.0. — [h2spec README](https://github.com/summerwind/h2spec)
  - RFC 9113 (June 2022) obsoletes RFC 7540 and RFC 8740, so h2spec's spec references are out of date. — [RFC 9113](https://www.rfc-editor.org/rfc/rfc9113)
- **h3spec:** A "test tool for error cases of QUIC and HTTP/3" by Kazu Yamamoto (Haskell; BSD-3-Clause; version 0.1.12 uploaded 2025-03-31). Ensō reported 39/49 passing with libquiche 0.29.3, with all failures in the QUIC transport layer. — [h3spec on Hackage](https://hackage.haskell.org/package/h3spec); [Ensō HTTP/3 conformance](https://cljdoc.org/d/com.s-exp/enso/1.0.0-alpha56/doc/http-3-conformance)
- **h3i (Cloudflare quiche):** "a highly configurable HTTP/3 client that can bend RFC rules in order to test the behavior of servers". It can open, fin, stop or reset streams at any point and send arbitrary legal or illegal frames, as a CLI or as a library with sync and async (tokio-quiche) clients. It isn't intended for production use. — [h3i docs](https://docs.rs/crate/h3i/0.7.0)
- **quic-interop-runner:**
  - Test cases: handshake, transfer (flow control and multiplexing), chacha20, keyupdate, retry, resumption, zerortt, http3, multiconnect (handshake loss), v2, rebind-port, rebind-addr and connectionmigration. Versionnegotiation is currently disabled.
  - Endpoints are Docker images driven by the `TESTCASE` environment variable. Exit code 127 means "unsupported test case", and linux/amd64 images are required for the online runner.
  - Supports `SSLKEYLOGFILE` and `QLOGDIR`. Live results are at interop.seemann.io. — [interop runner README](https://github.com/quic-interop/quic-interop-runner); [quic.md](https://github.com/quic-interop/quic-interop-runner/blob/master/quic.md)
- **JEP 517 interop peers:** The JDK team interop-tested its HTTP/3 client against Netty, quic-go, quiche, Neqo and nghttp3, which gives a ready peer list. — [JEP 517](https://openjdk.org/jeps/517)
- **HttpArena validation:**
  - For h2: confirms HTTP/2 was negotiated rather than a fallback to HTTP/1.1, uses randomized query parameters to catch hard-coded responses, and runs shared TLS checks.
  - For h3: four GET requests over QUIC with ALPN h3.
  - h2c profiles verify that the h2c port refuses HTTP/1.1. — [HttpArena baseline-h2 validation](https://github.com/MDA2AV/HttpArena/blob/main/site/content/docs/test-profiles/h2/baseline-h2/validation.md); [HttpArena baseline-h3 validation](https://github.com/MDA2AV/HttpArena/blob/main/site/content/docs/test-profiles/h3/baseline-h3/validation.md); [HttpArena H/2 profiles](https://github.com/MDA2AV/HttpArena/blob/main/site/content/docs/test-profiles/h2/_index.md)
- **Abuse resistance:** HTTP/2 must survive the "Rapid Reset" pattern (CVE-2023-44487), in which clients open and cancel streams at high rates. h2spec doesn't cover this. — [Cloudflare technical breakdown](https://blog.cloudflare.com/technical-breakdown-http2-rapid-reset-ddos-attack/)

### Inferences
- **Suggested CI order for Meleys:**
  1. h2spec in strict mode with a JUnit report, for h2 and h2c.
  2. h3spec plus a set of h3i scripts (bad frames, stream resets, oversized headers, QPACK edge cases).
  3. Interop runs with independent clients: the JDK 26+ HttpClient in `HTTP_3_URI_ONLY` mode, curl, h2load, and quic-go/ngtcp2-based clients.
  4. HttpArena-style anti-cheat validation.
  5. Performance runs.
- **Interop runner listing.** If Meleys builds its own QUIC transport, it should publish an interop-runner Docker endpoint (as Kwik does) to get continuous cross-implementation coverage. If it builds on Netty/quiche, transport interop is inherited, and the HTTP/3 and QPACK layers still need h3spec/h3i coverage.
- **Robustness under load.** Abuse-resistance scenarios (rapid reset, CONTINUATION and SETTINGS floods, slow readers against flow control) are worth running as robustness-under-load tests next to throughput runs. A fast framework that collapses under these isn't production-grade.

### Gaps
- I didn't verify whether h2spec is still maintained (latest release date) or whether an RFC 9113-updated fork exists.
- I didn't research QPACK-specific offline interop corpora or other HTTP/3 fuzzers beyond h3spec and h3i.
- I found no public conformance results for Netty's or Jetty's HTTP/3 servers.
