# 第4章: 集計関数とGROUP BY

## 概要

集計関数はデータを要約して統計情報を得るための重要な機能です。GROUP BY句と組み合わせることで、グループごとの集計値を計算できます。HoneySQLではすべての標準的な集計関数とPostgreSQL固有の高度な集計機能をサポートしています。

## 1. 基本的な集計関数

### COUNT - 行数を数える

#### ヘルパー関数アプローチ
```clojure
(-> (h/select [[:count :*] :total_users])
    (h/from :users)
    sql/format)
```

#### キーワードマップアプローチ
```clojure
(sql/format {:select [[[:count :*] :total_users]]
             :from   [:users]})
```

#### 生成されるSQL
```sql
SELECT COUNT(*) AS total_users FROM users
```

### SUM - 合計値

#### ヘルパー関数アプローチ
```clojure
(-> (h/select [[:sum :amount] :total_amount])
    (h/from :orders)
    sql/format)
```

#### キーワードマップアプローチ
```clojure
(sql/format {:select [[[:sum :amount] :total_amount]]
             :from   [:orders]})
```

#### 生成されるSQL
```sql
SELECT SUM(amount) AS total_amount FROM orders
```

### AVG - 平均値

#### ヘルパー関数アプローチ
```clojure
(-> (h/select [[:avg :age] :average_age])
    (h/from :users)
    sql/format)
```

#### 生成されるSQL
```sql
SELECT AVG(age) AS average_age FROM users
```

### MIN/MAX - 最小値/最大値

#### ヘルパー関数アプローチ
```clojure
(-> (h/select [[:min :created_at] :earliest_order]
              [[:max :created_at] :latest_order])
    (h/from :orders)
    sql/format)
```

#### 生成されるSQL
```sql
SELECT MIN(created_at) AS earliest_order, MAX(created_at) AS latest_order FROM orders
```

## 2. GROUP BY - グループ化

### 単一カラムでのグループ化

#### ヘルパー関数アプローチ
```clojure
(-> (h/select :department [[:count :*] :employee_count])
    (h/from :employees)
    (h/group-by :department)
    sql/format)
```

#### キーワードマップアプローチ
```clojure
(sql/format {:select [:department [[:count :*] :employee_count]]
             :from   [:employees]
             :group-by [:department]})
```

#### 生成されるSQL
```sql
SELECT department, COUNT(*) AS employee_count FROM employees GROUP BY department
```

### 複数カラムでのグループ化

#### ヘルパー関数アプローチ
```clojure
(-> (h/select :department :location [[:avg :salary] :avg_salary])
    (h/from :employees)
    (h/group-by :department :location)
    sql/format)
```

#### キーワードマップアプローチ
```clojure
(sql/format {:select [:department :location [[:avg :salary] :avg_salary]]
             :from   [:employees]
             :group-by [:department :location]})
```

#### 生成されるSQL
```sql
SELECT department, location, AVG(salary) AS avg_salary FROM employees GROUP BY department, location
```

## 3. HAVING句 - 集計結果のフィルタリング

### ヘルパー関数アプローチ
```clojure
(-> (h/select :department [[:count :*] :employee_count])
    (h/from :employees)
    (h/group-by :department)
    (h/having [:> [:count :*] 5])
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:department [[:count :*] :employee_count]]
             :from   [:employees]
             :group-by [:department]
             :having [:> [:count :*] 5]})
```

### 生成されるSQL
```sql
SELECT department, COUNT(*) AS employee_count FROM employees GROUP BY department HAVING COUNT(*) > ?
```

## 4. 複雑な集計例

### 売上データの月別集計

#### ヘルパー関数アプローチ
```clojure
(-> (h/select [[:extract "YEAR" :created_at] :year]
              [[:extract "MONTH" :created_at] :month]
              [[:count :*] :order_count]
              [[:sum :amount] :total_revenue]
              [[:avg :amount] :avg_order_value])
    (h/from :orders)
    (h/group-by [[:extract "YEAR" :created_at]]
                [[:extract "MONTH" :created_at]])
    (h/order-by [[:extract "YEAR" :created_at]]
                [[:extract "MONTH" :created_at]])
    sql/format)
```

#### キーワードマップアプローチ
```clojure
(sql/format {:select [[[:extract "YEAR" :created_at] :year]
                      [[:extract "MONTH" :created_at] :month]
                      [[:count :*] :order_count]
                      [[:sum :amount] :total_revenue]
                      [[:avg :amount] :avg_order_value]]
             :from   [:orders]
             :group-by [[:extract "YEAR" :created_at]
                        [:extract "MONTH" :created_at]]
             :order-by [[:extract "YEAR" :created_at]
                        [:extract "MONTH" :created_at]]})
```

#### 生成されるSQL
```sql
SELECT EXTRACT(YEAR FROM created_at) AS year, EXTRACT(MONTH FROM created_at) AS month, COUNT(*) AS order_count, SUM(amount) AS total_revenue, AVG(amount) AS avg_order_value FROM orders GROUP BY EXTRACT(YEAR FROM created_at), EXTRACT(MONTH FROM created_at) ORDER BY EXTRACT(YEAR FROM created_at), EXTRACT(MONTH FROM created_at)
```

## 5. JOINと集計の組み合わせ

### ヘルパー関数アプローチ
```clojure
(-> (h/select :u.name 
              [[:count :p.id] :post_count]
              [[:avg [:char_length :p.content]] :avg_post_length])
    (h/from [:users :u])
    (h/left-join [:posts :p] [:= :u.id :p.user_id])
    (h/group-by :u.id :u.name)
    (h/having [:> [:count :p.id] 0])
    (h/order-by [[:count :p.id] :desc])
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:u.name 
                      [[:count :p.id] :post_count]
                      [[:avg [:char_length :p.content]] :avg_post_length]]
             :from   [[:users :u]]
             :left-join [[:posts :p] [:= :u.id :p.user_id]]
             :group-by [:u.id :u.name]
             :having [:> [:count :p.id] 0]
             :order-by [[[:count :p.id] :desc]]})
```

### 生成されるSQL
```sql
SELECT u.name, COUNT(p.id) AS post_count, AVG(CHAR_LENGTH(p.content)) AS avg_post_length FROM users AS u LEFT JOIN posts AS p ON u.id = p.user_id GROUP BY u.id, u.name HAVING COUNT(p.id) > ? ORDER BY COUNT(p.id) DESC
```

## 6. COUNT DISTINCT - 重複を除外したカウント

### ヘルパー関数アプローチ
```clojure
(-> (h/select :category [[:count-distinct :user_id] :unique_customers])
    (h/from :orders)
    (h/group-by :category)
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:category [[:count-distinct :user_id] :unique_customers]]
             :from   [:orders]
             :group-by [:category]})
```

### 生成されるSQL
```sql
SELECT category, COUNT(DISTINCT user_id) AS unique_customers FROM orders GROUP BY category
```

## 7. 条件付き集計

### CASE文を使った条件付きカウント

#### ヘルパー関数アプローチ
```clojure
(-> (h/select :department
              [[:count :*] :total_employees]
              [[:sum [:case 
                      [:>= :salary 50000] 1 
                      :else 0]] :high_salary_count]
              [[:avg [:case 
                      [:= :gender "F"] :salary]] :avg_female_salary])
    (h/from :employees)
    (h/group-by :department)
    sql/format)
```

#### キーワードマップアプローチ
```clojure
(sql/format {:select [:department
                      [[:count :*] :total_employees]
                      [[:sum [:case 
                              [:>= :salary 50000] 1 
                              :else 0]] :high_salary_count]
                      [[:avg [:case 
                              [:= :gender "F"] :salary]] :avg_female_salary]]
             :from   [:employees]
             :group-by [:department]})
```

#### 生成されるSQL
```sql
SELECT department, COUNT(*) AS total_employees, SUM(CASE WHEN salary >= ? THEN ? ELSE ? END) AS high_salary_count, AVG(CASE WHEN gender = ? THEN salary END) AS avg_female_salary FROM employees GROUP BY department
```

## 8. PostgreSQL固有の集計関数

### STRING_AGG - 文字列集約

#### ヘルパー関数アプローチ
```clojure
(-> (h/select :department
              [[:string_agg :name ", "] :employee_names])
    (h/from :employees)
    (h/group-by :department)
    sql/format)
```

#### 生成されるSQL
```sql
SELECT department, STRING_AGG(name, ?) AS employee_names FROM employees GROUP BY department
```

### ARRAY_AGG - 配列集約

#### ヘルパー関数アプローチ
```clojure
(-> (h/select :user_id
              [[:array_agg :tag] :user_tags])
    (h/from :user_tags)
    (h/group-by :user_id)
    sql/format)
```

#### 生成されるSQL
```sql
SELECT user_id, ARRAY_AGG(tag) AS user_tags FROM user_tags GROUP BY user_id
```

### JSON_AGG - JSON集約

#### ヘルパー関数アプローチ
```clojure
(-> (h/select :department
              [[:json_agg [:json_build_object 
                           "name" :name 
                           "salary" :salary]] :employees])
    (h/from :employees)
    (h/group-by :department)
    sql/format)
```

#### 生成されるSQL
```sql
SELECT department, JSON_AGG(JSON_BUILD_OBJECT(?, name, ?, salary)) AS employees FROM employees GROUP BY department
```

## 9. ウィンドウ関数との比較

### 通常の集計 vs ウィンドウ関数

#### 通常の集計（各部署の平均給与）
```clojure
(-> (h/select :department [[:avg :salary] :dept_avg_salary])
    (h/from :employees)
    (h/group-by :department)
    sql/format)
```

#### ウィンドウ関数（各従業員と部署平均の比較）
```clojure
(-> (h/select :name :department :salary
              [[:avg :salary] :dept_avg_salary {:over [:partition-by :department]}])
    (h/from :employees)
    sql/format)
```

## 10. ROLLUP と CUBE（PostgreSQL）

### ROLLUP - 階層的な小計

#### ヘルパー関数アプローチ
```clojure
(-> (h/select :department :location [[:sum :salary] :total_salary])
    (h/from :employees)
    (h/group-by [[:rollup :department :location]])
    (h/order-by :department :location)
    sql/format)
```

#### 生成されるSQL
```sql
SELECT department, location, SUM(salary) AS total_salary FROM employees GROUP BY ROLLUP(department, location) ORDER BY department, location
```

### CUBE - 全ての組み合わせの小計

#### ヘルパー関数アプローチ
```clojure
(-> (h/select :department :location [[:sum :salary] :total_salary])
    (h/from :employees)
    (h/group-by [[:cube :department :location]])
    sql/format)
```

#### 生成されるSQL
```sql
SELECT department, location, SUM(salary) AS total_salary FROM employees GROUP BY CUBE(department, location)
```

## 11. GROUPING SETS

複数のグループ化を一つのクエリで実行：

#### ヘルパー関数アプローチ
```clojure
(-> (h/select :department :location [[:sum :salary] :total_salary])
    (h/from :employees)
    (h/group-by [[:grouping-sets 
                  [:department] 
                  [:location] 
                  [:department :location]
                  []]])
    sql/format)
```

#### 生成されるSQL
```sql
SELECT department, location, SUM(salary) AS total_salary FROM employees GROUP BY GROUPING SETS ((department), (location), (department, location), ())
```

## 演習問題

1. `sales`テーブルから、各販売員の総売上額と平均取引額を計算するクエリを作成してください。

2. `orders`テーブルと`order_items`テーブルをJOINして、各顧客の注文回数と総購入金額を計算し、購入金額が10000以上の顧客のみを表示するクエリを作成してください。

3. `products`テーブルから、カテゴリごとの商品数と価格帯（最高価格、最低価格、平均価格）を計算するクエリを作成してください。

4. PostgreSQLのSTRING_AGGを使って、各部署の従業員名をカンマ区切りで連結するクエリを作成してください。

## 次のステップ

次の章では、サブクエリとCTE（Common Table Expression）を使った高度なクエリ構築方法を学習します。