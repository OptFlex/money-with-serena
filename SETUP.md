# Supabaseセットアップ手順

## 1. Supabaseプロジェクトの作成

1. [Supabase](https://supabase.com/)にアクセスしてアカウントを作成
2. 「New Project」をクリック
3. プロジェクト名を入力（例: `money-with-serena`）
4. データベースパスワードを設定（メモしておく）
5. リージョンを選択（日本に近い場合は `Northeast Asia (Tokyo)`）
6. 「Create new project」をクリック

## 2. Google認証の設定

1. Supabaseダッシュボードで「Authentication」→「Providers」をクリック
2. 「Google」を探して「Enable」をオンにする
3. Google Cloud Consoleでの設定が必要です：
   - [Google Cloud Console](https://console.cloud.google.com/)にアクセス
   - 新しいプロジェクトを作成
   - 「APIs & Services」→「OAuth consent screen」で設定
   - 「Credentials」→「Create Credentials」→「OAuth 2.0 Client ID」
   - アプリケーションタイプ: Web application
   - 承認済みのリダイレクトURIに以下を追加:
     ```
     https://<your-project-ref>.supabase.co/auth/v1/callback
     ```
   - クライアントIDとクライアントシークレットをコピー
4. SupabaseのGoogle設定に戻り、クライアントIDとシークレットを貼り付け
5. 「Save」をクリック

## 3. データベーステーブルの作成

Supabaseダッシュボードで「SQL Editor」を開き、以下のSQLを実行：

```sql
-- ホワイトリストテーブル
CREATE TABLE allowed_emails (
  id uuid DEFAULT gen_random_uuid() PRIMARY KEY,
  email text UNIQUE NOT NULL,
  created_at timestamp with time zone DEFAULT timezone('utc'::text, now()) NOT NULL
);

-- トランザクションテーブル
CREATE TABLE transactions (
  id uuid DEFAULT gen_random_uuid() PRIMARY KEY,
  user_email text NOT NULL,
  type text NOT NULL CHECK (type IN ('lend', 'borrow', 'repay')),
  amount integer NOT NULL,
  description text NOT NULL,
  date date NOT NULL,
  repaid boolean DEFAULT false,
  repaid_transactions text[],
  created_at timestamp with time zone DEFAULT timezone('utc'::text, now()) NOT NULL
);

-- インデックスの作成（パフォーマンス向上）
CREATE INDEX idx_transactions_date ON transactions(date DESC);
CREATE INDEX idx_transactions_user_email ON transactions(user_email);
CREATE INDEX idx_transactions_repaid ON transactions(repaid) WHERE repaid = false;

-- ホワイトリストに許可するメールアドレスを追加（あなたと彼女のメールアドレス）
INSERT INTO allowed_emails (email) VALUES
  ('your-email@gmail.com'),
  ('girlfriend-email@gmail.com');
```

**重要**: `your-email@gmail.com` と `girlfriend-email@gmail.com` を実際のメールアドレスに変更してください。

**フィールドの説明:**
- `type`: 'lend'(貸した) / 'borrow'(借りた) / 'repay'(返済)
- `repaid`: 借りが返済済みかどうか（borrowの場合のみ使用）
- `repaid_transactions`: 返済時にどの借りを返したか（repayの場合のみ使用）

## 4. Row Level Security (RLS) の設定

セキュリティのため、以下のSQLも実行：

```sql
-- RLSを有効化
ALTER TABLE transactions ENABLE ROW LEVEL SECURITY;
ALTER TABLE allowed_emails ENABLE ROW LEVEL SECURITY;

-- 認証されたユーザーのみがトランザクションを読み書きできる
CREATE POLICY "Allow authenticated users to read transactions"
  ON transactions FOR SELECT
  TO authenticated
  USING (true);

CREATE POLICY "Allow authenticated users to insert transactions"
  ON transactions FOR INSERT
  TO authenticated
  WITH CHECK (true);

CREATE POLICY "Allow authenticated users to update transactions"
  ON transactions FOR UPDATE
  TO authenticated
  USING (true);

CREATE POLICY "Allow authenticated users to delete transactions"
  ON transactions FOR DELETE
  TO authenticated
  USING (true);

-- allowed_emailsは認証されたユーザーが読み取りのみ可能
CREATE POLICY "Allow authenticated users to read allowed emails"
  ON allowed_emails FOR SELECT
  TO authenticated
  USING (true);
```

## 5. Supabase設定情報を取得

1. Supabaseダッシュボードで「Settings」→「API」をクリック
2. 以下の情報をコピー：
   - Project URL（例: `https://xxxxx.supabase.co`）
   - `anon` `public` key

## 6. index.htmlファイルを更新

`index.html`の先頭付近にある以下の部分を、あなたの情報に書き換えてください：

```javascript
const SUPABASE_URL = 'YOUR_SUPABASE_URL';
const SUPABASE_ANON_KEY = 'YOUR_SUPABASE_ANON_KEY';
```

## 完了！

これで設定は完了です。`index.html`をブラウザで開いて、Googleでログインしてみてください。

---

## 参考資料

データベース設計の詳細については、`DATABASE_DESIGN.md`を参照してください。
- テーブル構造の詳細
- トランザクションタイプの説明
- 視点変換のロジック
- 返済フロー
- セキュリティ設計
