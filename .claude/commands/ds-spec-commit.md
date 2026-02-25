# ds-spec-commit
## 目的
spec-phase の成果物（requirements/doc/design/test/plan）が全て user 承認済みであることを前提に、plan_delta の Issue 戦略 YAML に従って GitHub Issues を登録し（git.auto=true 時）、phase を issues-phase に進める。
## 入力
`<Issue名>` — $ARGUMENTS
## 依存/前提チェック（最初に必ず実施）
- `issues/<Issue名>/state.issue.json` が存在
- artifacts.requirements_delta / doc_delta / design_delta / test_delta / plan_delta が全て `approved`
- `.dev-steroid/config.json` の `git` セクションを読み取り、`git.auto` の値を確認
- **git.auto=true の場合のみ**:
  - GitHub CLI（`gh`）が利用可能で、`gh auth status` が成功する（未達なら blocked）
## 参照（Read）
- `issues/<Issue名>/requirements_delta.md` `doc_delta.md` `design_delta.md` `test_delta.md` `plan_delta.md`
- `issues/<Issue名>/state.issue.json`
- `.dev-steroid/state.product.json`
- `.dev-steroid/config.json`（git セクション）
## 更新（Write）
- `issues/<Issue名>/state.issue.json`（remote, phase, local_status, impl_progress）
- `.dev-steroid/state.product.json`（該当 issue の remote / local_status / phase 反映）
## 注意
- remote Issue の provider は **GitHub 固定**。
- Issue 登録は**段階的に state を保存**し、途中失敗からの再実行で二重作成を防ぐ（冪等性保証）。

## Issue 戦略 YAML の解析
1) `plan_delta.md` から `# --- issue_strategy ---` と `# --- end ---` で囲まれた YAML ブロックを抽出する。
2) YAML を解析し、以下の構造を取得する:
   - `parent`: 親 Issue 情報（title, source）
   - `sub_issues`: sub-issue 配列（type, title, source, scope, sections）
   - `split`: 分割フラグ
3) 全 delta が approved であることを再確認する。

## git.auto=true の場合: Issue 登録手順

### 0) 冪等性チェック（再実行対応）
再実行時に Issue が二重作成されることを防ぐため、各手順の前に既存データを確認する:
- `state.issue.json` の `remote.parent` が既に存在する場合 → 手順1をスキップ
- `remote.sub_issues[]` に該当 type/scope の sub-issue が既に存在する場合 → その sub-issue の作成をスキップ
- `remote.sub_issues[]` の該当エントリに `linked: true` がある場合 → その紐付けをスキップ

### 1) 親 Issue 作成
- **スキップ条件**: `remote.parent` が既に存在する場合はスキップ
- `parent.source`（requirements_delta.md）の全文を本文に使用
- `gh issue create --title "<parent.title>" --body-file -` で作成
- URL から `issue_id` を抽出
- **即時保存**: `state.issue.json` の `remote` を以下で更新:
  ```json
  { "provider": "github", "parent": { "url": "...issues/100", "issue_id": 100 }, "sub_issues": [] }
  ```

### 2) 各 sub-issue 作成
- 各 sub-issue について:
  - **スキップ条件**: `remote.sub_issues[]` に同一 type/scope のエントリが既に存在する場合はスキップ
  - `source` の delta から本文を組み立てる
  - `split: true` かつ `sections` がある場合: delta 内の該当 sections を抽出して本文とする
  - `split: false` または `scope: all` の場合: delta 全文を本文とする
  - `gh issue create --title "<sub_issue.title>" --body-file -` で作成
  - URL から `issue_id` を抽出
  - **即時保存**: `state.issue.json` の `remote.sub_issues[]` に追記:
    ```json
    { "type": "design", "issue_id": 102, "url": "...", "scope": "...", "linked": false }
    ```

### 3) 親子紐付け（GraphQL）
- 各 sub-issue について:
  - **スキップ条件**: `remote.sub_issues[]` の該当エントリに `linked: true` がある場合はスキップ
  - GitHub GraphQL API の `addSubIssue` mutation で親 Issue に紐付ける
  - **即時保存**: 紐付け成功後、該当エントリの `linked` を `true` に更新

### シェルエスケープ（必須）
delta 本文に `$`, `` ` ``, `\`, `"` 等のシェル特殊文字が含まれるため、**`--body-file` パターンを第一選択**とする:
```bash
# 推奨: 一時ファイル経由
tmp_body=$(mktemp "${TMPDIR:-/tmp}/ds-body-XXXXXX")
cat > "$tmp_body" <<'BODY_EOF'
（delta 本文をそのまま展開）
BODY_EOF
gh issue create --title "$title" --body-file "$tmp_body"
rm -f "$tmp_body"
```
- ヒアドキュメントの区切り文字は **クォート付き**（`<<'BODY_EOF'`）を使い、シェル展開を完全に抑止する
- 一時ファイルは `.gitignore` 済みパスまたは `/tmp` に生成し、使用後即削除する
- GraphQL の mutation body にも同様に `--input` または一時ファイル経由で渡す

### 4) state 最終確認
- 手順1〜3で段階的に保存した `remote` が完全であることを確認する:
  - `remote.parent` が存在する
  - `remote.sub_issues` の件数が Issue 戦略 YAML の sub_issues 件数と一致する
  - 全 sub_issues の `linked` が `true` である
- 不整合がある場合は、不足分のみを再実行する（冪等性チェックにより安全）
- 確認後、`remote.sub_issues[]` の各エントリから `linked` フィールドを削除する（最終形に整形）
- 最終的な `remote` の構造:
  ```json
  {
    "provider": "github",
    "parent": { "url": "...issues/100", "issue_id": 100 },
    "sub_issues": [
      { "type": "doc",    "issue_id": 101, "url": "...", "scope": "all" },
      { "type": "design", "issue_id": 102, "url": "...", "scope": "..." },
      { "type": "test",   "issue_id": 103, "url": "...", "scope": "..." }
    ]
  }
  ```

### 5) impl_progress の拡張（split: true の場合）
- Issue 戦略の scope 一覧から `impl_progress.scopes` 構造を生成:
  ```json
  {
    "impl_progress": {
      "doc": "not_started",
      "scopes": {
        "<scope名>": { "test_impl": "not_started", "code": "not_started", "test_exec": "not_started" }
      }
    }
  }
  ```
- **[hybrid の場合]** `scope_order` に `deploy_gate` が含まれる場合、`impl_progress.scopes` に deploy_gate エントリを追加する:
  ```json
  {
    "impl_progress": {
      "doc": "not_started",
      "scopes": {
        "db-and-api": { "platform": "supabase", "test_impl": "not_started", "code": "not_started", "test_exec": "not_started" },
        "deploy_gate": { "status": "not_started", "depends_on": ["db-and-api"] },
        "bubble-ui": { "platform": "bubble", "depends_on": ["deploy_gate"], "test_impl": "not_started", "code": "not_started", "test_exec": "not_started" }
      }
    }
  }
  ```
- 各 scope の `platform` 属性は Issue 戦略 YAML の sub_issues から取得する
- `depends_on` は `scope_order` の順序から導出する（各 scope は直前の scope に依存）
- `split: false` の場合はフラット構造を維持（変更不要）

## git.auto=false の場合
1) 「Issue 登録スキップ（git.auto=false）」を表示
2) `remote` は `null` のまま
3) `impl_progress` の拡張は git.auto=true と同様に行う（split: true の場合は scopes 構造に拡張）

## 共通の state 更新ルール
- `state.issue.json`: phase=`issues-phase`, local_status=`committed`
- `state.issue.json` の `updated_at` を ISO8601（UTC）で更新する
- artifacts.*.status は維持（approved のまま）
- `state.product.json` の該当 issue に phase / local_status / remote を反映
- `state.product.json` の `current_phase` が `spec-phase` の場合は `issues-phase` に更新する（既に `issues-phase` なら維持）

## エラー時
- remote 作成の途中で失敗した場合:
  - **それまでに成功した分の state は保持する**（段階的に保存済みのため）。
  - 失敗理由（APIエラー/認証/権限/ネットワーク等）と、再実行の手順を提示して停止。
  - 再実行時は冪等性チェック（手順0）により、成功済みの手順を自動スキップする。

## 出力（必須フォーマット）
1) 実行結果（更新/生成ファイル）
- 更新したファイルパスを列挙
- git.auto=true の場合: 作成した Issue URL 一覧（親 + sub-issue）
2) state 要点（該当キー）
- phase / local_status / remote（provider / parent / sub_issues 概要）を要約
- impl_progress の構造（フラット or scopes）を要約
3) 次のアクション
- 次に実行すべきコマンド（通常：`/ds-issue-impl <Issue名>`）
