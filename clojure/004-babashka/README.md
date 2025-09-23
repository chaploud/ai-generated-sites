# Babashka スクリプト チートシート（中級・実用）

> 使い方（例）:
> 1) ファイル先頭に shebang を書く
> ```clojure
> #!/usr/bin/env bb
> (println "hello")
> ```
> 2) 実行権限を付与: `chmod +x script.clj`
> 3) 実行: `./script.clj` または `bb script.clj`

---

## 0. 基本小技（標準入出力・環境変数・エラーハンドリング）

```clojure
(println (slurp *in*))

(def home (or (System/getenv "HOME") "."))
(println "HOME=" home)

(try
  (throw (ex-info "問題が発生しました" {:code 123}))
  (catch Exception e
    (binding [*out* *err*]
      (println "エラー:" (.getMessage e)))
    (System/exit 1)))
```

---

## 1. ファイル操作

### 1-1. 読み・書き・追記
```clojure
(def txt (slurp "input.txt"))
(spit "output.txt" txt)
(spit "log.txt" "1行追加\n" :append true)
```

### 1-2. `babashka.fs`
```clojure
(require '[babashka.fs :as fs])

(fs/create-dirs "tmp/nested/dir")
(spit "tmp/nested/dir/file.txt" "hello")
(fs/copy "tmp/nested/dir/file.txt" "tmp/file2.txt" {:replace-existing true})
(fs/move "tmp/file2.txt" "tmp/file3.txt" {:replace-existing true})
(fs/delete "tmp/file3.txt")
(fs/delete-tree "tmp")

(doseq [p (fs/list-dir ".")]
  (println "ENTRY:" (str p)))
```

### 1-3. グロブ・原子的書換え
```clojure
(doseq [p (fs/glob "." "**/*.log")]
  (let [new (str (fs/strip-ext p) ".old")]
    (fs/move p new {:replace-existing true})))

(defn atomic-rewrite! [path f]
  (let [tmp (fs/create-temp-file)]
    (spit tmp (f (slurp path)))
    (fs/move tmp path {:replace-existing true})))
```

---

## 2. テキスト処理

### 2-1. 置換・抽出
```clojure
(require '[clojure.string :as str])
(str/replace "User123" #"\d" "X")

(let [m (re-matches #"(\w+)-(\d+)" "item-42")]
  (println "name=" (nth m 1) ", id=" (nth m 2)))
```

### 2-2. 行フィルタ
```clojure
(with-open [r (clojure.java.io/reader "access.log")]
  (let [lines (line-seq r)
        errs  (filter #(re-find #"ERROR" %) lines)]
    (spit "error.log" (str/join "\n" errs))))
```

### 2-3. トランスデューサ
```clojure
(with-open [r (clojure.java.io/reader "big.tsv")
            w (clojure.java.io/writer "filtered.tsv")]
  (let [xf (comp
            (map #(str/split % #"	"))
            (filter (fn [[_ score]] (<= 80 (Long/parseLong score))))
            (map #(str/join "	" %)))]
    (doseq [line (sequence xf (line-seq r))]
      (.write w line) (.write w "\n"))))
```

---

## 3. コマンドライン引数

### 3-1. 生引数
```clojure
(println *command-line-args*)
```

### 3-2. babashka.cli
```clojure
(require '[babashka.cli :as cli])
(let [{:keys [in out verbose]}
      (cli/parse-opts *command-line-args*
                      {:alias {:i :in :o :out :v :verbose}})]
  (when verbose (println "入力:" in "出力:" out))
  (spit out (clojure.string/upper-case (slurp in))))
```

---

## 4. 外部コマンド

### 4-1. 実行と結果取得
```clojure
(require '[babashka.process :refer [shell]])
(shell "echo" "Hello Babashka")

(let [{:keys [out]} (shell {:out :string} "ls" "-1")]
  (println out))
```

### 4-2. ストリーム読取
```clojure
(let [proc (shell {:out :stream} "ls")]
  (with-open [r (clojure.java.io/reader (:out proc))]
    (doseq [line (line-seq r)]
      (println "FILE:" line))))
```

### 4-3. パイプライン相当
```clojure
(let [proc (shell {:in "abc\n" :out :string} "tr" "a-z" "A-Z")]
  (println (:out proc)))
```

---

## 5. JSON / EDN / CSV / SQL

### 5-1. JSON
```clojure
(require '[cheshire.core :as json])
(def data {:user "alice" :age 30})
(spit "data.json" (json/generate-string data))
(json/parse-string (slurp "data.json") true)
```

### 5-2. EDN
```clojure
(require '[clojure.edn :as edn])
(spit "config.edn" (pr-str {:a 1}))
(edn/read-string (slurp "config.edn"))
```

### 5-3. CSV
```clojure
(require '[clojure.data.csv :as csv]
         '[clojure.java.io :as io])

(with-open [r (io/reader "data.csv")]
  (doseq [row (csv/read-csv r)]
    (println row)))
```

### 5-4. SQL (SQLite Pod)
```clojure
(require '[babashka.pods :as pods])
(pods/load-pod 'org.babashka/sqlite3 "0.1.0")
(require '[pod.babashka.sqlite3 :as sqlite])

(def conn (sqlite/open {:db ":memory:"}))
(sqlite/execute! conn ["CREATE TABLE users(id INT, name TEXT)"])
(sqlite/execute! conn ["INSERT INTO users VALUES(?,?)" 1 "alice"])
(sqlite/query conn ["SELECT * FROM users"])
```

---

## 6. 小技サンプル

### 6-1. 行番号付き出力
```clojure
(with-open [r (clojure.java.io/reader "README.md")]
  (doseq [[i line] (map-indexed vector (line-seq r))]
    (println (format "%3d: %s" (inc i) line))))
```

### 6-2. フィルタコマンド
```clojure
(let [pattern (re-pattern (first *command-line-args*))]
  (doseq [line (line-seq (clojure.java.io/reader *in*))]
    (when (re-find pattern line)
      (println line))))
```

---

## 7. bb.edn タスク化

```clojure
{:tasks
 {:hello (println "Hello Task")
  :build {:doc "サイト生成" :task (shell "npm run build")}
  :fmt   {:doc "整形" :task (shell "cljfmt fix")}}}
```

---

## 8. HTTP リクエスト（REST クライアント的に）

### 8-1. GET リクエスト
```clojure
(require '[babashka.curl :as curl])

(def resp (curl/get "https://jsonplaceholder.typicode.com/todos/1"
                    {:as :json}))
(println (:status resp))
(println (:body resp))
```

### 8-2. POST リクエスト（JSON ボディ）
```clojure
(require '[cheshire.core :as json])

(def new-todo {:userId 1 :title "新しいタスク" :completed false})

(def resp (curl/post "https://jsonplaceholder.typicode.com/todos"
                     {:headers {"Content-Type" "application/json"}
                      :body (json/generate-string new-todo)
                      :as :json}))
(println (:status resp))
(println (:body resp))
```

### 8-3. 認証ヘッダ付きリクエスト
```clojure
(def token (System/getenv "API_TOKEN"))

(def resp (curl/get "https://api.example.com/private"
                    {:headers {"Authorization" (str "Bearer " token)}
                     :as :json}))
(println (:body resp))
```

### 8-4. エラーハンドリング
```clojure
(let [resp (curl/get "https://api.example.com/notfound" {:throw false})]
  (if (= 200 (:status resp))
    (println "OK:" (:body resp))
    (binding [*out* *err*]
      (println "HTTP Error:" (:status resp)))))
```
