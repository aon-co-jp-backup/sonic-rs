# sonic-rs リポジトリの方針(CLAUDE.md)

## このリポジトリの役割
`sonic-rs`(本家はCloudWeGo/ByteDanceのApache-2.0ライブラリ)の評価記録と導入方針を持つ**文書のみ**のリポジトリ。**本家のコードを複製・移植しない**。コードの実体は
[`RS-JSON`](https://github.com/aon-co-jp/RS-JSON)(`parse_secure_sonic`、`fast`機能)にある。

## 守ること
- 数値は**実測のものだけ**を書く。測っていないことを「速い」と書かない。測定条件(データ量・回数・`--release`・環境)を併記する。
- 重要な事実: 速さの源はSIMDではなく**結果を連続領域に詰め込む値の型**。`sonic-rs`→`serde_json::Value`では速くならない(188 ms、標準200 ms)。この点を曖昧にしない。
- 依存先`sonic-rs`は内部で`unsafe`を使う。自前コードに`unsafe`は無いことと、依存先が使うことを区別して書く。
- 秘密(APIキー・パスワード・トークン)を書かない。強制pushしない。
- ドキュメントは日本語と英語の両方を更新する(`README`/`CLAUDE`/`PORTING`とその`-English`版)。

## 現状(2026-10-14)
- 実測済み: `serde_json`200 ms / `parse_secure`217 ms / `sonic-rs→serde_json::Value`188 ms / **`parse_secure_sonic`83 ms** / `sonic-rs`そのまま33 ms。
- `RS-JSON`に`parse_secure_sonic`を実装済み(テスト50件)。16リポジトリに方針を記載済み。
- **コードの切り替えは未実施。**

## 関連する運用上の注意(2026-10-14時点)
- GitHub上の`aruaru-llm`・`audiocafe-tokyo-rust`・`runo`は**履歴が書き直されている**(内容は同じでハッシュが違う)。手元とVPSは揃え済み、旧履歴は`backup-before-rewrite-20261014`/`vps-before-rewrite-20261014`ブランチ。誰が書き直したかは不明。強制pushしない。
- `open-web-server`はVPSに**GitHub未反映の独自コミット2件**(certbot webroot修正・ストリーミング対応)+未コミット16件がある。履歴を触らず、早めにpushする。
- `maid-cafe-se`は上流(push先)が未設定。

## 次にやること
`audiocafe-tokyo-rust`の`fetch_cache`(ページ表示のたびに約200KBを読み直す)を`sonic_rs::Value`に切り替え、実測で効果を確認する。

## 姉妹リポジトリ: `sonic-rs` と `RS-JSON`(2026-10-15)
[`sonic-rs`](https://github.com/aon-co-jp/sonic-rs)と[`RS-JSON`](https://github.com/aon-co-jp/RS-JSON)は**姉妹リポジトリ**。2つで1組として扱う。

| リポジトリ | 役割 |
|---|---|
| [`RS-JSON`](https://github.com/aon-co-jp/RS-JSON) | **コードの実体**。`parse_secure`(安全)・`parse_secure_sonic`(速い型+安全、`fast`機能)・`light`(WASM用)。テストと実測ベンチ(`examples/bench.rs`)を持つ |
| [`sonic-rs`](https://github.com/aon-co-jp/sonic-rs) | **評価の記録と導入方針**(文書のみ)。実測の読み方、使える所/使えない所、各リポジトリへの適用状況 |

- このリポジトリ(`sonic-rs`)はは評価の記録と導入方針を持つ(文書のみ)。`RS-JSON`はコードの実体と実測ベンチを持つ。
- **片方を更新したら、もう片方も追従する**(実測値・方針・適用状況が食い違わないようにする)。数値の正本は`RS-JSON`の`examples/bench.rs`の実測、方針の正本は`sonic-rs`の`PORTING.md`。
- 本家の`sonic-rs`(CloudWeGo/ByteDance、Apache-2.0)とは別物。`sonic-rs`リポジトリは本家のコードを含まず、提携関係もない。
