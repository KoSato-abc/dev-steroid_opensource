# ds-spec-init
## 目的
spec-phase の入口として、Issue ディレクトリと state を初期化し、以降の delta 作成がブレずに進む基盤を用意する（ローカルのみ）。
## 入力
`<Issue名>` — $ARGUMENTS
## 依存/前提チェック（最初に必ず実施）
- `.dev-steroid/state.product.json` が存在
- `.dev-steroid/config.json` の `docs_phase_map` において、`requirements` が `approved`、`env_setup` が `approved`（`platforms` 未定義の場合は `not_applicable` 扱い）、その他全キーが `approved` または `not_applicable`
- `issues/<Issue名>/` が未作成
## 参照（Read）
- `.dev-steroid/state.product.json`
- `.dev-steroid/config.json`
- `.dev-steroid/docs/**`（PRD: `approved` のみ参照。`not_applicable` は参照不要/ファイル不在を許容）
## 更新（Write）
- `issues/<Issue名>/state.issue.json`
- `.dev-steroid/state.product.json`（issues[] 追加）
## 書き込み制約
- `issues/<Issue名>/` には `state.issue.json` のみを作成する。delta ファイルは後続コマンドで生成。
- プロジェクトの実装対象パス（`.dev-steroid/config.json` の `code_paths` / `test_paths`）は絶対に触らない。

## state.issue.json 初期値（スキーマ準拠）
- phase: `spec-phase`
- local_status: `initialized`
- created_at: ISO8601（UTC）
- updated_at: ISO8601（UTC）（以降、各工程完了時や status 変更時に更新）
- meta:
  - impl_type: `code` | `no-code` | `saas` | `hybrid`
    - 自動推定ロジック:
      1. `config.json` の `platforms` を読む
      2. `platforms` に `type: "no-code"` が含まれる場合:
         - `type: "code"` も含まれる → `hybrid`
         - `type: "code"` なし → `no-code`
      3. `platforms` に `type: "code"` のみ → `code`
      4. `platforms` 未定義 → `code`（後方互換）
    - 推定結果をユーザーに提示し承認を得る
  - verify_type: `automated` | `manual` | `hybrid`
    - impl_type が `hybrid` の場合は `hybrid` を推定する
    - impl_type が `no-code` の場合は `manual` を推定する
    - impl_type が `code` の場合は `automated` を推定する
    - 推定結果をユーザーに提示し承認を得る
  - target_paths:
    - code_paths: `.dev-steroid/config.json` の `code_paths` をコピー（Issue 固有の追加/除外があればユーザー指示で調整）
    - test_paths: `.dev-steroid/config.json` の `test_paths` をコピー（同上）
- artifacts:
  - requirements_delta: `{ status: "not_started" }`
  - doc_delta: `{ status: "not_started" }`
  - design_delta: `{ status: "not_started" }`
  - test_delta: `{ status: "not_started" }`
  - plan_delta: `{ status: "not_started" }`
- impl_progress:
  - 分割なし（標準）:
    - doc: `not_started`
    - test_impl: `not_started`
    - code: `not_started`
    - test_exec: `not_started`
  - 分割あり（ds-spec-plan で確定後に scopes 構造へ拡張）:
    - doc: `not_started`
    - scopes:
      - `<scope名>`: `{ test_impl: "not_started", code: "not_started", test_exec: "not_started" }`
  - ※ 初期化時点ではフラット構造で生成する。分割の有無は ds-spec-plan で確定するため、ds-spec-commit が scopes 構造に拡張する。
- remote: null（ds-spec-commit で確定。git.auto=true 時に以下の構造で設定）
  ```json
  {
    "provider": "github",
    "parent": { "url": "...issues/100", "issue_id": 100 },
    "sub_issues": [
      { "type": "doc",    "issue_id": 101, "url": "...", "scope": "all" },
      { "type": "design", "issue_id": 102, "url": "...", "scope": "all" },
      { "type": "test",   "issue_id": 103, "url": "...", "scope": "all" }
    ]
  }
  ```

## product state の更新
- `.dev-steroid/state.product.json` の `issues[]` に新規要素を追加（既存スキーマ準拠：必要最小限）
  - issue_key: `<Issue名>`
  - phase: `spec-phase`
  - local_status: `initialized`
  - remote: null
- `current_phase` が `docs-phase` の場合は `spec-phase` に更新する（既に `spec-phase` 以上なら維持）

## 実行手順
1) `.dev-steroid/config.json` と `state.product.json` を読み、依存ゲート（docs-phase 全項目の approved/not_applicable）を確認。未達なら blocked で停止（不足一覧と解消手順を提示）。
2) `issues/<Issue名>/` が既に存在する場合は停止して報告。
3) PRD（`.dev-steroid/docs/**` の approved 分）と `config.json`（`platforms` 定義）を参照し、Issue のメタ情報の初期値を自動推定する（上記 meta の自動推定ロジックに従う）。
4) `state.issue.json` を初期値で生成する（impl_progress は初期化時点ではフラット構造）。
5) `state.product.json` の `issues[]` に新規エントリを追加し、`current_phase` を必要に応じて更新する。
6) メタ情報の承認依頼を出力する。

## 出力（必須フォーマット）
1) 実行結果（更新/生成ファイル）
- 変更した/生成したファイルパスを列挙
2) state 要点（該当キー）
- 関連する status / phase / local_status を要約
3) メタ情報の承認依頼
- 推定したメタ情報（impl_type / verify_type / target_paths）を提示
- 承認事項（実装形態/検証形態/対象パスが正しいかユーザーが判断）
- ユーザー操作：`承認` / `修正: <指示>`
- `承認` → メタ情報を確定し、次アクションを提示
4) 次のアクション
- 通常：`/ds-spec-requirements <Issue名>`
