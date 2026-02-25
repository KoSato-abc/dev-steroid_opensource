# ds-issue-status
## 目的
Issue の state を読み取り、現在地・未達条件・次アクションを可視化する（読み取りのみ）。
## 入力
`<Issue名>` — $ARGUMENTS
## 依存/前提チェック（最初に必ず実施）
- `issues/<Issue名>/state.issue.json` が存在
## 参照（Read）
- `issues/<Issue名>/state.issue.json`
- `.dev-steroid/state.product.json`（参照）
- `.dev-steroid/config.json`（git セクション参照）
## 更新（Write）
なし（読み取りのみ）
## 注意
- **禁止**：いかなるファイルも更新しない。

## 実行手順
1) `issues/<Issue名>/state.issue.json` を読み取る。
2) 必要に応じて `.dev-steroid/state.product.json` と `.dev-steroid/config.json` も参照する。
3) 以下を整理して出力する。

## 出力（必須フォーマット）
1) 現在地
- phase / local_status
- git.auto の値
2) メタ情報
- impl_type / verify_type / target_paths
3) artifacts 状態
- requirements_delta / doc_delta / design_delta / test_delta / plan_delta の status
4) remote 情報（git.auto=true かつ remote が設定済みの場合）
- parent: issue_id / url
- sub_issues 一覧: type / issue_id / scope
5) impl_progress（issues-phase の場合）
- doc の状態
- 分割なし: test_impl / code / test_exec の状態
- 分割あり: scope ごとの test_impl / code / test_exec の状態
  - 各 scope に `platform` 属性がある場合は表示する
  - `waiting_user` 状態の scope がある場合は **太字で強調** し、ユーザーの手動操作が必要な旨を明記:
    - code が `waiting_user`: 「操作指示書に従い Bubble Editor で実装後、`完了` と報告してください」
    - test_exec が `waiting_user`: 「証跡テンプレートに従い手動検証を実施し、結果を報告してください」
  - `deploy_gate` エントリがある場合は独立した行で表示:
    - `not_started`: 「先行 scope 完了後に実行」
    - `waiting_user`: **太字で強調** し、「クラウドへの反映を実施してください（`scripts/deploy.sh` 等）」
    - `done`: 「✅ クラウド反映完了」
6) 進行条件（未達があれば明示）
- spec-phase → commit の必須条件
- issues-phase（実装）では **RED→GREEN→REFACTOR 必須**（ACに対応する **検証（自動テスト/手動検証）無し変更** は差戻し）
7) 次のアクション
- 依存未達の解消手順（コマンド名と入力）
- 通常フロー例：
  - spec作成中：`/ds-spec-requirements|doc|design|test|plan <Issue名>`
  - 仕様承認後：`/ds-spec-commit <Issue名>`
  - 実装：`/ds-issue-impl <Issue名>`
