# vastlint #181 release candidate: regression and benchmark report

Date: 2026-10-24. Baseline: origin/main at v0.15.3 (ce64a09). Candidate: PR #181 head 6556197 (main merged in). Machine: Apple M4, 10 cores, load average about 3 during timed runs. Corpus: all 3,367 documents retained by vastlint.org as of today (22.5 MB), exported to the private volume; no XML is stored in this folder.

The end-to-end evaluation inside a local Prebid Server, including the module, both engines, latency impact and resilience, is in `pbs/EVALUATION.md`.

## Verdict

Ready to merge and release. Every behavior change is the intended parser hardening, every changed verdict was independently confirmed, the fixer is byte-identical, all three engines agree on every document, and the cost is 5 to 13 percent of validation time depending on document size.

## Correctness

| Check | Result |
|---|---|
| Workspace tests, default features (CI command) | 542 pass |
| Workspace tests, all features | 542 pass, 1 ignored (pre-existing) |
| Doc tests | 5 pass, 1 ignored (pre-existing) |
| Clippy, all targets, all features, warnings denied | clean |
| rustfmt | clean |
| GitHub CI on the PR head before the merge commit | 17 of 17 checks pass |
| Repository fixtures through the CLI, baseline vs RC | 170 of 170 identical JSON and exit codes |
| Corpus through the CLI, baseline vs RC | 3,326 identical; 41 differ (see below) |
| Corpus through the engine, baseline vs RC | 3,327 identical; 40 differ (see below) |
| Fixer output, baseline vs RC | identical SHA-256 on 3,367 of 3,367 |
| RC cross-engine parity: native vs cgo vs wazero | identical issue sets on 3,367 of 3,367, zero traps |
| Baseline cross-engine parity: native vs cgo | identical on 3,367 of 3,367 |
| wasm edge inputs (4 MiB, depth 129, 100,001 elements, control char, U+FFFE, BOM, DOCTYPE, non-UTF-8, empty) | documents, never traps |
| wasm concurrency: 16 instances x 200 calls | byte-identical output |
| wasm catalog export | 246 unique rules, 15 revenue-impact, version 0.15.3 |

### The 40 changed verdicts, explained

- 36 documents already invalid: the finding moves from `VAST-2.0-root-element` (error) to `VAST-2.0-parse-error` (error). Validity unchanged.
- 4 documents move from valid to invalid: ids 1475, 2284, 3143, 5218. An independent parser (Python expat) rejects all four: one unclosed token, three invalid tokens. The baseline had been validating broken XML.
- Of the 40, expat agrees 39 are malformed. The one it accepts, id 2074, is an HTML document starting with `<!DOCTYPE`; it was already invalid (root element error) and now trips the DOCTYPE-unsupported policy instead. This matches the #170 description.
- The CLI replay reports one more difference than the engine comparison: id 2518, identical findings and exit code, but the detected `version` field changes from 2.0 to unknown because the parser now stops at the error. Benign.

Corpus verdict counts: baseline 2,895 valid and 472 invalid; RC 2,891 valid and 476 invalid.

## Performance

All timings best of three passes after a warm pass, single thread unless stated.

### Native core, baseline vs RC

| Size bucket | Docs | Baseline p50 | RC p50 | RC p99 | Change at p50 |
|---|---|---|---|---|---|
| under 1 KB | 2,003 | 9.6 µs | 10.1 µs | 12.4 µs | +5% |
| 1 to 4 KB | 243 | 17.5 µs | 18.9 µs | 38.6 µs | +8% |
| 4 to 16 KB | 694 | 45.4 µs | 49.7 µs | 86.0 µs | +9% |
| 16 to 64 KB | 427 | 79.2 µs | 89.5 µs | 198.6 µs | +13% |

Throughput: 37.4k docs/s baseline, 34.0k docs/s RC, a 9 percent drop. Per-document median ratio 1.08x, p90 1.20x. Interleaved rounds agreed within 2 percent.

Criterion micro-benchmarks (fixture based, RC vs saved v0.15.3 baseline, same target dir): +0.6% (pod_10_ads) to +4.8% (valid_4.3_with_verification), all statistically significant, all small.

Attribution: the whole-document character check in #170 as originally written cost about 40 percent; the byte-scan commit in this PR brings the total cost of the parser hardening to the figures above. The remainder is the real new checks (attribute normalization, name validation, entity handling, structural limits).

### Go cgo binding, baseline library vs RC library

| | Throughput | Median vs native of same build |
|---|---|---|
| cgo baseline | 31.3k docs/s | 1.39x |
| cgo RC | 29.3k docs/s | 1.36x |

The binding overhead (string copy, JSON out, JSON in) is unchanged by the RC. Library on disk 20.9 MB to 6.98 MB; linked code 1.19 MB to 1.02 MB.

### wasm module through wazero (RC only, new)

| | Value |
|---|---|
| Module size (simd128, release-wasm) | 649 KB, sha256 a0430b30... |
| Compile once | 116 ms |
| Single thread | 10.1k docs/s, median 2.86x native RC |
| 16 to 64 KB p50 / p99 | 316 µs / 691 µs |
| 10 pooled instances | 59.5k docs/s |
| Linear memory after corpus | 1.5 MiB |
| Linear memory after a 4 MiB input | 17 MiB and does not shrink |

The last row is an operational note for the Go binding: cap input size before the call and recycle instances past a memory threshold.

### CLI

| | Baseline | RC |
|---|---|---|
| Binary size | 4,593,568 bytes | 3,188,800 bytes |
| `--version` wall time (50 runs) | 2.55 ms | 2.48 ms |
| Peak RSS on the largest corpus document (63 KB) | 3.84 MiB | 3.81 MiB |
| Median `check` wall per document, corpus | 5.65 ms | 5.64 ms |
| Clean release build time (LTO cost) | 12.6 s | 24.9 s |

## Notes for the release

- Rust callers constructing `ValidationResult` literals gain `input_status`.
- FFI result JSON gains `document_type` and `input_status`; existing fields unchanged. A NULL return now also means an internal panic was caught.
- Release builds take about twice as long because of fat LTO.
- Absolute throughput figures recorded on 2026-10-21 were taken under heavy machine load and are about 2.3x lower than today's for the same baseline build; ratios between engines and builds were consistent across both days.

## Reproduce

`harness/run-all.sh all` with the baseline and RC worktrees and the scratch builds described in the script header. Raw per-document NDJSON, logs and criterion output are in this folder; the corpus itself lives on the AdtechOSS volume.
