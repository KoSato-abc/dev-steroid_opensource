# ds-spec-plan
## 目的
Issue単位の作業プランと Issue 戦略（plan_delta）を作成し、AIレビュー→ユーザー承認まで遷移させる。
## 入力
`<Issue名>` — $ARGUMENTS
## 依存/前提チェック（最初に必ず実施）
- `issues/<Issue名>/state.issue.json` が存在
- phase が `spec-phase`
- `issues/<Issue名>/test_delta.md` が user 承認済み（artifacts.test_delta.status == approved）
## 参照（Read）
- `issues/<Issue名>/requirements_delta.md`
- `issues/<Issue名>/doc_delta.md`
- `issues/<Issue名>/design_delta.md`
- `issues/<Issue名>/test_delta.md`
- `issues/<Issue名>/state.issue.json`
## 更新（Write）
- `issues/<Issue名>/plan_delta.md`
- `issues/<Issue名>/state.issue.json`（artifacts.plan_delta, local_status）
## 書き込み制約
- spec-phase は `issues/<Issue名>/**` 以外に書き込まない（例外：product state の該当 issue 反映が必要な場合のみ）。
## 非交渉ルール
- 仕様追加禁止。未決や不確定は plan_delta に TODO として明示する。

## Issue 戦略 YAML スキーマ
plan_delta.md 内に以下の YAML ブロックで Issue 構造を定義する。ds-spec-commit はこの構造を**そのまま**実行する。

**分割なし（標準）:**
```yaml
# --- issue_strategy ---
parent:
  title: "<Issue名>"
  source: requirements_delta.md
sub_issues:
  - type: doc
    title: "[doc] <Issue名>"
    source: doc_delta.md
    scope: all
  - type: design
    title: "[design] <Issue名>"
    source: design_delta.md
    scope: all
  - type: test
    title: "[test] <Issue名>"
    source: test_delta.md
    scope: all
split: false
# --- end ---
```

**分割あり:**
```yaml
# --- issue_strategy ---
parent:
  title: "<Issue名>"
  source: requirements_delta.md
sub_issues:
  - type: doc
    title: "[doc] <Issue名>"
    source: doc_delta.md
    scope: all
  - type: design
    title: "[design] <Issue名>-<scope>"
    source: design_delta.md
    scope: <scope名>
    sections: ["<見出し番号 or ID>", ...]
  - type: test
    title: "[test] <Issue名>-<scope>"
    source: test_delta.md
    scope: <scope名>
    sections: ["<TC-ID>", ...]
split: true
split_reason: "<分割理由>"
# --- end ---
```

**スキーマルール:**
- `parent` は常に 1 つ。`source` は `requirements_delta.md` 固定
- `type: doc` の sub-issue は原則 1 つ（`scope: all`）
- `type: design` と `type: test` は対で分割する（同じ scope 値を持つ）
- `sections` は分割時に delta 内のどの部分を抽出するかの参照（見出し番号 or ID）
- `split: false` の場合、`sections` は不要
- ds-spec-commit はこの YAML を解析し、`source` の delta を `sections` で抽出して各 Issue 本文とする

## state 更新方針
- 開始時：`local_status` は `spec_in_progress` を維持
- runner 実行後：`artifacts.plan_delta.status = draft`
- reviewer approved：`artifacts.plan_delta.status = reviewed`（AIレビュー完了）
- ユーザー承認後：`artifacts.plan_delta.status = approved`
- **updated_at**：`state.issue.json` の status 変更のたびに ISO8601（UTC）で更新する
- **product state 同期**：`local_status` を変更した場合は、`.dev-steroid/state.product.json` の `issues[]` 該当エントリにも同じ値を反映する

## 実行手順
1) state を読み、依存未達なら blocked で停止。
2) runner（subagent）を起動：`.claude/agents/dev-steroid/spec-phase/ds-spec-plan/ds-spec-plan-runner.md`
   - runner への指示：requirements_delta パス、doc_delta パス、design_delta パス、test_delta パス、state.issue.json パスを伝える
3) runner 完了後、`artifacts.plan_delta.status` を `draft` に更新
4) reviewer（subagent）を起動：`.claude/agents/dev-steroid/spec-phase/ds-spec-plan/ds-spec-plan-reviewer.md`
   - reviewer への指示：全 delta パス、成果物パス（plan_delta）を伝える
5) reviewer の verdict に応じて：
   - `needs_revision`：runner に指摘を渡して再生成（最大3回）
   - `approved`：status を `reviewed` に更新し、ユーザー承認待ちの出力を提示
6) ユーザーが `承認` → status を `approved` に更新

## 出力（必須フォーマット）
1) 実行結果（更新/生成ファイル）
- 変更した/生成したファイルパスを列挙
2) state 要点（該当キー）
- phase / local_status / artifacts.plan_delta.status を要約
3) 次のアクション
- 承認待ちの場合：作業要約 / 検証状況 / 承認事項 / ユーザー操作（承認/差し戻し/保留）
- 承認済みなら：`/ds-spec-commit <Issue名>`
- 承認待ちなら：**承認後に実行する** `/ds-spec-commit <Issue名>`
