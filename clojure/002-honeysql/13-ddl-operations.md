# 第13章: DDL操作（Data Definition Language）

## 概要

DDL（Data Definition Language）は、データベースの構造を定義・変更するためのSQL文です。この章では、HoneySQLを使ったテーブル作成、変更、削除、インデックス管理、制約の追加など、データベーススキーマの操作方法を学習します。

## 1. CREATE TABLE - テーブル作成

### 基本的なテーブル作成

#### ヘルパー関数アプローチ
```clojure
(-> (h/create-table :users)
    (h/with-columns [[:id :serial [:primary-key]]
                     [:email :varchar [255] [:not-null] [:unique]]
                     [:name :varchar [100] [:not-null]]
                     [:age :integer]
                     [:created_at :timestamptz [:default [:now]]]
                     [:updated_at :timestamptz]])
    sql/format)
```

#### キーワードマップアプローチ
```clojure
(sql/format {:create-table [:users]
             :with-columns [[:id :serial [:primary-key]]
                            [:email :varchar [255] [:not-null] [:unique]]
                            [:name :varchar [100] [:not-null]]
                            [:age :integer]
                            [:created_at :timestamptz [:default [:now]]]
                            [:updated_at :timestamptz]]})
```

#### 生成されるSQL
```sql
CREATE TABLE users (id SERIAL PRIMARY KEY, email VARCHAR(255) NOT NULL UNIQUE, name VARCHAR(100) NOT NULL, age INTEGER, created_at TIMESTAMPTZ DEFAULT NOW(), updated_at TIMESTAMPTZ)
```

### PostgreSQL固有データ型でのテーブル作成

#### ヘルパー関数アプローチ
```clojure
(-> (h/create-table :products)
    (h/with-columns [[:id :uuid [:primary-key] [:default [:gen_random_uuid]]]
                     [:name :text [:not-null]]
                     [:description :text]
                     [:price :numeric [10 2] [:not-null]]
                     [:tags :text []]  ; 配列型
                     [:metadata :jsonb]
                     [:search_vector :tsvector]
                     [:availability_period :daterange]
                     [:created_at :timestamptz [:default [:now]]]
                     [:updated_at :timestamptz [:default [:now]]]])
    sql/format)
```

#### 生成されるSQL
```sql
CREATE TABLE products (id UUID PRIMARY KEY DEFAULT GEN_RANDOM_UUID(), name TEXT NOT NULL, description TEXT, price NUMERIC(10, 2) NOT NULL, tags TEXT[], metadata JSONB, search_vector TSVECTOR, availability_period DATERANGE, created_at TIMESTAMPTZ DEFAULT NOW(), updated_at TIMESTAMPTZ DEFAULT NOW())
```

### 制約付きテーブル作成

#### ヘルパー関数アプローチ
```clojure
(-> (h/create-table :orders)
    (h/with-columns [[:id :bigserial [:primary-key]]
                     [:customer_id :bigint [:not-null]]
                     [:order_date :date [:not-null]]
                     [:status :varchar [20] [:not-null] [:default "pending"]]
                     [:total_amount :numeric [12 2] [:not-null]]
                     [:shipping_address :jsonb]])
    (h/check [:> :total_amount 0])
    (h/check [:in :status ["pending" "processing" "shipped" "delivered" "cancelled"]])
    (h/foreign-key [:customer_id] :users [:id] :on-delete :cascade)
    sql/format)
```

#### 生成されるSQL
```sql
CREATE TABLE orders (id BIGSERIAL PRIMARY KEY, customer_id BIGINT NOT NULL, order_date DATE NOT NULL, status VARCHAR(20) NOT NULL DEFAULT ?, total_amount NUMERIC(12, 2) NOT NULL, shipping_address JSONB, CHECK (total_amount > ?), CHECK (status IN (?, ?, ?, ?, ?)), FOREIGN KEY (customer_id) REFERENCES users (id) ON DELETE CASCADE)
```

## 2. CREATE INDEX - インデックス作成

### 基本的なインデックス

#### ヘルパー関数アプローチ
```clojure
(-> (h/create-index :idx_users_email)
    (h/on :users :email)
    sql/format)
```

#### キーワードマップアプローチ
```clojure
(sql/format {:create-index :idx_users_email
             :on [:users :email]})
```

#### 生成されるSQL
```sql
CREATE INDEX idx_users_email ON users (email)
```

### 複合インデックス

#### ヘルパー関数アプローチ
```clojure
(-> (h/create-index :idx_orders_customer_date)
    (h/on :orders [:customer_id :order_date])
    sql/format)
```

#### 生成されるSQL
```sql
CREATE INDEX idx_orders_customer_date ON orders (customer_id, order_date)
```

### 部分インデックス（条件付きインデックス）

#### ヘルパー関数アプローチ
```clojure
(-> (h/create-index :idx_active_users_email)
    (h/on :users :email)
    (h/where [:= :status "active"])
    sql/format)
```

#### 生成されるSQL
```sql
CREATE INDEX idx_active_users_email ON users (email) WHERE status = ?
```

### 式インデックス

#### ヘルパー関数アプローチ
```clojure
(-> (h/create-index :idx_users_lower_email)
    (h/on :users [[:lower :email]])
    sql/format)
```

#### 生成されるSQL
```sql
CREATE INDEX idx_users_lower_email ON users (LOWER(email))
```

### GINインデックス（JSON/配列用）

#### ヘルパー関数アプローチ
```clojure
(-> (h/create-index :idx_products_metadata [:gin])
    (h/on :products :metadata)
    sql/format)
```

#### 生成されるSQL
```sql
CREATE INDEX idx_products_metadata ON products USING GIN (metadata)
```

### 全文検索インデックス

#### ヘルパー関数アプローチ
```clojure
(-> (h/create-index :idx_articles_search [:gin])
    (h/on :articles [[:to_tsvector "english" [:|| :title " " :content]]])
    sql/format)
```

#### 生成されるSQL
```sql
CREATE INDEX idx_articles_search ON articles USING GIN (TO_TSVECTOR(?, title || ? || content))
```

## 3. ALTER TABLE - テーブル変更

### カラムの追加

#### ヘルパー関数アプローチ
```clojure
(-> (h/alter-table :users)
    (h/add-column :phone_number :varchar [20])
    sql/format)
```

#### 生成されるSQL
```sql
ALTER TABLE users ADD COLUMN phone_number VARCHAR(20)
```

### カラムの削除

#### ヘルパー関数アプローチ
```clojure
(-> (h/alter-table :users)
    (h/drop-column :phone_number)
    sql/format)
```

#### 生成されるSQL
```sql
ALTER TABLE users DROP COLUMN phone_number
```

### カラムの型変更

#### ヘルパー関数アプローチ
```clojure
(-> (h/alter-table :products)
    (h/alter-column :price [:type :numeric [15 2]])
    sql/format)
```

#### 生成されるSQL
```sql
ALTER TABLE products ALTER COLUMN price TYPE NUMERIC(15, 2)
```

### カラムのNOT NULL制約追加

#### ヘルパー関数アプローチ
```clojure
(-> (h/alter-table :users)
    (h/alter-column :email [:set [:not-null]])
    sql/format)
```

#### 生成されるSQL
```sql
ALTER TABLE users ALTER COLUMN email SET NOT NULL
```

### デフォルト値の設定

#### ヘルパー関数アプローチ
```clojure
(-> (h/alter-table :orders)
    (h/alter-column :status [:set-default "pending"])
    sql/format)
```

#### 生成されるSQL
```sql
ALTER TABLE orders ALTER COLUMN status SET DEFAULT ?
```

### 制約の追加

#### ヘルパー関数アプローチ
```clojure
(-> (h/alter-table :orders)
    (h/add-constraint :chk_positive_amount [:check [:> :total_amount 0]])
    sql/format)
```

#### 生成されるSQL
```sql
ALTER TABLE orders ADD CONSTRAINT chk_positive_amount CHECK (total_amount > ?)
```

### 外部キー制約の追加

#### ヘルパー関数アプローチ
```clojure
(-> (h/alter-table :order_items)
    (h/add-constraint :fk_order_items_product 
                      [:foreign-key [:product_id] 
                       :references :products [:id] 
                       :on-delete :restrict])
    sql/format)
```

#### 生成されるSQL
```sql
ALTER TABLE order_items ADD CONSTRAINT fk_order_items_product FOREIGN KEY (product_id) REFERENCES products (id) ON DELETE RESTRICT
```

## 4. DROP操作

### テーブルの削除

#### ヘルパー関数アプローチ
```clojure
(-> (h/drop-table :temp_data)
    sql/format)
```

#### 生成されるSQL
```sql
DROP TABLE temp_data
```

### 安全なテーブル削除（IF EXISTS）

#### ヘルパー関数アプローチ
```clojure
(-> (h/drop-table [:if-exists :temp_data])
    sql/format)
```

#### 生成されるSQL
```sql
DROP TABLE IF EXISTS temp_data
```

### カスケード削除

#### ヘルパー関数アプローチ
```clojure
(-> (h/drop-table :categories [:cascade])
    sql/format)
```

#### 生成されるSQL
```sql
DROP TABLE categories CASCADE
```

### インデックスの削除

#### ヘルパー関数アプローチ
```clojure
(-> (h/drop-index :idx_users_email)
    sql/format)
```

#### 生成されるSQL
```sql
DROP INDEX idx_users_email
```

## 5. パーティショニング

### レンジパーティション

#### ヘルパー関数アプローチ
```clojure
(-> (h/create-table :sales_2023)
    (h/with-columns [[:id :bigserial]
                     [:sale_date :date [:not-null]]
                     [:amount :numeric [10 2]]
                     [:customer_id :bigint]])
    (h/partition-by [:range :sale_date])
    sql/format)
```

#### 生成されるSQL
```sql
CREATE TABLE sales_2023 (id BIGSERIAL, sale_date DATE NOT NULL, amount NUMERIC(10, 2), customer_id BIGINT) PARTITION BY RANGE (sale_date)
```

### パーティションテーブルの作成

#### ヘルパー関数アプローチ
```clojure
(-> (h/create-table :sales_2023_q1)
    (h/partition-of :sales_2023)
    (h/for-values-from "2023-01-01" :to "2023-04-01")
    sql/format)
```

#### 生成されるSQL
```sql
CREATE TABLE sales_2023_q1 PARTITION OF sales_2023 FOR VALUES FROM (?) TO (?)
```

## 6. ビューの作成

### 基本的なビュー

#### ヘルパー関数アプローチ
```clojure
(-> (h/create-view :active_users)
    (h/select :id :name :email :created_at)
    (h/from :users)
    (h/where [:= :status "active"])
    sql/format)
```

#### 生成されるSQL
```sql
CREATE VIEW active_users AS SELECT id, name, email, created_at FROM users WHERE status = ?
```

### マテリアライズドビュー

#### ヘルパー関数アプローチ
```clojure
(-> (h/create-materialized-view :monthly_sales_summary)
    (h/select [[:date_trunc "month" :order_date] :month]
              [[:sum :total_amount] :total_sales]
              [[:count :*] :order_count]
              [[:count-distinct :customer_id] :unique_customers])
    (h/from :orders)
    (h/group-by [[:date_trunc "month" :order_date]])
    sql/format)
```

#### 生成されるSQL
```sql
CREATE MATERIALIZED VIEW monthly_sales_summary AS SELECT DATE_TRUNC(?, order_date) AS month, SUM(total_amount) AS total_sales, COUNT(*) AS order_count, COUNT(DISTINCT customer_id) AS unique_customers FROM orders GROUP BY DATE_TRUNC(?, order_date)
```

## 7. トリガーの作成

### 基本的なトリガー関数

#### PostgreSQL関数の作成
```clojure
(sql/format {:raw "CREATE OR REPLACE FUNCTION update_modified_time()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;"})
```

### トリガーの作成

#### ヘルパー関数アプローチ
```clojure
(-> (h/create-trigger :trigger_update_users_modified_time)
    (h/before [:update])
    (h/on :users)
    (h/for-each-row)
    (h/execute :update_modified_time)
    sql/format)
```

#### 生成されるSQL
```sql
CREATE TRIGGER trigger_update_users_modified_time BEFORE UPDATE ON users FOR EACH ROW EXECUTE update_modified_time()
```

## 8. 拡張機能の管理

### 拡張機能の有効化

#### ヘルパー関数アプローチ
```clojure
(-> (h/create-extension [:if-not-exists :uuid-ossp])
    sql/format)
```

#### 生成されるSQL
```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp"
```

### PostGIS拡張の有効化

#### ヘルパー関数アプローチ
```clojure
(-> (h/create-extension [:if-not-exists :postgis])
    sql/format)
```

## 9. スキーマ管理

### スキーマの作成

#### ヘルパー関数アプローチ
```clojure
(-> (h/create-schema :analytics)
    sql/format)
```

#### 生成されるSQL
```sql
CREATE SCHEMA analytics
```

### スキーマ付きテーブル作成

#### ヘルパー関数アプローチ
```clojure
(-> (h/create-table :analytics.user_metrics)
    (h/with-columns [[:user_id :bigint [:not-null]]
                     [:metric_date :date [:not-null]]
                     [:page_views :integer [:default 0]]
                     [:session_time :interval]])
    sql/format)
```

#### 生成されるSQL
```sql
CREATE SCHEMA analytics; CREATE TABLE analytics.user_metrics (user_id BIGINT NOT NULL, metric_date DATE NOT NULL, page_views INTEGER DEFAULT ?, session_time INTERVAL)
```

## 10. 実用的なDDL例

### 完全なEコマースデータベース設計

#### ヘルパー関数アプローチ
```clojure
(defn create-ecommerce-schema []
  [
    ;; ユーザーテーブル
    (-> (h/create-table :users)
        (h/with-columns [[:id :uuid [:primary-key] [:default [:gen_random_uuid]]]
                         [:email :varchar [255] [:not-null] [:unique]]
                         [:password_hash :varchar [255] [:not-null]]
                         [:first_name :varchar [100]]
                         [:last_name :varchar [100]]
                         [:phone :varchar [20]]
                         [:status :varchar [20] [:default "active"]]
                         [:created_at :timestamptz [:default [:now]]]
                         [:updated_at :timestamptz [:default [:now]]]])
        (h/check [:in :status ["active" "inactive" "suspended"]])
        sql/format)
    
    ;; 商品テーブル
    (-> (h/create-table :products)
        (h/with-columns [[:id :uuid [:primary-key] [:default [:gen_random_uuid]]]
                         [:sku :varchar [100] [:not-null] [:unique]]
                         [:name :varchar [255] [:not-null]]
                         [:description :text]
                         [:price :numeric [10 2] [:not-null]]
                         [:cost :numeric [10 2]]
                         [:stock :integer [:default 0]]
                         [:category_id :uuid]
                         [:tags :text []]
                         [:metadata :jsonb]
                         [:is_active :boolean [:default true]]
                         [:created_at :timestamptz [:default [:now]]]
                         [:updated_at :timestamptz [:default [:now]]]])
        (h/check [:>= :price 0])
        (h/check [:>= :stock 0])
        sql/format)
    
    ;; 注文テーブル
    (-> (h/create-table :orders)
        (h/with-columns [[:id :uuid [:primary-key] [:default [:gen_random_uuid]]]
                         [:user_id :uuid [:not-null]]
                         [:order_number :varchar [50] [:not-null] [:unique]]
                         [:status :varchar [20] [:default "pending"]]
                         [:subtotal :numeric [12 2] [:not-null]]
                         [:tax_amount :numeric [12 2] [:default 0]]
                         [:shipping_amount :numeric [12 2] [:default 0]]
                         [:total_amount :numeric [12 2] [:not-null]]
                         [:shipping_address :jsonb]
                         [:billing_address :jsonb]
                         [:notes :text]
                         [:created_at :timestamptz [:default [:now]]]
                         [:updated_at :timestamptz [:default [:now]]]])
        (h/foreign-key [:user_id] :users [:id] :on-delete :cascade)
        (h/check [:> :total_amount 0])
        (h/check [:in :status ["pending" "processing" "shipped" "delivered" "cancelled"]])
        sql/format)
    
    ;; インデックス作成
    (-> (h/create-index :idx_products_category)
        (h/on :products :category_id)
        sql/format)
    
    (-> (h/create-index :idx_products_tags [:gin])
        (h/on :products :tags)
        sql/format)
    
    (-> (h/create-index :idx_orders_user_created)
        (h/on :orders [:user_id :created_at])
        sql/format)
  ])
```

## 演習問題

1. ブログシステムのための`posts`、`categories`、`comments`テーブルを作成し、適切な制約とインデックスを設定してください。

2. 時系列データを格納するための月次パーティションテーブルを作成してください。

3. ユーザーの活動ログを効率的に検索するためのGINインデックスとJSON型カラムを持つテーブルを設計してください。

4. 在庫管理システムのためのトリガーを作成し、商品の在庫が0になったときに自動的にステータスを更新するようにしてください。

## 次のステップ

最後に、これまで学習したCRUD操作を既存の教材に統合して、より実践的な例として完成させます。