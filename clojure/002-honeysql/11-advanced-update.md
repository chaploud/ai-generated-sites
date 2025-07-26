# 第11章: 高度なUPDATE操作

## 概要

この章では、基本的なUPDATE文を超えた高度な更新操作を学習します。JOINを使った更新、サブクエリによる更新、CTEとの組み合わせ、条件付き更新、JSONや配列フィールドの更新など、実用的なパターンを網羅します。

## 1. JOINを使ったUPDATE

### 基本的なJOIN UPDATE

#### ヘルパー関数アプローチ
```clojure
(-> (h/update :users)
    (h/set {:total_orders :stats.order_count
            :total_spent  :stats.total_amount
            :last_order   :stats.last_order_date})
    (h/from [[:select :user_id
              [[:count :*] :order_count]
              [[:sum :amount] :total_amount]
              [[:max :created_at] :last_order_date]]
             [:from :orders]
             [:group-by :user_id]] :stats)
    (h/where [:= :users.id :stats.user_id])
    sql/format)
```

#### キーワードマップアプローチ
```clojure
(sql/format {:update :users
             :set {:total_orders :stats.order_count
                   :total_spent  :stats.total_amount
                   :last_order   :stats.last_order_date}
             :from [[{:select [:user_id
                               [[:count :*] :order_count]
                               [[:sum :amount] :total_amount]
                               [[:max :created_at] :last_order_date]]
                      :from   [:orders]
                      :group-by [:user_id]} :stats]]
             :where [:= :users.id :stats.user_id]})
```

#### 生成されるSQL
```sql
UPDATE users SET total_orders = stats.order_count, total_spent = stats.total_amount, last_order = stats.last_order_date FROM (SELECT user_id, COUNT(*) AS order_count, SUM(amount) AS total_amount, MAX(created_at) AS last_order_date FROM orders GROUP BY user_id) AS stats WHERE users.id = stats.user_id
```

### 複数テーブルとのJOIN UPDATE

#### ヘルパー関数アプローチ
```clojure
(-> (h/update :products)
    (h/set {:category_name :c.name
            :supplier_name :s.name
            :updated_at    [:now]})
    (h/from [:categories :c] [:suppliers :s])
    (h/where [:and
              [:= :products.category_id :c.id]
              [:= :products.supplier_id :s.id]
              [:is :products.category_name nil]])
    sql/format)
```

#### 生成されるSQL
```sql
UPDATE products SET category_name = c.name, supplier_name = s.name, updated_at = NOW() FROM categories AS c, suppliers AS s WHERE (products.category_id = c.id) AND (products.supplier_id = s.id) AND (products.category_name IS NULL)
```

## 2. サブクエリを使ったUPDATE

### スカラーサブクエリでの更新

#### ヘルパー関数アプローチ
```clojure
(-> (h/update :employees)
    (h/set {:department_avg_salary [:select [[:avg :salary]]
                                    [:from :employees :e2]
                                    [:where [:= :e2.department_id :employees.department_id]]]
            :rank_in_department [:select [[:count :*]]
                                 [:from :employees :e3]
                                 [:where [:and
                                          [:= :e3.department_id :employees.department_id]
                                          [:> :e3.salary :employees.salary]]]]})
    (h/where [:= :status "active"])
    sql/format)
```

#### 生成されるSQL
```sql
UPDATE employees SET department_avg_salary = (SELECT AVG(salary) FROM employees AS e2 WHERE e2.department_id = employees.department_id), rank_in_department = (SELECT COUNT(*) FROM employees AS e3 WHERE (e3.department_id = employees.department_id) AND (e3.salary > employees.salary)) WHERE status = ?
```

### EXISTS句を使った条件付き更新

#### ヘルパー関数アプローチ
```clojure
(-> (h/update :users)
    (h/set {:status "vip" :updated_at [:now]})
    (h/where [:and
              [:= :status "active"]
              [:exists [:select 1
                        [:from :orders]
                        [:where [:and
                                 [:= :orders.user_id :users.id]
                                 [:> :orders.amount 1000]
                                 [:>= :orders.created_at "2023-01-01"]]]]]])
    sql/format)
```

#### 生成されるSQL
```sql
UPDATE users SET status = ?, updated_at = NOW() WHERE (status = ?) AND (EXISTS (SELECT ? FROM orders WHERE (orders.user_id = users.id) AND (orders.amount > ?) AND (orders.created_at >= ?)))
```

## 3. CTEを使ったUPDATE

### 複雑な計算を伴う更新

#### ヘルパー関数アプローチ
```clojure
(-> (h/with [:sales_stats (-> (h/select :salesperson_id
                                        [[:sum :amount] :total_sales]
                                        [[:count :*] :deals_closed]
                                        [[:avg :amount] :avg_deal_size])
                              (h/from :sales)
                              (h/where [:>= :close_date "2023-01-01"])
                              (h/group-by :salesperson_id))
             :rankings (-> (h/select :salesperson_id :total_sales :deals_closed :avg_deal_size
                                     [[:rank] :sales_rank {:over [:order-by [[:total_sales :desc]]]}]
                                     [[:ntile 4] :quartile {:over [:order-by [[:total_sales :desc]]]}])
                           (h/from :sales_stats))]
    (h/update :employees)
    (h/set {:annual_sales :r.total_sales
            :deals_closed :r.deals_closed
            :avg_deal_size :r.avg_deal_size
            :sales_rank :r.sales_rank
            :performance_quartile :r.quartile})
    (h/from [:rankings :r])
    (h/where [:= :employees.id :r.salesperson_id])
    sql/format)
```

#### 生成されるSQL
```sql
WITH sales_stats AS (SELECT salesperson_id, SUM(amount) AS total_sales, COUNT(*) AS deals_closed, AVG(amount) AS avg_deal_size FROM sales WHERE close_date >= ? GROUP BY salesperson_id), rankings AS (SELECT salesperson_id, total_sales, deals_closed, avg_deal_size, RANK() OVER (ORDER BY total_sales DESC) AS sales_rank, NTILE(?) OVER (ORDER BY total_sales DESC) AS quartile FROM sales_stats) UPDATE employees SET annual_sales = r.total_sales, deals_closed = r.deals_closed, avg_deal_size = r.avg_deal_size, sales_rank = r.sales_rank, performance_quartile = r.quartile FROM rankings AS r WHERE employees.id = r.salesperson_id
```

## 4. 条件付きUPDATE（CASE文）

### 複雑な条件分岐による更新

#### ヘルパー関数アプローチ
```clojure
(-> (h/update :products)
    (h/set {:discount_rate [:case
                            [:and [:>= :stock 100] [:< :days_since_last_sale 30]] 0.05
                            [:and [:>= :stock 50] [:< :days_since_last_sale 60]] 0.10
                            [:and [:>= :stock 20] [:< :days_since_last_sale 90]] 0.15
                            [:< :stock 10] 0.0
                            :else 0.20]
            :status [:case
                     [:= :stock 0] "out_of_stock"
                     [:< :stock 10] "low_stock"
                     :else "in_stock"]
            :updated_at [:now]})
    (h/where [:= :active true])
    sql/format)
```

#### 生成されるSQL
```sql
UPDATE products SET discount_rate = CASE WHEN (stock >= ?) AND (days_since_last_sale < ?) THEN ? WHEN (stock >= ?) AND (days_since_last_sale < ?) THEN ? WHEN (stock >= ?) AND (days_since_last_sale < ?) THEN ? WHEN stock < ? THEN ? ELSE ? END, status = CASE WHEN stock = ? THEN ? WHEN stock < ? THEN ? ELSE ? END, updated_at = NOW() WHERE active = ?
```

## 5. JSONフィールドの更新

### JSONBフィールドの部分更新

#### ヘルパー関数アプローチ
```clojure
(-> (h/update :user_preferences)
    (h/set {:settings [:jsonb_set :settings "{notifications,email}" "false"]
            :updated_at [:now]})
    (h/where [:= :user_id 123])
    sql/format)
```

#### 生成されるSQL
```sql
UPDATE user_preferences SET settings = JSONB_SET(settings, ?, ?), updated_at = NOW() WHERE user_id = ?
```

### JSONフィールドの要素追加

#### ヘルパー関数アプローチ
```clojure
(-> (h/update :products)
    (h/set {:features [:jsonb_set 
                       :features 
                       "{new_features}" 
                       [[:to_jsonb ["wireless_charging" "waterproof"]]]]
            :metadata [:|| :metadata {:last_updated [:now]}]})
    (h/where [:= :id 456])
    sql/format)
```

#### 生成されるSQL
```sql
UPDATE products SET features = JSONB_SET(features, ?, TO_JSONB(?)), metadata = metadata || ? WHERE id = ?
```

### JSON配列への要素追加

#### ヘルパー関数アプローチ
```clojure
(-> (h/update :articles)
    (h/set {:tags [:jsonb_set 
                   :tags 
                   [[:|| "{" [[:array_length [:jsonb_path_query_array :tags "$[*]"] 1]] "}"]]
                   [[:to_jsonb "new_tag"]]]})
    (h/where [:= :id 789])
    sql/format)
```

## 6. 配列フィールドの更新

### 配列要素の追加と削除

#### ヘルパー関数アプローチ
```clojure
(-> (h/update :user_skills)
    (h/set {:skills [:|| :skills ["postgresql" "honeysql"]]  ; 要素追加
            :removed_skills [:array_remove :skills "outdated_skill"]})  ; 要素削除
    (h/where [:= :user_id 123])
    sql/format)
```

#### 生成されるSQL
```sql
UPDATE user_skills SET skills = skills || ?, removed_skills = ARRAY_REMOVE(skills, ?) WHERE user_id = ?
```

### 配列の重複除去

#### ヘルパー関数アプローチ
```clojure
(-> (h/update :article_tags)
    (h/set {:tags [:select [[:array_agg [:distinct :tag]]]
                   [:from [:unnest :tags] :t :tag]]})
    (h/where [[:array_length :tags 1] :> 0])
    sql/format)
```

## 7. ウィンドウ関数を使った更新

### ランキングによる更新

#### ヘルパー関数アプローチ
```clojure
(-> (h/update :students)
    (h/set {:class_rank :ranked_students.rank
            :percentile :ranked_students.percentile})
    (h/from [[:select :id
              [[:rank] :rank {:over [:partition-by :class_id :order-by [[:score :desc]]]}]
              [[:percent_rank] :percentile {:over [:partition-by :class_id :order-by [[:score :desc]]]}]]
             [:from :students]] :ranked_students)
    (h/where [:= :students.id :ranked_students.id])
    sql/format)
```

#### 生成されるSQL
```sql
UPDATE students SET class_rank = ranked_students.rank, percentile = ranked_students.percentile FROM (SELECT id, RANK() OVER (PARTITION BY class_id ORDER BY score DESC) AS rank, PERCENT_RANK() OVER (PARTITION BY class_id ORDER BY score DESC) AS percentile FROM students) AS ranked_students WHERE students.id = ranked_students.id
```

## 8. 一括更新の最適化

### バッチ処理による大量データ更新

#### ヘルパー関数アプローチ
```clojure
;; 一時テーブルを使った効率的な一括更新
(-> (h/update :products)
    (h/set {:new_price :price_updates.new_price
            :discount_rate :price_updates.discount
            :updated_at [:now]})
    (h/from [:temp_price_updates :price_updates])
    (h/where [:= :products.sku :price_updates.sku])
    sql/format)
```

### 条件による段階的更新

#### ヘルパー関数アプローチ
```clojure
(defn update-inventory-status []
  [
   ;; ステップ1: 在庫切れ商品
   (-> (h/update :products)
       (h/set {:status "out_of_stock" :updated_at [:now]})
       (h/where [:and [:= :stock 0] [:!= :status "out_of_stock"]])
       sql/format)
   
   ;; ステップ2: 低在庫商品
   (-> (h/update :products)
       (h/set {:status "low_stock" :updated_at [:now]})
       (h/where [:and [:between :stock 1 10] [:!= :status "low_stock"]])
       sql/format)
   
   ;; ステップ3: 通常在庫商品
   (-> (h/update :products)
       (h/set {:status "in_stock" :updated_at [:now]})
       (h/where [:and [:> :stock 10] [:!= :status "in_stock"]])
       sql/format)
  ])
```

## 9. 更新の検証とログ

### RETURNING句を使った更新結果の確認

#### ヘルパー関数アプローチ
```clojure
(-> (h/update :orders)
    (h/set {:status "shipped"
            :shipped_at [:now]
            :tracking_number [:concat "TRK" [:extract "epoch" [:now]]]})
    (h/where [:and
              [:= :status "processing"]
              [:exists [:select 1
                        [:from :inventory]
                        [:where [:and
                                 [:= :inventory.product_id :orders.product_id]
                                 [:>= :inventory.quantity :orders.quantity]]]]]])
    (h/returning :id :customer_id :tracking_number :shipped_at)
    sql/format)
```

#### 生成されるSQL
```sql
UPDATE orders SET status = ?, shipped_at = NOW(), tracking_number = CONCAT(?, EXTRACT(EPOCH FROM NOW())) WHERE (status = ?) AND (EXISTS (SELECT ? FROM inventory WHERE (inventory.product_id = orders.product_id) AND (inventory.quantity >= orders.quantity))) RETURNING id, customer_id, tracking_number, shipped_at
```

## 10. 実践的な例：データ統合とクリーンアップ

### 重複データの統合

#### ヘルパー関数アプローチ
```clojure
(-> (h/with [:duplicate_groups (-> (h/select :email [[:min :id] :keep_id] [[:count :*] :duplicate_count])
                                   (h/from :users)
                                   (h/group-by :email)
                                   (h/having [:> [:count :*] 1]))
             :merge_data (-> (h/select :dg.keep_id
                                       [[:string_agg :u.name ", "] :merged_names]
                                       [[:max :u.created_at] :latest_created]
                                       [[:sum :u.login_count] :total_logins])
                             (h/from [:duplicate_groups :dg])
                             (h/join [:users :u] [:= :dg.email :u.email])
                             (h/group-by :dg.keep_id))]
    (h/update :users)
    (h/set {:name :md.merged_names
            :login_count :md.total_logins
            :created_at :md.latest_created
            :updated_at [:now]})
    (h/from [:merge_data :md])
    (h/where [:= :users.id :md.keep_id])
    sql/format)
```

## 11. エラーハンドリングと安全な更新

### 制約チェック付き更新

#### ヘルパー関数アプローチ
```clojure
(-> (h/update :bank_accounts)
    (h/set {:balance [:- :balance 100]
            :last_transaction [:now]})
    (h/where [:and
              [:= :account_id "ACC123"]
              [:>= [:- :balance 100] 0]])  ; 残高不足を防ぐ
    (h/returning :account_id :balance)
    sql/format)
```

## 演習問題

1. `products`テーブルと`order_items`テーブルをJOINして、各商品の累計販売数量を`products.total_sold`フィールドに更新するクエリを作成してください。

2. CTEを使って、各顧客の過去1年間の購入履歴から顧客ランク（Bronze, Silver, Gold, Platinum）を計算し、`customers.rank`フィールドを更新するクエリを作成してください。

3. JSONフィールド`user_settings`の`notifications.email`プロパティを条件に応じて更新し、かつ`last_updated`タイムスタンプを追加するクエリを作成してください。

4. ウィンドウ関数を使って、各部門の従業員に対して給与順位を計算し、トップ10%の従業員に`top_performer`フラグを設定するクエリを作成してください。

## 次のステップ

次の章では、DELETE操作の高度なテクニック（条件付き削除、カスケード削除、ソフトデリートなど）について学習します。