# 設計方針

[research.md](./research.md) の要件・調査結果を受けての実装方針。対象は
[upstream.toml](../upstream.toml) が指す Zed v1.20.2（コミット `7c451e6`）。既存の
`crates/go_to_line/src/cursor_position.rs`（`CursorPosition`、ステータスバー右側の既存アイテム）
を拡張する。PR #60113 と同じ土台だが、完成品の移植ではなくイベント購読・更新の間引き・設定配線を
参考にするに留めた。

## 集計と表示の分離

将来、別のステータスバー項目や別クレートから同じ集計を再利用できるようにするため、
「集計（文書全体の行数・文字数・ブロック数を数える処理）」と「表示（ラベル・区切り・順序を適用して
文字列に組み立てる処理）」を別モジュールに分ける。

- `crates/go_to_line/src/document_stats.rs`（新規）: `compute(&MultiBufferSnapshot) -> DocumentStatsValues`。
  GPUI・Settings・Render に一切依存しない純粋関数。
  - `lines`・`characters` は Rope のキャッシュ済みサマリー（`MultiBufferSnapshot::text_summary()`）
    から O(log n) で取得する。フルスキャンしない。
  - `blocks` のみ、0行目から最終行まで `line_len(row) == 0` を見ながら非空行の連続区間数を数える
    O(行数) の処理。空行判定に文字長しか使わないため、空白のみの行はブロックを分けない
    （要件の「2個以上の連続改行」を文字どおり解釈した結果）。文書先頭・末尾の空行はどのブロックにも
    属さない。
- `crates/go_to_line/src/cursor_position.rs`: 表示側。`write_document_stats` が
  `DocumentStatsValues` と設定 (`items`/`separator`) だけから文字列を組み立てる、`&self` を取らない
  純粋な整形関数。

このリポジトリでは実装コードを `patches/` の差分としてのみ管理し、`.rs` ファイルを別に複製しない
（[README](../README.md) 参照）。

## 設定

`status_bar.document_stats_button`（既定 `false`。PR #60113 の `word_count_button` 既定に合わせた
オプトイン）と、`status_bar.document_stats.{items, separator}` を追加した
（`crates/settings_content/src/workspace.rs`）。

- `items`: `{item, label}` の Vec。並び順がそのまま表示順になる。`item` は
  `lines`/`characters`/`blocks` の enum、`label` は省略時に既定ラベル（chars/blocks/lines）を使う。
  列挙型をキーにした `HashMap` は使わず要素ごとの構造体にし、JsonSchema 導出の不確実性を避けた。
- `separator`: 各項目間に挿入する区切り文字列。
- 解決済み設定は `crates/workspace/src/workspace_settings.rs` の `DocumentStats`
  （`items: Vec<(DocumentStatsItem, SharedString)>`、`separator: String`）に持たせる。
  `StatusBarSettings` は元々 `#[derive(Deserialize, ...)]` だったが、`DocumentStats` が
  `Deserialize` を実装しない（`SharedString` を含むため）ので、`Deserialize` 導出を外した
  （`WorkspaceSettings` など、解決済み設定構造体で `Deserialize` を導出しない前例に合わせた）。

## 常時計測・再計算

既存の `CursorPosition::update_position`（デバウンス済み非同期タスク、`cx.spawn_in`）に相乗りする。
`self.update_position` への再代入で古いタスクが自動的にキャンセルされる（gpui の `Task` drop）ため、
古い集計結果の破棄はこの仕組みに乗るだけでよい。

購読イベントに `EditorEvent::SelectionsChanged` だけでなく `EditorEvent::BufferEdited`
（他エディタ経由の編集を含む、下線バッファへのあらゆる変更）を追加し、選択を動かさない編集でも
再計算されるようにした。文書統計は選択の有無に関係なく常に計算・表示する点が、既存の選択統計
（1つ以上選択があるときだけ括弧書きで追記する `write_position`）と異なる。

## 行表示

`document_stats_button` が有効な場合、`Ln {現在行}/{総行数}, Col {文字位置}` の形式に変更する。
`GoToLine` モーダル（`go_to_line.rs`）は、このボタンの表示文字列をパースするのではなく
`UserCaretPosition`/`current_line` から直接文字列を組み立てて動作するため
（`anchor_from_query`/`line_and_char_from_query` は自身の `line_editor` の入力のみを見る）、
表示形式の変更はモーダルの動作に影響しない。

## テスト

`crates/go_to_line/src/go_to_line.rs` の既存テスト（`test_unicode_characters_selection` 等と同じ、
`FakeFs` → `Project::test` → `MultiWorkspace::test_new` → `CursorPosition` を右アイテムに追加 →
`advance_clock` のパターン）に `test_document_stats` を追加する。ブロック分割ルール
（単一改行では分割しない・空白のみの行は分割しない・2個以上の連続改行で分割する・先頭と末尾の
ブロックも数える）と、行数が折り返しではなく実際の改行数に基づくことを検証する。

`CursorPosition::document_stats()` という `#[cfg(test)]` アクセサを、既存の `selection_stats()`／
`position()` に倣って追加した。

## ビルド・テスト時の既知の落とし穴

`cargo test` を `.checkout/zed` の外に向けた `CARGO_TARGET_DIR` で実行すると、Zed のdev版アセット
読み込み（`util::dev_repo_root()`）がテストバイナリのパスから `.git` を持つ祖先を遡って探すため、
外側の別リポジトリを誤って「チェックアウトの場所」と認識し、`assets/settings/default.json` の
読み込みに失敗して**無改造の Zed ソースでも**全テストが落ちる。`scripts/check` はこれを避けるため
`CARGO_TARGET_DIR` を明示的に unset し、既定の `target/`（`.checkout/zed` 内）を使う。

## 今回のパッチに含めないこと

- zed-i18n の取り込み処理（パッチのバージョン指定・適用、ローカライズ済みツリーとの適用順序の検証）。
  zed-i18n 側で行う。
- 新規UI文字列（"chars"/"blocks" 等の既定ラベル）の翻訳対応。zed-i18n 側の抽出・翻訳パイプラインに委ねる。
- `crates/settings_ui`（設定GUI）への項目追加。
- 本家 `zed-industries/zed` の現行 `main` への追従（リベース）。PR提出時に本家のフォークで対応する。
