# データベース設計書

## 概要

このアプリは、二人（あなたと彼女）の間のお金の貸し借りを管理するシステムです。
各ユーザーが自分の視点でデータを見られるように設計されています。

## データモデル

### 1. allowed_emails テーブル

ログインを許可されたユーザーのメールアドレスを管理します。

| カラム名 | 型 | 制約 | 説明 |
|---------|-----|------|------|
| id | uuid | PRIMARY KEY | 自動生成されるID |
| email | text | UNIQUE, NOT NULL | 許可されたメールアドレス |
| created_at | timestamptz | NOT NULL | 作成日時 |

**データ例:**
```
id: "123e4567-e89b-12d3-a456-426614174000"
email: "user1@gmail.com"
created_at: "2025-01-05T10:00:00Z"
```

### 2. transactions テーブル

すべての貸し借り・返済の記録を保存します。

| カラム名 | 型 | 制約 | 説明 |
|---------|-----|------|------|
| id | uuid | PRIMARY KEY | 自動生成されるID |
| user_email | text | NOT NULL | 記録を登録したユーザーのメールアドレス |
| type | text | NOT NULL, CHECK | トランザクションの種類: 'lend', 'borrow', 'repay' |
| amount | integer | NOT NULL | 金額（円） |
| description | text | NOT NULL | 説明（例: "ランチ代"） |
| date | date | NOT NULL | 取引日 |
| repaid | boolean | NULLABLE | 返済済みフラグ（borrowの場合のみ使用） |
| repaid_transactions | text[] | NULLABLE | 返済した借りのIDリスト（repayの場合のみ使用） |
| created_at | timestamptz | NOT NULL | レコード作成日時 |

**トランザクションタイプの説明:**

#### `lend` (貸した)
- ユーザーが相手にお金を貸したことを記録
- `repaid`フィールドは使用しない

**例:**
```json
{
  "id": "abc123",
  "user_email": "user1@gmail.com",
  "type": "lend",
  "amount": 5000,
  "description": "ランチ代",
  "date": "2025-01-05",
  "repaid": null,
  "repaid_transactions": null
}
```

#### `borrow` (借りた)
- ユーザーが相手からお金を借りたことを記録
- `repaid`フィールドで返済済みかを管理（初期値: false）
- 返済済みの場合は`repaid: true`になる

**例（未返済）:**
```json
{
  "id": "def456",
  "user_email": "user2@gmail.com",
  "type": "borrow",
  "amount": 3000,
  "description": "映画代",
  "date": "2025-01-04",
  "repaid": false,
  "repaid_transactions": null
}
```

**例（返済済み）:**
```json
{
  "id": "def456",
  "user_email": "user2@gmail.com",
  "type": "borrow",
  "amount": 3000,
  "description": "映画代",
  "date": "2025-01-04",
  "repaid": true,
  "repaid_transactions": null
}
```

#### `repay` (返済)
- ユーザーが借りていたお金を返済したことを記録
- `repaid_transactions`フィールドにどの借り（borrowのID）を返済したかを記録
- 返済時、該当するborrowレコードの`repaid`を`true`に更新

**例:**
```json
{
  "id": "ghi789",
  "user_email": "user2@gmail.com",
  "type": "repay",
  "amount": 3000,
  "description": "返済",
  "date": "2025-01-05",
  "repaid": null,
  "repaid_transactions": ["def456"]
}
```

## データの視点変換

このアプリの特徴は、ログインしているユーザーの視点でデータを表示することです。

### 計算ロジック

#### 現在の収支
```
収支 = 貸した金額 + 返済された金額 - 借りた金額
     = (lendの合計 + repayの合計) - borrowの合計
```

#### ログインユーザーの視点での「貸した合計」
```
自分が貸した合計 =
  自分が登録した lend の合計 +
  相手が登録した borrow の合計
```

#### ログインユーザーの視点での「借りた合計」
```
自分が借りた合計 =
  自分が登録した borrow の合計 +
  相手が登録した lend の合計
```

### 例: User1とUser2の視点

**トランザクションデータ:**
1. User1が登録: type='lend', amount=5000
2. User2が登録: type='borrow', amount=3000

**User1でログインした場合:**
- 貸した合計: ¥8,000（自分のlend ¥5,000 + 相手のborrow ¥3,000）
- 借りた合計: ¥0

**User2でログインした場合:**
- 貸した合計: ¥0
- 借りた合計: ¥8,000（自分のborrow ¥3,000 + 相手のlend ¥5,000）

## 返済フロー

### 1. 借りの作成
```sql
INSERT INTO transactions (user_email, type, amount, description, date, repaid)
VALUES ('user1@gmail.com', 'borrow', 5000, 'ランチ代', '2025-01-05', false);
```

### 2. 返済の実行
```sql
-- 1. 返済レコードを作成
INSERT INTO transactions (user_email, type, amount, description, date, repaid_transactions)
VALUES ('user1@gmail.com', 'repay', 5000, '返済', '2025-01-06', ARRAY['借りのID']);

-- 2. 該当する借りを返済済みにマーク
UPDATE transactions
SET repaid = true
WHERE id = '借りのID';
```

### 3. 返済の削除（取り消し）
```sql
-- 1. 返済レコードを削除
DELETE FROM transactions WHERE id = '返済のID';

-- 2. 紐付けられた借りを未返済に戻す
UPDATE transactions
SET repaid = false
WHERE id IN (SELECT unnest(repaid_transactions) FROM transactions WHERE id = '返済のID');
```

## セキュリティ

### Row Level Security (RLS)

すべてのテーブルでRLSを有効化し、認証されたユーザーのみがアクセスできるようにします。

```sql
-- トランザクション: 認証ユーザーは全て読み書き可能
-- (二人しか使わないため、全員が全データを見られる設計)
CREATE POLICY "Allow authenticated users to read transactions"
  ON transactions FOR SELECT
  TO authenticated
  USING (true);

CREATE POLICY "Allow authenticated users to insert transactions"
  ON transactions FOR INSERT
  TO authenticated
  WITH CHECK (true);

CREATE POLICY "Allow authenticated users to delete transactions"
  ON transactions FOR DELETE
  TO authenticated
  USING (true);

CREATE POLICY "Allow authenticated users to update transactions"
  ON transactions FOR UPDATE
  TO authenticated
  USING (true);
```

### ホワイトリスト認証

ログイン時にメールアドレスが`allowed_emails`テーブルに登録されているかチェックします。
登録されていない場合はログインを拒否します。

## インデックス

パフォーマンス向上のため、以下のインデックスを作成することを推奨します：

```sql
-- 日付順の検索が頻繁なため
CREATE INDEX idx_transactions_date ON transactions(date DESC);

-- ユーザーごとの検索用
CREATE INDEX idx_transactions_user_email ON transactions(user_email);

-- 未返済の借りを探す用
CREATE INDEX idx_transactions_repaid ON transactions(repaid) WHERE repaid = false;
```

## 今後の拡張性

### 複数カップル対応
現在は二人専用の設計ですが、将来的に複数のカップルが使えるようにする場合：

1. `couples`テーブルを追加
   - couple_id
   - user1_email
   - user2_email

2. `transactions`テーブルに`couple_id`カラムを追加

3. RLSポリシーを変更して、自分のカップルのデータのみアクセス可能にする

### カテゴリ分類
支出のカテゴリ分けをしたい場合：

1. `categories`テーブルを追加
   - id
   - name (例: "食事", "交通費", "娯楽")
   - icon

2. `transactions`テーブルに`category_id`カラムを追加

## まとめ

この設計により：
- ✅ 二人が同じデータベースを共有しながら、それぞれの視点で閲覧可能
- ✅ 返済の追跡が可能（二重返済防止）
- ✅ 返済の取り消しが可能（元の状態に戻せる）
- ✅ セキュアなアクセス制御（ホワイトリスト + RLS）
- ✅ 拡張性を考慮した設計

が実現できています。
