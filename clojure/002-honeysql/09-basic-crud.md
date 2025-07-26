# 第9章: 基本的なCRUD操作

## 概要

CRUD（Create, Read, Update, Delete）は、データベースの基本的な4つの操作です。これまでの章では主にR（Read）であるSELECT文を学習してきました。この章では、CREATE（INSERT）、UPDATE、DELETEの基本的な使い方を学びます。

## 1. CREATE - データの挿入（INSERT）

### 基本的なINSERT文

#### ヘルパー関数アプローチ
```clojure
(-> (h/insert-into :users)
    (h/values [{:name "田中太郎" :email "tanaka@example.com" :age 25}])
    sql/format)
```

#### キーワードマップアプローチ
```clojure
(sql/format {:insert-into :users
             :values [{:name "田中太郎" :email "tanaka@example.com" :age 25}]})
```

#### 生成されるSQL
```sql
INSERT INTO users (name, email, age) VALUES (?, ?, ?)
```

### 複数行の一括挿入

#### ヘルパー関数アプローチ
```clojure
(-> (h/insert-into :users)
    (h/values [{:name "田中太郎" :email "tanaka@example.com" :age 25}
               {:name "佐藤花子" :email "sato@example.com" :age 30}
               {:name "鈴木次郎" :email "suzuki@example.com" :age 28}])
    sql/format)
```

#### キーワードマップアプローチ
```clojure
(sql/format {:insert-into :users
             :values [{:name "田中太郎" :email "tanaka@example.com" :age 25}
                      {:name "佐藤花子" :email "sato@example.com" :age 30}
                      {:name "鈴木次郎" :email "suzuki@example.com" :age 28}]})
```

#### 生成されるSQL
```sql
INSERT INTO users (name, email, age) VALUES (?, ?, ?), (?, ?, ?), (?, ?, ?)
```

### カラムを明示的に指定

#### ヘルパー関数アプローチ
```clojure
(-> (h/insert-into :users [:name :email])
    (h/values [["田中太郎" "tanaka@example.com"]
               ["佐藤花子" "sato@example.com"]])
    sql/format)
```

#### 生成されるSQL
```sql
INSERT INTO users (name, email) VALUES (?, ?), (?, ?)
```

### デフォルト値の使用

#### ヘルパー関数アプローチ
```clojure
(-> (h/insert-into :users)
    (h/values [{:name "山田太郎" :email "yamada@example.com" :created_at [:default]}])
    sql/format)
```

#### 生成されるSQL
```sql
INSERT INTO users (name, email, created_at) VALUES (?, ?, DEFAULT)
```

### RETURNING句（挿入されたデータを取得）

#### ヘルパー関数アプローチ
```clojure
(-> (h/insert-into :users)
    (h/values [{:name "新規ユーザー" :email "new@example.com"}])
    (h/returning :id :name :created_at)
    sql/format)
```

#### キーワードマップアプローチ
```clojure
(sql/format {:insert-into :users
             :values [{:name "新規ユーザー" :email "new@example.com"}]
             :returning [:id :name :created_at]})
```

#### 生成されるSQL
```sql
INSERT INTO users (name, email) VALUES (?, ?) RETURNING id, name, created_at
```

## 2. UPDATE - データの更新

### 基本的なUPDATE文

#### ヘルパー関数アプローチ
```clojure
(-> (h/update :users)
    (h/set {:name "田中太郎（更新）" :age 26})
    (h/where [:= :id 1])
    sql/format)
```

#### キーワードマップアプローチ
```clojure
(sql/format {:update :users
             :set {:name "田中太郎（更新）" :age 26}
             :where [:= :id 1]})
```

#### 生成されるSQL
```sql
UPDATE users SET name = ?, age = ? WHERE id = ?
```

### 計算による更新

#### ヘルパー関数アプローチ
```clojure
(-> (h/update :products)
    (h/set {:price [:* :price 1.1]  ; 10%値上げ
            :updated_at [:now]})
    (h/where [:= :category "electronics"])
    sql/format)
```

#### 生成されるSQL
```sql
UPDATE products SET price = price * ?, updated_at = NOW() WHERE category = ?
```

### 条件付きUPDATE（CASE文）

#### ヘルパー関数アプローチ
```clojure
(-> (h/update :employees)
    (h/set {:bonus [:case
                    [:>= :years_of_service 10] 100000
                    [:>= :years_of_service 5] 50000
                    :else 25000]
            :updated_at [:now]})
    (h/where [:= :department "sales"])
    sql/format)
```

#### 生成されるSQL
```sql
UPDATE employees SET bonus = CASE WHEN years_of_service >= ? THEN ? WHEN years_of_service >= ? THEN ? ELSE ? END, updated_at = NOW() WHERE department = ?
```

### JOINを使ったUPDATE

#### ヘルパー関数アプローチ
```clojure
(-> (h/update :orders)
    (h/set {:total_amount :oi.calculated_total})
    (h/from [[:select :order_id [[:sum [:* :quantity :unit_price]] :calculated_total]]
             [:from :order_items]
             [:group-by :order_id]] :oi)
    (h/where [:= :orders.id :oi.order_id])
    sql/format)
```

#### 生成されるSQL
```sql
UPDATE orders SET total_amount = oi.calculated_total FROM (SELECT order_id, SUM(quantity * unit_price) AS calculated_total FROM order_items GROUP BY order_id) AS oi WHERE orders.id = oi.order_id
```

### RETURNING句付きUPDATE

#### ヘルパー関数アプローチ
```clojure
(-> (h/update :users)
    (h/set {:last_login [:now] :login_count [:+ :login_count 1]})
    (h/where [:= :email "user@example.com"])
    (h/returning :id :name :last_login :login_count)
    sql/format)
```

#### 生成されるSQL
```sql
UPDATE users SET last_login = NOW(), login_count = login_count + ? WHERE email = ? RETURNING id, name, last_login, login_count
```

## 3. DELETE - データの削除

### 基本的なDELETE文

#### ヘルパー関数アプローチ
```clojure
(-> (h/delete-from :users)
    (h/where [:= :id 1])
    sql/format)
```

#### キーワードマップアプローチ
```clojure
(sql/format {:delete-from :users
             :where [:= :id 1]})
```

#### 生成されるSQL
```sql
DELETE FROM users WHERE id = ?
```

### 複数条件でのDELETE

#### ヘルパー関数アプローチ
```clojure
(-> (h/delete-from :sessions)
    (h/where [:and
              [:< :expires_at [:now]]
              [:= :active false]])
    sql/format)
```

#### 生成されるSQL
```sql
DELETE FROM sessions WHERE (expires_at < NOW()) AND (active = ?)
```

### JOINを使ったDELETE

#### ヘルパー関数アプローチ
```clojure
(-> (h/delete-from :order_items)
    (h/using :orders)
    (h/where [:and
              [:= :order_items.order_id :orders.id]
              [:= :orders.status "cancelled"]])
    sql/format)
```

#### 生成されるSQL
```sql
DELETE FROM order_items USING orders WHERE (order_items.order_id = orders.id) AND (orders.status = ?)
```

### サブクエリを使ったDELETE

#### ヘルパー関数アプローチ
```clojure
(-> (h/delete-from :old_logs)
    (h/where [:in :user_id [:select :id
                            [:from :inactive_users]
                            [:where [:< :last_login "2022-01-01"]]]])
    sql/format)
```

#### 生成されるSQL
```sql
DELETE FROM old_logs WHERE user_id IN (SELECT id FROM inactive_users WHERE last_login < ?)
```

### RETURNING句付きDELETE

#### ヘルパー関数アプローチ
```clojure
(-> (h/delete-from :users)
    (h/where [:= :status "inactive"])
    (h/returning :id :name :email)
    sql/format)
```

#### 生成されるSQL
```sql
DELETE FROM users WHERE status = ? RETURNING id, name, email
```

## 4. 実践的なCRUDパターン

### ユーザー登録の完全な例

#### ヘルパー関数アプローチ
```clojure
;; 新規ユーザー登録
(-> (h/insert-into :users)
    (h/values [{:name "新規ユーザー"
                :email "newuser@example.com"
                :password_hash "hashed_password"
                :created_at [:now]
                :updated_at [:now]
                :status "active"}])
    (h/on-conflict :email)
    (h/do-nothing)
    (h/returning :id :name :email :created_at)
    sql/format)
```

#### 生成されるSQL
```sql
INSERT INTO users (name, email, password_hash, created_at, updated_at, status) VALUES (?, ?, ?, NOW(), NOW(), ?) ON CONFLICT (email) DO NOTHING RETURNING id, name, email, created_at
```

### 注文処理の例

#### 注文作成
```clojure
(-> (h/insert-into :orders)
    (h/values [{:customer_id 123
                :order_date [:now]
                :status "pending"
                :total_amount 0}])
    (h/returning :id)
    sql/format)
```

#### 注文明細追加
```clojure
(-> (h/insert-into :order_items)
    (h/values [{:order_id 456 :product_id 789 :quantity 2 :unit_price 99.99}
               {:order_id 456 :product_id 790 :quantity 1 :unit_price 149.99}])
    sql/format)
```

#### 注文合計金額更新
```clojure
(-> (h/update :orders)
    (h/set {:total_amount [:select [[:sum [:* :quantity :unit_price]]]
                           [:from :order_items]
                           [:where [:= :order_id :orders.id]]]})
    (h/where [:= :id 456])
    sql/format)
```

### データクリーンアップの例

#### 古いデータの削除
```clojure
(-> (h/delete-from :audit_logs)
    (h/where [:< :created_at [:- [:now] [:interval "90 days"]]])
    sql/format)
```

#### 重複データの削除
```clojure
(-> (h/delete-from :duplicate_records)
    (h/where [:in :id [:select :id
                       [:from :duplicate_records]
                       [:except [:select [[:min :id]]
                                 [:from :duplicate_records]
                                 [:group-by :email]]]]])
    sql/format)
```

## 5. トランザクションとCRUD

### 一連のCRUD操作
```clojure
;; 通常は1つのトランザクション内で実行
[
  ;; 1. 商品在庫確認と更新
  (-> (h/update :products)
      (h/set {:stock [:- :stock 1]})
      (h/where [:and [:= :id 123] [:> :stock 0]])
      (h/returning :stock)
      sql/format)
  
  ;; 2. 注文作成
  (-> (h/insert-into :orders)
      (h/values [{:customer_id 456 :product_id 123 :quantity 1}])
      (h/returning :id)
      sql/format)
  
  ;; 3. 在庫ログ記録
  (-> (h/insert-into :stock_movements)
      (h/values [{:product_id 123 :change -1 :reason "sale"}])
      sql/format)
]
```

## 演習問題

1. `products`テーブルに新商品を3件一括で挿入し、挿入されたレコードのIDと名前を取得するクエリを作成してください。

2. `users`テーブルで、最終ログインが30日以上前のユーザーのステータスを"inactive"に更新するクエリを作成してください。

3. `orders`テーブルから、ステータスが"cancelled"で、かつ1年以上前の注文を削除するクエリを作成してください。

4. ユーザーの注文履歴を更新する際、`user_stats`テーブルの`total_orders`と`total_amount`を同時に更新するクエリを作成してください。

## 次のステップ

次の章では、より高度なINSERT操作（サブクエリからの挿入、Upsert、一括処理など）について詳しく学習します。