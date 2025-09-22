以下は、Babashka中級者向けの実用チートシート（Web掲載向けMarkdown）です。
対象範囲：ファイル操作 / テキスト処理 / コマンドライン引数 / 外部コマンド / JSON・EDN・CSV・SQL。
サンプルはやさしい→むずかしいの順に配置し、コメントは日本語で記述しています。

⸻

Babashka スクリプト チートシート（中級・実用）

使い方（例）:
	1.	ファイル先頭に shebang を書く

#!/usr/bin/env bb
(println "hello")

	2.	実行権限を付与: chmod +x script.clj
	3.	実行: ./script.clj または bb script.clj

⸻

0. 基本小技（標準入出力・環境変数・エラーハンドリング）

;; 標準入力を丸ごと読む（フィルタコマンド的ユース）
(println (slurp *in*))

;; 環境変数の取得（なければデフォルト）
(def home (or (System/getenv "HOME") "."))
(println "HOME=" home)

;; 例外を捕捉してエラーメッセージ＋終了コード
(try
  (throw (ex-info "問題が発生しました" {:code 123}))
  (catch Exception e
    (binding [*out* *err*]
      (println "エラー:" (.getMessage e)))
    (System/exit 1)))


⸻

1. ファイル操作（やさしい → 普通 → むずかしい）

1-1. もっとも基本：読み・書き・追記

;; 全文読み込み
(def txt (slurp "input.txt"))
;; 上書き書き込み（存在しない場合は作成）
(spit "output.txt" txt)
;; 追記（append）
(spit "log.txt" "1行追加\n" :append true)

1-2. babashka.fs でよくある処理

(require '[babashka.fs :as fs])

;; 存在チェック & 種別チェック
(println (fs/exists? "output.txt"))  ; true/false
(println (fs/directory? "."))        ; ディレクトリか？
(println (fs/regular-file? "output.txt"))

;; つくる・消す・移動・コピー
(fs/create-dirs "tmp/nested/dir")               ; ディレクトリ作成（親も含めて）
(spit "tmp/nested/dir/file.txt" "hello")        ; 中にファイル作成
(fs/copy "tmp/nested/dir/file.txt" "tmp/file2.txt" {:replace-existing true})
(fs/move "tmp/file2.txt" "tmp/file3.txt" {:replace-existing true})
(fs/delete "tmp/file3.txt")                     ; ファイル削除
(fs/delete-tree "tmp")                          ; ツリーごと削除（慎重に！）

;; 列挙
(doseq [p (fs/list-dir ".")]                     ; 直下のみ
  (println "ENTRY:" (str p)))

;; 再帰列挙（walk-file-tree）
(doseq [p (fs/walk "." {:max-depth 3})]
  (when (fs/regular-file? p)
    (println "FILE:" (str p))))

1-3. グロブ・一括リネーム・原子的書き込み

(require '[clojure.string :as str])

;; グロブで *.log を探して .old に拡張子変更
(doseq [p (fs/glob "." "**/*.log")]
  (let [new (str (fs/strip-ext p) ".old")]
    (fs/move p new {:replace-existing true})
    (println "RENAMED:" (str p) "->" new)))

;; 一時ファイルを使って「原子的に」置換結果を書き戻す
(defn atomic-rewrite! [path f]
  (let [tmp (fs/create-temp-file)]
    (spit tmp (f (slurp path)))
    (fs/move tmp path {:replace-existing true})))

(atomic-rewrite! "config.ini"
  (fn [s]
    (-> s
        (str/replace #"^debug\s*=\s*true" "debug=false"))))


⸻

2. テキスト処理（やさしい → 普通 → むずかしい）

2-1. 置換・抽出・分割

(require '[clojure.string :as str])

;; 単純置換（正規表現／グローバル）
(println (str/replace "User123" #"\d" "X"))  ; => "UserXXX"

;; グループ抽出
(let [m (re-matches #"(\w+)-(\d+)" "item-42")]
  (when m
    (println "name=" (nth m 1) ", id=" (nth m 2))))

;; 行単位処理（ファイル -> フィルタ -> 新ファイル）
(with-open [r (clojure.java.io/reader "access.log")]
  (let [lines (line-seq r)
        errs  (filter #(re-find #"\bERROR\b" %) lines)]
    (spit "error.log" (str/join "\n" errs))))

2-2. 置換コマンド（インプレースっぽい動作）

;; すべての *.md で、"TODO" を "DONE" に置換（原子的に書き戻す）
(doseq [p (fs/glob "." "**/*.md")]
  (atomic-rewrite! p #(str/replace % #"TODO" "DONE")))

2-3. トランスデューサ & パイプライン（効率よいストリーム処理）

(require '[clojure.java.io :as io])

;; 大きなファイルをフィルタ＆変換しながら出力
(with-open [r (io/reader "big.tsv")
            w (io/writer "filtered.tsv")]
  (let [xf (comp
            (map #(str/split % #"\t"))
            (filter (fn [[user score]] (<= 80 (Long/parseLong score))))
            (map #(str/join "\t" %)))]
    (doseq [line (sequence xf (line-seq r))]
      (.write w line) (.write w "\n"))))


⸻

3. コマンドライン引数（やさしい → 普通）

3-1. 生引数を使う（軽量）

;; ./script.clj foo bar baz
(println "ARGS:" *command-line-args*)  ; => ("foo" "bar" "baz")

3-2. babashka.cliでオプション解析（推奨）

(require '[babashka.cli :as cli])

;; -i/--in と -o/--out を受け取り、-v/--verbose はフラグ
(let [{:keys [in out verbose]} 
      (cli/parse-opts *command-line-args*
                      {:alias {:i :in :o :out :v :verbose}
                       :coerce {:in :string :out :string}})]
  (when (or (nil? in) (nil? out))
    (binding [*out* *err*]
      (println "Usage: script -i <in-file> -o <out-file> [-v]"))
    (System/exit 2))
  (when verbose (println "入力:" in "出力:" out))
  (spit out (str/upper-case (slurp in))))


⸻

4. 外部コマンド実行（やさしい → 普通 → むずかしい）

4-1. babashka.process/shell で実行＆標準出力を文字列取得

(require '[babashka.process :refer [shell]])

(let [{:keys [out err exit]} (shell {:out :string :err :string} "echo" "hello")]
  (println "OUT:" out)   ; "hello\n"
  (println "ERR:" err)   ; ""
  (println "EXIT:" exit)) ; 0

4-2. ストリーミング読取（長いログ対応）

(require '[clojure.java.io :as io])

(let [proc (shell {:out :stream} "ls" "-1")]
  (with-open [r (io/reader (:out proc))]
    (doseq [line (line-seq r)]
      (println "FILE:" line))))

4-3. パイプライン相当（入力をコマンドに渡す）

;; echo "abc" | tr a-z A-Z を Babashka で
(let [proc (shell {:in "abc\n" :out :string} "tr" "a-z" "A-Z")]
  (println (:out proc))) ; => "ABC\n"


⸻

5. JSON / EDN / CSV / SQL（やさしい → むずかしい）

5-1. JSON（Cheshire）

(require '[cheshire.core :as json])

(def data {:user "alice" :age 30 :roles ["admin" "user"]})

;; 文字列へ
(def s (json/generate-string data))    ; => "{\"user\":\"alice\",\"age\":30,...}"

;; 文字列から（キーワード化）
(def parsed (json/parse-string s true)) ; => {:user "alice", :age 30, ...}

;; ファイル入出力
(spit "data.json" (json/generate-string data))
(def parsed2 (json/parse-string (slurp "data.json") true))

5-2. EDN（標準）

(require '[clojure.edn :as edn])

(def cfg {:endpoint "https://api.example.com" :retries 3})
(spit "config.edn" (pr-str cfg))

(def cfg2 (edn/read-string (slurp "config.edn")))
(println (:endpoint cfg2)) ; => "https://api.example.com"

5-3. CSV（clojure.data.csv）

(require '[clojure.data.csv :as csv]
         '[clojure.java.io :as io]
         '[clojure.string :as str])

;; 読み込み（ヘッダ行付き→マップにする実用パターン）
(defn read-csv-as-maps [path]
  (with-open [r (io/reader path)]
    (let [[header & rows] (csv/read-csv r)
          ks (map keyword header)]
      (map (fn [row] (zipmap ks row)) rows))))

;; 書き込み
(with-open [w (io/writer "out.csv")]
  (csv/write-csv w
                 [["name" "age"]
                  ["alice" "30"]
                  ["bob"   "40"]]))

;; 例: age>=35 の行だけ抽出して別ファイルに
(let [rows (read-csv-as-maps "out.csv")
      sel  (filter #(<= 35 (Long/parseLong (% :age))) rows)]
  (with-open [w (io/writer "older.csv")]
    (csv/write-csv w
      (cons ["name" "age"]
            (map (fn [{:keys [name age]}] [name age]) sel)))))

5-4. SQL（SQLiteに対する軽量クエリ：Babashka Pod 利用）

ポイント: JDBCドライバをクラスパスに載せずに使いたい場合、Babashka Pod（例: SQLite）を使うのが手軽です。
ここでは org.babashka/sqlite3 Pod の想定例を示します（プロジェクトに合わせて実在のPod名・関数名は調整してください）。

(require '[babashka.pods :as pods])

;; SQLite3 Pod をロード（初回はダウンロードされることがあります）
(pods/load-pod 'org.babashka/sqlite3 "0.1.0") ; バージョンは適宜
(require '[pod.babashka.sqlite3 :as sqlite])

;; メモリDBを開く（:db "file:test.db?mode=memory&cache=shared" なども可）
(def conn (sqlite/open {:db ":memory:"}))

;; テーブル作成＋データ挿入
(sqlite/execute! conn ["CREATE TABLE users(id INTEGER, name TEXT)"])
(sqlite/execute! conn ["INSERT INTO users VALUES(?,?)" 1 "alice"])
(sqlite/execute! conn ["INSERT INTO users VALUES(?,?)" 2 "bob"])

;; 検索（ベクタのマップとして返る想定）
(def rows (sqlite/query conn ["SELECT * FROM users WHERE id >= ?" 1]))
(println rows) ; => [{:id 1, :name "alice"} {:id 2, :name "bob"}]

(sqlite/close conn)

もし JDBC でやりたい場合は、next.jdbc とドライバ（例: SQLite/H2/PostgreSQL）の依存を bb.edn の :deps に追加し、通常の Clojure と同様に next.jdbc を使えます（Babashka対応範囲内）。
例（超概略）：

(require '[next.jdbc :as jdbc])
(def ds (jdbc/get-datasource {:dbtype "h2:mem" :dbname "test"}))
(jdbc/execute! ds ["CREATE TABLE t(id INT, v VARCHAR(20))"])
(jdbc/execute! ds ["INSERT INTO t VALUES(?,?)" 1 "x"])
(jdbc/execute! ds ["SELECT * FROM t"])



5-5. 「SQLファイルをパース」する簡易テク（複文をセミコロン区切りで流す）

;; ※ 正確なSQL構文パースではなく、単純な「;」分割の例（コメントや文字列中の ; などは未対応）
(defn naive-sql-statements [s]
  (->> (clojure.string/split s #";")
       (map clojure.string/trim)
       (remove clojure.string/blank?)))

(let [sql (slurp "schema.sql")
      stmts (naive-sql-statements sql)]
  (doseq [st stmts]
    (println "EXEC:" st)
    ;; (sqlite/execute! conn [st]) ; 実際に投げるとき
    ))


⸻

6. 便利ユーティリティ（実用断片集）

6-1. 置換スクリプト（拡張子指定・バックアップ作成）

;; 指定拡張子の全ファイルを対象に正規表現置換し、.bak を残す
(defn replace-in-file! [p re replacement]
  (let [orig (slurp p)
        next (clojure.string/replace orig re replacement)]
    (when (not= orig next)
      (fs/copy p (str p ".bak") {:replace-existing true})
      (spit p next)
      (println "Rewrote:" (str p)))))

(doseq [p (fs/glob "." "**/*.conf")]
  (replace-in-file! p #"^debug\s*=\s*true" "debug=false"))

6-2. 行ダンプ（行番号付き）

(require '[clojure.java.io :as io])

(with-open [r (io/reader "README.md")]
  (doseq [[idx line] (map-indexed vector (line-seq r))]
    (println (format "%4d: %s" (inc idx) line))))

6-3. CSV ↔ JSON 変換（ヘッダ→マップ）

(require '[clojure.data.csv :as csv]
         '[clojure.java.io :as io]
         '[cheshire.core :as json])

(defn csv->maps [path]
  (with-open [r (io/reader path)]
    (let [[h & rows] (csv/read-csv r)
          ks (map keyword h)]
      (map #(zipmap ks %) rows))))

(spit "data.json" (json/generate-string (csv->maps "in.csv")))

6-4. フィルタコマンド化（標準入力→標準出力）

;; 使用: cat input.txt | ./filter.clj "ERROR"
(require '[clojure.string :as str])

(let [pattern (re-pattern (first *command-line-args*))]
  (doseq [line (line-seq (clojure.java.io/reader *in*))]
    (when (re-find pattern line)
      (println line))))


⸻

7. bb.edn を使ったタスク化（プロジェクト内での反復作業に）

;; bb.edn
{:tasks
 {:hello (println "Hello Task")
  :build {:doc "静的サイト生成"
          :task (shell "npm" "run" "build")}
  :fmt   {:doc "cljfmt で整形"
          :requires ([babashka.process :refer [shell]])
          :task (shell "cljfmt" "fix")}}}

# 実行例
bb hello
bb build
bb fmt


⸻

8. 参考スタイル（スクリプト構造の雛形）

#!/usr/bin/env bb
(ns mytool.core
  (:require
    [babashka.cli :as cli]
    [babashka.fs  :as fs]
    [clojure.string :as str]
    [babashka.process :refer [shell]]))

(def spec
  {:cmds {:upper {:doc "ファイルを大文字化して出力"
                  :args [:in]
                  :fn   (fn [{:keys [in]}]
                          (println (str/upper-case (slurp in))))}
          :grep  {:doc "正規表現で行フィルタ（stdin→stdout）"
                  :args [:pattern]
                  :fn   (fn [{:keys [pattern]}]
                          (let [re (re-pattern pattern)]
                            (doseq [line (line-seq (clojure.java.io/reader *in*))]
                              (when (re-find re line)
                                (println line)))))}
          :ls    {:doc "カレント配下のファイル一覧"
                  :fn  (fn [_]
                         (doseq [p (fs/walk ".")]
                           (when (fs/regular-file? p)
                             (println (str p)))))}}})

(defn -main [& argv]
  (cli/dispatch spec argv))

(apply -main *command-line-args*)


⸻

付録：よくある落とし穴・コツ
	•	巨大ファイルを slurp で丸読みするとメモリ圧迫。行ストリーム（line-seq）＋トランスデューサで処理すると安全。
	•	原子的書き換えは「一時ファイルに出力 → fs/move で置き換え」パターンが安心。
	•	外部コマンドの標準入出力は :in, :out を :string または :stream で扱い分ける。長時間・大量出力なら :stream。
	•	CSV の数値は文字列で来るので Long/parseLong / parse-double などで明示変換。
	•	SQLは「Podで軽量に」か「JDBC依存を bb.edn に足す」かを使い分ける。Podはドライバ不要で導入が楽。

⸻

必要であれば、このチートシートを**カテゴリ追加（例：YAML/TOML、HTTPリクエスト、簡易テンプレート生成など）**にも拡張できます。
「この用途のスニペットが欲しい」など、具体的な要望があれば言ってください。