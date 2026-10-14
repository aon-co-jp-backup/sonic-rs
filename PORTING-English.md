# PORTING (sonic-rs)

## Purpose
Make JSON reading faster and, at the same time, safe against outside input. This records the evaluation of `sonic-rs` and the policy for adopting it across the `aon-co-jp` repositories.

## Measurements (7 MB, 40,000 records, best of 7, `--release`)
| Method | Time | vs serde_json |
|---|---|---|
| `serde_json::from_str<Value>` (standard) | 200 ms | 1.00 |
| `parse_secure` (RS-JSON, secure, `serde_json::Value`) | 217 ms | +8% |
| `sonic-rs` → `serde_json::Value` (same type) | 188 ms | -6% (**almost no change**) |
| **`parse_secure_sonic` (fast type + secure)** | **83 ms** | **-58% (about 2.4x faster)** |
| raw `sonic-rs` (no defences, for reference) | 33 ms | about 6x |

Reproduce: in `RS-JSON`, `cargo run --release --features fast --example bench`.

## Why `serde_json::Value` does not get faster
The speed of `sonic-rs` comes less from scanning with SIMD than from **a value type that packs the parsed result into one contiguous region**. `serde_json::Value` allocates memory separately for every string, array and object,
so building it is the bottleneck: reading with `sonic-rs` only reaches 188 ms (standard 200 ms). **Adopting it without changing the type is pointless as far as speed goes.**

## How to adopt
1. `Cargo.toml`: `rust-json = { path = "../RS-JSON", features = ["fast"] }` (`RS-JSON` is referenced as a sibling directory by relative path; update `RS-JSON` before building on the VPS too).
2. Replace the reading site with `rust_json::parse_secure_sonic(&text, &Limits::small())`. The result is `sonic_rs::Value`.
3. Rewrite the code that uses the value with the methods of `sonic_rs::{JsonContainerTrait, JsonValueTrait}` (`get`, `as_str`, `as_array`, `as_object`, `iter`).
4. Measure to confirm the effect (small JSON shows no difference).

## Where it can / cannot be used
- ✅ Read-heavy code that parses large or frequent JSON (cache JSON, config, API responses, logs, indexes).
- ✅ Entrances for outside input (the goal is defence rather than speed; `parse_secure` is fine).
- ⚠️ Existing code that passes `serde_json::Value` between functions (the type has to be rewritten).
- ❌ WASM (`rust_json::light`, zero dependencies) and the JSON handling on the PHP / JavaScript side.

## Adoption status per repository (2026-10-14)
`RS-JSON`: provider (`parse_secure` and `parse_secure_sonic` implemented) / `aruaru-llm`: outside import already uses `parse_secure` / others: policy written only, code not yet switched.

## Done (2026-10-14)
`parse_secure_sonic` in `RS-JSON` (50 tests), the policy written in 16 repositories, 10 repositories updated on the VPS.

## Not done / needs a decision
1. The actual code switch-over (start with `fetch_cache` in `audiocafe-tokyo-rust`).
2. Push the un-pushed work of `open-web-server` on the VPS to GitHub (do not touch the history).
3. Set an upstream for `maid-cafe-se`.
4. An end-to-end migration test on a fresh VPS (not yet done).

## Notes
- The `sonic-rs` dependency uses `unsafe`. It is not built by default; it is only included when the `fast` feature is chosen.
- On GitHub, `aruaru-llm`, `audiocafe-tokyo-rust` and `runo` have rewritten history. Do not force-push.

## Sister repositories: `sonic-rs` and `RS-JSON` (2026-10-15)
[`sonic-rs`](https://github.com/aon-co-jp/sonic-rs) and [`RS-JSON`](https://github.com/aon-co-jp/RS-JSON) are **sister repositories**. Treat the two as a pair.

| Repository | Role |
|---|---|
| [`RS-JSON`](https://github.com/aon-co-jp/RS-JSON) | **The code.** `parse_secure` (secure), `parse_secure_sonic` (fast type + secure, `fast` feature) and `light` (for WASM). Holds the tests and the measurement bench (`examples/bench.rs`) |
| [`sonic-rs`](https://github.com/aon-co-jp/sonic-rs) | **Evaluation notes and adoption policy** (documents only): how to read the measurements, where it can / cannot be used, and each repository's adoption status |

- This repository (`sonic-rs`) holds the evaluation notes and the adoption policy (documents only). `RS-JSON` holds the code and the measurement bench.
- **When you update one, update the other too** so the measured numbers, the policy and the adoption status never disagree. The source of truth for numbers is the measurement in `RS-JSON`'s `examples/bench.rs`; for the policy it is `PORTING.md` in `sonic-rs`.
- It is unrelated to the upstream `sonic-rs` (CloudWeGo/ByteDance, Apache-2.0). The `sonic-rs` repository contains none of its code and has no affiliation with it.
