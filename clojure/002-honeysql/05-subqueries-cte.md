# 第5章: サブクエリとCTE（Common Table Expression）

## 概要

サブクエリとCTE（Common Table Expression）は、複雑なデータ処理を段階的に行うための強力な機能です。CTEは特に、再帰処理や複雑なクエリの可読性向上に有効です。HoneySQLではサブクエリとCTEの両方を完全にサポートしています。

## 1. 基本的なサブクエリ

### SELECT句でのサブクエリ（スカラーサブクエリ）

#### ヘルパー関数アプローチ
```clojure
(-> (h/select :name 
              :department
              [[:select [[:avg :salary]]
                [:from :employees :e2]
                [:where [:= :e2.department :e1.department]]] :dept_avg_salary])
    (h/from [:employees :e1])
    sql/format)
```

#### キーワードマップアプローチ
```clojure
(sql/format {:select [:name 
                      :department
                      [{:select [[[:avg :salary]]]
                        :from   [:employees :e2]
                        :where  [:= :e2.department :e1.department]} :dept_avg_salary]]
             :from   [[:employees :e1]]})
```

#### 生成されるSQL
```sql
SELECT name, department, (SELECT AVG(salary) FROM employees AS e2 WHERE e2.department = e1.department) AS dept_avg_salary FROM employees AS e1
```

### WHERE句でのサブクエリ

#### ヘルパー関数アプローチ
```clojure
(-> (h/select :name :salary)
    (h/from :employees)
    (h/where [:> :salary [:select [[:avg :salary]]
                          [:from :employees]]])
    sql/format)
```

#### キーワードマップアプローチ
```clojure
(sql/format {:select [:name :salary]
             :from   [:employees]
             :where  [:> :salary {:select [[[:avg :salary]]]
                                  :from   [:employees]}]})
```

#### 生成されるSQL
```sql
SELECT name, salary FROM employees WHERE salary > (SELECT AVG(salary) FROM employees)
```

### IN句でのサブクエリ

#### ヘルパー関数アプローチ
```clojure
(-> (h/select :name)
    (h/from :employees)
    (h/where [:in :department_id [:select :id
                                  [:from :departments]
                                  [:where [:like :name "%Tech%"]]]])
    sql/format)
```

#### キーワードマップアプローチ
```clojure
(sql/format {:select [:name]
             :from   [:employees]
             :where  [:in :department_id {:select [:id]
                                          :from   [:departments]
                                          :where  [:like :name "%Tech%"]}]})
```

#### 生成されるSQL
```sql
SELECT name FROM employees WHERE department_id IN (SELECT id FROM departments WHERE name LIKE ?)
```

## 2. EXISTS句

### ヘルパー関数アプローチ
```clojure
(-> (h/select :name)
    (h/from :users)
    (h/where [:exists [:select 1
                       [:from :orders]
                       [:where [:= :orders.user_id :users.id]]]])
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:name]
             :from   [:users]
             :where  [:exists {:select [1]
                               :from   [:orders]
                               :where  [:= :orders.user_id :users.id]}]})
```

### 生成されるSQL
```sql
SELECT name FROM users WHERE EXISTS (SELECT ? FROM orders WHERE orders.user_id = users.id)
```

## 3. NOT EXISTS句

### ヘルパー関数アプローチ
```clojure
(-> (h/select :name)
    (h/from :users)
    (h/where [:not-exists [:select 1
                           [:from :orders]
                           [:where [:= :orders.user_id :users.id]]]])
    sql/format)
```

### 生成されるSQL
```sql
SELECT name FROM users WHERE NOT EXISTS (SELECT ? FROM orders WHERE orders.user_id = users.id)
```

## 4. 基本的なCTE（WITH句）

### 単一のCTE

#### ヘルパー関数アプローチ
```clojure
(-> (h/with [:high_earners (-> (h/select :id :name :salary)
                               (h/from :employees)
                               (h/where [:> :salary 70000]))])
    (h/select :name :salary)
    (h/from :high_earners)
    (h/order-by [:salary :desc])
    sql/format)
```

#### キーワードマップアプローチ
```clojure
(sql/format {:with [[:high_earners {:select [:id :name :salary]
                                    :from   [:employees]
                                    :where  [:> :salary 70000]}]]
             :select [:name :salary]
             :from   [:high_earners]
             :order-by [[:salary :desc]]})
```

#### 生成されるSQL
```sql
WITH high_earners AS (SELECT id, name, salary FROM employees WHERE salary > ?) SELECT name, salary FROM high_earners ORDER BY salary DESC
```

### 複数のCTE

#### ヘルパー関数アプローチ
```clojure
(-> (h/with [:dept_stats (-> (h/select :department 
                                       [[:avg :salary] :avg_salary]
                                       [[:count :*] :emp_count])
                             (h/from :employees)
                             (h/group-by :department))
             :high_earners (-> (h/select :id :name :department :salary)
                               (h/from :employees)
                               (h/where [:> :salary 60000]))])
    (h/select :he.name :he.salary :ds.avg_salary :ds.emp_count)
    (h/from [:high_earners :he])
    (h/join [:dept_stats :ds] [:= :he.department :ds.department])
    sql/format)
```

#### キーワードマップアプローチ
```clojure
(sql/format {:with [[:dept_stats {:select [:department 
                                           [[:avg :salary] :avg_salary]
                                           [[:count :*] :emp_count]]
                                  :from   [:employees]
                                  :group-by [:department]}]
                    [:high_earners {:select [:id :name :department :salary]
                                    :from   [:employees]
                                    :where  [:> :salary 60000]}]]
             :select [:he.name :he.salary :ds.avg_salary :ds.emp_count]
             :from   [[:high_earners :he]]
             :join   [[:dept_stats :ds] [:= :he.department :ds.department]]})
```

#### 生成されるSQL
```sql
WITH dept_stats AS (SELECT department, AVG(salary) AS avg_salary, COUNT(*) AS emp_count FROM employees GROUP BY department), high_earners AS (SELECT id, name, department, salary FROM employees WHERE salary > ?) SELECT he.name, he.salary, ds.avg_salary, ds.emp_count FROM high_earners AS he INNER JOIN dept_stats AS ds ON he.department = ds.department
```

## 5. CTEの連鎖（依存関係）

### ヘルパー関数アプローチ
```clojure
(-> (h/with [:regional_sales (-> (h/select :region [[:sum :sales_amount] :total_sales])
                                 (h/from :sales)
                                 (h/group-by :region))
             :top_regions (-> (h/select :region :total_sales)
                              (h/from :regional_sales)
                              (h/where [:> :total_sales 100000]))]
    (h/select :region :total_sales)
    (h/from :top_regions)
    (h/order-by [:total_sales :desc])
    sql/format)
```

#### 生成されるSQL
```sql
WITH regional_sales AS (SELECT region, SUM(sales_amount) AS total_sales FROM sales GROUP BY region), top_regions AS (SELECT region, total_sales FROM regional_sales WHERE total_sales > ?) SELECT region, total_sales FROM top_regions ORDER BY total_sales DESC
```

## 6. 複雑なサブクエリの例

### FROM句でのサブクエリ（派生テーブル）

#### ヘルパー関数アプローチ
```clojure
(-> (h/select :dept_summary.department :dept_summary.avg_salary)
    (h/from [[:select :department [[:avg :salary] :avg_salary]]
             [:from :employees]
             [:group-by :department]
             [:having [:> [:count :*] 5]]] :dept_summary)
    (h/where [:> :dept_summary.avg_salary 50000])
    sql/format)
```

#### キーワードマップアプローチ
```clojure
(sql/format {:select [:dept_summary.department :dept_summary.avg_salary]
             :from   [[{:select [:department [[:avg :salary] :avg_salary]]
                        :from   [:employees]
                        :group-by [:department]
                        :having [:> [:count :*] 5]} :dept_summary]]
             :where  [:> :dept_summary.avg_salary 50000]})
```

#### 生成されるSQL
```sql
SELECT dept_summary.department, dept_summary.avg_salary FROM (SELECT department, AVG(salary) AS avg_salary FROM employees GROUP BY department HAVING COUNT(*) > ?) AS dept_summary WHERE dept_summary.avg_salary > ?
```

## 7. 相関サブクエリ

### ヘルパー関数アプローチ
```clojure
(-> (h/select :e1.name :e1.salary)
    (h/from [:employees :e1])
    (h/where [:< [:select [[:count :*]]
                  [:from :employees :e2]
                  [:where [:and 
                           [:= :e2.department :e1.department]
                           [:> :e2.salary :e1.salary]]]] 3])
    sql/format)
```

#### 生成されるSQL
```sql
SELECT e1.name, e1.salary FROM employees AS e1 WHERE (SELECT COUNT(*) FROM employees AS e2 WHERE (e2.department = e1.department) AND (e2.salary > e1.salary)) < ?
```

## 8. ANY/ALL演算子

### ANY演算子

#### ヘルパー関数アプローチ
```clojure
(-> (h/select :name :salary)
    (h/from :employees)
    (h/where [:> :salary [:any [:select :salary
                                [:from :employees]
                                [:where [:= :department "Sales"]]]]])
    sql/format)
```

#### 生成されるSQL
```sql
SELECT name, salary FROM employees WHERE salary > ANY (SELECT salary FROM employees WHERE department = ?)
```

### ALL演算子

#### ヘルパー関数アプローチ
```clojure
(-> (h/select :name :salary)
    (h/from :employees)
    (h/where [:> :salary [:all [:select :salary
                                [:from :employees]
                                [:where [:= :department "Sales"]]]]])
    sql/format)
```

#### 生成されるSQL
```sql
SELECT name, salary FROM employees WHERE salary > ALL (SELECT salary FROM employees WHERE department = ?)
```

## 9. CTEを使ったデータ変換の例

### 段階的なデータ処理

#### ヘルパー関数アプローチ
```clojure
(-> (h/with [:monthly_sales (-> (h/select [[:extract "YEAR" :order_date] :year]
                                          [[:extract "MONTH" :order_date] :month]
                                          [[:sum :amount] :monthly_total])
                                (h/from :orders)
                                (h/group-by [[:extract "YEAR" :order_date]]
                                            [[:extract "MONTH" :order_date]]))
             :monthly_growth (-> (h/select :year :month :monthly_total
                                           [[:lag :monthly_total 1] :prev_month_total {:over [:order-by :year :month]}])
                                 (h/from :monthly_sales))
             :growth_rates (-> (h/select :year :month :monthly_total :prev_month_total
                                         [[:case 
                                           [:is-not :prev_month_total nil]
                                           [[:* [[:/ [:- :monthly_total :prev_month_total] :prev_month_total]] 100]]
                                           :else nil] :growth_rate])
                               (h/from :monthly_growth))]
    (h/select :year :month :monthly_total :growth_rate)
    (h/from :growth_rates)
    (h/where [:is-not :growth_rate nil])
    (h/order-by :year :month)
    sql/format)
```

## 10. ウィンドウ関数との組み合わせ

### ヘルパー関数アプローチ
```clojure
(-> (h/with [:sales_with_rank (-> (h/select :salesperson 
                                            :region 
                                            :sales_amount
                                            [[:rank] :region_rank {:over [:partition-by :region 
                                                                           :order-by [[:sales_amount :desc]]]}])
                                  (h/from :sales))]
    (h/select :salesperson :region :sales_amount :region_rank)
    (h/from :sales_with_rank)
    (h/where [:<= :region_rank 3])
    (h/order-by :region :region_rank)
    sql/format)
```

#### 生成されるSQL
```sql
WITH sales_with_rank AS (SELECT salesperson, region, sales_amount, RANK() OVER (PARTITION BY region ORDER BY sales_amount DESC) AS region_rank FROM sales) SELECT salesperson, region, sales_amount, region_rank FROM sales_with_rank WHERE region_rank <= ? ORDER BY region, region_rank
```

## 11. CTEの実践的な活用例

### 複雑なビジネスロジックの実装

#### ヘルパー関数アプローチ
```clojure
(-> (h/with [:customer_orders (-> (h/select :customer_id 
                                            [[:count :*] :order_count]
                                            [[:sum :total_amount] :total_spent]
                                            [[:max :order_date] :last_order_date])
                                  (h/from :orders)
                                  (h/group-by :customer_id))
             :customer_segments (-> (h/select :customer_id 
                                              :order_count 
                                              :total_spent
                                              :last_order_date
                                              [[:case
                                                [:and [:>= :total_spent 10000] [:>= :order_count 10]] "VIP"
                                                [:and [:>= :total_spent 5000] [:>= :order_count 5]] "Premium"
                                                [:and [:>= :total_spent 1000] [:>= :order_count 2]] "Regular"
                                                :else "New"] :segment]
                                              [[:case
                                                [:> [:extract "DAY" [:- [:now] :last_order_date]] 90] "Inactive"
                                                :else "Active"] :activity_status])
                                    (h/from :customer_orders))]
    (h/select :cs.segment 
              :cs.activity_status
              [[:count :*] :customer_count]
              [[:avg :cs.total_spent] :avg_spent]
              [[:sum :cs.total_spent] :total_revenue])
    (h/from [:customer_segments :cs])
    (h/group-by :cs.segment :cs.activity_status)
    (h/order-by :cs.segment :cs.activity_status)
    sql/format)
```

## 演習問題

1. 各部署で最も給与の高い従業員を、サブクエリを使って取得するクエリを作成してください。

2. CTEを使って、各月の売上高と前月比成長率を計算するクエリを作成してください。

3. EXISTS句を使って、注文履歴があるユーザーのみを取得するクエリを作成してください。

4. 複数のCTEを使って、各商品カテゴリの売上ランキング（トップ5）を取得するクエリを作成してください。

## 次のステップ

次の章では、UNION/UNION ALLを使った複数クエリの結合方法を学習します。