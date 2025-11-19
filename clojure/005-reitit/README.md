# Reitit完全ガイド：基礎から実践まで

[reititキーワード集](./01_keywords.md)

## reitit（ライティット）とは何か

**reitit**は、Clojure/ClojureScript向けの高速でデータ駆動型のルーティングライブラリです。Metosin社が開発し、バックエンド（JVM上のRing）とフロントエンド（ブラウザ上のClojureScript）の両方で動作する統一されたルーティングソリューションを提供します。

### reitit の主な特徴

**データファースト設計** により、ルートは純粋なデータ構造（ベクタとマップ）として定義されます。これにより検査、変換、バリデーション、プログラムによる生成が容易になります。

**高性能** を実現するため、reitit はルートツリーを解析し、自動的に最適なルーターアルゴリズムを選択します。公式ベンチマークによれば、他のClojureルーティングライブラリより桁違いに高速で、Go言語の最速ルーターであるhttprouterと比較しても2倍未満の遅延に抑えられています。

**双方向ルーティング** により、パスからルートへのマッチングと、ルート名からパスの生成の両方が可能です。

**ファーストクラスのルートデータ** により、任意のマップデータをルートに添付でき、親から子へとメタマージされます。

**プラガブルなコアーション（型変換）** により、clojure.spec、Malli、Plumatic Schemaをサポートし、リクエスト/レスポンスパラメータの自動変換とバリデーションが可能です。

**ルート競合解決** により、ルーター作成時にルートが互いにマスクし合う問題を自動的に検出し、早期にエラーを出します。

### なぜreitit を選ぶべきか

既存のルーティングライブラリと比較した場合の利点：

**Compojureとの比較**：Compojureはマクロを使用しますが、reitit はデータを使用します。Compojure はパスの前にHTTPメソッドでディスパッチしますが、reitit はまずルートでマッチングします。Compojure は双方向ルーティングをサポートせず、ClojureScriptでも動作しませんが、reitit は両方をサポートします。

**Bidiとの比較**：reitit はBidiができることをすべて実行でき、しかも大幅に高速です。さらに、コアーション、バリデーションなど、より多くの機能を提供します。

### バージョン情報と要件

- **最新安定版**：0.9.1
- **要件**：Clojure 1.11+、Java 11+
- **ライセンス**：Eclipse Public License
- **コミュニティ**：Clojurians Slack の #reitit チャンネル

---

## 基本的なセットアップ

### インストール

全モジュールを含むバージョン（最も一般的）：

```clojure
;; deps.edn
{:deps {metosin/reitit {:mvn/version "0.9.1"}}}

;; または project.clj (Leiningen)
[metosin/reitit "0.9.1"]
```

### モジュール別インストール

より焦点を絞った使用のため、reitit は部分的にインストールできます：

```clojure
;; Ring Webアプリケーション用
{:deps {metosin/reitit-ring {:mvn/version "0.9.1"}}}

;; コアーション（Malli推奨）
{:deps {metosin/reitit-malli {:mvn/version "0.9.1"}}}

;; Swagger UI統合
{:deps {metosin/reitit-swagger-ui {:mvn/version "0.9.1"}}}
```

### Hello World - 最初のルーター

最もシンプルなルーターの例：

```clojure
(require '[reitit.core :as r])

;; ルーターを作成
(def router
  (r/router
    [["

/api/ping" ::ping]
     ["/api/orders/:id" ::order-by-id]]))

;; パスでマッチング（順方向ルーティング）
(r/match-by-path router "/api/ping")
; => #Match{:template "/api/ping"
;           :data {:name ::ping}
;           :path-params {}
;           :path "/api/ping"}

;; 名前でマッチング（逆方向ルーティング）
(r/match-by-name router ::order-by-id {:id 2})
; => #Match{:template "/api/orders/:id"
;           :path "/api/orders/2"
;           :path-params {:id 2}}
```

### Ring との統合 - 基本的なWebアプリケーション

```clojure
(require '[reitit.ring :as ring])

(defn handler [request]
  {:status 200
   :body "Hello, reitit!"})

(def app
  (ring/ring-handler
    (ring/router
      [["/"]
       {:get handler
        :name ::home}]
      [["/ping"
        {:get handler
         :name ::ping}]])))

;; テスト
(app {:request-method :get, :uri "/"})
; => {:status 200, :body "Hello, reitit!"}

(app {:request-method :get, :uri "/ping"})
; => {:status 200, :body "Hello, reitit!"}
```

### Jettyサーバーで起動

```clojure
(require '[ring.adapter.jetty :as jetty])

(jetty/run-jetty app {:port 3000, :join? false})
;; http://localhost:3000 にアクセス
```

---

## ルート構文とパターン

### 基本的なルート構造

reitit では、ルートは文字列パスとオプションのルート引数または子ルートを含むベクタとして定義されます：

```clojure
;; シンプルなルート
["/ping"]

;; 複数のルート
[["/ping"]
 ["/pong"]]

;; ルート引数を持つルート
[["/ping" ::ping]
 ["/pong" {:name ::pong}]]
```

### パスパラメータ - 2つの構文

reitit はパスパラメータに2つの構文をサポートします：

**コロン構文**（`:param`）：
```clojure
[["/users/:user-id"]
 ["/api/:version/ping"]
 ["/files/file-:number"]]
```

**ブラケット構文**（`{param}`）：
```clojure
[["/users/{user-id}"]
 ["/files/file-{number}.pdf"]
 ["/files/{name}.{extension}"]]
```

ブラケット構文は修飾キーワードをサポートします：
```clojure
[["/users/{user/id}"]
 ["/tasks/{project/id}/{task/id}"]]
```

**デフォルトの動作**：両方の構文がデフォルトでサポートされています。

### キャッチオールパラメータ（ワイルドカード）

パスの残りをキャプチャする場合：

```clojure
;; コロン構文
["/public/*path" {:handler get-file}]

;; ブラケット構文
["/public/{*path}" {:handler get-file}]

;; 例：
;; /public/images/logo.png にマッチし
;; :path は "images/logo.png" になります
```

### ネストされたルート

パスプレフィックスとルートデータを共有するため、ルートをネストできます：

```clojure
["/api"
 ["/admin" {:middleware [::admin]}
  ["" {:name ::admin}]           ;; /api/admin 用の空パス
  ["/db" {:name ::db}]]
 ["/ping" {:name ::ping}]]

;; 次のようにフラット化されます：
; ["/api/admin" {:middleware [::admin], :name ::admin}]
; ["/api/admin/db" {:middleware [::admin], :name ::db}]
; ["/api/ping" {:name ::ping}]
```

### パスパラメータの抽出

```clojure
(def router
  (r/router
    [["/users/:id" {:name ::user}]]))

(r/match-by-path router "/users/123")
; => #Match{:path-params {:id "123"}}

;; ハンドラー内で使用
(defn user-handler [{:keys [path-params]}]
  (let [id (:id path-params)]
    {:status 200
     :body (str "User ID: " id)}))
```

### 複数のパラメータ

```clojure
[["/projects/:project-id/tasks/:task-id"
  {:name ::task
   :get (fn [{:keys [path-params]}]
          (let [{:keys [project-id task-id]} path-params]
            {:status 200
             :body (str "Project: " project-id ", Task: " task-id)}))}]]

(r/match-by-path router "/projects/42/tasks/99")
; => #Match{:path-params {:project-id "42", :task-id "99"}}
```

### 逆方向ルーティング - URLの生成

名前付きルートからURLを生成：

```clojure
(def router
  (r/router
    [["/api/users/:id" {:name ::user}]]))

;; 名前とパラメータからパスを生成
(-> router
    (r/match-by-name ::user {:id 123})
    (r/match->path))
; => "/api/users/123"

;; クエリパラメータ付き
(-> router
    (r/match-by-name ::user {:id 123})
    (r/match->path {:detailed true}))
; => "/api/users/123?detailed=true"
```

### ルート定義のプログラム的生成

ルートはデータなので、プログラムで生成できます：

```clojure
(defn crud-routes [resource]
  [(str "/" resource)
   {:name (keyword resource "list")
    :get list-handler}
   [(str "/" resource "/:id")
    {:name (keyword resource "detail")
     :get get-handler
     :put update-handler
     :delete delete-handler}]])

(def router
  (r/router
    [(crud-routes "users")
     (crud-routes "orders")
     (crud-routes "products")]))
```

---

## ルートデータとミドルウェア

### ルートデータの基礎

ルートデータは reitit の重要な特徴です。任意のマップ状のデータをルートに添付できます：

```clojure
[["/ping" {:name ::ping}]
 ["/pong" {:handler identity}]
 ["/admin" {:roles #{:admin}
            :audit-log true
            :handler admin-handler}]]
```

### ルート引数の展開

非シーケンシャルなルート引数は、ルーターによって自動的に展開されます：

```clojure
;; キーワードは :name キーに展開
[["/ping" ::ping]]  ; => {:name ::ping}

;; 関数は :handler キーに展開
[["/pong" identity]]  ; => {:handler identity}

;; マップはそのまま
[["/users" {:get {:roles #{:admin}
                  :handler identity}}]]
```

### メタマージの動作

ネストされたルートの場合、データはルートから葉へ **メタマージ** を使用して蓄積されます：

```clojure
(def router
  (r/router
    ["/api" {:interceptors [::api]}
     ["/ping" ::ping]
     ["/admin" {:roles #{:admin}}
      ["/users" ::users]
      ["/db" {:interceptors [::db]
              :roles ^:replace #{:db-admin}}]]]))

;; 結果：
; ["/api/ping" {:interceptors [::api], :name ::ping}]
; ["/api/admin/users" {:interceptors [::api], :roles #{:admin}, :name ::users}]
; ["/api/admin/db" {:interceptors [::api ::db], :roles #{:db-admin}}]
```

`^:replace` メタデータを使用すると、マージではなく置換されます。

### ミドルウェアの基本

ミドルウェアはハンドラーをラップする関数です：`handler → request → response`

```clojure
(defn wrap-logging [handler]
  (fn [request]
    (println "Request:" (:uri request))
    (handler request)))

(def app
  (ring/ring-handler
    (ring/router
      [["/api" {:middleware [wrap-logging]}
        ["/ping" {:get handler}]]])))
```

### ルート固有 vs グローバルミドルウェア

**ルート固有のミドルウェア**：

```clojure
[["/api" {:middleware [[wrap :api]]}
  ["/admin" {:middleware [[wrap :admin]]}
   ["/users" {:get handler}]]]]

;; /api/admin/users へのリクエストでは：
;; :api → :admin → handler の順で実行
```

**ルーターレベルのミドルウェア**：

```clojure
(ring/router
  routes
  {:data {:middleware [wrap-session wrap-params]}})
```

**トップレベルのミドルウェア**：

```clojure
(ring/ring-handler
  (ring/router routes)
  nil
  {:middleware [wrap-cors]})
```

### HTTPメソッド固有のミドルウェア

```clojure
[["/users"
  {:get {:middleware [read-only-middleware]
         :handler list-users}
   :post {:middleware [validation-middleware]
          :handler create-user}}]]
```

### データ駆動型ミドルウェア

ミドルウェアをマップまたはレコードとして定義できます：

```clojure
(def wrap-roles
  {:name ::roles
   :wrap (fn [handler]
           (fn [{:keys [roles] :as request}]
             (let [required (-> request
                               ring/get-match
                               :data
                               :roles)]
               (if (and (seq required)
                       (not (set/subset? required roles)))
                 {:status 403, :body "Forbidden"}
                 (handler request)))))})

[["/admin" {:middleware [wrap-roles]
            :roles #{:admin}}
  ["/users" {:get list-users}]]]
```

### コンパイル済みミドルウェア

`:compile` キーを使用すると、ルーター作成時にミドルウェアをコンパイルできます：

```clojure
(def enforce-roles-middleware
  {:name ::enforce-roles
   :spec (s/keys :req-un [::roles])
   :compile (fn [{required :roles} _]
              (when (seq required)
                {:description (str "Requires roles " required)
                 :wrap (fn [handler]
                         (fn [{:keys [roles] :as request}]
                           (if (not (set/subset? required roles))
                             {:status 403, :body "forbidden"}
                             (handler request))))}))})
```

**利点**：
- ルート作成時に一度だけコンパイル
- 必要な場合のみミドルウェアをマウント
- リクエスト処理時の作業が少なく、高速

### 共通ミドルウェアパターン

**パラメータ処理**：

```clojure
(require '[reitit.ring.middleware.parameters :as parameters])

{:data {:middleware [parameters/parameters-middleware]}}
```

**例外処理**：

```clojure
(require '[reitit.ring.middleware.exception :as exception])

{:data {:middleware [exception/exception-middleware]}}
```

**コンテンツネゴシエーション**：

```clojure
(require '[muuntaja.core :as m])
(require '[reitit.ring.middleware.muuntaja :as muuntaja])

{:data {:muuntaja m/instance
        :middleware [muuntaja/format-middleware]}}
```

---

## コアーション（型変換とバリデーション）

### コアーションとは

コアーションは、パラメータとレスポンスをある形式から別の形式に変換するプロセスです。デフォルトでは、パスパラメータは文字列として解析されますが、コアーションにより定義されたスキーマに基づいて自動的な型変換とバリデーションが可能になります。

### サポートされているコアーションライブラリ

reitit は3つのコアーションモジュールを提供します：

1. **reitit.coercion.malli** - Malliスキーマ用（**推奨**）
2. **reitit.coercion.schema** - Plumatic Schema用
3. **reitit.coercion.spec** - clojure.spec用

### リクエストコアーション

パラメータは `:parameters` キーの下に定義されます：

| タイプ | ソース | 説明 |
|--------|--------|------|
| `:query` | `:query-params` | クエリ文字列パラメータ |
| `:body` | `:body-params` | リクエストボディ |
| `:form` | `:form-params` | フォームデータ |
| `:header` | `:header-params` | HTTPヘッダー |
| `:path` | `:path-params` | URLパスパラメータ |
| `:multipart` | `:multipart-params` | ファイルアップロード |

### Malliを使った完全な例

```clojure
(require '[reitit.ring :as ring])
(require '[reitit.coercion.malli])
(require '[reitit.ring.coercion :as rrc])
(require '[reitit.ring.middleware.parameters :as parameters])
(require '[reitit.ring.middleware.muuntaja :as muuntaja])
(require '[muuntaja.core :as m])

(def app
  (ring/ring-handler
    (ring/router
      ["/api"
       ["/math"
        {:get {:summary "2つの数を足す"
               :parameters {:query [:map
                                   [:x :int]
                                   [:y :int]]}
               :responses {200 {:body [:map [:total :int]]}}
               :handler (fn [{{{:keys [x y]} :query} :parameters}]
                         {:status 200
                          :body {:total (+ x y)}})}}]]
      {:data {:coercion reitit.coercion.malli/coercion
              :muuntaja m/instance
              :middleware [parameters/parameters-middleware
                          muuntaja/format-middleware
                          rrc/coerce-exceptions-middleware
                          rrc/coerce-request-middleware
                          rrc/coerce-response-middleware]}})))

;; 有効なリクエスト
(app {:request-method :get
      :uri "/api/math"
      :query-params {"x" "1", "y" "2"}})
; => {:status 200, :body {:total 3}}

;; 無効なリクエスト
(app {:request-method :get
      :uri "/api/math"
      :query-params {"x" "1", "y" "abc"}})
; => {:status 400, :body {:errors [...], :type :reitit.coercion/request-coercion}}
```

### clojure.specを使った例

```clojure
(require '[reitit.coercion.spec])
(require '[clojure.spec.alpha :as s])

(s/def ::x int?)
(s/def ::y int?)
(s/def ::query-params (s/keys :req-un [::x ::y]))

(def app
  (ring/ring-handler
    (ring/router
      ["/api"
       ["/math"
        {:get {:parameters {:query ::query-params}
               :responses {200 {:body (s/keys :req-un [::total])}}
               :handler (fn [{{{:keys [x y]} :query} :parameters}]
                         {:status 200
                          :body {:total (+ x y)}})}}]]
      {:data {:coercion reitit.coercion.spec/coercion
              :middleware [parameters/parameters-middleware
                          muuntaja/format-middleware
                          rrc/coerce-exceptions-middleware
                          rrc/coerce-request-middleware
                          rrc/coerce-response-middleware]}})))
```

### Schemaを使った例

```clojure
(require '[reitit.coercion.schema])
(require '[schema.core :as s])

(def PositiveInt (s/constrained s/Int pos? 'PositiveInt))

(def app
  (ring/ring-handler
    (ring/router
      ["/api"
       ["/plus/:z"
        {:post {:coercion reitit.coercion.schema/coercion
                :parameters {:query {:x s/Int}
                            :body {:y s/Int}
                            :path {:z s/Int}}
                :responses {200 {:body {:total PositiveInt}}}
                :handler (fn [{:keys [parameters]}]
                          (let [total (+ (-> parameters :query :x)
                                       (-> parameters :body :y)
                                       (-> parameters :path :z))]
                            {:status 200
                             :body {:total total}}))}}]]
      {:data {:middleware [parameters/parameters-middleware
                          muuntaja/format-middleware
                          rrc/coerce-exceptions-middleware
                          rrc/coerce-request-middleware
                          rrc/coerce-response-middleware]}})))
```

### レスポンスコアーション

レスポンスは `:responses` キーの下に、HTTPステータスコードとスキーマのマップとして定義されます：

```clojure
:responses {200 {:body [:map [:total :int]]}
            404 {:body [:map [:error :string]]}
            :default {:body [:map [:error :string]]}}
```

### コンテンツタイプ別のコアーション

`:body` の代わりに `:request` を使用すると、コンテンツタイプごとのスキーマを定義できます：

```clojure
["/example"
 {:post {:coercion reitit.coercion.malli/coercion
         :request {:content {"application/json" {:schema [:map [:y :int]]}
                            "application/edn" {:schema [:map [:z :int]]}
                            :default {:schema [:map [:yy :int]]}}}
         :responses {200 {:content {"application/json" {:schema [:map [:w :int]]}
                                    "application/edn" {:schema [:map [:x :int]]}
                                    :default {:schema [:map [:ww :int]]}}}}
         :handler ...}}]
```

### コアーションの実用的なパターン

**カスタム型定義**：

```clojure
;; Malli
(def UserId [:int {:min 1}])
(def Email [:re #".+@.+\..+"])

[["/users/:id"
  {:parameters {:path [:map [:id UserId]]
                :query [:map [:email {:optional true} Email]]}}]]

;; Spec
(s/def ::user-id (s/and int? pos?))
(s/def ::email (s/and string? #(re-matches #".+@.+\..+" %)))

[["/users/:id"
  {:parameters {:path (s/keys :req-un [::user-id])
                :query (s/keys :opt-un [::email])}}]]
```

**複雑なネストされたスキーマ**：

```clojure
(def Address
  [:map
   [:street :string]
   [:city :string]
   [:zip [:re #"\d{5}"]]])

(def User
  [:map
   [:id :int]
   [:name :string]
   [:email Email]
   [:address Address]])

[["/users"
  {:post {:parameters {:body User}
          :responses {201 {:body User}}
          :handler create-user-handler}}]]
```

---

## API ドキュメント生成

### Swagger 2 サポート

reitit は、ルート定義、コアーションパラメータ、ドキュメントキーから Swagger 2 ドキュメントを抽出できます。

### OpenAPI 3.1.0 サポート（推奨）

OpenAPI 3.1.0 は Swagger 2 よりも強化された機能を提供します：

- 完全なJSON Schemaサポート
- `oneOf`、`anyOf`、`allOf` 演算子
- リクエスト/レスポンスのコンテンツタイプごとのスキーマ
- パラメータの複数の名前付き例
- 強化された認証サポート

### OpenAPI の完全な実装例

```clojure
(require '[reitit.ring :as ring])
(require '[reitit.openapi :as openapi])
(require '[reitit.swagger-ui :as swagger-ui])
(require '[reitit.coercion.malli])
(require '[reitit.ring.coercion :as rrc])

(def app
  (ring/ring-handler
    (ring/router
      [["/openapi.json"
        {:get {:no-doc true
               :openapi {:info {:title "My API"
                               :description "reitit + OpenAPI 3"
                               :version "1.0.0"}
                        :tags [{:name "users"
                               :description "ユーザー管理API"}
                              {:name "products"
                               :description "商品管理API"}]}
               :handler (openapi/create-openapi-handler)}}]

       ["/swagger-ui/*"
        {:get {:no-doc true
               :handler (swagger-ui/create-swagger-ui-handler
                        {:url "/openapi.json"
                         :config {:validator-url nil}})}}]

       ["/api"
        ["/users"
         {:tags ["users"]}
         [""
          {:get {:summary "ユーザー一覧を取得"
                 :responses {200 {:body [:vector [:map
                                                 [:id :int]
                                                 [:name :string]
                                                 [:email :string]]]}}
                 :handler list-users}
           :post {:summary "新しいユーザーを作成"
                  :description "新しいユーザーを作成し、作成されたユーザー情報を返します"
                  :parameters {:body [:map
                                     [:name :string]
                                     [:email [:re #".+@.+\..+"]]
                                     [:age {:optional true} [:int {:min 0 :max 150}]]]}
                  :responses {201 {:body [:map
                                         [:id :int]
                                         [:name :string]
                                         [:email :string]
                                         [:age {:optional true} :int]]}}
                  :handler create-user}}]
         ["/:id"
          {:get {:summary "ユーザー詳細を取得"
                 :parameters {:path [:map [:id :int]]}
                 :responses {200 {:body [:map
                                        [:id :int]
                                        [:name :string]
                                        [:email :string]
                                        [:age {:optional true} :int]]}
                            404 {:body [:map [:error :string]]}}
                 :handler get-user}
           :put {:summary "ユーザー情報を更新"
                 :parameters {:path [:map [:id :int]]
                             :body [:map
                                   [:name {:optional true} :string]
                                   [:email {:optional true} :string]
                                   [:age {:optional true} [:int {:min 0 :max 150}]]]}
                 :responses {200 {:body [:map
                                        [:id :int]
                                        [:name :string]
                                        [:email :string]
                                        [:age {:optional true} :int]]}
                            404 {:body [:map [:error :string]]}}
                 :handler update-user}
           :delete {:summary "ユーザーを削除"
                    :parameters {:path [:map [:id :int]]}
                    :responses {204 {}
                               404 {:body [:map [:error :string]]}}
                    :handler delete-user}}]]

        ["/products"
         {:tags ["products"]}
         [""
          {:get {:summary "商品一覧を取得"
                 :parameters {:query [:map
                                     [:category {:optional true} :string]
                                     [:min-price {:optional true} [:int {:min 0}]]
                                     [:max-price {:optional true} [:int {:min 0}]]]}
                 :responses {200 {:body [:vector [:map
                                                 [:id :int]
                                                 [:name :string]
                                                 [:price :int]
                                                 [:category :string]]]}}
                 :handler list-products}}]]]]

      {:data {:coercion reitit.coercion.malli/coercion
              :middleware [parameters/parameters-middleware
                          muuntaja/format-middleware
                          rrc/coerce-exceptions-middleware
                          rrc/coerce-request-middleware
                          rrc/coerce-response-middleware]}})))

;; http://localhost:3000/swagger-ui にアクセスして
;; 対話的なAPIドキュメントを表示
```

### スキーマへのアノテーション

スキーマに例、説明、デフォルト値を追加できます：

```clojure
;; Malli
[:map
 [:name {:title "ユーザー名"
         :description "ユーザーの表示名"
         :json-schema/default "山田太郎"}
  :string]
 [:age {:title "年齢"
        :description "ユーザーの年齢（歳）"
        :json-schema/example 25}
  [:int {:min 0 :max 150}]]]

;; Spec（spec-toolsを使用）
(require '[spec-tools.core :as st])

(s/def ::name
  (st/spec
    {:spec string?
     :description "ユーザーの表示名"
     :openapi/example "山田太郎"
     :openapi/default "山田太郎"}))

(s/def ::age
  (st/spec
    {:spec (s/and int? #(<= 0 % 150))
     :description "ユーザーの年齢（歳）"
     :openapi/example 25}))
```

### 複数の例

コンテンツタイプごとに複数の例を提供できます：

```clojure
["/pizza"
 {:get {:summary "ピザを取得 | 複数のコンテンツタイプ、複数の例"
        :responses
        {200 {:content
              {"application/json"
               {:description "JSONとしてピザを取得"
                :schema [:map [:color :keyword] [:pineapple :boolean]]
                :examples {:white {:description "パイナップル入り白ピザ"
                                  :value {:color :white :pineapple true}}
                          :red {:description "赤ピザ"
                                :value {:color :red :pineapple false}}}}
               "application/edn"
               {:description "EDNとしてピザを取得"
                :schema [:map [:color :keyword] [:pineapple :boolean]]
                :examples {:red {:description "パイナップル入り赤ピザ"
                                :value (pr-str {:color :red :pineapple true})}}}}}}}]
```

---

## 高度な機能

### ネストされたルートとルート構成

複雑なルート構造を管理するため、ネストと構成パターンを使用できます：

```clojure
(def api-routes
  ["/api" {:middleware [api-middleware]}
   ["/v1" {:middleware [v1-middleware]}
    ["/users" user-routes]
    ["/products" product-routes]]
   ["/v2" {:middleware [v2-middleware]}
    ["/users" user-v2-routes]
    ["/products" product-v2-routes]]])

(def admin-routes
  ["/admin" {:middleware [admin-middleware]
             :roles #{:admin}}
   ["/dashboard" {:get dashboard-handler}]
   ["/settings" settings-routes]
   ["/users" admin-user-routes]])

(def app
  (ring/ring-handler
    (ring/router
      [api-routes
       admin-routes
       ["/health" {:get health-handler}]])))
```

### ルート競合の検出と解決

reitit は自動的にルート競合を検出します：

```clojure
(def routes
  [["/ping"]
   ["/:user-id/orders"]
   ["/bulk/:bulk-id"]
   ["/:version/status"]])

(r/router routes)
; => CompilerException: Router contains conflicting route paths

;; 意図的に競合を許可する場合
(def routes
  [["/ping"]
   ["/:user-id/orders" {:conflicting true}]
   ["/bulk/:bulk-id" {:conflicting true}]
   ["/:version/status" {:conflicting true}]])
```

### パフォーマンス最適化

**ルーターアルゴリズムの自動選択**：

reitit は自動的に最適なルーターを選択します：

- **:lookup-router** - 静的ルート用（最速、O(1)）
- **:segment-router** - ワイルドカード付きルート用（高速、ツリーベース）
- **:mixed-router** - 混在ルート用（最適、自動選択）
- **:linear-router** - フォールバック（最も遅い）

```clojure
;; このルートツリーには自動的に :mixed-router が使用されます
(r/router
  [["/api/ping" ::ping]           ; 静的 -> :lookup-router
   ["/api/users/:id" ::user]      ; ワイルドカード -> :segment-router
   ["/api/orders/:id" ::order]])  ; ワイルドカード -> :segment-router

(r/router-name router)
; => :mixed-router
```

**ミドルウェアのコンパイル**：

```clojure
{:compile coercion/compile-request-coercers}
```

これにより次のものがコンパイルされます：
- ミドルウェアチェーン
- コアーサー実装
- パスパラメータパーサー

**コンパイルはルーター作成時に一度だけ行われ、リクエストごとには行われません。**

### 動的ルーティング

Router プロトコルを使用してルーターを構成できます：

```clojure
(defn add-routes [router routes]
  (r/router
    (into (r/routes router) routes)
    (r/options router)))

(def base-router (r/router [["/ping" ::ping]]))
(def extended-router (add-routes base-router [["/pong" ::pong]]))
```

---

## フロントエンドルーティング

### reitit-frontend モジュール

reitit は ClojureScript フロントエンドルーティング専用の **`reitit-frontend`** モジュールを提供します：

```clojure
[metosin/reitit-frontend "0.9.1"]
```

### 主要な名前空間

**reitit.frontend**：
- ClojureScript向けのコアルーティング機能
- パスと名前によるルートマッチング
- クエリパラメータの解析とコアーション

**reitit.frontend.easy**：
- ルーター状態を自動管理する簡易ラッパー
- 一度に1つのルーターのみアクティブ

**reitit.frontend.history**：
- ブラウザ履歴統合
- FragmentHistory（ハッシュベース）
- Html5History（History API）

**reitit.frontend.controllers**：
- ルート入退時のコード実行
- リソース読み込みと状態更新に便利

### Reagent統合の例

```clojure
(ns frontend.core
  (:require [reagent.core :as r]
            [reagent.dom :as rd]
            [reitit.frontend :as rf]
            [reitit.frontend.easy :as rfe]
            [reitit.coercion.spec :as rss]))

;; ルート定義
(def routes
  [["/" {:name ::home
         :view home-page}]
   ["/about" {:name ::about
              :view about-page}]
   ["/items/:id" {:name ::item
                  :view item-page
                  :parameters {:path {:id int?}
                              :query {(ds/opt :tab) keyword?}}}]])

;; 状態
(defonce match (r/atom nil))

;; ビュー
(defn home-page []
  [:div
   [:h2 "ホーム"]
   [:p "reitit フロントエンドルーティングへようこそ"]
   [:button {:on-click #(rfe/push-state ::item {:id 1})}
    "アイテム1へ"]])

(defn item-page []
  (let [id (-> @match :parameters :path :id)
        tab (-> @match :parameters :query :tab)]
    [:div
     [:h2 "アイテム " id]
     [:p "タブ: " (or tab "default")]
     [:button {:on-click #(rfe/push-state ::home)}
      "ホームに戻る"]]))

(defn about-page []
  [:div
   [:h2 "このサイトについて"]])

;; メインコンポーネント
(defn current-page []
  [:div
   [:nav
    [:ul
     [:li [:a {:href (rfe/href ::home)} "ホーム"]]
     [:li [:a {:href (rfe/href ::about)} "About"]]
     [:li [:a {:href (rfe/href ::item {:id 1})} "アイテム1"]]
     [:li [:a {:href (rfe/href ::item {:id 2} {:tab :details})} "アイテム2（詳細）"]]]]
   (when @match
     (let [view (-> @match :data :view)]
       [view]))])

;; 初期化
(defn init! []
  (rfe/start!
    (rf/router routes {:data {:coercion rss/coercion}})
    (fn [m] (reset! match m))
    {:use-fragment true})  ; ハッシュベースルーティング
  (rd/render [current-page] (.getElementById js/document "app")))
```

### Re-frame統合の例

```clojure
(ns frontend-re-frame.core
  (:require [re-frame.core :as rf]
            [reagent.dom :as rd]
            [reitit.frontend :as reitit]
            [reitit.frontend.controllers :as rfc]
            [reitit.frontend.easy :as rfe]
            [reitit.coercion.spec :as rss]))

;; エフェクト
(rf/reg-fx :push-state
  (fn [route]
    (apply rfe/push-state route)))

;; イベント
(rf/reg-event-fx ::push-state
  (fn [_ [_ & route]]
    {:push-state route}))

(rf/reg-event-db ::navigated
  (fn [db [_ new-match]]
    (let [old-match (:current-route db)
          controllers (rfc/apply-controllers
                       (:controllers old-match) new-match)]
      (assoc db :current-route (assoc new-match :controllers controllers)))))

(rf/reg-event-db ::initialize-db
  (fn [_ _]
    {:current-route nil}))

;; サブスクリプション
(rf/reg-sub ::current-route
  (fn [db _]
    (:current-route db)))

;; ルート
(def routes
  [["/"
    {:name ::home
     :view home-page
     :link-text "ホーム"
     :controllers
     [{:start (fn [_] (js/console.log "ホームに入りました"))
       :stop (fn [_] (js/console.log "ホームを出ました"))}]}]
   ["/items/:id"
    {:name ::item
     :view item-page
     :link-text "アイテム"
     :controllers
     [{:parameters {:path [:id]}
       :start (fn [params]
                (rf/dispatch [:item/load (-> params :path :id)]))
       :stop (fn [_]
               (rf/dispatch [:item/cleanup]))}]}]])

(def router
  (reitit/router routes {:data {:coercion rss/coercion}}))

;; ビュー
(defn home-page []
  [:div
   [:h2 "ホーム"]
   [:button {:on-click #(rf/dispatch [::push-state ::item {:id 1}])}
    "アイテム1へ"]])

(defn item-page []
  (let [match @(rf/subscribe [::current-route])
        id (-> match :parameters :path :id)]
    [:div
     [:h2 "アイテム: " id]]))

(defn main-panel []
  (let [current-route @(rf/subscribe [::current-route])]
    [:div
     [:nav
      [:ul
       (for [route (reitit/routes router)]
         (when-let [link-text (-> route :data :link-text)]
           [:li {:key (-> route :data :name)}
            [:a {:href (rfe/href (-> route :data :name))}
             link-text]]))]]
     (when current-route
       (let [view (-> current-route :data :view)]
         [view]))]))

;; ナビゲーションコールバック
(defn on-navigate [new-match]
  (when new-match
    (rf/dispatch [::navigated new-match])))

;; 初期化
(defn init! []
  (rf/dispatch-sync [::initialize-db])
  (rfe/start! router on-navigate {:use-fragment false})
  (rd/render [main-panel] (.getElementById js/document "app")))
```

### コントローラーパターン

コントローラーはルートの入退時にコードを実行します：

```clojure
(def routes
  [["/"
    {:controllers [{:start (fn [_] (load-global-data!))}]}]
   ["/projects/:project-id"
    {:controllers [{:parameters {:path [:project-id]}
                    :start (fn [params]
                             (let [id (-> params :path :project-id)]
                               (load-project! id)))
                    :stop (fn [_]
                            (cleanup-project!))}]}]
   ["/projects/:project-id/tasks/:task-id"
    {:controllers [{:parameters {:path [:project-id :task-id]}
                    :start (fn [params]
                             (let [project-id (-> params :path :project-id)
                                   task-id (-> params :path :task-id)]
                               (load-task! project-id task-id)))
                    :stop (fn [_]
                            (cleanup-task!))}]}]])
```

### フラグメント vs HTML5 History

**フラグメントルーティング**（`:use-fragment true`）：
- ハッシュベースのURL（`#/path`）
- サーバー設定不要
- すべてのルートで index.html を返すだけ

**HTML5 History**（`:use-fragment false`）：
- クリーンなURL（`/path`）
- すべてのルートを処理するサーバー設定が必要
- より良いSEO

---

## ベストプラクティス

### プロジェクト構成の推奨

```
src/
  myapp/
    core.clj(s)         # エントリーポイント
    routes.clj(s)       # ルート定義
    handlers/
      users.clj         # ユーザーハンドラー
      products.clj      # 商品ハンドラー
    middleware/
      auth.clj          # 認証ミドルウェア
      logging.clj       # ログミドルウェア
    db/
      core.clj          # データベースアクセス
```

### ルート構造の推奨事項

**1. 名前付きルートを常に使用**：
```clojure
;; 良い
{:name ::user-profile}  ; ::myapp.routes/user-profile に展開

;; 避ける
{:name :profile}  ; 競合の可能性
```

**2. ネストを使用して階層を表現**：
```clojure
(def routes
  ["/api" {:middleware [api-middleware]}
   ["/v1"
    ["/users" user-routes]
    ["/products" product-routes]]
   ["/v2"
    ["/users" user-v2-routes]
    ["/products" product-v2-routes]]])
```

**3. ルートデータをファーストクラスに**：
```clojure
[["/admin" {:roles #{:admin}
            :audit-log true
            :rate-limit 100}
  ["/users" {:get admin-list-users}]]]
```

### ミドルウェアの適用レベル

**汎用ミドルウェア → トップレベル**：
```clojure
(ring/ring-handler
  router
  nil
  {:middleware [wrap-cors wrap-session]})
```

**API固有ミドルウェア → ルーターレベル**：
```clojure
{:data {:middleware [api-middleware authentication-middleware]}}
```

**ルート固有ミドルウェア → ルートレベル**：
```clojure
[["/admin" {:middleware [admin-middleware]}
  ["/dangerous" {:middleware [extra-validation-middleware]}]]]
```

### コアーションの推奨事項

**1. Malliを使用**（推奨）：
```clojure
;; 最も表現力が高く、パフォーマンスが良い
{:coercion reitit.coercion.malli/coercion}
```

**2. カスタム型を定義**：
```clojure
(def UserId [:int {:min 1}])
(def Email [:re #".+@.+\..+"])
(def PhoneNumber [:re #"\d{10,11}"])

(def User
  [:map
   [:id UserId]
   [:name :string]
   [:email Email]
   [:phone {:optional true} PhoneNumber]])
```

**3. エラーメッセージをカスタマイズ**：
```clojure
(def UserId
  [:int {:min 1
         :error/message "ユーザーIDは1以上の整数である必要があります"}])
```

### 双方向ルーティングの活用

**ハードコードされたURLを避ける**：
```clojure
;; 悪い
[:a {:href "/users/123"} "ユーザー123"]

;; 良い
[:a {:href (rfe/href ::user {:id 123})} "ユーザー123"]
```

### パフォーマンスのヒント

**1. ルーター作成は高コスト、マッチングは高速**：
- アプリケーション起動時に一度だけルーターを作成
- reitit がルートツリーを解析し最適なアルゴリズムを選択
- コアーサーとミドルウェアを作成時にプリコンパイル

**2. コアーションコンパイルを有効化**：
```clojure
(rf/router routes
  {:compile coercion/compile-request-coercers
   :data {:coercion rss/coercion}})
```

**3. ルートデータサイズを最小化**：
- ルートデータを最小限に保つ
- 複雑なロジックにはミドルウェア/コントローラーを使用
- 大きなオブジェクトをルートデータに保存しない

**4. 不要な場合はルーター/マッチ注入を無効化**：
```clojure
(ring/ring-handler
  (ring/router routes)
  {:inject-match? false
   :inject-router? false})
```

### エラー処理

**1. コアーションエラー**：
```clojure
(rf/match-by-path router path
  {:on-coercion-error
   (fn [match exception]
     (js/console.warn "コアーション失敗:" exception)
     nil)})
```

**2. 404ハンドリング**：
```clojure
(ring/ring-handler
  (ring/router routes)
  (ring/create-default-handler
    {:not-found (constantly {:status 404
                            :body "ページが見つかりません"})}))
```

**3. 例外ハンドリング**：
```clojure
(require '[reitit.ring.middleware.exception :as exception])

(def exception-middleware
  (exception/create-exception-middleware
    (merge exception/default-handlers
      {::error (fn [e req]
                 {:status 400
                  :body {:error "リクエストエラー"}})
       java.sql.SQLException (fn [e req]
                              {:status 500
                               :body {:error "データベースエラー"}})})))
```

### バリデーション

**ルートデータをspecでバリデート**：
```clojure
(require '[clojure.spec.alpha :as s])
(require '[reitit.spec :as rs])

(s/def ::role #{:admin :user :manager})
(s/def ::roles (s/coll-of ::role :into #{}))

(r/router
  routes
  {:spec (s/merge (s/keys :req-un [::roles]) ::rs/default-data)
   :validate rs/validate-spec!})
```

### セキュリティ

**1. ロールベースアクセス制御**：
```clojure
(def enforce-roles-middleware
  {:name ::enforce-roles
   :spec (s/keys :req-un [::roles])
   :compile (fn [{required :roles} _]
              (when (seq required)
                (fn [handler]
                  (fn [{:keys [user] :as request}]
                    (if (set/subset? required (:roles user))
                      (handler request)
                      {:status 403
                       :body "アクセスが拒否されました"})))))})
```

**2. CSRF保護**：
```clojure
(require '[ring.middleware.anti-forgery :refer [wrap-anti-forgery]])

{:middleware [wrap-anti-forgery]}
```

**3. レート制限**：
```clojure
(def rate-limit-middleware
  {:name ::rate-limit
   :compile (fn [route-data _]
              (when-let [limit (:rate-limit route-data)]
                (fn [handler]
                  (let [limiter (create-rate-limiter limit)]
                    (fn [request]
                      (if (check-rate-limit limiter request)
                        (handler request)
                        {:status 429
                         :body "レート制限を超えました"})))))})
```

### テスト

**ルーターのテスト**：
```clojure
(require '[clojure.test :refer [deftest is testing]])

(deftest router-test
  (testing "ルートマッチング"
    (let [router (r/router routes)]
      (is (= ::home
             (-> router
                 (r/match-by-path "/")
                 :data
                 :name)))
      (is (= ::user
             (-> router
                 (r/match-by-path "/users/123")
                 :data
                 :name)))
      (is (= "123"
             (-> router
                 (r/match-by-path "/users/123")
                 :path-params
                 :id)))))

  (testing "逆方向ルーティング"
    (let [router (r/router routes)]
      (is (= "/users/123"
             (-> router
                 (r/match-by-name ::user {:id 123})
                 r/match->path))))))
```

### デバッグ

**リクエスト差分の表示**：
```clojure
(require '[reitit.ring.middleware.dev :as dev])

(ring/router
  routes
  {:reitit.middleware/transform dev/print-request-diffs})
```

---

## まとめと次のステップ

### 学んだ主要概念

1. **データ駆動型アーキテクチャ** - ルートは純粋なデータ構造
2. **双方向ルーティング** - パスからルート、ルートからパス
3. **ファーストクラスのルートデータ** - 任意のデータをルートに添付
4. **プラガブルなコアーション** - Malli、Spec、Schemaのサポート
5. **コンパイル時最適化** - ミドルウェアとコアーサーのプリコンパイル
6. **自動API文書化** - OpenAPI 3.1.0サポート
7. **フロントエンド統合** - ClojureScript SPAサポート
8. **パフォーマンス** - 複数のルーターアルゴリズムと自動選択

### さらに学ぶために

**公式リソース**：
- GitHub: https://github.com/metosin/reitit
- ドキュメント: https://cljdoc.org/d/metosin/reitit
- Slack: Clojurians Slack の #reitit チャンネル

**実践プロジェクト**：
- GitHubリポジトリのサンプルプロジェクト
- Luminusフレームワーク（reitit を使用）
- Kitフレームワーク（reitit を使用）

### 実践のための提案

1. **小さく始める** - シンプルなCRUD APIから開始
2. **段階的に追加** - コアーション、バリデーション、ドキュメント化を追加
3. **パフォーマンスを測定** - ベンチマークを取り、最適化
4. **コミュニティに参加** - Slackで質問し、経験を共有

reitit は、Clojure/ClojureScriptエコシステムにおける最先端のルーティングソリューションであり、データ駆動型アプローチ、優れたパフォーマンス、強力な機能により、小規模なプロトタイプから大規模な本番アプリケーションまで、あらゆるプロジェクトに適しています。
