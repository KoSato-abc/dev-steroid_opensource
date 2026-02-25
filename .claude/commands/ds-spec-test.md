# ds-spec-test
## 目的
Issue単位のテスト情報（test_delta）を作成し、AIレビュー→ユーザー承認まで遷移させる。
## 入力
`<Issue名>` — $ARGUMENTS
## 依存/前提チェック（最初に必ず実施）
- `issues/<Issue名>/state.issue.json` が存在
- phase が `spec-phase`
- `issues/<Issue名>/design_delta.md` が user 承認済み（artifacts.design_delta.status == approved）
## 参照（Read）
- `.dev-steroid/docs/**`（承認済みPRD）
- `issues/<Issue名>/requirements_delta.md`
- `issues/<Issue名>/design_delta.md`
- `issues/<Issue名>/state.issue.json`
## 更新（Write）
- `issues/<Issue名>/test_delta.md`
- `issues/<Issue名>/state.issue.json`（artifacts.test_delta, local_status）
## 書き込み制約
- spec-phase は `issues/<Issue名>/**` 以外に書き込まない（例外：product state の該当 issue 反映が必要な場合のみ）。
## 非交渉ルール
- issues-phase の実装は **RED→GREEN→REFACTOR 必須**。`test_delta.md` には AC→検証（自動テスト/手動手順）の対応表を必ず含める。
- 仕様追加禁止。未決や不確定は test_delta に TODO として明示する。

## state 更新方針
- 開始時：`local_status` は `spec_in_progress` を維持
- runner 実行後：`artifacts.test_delta.status = draft`
- reviewer approved：`artifacts.test_delta.status = reviewed`（AIレビュー完了）
- ユーザー承認後：`artifacts.test_delta.status = approved`
- **updated_at**：`state.issue.json` の status 変更のたびに ISO8601（UTC）で更新する
- **product state 同期**：`local_status` を変更した場合は、`.dev-steroid/state.product.json` の `issues[]` 該当エントリにも同じ値を反映する

## 実行手順
1) state を読み、依存未達なら blocked で停止（不足一覧と次アクションを提示）。
2) runner（subagent）を起動：`.claude/agents/dev-steroid/spec-phase/ds-spec-test/ds-spec-test-runner.md`
   - runner への指示：PRDパス、requirements_delta パス、design_delta パス、state.issue.json パスを伝える
3) runner 完了後、`artifacts.test_delta.status` を `draft` に更新
4) reviewer（subagent）を起動：`.claude/agents/dev-steroid/spec-phase/ds-spec-test/ds-spec-test-reviewer.md`
   - reviewer への指示：PRDパス、requirements_delta パス、design_delta パス、成果物パスを伝える
5) reviewer の verdict に応じて：
   - `needs_revision`：runner に指摘を渡して再生成（最大3回）
   - `approved`：status を `reviewed` に更新し、ユーザー承認待ちの出力を提示
6) ユーザーが `承認` → status を `approved` に更新

## 出力（必須フォーマット）
1) 実行結果（更新/生成ファイル）
- 変更した/生成したファイルパスを列挙
2) state 要点（該当キー）
- phase / local_status / artifacts.test_delta.status を要約
3) 次のアクション
- 承認待ちの場合：作業要約 / 検証状況 / 承認事項 / ユーザー操作（承認/差し戻し/保留）
- 承認済みなら：`/ds-spec-plan <Issue名>`
- 承認待ちなら：**承認後に実行する** `/ds-spec-plan <Issue名>`
