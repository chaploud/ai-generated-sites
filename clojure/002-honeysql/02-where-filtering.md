# 第2章: WHERE句とフィルタリング

## 概要

WHERE句はSQLクエリの核心部分で、データをフィルタリングして必要な行のみを取得します。HoneySQLでは様々な条件演算子と論理演算子を使って複雑な条件を構築できます。

## 1. 基本的な等価条件

### ヘルパー関数アプローチ
```clojure
(-> (h/select :*)
    (h/from :users)
    (h/where [:= :status "active"])
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:*]
             :from   [:users]
             :where  [:= :status "active"]})
```

### 生成されるSQL
```sql
SELECT * FROM users WHERE status = ?
```

## 2. 数値比較

### ヘルパー関数アプローチ
```clojure
(-> (h/select :name :age)
    (h/from :users)
    (h/where [:>= :age 18])
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:name :age]
             :from   [:users]
             :where  [:>= :age 18]})
```

### 生成されるSQL
```sql
SELECT name, age FROM users WHERE age >= ?
```

## 3. 複数の条件（AND）

### ヘルパー関数アプローチ
```clojure
(-> (h/select :*)
    (h/from :users)
    (h/where [:= :status "active"]
             [:>= :age 18]
             [:<= :age 65])
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:*]
             :from   [:users]
             :where  [:and
                      [:= :status "active"]
                      [:>= :age 18]
                      [:<= :age 65]]})
```

### 生成されるSQL
```sql
SELECT * FROM users WHERE (status = ?) AND (age >= ?) AND (age <= ?)
```

## 4. OR条件

### ヘルパー関数アプローチ
```clojure
(-> (h/select :*)
    (h/from :users)
    (h/where [:or
              [:= :department "sales"]
              [:= :department "marketing"]])
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:*]
             :from   [:users]
             :where  [:or
                      [:= :department "sales"]
                      [:= :department "marketing"]]})
```

### 生成されるSQL
```sql
SELECT * FROM users WHERE (department = ?) OR (department = ?)
```

## 5. IN句

### ヘルパー関数アプローチ
```clojure
(-> (h/select :*)
    (h/from :users)
    (h/where [:in :department ["sales" "marketing" "engineering"]])
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:*]
             :from   [:users]
             :where  [:in :department ["sales" "marketing" "engineering"]]})
```

### 生成されるSQL
```sql
SELECT * FROM users WHERE department IN (?, ?, ?)
```

## 6. NOT IN句

### ヘルパー関数アプローチ
```clojure
(-> (h/select :*)
    (h/from :users)
    (h/where [:not-in :status ["inactive" "suspended"]])
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:*]
             :from   [:users]
             :where  [:not-in :status ["inactive" "suspended"]]})
```

### 生成されるSQL
```sql
SELECT * FROM users WHERE status NOT IN (?, ?)
```

## 7. LIKE文字列マッチング

### ヘルパー関数アプローチ
```clojure
(-> (h/select :*)
    (h/from :users)
    (h/where [:like :name "%john%"])
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:*]
             :from   [:users]
             :where  [:like :name "%john%"]})
```

### 生成されるSQL
```sql
SELECT * FROM users WHERE name LIKE ?
```

## 8. ILIKE（大文字小文字を区別しない）

### ヘルパー関数アプローチ
```clojure
(-> (h/select :*)
    (h/from :users)
    (h/where [:ilike :email "%@EXAMPLE.COM"])
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:*]
             :from   [:users]
             :where  [:ilike :email "%@EXAMPLE.COM"]})
```

### 生成されるSQL
```sql
SELECT * FROM users WHERE email ILIKE ?
```

## 9. BETWEEN範囲条件

### ヘルパー関数アプローチ
```clojure
(-> (h/select :*)
    (h/from :orders)
    (h/where [:between :created_at "2023-01-01" "2023-12-31"])
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:*]
             :from   [:orders]
             :where  [:between :created_at "2023-01-01" "2023-12-31"]})
```

### 生成されるSQL
```sql
SELECT * FROM orders WHERE created_at BETWEEN ? AND ?
```

## 10. NULL値のチェック

### ヘルパー関数アプローチ
```clojure
;; IS NULL
(-> (h/select :*)
    (h/from :users)
    (h/where [:is :phone nil])
    sql/format)

;; IS NOT NULL
(-> (h/select :*)
    (h/from :users)
    (h/where [:is-not :phone nil])
    sql/format)
```

### キーワードマップアプローチ
```clojure
;; IS NULL
(sql/format {:select [:*]
             :from   [:users]
             :where  [:is :phone nil]})

;; IS NOT NULL
(sql/format {:select [:*]
             :from   [:users]
             :where  [:is-not :phone nil]})
```

### 生成されるSQL
```sql
-- IS NULL
SELECT * FROM users WHERE phone IS NULL

-- IS NOT NULL
SELECT * FROM users WHERE phone IS NOT NULL
```

## 11. 複雑な条件の組み合わせ

### ヘルパー関数アプローチ
```clojure
(-> (h/select :*)
    (h/from :users)
    (h/where [:and
              [:= :status "active"]
              [:or
               [:>= :age 65]
               [:and
                [:>= :age 18]
                [:in :department ["management" "senior"]]]]])
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:*]
             :from   [:users]
             :where  [:and
                      [:= :status "active"]
                      [:or
                       [:>= :age 65]
                       [:and
                        [:>= :age 18]
                        [:in :department ["management" "senior"]]]]]})
```

### 生成されるSQL
```sql
SELECT * FROM users WHERE (status = ?) AND ((age >= ?) OR ((age >= ?) AND department IN (?, ?)))
```

## 12. PostgreSQL固有の演算子

### 正規表現マッチング
```clojure
;; ヘルパー関数アプローチ
(-> (h/select :*)
    (h/from :users)
    (h/where [:~ :email ".*@example\\.com$"])
    sql/format)

;; キーワードマップアプローチ
(sql/format {:select [:*]
             :from   [:users]
             :where  [:~ :email ".*@example\\.com$"]})
```

### 生成されるSQL
```sql
SELECT * FROM users WHERE email ~ ?
```

### 大文字小文字を区別しない正規表現
```clojure
;; ヘルパー関数アプローチ
(-> (h/select :*)
    (h/from :users)
    (h/where [:~* :name "jo?hn"])
    sql/format)
```

### 生成されるSQL
```sql
SELECT * FROM users WHERE name ~* ?
```

## 13. 配列演算子（PostgreSQL）

### ヘルパー関数アプローチ
```clojure
(-> (h/select :*)
    (h/from :users)
    (h/where [:@> :tags ["postgresql" "database"]])
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:*]
             :from   [:users]
             :where  [:@> :tags ["postgresql" "database"]]})
```

### 生成されるSQL
```sql
SELECT * FROM users WHERE tags @> ?
```

## 14. 動的WHERE条件の構築

時には、条件が動的に決まる場合があります：

```clojure
(defn build-user-query [filters]
  (let [base-query (-> (h/select :*)
                       (h/from :users))]
    (cond-> base-query
      (:name filters)
      (h/where [:like :name (str "%" (:name filters) "%")])
      
      (:min-age filters)
      (h/where [:>= :age (:min-age filters)])
      
      (:department filters)
      (h/where [:= :department (:department filters)])
      
      (:active-only filters)
      (h/where [:= :status "active"]))))

;; 使用例
(sql/format (build-user-query {:name "john" :min-age 25 :active-only true}))
```

## 演習問題

1. `products`テーブルから、価格が100以上500以下の商品を選択するクエリを作成してください。

2. `employees`テーブルから、部署が"IT"または"Engineering"で、かつ給与が50000以上の従業員を選択するクエリを作成してください。

3. `orders`テーブルから、顧客IDがnullではなく、注文日が2023年内の注文を選択するクエリを作成してください。

4. `users`テーブルから、メールアドレスが"@gmail.com"で終わるユーザーを正規表現を使って選択するクエリを作成してください。

## 次のステップ

次の章では、JOIN操作を使って複数のテーブルからデータを取得する方法を学習します。