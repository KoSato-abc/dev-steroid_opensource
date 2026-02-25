<!-- generation_hints:
- 認証・認可・入力検証・データ保護・監査の5層で設計する
- Supabase Auth + RLS + Edge Functions の3層防御を基本とする
- anon key と service_role key の使い分けを明確にする
- Bubble 側のクライアントバリデーションと Supabase 側のサーババリデーションの両方を記述する
- RLS ポリシーの詳細は データ設計書 を参照する形でよい（重複は避ける）
- API キー管理方針は特に丁寧に記述する（Bubble に露出する情報の整理）
-->
<!-- review_criteria:
- 認証方式が明確に定義されている
- 認可モデル（ロールと権限）が定義されている
- RLS ポリシーの方針が記述されている（詳細はデータ設計書参照で可）
- Edge Functions の認可チェックが定義されている
- anon key / service_role key の使い分けが明記されている
- Bubble に露出する API キーの範囲が整理されている
- 入力検証がクライアント（Bubble）とサーバ（Supabase）の両方で定義されている
- データ保護（通信暗号化、保存時暗号化）が記述されている
-->

# セキュリティ設計書（Supabase）
> 本書は Supabase Auth + RLS + Edge Functions の3層防御を基本としたセキュリティ設計を記述する。Bubble（クライアント）に露出する情報の管理を含む。
> **全セクション必須**。該当しない場合は「該当なし」と記載し、セクション自体は削除しない。

## 1. 認証設計（Supabase Auth）

### 1.1 認証方式

| 方式 | 対象 | 備考 |
|---|---|---|
| Email / Password | 一般ユーザー | |
| OAuth（Google 等） | 一般ユーザー | （使用する場合） |
| Magic Link | | （使用する場合） |

### 1.2 認証フロー（Bubble 連携）

```
1. ユーザーが Bubble のログインページにアクセス
2. Bubble → API Connector → Supabase Auth（/auth/v1/token）
3. Supabase Auth がアクセストークン（JWT）を返却
4. Bubble がトークンを Custom State に保存
5. 以降の API 呼び出しで Authorization: Bearer {token} を付与
6. トークンリフレッシュ: /auth/v1/token?grant_type=refresh_token
```

### 1.3 JWT 設計
- **有効期限**: （例: 3600秒）
- **リフレッシュトークン**: （有効期限、ローテーション方針）
- **カスタムクレーム**: （role 等を含める場合）

### 1.4 セッション管理
- Bubble 側: Custom State / Cookie でトークン管理
- ログアウト: Supabase Auth の /auth/v1/logout 呼び出し
- トークン失効時: 自動リフレッシュ or ログイン画面にリダイレクト

---

## 2. 認可設計

### 2.1 ロール / 権限モデル

| ロール | 説明 | 権限概要 |
|---|---|---|
| anonymous | 未認証ユーザー | 公開データの閲覧のみ |
| authenticated | 認証済みユーザー | 自分のデータの CRUD |
| admin | 管理者 | 全データの管理 |

### 2.2 RLS ポリシー方針
- **基本原則**: 全テーブルで RLS を有効にし、デフォルトは全拒否
- **ユーザーデータ**: `auth.uid() = user_id` で自分のデータのみアクセス
- **公開データ**: `FOR SELECT USING (is_public = true)` 等
- **管理者**: `auth.jwt() ->> 'role' = 'admin'` 等

※ RLS ポリシーの詳細な SQL は **データ設計書** を参照

### 2.3 Edge Functions の認可
- JWT からユーザー情報を取得（`supabase.auth.getUser()`）
- ロールチェック: JWT のカスタムクレーム or DB の role テーブルを参照
- リソース所有権チェック: DB のレコードの user_id と auth.uid() を照合

---

## 3. 入力検証

### 3.1 Bubble 側（クライアントバリデーション）
- フォーム入力: Bubble の Conditional / Regex で即時チェック
- **注意**: クライアントバリデーションはUXのためであり、セキュリティは保証しない

| 入力項目 | バリデーション | 方法 |
|---|---|---|

### 3.2 Supabase 側（サーババリデーション）

#### Edge Functions
- リクエストボディのスキーマ検証（型、必須、長さ、形式）
- 不正値の早期リジェクト

#### DB 制約
- NOT NULL / CHECK 制約
- 外部キー制約
- UNIQUE 制約

---

## 4. データ保護

### 4.1 通信暗号化
- Supabase: 全通信 HTTPS（TLS 1.2+）
- Bubble → Supabase: API Connector で HTTPS 強制

### 4.2 API キー管理方針

| キー | 用途 | 露出範囲 | 保護方法 |
|---|---|---|---|
| `anon key` | Bubble からの API 呼び出し | Bubble API Connector に設定（公開可） | RLS で保護 |
| `service_role key` | Edge Functions 内部 | **絶対に公開しない** | 環境変数のみ |

- **重要**: `anon key` は公開前提であり、RLS が唯一の防御層となる
- `service_role key` を Bubble や JavaScript（Toolbox）に絶対に設定しない

### 4.3 PII（個人情報）の取り扱い
（個人情報の保存方針、マスキング、削除方針）

---

## 5. 監査・ログ

### 5.1 監査ログ方針
（重要操作のログ記録方針。DB トリガー / Edge Functions での記録）

### 5.2 エラーログ
（Supabase のログ確認方法、Edge Functions のエラー追跡）

---

## 6. 未決・TODO
-

## 7. レビューステータス
- AIレビュー：未実施 / reviewed / 指摘あり
- ユーザー承認：未承認 / approved
- 備考：
