# VASTlint module for Prebid Server: feature report and evidence

Date: 2026-10-26. Audience: Prebid Server maintainers and host engineers deciding whether to review and run the module. Everything measured here ran on Linux: the load rig inside a Linux arm64 container (Debian 12, Go 1.25, 8 vCPU, 10 GB) against a fake bidder serving 1,449 real submitted VAST documents through the unmodified adprime adapter, plus benchmarks in the same container and on GitHub-hosted x86_64 runners. No creative markup is stored in this folder.

## What the module is

`modules/vastlint/validation` is a `raw_bidder_response` hook. For every video bid a bidder returns it validates the VAST, VMAP or DAAST markup in process with the vastlint rule catalog (246 rules, IAB VAST 2.0 through 4.3 plus the 4.4 working draft, VMAP 1.0, DAAST 1.0), then:

- in **monitor** mode stamps the bid with `bid.ext.vastlint` and reports a per-bid analytics tag;
- in **reject** mode also removes bids whose findings include a rule in the host's `reject_on` list;
- in **async** mode queues the creative and returns at once, so the auction never waits for the engine.

An optional verdict cache keyed by creative, rules and engine version makes repeated creatives free. Everything the module learns leaves through channels Prebid Server already has: analytics tags, the bid extension, the framework's module metrics, the debug trace and the server log. No core change, no new metric family, no external service.

## Why it matters: what vastlint is and what it finds

[vastlint](https://vastlint.org) is an open-source (Apache 2.0) validator for the IAB Tech Lab video ad formats: VAST 2.0 through 4.3 with the 4.4 working draft, VMAP 1.0 and DAAST 1.0, plus the SIMID 1.0 to 1.2 envelope, Open Measurement (OMID) verification blocks and the CTV Addendum 2024. It has been developed in the open since April 2026 (30 releases, 0.16.0 is current), ships as a Rust core with no runtime dependencies, and is used through a CLI, a hosted validator and MCP server at vastlint.org, VS Code and Chrome extensions, a Docker image, and bindings for Go, Java, Erlang/Elixir and Python. Release artifacts carry SLSA Build Level 2 provenance. Public usage so far: about 12,400 downloads of the core crate on crates.io (9,700 in the last three months) and about 2,600 downloads a month of the npm package.

### IAB Tech Lab already runs on it

At IAB Tech Lab's [2026 Ad Ops Workshop for Advanced TV](https://iabtechlab.com/event/ad-ops-workshop-for-advanced-tv-2026/) (August 2026), Tech Lab presented the [Next-Gen VAST Tester](https://iab-tech-lab-vast-tester.vastlint.org/), a rebuild of its VAST Tester on vastlint. Tech Lab's own recap, written by Brad Pipkin, Director of Product, Advanced TV ([CTV Ad Ops Grows Up](https://iabtechlab.com/ctv-ad-ops-grows-upfrom-promising-capabilities-to-repeatable-operations/)), says: "The tester, built on the open-source VASTlint project, evaluates structural validity, environment compatibility, playback, and tracking behavior. It also supports the CTV Ad Portfolio and VAST 4.4 features." Of the attendees polled, all planned to use or evaluate the tester (creative intake or pre-launch QA, automated workflow integration, or evaluation first) and none chose to keep their existing tools or named an alternative. The rebuild is submitted upstream as [IABTechLab/VAST-Tester#47](https://github.com/IABTechLab/VAST-Tester/pull/47). The same rule set, run at the bidder response instead of in a tester, is what this module adds to Prebid Server.

### The rule catalog

246 rules, each with a stable id, a severity, a spec reference and a fix description ([rule reference](https://vastlint.org/docs/rules), [how rules are derived](https://vastlint.org/docs/methodology/)). By source:

| Source | Rules |
|---|---|
| VAST specification prose (RFC 2119 normative language) | 79 |
| VAST XSD schemas | 39 |
| IAB VMAP 1.0 | 24 |
| IAB DAAST 1.0 (spec and XSD) | 26 |
| IAB CTV Ad Portfolio and CTV Addendum 2024 | 33 |
| Revenue-impact checks (findings that cost the impression or its measurement) | 14 |
| IAB SIMID | 12 |
| W3C XML 1.0, RFC 3986 URI syntax, inferred from spec examples | 17 |

Fifteen rules are flagged `revenue_impact`: missing or unreachable media, missing Impression, broken quartile tracking, HTTP resources on HTTPS inventory, malformed XML, wrapper without a tag URI, and similar. These are the ones a host would reject on; the rest are the ones a host would report.

### What broken video creatives look like in real traffic

The corpus used throughout this evaluation is 1,449 real VAST documents submitted to the public validator (median 9.4 KB). Replayed through Prebid Server as bids on Linux, 845,000 creatives in one run, the module found:

| Share of creatives | Finding |
|---|---|
| 31.9 percent | invalid (at least one error-level finding) |
| 8.4 percent | malformed XML (the player would fail before parsing an ad) |
| 0.8 percent | not VAST, VMAP or DAAST at all |
| 7.8 percent | no mezzanine file (CTV stitching cannot use the creative) |
| 7.7 percent | no quartile tracking events (completion cannot be measured) |
| 3.7 percent | an empty URL in a required element |
| 3.1 percent | no InLine Impression (the impression cannot be counted) |
| 3.0 percent | HTTP tracking URLs (blocked or mixed-content on HTTPS pages) |
| 2.2 percent | JSON-escaped markup (a serialization bug upstream) |

In reject mode with just two rules, parse error and missing InLine Impression, about 12 percent of the normal creatives in this feed are kept out of the auction (measured on the mixed feed: 31 percent dropped, of which 25 percent are the deliberately truncated creatives). The feed is submitted tags, so it skews toward documents people had trouble with; a host's own mix will differ, and the module reports the distribution per bidder so each host sees its own numbers.

### Why validate at the bidder response

Today a broken video creative is discovered by the player: the ad wins, the player fetches the media, fails, fires a VAST error code, and the publisher learns about it in aggregated player reports hours later, with no link back to the bidder or the rule that broke. By then the slot is gone and the discrepancy between the bidder's count and the publisher's is already booked. The `raw_bidder_response` stage is the one place where every video creative from every bidder passes through one process before a winner is chosen, so a host can:

- see, per bidder and per rule, what fraction of its video supply is unplayable or unmeasurable, through the analytics adapters it already runs;
- pass the verdict downstream in `bid.ext.vastlint`, so an ad server, a player wrapper or a later hook can decide with it;
- keep the worst creatives out of the auction per account, with the module's cost measured in this package.

### References

- IAB Tech Lab VAST: https://iabtechlab.com/standards/vast/
- IAB Tech Lab VMAP, DAAST, SIMID and Open Measurement: https://iabtechlab.com/standards/
- Rule reference with examples and fixes: https://vastlint.org/docs/rules
- Rule derivation methodology: https://vastlint.org/docs/methodology/
- The ten VAST errors that cost money (common errors guide): https://github.com/aleksUIX/vastlint/blob/main/docs/common-errors.md
- IAB Tech Lab workshop recap naming the vastlint-based tester: https://iabtechlab.com/ctv-ad-ops-grows-upfrom-promising-capabilities-to-repeatable-operations/
- Source: https://github.com/aleksUIX/vastlint and https://github.com/aleksUIX/vastlint-go

## Design in one diagram

```
bidder response ──► eligibility ──► [cache lookup] ──► engine (wasm | native) ──► tags + ext.vastlint ──► auction
   video bid        url/size/xml      hit: no engine     8 at a time, ctx deadline    reject: drop on reject_on
                    sample_percent    miss: validate     fail open on error/timeout
                                                         async: queue, return now
```

Engines: the default build embeds the vastlint engine compiled to WebAssembly (650 KB, SHA-256 pinned) and runs it with wazero, a pure Go runtime; `-tags vastlint_cgo` links the host-installed native library instead. Both produce identical findings; the module's tests run under both.

## Evidence

### Cost on the bid path and off it (native engine, 8 clients, 20 bids per response, verdict cache on)

Twenty video bids per response is an extreme density chosen to make the cost visible; divide by twenty for the per-creative figure. Three interleaved 60-second rounds, medians:

| | hooks off | async, off the bid path | monitor, on the bid path |
|---|---|---|---|
| auctions per second | 764 | 733 | 646 |
| p50 | 5.59 ms | 5.94 ms | 7.10 ms |
| p90 | 9.55 ms | 10.54 ms | 12.11 ms |
| p99 | 17.6 ms | 19.1 ms | 20.7 ms |
| CPU per auction | 4.12 ms | 4.64 ms | 4.93 ms |
| hook mean | 0 | 0.39 ms | 0.57 ms |
| hook timeouts per minute | 0 | 2 | 6 |
| resident memory, steady | 127 MB | 215 MB | 220 MB |
| OS threads, max | 15 | 24 | 23 |
| bids annotated | 0 | 0 | 100% |
| cache hit rate | | 99% | 99% |

Reading: off the bid path, the recommended deployment, the module costs about 0.5 ms of CPU and 0.35 ms of p50 per twenty-creative auction, touches no bid, and validates and reports every creative; per-bid annotation is the one thing it gives up. On the bid path it adds about 0.8 ms of CPU and 1.5 ms of p50 with every bid annotated and reject mode available. Memory grows by the cache (32 MiB allocated up front, plus its index). All nine runs returned HTTP 200 for every auction with zero panics. The feed loops 1,449 creatives, so the hit rate was 99 percent; production hit rates will be lower and the module can log them, which is the number to watch during a rollout.

### The common case: one video bid per response (8 clients, 40 s, verdict cache on)

| medians of two rounds | hooks off | monitor, on the bid path | reject, mixed feed |
|---|---|---|---|
| auctions per second | 5,678 | 6,010 | 7,859 |
| p50 | 0.93 ms | 0.90 ms | 0.76 ms |
| p90 | 1.85 ms | 1.61 ms | 1.37 ms |
| p99 | 3.56 ms | 2.88 ms | 2.53 ms |
| CPU per auction | 0.52 ms | 0.50 ms | 0.42 ms |
| hook mean | 0 | 35 µs | 27 µs |
| resident memory, steady | 93 MB | 156 MB | 160 MB |
| bids per request | 1.00 | 1.00 | 0.69 |

The first two columns are within noise of each other; the module's own cost for one cached video bid is the 35 µs hook. The reject run on the mixed feed (25 percent truncated, 25 percent URL markup, 50 percent normal) dropped 31 percent of bids as predicted with every auction HTTP 200.

### Per-call cost in the hook (Go benchmarks)

| | 1 bid, cache miss | 20 bids, cache miss (8-way fan-out) | 20 bids, all cached |
|---|---|---|---|
| Linux arm64 container (8 vCPU), wasm | 43 µs | 0.33 ms | 91 µs |
| Linux arm64 container, native | 56 µs | 0.36 ms | 93 µs |
| Linux x86_64 GitHub runner (4 vCPU AMD EPYC 9V45), wasm | 50 µs | 0.53 ms | 123 µs |
| Linux x86_64 GitHub runner, native | 56 µs | 0.44 ms | 125 µs |

Engine benchmarks in the container (vastlint-go, one document per call): small document 40 µs wasm against 51 µs native; 240 KB document 2.64 ms against 0.80 ms; small document with parallel callers 9.2 µs against 10.9 µs. On small documents the two engines cost the same; the native engine pulls ahead on large documents. Output: `results/pbs/linux-bench-docker.txt` (script `harness/linux/bench-linux.sh`).

### Resilience and a ten-minute soak

All in the Linux container with the verdict cache on. Every auction in every run returned HTTP 200; no run logged a panic or an engine error.

| Scenario | Outcome |
|---|---|
| Hook group timeout forced to 1 ms (wasm engine, 32 clients, one bid per response, 40 s) | 253,172 hook calls, 788 timed out (0.31 percent); the timed-out bids passed through unannotated and 99.7 percent of bids carried the annotation; p50 2.85 ms at 6,327 auctions per second |
| Every creative above `max_document_bytes` (300 KB against a 256 KB cap) | 48,604 of 51,753 calls were no-ops, the markup skipped before the engine and the bid kept and tagged `skipped_size`; resident memory flat (254 MB peak, 232 MB at the end) |
| Every creative malformed, reject mode on parse error | 520,844 bids, every one dropped, zero hook timeouts, p50 2.16 ms at 13,019 auctions per second |
| Ten-minute soak, monitor mode on the bid path, 8 clients, 20 bids per response | 331,625 auctions and 6.63 million creatives through the hook; 83 hook timeouts (0.02 percent); 99.98 percent of bids annotated; hook mean 0.67 ms; resident memory 203 MB at start, 222 MB steady, 255 MB peak, 215 MB at the end; 24 OS threads; cache hit rate 99 percent |

Reading: the fail-open contract holds under a budget twenty times tighter than the documented one, oversized markup never reaches the engine, reject mode removes exactly what it is told to, and ten minutes at full load leave memory where it started.

### Continuous integration

The fork carries a workflow that builds Prebid Server with the module on ubuntu-latest under Go 1.25 and 1.26, runs the module tests with the race detector under both engines (native library installed from the vastlint v0.16.0 release tarball), runs the vastlint-go parity suite, and prints the benchmarks: https://github.com/aleksUIX/prebid-server/actions/runs/40268500383, every step green. `./validate.sh --nofmt --cov --race 10`, the upstream CI command, passes on the full repository.

## Screenshots

All from the rig dashboard running in the Linux container, 2026-10-26.

1. `screenshots/01-linux-live-on-path.png`: the live view during an on-path run: throughput, client latency, hook outcomes, resident memory, the per-bid verdict feed and the server log.
2. `screenshots/02-linux-analysis-off-path-vs-baseline.png`: the Analysis tab for the off-path run against the hooks-off baseline of the same shape: anomaly checks, the module's cost per auction, whole-run charts, the bid-flow diagram and the error chart.
3. `screenshots/03-linux-analysis-on-path.png`: the Analysis tab for the on-path run, every bid annotated.
4. `screenshots/04-linux-findings-on-path.png`: the Findings tab for that run: rules fired with catalog descriptions and counts, invalid and malformed shares, skips.
5. `screenshots/05-linux-findings-reject-mixed-feed.png`: the Findings tab for the reject run on the mixed feed: drops by rule, skips by reason.
6. `screenshots/06-linux-analysis-reject-mixed-feed.png`: the Analysis tab for that reject run.

## Review before submission

An independent review pass over the module and the binding (fresh context, no access to the author's conclusions) ranked its findings; everything above LOW was fixed before the branch was published:

- Two shutdown panic paths in vastlint-go: the native engine closed its job channel while a call could still send on it, and the wasm engine could leave a nil slot in its pool. Fixed in vastlint-go v0.16.1 with a test that closes each engine under eight concurrent callers; the module also recovers panics in its own goroutines.
- Off-path workers could occupy the whole engine pool and starve on-path hooks: they now take at most half of it.
- Account `rules` replaced host `rules` instead of merging per rule id: now merged, documented, tested.
- The periodic report could miss changes that arrived after the last submission: it now gates on every counter.
- Smaller items: 32-bit overflow in the byte bounds, a queue-bytes floor below the document cap, an unreachable error branch, a dead field, byte-preserving annotation with sjson, a linear reject lookup, naming and interface assertion, and five new tests.

A second pass checked the module against the in-tree modules and Prebid's module guide: the README's execution plan is now a YAML mapping that Prebid Server's config loader accepts (a JSON string there fails at startup); config parsing uses `jsonutil` like the other modules; the analytics activity is `vast_validation` with `success-modify` on annotated bids; engine failures produce one hook warning per response instead of an error per bid; the periodic counters log is opt-in; the dead `enabled` field is gone; the shutdown-under-load test carries the `TestRace` prefix so `validate.sh --race` runs it; and the wasm engine returns a clean error instead of panicking on architectures without a compiler backend (vastlint-go v0.16.2).

## Where the raw data is

- `runs/*.json`: every Linux run, with per-second samples, the hook trace outcomes, the finding counts per rule and the computed analysis. File names carry the run name (`lx1-off`, `lx1-off-path-cache`, `lx1-on-path-cache` and so on for the three rounds; `lx*-off-1bid-c8` and `lx*-monitor-cache-1bid-c8` for the single-bid pairs; `lx-reject-mixed-cache-c8`; `lx-timeout-1ms-cache`, `lx-oversize-cache`, `lx-malformed-reject-cache`; `lx-soak-on-path-cache-10min`).
- `reports/linux-bench-docker.txt`: the container benchmark output, both engines.
- `reports/ENGINE-RC-SUMMARY-20261024.md`: the engine-level regression and benchmark report for the 0.16.0 release candidate.
- `screenshots/`: the six dashboard captures listed above.

The rig itself (fake bidder, load driver, dashboard, container scripts) lives with the vastlint evaluation tooling and is not part of this folder. No creative markup is stored here.
