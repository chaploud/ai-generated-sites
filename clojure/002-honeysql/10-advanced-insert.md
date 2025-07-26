# 第10章: 高度なINSERT操作

## 概要

この章では、基本的なINSERT文を超えた高度な挿入操作を学習します。サブクエリからの挿入、Upsert（INSERT ON CONFLICT）、条件付き挿入、一括処理、CTEとの組み合わせなど、実際のアプリケーションでよく使われるパターンを網羅します。

## 1. サブクエリからのINSERT（INSERT INTO ... SELECT）

### 基本的なSELECTからの挿入

#### ヘルパー関数アプローチ
```clojure
(-> (h/insert-into :user_backups [:id :name :email :created_at])
    (h/select :id :name :email :created_at)
    (h/from :users)
    (h/where [:< :last_login "2022-01-01"])
    sql/format)
```

#### キーワードマップアプローチ
```clojure
(sql/format {:insert-into [:user_backups [:id :name :email :created_at]]
             :select [:id :name :email :created_at]
             :from   [:users]
             :where  [:< :last_login "2022-01-01"]})
```

#### 生成されるSQL
```sql
INSERT INTO user_backups (id, name, email, created_at) SELECT id, name, email, created_at FROM users WHERE last_login < ?
```

### 計算値を含む挿入

#### ヘルパー関数アプローチ
```clojure
(-> (h/insert-into :monthly_reports [:month :user_count :total_orders :avg_order_value])
    (h/select [[:date_trunc "month" :created_at] :month]
              [[:count-distinct :user_id] :user_count]
              [[:count :*] :total_orders]
              [[:avg :amount] :avg_order_value])
    (h/from :orders)
    (h/where [:>= :created_at "2023-01-01"])
    (h/group-by [[:date_trunc "month" :created_at]])
    sql/format)
```

#### 生成されるSQL
```sql
INSERT INTO monthly_reports (month, user_count, total_orders, avg_order_value) SELECT DATE_TRUNC(?, created_at) AS month, COUNT(DISTINCT user_id) AS user_count, COUNT(*) AS total_orders, AVG(amount) AS avg_order_value FROM orders WHERE created_at >= ? GROUP BY DATE_TRUNC(?, created_at)
```

### JOINを使った複雑な挿入

#### ヘルパー関数アプローチ
```clojure
(-> (h/insert-into :user_profiles [:user_id :full_name :total_spent :order_count])
    (h/select :u.id 
              [[:concat :u.first_name " " :u.last_name] :full_name]
              [[:coalesce [:sum :o.amount] 0] :total_spent]
              [[:count :o.id] :order_count])
    (h/from [:users :u])
    (h/left-join [:orders :o] [:= :u.id :o.user_id])
    (h/where [:= :u.status "active"])
    (h/group-by :u.id :u.first_name :u.last_name)
    sql/format)
```

#### 生成されるSQL
```sql
INSERT INTO user_profiles (user_id, full_name, total_spent, order_count) SELECT u.id, CONCAT(u.first_name, ?, u.last_name) AS full_name, COALESCE(SUM(o.amount), ?) AS total_spent, COUNT(o.id) AS order_count FROM users AS u LEFT JOIN orders AS o ON u.id = o.user_id WHERE u.status = ? GROUP BY u.id, u.first_name, u.last_name
```

## 2. Upsert操作（INSERT ON CONFLICT）

### 基本的なUpsert

#### ヘルパー関数アプローチ
```clojure
(-> (h/insert-into :user_stats)
    (h/values [{:user_id 123 :login_count 1 :last_login [:now]}])
    (h/on-conflict :user_id)
    (h/do-update-set {:login_count [:+ :user_stats.login_count 1]
                      :last_login  [:now]})
    (h/returning :*)
    sql/format)
```

#### キーワードマップアプローチ
```clojure
(sql/format {:insert-into :user_stats
             :values [{:user_id 123 :login_count 1 :last_login [:now]}]
             :on-conflict [:user_id]
             :do-update-set {:login_count [:+ :user_stats.login_count 1]
                             :last_login  [:now]}
             :returning [:*]})
```

#### 生成されるSQL
```sql
INSERT INTO user_stats (user_id, login_count, last_login) VALUES (?, ?, NOW()) ON CONFLICT (user_id) DO UPDATE SET login_count = user_stats.login_count + ?, last_login = NOW() RETURNING *
```

### 条件付きUpsert

#### ヘルパー関数アプローチ
```clojure
(-> (h/insert-into :product_prices)
    (h/values [{:product_id 456 :price 99.99 :effective_date [:now]}])
    (h/on-conflict :product_id)
    (h/do-update-set {:price :excluded.price
                      :effective_date :excluded.effective_date})
    (h/where [:> :excluded.effective_date :product_prices.effective_date])
    sql/format)
```

#### 生成されるSQL
```sql
INSERT INTO product_prices (product_id, price, effective_date) VALUES (?, ?, NOW()) ON CONFLICT (product_id) DO UPDATE SET price = EXCLUDED.price, effective_date = EXCLUDED.effective_date WHERE EXCLUDED.effective_date > product_prices.effective_date
```

### 複合キーでのUpsert

#### ヘルパー関数アプローチ
```clojure
(-> (h/insert-into :daily_stats)
    (h/values [{:user_id 123 :date "2023-12-01" :page_views 15 :session_time 1800}])
    (h/on-conflict [:user_id :date])
    (h/do-update-set {:page_views   [:+ :daily_stats.page_views :excluded.page_views]
                      :session_time [:+ :daily_stats.session_time :excluded.session_time]})
    sql/format)
```

#### 生成されるSQL
```sql
INSERT INTO daily_stats (user_id, date, page_views, session_time) VALUES (?, ?, ?, ?) ON CONFLICT (user_id, date) DO UPDATE SET page_views = daily_stats.page_views + EXCLUDED.page_views, session_time = daily_stats.session_time + EXCLUDED.session_time
```

### DO NOTHINGによる重複無視

#### ヘルパー関数アプローチ
```clojure
(-> (h/insert-into :unique_visitors)
    (h/values [{:ip_address "192.168.1.1" :visit_date [:current_date]}])
    (h/on-conflict [:ip_address :visit_date])
    (h/do-nothing)
    sql/format)
```

#### 生成されるSQL
```sql
INSERT INTO unique_visitors (ip_address, visit_date) VALUES (?, CURRENT_DATE) ON CONFLICT (ip_address, visit_date) DO NOTHING
```

## 3. CTEと組み合わせたINSERT

### 複雑なデータ準備を伴う挿入

#### ヘルパー関数アプローチ
```clojure
(-> (h/with [:aggregated_data (-> (h/select :customer_id
                                            [[:count :*] :order_count]
                                            [[:sum :amount] :total_amount]
                                            [[:avg :amount] :avg_amount]
                                            [[:max :created_at] :last_order_date])
                                  (h/from :orders)
                                  (h/where [:>= :created_at "2023-01-01"])
                                  (h/group-by :customer_id))
             :customer_segments (-> (h/select :customer_id :order_count :total_amount :avg_amount :last_order_date
                                              [[:case
                                                [:>= :total_amount 10000] "VIP"
                                                [:>= :total_amount 5000] "Premium"
                                                [:>= :total_amount 1000] "Regular"
                                                :else "Basic"] :segment])
                                    (h/from :aggregated_data))]
    (h/insert-into :customer_analytics)
    (h/select :customer_id :segment :order_count :total_amount :avg_amount :last_order_date [:now] :calculated_at)
    (h/from :customer_segments)
    sql/format)
```

#### 生成されるSQL
```sql
WITH aggregated_data AS (SELECT customer_id, COUNT(*) AS order_count, SUM(amount) AS total_amount, AVG(amount) AS avg_amount, MAX(created_at) AS last_order_date FROM orders WHERE created_at >= ? GROUP BY customer_id), customer_segments AS (SELECT customer_id, order_count, total_amount, avg_amount, last_order_date, CASE WHEN total_amount >= ? THEN ? WHEN total_amount >= ? THEN ? WHEN total_amount >= ? THEN ? ELSE ? END AS segment FROM aggregated_data) INSERT INTO customer_analytics SELECT customer_id, segment, order_count, total_amount, avg_amount, last_order_date, NOW() AS calculated_at FROM customer_segments
```

## 4. 条件付きINSERT

### EXISTS句を使った条件付き挿入

#### ヘルパー関数アプローチ
```clojure
(-> (h/insert-into :notifications)
    (h/select [:inline 123] :user_id
              [:inline "Welcome bonus available!"] :message
              [:inline "bonus"] :type
              [:now] :created_at)
    (h/where [:and
              [:not-exists [:select 1
                            [:from :notifications]
                            [:where [:and
                                     [:= :user_id 123]
                                     [:= :type "bonus"]]]]]
              [:exists [:select 1
                        [:from :users]
                        [:where [:and
                                 [:= :id 123]
                                 [:= :status "active"]]]]]])
    sql/format)
```

#### 生成されるSQL
```sql
INSERT INTO notifications SELECT ?, ?, ?, NOW() WHERE (NOT EXISTS (SELECT ? FROM notifications WHERE (user_id = ?) AND (type = ?))) AND (EXISTS (SELECT ? FROM users WHERE (id = ?) AND (status = ?)))
```

## 5. 一括データ処理

### 大量データの効率的な挿入

#### ヘルパー関数アプローチ
```clojure
;; バッチサイズを制限した挿入
(-> (h/insert-into :audit_logs)
    (h/select :action :table_name :record_id :changed_data :created_at)
    (h/from [:temp_audit_data])
    (h/limit 10000)  ; バッチサイズ制限
    sql/format)
```

### データ変換を伴う一括挿入

#### ヘルパー関数アプローチ
```clojure
(-> (h/insert-into :normalized_addresses [:user_id :street :city :state :zip_code])
    (h/select :user_id
              [[:trim [:substring :full_address 1 50]] :street]
              [[:trim [:substring :full_address 51 30]] :city]
              [[:trim [:substring :full_address 81 10]] :state]
              [[:trim [:substring :full_address 91 10]] :zip_code])
    (h/from :raw_user_data)
    (h/where [:is-not :full_address nil])
    sql/format)
```

#### 生成されるSQL
```sql
INSERT INTO normalized_addresses (user_id, street, city, state, zip_code) SELECT user_id, TRIM(SUBSTRING(full_address, ?, ?)) AS street, TRIM(SUBSTRING(full_address, ?, ?)) AS city, TRIM(SUBSTRING(full_address, ?, ?)) AS state, TRIM(SUBSTRING(full_address, ?, ?)) AS zip_code FROM raw_user_data WHERE full_address IS NOT NULL
```

## 6. JSONデータの挿入

### JSONBフィールドへの挿入

#### ヘルパー関数アプローチ
```clojure
(-> (h/insert-into :user_preferences)
    (h/values [{:user_id 123
                :preferences {:theme "dark"
                              :language "ja"
                              :notifications {:email true :push false}
                              :privacy {:profile_visible true}}
                :created_at [:now]}])
    (h/on-conflict :user_id)
    (h/do-update-set {:preferences [:jsonb_set 
                                    :user_preferences.preferences 
                                    "{notifications}" 
                                    :excluded.preferences#>"{notifications}"]})
    sql/format)
```

#### 生成されるSQL
```sql
INSERT INTO user_preferences (user_id, preferences, created_at) VALUES (?, ?, NOW()) ON CONFLICT (user_id) DO UPDATE SET preferences = JSONB_SET(user_preferences.preferences, ?, EXCLUDED.preferences #> ?)
```

## 7. 配列データの挿入

### 配列フィールドの操作

#### ヘルパー関数アプローチ
```clojure
(-> (h/insert-into :article_tags)
    (h/values [{:article_id 456
                :tags ["postgresql" "database" "sql" "tutorial"]
                :created_at [:now]}])
    (h/on-conflict :article_id)
    (h/do-update-set {:tags [:|| :article_tags.tags :excluded.tags]})
    sql/format)
```

#### 生成されるSQL
```sql
INSERT INTO article_tags (article_id, tags, created_at) VALUES (?, ?, NOW()) ON CONFLICT (article_id) DO UPDATE SET tags = article_tags.tags || EXCLUDED.tags
```

## 8. 実践的な例：データ移行

### レガシーデータの正規化と挿入

#### ヘルパー関数アプローチ
```clojure
(-> (h/with [:cleaned_data (-> (h/select [[:trim :name] :clean_name]
                                         [[:lower [:trim :email]] :clean_email]
                                         [[:case
                                           [:or [:= :status "1"] [:= :status "active"]] "active"
                                           [:or [:= :status "0"] [:= :status "inactive"]] "inactive"
                                           :else "unknown"] :normalized_status]
                                         :legacy_id
                                         [[:coalesce :created_date [:now]] :safe_created_at])
                               (h/from :legacy_users)
                               (h/where [:and
                                         [:is-not :name nil]
                                         [:like :email "%@%"]]))
             :deduplicated (-> (h/select :* [[:row_number] :rn {:over [:partition-by :clean_email :order-by :legacy_id]}])
                               (h/from :cleaned_data))]
    (h/insert-into :users [:name :email :status :legacy_id :created_at])
    (h/select :clean_name :clean_email :normalized_status :legacy_id :safe_created_at)
    (h/from :deduplicated)
    (h/where [:= :rn 1])
    (h/on-conflict :email)
    (h/do-update-set {:legacy_id :excluded.legacy_id})
    sql/format)
```

## 9. パフォーマンス最適化

### バッチ挿入の最適化

#### ヘルパー関数アプローチ
```clojure
;; 大量データをチャンクに分けて処理する例
(defn bulk-insert-users [users chunk-size]
  (map (fn [chunk]
         (-> (h/insert-into :users)
             (h/values chunk)
             (h/on-conflict :email)
             (h/do-nothing)
             sql/format))
       (partition-all chunk-size users)))

;; 使用例
(bulk-insert-users user-data 1000)
```

### インデックスを考慮した挿入順序

#### ヘルパー関数アプローチ
```clojure
;; 挿入順序を最適化（インデックス効率を考慮）
(-> (h/insert-into :time_series_data)
    (h/select :sensor_id :timestamp :value :metadata)
    (h/from :raw_sensor_data)
    (h/order-by :sensor_id :timestamp)  ; パーティショニングキー順
    sql/format)
```

## 演習問題

1. `orders`テーブルから月次売上レポートを`monthly_sales`テーブルに自動生成するINSERT文を作成してください（既存データがある場合は更新）。

2. ユーザーのアクティビティログから、デイリーアクティビティサマリーを作成し、既存の場合は累積するUpsert文を作成してください。

3. CTEを使って、商品カテゴリ別の売上ランキングを計算し、`category_rankings`テーブルに挿入するクエリを作成してください。

4. JSONデータを持つ`raw_events`テーブルから、正規化された`processed_events`テーブルにデータを変換して挿入するクエリを作成してください。

## 次のステップ

次の章では、UPDATE操作の高度なテクニック（条件付き更新、JOINを使った更新、一括更新など）について学習します。