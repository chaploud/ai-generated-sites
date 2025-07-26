# 第8章: PostgreSQL固有の高度な機能

## 概要

PostgreSQLは豊富な拡張機能を持つ高機能データベースです。HoneySQLはこれらの機能を完全にサポートしており、JSON/JSONB操作、配列操作、ウィンドウ関数、Upsert、全文検索など、PostgreSQL固有の強力な機能を活用できます。

## 1. JSON/JSONB操作

### 基本的なJSON操作

#### JSONデータの取得

##### ヘルパー関数アプローチ
```clojure
(-> (h/select :id 
              [:-> :metadata "name"] :product_name
              [:->> :metadata "tags" 0] :first_tag
              [:# :metadata "features"] :feature_count)
    (h/from :products)
    (h/where [:@> :metadata {:category "electronics"}])
    sql/format)
```

##### キーワードマップアプローチ
```clojure
(sql/format {:select [:id 
                      [:-> :metadata "name"] :product_name
                      [:->> :metadata "tags" 0] :first_tag
                      [:# :metadata "features"] :feature_count]
             :from   [:products]
             :where  [:@> :metadata {:category "electronics"}]})
```

##### 生成されるSQL
```sql
SELECT id, metadata -> ? AS product_name, metadata ->> ? AS first_tag, metadata # ? AS feature_count FROM products WHERE metadata @> ?
```

### JSONパス操作

#### ヘルパー関数アプローチ
```clojure
(-> (h/select :id
              [:# :user_data "profile,address,zipcode"] :zipcode
              [:# :user_data "orders,0,total"] :first_order_total)
    (h/from :users)
    (h/where [:is-not [:# :user_data "profile,address"] nil])
    sql/format)
```

##### 生成されるSQL
```sql
SELECT id, user_data # ? AS zipcode, user_data # ? AS first_order_total FROM users WHERE user_data # ? IS NOT NULL
```

### JSON集約関数

#### ヘルパー関数アプローチ
```clojure
(-> (h/select :category
              [[:json_agg [:json_build_object 
                           "id" :id
                           "name" :name
                           "price" :price]] :products])
    (h/from :products)
    (h/group-by :category)
    sql/format)
```

##### 生成されるSQL
```sql
SELECT category, JSON_AGG(JSON_BUILD_OBJECT(?, id, ?, name, ?, price)) AS products FROM products GROUP BY category
```

### JSONB更新操作

#### ヘルパー関数アプローチ
```clojure
;; JSONBフィールドの一部を更新
(-> (h/update :users)
    (h/set {:profile [:jsonb_set :profile "{address,city}" "\"Tokyo\"" true]})
    (h/where [:= :id 123])
    sql/format)
```

##### 生成されるSQL
```sql
UPDATE users SET profile = JSONB_SET(profile, ?, ?, ?) WHERE id = ?
```

## 2. 配列操作

### 配列の基本操作

#### ヘルパー関数アプローチ
```clojure
(-> (h/select :id :name
              [[:array_length :tags 1] :tag_count]
              [[:array_position :tags "postgresql"] :pg_position]
              [:tags :|| ["new_tag"]] :updated_tags)
    (h/from :articles)
    (h/where [:@> :tags ["database" "sql"]])
    sql/format)
```

##### キーワードマップアプローチ
```clojure
(sql/format {:select [:id :name
                      [[:array_length :tags 1] :tag_count]
                      [[:array_position :tags "postgresql"] :pg_position]
                      [:|| :tags ["new_tag"]] :updated_tags]
             :from   [:articles]
             :where  [:@> :tags ["database" "sql"]]})
```

##### 生成されるSQL
```sql
SELECT id, name, ARRAY_LENGTH(tags, ?) AS tag_count, ARRAY_POSITION(tags, ?) AS pg_position, tags || ? AS updated_tags FROM articles WHERE tags @> ?
```

### 配列の展開（UNNEST）

#### ヘルパー関数アプローチ
```clojure
(-> (h/select :a.id :a.title :tag.tag_name)
    (h/from [:articles :a])
    (h/cross-join [[:unnest :a.tags] :tag :tag_name])
    (h/where [:like :tag.tag_name "%data%"])
    sql/format)
```

##### 生成されるSQL
```sql
SELECT a.id, a.title, tag.tag_name FROM articles AS a CROSS JOIN UNNEST(a.tags) AS tag(tag_name) WHERE tag.tag_name LIKE ?
```

## 3. ウィンドウ関数

### 基本的なウィンドウ関数

#### ヘルパー関数アプローチ
```clojure
(-> (h/select :name :department :salary
              [[:rank] :dept_rank {:over [:partition-by :department :order-by [[:salary :desc]]]}]
              [[:dense_rank] :dense_rank {:over [:partition-by :department :order-by [[:salary :desc]]]}]
              [[:row_number] :row_num {:over [:partition-by :department :order-by [[:salary :desc]]]}])
    (h/from :employees)
    sql/format)
```

##### キーワードマップアプローチ
```clojure
(sql/format {:select [:name :department :salary
                      [[:rank] :dept_rank {:over [:partition-by :department :order-by [[:salary :desc]]]}]
                      [[:dense_rank] :dense_rank {:over [:partition-by :department :order-by [[:salary :desc]]]}]
                      [[:row_number] :row_num {:over [:partition-by :department :order-by [[:salary :desc]]]}]]
             :from   [:employees]})
```

##### 生成されるSQL
```sql
SELECT name, department, salary, RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank, DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dense_rank, ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS row_num FROM employees
```

### 分析関数（LAG/LEAD）

#### ヘルパー関数アプローチ
```clojure
(-> (h/select :order_date :amount
              [[:lag :amount 1] :prev_amount {:over [:order-by :order_date]}]
              [[:lead :amount 1] :next_amount {:over [:order-by :order_date]}]
              [[:- :amount [:lag :amount 1 0]] :amount_change {:over [:order-by :order_date]}])
    (h/from :daily_sales)
    (h/order-by :order_date)
    sql/format)
```

##### 生成されるSQL
```sql
SELECT order_date, amount, LAG(amount, ?) OVER (ORDER BY order_date) AS prev_amount, LEAD(amount, ?) OVER (ORDER BY order_date) AS next_amount, amount - LAG(amount, ?, ?) OVER (ORDER BY order_date) AS amount_change FROM daily_sales ORDER BY order_date
```

### 累積計算とフレーム指定

#### ヘルパー関数アプローチ
```clojure
(-> (h/select :order_date :amount
              [[:sum :amount] :running_total {:over [:order-by :order_date :rows [:unbounded-preceding :current-row]]}]
              [[:avg :amount] :moving_avg {:over [:order-by :order_date :rows [[:preceding 6] :current-row]]}]
              [[:count :*] :order_count {:over [:order-by :order_date]}])
    (h/from :daily_sales)
    (h/order-by :order_date)
    sql/format)
```

##### 生成されるSQL
```sql
SELECT order_date, amount, SUM(amount) OVER (ORDER BY order_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total, AVG(amount) OVER (ORDER BY order_date ROWS BETWEEN ? PRECEDING AND CURRENT ROW) AS moving_avg, COUNT(*) OVER (ORDER BY order_date) AS order_count FROM daily_sales ORDER BY order_date
```

## 4. Upsert操作（ON CONFLICT）

### 基本的なUpsert

#### ヘルパー関数アプローチ
```clojure
(-> (h/insert-into :users)
    (h/values [{:email "john@example.com" :name "John Doe" :last_login (sql/call :now)}])
    (h/on-conflict :email)
    (h/do-update-set :name :last_login)
    (h/returning :*)
    sql/format)
```

##### キーワードマップアプローチ
```clojure
(sql/format {:insert-into :users
             :values [{:email "john@example.com" :name "John Doe" :last_login [:now]}]
             :on-conflict [:email]
             :do-update-set [:name :last_login]
             :returning [:*]})
```

##### 生成されるSQL
```sql
INSERT INTO users (email, name, last_login) VALUES (?, ?, NOW()) ON CONFLICT (email) DO UPDATE SET name = EXCLUDED.name, last_login = EXCLUDED.last_login RETURNING *
```

### 条件付きUpsert

#### ヘルパー関数アプローチ
```clojure
(-> (h/insert-into :products)
    (h/values [{:sku "ABC123" :name "Product A" :price 99.99 :updated_at (sql/call :now)}])
    (h/on-conflict :sku)
    (h/do-update-set :name :price :updated_at)
    (h/where [:< :products.updated_at :excluded.updated_at])
    sql/format)
```

##### 生成されるSQL
```sql
INSERT INTO products (sku, name, price, updated_at) VALUES (?, ?, ?, NOW()) ON CONFLICT (sku) DO UPDATE SET name = EXCLUDED.name, price = EXCLUDED.price, updated_at = EXCLUDED.updated_at WHERE products.updated_at < EXCLUDED.updated_at
```

## 5. 全文検索（Full Text Search）

### 基本的な全文検索

#### ヘルパー関数アプローチ
```clojure
(-> (h/select :id :title :content
              [[:ts_rank [:to_tsvector "english" :content] [:plainto_tsquery "english" "database tutorial"]] :rank])
    (h/from :articles)
    (h/where [:@@ [:to_tsvector "english" [:|| :title " " :content]] [:plainto_tsquery "english" "database tutorial"]])
    (h/order-by [[:ts_rank [:to_tsvector "english" :content] [:plainto_tsquery "english" "database tutorial"]] :desc])
    sql/format)
```

##### 生成されるSQL
```sql
SELECT id, title, content, TS_RANK(TO_TSVECTOR(?, content), PLAINTO_TSQUERY(?, ?)) AS rank FROM articles WHERE TO_TSVECTOR(?, title || ? || content) @@ PLAINTO_TSQUERY(?, ?) ORDER BY TS_RANK(TO_TSVECTOR(?, content), PLAINTO_TSQUERY(?, ?)) DESC
```

### 検索結果のハイライト

#### ヘルパー関数アプローチ
```clojure
(-> (h/select :id :title
              [[:ts_headline "english" :content [:plainto_tsquery "english" "database"] "MaxWords=20, MinWords=5"] :snippet])
    (h/from :articles)
    (h/where [:@@ [:to_tsvector "english" :content] [:plainto_tsquery "english" "database"]])
    sql/format)
```

##### 生成されるSQL
```sql
SELECT id, title, TS_HEADLINE(?, content, PLAINTO_TSQUERY(?, ?), ?) AS snippet FROM articles WHERE TO_TSVECTOR(?, content) @@ PLAINTO_TSQUERY(?, ?)
```

## 6. 範囲型（Range Types）

### 日付範囲での操作

#### ヘルパー関数アプローチ
```clojure
(-> (h/select :id :name :active_period
              [[:lower :active_period] :start_date]
              [[:upper :active_period] :end_date]
              [[:overlaps :active_period [:daterange "2023-01-01" "2023-12-31"]] :was_active_in_2023])
    (h/from :campaigns)
    (h/where [:&& :active_period [:daterange "2023-06-01" "2023-08-31"]])
    sql/format)
```

##### 生成されるSQL
```sql
SELECT id, name, active_period, LOWER(active_period) AS start_date, UPPER(active_period) AS end_date, OVERLAPS(active_period, DATERANGE(?, ?)) AS was_active_in_2023 FROM campaigns WHERE active_period && DATERANGE(?, ?)
```

## 7. PostGIS（地理空間データ）

### 基本的なジオメトリ操作

#### ヘルパー関数アプローチ
```clojure
(-> (h/select :id :name
              [[:st_distance :location [:st_point -74.006 40.7128]] :distance_from_nyc]
              [[:st_within :location [:st_buffer [:st_point -74.006 40.7128] 0.1]] :near_nyc])
    (h/from :stores)
    (h/where [:<= [:st_distance :location [:st_point -74.006 40.7128]] 0.1])
    (h/order-by [:st_distance :location [:st_point -74.006 40.7128]])
    sql/format)
```

##### 生成されるSQL
```sql
SELECT id, name, ST_DISTANCE(location, ST_POINT(?, ?)) AS distance_from_nyc, ST_WITHIN(location, ST_BUFFER(ST_POINT(?, ?), ?)) AS near_nyc FROM stores WHERE ST_DISTANCE(location, ST_POINT(?, ?)) <= ? ORDER BY ST_DISTANCE(location, ST_POINT(?, ?))
```

## 8. 高度な集計関数

### パーセンタイルと統計関数

#### ヘルパー関数アプローチ
```clojure
(-> (h/select :department
              [[:percentile_cont 0.5] :median_salary {:within-group [:order-by :salary]}]
              [[:percentile_cont 0.25] :q1_salary {:within-group [:order-by :salary]}]
              [[:percentile_cont 0.75] :q3_salary {:within-group [:order-by :salary]}]
              [[:mode] :mode_salary {:within-group [:order-by :salary]}])
    (h/from :employees)
    (h/group-by :department)
    sql/format)
```

##### 生成されるSQL
```sql
SELECT department, PERCENTILE_CONT(?) WITHIN GROUP (ORDER BY salary) AS median_salary, PERCENTILE_CONT(?) WITHIN GROUP (ORDER BY salary) AS q1_salary, PERCENTILE_CONT(?) WITHIN GROUP (ORDER BY salary) AS q3_salary, MODE() WITHIN GROUP (ORDER BY salary) AS mode_salary FROM employees GROUP BY department
```

## 9. 条件付きインデックスとパーシャルインデックス

### パフォーマンス最適化のためのクエリ

#### ヘルパー関数アプローチ
```clojure
;; アクティブなユーザーのみを効率的に検索
(-> (h/select :id :name :email)
    (h/from :users)
    (h/where [:and 
              [:= :status "active"]
              [:> :last_login "2023-01-01"]])
    (h/order-by [:last_login :desc])
    sql/format)
```

### 部分一意制約の活用

#### ヘルパー関数アプローチ
```clojure
;; アクティブなユーザーのみでメール重複チェック
(-> (h/insert-into :users)
    (h/values [{:email "test@example.com" :name "Test User" :status "active"}])
    (h/on-conflict [:email] [:where [:= :status "active"]])
    (h/do-nothing)
    sql/format)
```

## 10. 実用的な例：複合クエリ

### ダッシュボード用データ集約

#### ヘルパー関数アプローチ
```clojure
(-> (h/with [:monthly_metrics (-> (h/select [[:extract "YEAR" :created_at] :year]
                                            [[:extract "MONTH" :created_at] :month]
                                            [[:count :*] :order_count]
                                            [[:sum :amount] :revenue]
                                            [[:count-distinct :customer_id] :unique_customers]
                                            [[:avg :amount] :avg_order_value])
                                  (h/from :orders)
                                  (h/where [:>= :created_at "2023-01-01"])
                                  (h/group-by [[:extract "YEAR" :created_at]]
                                              [[:extract "MONTH" :created_at]]))
             :with_growth (-> (h/select :year :month :order_count :revenue :unique_customers :avg_order_value
                                        [[:lag :revenue 1] :prev_month_revenue {:over [:order-by :year :month]}]
                                        [[:lag :order_count 1] :prev_month_orders {:over [:order-by :year :month]}])
                              (h/from :monthly_metrics))]
    (h/select :year :month :order_count :revenue :unique_customers :avg_order_value
              [[:case 
                [:is-not :prev_month_revenue nil]
                [[:* [[:/ [:- :revenue :prev_month_revenue] :prev_month_revenue]] 100]]
                :else nil] :revenue_growth_pct]
              [[:case 
                [:is-not :prev_month_orders nil]
                [[:* [[:/ [:- :order_count :prev_month_orders] :prev_month_orders]] 100]]
                :else nil] :order_growth_pct])
    (h/from :with_growth)
    (h/order-by :year :month)
    sql/format)
```

## 演習問題

1. `products`テーブルのJSONBカラム`specifications`から、特定の仕様値を持つ商品を検索し、その仕様をJSONとして整形して返すクエリを作成してください。

2. ウィンドウ関数を使って、各部署の給与ランキングと部署内での給与パーセンタイルを計算するクエリを作成してください。

3. Upsert操作を使って、商品マスタに新商品を追加し、既存商品の場合は価格のみを更新するクエリを作成してください。

4. 全文検索を使って、記事の内容から特定のキーワードを検索し、関連度でランキングして結果をハイライト表示するクエリを作成してください。

## まとめ

このPostgreSQL固有機能の章では、以下の高度な機能を学習しました：

- **JSON/JSONB操作**: データの柔軟な格納と検索
- **配列操作**: 複数値の効率的な処理
- **ウィンドウ関数**: 高度な分析クエリ
- **Upsert操作**: データの安全な更新・挿入
- **全文検索**: 高性能なテキスト検索
- **範囲型**: 期間や範囲データの処理
- **PostGIS**: 地理空間データの操作
- **統計関数**: 高度な統計計算

これらの機能を組み合わせることで、PostgreSQLの真の力を発揮した高性能なアプリケーションを構築できます。HoneySQLはこれらすべての機能を直感的で読みやすい方法で利用することを可能にします。