# ds-doc-refactoring
## 目的
全 approved 設計書を横断的に再レビューし、テンプレート準拠性・設計書間整合性・
分割粒度の妥当性・記述不足を検出・修正する。
修正後はユーザーの明示的承認によりコミットする。

## 入力
なし（$ARGUMENTS は無視）

## 依存/前提チェック（最初に必ず実施）
- `.dev-steroid/state.product.json` が存在
- `docs_phase` に `approved` が 1 つ以上

## 参照（Read）
- `.dev-steroid/docs/**`（全 approved 設計書）
- `.dev-steroid/template/**`（全テンプレート）
- `.dev-steroid/config.json`
- `.dev-steroid/state.product.json`

## 更新（Write）
- `.dev-steroid/docs/**`（修正対象の設計書のみ）
- `.dev-steroid/state.product.json`（`updated_at` のみ。status は `approved` 維持）

## 書き込み制約
- docs-phase は `.dev-steroid/docs/**` と `.dev-steroid/state.product.json` 以外に書き込まない。
- 設計書の status は `approved` を維持する（refactoring は承認済み設計書の品質向上であり、status 遷移ではない）。

## 実行手順

### 1) 前提チェック
- `state.product.json` を読み取り、`docs_phase` の各キーの status を確認する。
- `approved` が 1 つもなければ停止（「先に `/ds-doc` で設計書を作成・承認してください」と案内）。
- approved な doc_type の一覧を取得し、config.json の `docs_phase_map` から依存順（order）でソートする。

### 2) analyzer 起動
subagent を起動：`.claude/agents/dev-steroid/docs-phase/ds-doc-refactoring/ds-doc-refactoring-analyzer.md`

analyzer への指示：
- `approved_docs`：approved な doc_type の一覧と、各 doc_type の出力先ディレクトリパス
- `template_root`：`.dev-steroid/template/`
- `config_path`：`.dev-steroid/config.json`

analyzer は横断分析レポート（`.dev-steroid/docs/_refactoring_report.md`）を生成する。

### 3) 指摘件数による分岐
- must 0 件 かつ should 0 件 → 「修正不要。全設計書は良好な状態です」で終了。レポートを削除。
- must 0 件 かつ should のみ → should 一覧を提示し、修正するか確認。
  - ユーザーが修正を指示 → 手順 4 に進む
  - ユーザーがスキップ → レポートを削除して終了
- must 1 件以上 → 修正に進む（手順 4）

### 4) fixer 起動（doc_type の依存順に逐次）
修正対象の doc_type を config.json の `docs_phase_map` の `order` 順にソートし、1 doc_type ずつ処理する：

a) fixer subagent を起動：`.claude/agents/dev-steroid/docs-phase/ds-doc-refactoring/ds-doc-refactoring-fixer.md`
   - fixer への指示：
     - `report_path`：`.dev-steroid/docs/_refactoring_report.md`
     - `target_doc_type`：修正対象の doc_type
     - `target_doc_path`：対象 doc_type の出力先ディレクトリパス
     - `template_path`：対象 doc_type のテンプレートディレクトリパス
     - `all_docs_root`：`.dev-steroid/docs/`（他の設計書との整合をとるための参照）

b) fixer 完了後、既存の ds-doc-reviewer で再レビュー：
   - `.claude/agents/dev-steroid/docs-phase/ds-doc/ds-doc-reviewer.md`
   - reviewer への指示：テンプレパス、成果物パスを明示的に伝える

c) reviewer の verdict に応じて：
   - `needs_revision`：fixer に指摘を渡して再修正（最大 3 回）
   - `approved`：次の doc_type へ

d) 最大 3 回の修正ループを超えた場合：停止してユーザーに相談（判断材料を提示）

### 5) 全 doc_type の修正完了後
- 分析レポート（`_refactoring_report.md`）を削除する
- 修正 diff サマリを作成しユーザーに提示する：
  - doc_type ごとの変更概要（何を、どう修正したか）
  - 分析で検出した指摘件数と対応状況

### 6) ユーザー承認待ち
- **承認**：
  - 修正を行った doc_type の `updated_at` を更新（status は `approved` 維持。修正していない doc_type は更新しない）
  - `state.product.json` の `last_updated` を更新
  - git.auto=true の場合：
    ```
    git add .dev-steroid/docs/ .dev-steroid/state.product.json
    git commit -m "refactor(docs): 設計書横断リファクタリング"
    ```
- **差し戻し: <指示>**：
  - ユーザーの指示に基づき、指定の doc_type について fixer を再起動（手順 4 に戻る）
- **却下**：
  - git.auto=true の場合：`git checkout -- .dev-steroid/docs/ .dev-steroid/state.product.json`
  - git.auto=false の場合：手動での復元手順を案内
  - レポートを削除して終了

## git 操作（git.auto=true の場合）
ユーザー承認後に一括コミット。`cd` は使わず相対パスで実行。

## 出力（必須フォーマット）
1) 分析結果サマリ
- 指摘件数: must / should × カテゴリ別
- カテゴリ: A.テンプレ準拠 / B.ID整合 / C.用語統一 / D.記述不足 / E.リンク整合 / F.分割粒度
2) 修正内容サマリ（修正した場合）
- doc_type ごとの変更概要
3) ユーザー操作
- 承認 / 差し戻し: <指示> / 却下
4) 次のアクション
- 承認後：通常のワークフローに復帰（`/ds-spec-init` 等）
- 差し戻し後：再修正 → 再提示
