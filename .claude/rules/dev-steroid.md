# dev-steroid
Claude Code の commands / agents 構造を使い、次の3フェーズを state で制御する運用キットです。
- docs-phase：PRD（要件〜テスト設計）をローカル生成・AIレビュー・ユーザー承認 + 環境構築（`/ds-env-setup`）
- spec-phase：Issue（ローカル）を作り、5つの delta（差分）をローカルで確定し、GitHub Issues 登録（親 Issue + sub-issue）まで
- issues-phase：sub-issue を根拠に実装（doc→scope別ループ[test_impl→code→test_exec]→PR）
  - ①doc：納品物PRD（`.dev-steroid/docs/**`）を doc delta に合わせて更新
  - ②test_impl：ACに基づくテストを先に実装（RED）。no-code scope は手動検証手順書を確定。
  - ③code：テストを通す最小実装（GREEN）→ 改善（REFACTOR）。no-code scope は操作指示書を生成し、ユーザーが手動実行。
  - ④test_exec：全テスト実行 + 回帰確認。失敗時は③に戻って修正→④再実行。no-code scope は手動検証 + 証跡記録。
  - 検証形態（automated/manual/hybrid）は `state.issue.json` の `meta.verify_type` で宣言する。
  - hybrid の場合、code scope 完了後に **deploy_gate**（クラウド反映待ち）を挟み、その後 no-code scope を実行する。
## Bash 運用ルール（必須）
- **`cd` コマンドを使わない**。全ての Bash コマンドはプロジェクトルートからの相対パスで実行する。
- **`&&` / `||` / `;` で複合コマンドを組まない**。各コマンドは個別の Bash 呼び出しで実行する（permission パターンが複合コマンドにマッチしないため）。
- 例：`cd .dev-steroid && git add .` → NG。`git add .dev-steroid/` → OK。

## 参照ディレクトリ
- PRD出力：`.dev-steroid/docs/**`（docs-phase は `.dev-steroid/docs/**` と `.dev-steroid/state.product.json` 以外に書かない）
- PRDテンプレ：`.dev-steroid/template/**`
- state：`.dev-steroid/state.product.json` と `issues/<Issue>/state.issue.json` のみ（Issue のメタ情報は `state.issue.json` の `meta` に格納）。Issue ディレクトリは `issues/` 固定。
- Issue成果物：`issues/<Issue>/**`（spec-phase は `issues/<Issue>/**` と `.dev-steroid/state.product.json` 以外に書かない。例外：`/ds-issue-create-from-docs` は `.dev-steroid/issue-plan.md` にも書く）
- 操作指示書：`issues/<Issue>/guides/<scope>_operation_guide.md`（no-code scope の ③code 工程で生成）
- 検証証跡：`issues/<Issue>/evidence/<scope>_evidence.md`（no-code scope の ④test_exec 工程で生成）
- 実装（変更許可パス）：`.dev-steroid/config.json` の `code_paths`（既定：`supabase/migrations/**`, `supabase/functions/**`, `src/**`）と `test_paths`（既定：`supabase/tests/**`, `tests/**`）。issues-phase のみ更新。
## コマンド一覧（本キットに含むもののみ）
- 初期化：`/ds-init`
- docs-phase：`/ds-doc <doc_type>`（doc_type: requirements / basic_design / detailed_design / data_design / security_design / test_design / all） `/ds-doc-refactoring`（設計書横断リファクタリング。全 approved 設計書を再レビューし修正） `/ds-env-setup`（全設計書 approved 後に実行。platforms 定義に基づく開発環境構築）
- spec-phase：`/ds-spec-init <Issue名>` `/ds-spec-requirements <Issue名>` `/ds-spec-doc <Issue名>` `/ds-spec-design <Issue名>` `/ds-spec-test <Issue名>` `/ds-spec-plan <Issue名>` `/ds-spec-commit <Issue名>` `/ds-issue-create-from-docs`（設計書→Issue 一括起票。分割戦略提案→delta 一括生成→ユーザー承認）
- issues-phase：`/ds-issue-impl <Issue名>`（doc→scope別[test_impl→code→test_exec]→PR） `/ds-issue-status <Issue名>`

## spec-phase の流れ
spec-phase は ds-spec-init → requirements → doc → design → test → plan → commit の順に 5 つの delta を作成し、最後に GitHub Issues を登録する。各コマンドの体系図・delta 構成表・Sub-issue 構造の詳細は `.dev-steroid/README.md` を参照。

## Git 運用
`config.json` の `git.auto`（`true` / `false`）と `base_branch` で制御する。

- **git.auto=true**: docs-phase で commit、spec-phase で Issue 登録（GraphQL 紐付け）、issues-phase で feature branch → 工程別 commit → PR 作成。Conventional Commits 形式。
- **git.auto=false**: Issue 登録・git 操作なし。仕様参照はローカル delta を直接 Read。`remote` は null。delta は issues-phase 完了まで永続。

各フェーズの git 操作詳細は `ds-doc.md` / `ds-spec-commit.md` / `ds-issue-impl.md` を参照。

## Git コミットルール
- コミットメッセージは必ず1行で `-m "..."` を使うこと
- Co-Authored-By は `-m` フラグを追加して同一行コマンドにすること
  例: git commit -m "docs: 要件定義書を承認" -m "Co-Authored-By: Claude <noreply@anthropic.com>"
- 改行を含むコミットメッセージは絶対に使わないこと（パーミッション制御が効かなくなるため）

## フェーズ遷移のルール（必須）
- spec-phase に進むには、`.dev-steroid/config.json` の `docs_phase_map` において `requirements` が `approved`、`env_setup` が `approved`（`platforms` 未定義の場合は `not_applicable` として扱う）、その他全キーが `approved` または `not_applicable`。
- issues-phase に進むには、当該 Issue の artifacts（requirements/doc/design/test/plan）が全て `approved` であり、かつ `local_status` が `committed` であること。git.auto=true の場合はさらに `remote.parent` が存在すること。
## issues-phase の品質ルール（必須）
- **検証無し変更は禁止**（ACごとに **自動テスト** または **手動検証** の証跡が必要）
- 検証は「壊れにくさ」と「失敗時の診断性」を重視し、最終的に回帰（関連範囲）まで含めた結果を残す
- 検証形態（automated/manual/hybrid）と対象パスは `state.issue.json` の `meta` で宣言する
- **工程順序**：①doc（PRD更新）→ scope 別ループ[ ②test_impl（テスト実装/RED）→ ③code（実装/GREEN→REFACTOR）→ ④test_exec（テスト実行/回帰）]
- ④で失敗した場合は③に戻って修正 → ④再実行（最大3回）
- 分割時は scope ごとに②③④をループし、全 scope 完了後に PR を作成する
- **no-code scope の工程**: ②手動検証手順書確定 → ③操作指示書生成+ユーザー手動実行 → ④手動検証+証跡記録
- **deploy_gate**: hybrid の場合、code scope 完了後・no-code scope 開始前にクラウド反映ゲートを設ける

## issues-phase の仕様参照方法
各工程で sub-issue の仕様を参照する際は `config.json` の `git.auto` に応じて分岐する:
- **git.auto=true**: `gh issue view <sub_issue_id> --json body -q .body`（sub_issue_id は `state.issue.json` の `remote.sub_issues` から取得）
- **git.auto=false**: `issues/<Issue名>/<delta>.md` を直接 Read（分割時は delta 内の該当 sections を参照）

## 役割の扱い（commands / subagent）
- `commands/*.md`：手順書（統括）。ユーザーのスラッシュコマンドで起動し、runner/reviewer の subagent を管理する。**state ファイルの更新は commands（main agent）のみが行う。**
- `agents/**/*-runner.md` / `*-reviewer.md`：subagent（YAML frontmatter を持つ）。runner は生成/変更、reviewer は監査（Bash による検証実行も可）。**subagent は `.dev-steroid/state.product.json` および `issues/*/state.issue.json` を絶対に更新しない。**

## subagent 起動の標準パラメータ（全コマンド共通）
subagent を起動する際は、以下の形式で指示を渡す：
- `issue_path`：`issues/<Issue名>/`（該当する場合）
- `template_path`：テンプレートディレクトリ（該当する場合）
- `artifact_path`：成果物の出力先ファイルパス
- `reference_paths`：参照すべきファイルパスのリスト（PRD、delta 等）
- `review_feedback`：差し戻し時のレビュー指摘（該当する場合）
- `scope`：実装対象の scope 名（分割時、issues-phase の agent に渡す）
- `git_auto`：`config.json` の `git.auto` 値（仕様参照方法の分岐に使用）

## 共通の出力要件（全コマンド必須）
1) 実行結果（更新/生成ファイル）
2) state 要点（該当キーの status）
3) 次のアクション
- 次に実行すべきコマンドを提示。
- **承認待ち（status=reviewed）の場合**は、必ず「作業要約」「検証状況」「承認事項」「ユーザー操作（承認/差し戻し/保留）」を出力に含める。

## 承認フロー（必須）
- AIレビューが `approved` → state は `reviewed`（承認待ち）に遷移。`approved` にはならない。
- 承認待ちの出力には「作業要約」「検証状況」「承認事項」「ユーザー操作（承認/差し戻し/保留）」を必ず含める。
- ユーザーの `承認` は、**直前に承認依頼を出した成果物**にのみ適用する。

## 再実行ルール（全コマンド共通）
- status が `approved` の成果物に対してコマンドを再実行した場合：
  - 「既に approved です。上書きして再生成しますか？」とユーザーに確認する。
  - ユーザーが明示的に再生成を指示した場合のみ、status を `not_started` に戻して実行する。
  - 確認なしに上書きしない。

## state 更新の安全手順（全コマンド共通）
1. state ファイルを読み取る
2. 更新対象のキーのみを変更する（他のキーは保持）。`updated_at` が存在するファイルでは ISO8601（UTC）で同時更新する。
3. 書き込み後、再度読み取って意図した値になっていることを確認する
4. 確認できない場合は停止し、ユーザーに報告する

### 複数 state ファイルの更新順序（必須）
`state.issue.json` と `state.product.json` の両方を更新するコマンドでは、以下の順序を守る:
1. `state.issue.json` を先に更新する（詳細情報＝信頼できる情報源）
2. 書き込み後の検証を完了してから `state.product.json` を更新する（要約情報＝派生）
3. 中断・エラー時は `state.issue.json` を正とし、`state.product.json` は次回コマンド実行時に自動修復する

> **タイムスタンプの命名規則**: `state.product.json` のトップレベルは `last_updated`（ds-init が管理）、各エントリ（`docs_phase.*` / `state.issue.json`）は `updated_at`（各コマンドが管理）。

## 用語（state語彙）と遷移
- `not_started`：未着手（未生成）。
- `draft`：生成直後（未レビュー）。
- `reviewed`：AIレビュー完了（承認待ち）。
- `approved`：ユーザー承認（ゲート通過）。
- `not_applicable`：対象外（このプロジェクト/Issueでは不要。依存ゲートは通過扱い）。※ docs-phase の `requirements` には使用不可（常に必須）。

### docs_phase.*.status（PRD単位）の遷移
not_started → draft → reviewed → approved
（ユーザーが「対象外」と明示 → not_applicable）

### issue の local_status（Issue単位）の遷移
initialized → spec_in_progress → committed → implementing → done
（任意の状態から blocked に遷移可能。原因解消後に直前の状態に復帰）

> **注意**: `blocked` は Issue（`local_status`）固有の状態語彙。`ds-init` 等の Issue 前のコマンドではエラー時に state を更新せず停止する。

### impl_progress の状態語彙
- `not_started`：未着手
- `in_progress`：実行中
- `waiting_user`：ユーザーの手動操作完了待ち（no-code scope の ③code 完了後、deploy_gate のクラウド反映待ちで使用）。ユーザーが「完了」と報告した時点で `done` に遷移する。
- `done`：完了
- 分割時は `impl_progress.scopes.<scope名>.{test_impl|code|test_exec}` で scope ごとに管理する
- hybrid の場合、`impl_progress.scopes.deploy_gate.status` で code scope → no-code scope 間のクラウド反映ゲートを管理する

### current_phase の遷移ルール
- `docs-phase` → `spec-phase`：ds-spec-init 実行時に自動遷移（前提：docs ゲート通過）
- `spec-phase` → `issues-phase`：ds-spec-commit 成功時に自動遷移
- 一度 issues-phase に遷移した後も、新規 Issue の ds-spec-init は実行可能（current_phase は最も進んだフェーズを表す）

## state.issue.json
完全なスキーマと初期値は `ds-spec-init.md` を参照。主要構造のみ以下に示す:
- `phase` / `local_status` / `meta`（impl_type / verify_type / target_paths）/ `artifacts`（5 delta の status）/ `impl_progress` / `remote`
- `remote` と `impl_progress.scopes` の詳細は `ds-spec-commit.md` を参照

## カスタマイズ
カスタマイズポイント一覧（config.json / settings.json / テンプレート / agent YAML 等）は `.dev-steroid/README.md` のカスタマイズポイント表を参照。

## 拡張
現状は commands（スラッシュコマンド）中心の構成を維持する。必要になったタイミングで skills に寄せてもよい。

## impl_type による分岐（no-code / hybrid 対応）

### platforms 定義と impl_type の関係
`config.json` の `platforms` 定義と `state.issue.json` の `meta.impl_type` に基づき、各フェーズの挙動が分岐する。

- `platforms` 内の `type: "code"`: ファイル R/W で実装可能。AI が自律的に完結する。
- `platforms` 内の `type: "no-code"`: GUI 操作が必要。AI は操作指示書を生成し、人間が手動実行する。
- `platforms` 未定義時は全て code として動作する（後方互換）。

### impl_type 挙動一覧
| impl_type | docs-phase | spec-phase | issues-phase |
|---|---|---|---|
| `code` | 既存テンプレート | 既存 delta 構成 | file R/W + 自動テスト（現状通り） |
| `no-code` | 専用テンプレート | delta に「操作手順」セクション追加 | ③code → 操作指示書。④test_exec → 手動証跡 |
| `hybrid` | 専用テンプレート | scope ごとに platform 属性付き | scope 単位で code パス / no-code パスを切替 |
| `saas` | no-code と同等 | no-code と同等 | no-code と同等 |

### docs-phase テンプレート選択
`config.json` の `docs_phase_map` 各エントリの `template` 属性でテンプレートを解決する。`platforms` の構成に応じて専用テンプレート名を設定しておく。

### spec-phase の分岐
- `ds-spec-init` で `impl_type` を自動推定（platforms に no-code あり + code あり → hybrid、no-code のみ → no-code、code のみ or platforms 未定義 → code）。
- `ds-spec-design` の design_delta に no-code セクション（ページ/ワークフロー変更一覧、操作順序）を追加。
- `ds-spec-test` の test_delta に手動検証シナリオ + 実施条件を追加。
- `ds-spec-plan` の plan_delta issue_strategy YAML に `platform` 属性と `scope_order`（deploy_gate 含む）を追加。

### issues-phase の分岐（scope 単位）
command が scope の platform.type を判定し、工程の成果物を切替える:
- **type=code**: 既存の code-runner/reviewer で file R/W + 自動テスト。
- **type=no-code**: guide-runner/reviewer で操作指示書生成 + ユーザー手動実行 + 証跡記録。
  - ③code 完了後に `waiting_user` 状態になり、ユーザーの手動実行完了報告を待つ。
  - ④test_exec でも同様に、ユーザーの手動検証結果報告を待つ。

### deploy_gate（hybrid 専用）
code scope と no-code scope の間にクラウド反映ゲートを設ける。

- 配置: code scope 全完了後、no-code scope 開始前
- 動作: AI がクラウド反映手順（`config.json` の `platforms.*.deploy_commands`）をユーザーに案内し、完了報告を待つ
- 状態: `impl_progress.scopes.deploy_gate.status`: `not_started` → `waiting_user` → `done`
- 案内内容: DB マイグレーション（リモート）、Functions デプロイ、API Connector 接続先確認

### /ds-env-setup（docs-phase 末尾）
全設計書が approved になった後に実行する環境構築コマンド。

- `/ds-init` 時点では技術スタックが不明なため、環境構築は設計確定後に行う。
- `platforms` 定義に基づき、code プラットフォーム（CLI チェック、ディレクトリ作成等）と no-code プラットフォーム（URL・接続情報確認）の環境を準備する。
- `platforms` 未定義の場合は `env_setup` を `not_applicable` として扱い、スキップ可能。
- 環境構築記録を `.dev-steroid/docs/70_env_setup/env-setup-record.md` に出力する。
