# ds-init
## 目的
`.dev-steroid/state.product.json` の存在保証・整合修復を行い、以後のフェーズで参照する基盤 state を整える。
## 入力
なし（引数不要: $ARGUMENTS は無視する）
## 依存/前提チェック（最初に必ず実施）
- `.dev-steroid/config.json` が存在すること（無ければ停止し、作成手順を案内する）
## 参照（Read）
- `.dev-steroid/config.json`
- `.dev-steroid/state.product.json`（存在する場合）
## 更新（Write）
- `.dev-steroid/state.product.json`（生成または整合修復）
## 禁止
- `.dev-steroid/state.product.json` 以外のファイルを更新しない（`issues/**`, `.dev-steroid/docs/**`, `.dev-steroid/config.json`, ならびに `code_paths` / `test_paths` の対象範囲を含む）。

## 実行手順

### 共通チェック（git.auto 問わず）
1. `.dev-steroid/config.json` を読み、以下の必須キーを確認する（欠落/不正なら停止し、不足内容と解消手順を報告する。state は更新しない）：
   - `schema_version`
   - `git`（`auto` / `base_branch` が存在）
   - `docs_root` / `template_root` / `state_product_path`
   - `docs_phase_map`（最低1エントリ。`requirements` キーは必須。各エントリに `order` / `dir` が存在。`template` 属性の有無は任意だがあれば記録）
   - `code_paths` / `test_paths`（各1要素以上の配列）
   - `platforms`（任意。存在する場合は構造確認のみ。**platforms 固有の CLI チェック（supabase CLI 等）はここでは行わない。** 設計確定後の `/ds-env-setup` で実施する）
2. 環境チェック（Bash で直接実行）:
   - `git --version` が成功すること（未導入なら導入手順を案内）
   - `node --version` が成功すること（Node.js 18+ を推奨。未導入なら案内）
   - いずれかが失敗した場合は停止し、解消手順を報告する。state は更新しない。
3. `config.json` の `git.auto` 値を確認し、以降の分岐を決定する。

### git.auto=true の場合のみ
4. **sub-issue API チェック（GraphQL）**:
   - `gh api graphql -f query='{ __type(name: "AddSubIssueInput") { name } }'` で `addSubIssue` mutation が利用可能かを確認する。
   - 失敗した場合は停止し、「GitHub の Sub-issues 機能が利用できません」と報告する。
5. **GitHub CLI 認証チェック**:
   - `gh --version` が成功すること（未導入なら導入手順を案内）
   - `gh auth status` で認証済みであること（未認証なら `gh auth login --web` 等で認証を案内）
   - `git remote get-url origin` が GitHub を指していること
   - `gh repo view` が成功すること（権限不足ならトークン/SSO/Org設定を見直す）

### git.auto=false の場合
4. git/gh 関連チェックは**全てスキップ**する。

### 共通の state 処理（git.auto 問わず）
7. `state_product_path`（既定：`.dev-steroid/state.product.json`）が無ければ作成する。
8. 既存 state がある場合は **整合修復**のみ行う（値の意味を変えない範囲で、必須キーの欠落補完・不正値の是正）。
   - 必須キー：`schema_version, project_name, last_updated, current_phase, docs_phase, issues`
   - `current_phase` は `"docs-phase" | "spec-phase" | "issues-phase"` のいずれか
   - `docs_phase` は `.dev-steroid/config.json` の `docs_phase_map` の全キーに対応するエントリを持つ
   - `docs_phase_map` に `env_setup` エントリがある場合、`docs_phase.env_setup` を同様に管理する。`platforms` 未定義の場合は `env_setup.status` を `not_applicable` に自動設定する。
   - status は `"not_started" | "draft" | "reviewed" | "approved" | "not_applicable"` のいずれか
   - `issues` は配列。要素は `{issue_key, phase, local_status, remote}` 形式に整える（不明な値は保持しつつ、出力で注意点として列挙）
9. `last_updated` を ISO8601（UTC）で更新する。
10. **次のアクション**として `/ds-doc requirements` を提示する。

## 出力（必須フォーマット）
1) 実行結果（更新/生成ファイル）
- 変更した/生成したファイルパスを列挙
- git.auto の値と、チェック結果の要約
2) state 要点（該当キー）
- `current_phase` / `docs_phase.*.status` / `issues[]`（存在する場合は phase/local_status/remote）を要約
3) 次のアクション
- 次に実行すべきコマンドを1つ以上提示（依存未達なら解消手順）
