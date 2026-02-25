# dev-steroid — はじめに

Claude Code でウォーターフォール開発を回すためのプラグインキットです。
スラッシュコマンドで PRD 作成 → Issue 仕様化 → 実装まで、state 管理付きで段階的に進められます。

## 前提

- **Claude Code** 1.0.33 以上
- Git リポジトリのルートにキットを配置していること
- **git.auto=true の場合のみ**: GitHub CLI (`gh`) が認証済み、Sub-issues 機能が利用可能
- **Supabase を使う場合**: Supabase CLI + Docker Desktop がインストール済み
- **Bubble を使う場合**: Bubble アカウントと対象アプリへのアクセス権

## セットアップ

```bash
# 1. キットをリポジトリルートに展開
#    .claude/  .dev-steroid/ が配置される
#    ※ CLAUDE.md は配置されない（既存の CLAUDE.md を上書きしない）
#    ※ dev-steroid のルールは .claude/rules/dev-steroid.md に格納
#      （Claude Code が自動で読み込む）

# 2. config.json の設定を確認
#    git.auto: true（GitHub連携あり） or false（ローカルのみ）
#    platforms: プロジェクトで使用するプラットフォーム（後述）

# 3. Claude Code を起動して初期化
/ds-init
```

`/ds-init` は環境チェック（git.auto=true の場合は gh 認証・プラグイン確認も実施）と `state.product.json` の初期化を行います。

### platforms 設定（No-Code / Hybrid プロジェクト対応）

`config.json` の `platforms` セクションで、プロジェクトが使用するプラットフォームを定義します。

```json
{
  "platforms": {
    "supabase": {
      "type": "code",
      "deploy_commands": {
        "db": "supabase db push --linked",
        "functions": "supabase functions deploy"
      }
    },
    "bubble": {
      "type": "no-code",
      "name": "Bubble",
      "artifact_format": "operation_guide"
    }
  }
}
```

- `type: "code"` — AI が自律的にファイル生成・テスト実行する（Supabase, 通常のコード等）
- `type: "no-code"` — AI が操作指示書を生成し、ユーザーが GUI で手動実装する（Bubble 等）
- `platforms` 未定義の場合は従来通り全て `code` として動作（後方互換）

`type` の組み合わせで `impl_type` が自動推定されます:

| platforms 構成 | impl_type | 動作 |
|---|---|---|
| code のみ / 未定義 | `code` | 全工程 AI 自律（従来動作） |
| no-code のみ | `no-code` | 全工程で操作指示書 + 手動実装 |
| code + no-code | `hybrid` | code scope は AI 自律、deploy_gate 後に no-code scope を手動実装 |

## ワークフロー概要

```
docs-phase          spec-phase                       issues-phase
──────────          ──────────                       ────────────
/ds-doc             /ds-spec-init                    /ds-issue-impl
  requirements        ↓                                ①doc（PRD更新）
  basic_design      /ds-spec-requirements              ↓
  detailed_design     → requirements_delta.md          scope別ループ:
  data_design         ↓                              [code scope]
  security_design   /ds-spec-doc                       ②test_impl（RED）
  test_design         → doc_delta.md                   ③code（GREEN→REFACTOR）
                      ↓                                ④test_exec（回帰）
/ds-env-setup       /ds-spec-design                  [deploy_gate]  ← hybrid時のみ
  Supabase CLI/DB     → design_delta.md                ユーザーがクラウド反映
  Bubble URL確認      ↓                              [no-code scope] ← hybrid/no-code時
                    /ds-spec-test                      ②test_impl（手動検証手順策定）
/ds-doc-refactoring   → test_delta.md                  ③code（操作指示書→ユーザー手動実装）
  （横断レビュー）    ↓                                ④test_exec（手動検証→証跡記録）
                    /ds-spec-plan                      ↓
                      → plan_delta.md                [git.auto=true]
                      (Issue戦略YAML含む)             各工程後 commit
                      ↓                              全完了後 PR 作成
                    /ds-spec-commit                    ↓
                      [git.auto=true]                /ds-issue-status
                      → GitHub Issues 自動作成
                        親Issue + sub-issues
                      [git.auto=false]
                      → state更新のみ（Issue登録・git操作なし）

                    --- or ---

                    /ds-issue-create-from-docs
                      Phase A: ギャップ分析→分割戦略承認
                      Phase B: delta一括生成→Issue単位承認
                      → /ds-spec-commit で登録
```

各コマンドは「AI 生成 → AI レビュー → ユーザー承認」のサイクルで品質を担保します。

## クイックスタート（最短パス）

```
/ds-init                          # 環境チェック + state 初期化
/ds-doc requirements              # 要件定義PRD 作成 → 承認
/ds-doc basic_design              # 基本設計PRD → 承認（不要なら「対象外」）
/ds-doc all                       # 残りの PRD を順次作成（不要分は対象外）
/ds-env-setup                     # 開発環境構築（platforms定義時。Supabase CLI/Bubble URL確認等）
/ds-doc-refactoring               # （任意）設計書を横断レビュー・修正
/ds-issue-create-from-docs        # （任意）設計書から Issue を一括起票
                                  #   → 分割戦略を承認 → delta 一括生成 → Issue 単位で承認
                                  #   → /ds-spec-commit で GitHub Issues 登録
                                  # ↑ 一括起票を使わない場合は、以下の手動パスで1 Issue ずつ作成 ↓
/ds-spec-init add-login           # Issue「add-login」の初期化 → メタ承認
/ds-spec-requirements add-login   # 要件 delta → 承認
/ds-spec-doc add-login            # PRD修正情報 delta → 承認
/ds-spec-design add-login         # 設計 delta → 承認
/ds-spec-test add-login           # テスト delta → 承認
/ds-spec-plan add-login           # 作業計画 + Issue戦略 → 承認
/ds-spec-commit add-login         # GitHub Issues 登録（or ローカル commit）
/ds-issue-impl add-login          # 実装（doc→scope別[test_impl→code→test_exec]→PR）
/ds-issue-status add-login        # 進捗確認
```

## 3つのフェーズ

### 1. docs-phase — PRD を作る

`/ds-doc <doc_type>` でテンプレートに沿った PRD を生成します。

- `requirements`（必須）→ `basic_design` → `detailed_design` → `data_design` → `security_design` → `test_design`
- 各ドキュメントは依存順に作成。不要なものは「対象外」と伝えれば `not_applicable` にできます（`requirements` を除く）。
- `/ds-doc all` で未完了分を一括処理。1 ドキュメントごとに承認を求めて停止します。
- **git.auto=true**: 承認後に自動 commit（`docs(<doc_type>): <要約>`）

#### 環境構築（/ds-env-setup）

`/ds-env-setup` は全設計書が approved になった後に実行し、`config.json` の `platforms` 定義に基づいて開発環境を構築します。

- **Supabase**: CLI 確認、Docker 確認、`supabase init`/`start` の案内（ユーザーが実行）、ディレクトリ作成、deploy.sh 生成
- **Bubble**: アプリ URL 確認、API Connector の設定状況確認
- 環境構築記録を `.dev-steroid/docs/70_env_setup/env-setup-record.md` に出力
- `platforms` 未定義の場合は自動的に `not_applicable` となりスキップ可能

#### 設計書リファクタリング

`/ds-doc-refactoring` で全 approved 設計書を横断的に再レビュー・修正します。

- テンプレート準拠性、設計書間の ID 整合性、用語統一、記述不足、リンク整合、分割粒度の妥当性を一括チェック
- 検出した問題を依存順に自動修正し、既存の ds-doc-reviewer で再検証
- 修正後はユーザーの明示的承認によりコミット（status は `approved` を維持）
- 設計書テンプレート修正後や、任意のタイミングで実行可能

### 2. spec-phase — Issue を仕様化する

PRD が揃ったら Issue 単位で仕様を切ります。5 つの delta を順に作成し、最後に GitHub Issues に登録します。

#### spec-phase コマンド体系

```
ds-spec-init
  ↓
ds-spec-requirements  → requirements_delta.md（要件確定）
  ↓
ds-spec-doc           → doc_delta.md（PRD 修正情報）
  ↓
ds-spec-design        → design_delta.md（実装情報）
  ↓
ds-spec-test          → test_delta.md（テスト情報）
  ↓
ds-spec-plan          → plan_delta.md（作業プラン + Issue 戦略）
  ↓
ds-spec-commit        → GitHub Issues 登録（親 + sub-issue）
```

#### delta 構成（5 delta）

| delta | 出力ファイル | 目的 |
|---|---|---|
| requirements_delta | `issues/<Issue>/requirements_delta.md` | 要件確定 |
| doc_delta | `issues/<Issue>/doc_delta.md` | PRD 修正情報 |
| design_delta | `issues/<Issue>/design_delta.md` | 実装情報 |
| test_delta | `issues/<Issue>/test_delta.md` | テスト情報 |
| plan_delta | `issues/<Issue>/plan_delta.md` | 作業プラン + Issue 戦略 |

#### コマンド詳細

1. `/ds-spec-init <Issue名>` — `issues/<Issue名>/` を作成し、メタ情報を設定
2. `/ds-spec-requirements` — 要件 delta（requirements_delta.md）
3. `/ds-spec-doc` — PRD修正情報 delta（doc_delta.md）
4. `/ds-spec-design` — 設計 delta（design_delta.md）
5. `/ds-spec-test` — テスト delta（test_delta.md）
6. `/ds-spec-plan` — 作業計画 + Issue戦略（plan_delta.md）
7. `/ds-spec-commit` — GitHub Issues 登録（親 + sub-issues）or ローカル commit

#### 設計書からの一括 Issue 起票

`/ds-issue-create-from-docs` で設計書を正として実装のギャップを分析し、Issue を一括起票します。

- **Phase A**: 設計書 vs コードベースのギャップ分析 → Issue 分割戦略を提案 → ユーザー承認
- **Phase B**: 承認された戦略に基づき、各 Issue の 5 delta を一括生成 → 既存 reviewer で AI レビュー → Issue 単位でユーザー承認（一括・複数指定対応）
- 承認後は `/ds-spec-commit <Issue名>` で GitHub Issues 登録に進む
- docフェーズ直後（実装ゼロ）でも、設計書修正後の差分検出でも使用可能

#### Sub-issue 構造

ds-spec-commit は plan_delta 内の Issue 戦略 YAML に従い、GitHub 上に親 Issue + sub-issue を作成します。

```
#100 add-login（親 Issue ← requirements_delta）
 ├── #101 [doc] add-login（← doc_delta）
 ├── #102 [design] add-login（← design_delta）
 └── #103 [test] add-login（← test_delta）
```

分割時は design/test が scope ごとに複数 sub-issue になります。

### 3. issues-phase — 実装する

`/ds-issue-impl <Issue名>` で実装を開始します。

- **①doc 工程**：納品物PRD（`.dev-steroid/docs/**`）を doc sub-issue の仕様に合わせて更新
- **scope 別ループ**（分割時は scope ごとに繰り返し）:
  - **②test_impl 工程**：ACに基づくテストを先に実装（RED 確認）
  - **③code 工程**：テストを通す最小実装（GREEN）→ 改善（REFACTOR）
  - **④test_exec 工程**：全テスト実行 + 回帰確認。失敗時は③に戻って修正
- **git.auto=true**: 各工程完了後に自動 commit、全完了後に PR 作成

検証形態は `state.issue.json` の `meta.verify_type`（`automated` / `manual` / `hybrid`）で宣言します。

仕様の参照方法は `git.auto` に応じて自動切替:
- **git.auto=true**: `gh issue view <sub_issue_id>` で sub-issue 本文を取得
- **git.auto=false**: ローカルの delta ファイルを直接参照

#### No-Code / Hybrid の場合（Bubble + Supabase 等）

`impl_type` が `hybrid` または `no-code` の場合、scope の `platform.type` に応じて工程が切り替わります。

**code scope（Supabase 等）** — AI が自律実行（従来動作と同一）:
- ②test_impl: 自動テスト実装（pgTAP, Deno Test 等）
- ③code: DDL/RLS/Edge Functions のファイル生成
- ④test_exec: ローカル DB でテスト実行

**deploy_gate**（hybrid 時のみ、code scope と no-code scope の間に挿入）:
- AI がクラウド反映手順を案内（`supabase db push --linked`, `functions deploy` 等）
- ユーザーが手動でクラウド反映を実施し、「デプロイ完了」と報告

**no-code scope（Bubble 等）** — AI が指示書を生成し、ユーザーが手動実装:
- ②test_impl: 手動検証手順書を策定（テスト実施条件含む）
- ③code: **操作指示書を生成** → ユーザーが Bubble Editor で手動実装 → 「完了」報告
- ④test_exec: **証跡テンプレートを生成** → ユーザーが手動検証 → 結果報告 → AI が判定

```
hybrid の実行フロー:
  ①doc（AI自律）
  ↓
  [db-and-api scope] ②→③→④（AI自律、ローカルDB）
  ↓
  【deploy_gate】ユーザーがクラウド反映
  ↓
  [bubble-ui scope] ②→③→④（操作指示書→手動実装→手動検証）
  ↓
  PR 作成
```

操作指示書は `issues/<Issue>/guides/<scope>_operation_guide.md` に、検証証跡は `issues/<Issue>/evidence/<scope>_evidence.md` に出力されます。

## 承認フロー

すべての成果物は以下のサイクルで進みます：

```
AI生成(draft) → AIレビュー → reviewed(承認待ち) → ユーザー承認(approved)
```

承認待ちになると、Claude が作業要約・検証状況・承認事項を提示します。
ユーザーの応答は 3 種類：

- `承認` — 次のステップに進む
- `差し戻し: <指示>` — 修正して再レビュー
- `保留` — state は変更せず、追加質問のみ

## ディレクトリ構造

```
.claude/
  commands/          # スラッシュコマンド（統括手順を内包）
  agents/            # subagent 定義（runner / reviewer / analyzer / fixer / generator）
  settings.json      # 権限設定
.dev-steroid/
  config.json        # git設定、platforms定義、PRD種類、実装パス等
  state.product.json # プロジェクト全体の state
  docs/              # 生成された PRD（ds-doc で自動作成）
    70_env_setup/    # 環境構築記録（ds-env-setup で生成）
  template/          # PRD テンプレート（6種 + platforms対応5種）
    prd-requirements/
    prd-basic-design/
    ...
    prd-supabase-data-design/       # Supabase データ設計
    prd-bubble-supabase-basic-design/   # Bubble+Supabase 基本設計
    prd-bubble-supabase-detailed-design/ # Bubble+Supabase 詳細設計
    prd-supabase-security-design/   # Supabase セキュリティ設計
    prd-bubble-supabase-test-design/    # Bubble+Supabase テスト設計
  README.md          # 本ファイル
  issue-plan.md      # Issue 分割戦略（ds-issue-create-from-docs で生成）
issues/              # Issue ごとの成果物（ds-spec-init で自動作成）
  <Issue名>/
    state.issue.json  # メタ情報（impl_type/verify_type/target_paths）含む
    requirements_delta.md   # 要件確定
    doc_delta.md            # PRD修正情報
    design_delta.md         # 実装情報
    test_delta.md           # テスト情報
    plan_delta.md           # 作業プラン + Issue戦略YAML
    guides/                 # 操作指示書（no-code scope の ③code で生成）
      <scope>_operation_guide.md
    evidence/               # 検証証跡（no-code scope の ④test_exec で生成）
      <scope>_evidence.md
CLAUDE.md            # Claude Code が最初に読むルールファイル
```

## カスタマイズ

| やりたいこと | 変更場所 |
|---|---|
| Git 自動操作の有効/無効 | `config.json` の `git.auto`（`true` / `false`） |
| feature branch の起点 | `config.json` の `git.base_branch` |
| PRD の種類を増減 | `config.json` の `docs_phase_map` |
| platforms（No-Code対応）を設定 | `config.json` の `platforms`（type: code / no-code） |
| deploy_gate のデプロイ手順を変更 | `config.json` の `platforms.<name>.deploy_commands` |
| Bubble アプリの URL を変更 | `/ds-env-setup` で再設定（`.dev-steroid/docs/70_env_setup/env-setup-record.md` に記録） |
| テンプレートを変える | `.dev-steroid/template/prd-{name}/template.md` |
| 生成ルールを追加 | テンプレ内 `<!-- generation_hints: ... -->` |
| レビュー基準を追加 | テンプレ内 `<!-- review_criteria: ... -->` |
| 実装対象パスを変更 | `config.json` の `code_paths` / `test_paths` |
| Bash を確認なし実行 | `settings.json` で `Bash` を `allow` に移動 |
| reviewer のモデルを変える | 各 reviewer の YAML frontmatter `model`（`inherit` / `sonnet` / `haiku`） |
| docs-phase runner に Bash を追加 | `agents/.../ds-doc-runner.md` の YAML frontmatter に `Bash` を追加 |
| 秘匿ファイルを追加 | `settings.json` の `deny` にパターン追加 |
| Supabase リモート操作を AI に許可 | `settings.json` の `deny` から該当コマンドを削除（**非推奨**） |
| 外部 URL アクセスを許可 | `settings.json` の `deny` から `WebFetch` を削除（セキュリティリスクに注意） |

> **注意**: `docs_phase_map` の `depends_on` は単一キーのみ対応。並列依存（複数の前提条件）が必要な場合は、線形チェーンに変換するか、本キットの拡張が必要です。

## トラブルシューティング

**`/ds-spec-commit` で blocked になる**
→ `artifacts` の全項目（requirements_delta / doc_delta / design_delta / test_delta / plan_delta）が `approved` か確認。`/ds-issue-status <Issue名>` で現状を確認できます。

**`gh` 関連のエラー（git.auto=true の場合）**
→ `gh auth status` で認証状態を確認。`gh repo view` でリポジトリへのアクセス権を確認。

**Sub-issue API が利用できない（git.auto=true の場合）**
→ GitHub の Sub-issues 機能はリポジトリ/Org の設定で有効化が必要な場合があります。`/ds-init` で事前チェックされます。利用できない場合は `git.auto=false` で運用してください。

**「仕様追加禁止」で止まる**
→ Issue + delta にない変更は blocked になります。仕様を追加したい場合は spec-phase のコマンドで delta を修正してください。

**PRD を対象外にしたい**
→ `/ds-doc <doc_type>` 実行時に「このドキュメントは不要」と伝えてください（`requirements` を除く）。

**`/ds-env-setup` で blocked になる**
→ `/ds-env-setup` は全設計書が approved になった後に実行します。`/ds-doc all` で未完了の設計書を処理してください。`platforms` 未定義の場合は自動的に `not_applicable` となるため、実行不要です。

**Supabase CLI が見つからない**
→ `supabase --version` で CLI の存在を確認。未インストールの場合は公式ドキュメントに従いインストールしてください。Docker Desktop も必要です。

**deploy_gate で止まっている**
→ `supabase db push --linked` と `supabase functions deploy` を手動実行し、Bubble の API Connector URL がクラウド Supabase を指していることを確認してから「デプロイ完了」と報告してください。

**Bubble 手動テストが全て FAIL になる**
→ Supabase 側のクラウド反映（DDL/RLS/Edge Functions）が完了していることを確認してください。deploy_gate をスキップすると手動テストは全て失敗します。

**操作指示書の手順が不明瞭**
→ `差し戻し: <具体的な問題点>` で guide-runner に再生成を指示できます。Bubble Editor の設定画面名やプロパティ名を含めて指摘すると精度が上がります。

**`waiting_user` 状態から進まない**
→ AI はユーザーの手動操作完了報告を待っています。操作完了後「完了」と報告してください。`/ds-issue-status <Issue名>` で待機中の工程を確認できます。
