# sonic-rs (evaluation notes and adoption policy)

A repository that records the **evaluation** and the **adoption policy** of [`sonic-rs`](https://github.com/cloudwego/sonic-rs) (a SIMD-based Rust JSON library) within `aon-co-jp`.

> **This is not the upstream `sonic-rs`.** The upstream is an Apache-2.0 library by CloudWeGo's (ByteDance) Volo team
> ([cloudwego/sonic-rs](https://github.com/cloudwego/sonic-rs), [crates.io](https://crates.io/crates/sonic-rs)).
> This repository contains none of its code and has no affiliation with it. It only records measurements and how we use it inside `aon-co-jp`.

## Key finding from measurements

Simply adding `sonic-rs` did not make anything faster. Conditions: 7 MB, 40,000 objects, best of 7 runs (`examples/bench.rs` in `RS-JSON`, `--release`).

| Method | Time | vs serde_json |
|---|---|---|
| `serde_json::from_str<Value>` (standard) | 200 ms | 1.00 |
| `parse_secure` (RS-JSON, secure, `serde_json::Value`) | 217 ms | +8% |
| `sonic-rs` → `serde_json::Value` (same type) | 188 ms | -6% (**almost no change**) |
| **`parse_secure_sonic` (fast type + secure)** | **83 ms** | **-58% (about 2.4x faster)** |
| raw `sonic-rs` (no defences, for reference) | 33 ms | about 6x |

**The speed does not come from SIMD but from "the value type that packs the result together".** `sonic-rs` is fast because it stores the parsed result in one contiguous region as its own value type, `sonic_rs::Value`.
As long as the result is built into `serde_json::Value` (a type that allocates memory separately per string and array), scanning with SIMD does not help.
**To get the speed you must keep the result as `sonic_rs::Value`, which means rewriting the code that uses it.**

## What was done
- **`RS-JSON`**: added `parse_secure_sonic`, a secure mode that reads into the fast value type (`fast` feature) ([`aon-co-jp/RS-JSON`](https://github.com/aon-co-jp/RS-JSON) 07eaee3). 50 tests pass. Defences: size limit, depth limit, duplicate-key rejection.
- **Policy written down**: in README, CLAUDE and PORTING of 16 repositories, in Japanese and English: where it can / cannot be used, measured values, how to adopt, and each repository's adoption status.
- **`runo` (meta index)**: only `CLAUDE.md` and `PORTING.md` (its `README.md` is an index table, so no long policy text there).
- **VPS**: 10 repositories were updated by fast-forward.

## Policy (summary)
Wherever JSON is read, use `parse_secure_sonic` from `rust-json` (the result is `sonic_rs::Value`) **everywhere it can be used**. Where it cannot, use `parse_secure` (the result is `serde_json::Value`).
Untrusted outside input **must always** be read with size / depth / duplicate-key limits. Details in [`PORTING-English.md`](PORTING-English.md).

## Notes (honest disclosure)
- The `sonic-rs` dependency uses `unsafe` internally (our own code has none).
- The code itself has **not yet been switched over** (policy only). The benefit is expected where large JSON is read often (e.g. `fetch_cache` in `audiocafe-tokyo-rust` re-reads ~200 KB on every page view). Small JSON (a few KB) shows no difference.
- Measured on one PC, best of 7. Numbers will vary by environment.

## Next candidate
The actual code switch-over, starting with `fetch_cache` in `audiocafe-tokyo-rust`. Where to start is undecided.

## Related
[`RS-JSON`](https://github.com/aon-co-jp/RS-JSON) (provider), [`aruaru-llm`](https://github.com/aon-co-jp/aruaru-llm) (`parse_secure` already applied), [`audiocafe-tokyo-rust`](https://github.com/aon-co-jp/audiocafe-tokyo-rust) (first switch-over candidate).

License: not set (documents only). The upstream `sonic-rs` is Apache-2.0.

## Sister repositories: `sonic-rs` and `RS-JSON` (2026-10-15)
[`sonic-rs`](https://github.com/aon-co-jp/sonic-rs) and [`RS-JSON`](https://github.com/aon-co-jp/RS-JSON) are **sister repositories**. Treat the two as a pair.

| Repository | Role |
|---|---|
| [`RS-JSON`](https://github.com/aon-co-jp/RS-JSON) | **The code.** `parse_secure` (secure), `parse_secure_sonic` (fast type + secure, `fast` feature) and `light` (for WASM). Holds the tests and the measurement bench (`examples/bench.rs`) |
| [`sonic-rs`](https://github.com/aon-co-jp/sonic-rs) | **Evaluation notes and adoption policy** (documents only): how to read the measurements, where it can / cannot be used, and each repository's adoption status |

- This repository (`sonic-rs`) holds the evaluation notes and the adoption policy (documents only). `RS-JSON` holds the code and the measurement bench.
- **When you update one, update the other too** so the measured numbers, the policy and the adoption status never disagree. The source of truth for numbers is the measurement in `RS-JSON`'s `examples/bench.rs`; for the policy it is `PORTING.md` in `sonic-rs`.
- It is unrelated to the upstream `sonic-rs` (CloudWeGo/ByteDance, Apache-2.0). The `sonic-rs` repository contains none of its code and has no affiliation with it.
