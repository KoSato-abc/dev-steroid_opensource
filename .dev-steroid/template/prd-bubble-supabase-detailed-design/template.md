<!-- generation_hints:
- ファイル分割必須: index.md + 機能単位のサブファイル
- 各サブファイル冒頭に「対象範囲」「対応 BPG/BWF/IF-ID」を明記する
- Bubble 側は「Editor で何をどう設定するか」が再現できる具体性で記述する
- Supabase 側は「supabase/ 配下のどのファイルに何を書くか」が明確になる粒度で記述する
- DDL / RLS は実際の SQL に近い粒度（コピペで使えるレベル）
- Bubble → Supabase のパラメータマッピングを明示する
- ワークフローは各 Step のプロパティ設定・条件分岐・パラメータマッピングを記述する
- Edge Functions の処理フローは擬似コードで記述する
-->
<!-- review_criteria:
- Bubble の設定が Editor 上で再現可能な粒度である
- Supabase のファイルパスが具体的に指定されている
- DDL が CREATE TABLE 文として実行可能な粒度である
- RLS が CREATE POLICY 文として実行可能な粒度である
- Edge Functions の処理フローが擬似コードで記述されている
- Bubble → Supabase のパラメータマッピングが明示されている
- 各ワークフローの Step にプロパティ設定が記述されている
- 分割型: index.md にサブファイル一覧があり、サブファイル冒頭に対象範囲が明記されている
-->

# 詳細設計書（Bubble + Supabase）
> 本書は Bubble（フロントエンド）+ Supabase（バックエンド）構成での詳細設計を記述する。Bubble 側は Editor で再現可能な粒度、Supabase 側はファイルに直接記述可能な粒度で記述する。
> **全セクション必須**。該当しない場合は「該当なし」と記載し、セクション自体は削除しない。

## 1. Bubble 実装詳細

### 1.1 Option Sets

| Option Set 名 | 値 | Display | 用途 |
|---|---|---|---|

### 1.2 Custom Data Types（App Data）

| Data Type 名 | フィールド | 型 | 概要 |
|---|---|---|---|

※ Supabase に DB を置く場合、Bubble の App Data は最小限にし、Supabase を正とする。
ここでは Bubble 内部でのみ使用するデータ（一時的な状態等）を定義する。

### 1.3 Custom States 設計

| ページ/Element | State 名 | 型 | 初期値 | 用途 |
|---|---|---|---|---|

### 1.4 ページ詳細

#### BPG-xxx: {ページ名}

**レイアウト構成**:
```
[Page: {ページ名}]
  ├── [Group: header]
  │   ├── [Text: title]
  │   └── [Button: action]
  ├── [Group: content]
  │   ├── [Repeating Group: list] ← Data Source: ...
  │   │   └── [Group: item]
  │   │       ├── [Text: name]
  │   │       └── [Button: edit]
  └── [Group: footer]
```

**データバインディング**:
| Element | プロパティ | 値/式 |
|---|---|---|

**Conditional 表示**:
| Element | 条件 | 表示変更 |
|---|---|---|

（ページごとに繰り返す）

### 1.5 ワークフロー詳細

#### BWF-xxx: {ワークフロー名}

| Step | アクション | 設定内容 |
|---|---|---|
| 1 | {アクション名} | **プロパティ**: ... **パラメータ**: ... |
| 2 | {アクション名} | **条件分岐**: Only when ... |
| 3 | {アクション名} | **Supabase 呼び出し**: IF-xxx |

**パラメータマッピング（Bubble → Supabase）**:
| Bubble 側 | → | Supabase 側 | 備考 |
|---|---|---|---|

（ワークフローごとに繰り返す）

---

## 2. Supabase Edge Functions 実装詳細

### 2.1 共通設計

#### CORS 設定
```typescript
// supabase/functions/_shared/cors.ts
export const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type',
};
```

#### JWT 検証パターン
```typescript
// 共通の認証チェック
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

const supabase = createClient(
  Deno.env.get('SUPABASE_URL')!,
  Deno.env.get('SUPABASE_ANON_KEY')!,
  { global: { headers: { Authorization: req.headers.get('Authorization')! } } }
);

const { data: { user }, error } = await supabase.auth.getUser();
if (error || !user) return new Response('Unauthorized', { status: 401 });
```

#### エラーレスポンス形式
```typescript
// 統一エラー形式
return new Response(
  JSON.stringify({ error: { code: 'XXX', message: '...' } }),
  { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
);
```

### 2.2 関数詳細

#### IF-xxx: {関数名}
- **ファイルパス**: `supabase/functions/{関数名}/index.ts`

**処理フロー（擬似コード）**:
```
1. リクエストバリデーション
   - Method チェック
   - Content-Type チェック
   - パラメータ検証
2. 認証チェック
   - JWT からユーザー取得
   - 権限チェック
3. メイン処理
   - DB 操作（INSERT/UPDATE/SELECT/DELETE）
   - 外部 API 呼び出し（必要な場合）
4. レスポンス生成
   - 成功: { data: ... }
   - エラー: { error: { code, message } }
```

**エラーケース**:
| エラー条件 | HTTP Status | エラーコード | Bubble 側の表示 |
|---|---|---|---|

（関数ごとに繰り返す）

---

## 3. DDL 詳細

### 3.1 マイグレーション一覧

| ファイル名 | 内容 | 依存 |
|---|---|---|
| `{timestamp}_create_{table}.sql` | {テーブル名} 作成 | なし |

### 3.2 DDL 詳細

#### {timestamp}_create_{table}.sql

```sql
CREATE TABLE IF NOT EXISTS public.{table_name} (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  -- カラム定義
  created_at TIMESTAMPTZ DEFAULT now() NOT NULL,
  updated_at TIMESTAMPTZ DEFAULT now() NOT NULL
);

-- インデックス
CREATE INDEX idx_{table_name}_user_id ON public.{table_name}(user_id);

-- updated_at 自動更新トリガー
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER set_{table_name}_updated_at
  BEFORE UPDATE ON public.{table_name}
  FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

（マイグレーションファイルごとに繰り返す）

---

## 4. RLS 詳細

#### {table_name}

```sql
ALTER TABLE public.{table_name} ENABLE ROW LEVEL SECURITY;

-- SELECT: 自分のデータのみ参照可能
CREATE POLICY "{table_name}_select_own"
  ON public.{table_name} FOR SELECT
  USING (auth.uid() = user_id);

-- INSERT: 自分の user_id でのみ挿入可能
CREATE POLICY "{table_name}_insert_own"
  ON public.{table_name} FOR INSERT
  WITH CHECK (auth.uid() = user_id);

-- UPDATE: 自分のデータのみ更新可能
CREATE POLICY "{table_name}_update_own"
  ON public.{table_name} FOR UPDATE
  USING (auth.uid() = user_id)
  WITH CHECK (auth.uid() = user_id);

-- DELETE: 自分のデータのみ削除可能
CREATE POLICY "{table_name}_delete_own"
  ON public.{table_name} FOR DELETE
  USING (auth.uid() = user_id);
```

（テーブルごとに繰り返す）

---

## 5. 未決・TODO
-

## 6. レビューステータス
- AIレビュー：未実施 / reviewed / 指摘あり
- ユーザー承認：未承認 / approved
- 備考：
