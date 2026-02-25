# ds-issue-create-from-docs
## 目的
設計書を正として実装のギャップを分析し、Issue を分割・起票する。
起票はローカル delta の作成（全フェーズ分）までとし、ユーザーの `/ds-spec-commit` 待ち状態までもっていく。
2 フェーズ構成: Phase A（分割戦略の提案・承認）→ Phase B（delta 一括生成・承認）。

## 入力
なし（$ARGUMENTS は無視）

## 依存/前提チェック（最初に必ず実施）
- `.dev-steroid/state.product.json` が存在
- `.dev-steroid/config.json` の `docs_phase_map` において、`requirements` が `approved`、その他全キーが `approved` または `not_applicable`（= spec-phase 進入ゲート通過済み）

## 参照（Read）
- `.dev-steroid/docs/**`（全 approved 設計書）
- `.dev-steroid/config.json`
- `.dev-steroid/state.product.json`
- `code_paths` / `test_paths` 配下の実装ファイル

## 更新（Write）
- `.dev-steroid/issue-plan.md`（Phase A: 分割戦略。永続ファイル — 再実行時の再利用チェック（A-2）に使用。手動削除可）
- `issues/<Issue名>/state.issue.json`（Phase B: 各 Issue）
- `issues/<Issue名>/*.md`（Phase B: 各 delta）
- `.dev-steroid/state.product.json`（issues[] 追加、current_phase 更新）

## 書き込み制約
- `.dev-steroid/docs/**` は更新しない（設計書は参照のみ）。
- `.dev-steroid/config.json` は更新しない。

---

## Phase A: 分割戦略の提案・承認

### A-1) 前提チェック
- `state.product.json` を読み取り、docs_phase ゲート通過を確認。
- 未達なら blocked で停止（不足一覧と `/ds-doc` での解消手順を提示）。

### A-2) 再開判定
- `.dev-steroid/issue-plan.md` が既に存在する場合:
  - 「既存の分割戦略が見つかりました。再利用しますか？」とユーザーに確認。
  - 再利用 → Phase B に進む（A-3 をスキップ）。
  - 再作成 → 既存の issue-plan.md を上書きして A-3 を実行。

### A-3) gap-analyzer 起動

#### A-3a) No-Code プラットフォームの実装状況収集（platforms に no-code がある場合）
`config.json` の `platforms` に `type: "no-code"` のエントリがある場合、gap-analyzer 起動前にユーザーから現在の実装状況を収集する:

```
Bubble 側の現在の実装状況を教えてください:
- 作成済みページ一覧（ページ名）:
- 作成済みワークフロー一覧（概要）:
- API Connector 設定済み一覧:
- 未実装の機能（設計書にあるがまだ作っていないもの）:
```

回答を `bubble_inventory` パラメータとして gap-analyzer に渡す。
初回（全くの新規開発）の場合は「全て未実装」としてスキップ可能。

#### A-3b) gap-analyzer 起動
subagent を起動：`.claude/agents/dev-steroid/spec-phase/ds-issue-create/ds-issue-create-gap-analyzer.md`

gap-analyzer への指示：
- `approved_docs`：approved な doc_type の一覧と各出力先ディレクトリパス
- `config_path`：`.dev-steroid/config.json`
- `code_paths`：config.json の `code_paths`
- `test_paths`：config.json の `test_paths`
- `bubble_inventory`：（A-3a で収集した情報。no-code プラットフォームがない場合は省略）

gap-analyzer は `.dev-steroid/issue-plan.md` を生成する。

### A-4) ユーザーに分割戦略を提示

以下の形式でサマリを提示する：

```
## ギャップ分析結果
- 設計書の機能/画面/API: N 件
- 実装済み: N 件
- 未実装（Issue 化対象）: N 件

## Issue 分割戦略
| # | Issue名 | スコープ | 優先順 | 依存 | 概要 |
|---|---------|---------|--------|------|------|
| 1 | ... | ... | 1 | なし | ... |
| 2 | ... | ... | 2 | 1 | ... |

## ユーザー操作
- 承認 → Phase B（delta 生成）に進む
- 修正指示 → gap-analyzer を再起動
- Issue の追加/削除/統合/名称変更も可能
```

### A-5) ユーザー承認処理
- **承認** → Phase B に進む
- **修正指示** → gap-analyzer に修正指示を渡して再起動（A-3 に戻る）。issue-plan.md を上書き更新。

---

## Phase B: delta 一括生成・承認

### B-1) issue-plan.md の読み取り
- `.dev-steroid/issue-plan.md` から Issue 一覧を取得する。
- 優先順（依存関係）でソートする。

### B-2) 各 Issue の処理（優先順に逐次）

各 Issue について以下を実行する。コンテキスト管理のため **1 Issue ずつ** 処理する。

#### B-2a) Issue ディレクトリ初期化（ds-spec-init 相当）
- `issues/<Issue名>/` が既に存在する場合：
  - state.issue.json が存在し、artifacts が全て reviewed or approved なら「既存の delta を再利用しますか？」と確認
  - 再利用 → 当該 Issue のレビュー/承認処理（B-2c）にスキップ
  - 再作成 → ディレクトリを削除して以下の初期化を実行
  - state.product.json に当該 Issue が既にあれば、エントリの上書き更新とする
- `issues/<Issue名>/` を作成
- `state.issue.json` を初期化（ds-spec-init と同じスキーマ）:
  - phase: `spec-phase`
  - local_status: `initialized`
  - meta の推定: issue-plan.md のスコープ情報から `impl_type` / `verify_type` / `target_paths` を自動導出
- `.dev-steroid/state.product.json` の `issues[]` に新規エントリを追加
- `current_phase` が `docs-phase` の場合は `spec-phase` に更新

**メタ情報の自動導出ルール:**
- `impl_type`: issue-plan.md のスコープに DB マイグレーションのみ → `no-code`、UI/API あり → `code`、混合 → `hybrid`。判断不能時は `code`（デフォルト）
- `verify_type`: issue-plan.md のスコープにテスト対象パスが含まれる → `automated`、手動検証のみ → `manual`、混合 → `hybrid`。判断不能時は `automated`（デフォルト）
- `target_paths`: config.json の `code_paths` / `test_paths` をコピー（Issue 固有の絞り込みは issue-plan.md の情報があれば適用）

#### B-2b) delta-generator 起動
subagent を起動：`.claude/agents/dev-steroid/spec-phase/ds-issue-create/ds-issue-create-delta-generator.md`

delta-generator への指示：
- `issue_path`：`issues/<Issue名>/`
- `issue_plan_path`：`.dev-steroid/issue-plan.md`
- `target_issue_name`：当該 Issue 名
- `approved_docs`：全 approved 設計書パス
- `config_path`：`.dev-steroid/config.json`

delta-generator は 5 delta を一括生成する：
- `requirements_delta.md`
- `doc_delta.md`
- `design_delta.md`
- `test_delta.md`
- `plan_delta.md`（issue_strategy YAML 含む）

生成後、`state.issue.json` の全 artifacts の status を `draft` に更新する。
`local_status` を `spec_in_progress` に更新する。

#### B-2c) 各 delta を既存 reviewer でレビュー
以下の順序で 5 つの既存 reviewer を起動する（全て Read only の AI レビュー）：

1. `.claude/agents/dev-steroid/spec-phase/ds-spec-requirements/ds-spec-requirements-reviewer.md`
   → `requirements_delta.md` をレビュー
2. `.claude/agents/dev-steroid/spec-phase/ds-spec-doc/ds-spec-doc-reviewer.md`
   → `doc_delta.md` をレビュー
3. `.claude/agents/dev-steroid/spec-phase/ds-spec-design/ds-spec-design-reviewer.md`
   → `design_delta.md` をレビュー
4. `.claude/agents/dev-steroid/spec-phase/ds-spec-test/ds-spec-test-reviewer.md`
   → `test_delta.md` をレビュー
5. `.claude/agents/dev-steroid/spec-phase/ds-spec-plan/ds-spec-plan-reviewer.md`
   → `plan_delta.md` をレビュー

各 reviewer への共通パラメータ：
- PRD パス：`.dev-steroid/docs/**`
- state.issue.json パス：`issues/<Issue名>/state.issue.json`
- 成果物パス：`issues/<Issue名>/<対象 delta>`
- config パス：`.dev-steroid/config.json`

#### B-2d) reviewer の verdict 処理
- **全 delta が approved**:
  - 全 artifacts の status を `reviewed` に更新
  - 次の Issue の処理に進む
- **いずれかが needs_revision**:
  - 指摘を delta-generator に渡して再生成（最大 3 回）
  - 3 回超えたら当該 Issue の `local_status` を `blocked` に設定し、次の Issue に進む（ユーザーに後で報告）
  - 再生成後、再度 reviewer を起動

### B-3) 全 Issue の処理完了後、サマリを提示

```
## 生成結果サマリ
| # | Issue名 | delta 状態 | 分割 | scope 数 | 備考 |
|---|---------|-----------|------|---------|------|
| 1 | setup-db | 全 reviewed | false | 1 | |
| 2 | auth | 全 reviewed | true | 2 | |
| 3 | recipe-crud | blocked | - | - | design_delta レビュー不合格 |

## ユーザー操作
- `承認 all` → 全 reviewed Issue を一括承認
- `承認 1,2` or `承認 setup-db, auth` → 指定 Issue を承認（番号/名前どちらも可）
- `差し戻し 3: <理由>` → 指定 Issue を差し戻し（delta-generator 再実行）
- `詳細 <Issue名>` → 当該 Issue の delta 要約を表示
- `却下 <Issue名>` → 当該 Issue のディレクトリと state を削除
```

### B-4) ユーザー承認処理

#### 承認された Issue について:
- 全 artifacts.*.status を `approved` に更新
- `local_status` を `spec_in_progress` のまま維持（`/ds-spec-commit` で `committed` に遷移）
- `state.product.json` の該当エントリを同期

#### 差し戻された Issue について:
- delta-generator を差し戻し理由付きで再起動
- 再レビュー → 再度サマリ提示（差し戻された Issue のみ。既に承認済みの Issue は表示に含めるが操作対象外）

#### 却下された Issue について:
- `issues/<Issue名>/` ディレクトリを削除
- `state.product.json` の `issues[]` から該当エントリを削除

### B-5) 次のアクション
承認済み Issue ごとに `/ds-spec-commit <Issue名>` を依存順で提示する。

```
## 次のアクション（依存順に実行）
1. /ds-spec-commit setup-db
2. /ds-spec-commit auth      （setup-db 完了後）
3. /ds-spec-commit recipe-crud（setup-db 完了後）
```

---

## State 管理の注意点

### state.issue.json の初期化
ds-spec-init と同じスキーマに準拠する。詳細は `ds-spec-init.md` を参照。

### state.product.json の更新
- Phase B で Issue を初期化するたびに `issues[]` に追加
- `current_phase` は最初の Issue 初期化時に `spec-phase` に更新
- 各 Issue の承認/却下時に `issues[]` を同期

### 複数 state ファイルの更新順序
`state.issue.json` → `state.product.json` の順序を守る（rules の「複数 state ファイルの更新順序」参照）。

## git 操作
このコマンドでは git 操作を行わない（delta はローカルのみ）。
`/ds-spec-commit` で初めて git 操作が発生する。

## 出力（必須フォーマット）
### Phase A 完了時
1) ギャップ分析サマリ
2) Issue 分割戦略（テーブル形式）
3) ユーザー操作: 承認 / 修正指示

### Phase B 完了時
1) 生成結果サマリ（Issue × delta 状態のテーブル）
2) ユーザー操作: 承認 all / 承認 <指定> / 差し戻し <指定>: <理由> / 詳細 <Issue名> / 却下 <Issue名>
3) 次のアクション: `/ds-spec-commit` の実行順
