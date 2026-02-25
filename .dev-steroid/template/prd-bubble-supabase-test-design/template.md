<!-- generation_hints:
- テスト戦略は Supabase（自動）+ Bubble（手動）の2系統で構成する
- Bubble 手動テストの前提条件（Supabase クラウド反映済み）を必ず明記する
- 依存チェーン（ローカル自動 → クラウド反映 → 手動テスト）を明確に示す
- テスト ID 体系: TC-Sxxx（Supabase 自動）, TC-Bxxx（Bubble 手動）
- pgTAP テストは supabase/tests/ 配下、Deno Test は tests/functions/ 配下
- 手動テストシナリオは「操作手順」「期待結果」「前提条件」を含む
-->
<!-- review_criteria:
- テスト戦略が2系統（自動+手動）で明確に定義されている
- 全 AC-ID が少なくとも1つの TC-ID にトレースされている
- Bubble 手動テストの前提条件に「Supabase クラウド反映済み」が明記されている
- テスト実行順序（依存チェーン）が定義されている
- pgTAP テストのファイル配置が supabase/tests/ 配下で明示されている
- Edge Functions テストのファイル配置が tests/functions/ 配下で明示されている
- 手動テストシナリオに操作手順・期待結果・前提条件が含まれている
- 回帰テスト方針が定義されている
-->

# テスト設計書（Bubble + Supabase）
> 本書は Supabase（自動テスト: pgTAP + Deno Test）+ Bubble（手動テスト）の2系統でテスト設計を記述する。
> **Bubble の手動テストは Supabase のクラウド反映が完了していることが前提**。この依存チェーンを厳守する。
> **全セクション必須**。該当しない場合は「該当なし」と記載し、セクション自体は削除しない。

## 1. テスト戦略

### 1.1 概要
本プロジェクトは Supabase（自動テスト）と Bubble（手動テスト）の **2系統** でテストを実施する。

| テスト系統 | プラットフォーム | 方式 | ツール | 実行者 |
|---|---|---|---|---|
| 自動テスト | Supabase | pgTAP + Deno Test | `supabase test db` / `deno test` | AI / CI |
| 手動テスト | Bubble | 手動操作 + 目視確認 | Bubble プレビューモード | 人間 |

### 1.2 テストレベル × プラットフォーム対応表

| テストレベル | Supabase（自動） | Bubble（手動） |
|---|---|---|
| 単体テスト | pgTAP（テーブル/RLS/関数） | N/A |
| 結合テスト | Deno Test（Edge Functions） | API Connector → Functions 疎通 |
| E2E テスト | N/A | 画面操作 → DB 反映確認 |

---

## 2. テスト環境

### 2.1 自動テスト環境（ローカル）
- **前提**: `supabase start` でローカル DB が起動済み
- **pgTAP**: `supabase test db` で実行（supabase/tests/*.test.sql）
- **Deno Test**: `deno test tests/functions/` で実行
- **CI**: GitHub Actions で `supabase start` → `supabase test db` → `deno test`

### 2.2 手動テスト環境

> ⚠️ **最重要: 手動テストの前提条件**
>
> Bubble の手動テストは、以下が **全てクラウドの Supabase に反映済み** であることが前提:
> 1. DDL（テーブル定義）: `supabase db push --linked`
> 2. RLS ポリシー: 同上
> 3. Edge Functions: `supabase functions deploy`
> 4. Bubble の API Connector がクラウド Supabase の URL を指している
>
> **未反映の場合、全ての手動テストは FAIL となる。**

- **Bubble**: プレビューモード（開発版）
- **Supabase**: クラウドの開発用プロジェクト

---

## 3. トレーサビリティ

| AC-ID | TC-ID | テスト種別 | プラットフォーム | テスト手段 |
|---|---|---|---|---|

---

## 4. Supabase 自動テスト

### 4.1 pgTAP テスト（DB テスト）

| TC-ID | ファイル | テスト対象 | 概要 | 対応 AC |
|---|---|---|---|---|

#### テスト詳細

##### TC-Sxxx: {テスト名}
- **ファイル**: `supabase/tests/{テスト名}.test.sql`
- **テスト対象**: {テーブル名 / RLS ポリシー / 関数}
- **テスト内容**:
  ```sql
  BEGIN;
  SELECT plan(N);  -- N = テスト数

  -- テスト 1: テーブルが存在する
  SELECT has_table('public', '{table_name}', 'テーブル {table_name} が存在する');

  -- テスト 2: RLS が有効
  SELECT policies_are('public', '{table_name}', ARRAY['{policy_name}']);

  -- テスト 3: 認証ユーザーが自分のデータを参照できる
  -- （テストユーザーのセッション設定 → SELECT → 結果検証）

  SELECT * FROM finish();
  ROLLBACK;
  ```
- **実行**: `supabase test db`

（テストケースごとに繰り返す）

### 4.2 Deno Test（Edge Functions テスト）

| TC-ID | ファイル | テスト対象 | 概要 | 対応 AC |
|---|---|---|---|---|

#### テスト詳細

##### TC-Sxxx: {テスト名}
- **ファイル**: `tests/functions/{テスト名}.test.ts`
- **テスト対象**: {Edge Functions 名}
- **テスト内容**:
  ```typescript
  import { assertEquals } from "https://deno.land/std/assert/mod.ts";

  Deno.test("{テスト名}", async () => {
    const response = await fetch("http://localhost:54321/functions/v1/{function_name}", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Authorization": "Bearer {test_token}"
      },
      body: JSON.stringify({ /* テストデータ */ })
    });
    assertEquals(response.status, 200);
    const data = await response.json();
    // アサーション
  });
  ```
- **実行**: `deno test tests/functions/`

（テストケースごとに繰り返す）

---

## 5. Bubble 手動テスト

### 5.1 手動テスト実施条件（必須確認）

> 以下のチェックリストを**テスト実施前に必ず確認**する:
> - [ ] Supabase DDL がクラウドに反映済み（`supabase db push --linked`）
> - [ ] Supabase RLS ポリシーがクラウドに反映済み
> - [ ] Edge Functions がデプロイ済み（`supabase functions deploy`）
> - [ ] Bubble API Connector がクラウド Supabase URL を指している
> - [ ] テスト用アカウントが作成済み

### 5.2 手動テストシナリオ

| TC-ID | AC-ID | ページ | 操作手順 | 期待結果 | 前提条件 |
|---|---|---|---|---|---|

#### シナリオ詳細

##### TC-Bxxx: {テスト名}
- **対応 AC**: AC-xxx
- **ページ**: {ページ名}
- **前提条件**: 
  - （必要なデータ、ログイン状態等）
- **操作手順**:
  1. {URL} にアクセス
  2. {操作内容}
  3. {操作内容}
- **期待結果**: 
  - （画面表示、DB 状態、エラー有無）
- **判定基準**: 
  - （Pass / Fail の明確な基準）

（テストケースごとに繰り返す）

---

## 6. テスト実行順序（依存チェーン）

```
①ローカル自動テスト（AI / CI が実行）
  supabase test db → PASS
  deno test → PASS
      │
      ▼
②クラウド反映（人間が実行）
  supabase db push --linked
  supabase functions deploy
  Bubble API Connector URL 確認
      │
      ▼
③ Bubble 手動テスト（人間が実行）
  ブラウザで Bubble プレビュー → シナリオ実行 → 証跡記録
```

---

## 7. 回帰テスト方針

### 7.1 自動テスト（Supabase）
- 全テストを毎回実行（`supabase test db` + `deno test`）
- CI で PR ごとに自動実行

### 7.2 手動テスト（Bubble）
- 変更影響範囲の手動テストを実施
- 全機能の手動テストはリリース前に実施

---

## 8. テストデータ方針

### 8.1 自動テスト用
- pgTAP テスト内で `BEGIN` → テストデータ INSERT → テスト → `ROLLBACK`
- Deno Test: ローカル Supabase に対してテストデータを投入

### 8.2 手動テスト用
- クラウド Supabase の開発環境にシードデータを投入
- テスト用アカウント（メール/パスワード）を事前作成

---

## 9. 未決・TODO
-

## 10. レビューステータス
- AIレビュー：未実施 / reviewed / 指摘あり
- ユーザー承認：未承認 / approved
- 備考：
