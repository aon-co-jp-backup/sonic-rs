# Policy of the sonic-rs repository (CLAUDE-English.md)

## Role of this repository
A **documents-only** repository holding the evaluation notes and adoption policy for `sonic-rs` (the upstream is an Apache-2.0 library by CloudWeGo/ByteDance). **Do not copy or port the upstream code.** The code itself lives in
[`RS-JSON`](https://github.com/aon-co-jp/RS-JSON) (`parse_secure_sonic`, `fast` feature).

## Rules
- Write **measured numbers only**. Never call something "fast" without measuring it. State the conditions (data size, run count, `--release`, environment).
- Key fact: the speed comes not from SIMD but from **the value type that packs the result into contiguous memory**. `sonic-rs` → `serde_json::Value` is not faster (188 ms vs 200 ms standard). Do not blur this.
- The `sonic-rs` dependency uses `unsafe` internally. Distinguish "our code has no `unsafe`" from "the dependency uses it".
- Never write secrets (API keys, passwords, tokens). Do not force-push.
- Update both Japanese and English documents (`README` / `CLAUDE` / `PORTING` and their `-English` versions).

## Status (2026-10-14)
- Measured: `serde_json` 200 ms / `parse_secure` 217 ms / `sonic-rs`→`serde_json::Value` 188 ms / **`parse_secure_sonic` 83 ms** / raw `sonic-rs` 33 ms.
- `parse_secure_sonic` is implemented in `RS-JSON` (50 tests). The policy is written in 16 repositories.
- **The code has not been switched over yet.**

## Related operational notes (as of 2026-10-14)
- On GitHub, `aruaru-llm`, `audiocafe-tokyo-rust` and `runo` have **rewritten history** (same content, different hashes). Local copies and the VPS are aligned; the old history is in the `backup-before-rewrite-20261014` / `vps-before-rewrite-20261014` branches. Who rewrote it is unknown. Do not force-push.
- `open-web-server` has **two commits on the VPS that are not on GitHub** (certbot webroot fix, streaming support) plus 16 uncommitted files. Do not touch the history; push it soon.
- `maid-cafe-se` has no upstream (push target) configured.

## Next
Switch `fetch_cache` in `audiocafe-tokyo-rust` (re-reads ~200 KB on every page view) to `sonic_rs::Value` and confirm the effect by measurement.

## Sister repositories: `sonic-rs` and `RS-JSON` (2026-10-15)
[`sonic-rs`](https://github.com/aon-co-jp/sonic-rs) and [`RS-JSON`](https://github.com/aon-co-jp/RS-JSON) are **sister repositories**. Treat the two as a pair.

| Repository | Role |
|---|---|
| [`RS-JSON`](https://github.com/aon-co-jp/RS-JSON) | **The code.** `parse_secure` (secure), `parse_secure_sonic` (fast type + secure, `fast` feature) and `light` (for WASM). Holds the tests and the measurement bench (`examples/bench.rs`) |
| [`sonic-rs`](https://github.com/aon-co-jp/sonic-rs) | **Evaluation notes and adoption policy** (documents only): how to read the measurements, where it can / cannot be used, and each repository's adoption status |

- This repository (`sonic-rs`) holds the evaluation notes and the adoption policy (documents only). `RS-JSON` holds the code and the measurement bench.
- **When you update one, update the other too** so the measured numbers, the policy and the adoption status never disagree. The source of truth for numbers is the measurement in `RS-JSON`'s `examples/bench.rs`; for the policy it is `PORTING.md` in `sonic-rs`.
- It is unrelated to the upstream `sonic-rs` (CloudWeGo/ByteDance, Apache-2.0). The `sonic-rs` repository contains none of its code and has no affiliation with it.
