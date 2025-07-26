# 第3章: JOIN操作

## 概要

JOIN操作は複数のテーブルからデータを組み合わせて取得するための重要な機能です。HoneySQLではINNER JOIN、LEFT JOIN、RIGHT JOIN、FULL OUTER JOINなど、すべてのJOINタイプをサポートしています。

## 1. INNER JOIN（内部結合）

### ヘルパー関数アプローチ
```clojure
(-> (h/select :u.name :p.title)
    (h/from [:users :u])
    (h/join [:posts :p] [:= :u.id :p.user_id])
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:u.name :p.title]
             :from   [[:users :u]]
             :join   [[:posts :p] [:= :u.id :p.user_id]]})
```

### 生成されるSQL
```sql
SELECT u.name, p.title FROM users AS u INNER JOIN posts AS p ON u.id = p.user_id
```

## 2. LEFT JOIN（左外部結合）

### ヘルパー関数アプローチ
```clojure
(-> (h/select :u.name :p.title)
    (h/from [:users :u])
    (h/left-join [:posts :p] [:= :u.id :p.user_id])
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:u.name :p.title]
             :from   [[:users :u]]
             :left-join [[:posts :p] [:= :u.id :p.user_id]]})
```

### 生成されるSQL
```sql
SELECT u.name, p.title FROM users AS u LEFT JOIN posts AS p ON u.id = p.user_id
```

## 3. RIGHT JOIN（右外部結合）

### ヘルパー関数アプローチ
```clojure
(-> (h/select :u.name :p.title)
    (h/from [:users :u])
    (h/right-join [:posts :p] [:= :u.id :p.user_id])
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:u.name :p.title]
             :from   [[:users :u]]
             :right-join [[:posts :p] [:= :u.id :p.user_id]]})
```

### 生成されるSQL
```sql
SELECT u.name, p.title FROM users AS u RIGHT JOIN posts AS p ON u.id = p.user_id
```

## 4. FULL OUTER JOIN（完全外部結合）

### ヘルパー関数アプローチ
```clojure
(-> (h/select :u.name :p.title)
    (h/from [:users :u])
    (h/full-join [:posts :p] [:= :u.id :p.user_id])
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:u.name :p.title]
             :from   [[:users :u]]
             :full-join [[:posts :p] [:= :u.id :p.user_id]]})
```

### 生成されるSQL
```sql
SELECT u.name, p.title FROM users AS u FULL JOIN posts AS p ON u.id = p.user_id
```

## 5. 複数テーブルのJOIN

### ヘルパー関数アプローチ
```clojure
(-> (h/select :u.name :p.title :c.content)
    (h/from [:users :u])
    (h/join [:posts :p] [:= :u.id :p.user_id])
    (h/join [:comments :c] [:= :p.id :c.post_id])
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:u.name :p.title :c.content]
             :from   [[:users :u]]
             :join   [[:posts :p] [:= :u.id :p.user_id]
                      [:comments :c] [:= :p.id :c.post_id]]})
```

### 生成されるSQL
```sql
SELECT u.name, p.title, c.content FROM users AS u INNER JOIN posts AS p ON u.id = p.user_id INNER JOIN comments AS c ON p.id = c.post_id
```

## 6. 複数条件でのJOIN

### ヘルパー関数アプローチ
```clojure
(-> (h/select :u.name :p.title)
    (h/from [:users :u])
    (h/join [:posts :p] [:and
                         [:= :u.id :p.user_id]
                         [:= :p.status "published"]])
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:u.name :p.title]
             :from   [[:users :u]]
             :join   [[:posts :p] [:and
                                   [:= :u.id :p.user_id]
                                   [:= :p.status "published"]]]})
```

### 生成されるSQL
```sql
SELECT u.name, p.title FROM users AS u INNER JOIN posts AS p ON (u.id = p.user_id) AND (p.status = ?)
```

## 7. CROSS JOIN（クロス結合）

### ヘルパー関数アプローチ
```clojure
(-> (h/select :s.name :c.name)
    (h/from [:students :s])
    (h/cross-join [:courses :c])
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:s.name :c.name]
             :from   [[:students :s]]
             :cross-join [[:courses :c]]})
```

### 生成されるSQL
```sql
SELECT s.name, c.name FROM students AS s CROSS JOIN courses AS c
```

## 8. サブクエリとのJOIN

### ヘルパー関数アプローチ
```clojure
(-> (h/select :u.name :recent_posts.post_count)
    (h/from [:users :u])
    (h/join [[:select :user_id [[:count :*] :post_count]]
             [:from :posts]
             [:where [:> :created_at "2023-01-01"]]
             [:group-by :user_id]] :recent_posts]
            [:= :u.id :recent_posts.user_id])
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:u.name :recent_posts.post_count]
             :from   [[:users :u]]
             :join   [[{:select [:user_id [[:count :*] :post_count]]
                        :from   [:posts]
                        :where  [:> :created_at "2023-01-01"]
                        :group-by [:user_id]} :recent_posts]
                      [:= :u.id :recent_posts.user_id]]})
```

### 生成されるSQL
```sql
SELECT u.name, recent_posts.post_count FROM users AS u INNER JOIN (SELECT user_id, COUNT(*) AS post_count FROM posts WHERE created_at > ? GROUP BY user_id) AS recent_posts ON u.id = recent_posts.user_id
```

## 9. LATERAL JOIN（PostgreSQL固有）

LATERAL JOINは、右側のテーブル式が左側の行を参照できる特殊なJOINです。

### ヘルパー関数アプローチ
```clojure
(-> (h/select :u.name :latest_post.title)
    (h/from [:users :u])
    (h/join-lateral [[:select :title :created_at]
                     [:from :posts]
                     [:where [:= :user_id :u.id]]
                     [:order-by [:created_at :desc]]
                     [:limit 1]] :latest_post]
                    [:= 1 1])  ; 常に真の条件
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:u.name :latest_post.title]
             :from   [[:users :u]]
             :join-lateral [[{:select [:title :created_at]
                              :from   [:posts]
                              :where  [:= :user_id :u.id]
                              :order-by [[:created_at :desc]]
                              :limit  1} :latest_post]
                            [:= 1 1]]})
```

### 生成されるSQL
```sql
SELECT u.name, latest_post.title FROM users AS u INNER JOIN LATERAL (SELECT title, created_at FROM posts WHERE user_id = u.id ORDER BY created_at DESC LIMIT ?) AS latest_post ON ? = ?
```

## 10. LEFT JOIN LATERAL

### ヘルパー関数アプローチ
```clojure
(-> (h/select :u.name :latest_post.title)
    (h/from [:users :u])
    (h/left-join-lateral [[:select :title]
                          [:from :posts]
                          [:where [:= :user_id :u.id]]
                          [:order-by [:created_at :desc]]
                          [:limit 1]] :latest_post]
                         [:= 1 1])
    sql/format)
```

### 生成されるSQL
```sql
SELECT u.name, latest_post.title FROM users AS u LEFT JOIN LATERAL (SELECT title FROM posts WHERE user_id = u.id ORDER BY created_at DESC LIMIT ?) AS latest_post ON ? = ?
```

## 11. 自己結合（Self JOIN）

### ヘルパー関数アプローチ
```clojure
(-> (h/select :e.name :m.name)
    (h/from [:employees :e])
    (h/left-join [:employees :m] [:= :e.manager_id :m.id])
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:e.name :m.name]
             :from   [[:employees :e]]
             :left-join [[:employees :m] [:= :e.manager_id :m.id]]})
```

### 生成されるSQL
```sql
SELECT e.name, m.name FROM employees AS e LEFT JOIN employees AS m ON e.manager_id = m.id
```

## 12. JOINとWHERE句の組み合わせ

### ヘルパー関数アプローチ
```clojure
(-> (h/select :u.name :p.title)
    (h/from [:users :u])
    (h/join [:posts :p] [:= :u.id :p.user_id])
    (h/where [:= :u.status "active"]
             [:> :p.created_at "2023-01-01"])
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:u.name :p.title]
             :from   [[:users :u]]
             :join   [[:posts :p] [:= :u.id :p.user_id]]
             :where  [:and
                      [:= :u.status "active"]
                      [:> :p.created_at "2023-01-01"]]})
```

### 生成されるSQL
```sql
SELECT u.name, p.title FROM users AS u INNER JOIN posts AS p ON u.id = p.user_id WHERE (u.status = ?) AND (p.created_at > ?)
```

## 13. 集計とJOINの組み合わせ

### ヘルパー関数アプローチ
```clojure
(-> (h/select :u.name [[:count :p.id] :post_count])
    (h/from [:users :u])
    (h/left-join [:posts :p] [:= :u.id :p.user_id])
    (h/group-by :u.id :u.name)
    (h/order-by [:post_count :desc])
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:u.name [[:count :p.id] :post_count]]
             :from   [[:users :u]]
             :left-join [[:posts :p] [:= :u.id :p.user_id]]
             :group-by [:u.id :u.name]
             :order-by [[:post_count :desc]]})
```

### 生成されるSQL
```sql
SELECT u.name, COUNT(p.id) AS post_count FROM users AS u LEFT JOIN posts AS p ON u.id = p.user_id GROUP BY u.id, u.name ORDER BY post_count DESC
```

## 演習問題

1. `customers`テーブルと`orders`テーブルをJOINして、すべての顧客と彼らの注文情報（注文がない顧客も含む）を取得するクエリを作成してください。

2. `products`、`order_items`、`orders`の3つのテーブルをJOINして、各商品の総売上額を計算するクエリを作成してください。

3. `employees`テーブルを自己結合して、各従業員とその直属の上司の名前を取得するクエリを作成してください。

4. LATERAL JOINを使って、各ユーザーの最新の3つの投稿を取得するクエリを作成してください。

## 次のステップ

次の章では、集計関数とGROUP BY句を使ったデータの集約方法を学習します。