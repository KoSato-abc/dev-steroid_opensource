# ds-doc
## 目的
指定された `<doc_type>` のPRDをテンプレ準拠で生成し、AIレビュー→ユーザー承認まで遷移させる。
`all` の場合は未完了分を順次実行し、各完了ごとにユーザー承認を求める。
## 入力
`<doc_type>` — $ARGUMENTS。`.dev-steroid/config.json` の `docs_phase_map` のキー。`all` の場合は未完了分を順次実行。
## 依存/前提チェック（最初に必ず実施）
- `/ds-init` 済み
- `.dev-steroid/config.json` の `docs_phase_map` に `<doc_type>` が存在する
- 依存先（`depends_on`）の status が `approved` または `not_applicable`
## 参照（Read）
- `.dev-steroid/state.product.json`
- `.dev-steroid/config.json`
- テンプレ：`{template_root}/prd-{doc_type_hyphen}/**`（doc_type のアンダースコアをハイフンに変換）
- 既存ドキュメント（存在する場合）
## 更新（Write）
- `.dev-steroid/docs/{dir}/**`（config から解決される出力ディレクトリ）
- `.dev-steroid/state.product.json`（該当 status/updated_at）
## 書き込み制約
- docs-phase は `.dev-steroid/docs/**` と `.dev-steroid/state.product.json` 以外に書き込まない。

## パス解決（config.json から動的に導出）
- テンプレ：`docs_phase_map` エントリに `template` 属性がある場合は `{template_root}/{template}/**` を使用する。`template` 属性がない場合は従来の規約 `{template_root}/prd-{doc_type_hyphen}/**`（`doc_type` のアンダースコアをハイフンに変換。例：`basic_design` → `prd-basic-design`）にフォールバック。
- 出力先（単一ファイル型）：`{docs_root}/{dir}/{doc_type}.md`（`dir` は config のエントリから取得）
- 出力先（分割型）：`{docs_root}/{dir}/index.md`（エントリポイント）＋ `{docs_root}/{dir}/*.md`（サブファイル）
- テンプレの `generation_hints` に「ファイル分割必須」がある場合は分割型を使用する（basic_design, detailed_design が該当）
- 既存ドキュメント：出力先ディレクトリ配下の全 .md ファイル（存在する場合）

## 対象外（not_applicable）処理
- ユーザーが「このドキュメントは不要/対象外」と**明示**した場合は、生成・レビューを行わず `status = not_applicable` に更新する。承認サイクル（reviewed→承認待ち）は経由しない（`all` の場合は即座に次の doc_type へ進む）。
- **ただし `requirements` は not_applicable にできない**（常に必須）。ユーザーが対象外を要求した場合は、その旨を説明して blocked。
- 出力には「対象外とした理由（1〜3行）」と「次のアクション」を必ず含める。

## 実行手順（単一 doc_type）
1) config.json からパスを解決する
2) **env_setup ガード**: `doc_type` が `env_setup`（または `docs_phase_map` エントリに `"type": "env_setup"` がある）場合は、テンプレベースの PRD 生成を行わない。代わりに「`/ds-env-setup` を実行してください」と案内して停止する。
3) runner（subagent）を起動：`.claude/agents/dev-steroid/docs-phase/ds-doc/ds-doc-runner.md`
   - runner への指示：テンプレパス、出力先パス（分割型の場合は出力先ディレクトリ）、既存ドキュメントパスを明示的に伝える
   - テンプレの `generation_hints` に「ファイル分割必須」がある場合は分割を指示する
4) runner 完了後、`state.product.json` の `docs_phase.{doc_type}.status` を `draft` に更新
5) reviewer（subagent）を起動：`.claude/agents/dev-steroid/docs-phase/ds-doc/ds-doc-reviewer.md`
   - reviewer への指示：テンプレパス、成果物パスを明示的に伝える
6) reviewer の verdict に応じて：
   - `needs_revision`：runner に指摘を渡して再生成（最大3回）
   - `approved`：status を `reviewed` に更新し、ユーザー承認待ちの出力を提示
7) ユーザーが `承認` → status を `approved` に更新

## 実行手順（all）
1) config.json の `docs_phase_map` を `order` 順に走査する
2) 各 doc_type について：
   - status が `approved` or `not_applicable` → スキップ
   - `docs_phase_map` エントリに `"type": "env_setup"` がある場合 → テンプレベースの生成は行わず、「全設計書の承認が完了しました。次に `/ds-env-setup` を実行して開発環境を構築してください」と案内して停止する
   - それ以外 → 上記「単一 doc_type」の手順を実行
   - **1つの doc_type が `reviewed` に到達したら、ユーザーに承認を求めて停止する**
   - ユーザーが承認したら、次の doc_type に進む
3) 全 doc_type が `approved` or `not_applicable` になったら完了

### all モードの中断と再開
- 承認待ちで停止した後、ユーザーが承認したら**同一コマンド `/ds-doc all` を再実行**して残りの doc_type を続行する
- 次のアクション出力には、承認後に実行すべきコマンドとして `/ds-doc all` を必ず含める
- 再実行時は `approved` / `not_applicable` の doc_type を自動スキップするため、途中からの継続となる

## git 操作（git.auto=true の場合）
ユーザー承認（`approved`）後:
1) staging: `.dev-steroid/docs/{dir}/`, `state.product.json`（ディレクトリ内の全ファイルを staging）
2) commit: `docs(<doc_type>): <要約>`
- git.auto=false の場合はスキップ（commit なし）
- **`cd` は使わず、プロジェクトルートから相対パスで `git add` する**

## 出力（必須フォーマット）
1) 実行結果（更新/生成ファイル）
- 変更した/生成したファイルパスを列挙
2) state 要点（該当キー）
- `docs_phase.{doc_type}.status` / `updated_at` を要約
3) 次のアクション
- 承認待ちの場合：作業要約 / 検証状況 / 承認事項 / ユーザー操作（承認/差し戻し/保留）
- 次の doc_type がある場合：`/ds-doc {next}`
- env_setup が残っている場合：`/ds-env-setup`
- 全完了の場合（env_setup も approved / not_applicable）：`/ds-spec-init <Issue名>`
