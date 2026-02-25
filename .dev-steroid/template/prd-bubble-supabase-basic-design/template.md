<!-- generation_hints:
- ファイル分割必須: index.md + 機能単位のサブファイル（例: auth-flow.md, recipe-crud.md）
- 各サブファイル冒頭に「対象範囲」「対応 FR-ID / AC-ID」を明記する
- Bubble のページ設計は「どの Supabase リソースにどう接続するか」を必ず含める
- ワークフローのアクション列は Step 単位で Bubble Editor での操作に落ちる粒度にする
- Edge Functions は supabase/functions/ 配下のファイルパスを明示する
- 要件 → Bubble ページ/WF → Supabase Functions/DB のトレースを維持する
- ID 体系: BPG-xxx（ページ）, BRE-xxx（Reusable Element）, BWF-xxx（ワークフロー）, IF-xxx（Edge Functions）
-->
<!-- review_criteria:
- 全 FR-ID が少なくとも1つの BPG/BWF/IF にトレースされている
- Bubble → Supabase の接続方式（Plugin / API Connector / Toolbox）が明確に選定されている
- ページ設計にデータソースと認証要否が明記されている
- ワークフロー設計に呼出先の Supabase リソースが明記されている
- Edge Functions の入出力が定義されている
- エラーハンドリング方針が Supabase 側・Bubble 側の両方で定義されている
- 画面遷移図が存在し、ページ一覧と整合している
- 分割型: index.md にサブファイル一覧があり、サブファイル冒頭に対象範囲が明記されている
-->

# 基本設計書（Bubble + Supabase）
> 本書は Bubble（フロントエンド）+ Supabase（バックエンド）構成での基本設計を記述する。要件定義（FR/AC）からのトレースを前提とする。
> **全セクション必須**。該当しない場合は「該当なし」と記載し、セクション自体は削除しない。

## 1. サマリ

### 全体アーキテクチャ
（Bubble → Supabase の接続構成を文章で説明）

### システム構成図

```
[ユーザー] → [Bubble（フロントエンド）]
                    │
                    ├─ API Connector → [Supabase Edge Functions]
                    │                         │
                    │                         ▼
                    └─ Supabase Plugin → [Supabase PostgreSQL]
                                              │
                                         [Supabase Auth]
                                         [Supabase Storage]
```

### 接続方式一覧

| 接続手段 | 用途 | 認証方式 |
|---|---|---|
| Supabase Plugin | DB 直接読み書き | anon key + RLS |
| API Connector | Edge Functions 呼び出し | Bearer Token (JWT) |
| Toolbox (JavaScript) | クライアント側ロジック | N/A |

### データフロー概要
（主要な操作のデータの流れを簡潔に記述）

---

## 2. 要件トレース

| FR-ID | 機能名 | Bubble ページ (BPG) | Bubble WF (BWF) | Edge Functions (IF) | DB テーブル |
|---|---|---|---|---|---|

---

## 3. Bubble ページ設計

### 3.1 ページ一覧

| BPG-ID | ページ名 | URL パス | 概要 | 認証要否 |
|---|---|---|---|---|

### 3.2 画面遷移図

```mermaid
graph TD
    （画面遷移図をMermaid形式で記述）
```

### 3.3 ページ詳細

#### BPG-xxx: {ページ名}
- **目的**: 
- **URL**: 
- **認証**: 要 / 不要
- **データソース**: （Supabase のどのテーブル/Functions から取得するか）
- **主要入力要素**: 
- **権限制御**: （RLS で制御 / ページレベルでリダイレクト等）
- **対応 FR**: 

（ページごとに繰り返す）

---

## 4. Bubble Reusable Element 一覧

| BRE-ID | 要素名 | 概要 | 使用ページ |
|---|---|---|---|

---

## 5. Bubble ワークフロー設計

### 5.1 ワークフロー一覧

| BWF-ID | ページ | トリガー | 概要 | 呼出先 Supabase | 対応 AC |
|---|---|---|---|---|---|

### 5.2 ワークフロー詳細

#### BWF-xxx: {ワークフロー名}
- **ページ**: 
- **トリガー**: 
- **アクション列**:
  1. Step 1: {アクション概要}（→ 呼出先）
  2. Step 2: {アクション概要}
  3. Step 3: {アクション概要}
- **エラー時**: 
- **対応 AC**: 

（ワークフローごとに繰り返す）

---

## 6. Supabase Edge Functions 設計

### 6.1 一覧

| IF-ID | 関数名 | ファイルパス | Method | パス | 概要 |
|---|---|---|---|---|---|

### 6.2 詳細

#### IF-xxx: {関数名}
- **ファイルパス**: `supabase/functions/{関数名}/index.ts`
- **Method / Path**: 
- **認証**: Bearer Token (JWT) / 不要
- **入力**: 
  ```json
  { "field": "type" }
  ```
- **出力（成功）**: 
  ```json
  { "data": "..." }
  ```
- **出力（エラー）**: 
  ```json
  { "error": "message" }
  ```
- **対応 BWF**: 

（関数ごとに繰り返す）

---

## 7. エラーハンドリング方針

### 7.1 Supabase 側
- Edge Functions: HTTP ステータスコード + JSON エラーレスポンス
- DB: RLS 違反時のレスポンス
- 共通エラー形式:
  ```json
  { "error": { "code": "xxx", "message": "..." } }
  ```

### 7.2 Bubble 側
- API Connector のエラーハンドリング（Workflow: API 呼び出し → エラー時分岐）
- ユーザーへのエラー表示方法（Alert / Custom State / ページ遷移）

---

## 8. 非機能設計（要点）

- **パフォーマンス**: （ページネーション、キャッシュ方針等）
- **スケーラビリティ**: （Supabase のプラン、Bubble のキャパシティ等）
- **可用性**: （Supabase の SLA、Bubble のダウンタイム対応）

---

## 9. 未決・TODO
-

## 10. レビューステータス
- AIレビュー：未実施 / reviewed / 指摘あり
- ユーザー承認：未承認 / approved
- 備考：
