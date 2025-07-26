# 第6章: UNION/UNION ALL操作

## 概要

UNION操作は複数のSELECT文の結果を結合するための機能です。UNIONは重複行を除去し、UNION ALLは重複を含むすべての行を返します。HoneySQLでは、これらの操作を簡潔に記述できます。また、INTERSECT（積集合）やEXCEPT（差集合）もサポートしています。

## 1. 基本的なUNION

### ヘルパー関数アプローチ
```clojure
(-> (h/select :name :email)
    (h/from :customers)
    (h/union (-> (h/select :name :email)
                 (h/from :suppliers)))
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:union [{:select [:name :email]
                      :from   [:customers]}
                     {:select [:name :email]
                      :from   [:suppliers]}]})
```

### 生成されるSQL
```sql
SELECT name, email FROM customers UNION SELECT name, email FROM suppliers
```

## 2. UNION ALL（重複を保持）

### ヘルパー関数アプローチ
```clojure
(-> (h/select :name :email)
    (h/from :customers)
    (h/union-all (-> (h/select :name :email)
                     (h/from :suppliers)))
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:union-all [{:select [:name :email]
                          :from   [:customers]}
                         {:select [:name :email]
                          :from   [:suppliers]}]})
```

### 生成されるSQL
```sql
SELECT name, email FROM customers UNION ALL SELECT name, email FROM suppliers
```

## 3. 複数クエリのUNION

### ヘルパー関数アプローチ
```clojure
(-> (h/select :name :email [:inline "customer"] :type)
    (h/from :customers)
    (h/union (-> (h/select :name :email [:inline "supplier"])
                 (h/from :suppliers))
             (-> (h/select :name :email [:inline "employee"])
                 (h/from :employees)))
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:union [{:select [:name :email [:inline "customer"] :type]
                      :from   [:customers]}
                     {:select [:name :email [:inline "supplier"]]
                      :from   [:suppliers]}
                     {:select [:name :email [:inline "employee"]]
                      :from   [:employees]}]})
```

### 生成されるSQL
```sql
SELECT name, email, 'customer' AS type FROM customers UNION SELECT name, email, 'supplier' FROM suppliers UNION SELECT name, email, 'employee' FROM employees
```

## 4. UNIONでの並び順

### ヘルパー関数アプローチ
```clojure
(-> (h/select :name :created_at)
    (h/from :posts)
    (h/where [:= :status "published"])
    (h/union-all (-> (h/select :name :created_at)
                     (h/from :pages)
                     (h/where [:= :published true])))
    (h/order-by [:created_at :desc])
    (h/limit 10)
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:union-all [{:select [:name :created_at]
                          :from   [:posts]
                          :where  [:= :status "published"]}
                         {:select [:name :created_at]
                          :from   [:pages]
                          :where  [:= :published true]}]
             :order-by [[:created_at :desc]]
             :limit 10})
```

### 生成されるSQL
```sql
SELECT name, created_at FROM posts WHERE status = ? UNION ALL SELECT name, created_at FROM pages WHERE published = ? ORDER BY created_at DESC LIMIT ?
```

## 5. 異なるデータ型の統合

### ヘルパー関数アプローチ
```clojure
(-> (h/select [:cast :user_id :text] :id
              :username :name
              [:inline "user"] :type)
    (h/from :users)
    (h/union-all (-> (h/select [:cast :company_id :text]
                               :company_name
                               [:inline "company"])
                     (h/from :companies)))
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:union-all [{:select [[:cast :user_id :text] :id
                                   :username :name
                                   [:inline "user"] :type]
                          :from   [:users]}
                         {:select [[:cast :company_id :text]
                                   :company_name
                                   [:inline "company"]]
                          :from   [:companies]}]})
```

### 生成されるSQL
```sql
SELECT CAST(user_id AS TEXT) AS id, username AS name, 'user' AS type FROM users UNION ALL SELECT CAST(company_id AS TEXT), company_name, 'company' FROM companies
```

## 6. CTEとUNIONの組み合わせ

### ヘルパー関数アプローチ
```clojure
(-> (h/with [:recent_orders (-> (h/select :customer_id :order_date :amount)
                                (h/from :orders)
                                (h/where [:> :order_date "2023-01-01"]))
             :refunds (-> (h/select :customer_id :refund_date :amount)
                          (h/from :refunds)
                          (h/where [:> :refund_date "2023-01-01"]))]
    (h/select :customer_id :order_date :amount [:inline "order"] :transaction_type)
    (h/from :recent_orders)
    (h/union-all (-> (h/select :customer_id :refund_date [[:* :amount -1]] [:inline "refund"])
                     (h/from :refunds)))
    (h/order-by :customer_id :order_date)
    sql/format)
```

### 生成されるSQL
```sql
WITH recent_orders AS (SELECT customer_id, order_date, amount FROM orders WHERE order_date > ?), refunds AS (SELECT customer_id, refund_date, amount FROM refunds WHERE refund_date > ?) SELECT customer_id, order_date, amount, 'order' AS transaction_type FROM recent_orders UNION ALL SELECT customer_id, refund_date, amount * ?, 'refund' FROM refunds ORDER BY customer_id, order_date
```

## 7. INTERSECT（積集合）

### ヘルパー関数アプローチ
```clojure
(-> (h/select :email)
    (h/from :customers)
    (h/intersect (-> (h/select :email)
                     (h/from :newsletter_subscribers)))
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:intersect [{:select [:email]
                          :from   [:customers]}
                         {:select [:email]
                          :from   [:newsletter_subscribers]}]})
```

### 生成されるSQL
```sql
SELECT email FROM customers INTERSECT SELECT email FROM newsletter_subscribers
```

## 8. EXCEPT/MINUS（差集合）

### ヘルパー関数アプローチ
```clojure
(-> (h/select :email)
    (h/from :customers)
    (h/except (-> (h/select :email)
                  (h/from :unsubscribed_emails)))
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:except [{:select [:email]
                       :from   [:customers]}
                      {:select [:email]
                       :from   [:unsubscribed_emails]}]})
```

### 生成されるSQL
```sql
SELECT email FROM customers EXCEPT SELECT email FROM unsubscribed_emails
```

## 9. 複雑なUNION例：レポート作成

### 売上サマリーレポート

#### ヘルパー関数アプローチ
```clojure
(-> (h/select [:inline "Daily Sales"] :report_type
              [[:to_char :order_date "YYYY-MM-DD"] :period]
              [[:sum :amount] :total])
    (h/from :orders)
    (h/where [:>= :order_date "2023-01-01"])
    (h/group-by :order_date)
    (h/union-all 
      (-> (h/select [:inline "Monthly Sales"]
                    [[:to_char :order_date "YYYY-MM"] :period]
                    [[:sum :amount] :total])
          (h/from :orders)
          (h/where [:>= :order_date "2023-01-01"])
          (h/group-by [[:to_char :order_date "YYYY-MM"]]))
      (-> (h/select [:inline "Yearly Sales"]
                    [[:to_char :order_date "YYYY"] :period]
                    [[:sum :amount] :total])
          (h/from :orders)
          (h/where [:>= :order_date "2023-01-01"])
          (h/group-by [[:to_char :order_date "YYYY"]])))
    (h/order-by :report_type :period)
    sql/format)
```

### 生成されるSQL
```sql
SELECT 'Daily Sales' AS report_type, TO_CHAR(order_date, ?) AS period, SUM(amount) AS total FROM orders WHERE order_date >= ? GROUP BY order_date UNION ALL SELECT 'Monthly Sales', TO_CHAR(order_date, ?) AS period, SUM(amount) AS total FROM orders WHERE order_date >= ? GROUP BY TO_CHAR(order_date, ?) UNION ALL SELECT 'Yearly Sales', TO_CHAR(order_date, ?) AS period, SUM(amount) AS total FROM orders WHERE order_date >= ? GROUP BY TO_CHAR(order_date, ?) ORDER BY report_type, period
```

## 10. UNION と集計の組み合わせ

### ヘルパー関数アプローチ
```clojure
(-> (h/with [:all_transactions (h/union-all 
                                 (-> (h/select :customer_id :amount [:inline "sale"] :type)
                                     (h/from :sales))
                                 (-> (h/select :customer_id [[:* :amount -1]] [:inline "refund"])
                                     (h/from :refunds)))])
    (h/select :customer_id 
              [[:sum [:case [:= :type "sale"] :amount :else 0]] :total_sales]
              [[:sum [:case [:= :type "refund"] :amount :else 0]] :total_refunds]
              [[:sum :amount] :net_amount])
    (h/from :all_transactions)
    (h/group-by :customer_id)
    (h/having [:> [:sum :amount] 0])
    sql/format)
```

### 生成されるSQL
```sql
WITH all_transactions AS (SELECT customer_id, amount, 'sale' AS type FROM sales UNION ALL SELECT customer_id, amount * ?, 'refund' FROM refunds) SELECT customer_id, SUM(CASE WHEN type = ? THEN amount ELSE ? END) AS total_sales, SUM(CASE WHEN type = ? THEN amount ELSE ? END) AS total_refunds, SUM(amount) AS net_amount FROM all_transactions GROUP BY customer_id HAVING SUM(amount) > ?
```

## 11. UNION ALLのパフォーマンス考慮

### ヘルパー関数アプローチ（大量データの結合）
```clojure
(-> (h/select :id :name :created_at [:inline "active"] :status)
    (h/from :active_users)
    (h/union-all (-> (h/select :id :name :created_at [:inline "inactive"])
                     (h/from :inactive_users)))
    (h/where [:> :created_at "2023-01-01"])  ; UNIONの後でフィルタ
    (h/order-by [:created_at :desc])
    (h/limit 100)
    sql/format)
```

### 生成されるSQL
```sql
SELECT id, name, created_at, 'active' AS status FROM active_users UNION ALL SELECT id, name, created_at, 'inactive' FROM inactive_users WHERE created_at > ? ORDER BY created_at DESC LIMIT ?
```

## 12. 動的UNION構築

### 実行時にUNIONクエリを動的に構築
```clojure
(defn build-multi-table-search [tables search-term]
  (let [queries (map (fn [table]
                       (-> (h/select :id :name [:inline (name table)] :source)
                           (h/from table)
                           (h/where [:ilike :name (str "%" search-term "%")])))
                     tables)]
    (reduce h/union-all (first queries) (rest queries))))

;; 使用例
(sql/format (build-multi-table-search [:customers :suppliers :employees] "john"))
```

## 13. ページネーション付きUNION

### ヘルパー関数アプローチ
```clojure
(-> (h/with [:combined_results (h/union-all
                                 (-> (h/select :id :title :created_at [:inline "post"] :type)
                                     (h/from :posts)
                                     (h/where [:= :published true]))
                                 (-> (h/select :id :title :created_at [:inline "page"])
                                     (h/from :pages)
                                     (h/where [:= :published true])))]
    (h/select :*)
    (h/from :combined_results)
    (h/order-by [:created_at :desc])
    (h/offset 20)
    (h/limit 10)
    sql/format)
```

### 生成されるSQL
```sql
WITH combined_results AS (SELECT id, title, created_at, 'post' AS type FROM posts WHERE published = ? UNION ALL SELECT id, title, created_at, 'page' FROM pages WHERE published = ?) SELECT * FROM combined_results ORDER BY created_at DESC OFFSET ? LIMIT ?
```

## 演習問題

1. `customers`テーブルと`prospects`テーブルから、すべての連絡先（名前とメールアドレス）を重複なしで取得するクエリを作成してください。

2. 過去3ヶ月の売上データと返品データを結合し、各顧客の正味売上（売上 - 返品）を計算するクエリを作成してください。

3. `products`、`services`、`subscriptions`の3つのテーブルから、名前に特定のキーワードを含むアイテムをすべて検索するクエリを作成してください。

4. INTERSECTを使って、両方のメーリングリスト（`newsletter_a`と`newsletter_b`）に登録されているメールアドレスを取得するクエリを作成してください。

## 次のステップ

次の章では、WITH RECURSIVEを使った再帰クエリの構築方法を学習します。