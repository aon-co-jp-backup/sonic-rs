# PORTING(sonic-rs)

## 目的
JSONの読み取りを速くし、同時に外部入力に対して安全にする。そのための`sonic-rs`の評価結果と、`aon-co-jp`の各リポジトリへの導入方針。

## 実測(7 MB・4万件、最速の7回、`--release`)
| 方式 | 時間 | serde_json 比 |
|---|---|---|
| `serde_json::from_str<Value>`(標準) | 200 ms | 1.00 |
| `parse_secure`(RS-JSON、安全・`serde_json::Value`) | 217 ms | +8% |
| `sonic-rs` → `serde_json::Value`(型をそのまま) | 188 ms | -6%(**ほぼ変わらない**) |
| **`parse_secure_sonic`(速い型+安全)** | **83 ms** | **-58%(約2.4倍速い)** |
| `sonic-rs` そのまま(防御なし、参考) | 33 ms | 約6倍 |

再現: `RS-JSON`で `cargo run --release --features fast --example bench`。

## なぜ`serde_json::Value`では速くならないか
`sonic-rs`の速さは、SIMDで文字を走査することよりも、**読み取った結果を1つの連続した領域に詰め込む値の型**による。`serde_json::Value`は文字列・配列・オブジェクトごとに
別々にメモリを確保するので、組み立てがボトルネックになり、`sonic-rs`で読んでも188 ms(標準200 ms)に留まる。**型を替えない導入は、速度の面では意味がない。**

## 導入手順
1. `Cargo.toml`: `rust-json = { path = "../RS-JSON", features = ["fast"] }`(`RS-JSON`は兄弟ディレクトリを相対パスで参照する。VPSでも`RS-JSON`を最新にしてからビルドする)。
2. 読む場所を`rust_json::parse_secure_sonic(&text, &Limits::small())`に替える。結果は`sonic_rs::Value`。
3. 値を使う側を、`sonic_rs::{JsonContainerTrait, JsonValueTrait}`のメソッド(`get`・`as_str`・`as_array`・`as_object`・`iter`)に書き換える。
4. 実測して効果を確認する(小さなJSONでは差が出ない)。

## 使える所 / 使えない所
- ✅ 読み取り中心で、JSONを大きく・頻繁に読む所(キャッシュJSON、設定、APIの応答、ログ、インデックス)。
- ✅ 外部からの入力の入口(速さより防御が目的。`parse_secure`でも可)。
- ⚠️ `serde_json::Value`を関数間で受け渡している既存コード(型の書き換えが要る)。
- ❌ WASM(`rust_json::light`、依存ゼロ)、PHP/JavaScript側のJSON処理。

## 各リポジトリの適用状況(2026-10-14)
`RS-JSON`: 提供元(`parse_secure`・`parse_secure_sonic`実装済み) / `aruaru-llm`: 外部取り込みは`parse_secure`に適用済み / その他: 方針の記載のみ・コードは未切り替え。

## 完了済み(2026-10-14)
`RS-JSON`に`parse_secure_sonic`(テスト50件)、16リポジトリへの方針の記載、VPS 10リポジトリの反映。

## 未完了・要判断
1. 実コードの切り替え(まず`audiocafe-tokyo-rust`の`fetch_cache`)。
2. `open-web-server`のVPS上の未push作業をGitHubへpush(履歴を触らない)。
3. `maid-cafe-se`の上流設定。
4. 新品VPSでの引越しの通し試験(未実施)。

## 注意
- 依存先`sonic-rs`は`unsafe`を使う。既定ではビルドに入らず、`fast`機能を選んだときだけ入る。
- GitHub上の`aruaru-llm`・`audiocafe-tokyo-rust`・`runo`は履歴が書き直されている。強制pushしない。

## 姉妹リポジトリ: `sonic-rs` と `RS-JSON`(2026-10-15)
[`sonic-rs`](https://github.com/aon-co-jp/sonic-rs)と[`RS-JSON`](https://github.com/aon-co-jp/RS-JSON)は**姉妹リポジトリ**。2つで1組として扱う。

| リポジトリ | 役割 |
|---|---|
| [`RS-JSON`](https://github.com/aon-co-jp/RS-JSON) | **コードの実体**。`parse_secure`(安全)・`parse_secure_sonic`(速い型+安全、`fast`機能)・`light`(WASM用)。テストと実測ベンチ(`examples/bench.rs`)を持つ |
| [`sonic-rs`](https://github.com/aon-co-jp/sonic-rs) | **評価の記録と導入方針**(文書のみ)。実測の読み方、使える所/使えない所、各リポジトリへの適用状況 |

- このリポジトリ(`sonic-rs`)はは評価の記録と導入方針を持つ(文書のみ)。`RS-JSON`はコードの実体と実測ベンチを持つ。
- **片方を更新したら、もう片方も追従する**(実測値・方針・適用状況が食い違わないようにする)。数値の正本は`RS-JSON`の`examples/bench.rs`の実測、方針の正本は`sonic-rs`の`PORTING.md`。
- 本家の`sonic-rs`(CloudWeGo/ByteDance、Apache-2.0)とは別物。`sonic-rs`リポジトリは本家のコードを含まず、提携関係もない。
