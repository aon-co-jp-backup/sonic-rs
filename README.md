# sonic-rs(評価記録と導入方針)

`aon-co-jp`での[`sonic-rs`](https://github.com/cloudwego/sonic-rs)(SIMDを使うRustのJSONライブラリ)の**評価の記録**と**導入方針**をまとめるリポジトリ。

> **これは`sonic-rs`の本家ではありません。** 本家は CloudWeGo(ByteDance)の Volo チームによる Apache-2.0 のライブラリ
> ([cloudwego/sonic-rs](https://github.com/cloudwego/sonic-rs)、[crates.io](https://crates.io/crates/sonic-rs))。
> このリポジトリは本家のコードを含まず、提携関係もない。実測と、`aon-co-jp`内での使い方を記録するだけ。

## 実測で分かった重要なこと

`sonic-rs` を入れるだけでは、速くなりませんでした。条件: 7 MB・4万件のオブジェクト、最速の7回(`RS-JSON`の`examples/bench.rs`、`--release`)。

| 方式 | 時間 | serde_json 比 |
|---|---|---|
| `serde_json::from_str<Value>`(標準) | 200 ms | 1.00 |
| `parse_secure`(RS-JSON、安全・`serde_json::Value`) | 217 ms | +8% |
| `sonic-rs` → `serde_json::Value`(型をそのまま) | 188 ms | -6%(**ほぼ変わらない**) |
| **`parse_secure_sonic`(速い型+安全)** | **83 ms** | **-58%(約2.4倍速い)** |
| `sonic-rs` そのまま(防御なし、参考) | 33 ms | 約6倍 |

**速さの源は SIMD ではなく、「結果をまとめて詰め込む値の型」です。** `sonic-rs`が速いのは、読み取った結果を1つの連続した領域に詰め込む自前の値の型`sonic_rs::Value`のため。
結果を`serde_json::Value`(文字列や配列ごとに別々にメモリを確保する型)に組み立てる限り、SIMDで走査しても速くならない。
**速くするには、結果を`sonic_rs::Value`のまま使う必要があり、使う側のコードの書き換えが要る。**

## 実施したこと
- **`RS-JSON`**: 速い値の型で読む安全モード`parse_secure_sonic`(`fast`機能)を追加([`aon-co-jp/RS-JSON`](https://github.com/aon-co-jp/RS-JSON) 07eaee3)。テスト50件合格。防御はサイズ・深さ・重複キー拒否。
- **方針の記載**: 16リポジトリのREADME・CLAUDE・PORTINGに、日本語と英語で書いた。使える所と使えない所、実測値、導入方法、各リポジトリの適用状況を載せている。
- **`runo`(メタ索引)**: `CLAUDE.md`と`PORTING.md`のみ(`README.md`は索引の一覧表なので長い方針の文章は入れない)。
- **VPS**: 10リポジトリをfast-forwardで反映した。

## 方針(要約)
JSONを読む所は、**使える所では全て**`rust-json`の`parse_secure_sonic`(結果は`sonic_rs::Value`)を使う。使えない所は`parse_secure`(結果は`serde_json::Value`)。
外部から来る信頼できない入力は**必ず**サイズ・深さ・重複キーの上限つきで読む。詳細は[`PORTING.md`](PORTING.md)。

## 注意(正直な開示)
- 依存先の`sonic-rs`は内部で`unsafe`を使う(自前コードに`unsafe`は無い)。
- コード自体の切り替えは**まだ行っていない**(方針のみ)。効果が見込めるのは、JSONを大きく・頻繁に読む場所(例: `audiocafe-tokyo-rust`の`fetch_cache`は、ページ表示のたびに約200KBを読み直す)。小さなJSON(数KB)では差が出ない。
- 測定は1台のPCでの1回分(最速7回)。環境で数値は変わる。

## 次の候補
実際のコードの切り替え。`audiocafe-tokyo-rust`の`fetch_cache`から。どこから進めるかは未定。

## 関連
[`RS-JSON`](https://github.com/aon-co-jp/RS-JSON)(提供元)、[`aruaru-llm`](https://github.com/aon-co-jp/aruaru-llm)(`parse_secure`適用済み)、[`audiocafe-tokyo-rust`](https://github.com/aon-co-jp/audiocafe-tokyo-rust)(最初の切り替え候補)。

ライセンス: 未設定(文書のみ)。本家`sonic-rs`は Apache-2.0。

## 姉妹リポジトリ: `sonic-rs` と `RS-JSON`(2026-10-15)
[`sonic-rs`](https://github.com/aon-co-jp/sonic-rs)と[`RS-JSON`](https://github.com/aon-co-jp/RS-JSON)は**姉妹リポジトリ**。2つで1組として扱う。

| リポジトリ | 役割 |
|---|---|
| [`RS-JSON`](https://github.com/aon-co-jp/RS-JSON) | **コードの実体**。`parse_secure`(安全)・`parse_secure_sonic`(速い型+安全、`fast`機能)・`light`(WASM用)。テストと実測ベンチ(`examples/bench.rs`)を持つ |
| [`sonic-rs`](https://github.com/aon-co-jp/sonic-rs) | **評価の記録と導入方針**(文書のみ)。実測の読み方、使える所/使えない所、各リポジトリへの適用状況 |

- このリポジトリ(`sonic-rs`)はは評価の記録と導入方針を持つ(文書のみ)。`RS-JSON`はコードの実体と実測ベンチを持つ。
- **片方を更新したら、もう片方も追従する**(実測値・方針・適用状況が食い違わないようにする)。数値の正本は`RS-JSON`の`examples/bench.rs`の実測、方針の正本は`sonic-rs`の`PORTING.md`。
- 本家の`sonic-rs`(CloudWeGo/ByteDance、Apache-2.0)とは別物。`sonic-rs`リポジトリは本家のコードを含まず、提携関係もない。
