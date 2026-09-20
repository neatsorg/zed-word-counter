# zed-word-counter

Zed のステータスバーに、保存前のアクティブ文書全体を対象とした行数・文字数・ブロック（段落）数を
常時表示する機能のパッチを管理するリポジトリ。Zed 本体（Rust/GPUI）のソースに対する差分そのものが
成果物であり、独立した Rust ソースツリーは持たない。

## 構成

- `upstream.toml` — パッチの対象とする Zed のコミット。
- `patches/` — 対象コミットに適用する差分（`git diff` 形式）。
- `docs/research.md` — 先行実装・関連PRの調査記録。
- `docs/design.md` — 要件の解釈と実装方針。
- `scripts/prepare` — `upstream.toml` が指すコミットをチェックアウトし、`patches/` を適用する。
- `scripts/check` — 適用後のチェックアウトに対して `cargo check`/`cargo test` を実行する。

## 使い方

```sh
scripts/prepare   # .checkout/zed に対象コミットを用意し、パッチを当てる
scripts/check     # 型検査とテストを実行する
```

開発中の変更は `.checkout/zed` の中で直接行い、確認できたら
`git -C .checkout/zed diff > patches/0001-status-bar-document-stats.patch`
のように差分を書き出して `patches/` を更新する。実装コードを `patches/` とは別に
このリポジトリ内へ独立ファイルとして複製することはしない（二重管理を避けるため）。

## 位置づけ

## ライセンスと公開範囲

このリポジトリのパッチ、スクリプト、文書は GPL-3.0-or-later です。本文は
[LICENSE](LICENSE)、Zed由来部分と変更範囲は [NOTICE](NOTICE) を参照してください。
これはZed公式配布物ではありません。`.checkout/` とビルド生成物は公開対象に含めません。

パッチ適用済みバイナリを配布する場合は、対応するZedソース、依存ライセンス表示、
ロックファイル、再現可能なビルド手順を別途提供してください。

- **[zed-writing-tools](../zed-writing-tools)**（変換・校正・翻訳の Zed 拡張／LSPサーバー群）とは独立。
  日本語での執筆支援という上位の目的は共有するが、実装・配布経路は別。
- **[zed-i18n](../zed-i18n)** はこのパッチの取り込み側。多言語対応パッチとは責任範囲を分け、
  zed-i18n 側では本パッチのバージョンを指定して適用したうえで、必要な翻訳（ラベル文字列等）を追加する。
- 本家 [zed-industries/zed](https://github.com/zed-industries/zed) へは、このリポジトリから直接
  PR を出すのではなく、本家のフォークに作った機能ブランチから提出する。ここは提出用のパッチを
  管理する場所であり、機能を二重に開発する場所ではない。
