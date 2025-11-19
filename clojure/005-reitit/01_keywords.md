# reititキーワード集

この利用事例を読んで、理解して、馴染ませておく(何があるかを把握しておく)

https://github.com/metosin/reitit/tree/master/examples


## reititファミリ

- reitit: 全バンドル
- reitit-core: ルーティングのコア
- reitit-ring: Ring形式のルーター
- reitit-middleware: よく使われるミドルウェア
  - wrap-params: クエリ、フォームパラメータの解析・利用
  - exception handling: 例外処理
  - content negotiation: muuntajaなどと連携してデータ変換
  - multipart-middleware: ファイルアップロードなど
  - print-request-diffs: ミドルウェアの変換前後の差分を調査する
- reitit-spec: clojure.specによるcoercion
- reitit-malli: malliによるcoercion ← おすすめ
- reitit-schema: Scehemaによるcoercion
- fi.metosin/reitit-openapi: OpenAPIドキュメント生成 ← 最新
- reitit-swagger: Swagger2ドキュメント生成
- reitit-swagger-ui: Swagger UIの提供
- reitit-frontend: フロントエンドルーター
- reitit-http: 共通前・後処理、メソッド分岐、インターセプター
- reitit-interceptors: インターセプター集
  - parameters-interceptor: パラメータについての処理
  - exception-interceptor: 例外処理
  - format-interceptor
  - format-negotiate-interceptor
  - format-request-interceptor
  - format-response-interceptor
  - multipart-interceptor
- reitit-sieppari: Sieppariインターセプターライブラリとの連携
- reitit-dev
  - ルーター定義回りのデバッグで見やすくする
