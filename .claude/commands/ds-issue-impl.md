# ds-issue-impl
## 目的
sub-issue の仕様を根拠に、doc → scope 別ループ[test_impl → code → test_exec] で実装を進め、全工程完了後に PR を作成する。
## 非交渉ルール
- **RED→GREEN→REFACTOR 必須**：②test_impl でテストを書き（RED）、③code で最小実装（GREEN）→ 改善（REFACTOR）、④test_exec で実行確認。
  - ACに対応する **検証（自動テスト or 手動検証）** 無し変更は不合格（needs_revision）。
  - 検証形態（automated/manual/hybrid）は `state.issue.json` の `meta.verify_type` で宣言。
- **仕様追加禁止**：sub-issue / delta に無い機能追加は行わない。曖昧・矛盾は推測で埋めず blocked として止める。
## 入力
`<Issue名>` — $ARGUMENTS（常に親 Issue 名で呼ぶ）
## 依存/前提チェック（最初に必ず実施）
- `issues/<Issue名>/state.issue.json` が存在
- `local_status == committed`
- `.dev-steroid/config.json` が存在（`git` セクションを読み取る）
- `state.issue.json` の `meta`（impl_type / verify_type / target_paths）が確定していること
- **git.auto=true の場合**: `remote.parent` が存在すること
- **git.auto=false の場合**: `remote` が null であること（ローカル delta を直接参照）
## 参照（Read）
- `issues/<Issue名>/**`（delta, state, remote情報）
- `.dev-steroid/config.json`
- `.dev-steroid/docs/**`（納品物PRD。doc工程で参照・更新対象）
- `meta.target_paths` の対象範囲
## 更新（Write）
- `.dev-steroid/docs/**`（doc 工程：納品物PRDへの Issue 内容反映）
- `meta.target_paths.test_paths` に一致するファイル（test_impl 工程：テスト実装）
- `meta.target_paths.code_paths` に一致するファイル（code 工程：必要最小の実装）
- `issues/<Issue名>/state.issue.json`（local_status, impl_progress）
- `.dev-steroid/state.product.json`（該当 issue の local_status 反映のみ）
## 書き込み制約
- **更新禁止**：`.dev-steroid/config.json`, `issues/<Issue名>/*.md`（delta）
- 仕様の穴/矛盾/未決が原因で前進できない場合は、**blocked** として停止し、修正すべき根拠（sub-issue 本文 or delta）と再実行手順（spec-phase の該当コマンド）を提示する。

## 仕様の参照方法（条件分岐）
各工程で sub-issue の仕様を参照する際:
```
git.auto=true:
  Bash: gh issue view <sub_issue_id> --json body -q .body
  → sub_issue_id は state.issue.json の remote.sub_issues から取得
  → type と scope で対象の sub-issue を特定

git.auto=false:
  ローカル: issues/<Issue名>/<delta>.md を直接 Read
  → 分割時は delta 内の該当 sections を参照
  → sections は state.issue.json に保存されていないため、plan_delta の issue_strategy YAML から取得
```

## state 更新方針
- 開始時：`local_status` を `implementing` に更新（既に `implementing` なら維持）
- 各工程完了時：`impl_progress` の該当キーを `done` に更新（開始時は `in_progress`）
- 全工程が done で完了：`local_status` を `done` に更新
- blocked 発生時：`local_status` を `blocked` に更新（原因解消後に `implementing` に復帰）
- **updated_at**：`state.issue.json` の `updated_at` を status/impl_progress 変更のたびに ISO8601（UTC）で更新する
- **product state 同期**：`state.issue.json` の `local_status` を更新するたびに、`.dev-steroid/state.product.json` の `issues[]` 該当エントリの `local_status` にも同じ値を反映する

## 実行順（固定）・各工程の共通ルール
- 順序：①doc → scope 別ループ[ ②test_impl → ③code → ④test_exec ] → PR
- 各工程：runner → reviewer。`needs_revision` は最大3回まで差し戻し。
- reviewer の must 指摘が 0 になるまで次工程へ進まない。
- 修正ループ最大3回。超えたら停止してユーザーへ相談（判断材料を提示）。
- **③→④ループ**：④test_exec で失敗した場合は③code に戻って修正 → ④再実行。このループも最大3回。

## 実行手順

### 0) 事前準備
1) state を読み、依存未達なら blocked で停止（理由と次のアクションを提示）。
2) `state.issue.json` の `meta` 確認。欠けていれば blocked。
3) `.dev-steroid/config.json` の `git.auto` を確認。
4) **[hybrid/no-code の場合]** `config.json` の `platforms` 定義を読み取る。
5) **[hybrid の場合]** plan_delta の issue_strategy YAML から `scope_order` を読む。`scope_order` に `deploy_gate` が含まれる場合、`impl_progress.scopes.deploy_gate` が初期化済みであることを確認する。
6) **再開判定**（`local_status` が `implementing` の場合）:
   - `impl_progress` を読み、各工程の状態を判定する:
     - `done` → スキップ（再実行しない）
     - `in_progress` → 中断とみなし、その工程を最初から再実行する
     - `waiting_user` → ユーザーの手動操作完了報告を待つ状態。ユーザーに「前回の手動操作は完了しましたか？」と確認し、完了報告があれば `done` に進める。未完了なら再度待機。
     - `not_started` → 通常通り実行
   - 「工程 ①doc: done → スキップ / ②test_impl [scope-a]: in_progress → 再開」のように再開計画をユーザーに提示し、確認後に進む。
   - `done` の工程を再実行したい場合はユーザーが明示的に指示する必要がある。
7) **[git.auto=true]** feature branch を作成（または既存ブランチに切り替え）:
   ```
   # 新規の場合
   git checkout <base_branch>
   git pull
   git checkout -b feature/<parent_issue_id>-<Issue名>
   # 再開の場合（ブランチが既に存在すれば切り替えのみ）
   git checkout feature/<parent_issue_id>-<Issue名>
   ```
8) `state.issue.json` から sub_issues 構造と scope 一覧を取得:
   - `remote.sub_issues`（git.auto=true）または plan_delta の issue_strategy YAML（git.auto=false）から取得

### 1) ①doc 工程
- `impl_progress.doc` を `in_progress` → 完了後 `done`
- 目的：sub-issue の仕様を **納品物PRD（`.dev-steroid/docs/**`）** に反映する。
- 仕様参照: `type: doc`（scope: all）の sub-issue
- runner（subagent）：`.claude/agents/dev-steroid/issues-phase/ds-issue-impl/ds-issue-impl-doc-runner.md`
  - 追加パラメータ: `git_auto`, `scope: all`
- reviewer（subagent）：`.claude/agents/dev-steroid/issues-phase/ds-issue-impl/ds-issue-impl-doc-reviewer.md`
- blocked が出たら停止。
- **[git.auto=true]** 完了後 commit:
  ```
  feat(<Issue名>): doc - <要約>
  ```

### 2) scope 別ループ
`scope_order`（plan_delta で定義。未定義時は sub_issues の順序）に従い、各 scope について以下を判定・実行する。分割なしの場合は scope: all のみ。

#### scope の判定
1. scope 名が `deploy_gate` → **デプロイゲート処理（C）**（platform.type 判定をスキップ）
2. `impl_progress.scopes.<scope>` から `platform` を取得
3. `config.json` の `platforms` から `type` を取得
4. `type` に応じて工程を切替:
   - `type=code` → **既存パス（A）**
   - `type=no-code` → **no-code パス（B）**
5. `platforms` 未定義、または `platform` 属性なしの scope → **既存パス（A）**

#### (A) type=code の場合（既存パスと同一）

##### ②test_impl 工程
- `impl_progress.test_impl`（フラット）または `impl_progress.scopes.<scope>.test_impl`（分割時）を `in_progress` → 完了後 `done`
- 目的：ACに基づくテストを **先に実装**する（RED）。変更前に失敗することを確認。
- 仕様参照: `type: test`（当該 scope）の sub-issue
- runner（subagent）：`.claude/agents/dev-steroid/issues-phase/ds-issue-impl/ds-issue-impl-test-runner.md`
  - 起動モード：`test_impl`
  - 追加パラメータ: `git_auto`, `scope`
- reviewer（subagent）：`.claude/agents/dev-steroid/issues-phase/ds-issue-impl/ds-issue-impl-test-reviewer.md`
  - 起動モード：`test_impl`
- `manual` の場合は手動検証手順の策定のみ（テストコード不要）。
- **[git.auto=true]** 完了後 commit:
  ```
  test(<Issue名>): test_impl [<scope>] - <要約>
  ```

##### ③code 工程
- `impl_progress.code`（フラット）または `impl_progress.scopes.<scope>.code`（分割時）を `in_progress` → 完了後 `done`
- 目的：②で書いたテストを通す **最小実装**（GREEN）→ 改善（REFACTOR）。
- 仕様参照: `type: design`（当該 scope）の sub-issue
- runner（subagent）：`.claude/agents/dev-steroid/issues-phase/ds-issue-impl/ds-issue-impl-code-runner.md`
  - 追加パラメータ: `git_auto`, `scope`
- reviewer（subagent）：`.claude/agents/dev-steroid/issues-phase/ds-issue-impl/ds-issue-impl-code-reviewer.md`
- **検証（自動テスト/手動検証）無し変更は即差し戻し**。
- **[git.auto=true]** 完了後 commit:
  ```
  feat(<Issue名>): code [<scope>] - <要約>
  ```

##### ④test_exec 工程
- `impl_progress.test_exec`（フラット）または `impl_progress.scopes.<scope>.test_exec`（分割時）を `in_progress` → 完了後 `done`
- 目的：全テスト実行 + 回帰確認。
- runner（subagent）：`.claude/agents/dev-steroid/issues-phase/ds-issue-impl/ds-issue-impl-test-runner.md`
  - 起動モード：`test_exec`
  - 追加パラメータ: `git_auto`, `scope`
- reviewer（subagent）：`.claude/agents/dev-steroid/issues-phase/ds-issue-impl/ds-issue-impl-test-reviewer.md`
  - 起動モード：`test_exec`
- **失敗時**：③code に戻って修正 → ④再実行（最大3回）。
- **[git.auto=true]** 完了後 commit:
  ```
  test(<Issue名>): test_exec [<scope>] - <要約>
  ```

#### (B) type=no-code の場合

##### ②test_impl 工程（手動検証手順の確定）
- `impl_progress.scopes.<scope>.test_impl` を `in_progress` → 完了後 `done`
- 目的：test_delta の手動検証シナリオを **実行可能な手順書** として確定する。
- 仕様参照: `type: test`（当該 scope）の sub-issue
- runner（subagent）：`.claude/agents/dev-steroid/issues-phase/ds-issue-impl/ds-issue-impl-test-runner.md`（test_impl モード、manual）
  - test_delta の手動検証シナリオを整形
  - **テスト実施条件（Supabase クラウド反映済み等）を手順書に明記**
- reviewer（subagent）：`.claude/agents/dev-steroid/issues-phase/ds-issue-impl/ds-issue-impl-test-reviewer.md`（test_impl モード、manual）
  - AC 網羅チェック
- **[git.auto=true]** 完了後 commit:
  ```
  test(<Issue名>): test_impl [<scope>] - 手動検証手順書
  ```

##### ③code 工程（操作指示書生成 → ユーザー手動実行）
- `impl_progress.scopes.<scope>.code` を `in_progress`
- 目的：design_delta の no-code セクション + sub-issue 仕様を元に **操作指示書** を生成し、ユーザーに手動実行を依頼する。
- 仕様参照: `type: design`（当該 scope）の sub-issue
- runner（subagent）：`.claude/agents/dev-steroid/issues-phase/ds-issue-impl/ds-issue-impl-guide-runner.md`
  - 成果物: `issues/<Issue>/guides/<scope>_operation_guide.md`
- reviewer（subagent）：`.claude/agents/dev-steroid/issues-phase/ds-issue-impl/ds-issue-impl-guide-reviewer.md`
  - 手順の実行可能性、AC 網羅性、依存順序を監査
- **[git.auto=true]** 操作指示書作成後 commit:
  ```
  feat(<Issue名>): code [<scope>] - 操作指示書
  ```
- `impl_progress.scopes.<scope>.code` を `waiting_user` に設定
- ユーザーに操作指示書を提示し、手動実行を依頼:
  ```
  操作指示書を作成しました: issues/<Issue>/guides/<scope>_operation_guide.md
  Bubble Editor で手順に従って実装を行い、完了後「完了」と報告してください。
  ```
- **待機**: ユーザーが「完了」と報告するまで停止
- 完了報告受領後: `impl_progress.scopes.<scope>.code` を `done` に更新

##### ④test_exec 工程（手動検証 → 証跡記録）
- `impl_progress.scopes.<scope>.test_exec` を `in_progress`
- 目的：手動検証を実施し、証跡を記録する。
- runner（subagent）：`.claude/agents/dev-steroid/issues-phase/ds-issue-impl/ds-issue-impl-test-runner.md`（test_exec モード、manual）
  - 証跡テンプレートを生成: `issues/<Issue>/evidence/<scope>_evidence.md`
  - 構成: 検証環境 / 前提確認チェックリスト / AC 検証結果テーブル / 回帰確認 / スクリーンショット欄
- ユーザーに検証実施を依頼:
  ```
  証跡テンプレートを作成しました: issues/<Issue>/evidence/<scope>_evidence.md
  Bubble プレビューで手動検証を実施し、結果を報告してください。
  ```
- **待機**: ユーザーが証跡を報告するまで停止
- reviewer（subagent）：`.claude/agents/dev-steroid/issues-phase/ds-issue-impl/ds-issue-impl-test-reviewer.md`（test_exec モード、manual）
  - 証跡と AC の突合、網羅性判定
  - 合格 → `done` / 不合格 → ③code に戻る（**最大3回。超過時はユーザーに相談**）
- **[git.auto=true]** 完了後 commit:
  ```
  test(<Issue名>): test_exec [<scope>] - 手動検証完了
  ```

#### (C) deploy_gate の処理

scope_order に `deploy_gate` が含まれ、かつ先行 scope が全て `done` の場合に実行する:

1. `config.json` の `platforms` から `deploy_commands` を読む
2. クラウド反映手順をユーザーに案内:
   ```
   Supabase 側のローカルテスト完了。クラウドへの反映を実施してください:

   1. scripts/deploy.sh を実行（または手動で以下を実行）
      - supabase db push --linked（テーブル + RLS 反映）
      - supabase functions deploy（Edge Functions デプロイ）
   2. Bubble の API Connector URL がクラウド Supabase を指していることを確認

   完了後「デプロイ完了」と報告してください。
   ```
3. `impl_progress.scopes.deploy_gate.status` を `waiting_user` に設定
4. **待機**: ユーザーが「デプロイ完了」と報告するまで停止
5. 完了報告受領後: `deploy_gate.status` を `done` に更新
6. 次の scope（no-code 側）に進む

### 3) 完了処理
1) 全工程完了後、`local_status` を `done` に更新（product state にも同期）。
2) **[git.auto=true]** PR を作成（HEREDOC で本文を渡す）:
   ```
   gh pr create --title "<Issue名>" --body "$(cat <<'EOF'
   ## 概要
   Closes #<parent_issue_id>

   <Issue名> の実装。

   ## 作業内容
   - ①doc: <PRD 更新の要点>
   - ②test_impl: <テスト追加の要点>
   - ③code: <実装の要点>
   - ④test_exec: <テスト結果の要点>

   ## AC 検証結果
   | AC-ID | 検証方法 | 結果 |
   |---|---|---|
   | AC-001 | 自動テスト | ✅ pass |

   ## 確認事項
   - <レビュアーに確認してほしい点>

   ## 参照
   - 親 Issue: #<parent_id>
   - Sub-issues: #<doc_id>, #<design_id>, #<test_id>
   EOF
   )"
   ```

## 出力（必須フォーマット）
1) 実行結果（更新/生成ファイル）
- 変更した/生成したファイルパスを列挙
2) 検証要点（RED/GREEN/REFACTOR 要約）
3) state 要点（該当キー）
- 関連する status / phase / local_status / impl_progress / remote を要約
4) 次のアクション
- 次に実行すべきコマンド（通常：`/ds-issue-status <Issue名>`）
