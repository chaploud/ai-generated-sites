# HoneySQLで学ぶPostgreSQLクエリ構築ガイド

HoneySQL 2.xを使ってPostgreSQLクエリを構築するための日本語学習教材です。基本的なSELECT文から高度な再帰クエリ、PostgreSQL固有機能まで、段階的に学習できるように構成されています。

## リンク

- [保守性の高いSQLコンポーネント戦略](14-dop-component.pdf)

## 📚 学習コース

### 基礎編 - データ取得（READ）

#### [第1章: 基本的なSELECT文](01-basic-select.md)
- HoneySQLの基本的な使い方
- ヘルパー関数とキーワードマップの2つのアプローチ
- カラム選択、エイリアス、テーブル指定
- 計算式とリテラル値の扱い
- DISTINCT句の使用

**学習目標**: HoneySQLの基本的な文法を理解し、シンプルなSELECT文を作成できるようになる

#### [第2章: WHERE句とフィルタリング](02-where-filtering.md)
- 基本的な条件演算子（=, >, <, >=, <=）
- 論理演算子（AND, OR, NOT）
- IN、NOT IN、LIKE、ILIKE
- BETWEEN、IS NULL、IS NOT NULL
- PostgreSQL固有の演算子（正規表現、配列演算子）
- 動的条件の構築

**学習目標**: 様々な条件を組み合わせて、必要なデータを効率的にフィルタリングできるようになる

#### [第3章: JOIN操作](03-joins.md)
- INNER JOIN、LEFT JOIN、RIGHT JOIN、FULL OUTER JOIN
- CROSS JOIN、LATERAL JOIN
- 複数テーブルの結合
- 自己結合（Self JOIN）
- サブクエリとのJOIN
- JOINとWHERE句の組み合わせ

**学習目標**: 複数のテーブルからデータを適切に結合して取得できるようになる

### 中級編 - 高度なデータ取得と操作

#### [第4章: 集計関数とGROUP BY](04-aggregation-groupby.md)
- 基本集計関数（COUNT, SUM, AVG, MIN, MAX）
- GROUP BY句によるグループ化
- HAVING句による集計結果のフィルタリング
- 複雑な集計例と条件付き集計
- PostgreSQL固有の集計関数（STRING_AGG, ARRAY_AGG, JSON_AGG）
- ROLLUP、CUBE、GROUPING SETS

**学習目標**: データの集約と統計計算を効率的に実行できるようになる

#### [第5章: サブクエリとCTE](05-subqueries-cte.md)
- SELECT句、WHERE句、FROM句でのサブクエリ
- EXISTS、NOT EXISTS、ANY、ALL演算子
- CTE（Common Table Expression）の基本
- 複数のCTEと依存関係
- CTEを使ったデータ変換と段階的処理

**学習目標**: 複雑なビジネスロジックを段階的に分解して記述できるようになる

#### [第6章: UNION/UNION ALL操作](06-union-operations.md)
- UNION、UNION ALL、INTERSECT、EXCEPT
- 複数クエリの結合と並び順
- CTEとUNIONの組み合わせ
- 動的UNION構築
- パフォーマンス考慮事項

**学習目標**: 複数のデータソースを統合したレポートやビューを作成できるようになる

### 上級編 - 高度なクエリ技法

#### [第7章: WITH RECURSIVE（再帰クエリ）](07-recursive-queries.md)
- 再帰CTEの基本構造
- 組織階層の取得
- カテゴリの親子関係処理
- グラフ探索とネットワーク分析
- 循環参照の検出と安全な再帰
- 実用的な階層データ処理

**学習目標**: 階層データや木構造を効率的に処理できるようになる

#### [第8章: PostgreSQL固有の高度な機能](08-postgresql-advanced.md)
- JSON/JSONB操作とパス指定
- 配列操作とUNNEST
- ウィンドウ関数（RANK, LAG, LEAD等）
- Upsert操作（ON CONFLICT）
- 全文検索（Full Text Search）
- 範囲型、PostGIS、統計関数

**学習目標**: PostgreSQLの高度な機能を活用して実践的なアプリケーションを構築できるようになる

### CRUD編 - データ操作の基本と応用

#### [第9章: 基本的なCRUD操作](09-basic-crud.md)
- INSERT文の基本（単一・複数行挿入）
- UPDATE文の基本（条件付き更新）
- DELETE文の基本（条件付き削除）
- RETURNING句の使用
- トランザクションとCRUD

**学習目標**: 基本的なデータ操作（作成・更新・削除）を安全に実行できるようになる

#### [第10章: 高度なINSERT操作](10-advanced-insert.md)
- サブクエリからのINSERT（INSERT INTO ... SELECT）
- Upsert操作（INSERT ON CONFLICT）
- CTEを使った複雑な挿入
- 条件付きINSERT
- 一括データ処理とパフォーマンス最適化
- JSONや配列データの挿入

**学習目標**: 複雑なデータ挿入シナリオに対応できるようになる

#### [第11章: 高度なUPDATE操作](11-advanced-update.md)
- JOINを使った更新
- サブクエリによる更新
- CTEを使った複雑な更新
- 条件付きUPDATE（CASE文）
- JSON/配列フィールドの部分更新
- ウィンドウ関数を使った更新
- 一括更新の最適化

**学習目標**: 複雑なデータ更新ロジックを効率的に実装できるようになる

#### [第12章: 高度なDELETE操作](12-advanced-delete.md)
- JOINを使った削除
- サブクエリを使った削除
- CTEを使った条件付き削除
- ソフトデリートの実装
- カスケード削除とデータ整合性
- 一括削除の最適化
- 削除操作の監査とログ

**学習目標**: 安全で効率的なデータ削除戦略を実装できるようになる

### DDL編 - データベース設計と管理

#### [第13章: DDL操作（Data Definition Language）](13-ddl-operations.md)
- CREATE TABLE（基本・制約・PostgreSQL固有型）
- CREATE INDEX（基本・複合・部分・式・GIN）
- ALTER TABLE（カラム追加・削除・変更・制約）
- DROP操作（テーブル・インデックス・安全な削除）
- パーティショニング
- ビューとマテリアライズドビュー
- トリガーと拡張機能
- スキーマ管理

**学習目標**: データベーススキーマの設計・変更・管理を適切に行えるようになる

## 🚀 学習の進め方

### 1. 環境準備
```clojure
;; project.cljの依存関係
{:deps {com.github.seancorfield/honeysql {:mvn/version "2.4.1045"}
        org.postgresql/postgresql {:mvn/version "42.6.0"}}}

;; REPL で必要なrequire
(require '[honey.sql :as sql]
         '[honey.sql.helpers :as h])
```

### 2. 推奨学習パス

**完全初心者向け**（Clojure初心者またはSQL初心者）:
1. 第1章 → 第2章 → 第3章 → 第9章（基本CRUD）
2. 各章の演習問題を完了
3. 実際のデータベースで動作確認

**データ分析重視**（レポート作成やデータ分析が中心）:
1. 第1章 → 第2章 → 第3章 → 第4章 → 第5章 → 第6章
2. 実際のプロジェクトでCTEやUNIONを活用

**アプリケーション開発者向け**（Webアプリケーション開発）:
1. 第1章 → 第2章 → 第9章 → 第10章 → 第11章 → 第12章
2. CRUD操作を中心に学習
3. 第13章でデータベース設計も学習

**データベース管理者向け**（DBAやインフラエンジニア）:
1. 全章を順序通り学習
2. 特に第13章（DDL）を重点的に学習
3. パフォーマンスチューニングと最適化に注力

**上級者向け**（PostgreSQLの高度な機能を学びたい）:
1. 第7章 → 第8章 → 第10章 → 第11章 → 第12章
2. 実際のデータで再帰クエリやJSON操作を実践

### 3. 各章の構成

各章は以下の構成になっています：

- **概要**: その章で学ぶ内容の説明
- **具体例**: ヘルパー関数とキーワードマップ両方のアプローチ
- **生成されるSQL**: HoneySQLが生成する実際のSQL文
- **実用例**: 実際のアプリケーションで使えるパターン
- **演習問題**: 理解を深めるための課題
- **次のステップ**: 次章への導入

### 4. コード例の見方

すべてのコード例は以下の形式で提供されています：

```clojure
;; ヘルパー関数アプローチ
(-> (h/select :column)
    (h/from :table)
    sql/format)

;; キーワードマップアプローチ
(sql/format {:select [:column]
             :from   [:table]})

;; 生成されるSQL
;; SELECT column FROM table
```

## 💡 学習のコツ

### 1. 両方のアプローチを理解する
HoneySQLには2つの記述方法があります：
- **ヘルパー関数**: 関数型プログラミングスタイル、チェイン可能
- **キーワードマップ**: データ構造として記述、動的構築が容易

どちらも同じSQLを生成しますが、用途に応じて使い分けることが重要です。

### 2. 生成されるSQLを確認する
`sql/format`の結果を常に確認して、期待通りのSQLが生成されているかチェックしましょう。

### 3. REPLで実際に動かす
各例をREPLで実際に実行し、結果を確認することで理解が深まります。

### 4. 実データで練習する
可能であれば実際のデータベースとテーブルを用意して練習することを推奨します。

## 🔧 便利なリファレンス

### よく使う関数

| 機能             | ヘルパー関数     | キーワード      |
| ---------------- | ---------------- | --------------- |
| **SELECT系**     |
| 選択             | `h/select`       | `:select`       |
| テーブル指定     | `h/from`         | `:from`         |
| 条件             | `h/where`        | `:where`        |
| 結合             | `h/join`         | `:join`         |
| グループ化       | `h/group-by`     | `:group-by`     |
| 並び順           | `h/order-by`     | `:order-by`     |
| 制限             | `h/limit`        | `:limit`        |
| **CRUD系**       |
| 挿入             | `h/insert-into`  | `:insert-into`  |
| 更新             | `h/update`       | `:update`       |
| 削除             | `h/delete-from`  | `:delete-from`  |
| 値設定           | `h/values`       | `:values`       |
| フィールド設定   | `h/set`          | `:set`          |
| 戻り値           | `h/returning`    | `:returning`    |
| **DDL系**        |
| テーブル作成     | `h/create-table` | `:create-table` |
| インデックス作成 | `h/create-index` | `:create-index` |
| テーブル変更     | `h/alter-table`  | `:alter-table`  |
| 削除             | `h/drop-table`   | `:drop-table`   |

### PostgreSQL演算子

| 演算子 | 説明             | 例                        |
| ------ | ---------------- | ------------------------- |
| `->`   | JSON要素取得     | `:data -> "key"`          |
| `->>`  | JSONテキスト取得 | `:data ->> "key"`         |
| `@>`   | JSON包含         | `:data @> {:key "value"}` |
| `~`    | 正規表現マッチ   | `:text ~ "pattern"`       |
| `@>`   | 配列包含         | `:tags @> ["tag1"]`       |

## 📖 追加リソース

- [HoneySQLの公式ドキュメント](https://github.com/seancorfield/honeysql)
- [PostgreSQLの公式ドキュメント](https://www.postgresql.org/docs/)
- HoneySQLプロジェクトの`doc/`ディレクトリ内の詳細資料

## 🤝 学習サポート

学習中に疑問があれば：
1. 各章の演習問題に取り組む
2. HoneySQLのテストファイルで実例を確認
3. PostgreSQLの公式ドキュメントで詳細を確認

---

**Happy Learning with HoneSQL! 🍯📚**