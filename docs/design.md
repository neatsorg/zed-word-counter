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
（他エディタ経由の編集を含む、下線バッファへのあらゆる変更）を追加した。文書統計は選択の有無に
関係なく常に表示する点が、既存の選択統計（1つ以上選択があるときだけ括弧書きで追記する
`write_position`）と異なる。

ただし文書統計の**再計算**は、以下の条件をすべて満たすときだけ行う（それ以外は前回計算した値を
そのまま使い回す）。ブロック数の計算が O(行数) の全行走査であり、カーソル移動のたびに長大な文書を
毎回スキャンし直すのは無駄が大きいため（レビュー指摘で判明、初版はこの制御が漏れていた）。

- `update_position` を呼んだイベントが `EditorEvent::BufferEdited`（本文の変更）であること。
  `EditorEvent::SelectionsChanged`（カーソル移動・選択変更のみ）では再計算しない
  （`update_position` に `recompute_document_stats: bool` を追加し、呼び出し側で
  `SelectionsChanged` は `false`、`BufferEdited` と `set_active_pane_item` での初回呼び出しは
  `true` を渡す）。文書統計は本文の内容だけに依存し、カーソル位置には依存しないため、
  カーソル移動時は前回の値をそのまま再利用してよい。
- `status_bar.document_stats_button` が有効であること。無効時（既定）は集計そのものを省略する。
- `editor.buffer().read(cx).is_singleton()` が真であること（後述）。

### 設定変更時の即時反映（2026-09-20、レビュー指摘で追加）

上記の「初版」では、`document_stats_button` を無効から有効へ切り替えても、次にバッファが編集される
かタブが再アクティブ化されるまで表示が更新されなかった（設定変更を監視していなかったため）。
実用上ほぼ問題にならないと判断していたが、レビューで「開いている文書で設定を有効にしても統計が
表示されない」と指摘され、修正した。

`CursorPosition` に `active_editor: Option<WeakEntity<Editor>>` を追加し、`set_active_pane_item`
で現在のエディタを弱参照として保持する。`CursorPosition::new` で
`cx.observe_global::<SettingsStore>(|cursor_position, cx| {...})`
（`crates/agent_ui/src/profile_selector.rs` の `settings_subscription` 等、既存コードベースで
広く使われている購読パターン）を一度だけ登録し、設定が変わるたびに `active_editor` を辿って
（アップグレードできる場合のみ）文書統計を同期的に再計算する。デバウンス付き非同期タスクである
`update_position` とは別経路（`cx.observe_global` のコールバックは `Window` を受け取らないため
`cx.spawn_in` は使えない）だが、計算そのものは `update_position` 側と同じロジック
（`is_singleton` 判定・設定判定・`document_stats::compute` 呼び出し）を踏襲している。
設定変更は稀な操作なのでデバウンスは不要と判断した。

`test_document_stats_populates_when_enabled_while_open` を追加し、文書を開いた直後（設定オフ）は
統計が計算されず、その後に設定を有効化すると編集やタブ切り替えなしに即座に統計が populate される
ことを検証する。このテストも、購読を無効化すると実際に失敗することを確認済み。

### 複数バッファ（マルチバッファ）表示での扱い

`position.line` は `UserCaretPosition::at_selection_end` 経由で、選択終端の位置を
`MultiBufferSnapshot::point_to_buffer_point` で元の実バッファに変換できた場合、その**実バッファ内の
行番号**になる（検索結果や複数ファイルの抜粋を1つのエディタにまとめて表示する、Zed の
マルチバッファ機能での既存の挙動）。一方 `document_stats::compute` は渡された
`MultiBufferSnapshot` 全体（複数ファイルの抜粋をまとめた表示全体）を対象に集計するため、
両者は異なる文書を指してしまう（レビュー指摘で判明。例: 元ファイルの100行目を含む3行の抜粋を
表示している場合、`position.line` は100、`document_stats.lines` は抜粋全体の行数になり、
「100/3」のような無意味な表示になる）。

対策として、`update_position`内で既に計算している `is_singleton`
（`editor.buffer().read(cx).is_singleton()`。既存のデバウンス要否判定にも使われている）を
文書統計の再計算条件に加え、マルチバッファ（`is_singleton == false`）では文書統計を計算せず
`None` のままにする。マルチバッファ表示では文書統計セグメント自体を非表示にする、という
レビューが提示した2案のうち後者を採った（元の実バッファ側に揃える案は、複数の抜粋が異なる
ファイルにまたがる場合に「どのファイルを『文書』とするか」が曖昧になるため見送った）。

## 行表示

`document_stats_button` が有効な場合、`Ln {現在行}/{総行数}, Col {文字位置}` の形式に変更する。
`GoToLine` モーダル（`go_to_line.rs`）は、このボタンの表示文字列をパースするのではなく
`UserCaretPosition`/`current_line` から直接文字列を組み立てて動作するため
（`anchor_from_query`/`line_and_char_from_query` は自身の `line_editor` の入力のみを見る）、
表示形式の変更はモーダルの動作に影響しない。

## テスト

`crates/go_to_line/src/go_to_line.rs` の既存テスト（`test_unicode_characters_selection` 等と同じ、
`FakeFs` → `Project::test` → `MultiWorkspace::test_new` → `CursorPosition` を右アイテムに追加 →
`advance_clock` のパターン）に以下を追加した。

- `test_document_stats`: `status_bar.document_stats_button` を有効にした上で、ブロック分割ルール
  （単一改行では分割しない・空白のみの行は分割しない・2個以上の連続改行で分割する・先頭と末尾の
  ブロックも数える）と、行数が折り返しではなく実際の改行数に基づくことを検証する。加えて、
  内容を変更しないカーソル移動（`MoveRight`）の前後で `document_stats()` の値が変化しないこと
  （再計算がスキップされ、前回の値がそのまま使われること）も検証する。
- `test_document_stats_disabled_by_default`: 設定を有効化しない（既定値のまま）場合、
  文書統計が一切計算されない（`document_stats()` が `None`）ことを検証する。
- `test_document_stats_hidden_for_multibuffer`: `MultiBuffer::set_excerpts_for_path` で組んだ
  複数抜粋のマルチバッファ（`test_go_to_line_uses_buffer_rows_in_multibuffers` と同じ手法）を
  `CursorPosition::set_active_pane_item` に直接渡し、`is_singleton` が偽のときは文書統計の設定を
  有効にしていても `None` のままであることを検証する。このテストは実際に `is_singleton` の
  ガードを外すと失敗することを手元で確認済み（回帰検出力の確認）。
- `test_document_stats_populates_when_enabled_while_open`: 設定オフのまま文書を開き、統計が
  計算されないことを確認した後、その場で設定を有効化し、編集やタブ切り替えを挟まずに統計が
  即座に populate されることを検証する。`cx.observe_global::<SettingsStore>` の購読を
  無効化すると実際に失敗することを手元で確認済み。

`CursorPosition::document_stats()` という `#[cfg(test)]` アクセサを、既存の `selection_stats()`／
`position()` に倣って追加した。

## ビルド・テスト時の既知の落とし穴

`cargo test` を `.checkout/zed` の外に向けた `CARGO_TARGET_DIR` で実行すると、Zed のdev版アセット
読み込み（`util::dev_repo_root()`）がテストバイナリのパスから `.git` を持つ祖先を遡って探すため、
外側の別リポジトリを誤って「チェックアウトの場所」と認識し、`assets/settings/default.json` の
読み込みに失敗して**無改造の Zed ソースでも**全テストが落ちる。`scripts/check` はこれを避けるため
`CARGO_TARGET_DIR` を明示的に unset し、既定の `target/`（`.checkout/zed` 内）を使う。

## 今回のパッチに含めないこと

- 新規UI文字列（"chars"/"blocks" 等の既定ラベル）の翻訳対応。zed-i18n 側の抽出・翻訳パイプラインに委ねる。
- `crates/settings_ui`（設定GUI）への項目追加。
- 本家 `zed-industries/zed` の現行 `main` への追従（リベース）。PR提出時に本家のフォークで対応する。

## zed-i18n への取り込み（2026-09-19、実施済み）

`tools/zed_i18n/apply_universal.py`（`_plan_zed_runtime_patches` 内、`_register_document_stats_patches`
関数）に、この `patches/0001-status-bar-document-stats.patch` と同内容を、zed-i18n 既存の流儀
（`patch(relative, old, new)` による文字列レベルの構造化パッチ、新規ファイルはランタイム
オーバーレイ）で再実装した。zed-i18n は独自の localization エンジンが Python の文字列/AST変換を
前提にしており、`git apply` のような生パッチ適用の仕組みを持たないため、Rustソースの変更内容自体は
このリポジトリの `patches/` を正本としつつ、zed-i18n 側では同じ変更を自分の慣用句で保守する
（`_register_document_stats_patches` の docstring にこのリポジトリへの参照とバージョン
（v1.20.2）を明記し、将来の追従漏れに備える）。

- `crates/go_to_line/src/document_stats.rs`（新規ファイル）は
  `tools/zed_i18n/runtime_overlay/crates/go_to_line/src/document_stats.rs` として全文コピーし、
  既存のオーバーレイ機構（新規ファイル一覧への追加）で配置する。
- 既存ファイル（`cursor_position.rs`／`go_to_line.rs`／`settings_content/workspace.rs`／
  `workspace/workspace.rs`／`workspace/workspace_settings.rs`／`settings/vscode_import.rs`／
  `assets/settings/default.json`）への変更は、すべて `patch(relative, old, new)` の呼び出しとして
  移植した。
- `tests/test_runtime_overlay_patches.py` の `PATCH_TARGETS` に対象ファイルを追加し、
  適用結果とべき等性（2回適用しても2回目は無変更）を検証するアサーションを追加した。
  ローカライズ済みの実チェックアウト全体（zed-i18n の既存75ファイル分の変更）に対しても
  `apply_zed_runtime_patches` がエラーなく適用できることを確認済み。
- 新規UI文字列（既定ラベル）は、この統合作業ではローカライズ抽出パイプラインに一切触れていない
  （`apply_universal()` の文字列分類は本パッチ適用前の状態に対して行われるため、影響しないことを
  ソースを読んで確認済み）。翻訳対応は引き続き未着手。
- ビルドの本流（`generate-runtime-bundles` → `apply-universal` → 配布パッチ）を通したフルビルド
  確認は未実施（`crates/localization` 側の別の前提ステップが必要なため、今回のスコープ外とした）。
