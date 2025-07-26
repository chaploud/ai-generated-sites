# 第12章: 高度なDELETE操作

## 概要

この章では、基本的なDELETE文を超えた高度な削除操作を学習します。JOINを使った削除、サブクエリによる削除、CTEとの組み合わせ、ソフトデリート、カスケード削除、条件付き削除など、実用的なパターンを網羅します。

## 1. JOINを使ったDELETE

### 基本的なJOIN DELETE

#### ヘルパー関数アプローチ
```clojure
(-> (h/delete-from :order_items)
    (h/using :orders)
    (h/where [:and
              [:= :order_items.order_id :orders.id]
              [:= :orders.status "cancelled"]
              [:< :orders.created_at "2023-01-01"]])
    sql/format)
```

#### キーワードマップアプローチ
```clojure
(sql/format {:delete-from :order_items
             :using [:orders]
             :where [:and
                     [:= :order_items.order_id :orders.id]
                     [:= :orders.status "cancelled"]
                     [:< :orders.created_at "2023-01-01"]]})
```

#### 生成されるSQL
```sql
DELETE FROM order_items USING orders WHERE (order_items.order_id = orders.id) AND (orders.status = ?) AND (orders.created_at < ?)
```

### 複数テーブルとのJOIN DELETE

#### ヘルパー関数アプローチ
```clojure
(-> (h/delete-from :user_sessions)
    (h/using [:users :u] [:user_devices :ud])
    (h/where [:and
              [:= :user_sessions.user_id :u.id]
              [:= :user_sessions.device_id :ud.id]
              [:= :u.status "inactive"]
              [:= :ud.trusted false]
              [:< :user_sessions.last_activity "2023-01-01"]])
    sql/format)
```

#### 生成されるSQL
```sql
DELETE FROM user_sessions USING users AS u, user_devices AS ud WHERE (user_sessions.user_id = u.id) AND (user_sessions.device_id = ud.id) AND (u.status = ?) AND (ud.trusted = ?) AND (user_sessions.last_activity < ?)
```

## 2. サブクエリを使ったDELETE

### IN句でのサブクエリ削除

#### ヘルパー関数アプローチ
```clojure
(-> (h/delete-from :audit_logs)
    (h/where [:in :user_id [:select :id
                            [:from :users]
                            [:where [:and
                                     [:= :status "deleted"]
                                     [:< :deleted_at "2022-01-01"]]]]])
    sql/format)
```

#### 生成されるSQL
```sql
DELETE FROM audit_logs WHERE user_id IN (SELECT id FROM users WHERE (status = ?) AND (deleted_at < ?))
```

### EXISTS句でのサブクエリ削除

#### ヘルパー関数アプローチ
```clojure
(-> (h/delete-from :temporary_files)
    (h/where [:and
              [:< :created_at [:- [:now] [:interval "24 hours"]]]
              [:not-exists [:select 1
                            [:from :active_processes]
                            [:where [:= :active_processes.temp_file_id :temporary_files.id]]]]])
    sql/format)
```

#### 生成されるSQL
```sql
DELETE FROM temporary_files WHERE (created_at < NOW() - INTERVAL ?) AND (NOT EXISTS (SELECT ? FROM active_processes WHERE active_processes.temp_file_id = temporary_files.id))
```

### 相関サブクエリでの削除

#### ヘルパー関数アプローチ
```clojure
(-> (h/delete-from :duplicate_emails)
    (h/where [:> :id [:select [[:min :id]]
                      [:from :duplicate_emails :de2]
                      [:where [:= :de2.email :duplicate_emails.email]]]])
    sql/format)
```

#### 生成されるSQL
```sql
DELETE FROM duplicate_emails WHERE id > (SELECT MIN(id) FROM duplicate_emails AS de2 WHERE de2.email = duplicate_emails.email)
```

## 3. CTEを使ったDELETE

### 複雑な条件による削除

#### ヘルパー関数アプローチ
```clojure
(-> (h/with [:inactive_users (-> (h/select :id :email)
                                 (h/from :users)
                                 (h/where [:and
                                           [:< :last_login "2022-01-01"]
                                           [:= :email_verified false]]))
             :related_data (-> (h/select [:distinct :user_id])
                               (h/from :user_activities)
                               (h/where [:in :user_id [:select :id [:from :inactive_users]]])))
    (h/delete-from :users)
    (h/where [:and
              [:in :id [:select :id [:from :inactive_users]]]
              [:not-in :id [:select :user_id [:from :related_data]]]])
    (h/returning :id :email)
    sql/format)
```

#### 生成されるSQL
```sql
WITH inactive_users AS (SELECT id, email FROM users WHERE (last_login < ?) AND (email_verified = ?)), related_data AS (SELECT DISTINCT user_id FROM user_activities WHERE user_id IN (SELECT id FROM inactive_users)) DELETE FROM users WHERE (id IN (SELECT id FROM inactive_users)) AND (id NOT IN (SELECT user_id FROM related_data)) RETURNING id, email
```

## 4. ソフトデリート

### 論理削除の実装

#### ヘルパー関数アプローチ
```clojure
(-> (h/update :users)
    (h/set {:deleted_at [:now]
            :status "deleted"
            :email [:concat :email "_deleted_" [:extract "epoch" [:now]]]})
    (h/where [:and
              [:= :id 123]
              [:is :deleted_at nil]])
    (h/returning :id :email :deleted_at)
    sql/format)
```

#### 生成されるSQL
```sql
UPDATE users SET deleted_at = NOW(), status = ?, email = CONCAT(email, ?, EXTRACT(EPOCH FROM NOW())) WHERE (id = ?) AND (deleted_at IS NULL) RETURNING id, email, deleted_at
```

### ソフトデリートされたデータの物理削除

#### ヘルパー関数アプローチ
```clojure
(-> (h/delete-from :users)
    (h/where [:and
              [:is-not :deleted_at nil]
              [:< :deleted_at [:- [:now] [:interval "1 year"]]]])
    (h/returning :id :email :deleted_at)
    sql/format)
```

#### 生成されるSQL
```sql
DELETE FROM users WHERE (deleted_at IS NOT NULL) AND (deleted_at < NOW() - INTERVAL ?) RETURNING id, email, deleted_at
```

## 5. 条件付きDELETE

### 複雑な業務ルールによる削除

#### ヘルパー関数アプローチ
```clojure
(-> (h/delete-from :shopping_cart_items)
    (h/where [:and
              [:< :created_at [:- [:now] [:interval "30 days"]]]
              [:or
               [:not-exists [:select 1
                             [:from :users]
                             [:where [:and
                                      [:= :users.id :shopping_cart_items.user_id]
                                      [:= :users.status "active"]]]]]
               [:not-exists [:select 1
                             [:from :products]
                             [:where [:and
                                      [:= :products.id :shopping_cart_items.product_id]
                                      [:= :products.available true]]]]]]])
    sql/format)
```

#### 生成されるSQL
```sql
DELETE FROM shopping_cart_items WHERE (created_at < NOW() - INTERVAL ?) AND ((NOT EXISTS (SELECT ? FROM users WHERE (users.id = shopping_cart_items.user_id) AND (users.status = ?))) OR (NOT EXISTS (SELECT ? FROM products WHERE (products.id = shopping_cart_items.product_id) AND (products.available = ?))))
```

## 6. カスケード削除の実装

### 手動カスケード削除

#### ヘルパー関数アプローチ
```clojure
(defn cascade-delete-user [user-id]
  [
   ;; 1. 関連するコメント削除
   (-> (h/delete-from :comments)
       (h/where [:= :user_id user-id])
       sql/format)
   
   ;; 2. 投稿削除
   (-> (h/delete-from :posts)
       (h/where [:= :author_id user-id])
       sql/format)
   
   ;; 3. セッション削除
   (-> (h/delete-from :user_sessions)
       (h/where [:= :user_id user-id])
       sql/format)
   
   ;; 4. プロフィール削除
   (-> (h/delete-from :user_profiles)
       (h/where [:= :user_id user-id])
       sql/format)
   
   ;; 5. 最後にユーザー削除
   (-> (h/delete-from :users)
       (h/where [:= :id user-id])
       (h/returning :id :email)
       sql/format)
  ])
```

### 依存関係チェック付き削除

#### ヘルパー関数アプローチ
```clojure
(-> (h/delete-from :categories)
    (h/where [:and
              [:= :id 123]
              [:not-exists [:select 1
                            [:from :products]
                            [:where [:= :category_id 123]]]]
              [:not-exists [:select 1
                            [:from :categories :c]
                            [:where [:= :c.parent_id 123]]]]])
    (h/returning :id :name)
    sql/format)
```

#### 生成されるSQL
```sql
DELETE FROM categories WHERE (id = ?) AND (NOT EXISTS (SELECT ? FROM products WHERE category_id = ?)) AND (NOT EXISTS (SELECT ? FROM categories AS c WHERE c.parent_id = ?)) RETURNING id, name
```

## 7. パーティション対応の削除

### 日付パーティションからの削除

#### ヘルパー関数アプローチ
```clojure
(-> (h/delete-from :logs_2023_01)
    (h/where [:and
              [:>= :created_at "2023-01-01"]
              [:< :created_at "2023-02-01"]
              [:= :log_level "DEBUG"]])
    sql/format)
```

### 古いパーティションの一括削除

#### ヘルパー関数アプローチ
```clojure
(-> (h/delete-from :audit_logs)
    (h/where [:< :created_at [:- [:now] [:interval "2 years"]]])
    sql/format)
```

## 8. 一括削除の最適化

### バッチ削除

#### ヘルパー関数アプローチ
```clojure
(defn batch-delete-old-records [table date-column cutoff-date batch-size]
  (-> (h/delete-from table)
      (h/where [:and
                [:< date-column cutoff-date]
                [:in :id [:select :id
                          [:from table]
                          [:where [:< date-column cutoff-date]]
                          [:limit batch-size]]]])
      sql/format))

;; 使用例
(batch-delete-old-records :old_logs :created_at "2022-01-01" 10000)
```

### 削除前のデータバックアップ

#### ヘルパー関数アプローチ
```clojure
[
  ;; 1. バックアップテーブルにデータをコピー
  (-> (h/insert-into :deleted_orders_backup)
      (h/select :*)
      (h/from :orders)
      (h/where [:< :created_at "2022-01-01"])
      sql/format)
  
  ;; 2. 元テーブルから削除
  (-> (h/delete-from :orders)
      (h/where [:< :created_at "2022-01-01"])
      (h/returning [[:count :*] :deleted_count])
      sql/format)
]
```

## 9. トランザクション内でのDELETE

### 安全な削除処理

#### ヘルパー関数アプローチ
```clojure
(defn safe-delete-user-data [user-id]
  [
   ;; 1. ユーザー存在確認
   (-> (h/select [:exists [:select 1 [:from :users] [:where [:= :id user-id]]]] :user_exists)
       sql/format)
   
   ;; 2. 関連データ数確認
   (-> (h/select [[:count :*] :related_count])
       (h/from :user_activities)
       (h/where [:= :user_id user-id])
       sql/format)
   
   ;; 3. 条件に応じた削除
   (-> (h/delete-from :user_activities)
       (h/where [:and
                 [:= :user_id user-id]
                 [:< :created_at "2023-01-01"]])
       (h/returning [[:count :*] :deleted_count])
       sql/format)
  ])
```

## 10. 削除操作の監査

### 削除ログの記録

#### ヘルパー関数アプローチ
```clojure
(-> (h/with [:deleted_records (-> (h/delete-from :sensitive_data)
                                  (h/where [:< :created_at "2022-01-01"])
                                  (h/returning :id :created_at :data_type))]
    (h/insert-into :deletion_audit)
    (h/select :id 
              :data_type
              [:inline "sensitive_data"] :table_name
              [:inline "automated_cleanup"] :deletion_reason
              [:now] :deleted_at
              [:current_user] :deleted_by)
    (h/from :deleted_records)
    sql/format)
```

#### 生成されるSQL
```sql
WITH deleted_records AS (DELETE FROM sensitive_data WHERE created_at < ? RETURNING id, created_at, data_type) INSERT INTO deletion_audit SELECT id, data_type, ?, ?, NOW(), CURRENT_USER FROM deleted_records
```

## 11. JSON/配列フィールドの部分削除

### JSON要素の削除

#### ヘルパー関数アプローチ
```clojure
(-> (h/update :user_preferences)
    (h/set {:settings [:- :settings "deprecated_feature"]
            :updated_at [:now]})
    (h/where [:? :settings "deprecated_feature"])
    sql/format)
```

### 配列要素の削除

#### ヘルパー関数アプローチ
```clojure
(-> (h/update :article_tags)
    (h/set {:tags [:array_remove :tags "obsolete_tag"]})
    (h/where [:@> :tags ["obsolete_tag"]])
    sql/format)
```

## 12. 実践的な例：データ保持ポリシー

### 階層的データ保持

#### ヘルパー関数アプローチ
```clojure
(-> (h/with [:retention_policy (h/values [["error_logs" "7 days"]
                                          ["debug_logs" "1 day"]
                                          ["info_logs" "30 days"]
                                          ["audit_logs" "7 years"]] 
                                         [:log_type :retention_period])
             :expired_logs (-> (h/select :l.id :l.log_type :l.created_at)
                               (h/from [:application_logs :l])
                               (h/join [:retention_policy :rp] [:= :l.log_type :rp.log_type])
                               (h/where [:< :l.created_at 
                                         [:- [:now] [:cast [:concat :rp.retention_period] :interval]]]))]
    (h/delete-from :application_logs)
    (h/where [:in :id [:select :id [:from :expired_logs]]])
    (h/returning [[:count :*] :deleted_count])
    sql/format)
```

## 演習問題

1. `orders`テーブルから、関連する`order_items`がすべて削除されている注文レコードを削除するクエリを作成してください。

2. ソフトデリートを実装し、削除から1年経過したユーザーデータを物理削除するクエリを作成してください。

3. CTEを使って、重複する商品データから最新のもの以外をすべて削除するクエリを作成してください。

4. カスケード削除を実装し、プロジェクト削除時に関連するタスク、コメント、添付ファイルもすべて削除する一連のクエリを作成してください。

## 次のステップ

次の章では、DDL操作（CREATE TABLE、ALTER TABLE、DROP TABLE等）について学習します。