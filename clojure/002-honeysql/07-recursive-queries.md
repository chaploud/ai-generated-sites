# 第7章: WITH RECURSIVE（再帰クエリ）

## 概要

WITH RECURSIVEは、階層データや木構造、グラフ構造を扱うための強力な機能です。従業員の組織階層、カテゴリの親子関係、フォルダ構造など、自己参照する関係を持つデータを効率的に処理できます。HoneySQLではPostgreSQLのWITH RECURSIVE構文を完全にサポートしています。

## 1. 基本的な再帰クエリの構造

再帰CTEは以下の3つの部分で構成されます：
1. **初期クエリ（Anchor）**: 再帰の開始点
2. **UNION/UNION ALL**: 初期クエリと再帰部分を結合
3. **再帰クエリ（Recursive）**: 自分自身を参照するクエリ

### 基本的な数列生成

#### ヘルパー関数アプローチ
```clojure
(-> (h/with-recursive [:numbers (h/union-all
                                  (h/select [:inline 1] :n)  ; 初期値
                                  (-> (h/select [[:+ :n 1]])  ; 再帰部分
                                      (h/from :numbers)
                                      (h/where [:<= :n 10])))])
    (h/select :n)
    (h/from :numbers)
    sql/format)
```

#### キーワードマップアプローチ
```clojure
(sql/format {:with-recursive [[:numbers {:union-all [{:select [[:inline 1] :n]}
                                                     {:select [[[:+ :n 1]]]
                                                      :from   [:numbers]
                                                      :where  [:<= :n 10]}]}]]
             :select [:n]
             :from   [:numbers]})
```

#### 生成されるSQL
```sql
WITH RECURSIVE numbers AS (SELECT ? AS n UNION ALL SELECT n + ? FROM numbers WHERE n <= ?) SELECT n FROM numbers
```

## 2. 組織階層の取得

### 特定の上司の配下すべてを取得

#### ヘルパー関数アプローチ
```clojure
(-> (h/with-recursive [:subordinates (h/union-all
                                       ;; 初期クエリ：直接の部下
                                       (-> (h/select :id :name :manager_id :level)
                                           (h/from :employees)
                                           (h/where [:= :manager_id 1]))
                                       ;; 再帰クエリ：部下の部下
                                       (-> (h/select :e.id :e.name :e.manager_id [[:+ :s.level 1] :level])
                                           (h/from [:employees :e])
                                           (h/join [:subordinates :s] [:= :e.manager_id :s.id])))])
    (h/select :id :name :manager_id :level)
    (h/from :subordinates)
    (h/order-by :level :name)
    sql/format)
```

#### キーワードマップアプローチ
```clojure
(sql/format {:with-recursive [[:subordinates 
                               {:union-all 
                                [{:select [:id :name :manager_id :level]
                                  :from   [:employees]
                                  :where  [:= :manager_id 1]}
                                 {:select [:e.id :e.name :e.manager_id [[:+ :s.level 1] :level]]
                                  :from   [[:employees :e]]
                                  :join   [[:subordinates :s] [:= :e.manager_id :s.id]]}]}]]
             :select [:id :name :manager_id :level]
             :from   [:subordinates]
             :order-by [:level :name]})
```

#### 生成されるSQL
```sql
WITH RECURSIVE subordinates AS (SELECT id, name, manager_id, level FROM employees WHERE manager_id = ? UNION ALL SELECT e.id, e.name, e.manager_id, s.level + ? AS level FROM employees AS e INNER JOIN subordinates AS s ON e.manager_id = s.id) SELECT id, name, manager_id, level FROM subordinates ORDER BY level, name
```

## 3. 上位階層の取得（逆方向）

### 特定の従業員から経営陣までのパスを取得

#### ヘルパー関数アプローチ
```clojure
(-> (h/with-recursive [:management_chain (h/union-all
                                           ;; 初期クエリ：起点となる従業員
                                           (-> (h/select :id :name :manager_id [:inline 0] :level)
                                               (h/from :employees)
                                               (h/where [:= :id 15]))
                                           ;; 再帰クエリ：上司を辿る
                                           (-> (h/select :e.id :e.name :e.manager_id [[:+ :mc.level 1] :level])
                                               (h/from [:employees :e])
                                               (h/join [:management_chain :mc] [:= :e.id :mc.manager_id])))])
    (h/select :id :name :level)
    (h/from :management_chain)
    (h/order-by :level)
    sql/format)
```

#### 生成されるSQL
```sql
WITH RECURSIVE management_chain AS (SELECT id, name, manager_id, ? AS level FROM employees WHERE id = ? UNION ALL SELECT e.id, e.name, e.manager_id, mc.level + ? AS level FROM employees AS e INNER JOIN management_chain AS mc ON e.id = mc.manager_id) SELECT id, name, level FROM management_chain ORDER BY level
```

## 4. カテゴリ階層の処理

### 製品カテゴリの親子関係

#### ヘルパー関数アプローチ
```clojure
(-> (h/with-recursive [:category_tree (h/union-all
                                        ;; ルートカテゴリ
                                        (-> (h/select :id :name :parent_id [:inline 0] :depth [:name :path])
                                            (h/from :categories)
                                            (h/where [:is :parent_id nil]))
                                        ;; 子カテゴリ
                                        (-> (h/select :c.id :c.name :c.parent_id [[:+ :ct.depth 1] :depth]
                                                      [[:concat :ct.path " > " :c.name] :path])
                                            (h/from [:categories :c])
                                            (h/join [:category_tree :ct] [:= :c.parent_id :ct.id])))])
    (h/select :id :name :depth :path)
    (h/from :category_tree)
    (h/order-by :path)
    sql/format)
```

#### 生成されるSQL
```sql
WITH RECURSIVE category_tree AS (SELECT id, name, parent_id, ? AS depth, name AS path FROM categories WHERE parent_id IS NULL UNION ALL SELECT c.id, c.name, c.parent_id, ct.depth + ? AS depth, CONCAT(ct.path, ?, c.name) AS path FROM categories AS c INNER JOIN category_tree AS ct ON c.parent_id = ct.id) SELECT id, name, depth, path FROM category_tree ORDER BY path
```

## 5. グラフの探索（ネットワーク分析）

### 友達の友達を探索（ソーシャルネットワーク）

#### ヘルパー関数アプローチ
```clojure
(-> (h/with-recursive [:friend_network (h/union-all
                                         ;; 直接の友達
                                         (-> (h/select :friend_id :user_id [:inline 1] :degree)
                                             (h/from :friendships)
                                             (h/where [:= :user_id 123]))
                                         ;; 友達の友達（3度まで）
                                         (-> (h/select :f.friend_id :fn.user_id [[:+ :fn.degree 1] :degree])
                                             (h/from [:friendships :f])
                                             (h/join [:friend_network :fn] [:= :f.user_id :fn.friend_id])
                                             (h/where [:<= :fn.degree 2])))])
    (h/select :friend_id :degree)
    (h/from :friend_network)
    (h/where [:!= :friend_id 123])  ; 自分自身を除外
    (h/order-by :degree :friend_id)
    sql/format)
```

#### 生成されるSQL
```sql
WITH RECURSIVE friend_network AS (SELECT friend_id, user_id, ? AS degree FROM friendships WHERE user_id = ? UNION ALL SELECT f.friend_id, fn.user_id, fn.degree + ? AS degree FROM friendships AS f INNER JOIN friend_network AS fn ON f.user_id = fn.friend_id WHERE fn.degree <= ?) SELECT friend_id, degree FROM friend_network WHERE friend_id != ? ORDER BY degree, friend_id
```

## 6. フィボナッチ数列の生成

#### ヘルパー関数アプローチ
```clojure
(-> (h/with-recursive [:fibonacci (h/union-all
                                    ;; 初期値
                                    (h/select [:inline 1] :n [:inline 0] :fib_n [:inline 1] :fib_n_plus_1)
                                    ;; 再帰計算
                                    (-> (h/select [[:+ :n 1]]
                                                  :fib_n_plus_1
                                                  [[:+ :fib_n :fib_n_plus_1]])
                                        (h/from :fibonacci)
                                        (h/where [:<= :n 20])))])
    (h/select :n :fib_n)
    (h/from :fibonacci)
    sql/format)
```

#### 生成されるSQL
```sql
WITH RECURSIVE fibonacci AS (SELECT ? AS n, ? AS fib_n, ? AS fib_n_plus_1 UNION ALL SELECT n + ?, fib_n_plus_1, fib_n + fib_n_plus_1 FROM fibonacci WHERE n <= ?) SELECT n, fib_n FROM fibonacci
```

## 7. ディレクトリ構造の探索

### ファイルシステムの階層表示

#### ヘルパー関数アプローチ
```clojure
(-> (h/with-recursive [:directory_tree (h/union-all
                                         ;; ルートディレクトリ
                                         (-> (h/select :id :name :parent_id [:inline 0] :level 
                                                       [:name :full_path]
                                                       [[:case [:= :type "file"] :size :else 0] :total_size])
                                             (h/from :filesystem)
                                             (h/where [:is :parent_id nil]))
                                         ;; 子要素
                                         (-> (h/select :fs.id :fs.name :fs.parent_id [[:+ :dt.level 1] :level]
                                                       [[:concat :dt.full_path "/" :fs.name] :full_path]
                                                       [[:case [:= :fs.type "file"] :fs.size :else 0] :total_size])
                                             (h/from [:filesystem :fs])
                                             (h/join [:directory_tree :dt] [:= :fs.parent_id :dt.id])))])
    (h/select :id :name :level :full_path
              [[:sum :total_size] :directory_size {:over [:partition-by :full_path]}])
    (h/from :directory_tree)
    (h/order-by :full_path)
    sql/format)
```

#### 生成されるSQL
```sql
WITH RECURSIVE directory_tree AS (SELECT id, name, parent_id, ? AS level, name AS full_path, CASE WHEN type = ? THEN size ELSE ? END AS total_size FROM filesystem WHERE parent_id IS NULL UNION ALL SELECT fs.id, fs.name, fs.parent_id, dt.level + ? AS level, CONCAT(dt.full_path, ?, fs.name) AS full_path, CASE WHEN fs.type = ? THEN fs.size ELSE ? END AS total_size FROM filesystem AS fs INNER JOIN directory_tree AS dt ON fs.parent_id = dt.id) SELECT id, name, level, full_path, SUM(total_size) OVER (PARTITION BY full_path) AS directory_size FROM directory_tree ORDER BY full_path
```

## 8. 循環参照の検出

### 無限ループを防ぐ安全な再帰

#### ヘルパー関数アプローチ
```clojure
(-> (h/with-recursive [:safe_hierarchy (h/union-all
                                         ;; 初期クエリ
                                         (-> (h/select :id :name :manager_id 
                                                       [[:array :id] :path]
                                                       [:inline 0] :depth)
                                             (h/from :employees)
                                             (h/where [:= :id 1]))
                                         ;; 循環参照チェック付き再帰
                                         (-> (h/select :e.id :e.name :e.manager_id
                                                       [[:array_append :sh.path :e.id] :path]
                                                       [[:+ :sh.depth 1] :depth])
                                             (h/from [:employees :e])
                                             (h/join [:safe_hierarchy :sh] [:= :e.manager_id :sh.id])
                                             (h/where [:and 
                                                       [:<= :sh.depth 10]  ; 最大深度制限
                                                       [:not [:= [:any :sh.path] :e.id]]])))])  ; 循環検出
    (h/select :id :name :depth :path)
    (h/from :safe_hierarchy)
    (h/order-by :depth :id)
    sql/format)
```

#### 生成されるSQL
```sql
WITH RECURSIVE safe_hierarchy AS (SELECT id, name, manager_id, ARRAY[id] AS path, ? AS depth FROM employees WHERE id = ? UNION ALL SELECT e.id, e.name, e.manager_id, ARRAY_APPEND(sh.path, e.id) AS path, sh.depth + ? AS depth FROM employees AS e INNER JOIN safe_hierarchy AS sh ON e.manager_id = sh.id WHERE (sh.depth <= ?) AND (NOT (? = ANY(sh.path)))) SELECT id, name, depth, path FROM safe_hierarchy ORDER BY depth, id
```

## 9. 複雑な階層集計

### 各部門の総予算計算（子部門を含む）

#### ヘルパー関数アプローチ
```clojure
(-> (h/with-recursive [:dept_hierarchy (h/union-all
                                         ;; リーフ部門（子がない部門）
                                         (-> (h/select :id :name :parent_id :budget [:budget :total_budget])
                                             (h/from :departments)
                                             (h/where [:not-exists [:select 1
                                                                    [:from :departments :d2]
                                                                    [:where [:= :d2.parent_id :departments.id]]]]))
                                         ;; 親部門（子部門の予算を含む）
                                         (-> (h/select :d.id :d.name :d.parent_id :d.budget
                                                       [[:+ :d.budget [:sum :dh.total_budget]] :total_budget])
                                             (h/from [:departments :d])
                                             (h/join [:dept_hierarchy :dh] [:= :dh.parent_id :d.id])
                                             (h/group-by :d.id :d.name :d.parent_id :d.budget)))])
    (h/select :id :name :budget :total_budget)
    (h/from :dept_hierarchy)
    (h/order-by :total_budget)
    sql/format)
```

#### 生成されるSQL
```sql
WITH RECURSIVE dept_hierarchy AS (SELECT id, name, parent_id, budget, budget AS total_budget FROM departments WHERE NOT EXISTS (SELECT ? FROM departments AS d2 WHERE d2.parent_id = departments.id) UNION ALL SELECT d.id, d.name, d.parent_id, d.budget, d.budget + SUM(dh.total_budget) AS total_budget FROM departments AS d INNER JOIN dept_hierarchy AS dh ON dh.parent_id = d.id GROUP BY d.id, d.name, d.parent_id, d.budget) SELECT id, name, budget, total_budget FROM dept_hierarchy ORDER BY total_budget
```

## 10. 実用的な例：掲示板のスレッド表示

### ネストしたコメントの階層表示

#### ヘルパー関数アプローチ
```clojure
(-> (h/with-recursive [:comment_thread (h/union-all
                                         ;; トップレベルコメント
                                         (-> (h/select :id :content :author :parent_id :created_at
                                                       [:inline 0] :depth
                                                       [[:cast :id :text] :sort_path])
                                             (h/from :comments)
                                             (h/where [:and 
                                                       [:= :post_id 123]
                                                       [:is :parent_id nil]]))
                                         ;; 返信コメント
                                         (-> (h/select :c.id :c.content :c.author :c.parent_id :c.created_at
                                                       [[:+ :ct.depth 1] :depth]
                                                       [[:concat :ct.sort_path "-" [:cast :c.id :text]] :sort_path])
                                             (h/from [:comments :c])
                                             (h/join [:comment_thread :ct] [:= :c.parent_id :ct.id])
                                             (h/where [:= :c.post_id 123])))])
    (h/select :id :content :author :depth :sort_path :created_at)
    (h/from :comment_thread)
    (h/order-by :sort_path)
    sql/format)
```

#### 生成されるSQL
```sql
WITH RECURSIVE comment_thread AS (SELECT id, content, author, parent_id, created_at, ? AS depth, CAST(id AS TEXT) AS sort_path FROM comments WHERE (post_id = ?) AND (parent_id IS NULL) UNION ALL SELECT c.id, c.content, c.author, c.parent_id, c.created_at, ct.depth + ? AS depth, CONCAT(ct.sort_path, ?, CAST(c.id AS TEXT)) AS sort_path FROM comments AS c INNER JOIN comment_thread AS ct ON c.parent_id = ct.id WHERE c.post_id = ?) SELECT id, content, author, depth, sort_path, created_at FROM comment_thread ORDER BY sort_path
```

## 演習問題

1. `categories`テーブルで、特定のカテゴリとそのすべての子カテゴリ（孫カテゴリも含む）を取得する再帰クエリを作成してください。

2. `employees`テーブルで、特定の従業員から最上位の経営陣までの管理ライン（管理者の階層）を取得するクエリを作成してください。

3. `regions`テーブル（親子関係を持つ地域データ）で、各地域の総人口（子地域の人口も含む）を計算する再帰クエリを作成してください。

4. `task_dependencies`テーブル（タスクの依存関係）で、特定のタスクが完了するために必要なすべての前提タスクを取得する再帰クエリを作成してください。

## 次のステップ

次の章では、PostgreSQL固有の高度な機能（JSON操作、配列、ウィンドウ関数など）を学習します。