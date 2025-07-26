# 第1章: 基本的なSELECT文

## 概要

HoneySQLでの最も基本的なSELECT文の作成方法を学びます。HoneySQLは2つの主要なアプローチでSQLを構築できます：

1. **ヘルパー関数アプローチ**: `select`, `from`などの関数を使用
2. **キーワードマップアプローチ**: Clojureのマップ形式でSQL句を記述

どちらの方法でも同じSQLが生成されますが、ヘルパー関数の方が関数型プログラミングスタイルでチェイン可能です。

## 必要なrequire文

```clojure
(require '[honey.sql :as sql]
         '[honey.sql.helpers :as h])
```

## 1. 最も単純なSELECT文

### ヘルパー関数アプローチ
```clojure
(-> (h/select :*)
    (h/from :users)
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:*]
             :from   [:users]})
```

### 生成されるSQL
```sql
SELECT * FROM users
```

## 2. 特定のカラムを選択

### ヘルパー関数アプローチ
```clojure
(-> (h/select :id :name :email)
    (h/from :users)
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:id :name :email]
             :from   [:users]})
```

### 生成されるSQL
```sql
SELECT id, name, email FROM users
```

## 3. カラムの別名（エイリアス）

### ヘルパー関数アプローチ
```clojure
(-> (h/select [:id :user_id] [:name :user_name] :email)
    (h/from :users)
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [[:id :user_id] [:name :user_name] :email]
             :from   [:users]})
```

### 生成されるSQL
```sql
SELECT id AS user_id, name AS user_name, email FROM users
```

## 4. テーブルの別名

### ヘルパー関数アプローチ
```clojure
(-> (h/select :u.id :u.name :u.email)
    (h/from [:users :u])
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:u.id :u.name :u.email]
             :from   [[:users :u]]})
```

### 生成されるSQL
```sql
SELECT u.id, u.name, u.email FROM users AS u
```

## 5. 計算式とリテラル値

### ヘルパー関数アプローチ
```clojure
(-> (h/select :name 
              [:age :current_age]
              [[:+ :age 5] :age_plus_five]
              [[:concat :first_name " " :last_name] :full_name])
    (h/from :users)
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:name 
                      [:age :current_age]
                      [[:+ :age 5] :age_plus_five]
                      [[:concat :first_name " " :last_name] :full_name]]
             :from   [:users]})
```

### 生成されるSQL
```sql
SELECT name, age AS current_age, age + ? AS age_plus_five, CONCAT(first_name, ?, last_name) AS full_name FROM users
```

**注意**: HoneySQLは値を自動的にパラメータ化するため、リテラル値は`?`として表示され、実際の値は戻り値の2番目の要素に含まれます。

## 6. DISTINCT

### ヘルパー関数アプローチ
```clojure
(-> (h/select-distinct :department)
    (h/from :employees)
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select-distinct [:department]
             :from [:employees]})
```

### 生成されるSQL
```sql
SELECT DISTINCT department FROM employees
```

## 7. 複数のテーブルから選択（暗黙のCROSS JOIN）

### ヘルパー関数アプローチ
```clojure
(-> (h/select :u.name :p.title)
    (h/from :users :posts)
    sql/format)
```

### キーワードマップアプローチ
```clojure
(sql/format {:select [:u.name :p.title]
             :from   [:users :posts]})
```

### 生成されるSQL
```sql
SELECT u.name, p.title FROM users, posts
```

## 実行結果の例

HoneySQLの`format`関数は、SQLクエリ文字列とパラメータのベクタを返します：

```clojure
;; 戻り値の例
["SELECT name, age + ? FROM users" [5]]
```

最初の要素がSQL文字列、2番目の要素がパラメータです。実際のデータベース実行時には、これらのパラメータが安全にバインドされます。

## 演習問題

1. `products`テーブルから商品名（`name`）と価格（`price`）を選択するクエリを、両方のアプローチで書いてください。

2. `employees`テーブルから、従業員ID（`id`を`emp_id`として）と フルネーム（`first_name`と`last_name`を連結して`full_name`として）を選択するクエリを作成してください。

3. `orders`テーブル（別名`o`）から、重複のない顧客ID（`customer_id`）を選択するクエリを書いてください。

## 次のステップ

次の章では、WHERE句を使ったデータのフィルタリング方法を学習します。