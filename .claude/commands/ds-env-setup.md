# ds-env-setup
## 目的
docs-phase の設計書が確定した後、設計内容に基づいて開発環境を構築する。`config.json` の `platforms` 定義に応じた CLI ツール・ディレクトリ・設定ファイルをセットアップし、issues-phase の実装に進める状態にする。

## 入力
なし（$ARGUMENTS は無視）

## 依存/前提チェック（最初に必ず実施）
- `.dev-steroid/state.product.json` が存在
- `.dev-steroid/config.json` の `docs_phase_map` において、`env_setup` を除く全キーが `approved` または `not_applicable`（= 全設計書が確定済み）
- `docs_phase.env_setup.status` が `approved` でないこと（`approved` なら再実行確認ルールに従う）

## 参照（Read）
- `.dev-steroid/state.product.json`
- `.dev-steroid/config.json`（`platforms` 定義）
- `.dev-steroid/docs/**`（approved 設計書。技術スタック確認用）

## 更新（Write）
- プロジェクトルート直下のディレクトリ・ファイル（Supabase 初期化で生成されるもの等）
- `scripts/deploy.sh`（クラウド反映手順スクリプト）
- `.dev-steroid/docs/70_env_setup/env-setup-record.md`（環境構築記録）
- `.dev-steroid/state.product.json`（`docs_phase.env_setup`）
- `.gitignore`（必要に応じて追記）

## 書き込み制約
- 既存の設計書（`.dev-steroid/docs/10_requirements/` 〜 `60_test_design/`）は変更しない
- `.dev-steroid/config.json` は変更しない（設定変更が必要な場合はユーザーに案内のみ）
- `.env` ファイルは作成しない（秘密情報は人間が設定）

## 実行手順

### 1) 設計書から技術スタックを確認
1. `config.json` の `platforms` 定義を読む
   - `platforms` が未定義（キーが存在しない or 空オブジェクト）の場合は、環境構築不要と判断する。`docs_phase.env_setup.status` を `not_applicable` に更新し、「platforms 未定義のため環境構築をスキップします。次に `/ds-spec-init <Issue名>` を実行してください」と報告して終了する。
2. approved 設計書（特に requirements, basic_design, data_design）から使用技術・外部サービスの一覧を抽出
3. `platforms` 定義との突合を行い、必要な環境を特定

### 2) プラットフォーム別の環境チェック・セットアップ

#### Supabase（type: code）の場合

a) **CLI チェック**:
   - `supabase --version` → 未導入なら導入手順を案内して停止
   - Docker デーモンの起動確認（`docker info` など） → 未起動なら案内

b) **プロジェクト初期化**（ユーザーに実行を案内し、完了確認を取る）:
   - `supabase/` ディレクトリ未作成なら `supabase init` の実行を案内
   - `supabase start` でローカル DB 起動を案内
   - `supabase status` で起動確認
   - **注意**: `supabase init` と `supabase start` は環境に大きな変更を加えるため、AI が自動実行せずユーザーに実行を案内し、完了確認を取る。

c) **ディレクトリ構造の作成**（AI が直接作成）:
   - `supabase/migrations/`（`supabase init` で作成済みなら確認のみ）
   - `supabase/functions/`
   - `supabase/functions/_shared/`
   - `supabase/tests/`
   - `src/types/`（型定義の出力先）
   - `tests/functions/`（Edge Functions テスト）
   - `scripts/`（ユーティリティ）

d) **デプロイスクリプト生成**:
   - `scripts/deploy.sh` を生成。内容:
     ```bash
     #!/bin/bash
     set -euo pipefail
     echo "=== Supabase クラウド反映 ==="
     echo "1. DB マイグレーション反映..."
     supabase db push --linked
     echo "2. Edge Functions デプロイ..."
     supabase functions deploy
     echo "=== 完了 ==="
     echo "Bubble の API Connector URL がクラウド Supabase を指していることを確認してください。"
     ```

e) **git.auto=true の場合**:
   - `gh auth status` → 未認証なら案内
   - `gh repo view` → リポジトリアクセス確認
   - sub-issue API チェック（GraphQL テストクエリ）
   - ※ これらは `/ds-init` で実施済みであれば確認のみ。

f) **.gitignore 更新**:
   - 以下が `.gitignore` に含まれていなければ追記:
     - `supabase/.temp/`
     - `.env`
     - `.env.*`

#### Bubble（type: no-code）の場合

a) **環境チェック**（ユーザーへの質問形式）:
   - Bubble アプリの URL
   - API Connector プラグインの導入状況
   - 開発環境用の Supabase プロジェクト（クラウド）の有無
   - Supabase プロジェクトの URL と anon key の設定方針

b) **成果物ディレクトリ**: 
   - `issues/` 配下の `guides/` と `evidence/` は `/ds-issue-impl` 実行時に自動作成されるため、ここでは作成不要。

### 3) 環境構築記録の生成

`.dev-steroid/docs/70_env_setup/env-setup-record.md` を生成する:

```markdown
# 環境構築記録

## 構築日時
（ISO8601 UTC）

## プラットフォーム

### Supabase
- CLI バージョン:
- Docker: 起動済み / 未起動
- ローカル DB: localhost:54322
- プロジェクト Ref:（クラウド連携設定済みの場合）
- リモート URL:（クラウド連携設定済みの場合）

### Bubble
- アプリ URL:
- API Connector: 導入済み / 未導入

## ディレクトリ構造
（作成したディレクトリの一覧）

## クラウド反映手順
1. `scripts/deploy.sh` を実行（または手動で以下を実行）:
   - `supabase db push --linked`（テーブル + RLS 反映）
   - `supabase functions deploy`（Edge Functions デプロイ）
2. Bubble の API Connector でリモート Supabase URL を設定

## 注意事項
- `.env` ファイルの内容はこのファイルに記載しない
- Supabase の service_role key は絶対に公開しない
```

### 4) state 更新
- `state.product.json` の `docs_phase.env_setup.status` を `reviewed` に設定
- `docs_phase.env_setup.updated_at` を ISO8601（UTC）で更新

### 5) 承認待ち出力
- 環境構築結果と記録を提示
- 作業要約 / 検証状況 / 承認事項 / ユーザー操作（承認 / 差し戻し / 保留）

### 6) 承認後
- `docs_phase.env_setup.status` → `approved`
- `docs_phase.env_setup.updated_at` を更新
- **[git.auto=true]** commit:
  - ステージング対象: 作成した全ディレクトリ・ファイル + `env-setup-record.md` + `.gitignore`（変更があれば）+ `state.product.json`
  - `git commit -m "chore(env): 開発環境セットアップ"`

## 出力（必須フォーマット）
1) 実行結果（作成/確認したファイル・ディレクトリ）
2) state 要点（`docs_phase.env_setup.status`）
3) 承認待ちの場合: 作業要約 / 検証状況 / 承認事項 / ユーザー操作
4) 次のアクション
- 承認後: `/ds-spec-init <Issue名>` で spec-phase へ
