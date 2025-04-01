---
title: "実行計画を元にした Spanner クエリ最適化の実践"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [spanner]
published: false
---

私は「Spanner でも他の RDBMS で当たり前に行われているのと同程度には実行計画を理解する」ことをテーマに情報の公開や社内での啓蒙・トラブルシューティングなどの活動を行ってきました。

- [Cloud Spanner as a SQL System](https://docs.google.com/presentation/d/17RDfrKLpeZrVKuuskeO_E9x0wLeBeTNzPjqrdk97MFs/edit?usp=sharing)
- [Cloud Spanner の実行計画の活用に関する取り組み](https://engineering.mercari.com/blog/entry/20201210-cloud-spanner-query-plan/)
- [Cloud Spanner Unofficial Hacks](https://spanner-hacks.apstn.dev/) 

私見として、 2017年の Spanner のリリースから8年経った今も実行計画の最適化についての世の中の情報は多くありません。
実行方法が自明ではない SQL をどう現実的に実行可能な程度に最適化するかというような話はまだあまり増えていないと感じています。

この文書は私が株式会社メルペイを退職する前から社外に公開するという約束で書いていたものですが、一年半以上放置してこのまま墓場まで持っていくことになる可能性が出てきました。
時代遅れになっている記述や洗練されていないままの部分も残っていますが公開しておきます。もしもまた Spanner の仕事をすることがあったら更新するかもしれません。

:::message
この文書は正解のない Read と Write のトレードオフが存在するのは当たり前のこととして、世間では十分に説明されていないクエリの最適化の方法論の一部について説明しています。
実際にどの選択肢を選ぶかについてはトレードオフを意識し、負荷試験なども通して個別に決定しましょう。

なお、 2023年頃の記述からほぼ更新していないため、実行計画等現在のものとは異なる場合があります。
例えば [`scan_method`, `execution_method`](https://cloud.google.com/spanner/docs/sql-best-practices#execution-method-vs-scan-method) は当時ありませんでした。基本は変わっていないので適当によろしくお願いします。
:::

## この記事の意図ではないこと

- Spanner ユーザでない人に何らかの先入観を与えることは意図していませんので、 Spanner ユーザ以外は読む必要はありません。
- Spanner で複雑な分析クエリを実行することはもちろん推奨しません。何らかの分散処理基盤にオフロードすることを推奨します。
  - 例えば必要に応じて [Data Boost](https://cloud.google.com/spanner/docs/databoost/databoost-overview) を有効にし、 BigQuery([`EXTERNAL_QUERY` もしくは外部データセット](https://cloud.google.com/bigquery/docs/spanner-federated-queries?hl=en)), [Cloud Dataflow](https://cloud.google.com/spanner/docs/dataflow-connector?hl=en) などを使う選択肢があります。
- この記事に書かれたプラクティスが一般的に適用可能で常に問題が解決できるというような主張はしません。個別事象で問題を解決するために検討できる選択肢を増やすのが目的です。
- トレードオフが存在することを否定する意図はありません。

## 指針

[SQLパフォーマンス詳解](https://sql-performance-explained.jp/)、もしくはその Web 版の [Use The Index, Luke](https://use-the-index-luke.com/ja/sql/performance-scalability/system-load) より

> 注意深い実行計画の調査結果は、 うわべだけのベンチマークよりも信用のおけるものです。
> 完全な負荷テストは意味のあることですが、そのための手間はかかります。

* 本質的に非効率な処理を含むクエリの多くは、注意深く実行計画を読むことで負荷試験よりも早いフェーズで発見可能です。
  * これは本番相当のデータがない開発用データベースしかない状態であっても例外ではありません。
* クエリの実行計画をできるだけ開発中の早期にレビュー可能な状態で共有しましょう。
  * 例えば、 GitHub に添付可能なテキスト形式で得られる `spanner-cli` 等の `EXPLAIN` の結果を GitHub の Pull Request に貼るだけで良いでしょう。
    * 時期尚早な最適化をする必要はありませんが、クエリが期待と大きく違わない形で処理可能なことを他の人もレビューできる状態にすることが重要です。
        * 最初は読めなくても、事後に振り返りをしているうちに読めるようになるでしょう。
* データの有無によってコストベース最適化によって異なる実行計画が選ばれる場合はありますが、運良く良くなることに期待するのではなく、運悪く悪くなる可能性を想定しましょう。
  * オプティマイザバージョンを下げた方が予測可能性が高いケースもあります。具体的には[バージョン5](https://cloud.google.com/spanner/docs/query-optimizer/versions#version-5)からコストベースで最適化される項目が増えたので[バージョン4](https://cloud.google.com/spanner/docs/query-optimizer/versions#version-4)に固定することも検討の価値があります。
* 定期的に非効率なクエリの存在をレビューしましょう。
    * Query Stats https://cloud.google.com/spanner/docs/introspection/query-statistics
    * Query Insights https://cloud.google.com/spanner/docs/using-query-insights
* 発行される可能性があるクエリが変わった結果、無駄に Write コストだけ掛かっているインデックスがないかどうかは定期的にチェックしましょう。

### トレードオフについて

多くの場合 Read(Query) と Write にはトレードオフが存在します。

クエリに必要なセカンダリインデックスは Mutation を使った Write には必要ないため、 Write に最も最適化した選択肢ではセカンダリインデックスは存在しないでしょう。
クエリを最適化するためにセカンダリインデックスを作ったり、そのセカンダリインデックスに更に `STORING` 列を追加することは Write のコストを上げることになります。
Read のパフォーマンスを高めるには Write を犠牲にすることを単純化すると下のグラフのようになります。

![spanner-read-write-tradeoff.png](/images/spanner-read-write-tradeoff.png)

:::message
分散コミットに参加する participants やリモートコールの有無によって非連続的にコストは変化するため、実際にはこのようになだらかな線にはならないでしょう。
:::

しかし、トレードオフ以前の問題になっている場面はよくあります。先ほどのグラフで言えば、線上の点がトレードオフした上でのそれぞれの最適解だとすれば、それよりも非効率的な選択肢は無数に存在します。

![spanner-read-write-tradeoff-unoptimized.png](/images/spanner-read-write-tradeoff-unoptimized.png)

このような非効率な状態となる理由の一部として、クエリで十分に役立たないセカンダリインデックスに Write のコストを掛けているというものがあります。
この記事では、どのようにクエリに最適化することで Write のコストを無駄にせずに、 Read のパフォーマンスを上げられるかについてフォーカスします。

:::message
この記事では触れませんが、クエリ結果を使って更新を行う DML は純粋な更新ではなくクエリの側面を持っているので、セカンダリインデックスによる最適化が必要な場合があります。
また、トレードオフが発生しない場合もあります。例えばプライマリキーよりもそのクエリに適したセカンダリインデックスが存在しないクエリにはセカンダリインデックスは不要なため、 Read と Write の最適は一致するためトレードオフは発生しません。これは Spanner のスキーマ設計の一つの理想だと言えるでしょう。
:::

## Introspection

### How to get query plans and query profiles using [`spanner-cli`](https://github.com/cloudspannerecosystem/spanner-cli)

Google 公式の Spanner Studio の実行計画表示はあまり一覧性が高くなくフィルタ述語等のレビューが困難なため、実行計画全体をテキストで共有することを推奨します。
この用途に使うことができるツールとしては下記のようなものがあります。

- [cloudspannerecosystem/spanner-cli](https://github.com/cloudspannerecosystem/spanner-cli)
- [apstndb/spanner-mycli](https://github.com/apstndb/spanner-mycli)
- [rendertree in apstndb/spannerplanviz](https://github.com/apstndb/spannerplanviz/tree/main/cmd/rendertree)

:::message
この記事の例では spanner-cli を使って取得しています。
:::

#### `EXPLAIN` (PLAN)

`EXPLAIN` で実行計画を見ることができます。

```
spanner> EXPLAIN
-> SELECT LastName
-> FROM Singers
-> WHERE SingerId = 1;
+----+--------------------------------------------------------------+
| ID | Query_Execution_Plan                                         |
+----+--------------------------------------------------------------+
| *0 | Distributed Union                                            |
|  1 | +- Local Distributed Union                                   |
|  2 |    +- Serialize Result                                       |
| *3 |       +- Filter Scan (seekable_key_size: 1)                  |
|  4 |          +- Table Scan (Table: Singers, scan_method: Scalar) |
+----+--------------------------------------------------------------+
Predicates(identified by ID):
0: Split Range: ($SingerId = 1)
3: Seek Condition: ($SingerId = 1)
```

クエリの実行やデータアクセスを伴わないため、実行計画の取得は本番環境で行っても安全です。

#### `EXPLAIN ANALYZE` (PROFILE)

`EXPLAIN ANALYZE` を使うことで実行プロファイルを取得できます。テストデータもしくは本番データを使って量的な性能特性を示したい場合はこちらを使いましょう。

```
spanner> EXPLAIN ANALYZE
-> SELECT LastName
-> FROM Singers
-> WHERE SingerId = 1;
+----+--------------------------------------------------------------+---------------+------------+---------------+
| ID | Query_Execution_Plan                                         | Rows_Returned | Executions | Total_Latency |
+----+--------------------------------------------------------------+---------------+------------+---------------+
| *0 | Distributed Union                                            | 1             | 1          | 2.24 msecs    |
|  1 | +- Local Distributed Union                                   | 1             | 1          | 2.23 msecs    |
|  2 |    +- Serialize Result                                       | 1             | 1          | 2.23 msecs    |
| *3 |       +- Filter Scan (seekable_key_size: 1)                  | 1             | 1          | 2.22 msecs    |
|  4 |          +- Table Scan (Table: Singers, scan_method: Scalar) | 1             | 1          | 2.22 msecs    |
+----+--------------------------------------------------------------+---------------+------------+---------------+
Predicates(identified by ID):
0: Split Range: ($SingerId = 1)
3: Seek Condition: ($SingerId = 1)

1 rows in set (4.26 msecs)
timestamp:            2023-06-19T13:04:42.434401+09:00
cpu time:             2.06 msecs
rows scanned:         1 rows
deleted rows scanned: 0 rows
optimizer version:    5
optimizer statistics: auto_20230616_13_28_03UTC
```

`EXPLAIN` とは異なりこの機能は実際にクエリを実行します。意図せずに重いクエリを本番環境で実行することがないように気をつけましょう。

### Query statistics

Query Statistics を使うことで本番環境で負荷が多いクエリを見つけることができます。

<!--
Cautionary Note

* Workloads that use Partition Query, such as export jobs for backup, are not included in Query Statistics.
* RPCs that are reads, not SQL queries, are only included in Read Statistics, not Query Statistics.
-->

<!--
Example: query to get the load per query from the last 7 days in order of total CPU load
-->

例: 過去7日間からクエリごとの負荷を合計の CPU 負荷順で取得するクエリ


```
SELECT TEXT_FINGERPRINT,
ANY_VALUE(text) AS text,
ANY_VALUE(request_tag) AS request_tag,
SUM(execution_count) AS execution_count,
SUM(execution_count * avg_cpu_seconds) AS total_cpu_seconds,
SUM(execution_count * avg_bytes) AS total_bytes,
SUM(execution_count * avg_latency_seconds) AS total_latency_seconds,
SUM(execution_count * avg_rows) AS total_rows,
SUM(execution_count * avg_rows_scanned) AS total_rows_scanned,
SUM(execution_count * avg_cpu_seconds) / SUM(execution_count) AS avg_cpu_seconds,
SUM(execution_count * avg_bytes) / SUM(execution_count) AS avg_bytes,
SUM(execution_count * avg_latency_seconds) / SUM(execution_count) AS avg_latency_seconds,
SUM(execution_count * avg_rows) / SUM(execution_count) AS avg_rows,
SUM(execution_count * avg_rows_scanned) / SUM(execution_count) AS avg_rows_scanned,
FROM spanner_sys.query_stats_top_hour
WHERE INTERVAL_END > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
GROUP BY TEXT_FINGERPRINT
ORDER BY total_cpu_seconds DESC
```

<!--
Since it is easier to handle when imported into a spreadsheet, use [execspansql](https://github.com/apstndb/execspansql) to retrieve the query execution results in CSV.
-->

結果を共有するために CSV で出力して Google Sheets などのスプレッドシートにインポートするのも一つの方法です。

合計の負荷が大きい順に取得しているため、上位にあるものはどれもパフォーマンス改善上重要なクエリであると考えられます。観点としては下記のようなものがあります。

* `execution_count` が非常に大きい。
  * 単体では負荷とレイテンシは小さくても、細かい改善が負荷に有意な差をもたらす可能性があります。
* クエリ結果の `avg_rows` に対して `avg_rows_scanned` が2倍以上大きい。
  * 結果を返すために必要な最低限の行をセカンダリインデックスとテーブルの両方を読んでいる場合よりも多い場合は、無駄にスキャンして処理したあとで捨てられている行がある可能性があります。
  * 例外として集約関数を使っている場合は入力の `avg_rows_scanned` に対して出力は `avg_rows` は少なくなります。
* `avg_cpu_seconds`, `avg_latency_seconds` が大きい。
  * 単体で負荷が高いクエリは実行時に負荷のスパイクを作る場合があります。

<!--
Since the queries are taken in order of total load, any of the top ones are considered to be important queries for performance improvement. The following perspectives are available

* `execution_count` is very large.
    * Even if the load and latency are small by themselves, minor improvements may make a significant difference in load.
* `avg_rows_scanned` is more than twice as large as `avg_rows` in the query result.
    * If the minimum number of rows needed to return a result is more than that required to read both the secondary index and the table, it is possible that some rows are being discarded after being needlessly scanned and processed.
    * If you are using the Aggregate Function as an exception, `avg_rows` becomes less.
* `avg_cpu_seconds` and `avg_latency_seconds` are large.
    * Queries with high load by itself may create load spikes at runtime.
-->

<!-- ## Optimization Patterns for each SQL clauses -->

## SQL の句ごとの最適化パターン

<!--
* `WHERE`
    * As a general rule, if a table has enough rows that a full scan is not appropriate and a full scan is seen in the execution plan, a secondary index corresponding to the `WHERE` clause should be used to resolve the problem.
        * Only consider sharding when a global search is necessary for columns that would become hotspots if a secondary index is created for timestamps, sequential numbers, etc.
            * **"Useless use of sharding" is an anti-pattern.**
        * Filtering for timestamps may be more efficient than secondary indexes that require distributed joins if commit timestamp optimization is available on the base table.
    * Whenever possible, include filters in secondary indexes so that they are processed on the Index Scan side rather than the Table Scan side after the back join.
        * See: https://cloud.google.com/spanner/docs/query-execution-plans?hl=en#index_and_back_join_queries
    * Whenever possible, make sure filters are processed in Seek Condition rather than Residual Condition, which filters in-memory at scan time.
        * See: https://cloud.google.com/spanner/docs/query-execution-operators?hl=en#filter_scan
    * Ensure that the Seek Condition makes sense as a contiguous region scan.
        * Example: `UserId BETWEEN 0 AND 999999999 AND CreatedAt BETWEEN @from AND @to` for UserId, CreatedAt DESC is a Seek Condition and not a full scan. However, it is meaningless because it scans the area divided by countless UserId. If the frequency is high, introduce ShardId to treat it as a scan of a few contiguous regions.
-->

#### WHERE

* 原則として、一定以上の行数があるテーブルに対する OLTP 処理ではフルスキャンは許容できないので、実行計画上 Full Scan を見た場合は `WHERE` 句に対応したセカンダリインデックス等で解消すること。
    * タイムスタンプや連番など素直にセカンダリインデックスを作るとホットスポットになるカラムをグローバルに検索しなければならない場合にはアプリケーションレベルシャーディングを検討すること。
    * 要件を満たすために必要ではないアプリケーションレベルシャーディングの濫用はアンチパターンです。無闇にキー空間上の分布を均等にすることはランダムスキャンに対するレンジスキャンの優位性などを損なう危険があります。
* タイムスタンプに対するフィルタはベーステーブル上で Commit timestamp optimization できる場合は分散 JOIN が必要なセカンダリインデックスよりも効率的な場合があります。
* 可能な限り、 back join 後の Table Scan ではなく Index Scan 側でフィルタが処理されるようにセカンダリインデックスに含めること。
* 可能な限り、フィルタがスキャン時にインメモリでフィルタする Residual Condition ではなく Seek Condition で処理できていることを確認すること。
  * See also: https://cloud.google.com/spanner/docs/query-execution-operators?hl=en#filter_scan
* Seek Condition は連続領域のスキャンとして意味があることを確認すること。
  * 例: UserId, CreatedAt DESC に対する `UserId BETWEEN 0 AND 999999999 AND CreatedAt BETWEEN @from AND @to` は Seek Condition となり full scan にはならないが、無数の UserId ごとに分割された領域をスキャンすることになるため意味がない。頻度が高い場合は ShardId の導入などで数少ない連続領域のスキャンとして扱えるようにする。

<!--
* ORDER BY & LIMIT
    * Utilize PK or secondary indexes that match the `ORDER BY` order whenever possible, and ensure that Sort is not present in the execution plan.
        * Cloud Spanner indexes cannot be sorted in reverse order. Reference:.
            * https://cloud.google.com/spanner/docs/secondary-indexes?hl=en#index-scanning
    * Ensure that the amount of scanning is also reduced when the number specified in `LIMIT` is reduced.
    * If it is difficult to match the index order, `STORING` the columns written to `ORDER BY` so that `SORT LIMIT` can be performed on the index by itself.
-->

#### ORDER BY & LIMIT

* 可能な限り ORDER BY の順序と一致する PK もしくはセカンダリインデックスを活用し、 Sort が実行計画上存在しないことを確認すること。
* **Spanner のインデックスは逆順ソートができません。**
  * 参考: https://cloud.google.com/spanner/docs/secondary-indexes?hl=en#index-scanning
* `LIMIT` に指定した数を減らした際にスキャンする量も減ることを確認すること。
* もしもインデックス順と一致させることが難しい場合は、インデックス単体で `SORT LIMIT` を行うことができるように `ORDER BY` に書かれた列を `STORING` すること。


<!--
* Aggregate Function & `GROUP BY`
    * Columns used for Aggregate Function should be included in the secondary index as part of the key or in the form of `STORING` to avoid a large number of distributed `JOIN`s.
    * Ensure that the columns are scanned in the same order as the columns specified in `GROUP BY` whenever possible, so that they can be handled by Stream Aggregate rather than Hash Aggregate.
        * See: https://cloud.google.com/spanner/docs/query-execution-operators?hl=en#aggregate
-->

#### 集約関数 & `GROUP BY`

* 集約関数 に使われる列はキーの一部か `STORING` の形でセカンダリインデックスに含め、大量の分散 JOIN を回避すること。
可能な限り GROUP BY に指定した列と一致した順序でスキャンすることで、 Hash Aggregate ではなく Stream Aggregate で処理できることを確認する。
  * See: https://cloud.google.com/spanner/docs/query-execution-operators?hl=en#aggregate

<!--
* `JOIN` & Anti/Semi `JOIN`(`EXISTS`, `NOT EXISTS`, `IN` subqueries)
    * If possible, eliminate distributed JOINs by leveraging INTERLEAVE early in the schema design process.
        * https://cloud.google.com/spanner/docs/whitepapers/optimizing-schema-design?hl=en#storing_index_clause
    * If distributed `JOIN`s can be avoided by using an `INTERLEAVE`ed secondary index with the table being `JOIN`ed, consider doing so.
    * In other cases, Hash JOIN often requires the construction of a huge hash table because it cannot utilize the keys, so try to optimize by using Distributed Cross Apply or Merge JOIN with appropriate keys.
        * Distributed Cross Apply optimizes both Input and Map sides as much as possible. For example, include all columns in the JOIN condition in the secondary index.
        * Consider Merge `JOIN` when `JOIN`ing the same range in the same order.
-->

#### `JOIN` & Anti/Semi `JOIN`(`EXISTS`, `NOT EXISTS`, `IN` サブクエリ)

* 可能であれば、スキーマ設計の初期の時点で双方のテーブルを INTERLEAVE を活用することによって分散 JOIN 不要にすることを検討する。
* または JOIN 対象となるテーブルと INTERLEAVE されたセカンダリインデックスを使うことで分散 JOIN が回避できるかどうかも検討する。
* それ以外の場合でも Hash JOIN はキーを活用できず巨大なハッシュテーブルの構築を必要とする場合が多いので適切なキーを使った Distributed Cross Apply や Merge JOIN で最適化を試みること。
    * Distributed Cross Apply は Input 側、 Map 側双方をできるだけ最適化する。例えば、 JOIN 条件の列を全てセカンダリインデックスに含める。
        * https://cloud.google.com/spanner/docs/whitepapers/optimizing-schema-design?hl=en#storing_index_clause
    * 同じ順序の同じ範囲を JOIN する場合は Merge JOIN を検討する。

<!--
* SELECT
    * Back joins by SELECTing columns not in the secondary index are distributed JOINs for non-interleave indexes and incur non-negligible costs. If it is easier to scan only indexes, do so.
        * See: https://cloud.google.com/spanner/docs/query-execution-plans?hl=en#index_and_back_join_queries
    * Even for queries that require SELECTing nearly all of a table's columns, consider avoiding a distributed JOIN of the secondary index and base table by STORING in cases where read performance improvement is very important. For example:
        * Queries with very strict latency requirements that require stable performance under heavy load.
        * Queries that account for the majority of the load on the instance and can be expected to provide significant load reduction.
-->

## Examples

<!--
This section mainly uses the sample schema included in the official Cloud Spanner documentation.
-->

ここからは実際の実行計画とプロファイル情報を元に解説します。
慣れれば実行計画だけでも読み取れることは多いですが、数で説明した方がわかりやすいと思うので Spanner の公式ドキュメントに含まれるサンプルスキーマ上にランダムに入れたデータを使い解説します。

実行計画状のオペレータの処理については細かくは書きませんが、ある程度は公式にも解説があるので必要に応じて適当に読んでおいてください。

* https://cloud.google.com/spanner/docs/query-execution-plans
* https://cloud.google.com/spanner/docs/query-execution-operators

### WHERE

<!-- #### Avoid full scans and prefer Seek Condition -->
#### フルスキャンを避け、 Seek Condition になるようにする

<!--
As a general rule, make sure that queries with WHERE clauses are not full scans so that the more data you have, the slower the query will be.
However, the simple use of a secondary index often results in distributed JOINs, so using a secondary index may not improve performance in the following cases

* Data that is small and not expected to grow in volume (x = 100 or so)
    * For example, master data
* Queries that retrieve a large percentage (x > 20%) of the base table

Also, be sure to check not just that the query is using secondary indexes, but that they are being used efficiently.
-->

フルスキャンがあるクエリは通常データが増えるほど遅くなりますし、 `LIMIT` があったとしても read-write transaction の中で発行するとそのテーブルの全てをロックします。
そのようなクエリが生まれないように `WHERE` 句があるクエリはフルスキャンとして処理されていないかどうかを確認しましょう。
ただし、セカンダリインデックスの単純な使用は多くの場合分散 `JOIN` を発生するため、下記のような場合ではセカンダリインデックスを使ってもパフォーマンスが上がらない可能性もあります。

* 小さく、今後データ量が増えないことが想定されるデータ(例えば100行程度)であれば、範囲スキャンで全て読んでも良いかもしれません。
  * 例えばマスタデータなど
* ベーステーブルの多くの割合(例えば x > 20%)を取得するクエリは、セカンダリインデックスとベーステーブルの暗黙の分散 JOIN (バックジョイン) が発生することから、フルスキャンよりも重いかもしれません。
  * このようなクエリはそもそも OLTP では処理不可能なので、 BigQuery + Data Boost などのオフロードを検討しましょう。

また、単にクエリがセカンダリインデックスを使っていることを確認するだけでなく、効率的に使えていることを必ず確認しましょう。

Worse

```
spanner> EXPLAIN
-> SELECT s.SingerId
-> FROM Singers AS s
-> WHERE s.LastName = 'Smith';
+----+-------------------------------------------------------------------------------+
| ID | Query_Execution_Plan                                                          |
+----+-------------------------------------------------------------------------------+
|  0 | Distributed Union                                                             |
|  1 | +- Local Distributed Union                                                    |
|  2 |    +- Serialize Result                                                        |
| *3 |       +- Filter Scan                                                          |
|  4 |          +- Table Scan (Full scan: true, Table: Singers, scan_method: Scalar) |
+----+-------------------------------------------------------------------------------+
Predicates(identified by ID):
3: Residual Condition: ($LastName = 'Smith')
```

<!--This query is expected to slow down as the table grows larger because of the presence of full scans.-->
このクエリはフルスキャンが存在するため、テーブルが大きくなるにつれて遅くなることが想定されます。見るべき点は下記の2つです。

- ID 4 の Table Scan オペレータに `Full scan: true` がある。
- ID 3 の Filter Scan オペレータの Predicates は [Residual Condition](https://cloud.google.com/spanner/docs/query-execution-operators#filter_scan) であり、シーク範囲を狭くするために役立っていない。

Better

下記のようないセカンダリインデックスがあると、実行計画からフルスキャンはなくなります。

```
CREATE INDEX SingersByLastName ON Singers(LastName);
```

```
spanner> EXPLAIN
-> SELECT s.SingerId
-> FROM Singers@{FORCE_INDEX=SingersByLastName} AS s
-> WHERE s.LastName = 'Smith';
+----+------------------------------------------------------------------------+
| ID | Query_Execution_Plan                                                   |
+----+------------------------------------------------------------------------+
| *0 | Distributed Union                                                      |
|  1 | +- Local Distributed Union                                             |
|  2 |    +- Serialize Result                                                 |
| *3 |       +- Filter Scan                                                   |
|  4 |          +- Index Scan (Index: SingersByLastName, scan_method: Scalar) |
+----+------------------------------------------------------------------------+
Predicates(identified by ID):
0: Split Range: ($LastName = 'Smith')
3: Seek Condition: IS_NOT_DISTINCT_FROM($LastName, 'Smith')
```

<!-- Be sure to verify that you are not only scanning the secondary index, but that the primary filter is being handled by the Seek Condition of the Filter Scan. -->

セカンダリインデックスを使っているかどうかだけでなく、主要なフィルタが Filter Scan オペレータの Seek Condition として処理できているかどうかを確認しましょう。

See also: https://cloud.google.com/spanner/docs/query-execution-operators?hl=en#filter_scan

<!-- #### Avoid filtering after JOINs -->

#### JOIN 後よりも JOIN 前にフィルタできるようにする

<!-- Allowing filters to be processed on the secondary index side can greatly improve performance, even if the filter cannot be a Seek Condition on the secondary index side. -->

フィルタをセカンダリインデックス側の Seek Condition にはできないとしても、フィルタをセカンダリインデックス側で処理できるようにすることは大きくパフォーマンスを改善させることがあります。

Worse

```
spanner> EXPLAIN ANALYZE SELECT * FROM Songs@{FORCE_INDEX=SongsBySongName} WHERE SongName LIKE 'A%' AND Duration > 290;
+-----+----------------------------------------------------------------------------+---------------+------------+---------------+
| ID  | Query_Execution_Plan                                                       | Rows_Returned | Executions | Total_Latency |
+-----+----------------------------------------------------------------------------+---------------+------------+---------------+
|  *0 | Distributed Union                                                          | 815           | 1          | 163.97 msecs  |
|  *1 | +- Distributed Cross Apply                                                 | 815           | 1          | 163.82 msecs  |
|   2 |    +- [Input] Create Batch                                                 |               |            |               |
|   3 |    |  +- Local Distributed Union                                           | 16949         | 1          | 15.77 msecs   |
|   4 |    |     +- Compute Struct                                                 | 16949         | 1          | 14.38 msecs   |
|  *5 |    |        +- Filter Scan (seekable_key_size: 1)                          | 16949         | 1          | 9.9 msecs     |
|   6 |    |           +- Index Scan (Index: SongsBySongName, scan_method: Scalar) | 16949         | 1          | 7.65 msecs    |
|  20 |    +- [Map] Serialize Result                                               | 815           | 1          | 114 msecs     |
|  21 |       +- Cross Apply                                                       | 815           | 1          | 113.63 msecs  |
|  22 |          +- [Input] KeyRangeAccumulator                                    |               |            |               |
|  23 |          |  +- Batch Scan (Batch: $v2, scan_method: Scalar)                |               |            |               |
|  28 |          +- [Map] Local Distributed Union                                  | 815           | 16949      | 105.71 msecs  |
| *29 |             +- Filter Scan (seekable_key_size: 3)                          | 815           | 16949      | 99.63 msecs   |
|  30 |                +- Table Scan (Table: Songs, scan_method: Scalar)           | 815           | 16949      | 92.02 msecs   |
+-----+----------------------------------------------------------------------------+---------------+------------+---------------+
Predicates(identified by ID):
0: Split Range: STARTS_WITH($SongName, 'A')
1: Split Range: ($SingerId' = $SingerId)
5: Seek Condition: STARTS_WITH($SongName, 'A')
29: Seek Condition: (($SingerId' = $batched_SingerId) AND ($AlbumId' = $batched_AlbumId)) AND ($TrackId' = $batched_TrackId)
Residual Condition: ($Duration > 290)

815 rows in set (173.22 msecs)
timestamp:            2023-06-21T15:23:13.695614+09:00
cpu time:             165.91 msecs
rows scanned:         33898 rows
deleted rows scanned: 0 rows
optimizer version:    5
optimizer statistics: auto_20230619_14_33_08UTC
```

<!--
Since the Duration column is not included in the secondary index SongsBySongName, this query does a distributed JOIN of over 16000 entries in the Songs table and then filters for Duration in the Table Scan side Filter Scan.
As a result, this query is not fast.
-->

`Duration` カラムはセカンダリインデックス `SongsBySongName` に含まれていないため、このクエリは 16949行をセカンダリインデックスから読み取った後で、全ての列を取得するために Songs テーブルと分散 JOIN してから Table Scan 側の Filter Scan で `Duration` のフィルタを行い、最終的な815行の結果を得ています。
つまり、わざわざ分散 JOIN した16000行以上を捨てています。
この例では 170ms 程度で結果が返ってくるので許容可能な場合もあるかもしれませんが、大部分を占める処理が無駄になることはあまり良いこととは言えませんね。

Better

```
CREATE INDEX SongsBySongNameStoring ON Songs(SongName) STORING(Duration);
```

```
spanner> EXPLAIN ANALYZE SELECT * FROM Songs@{FORCE_INDEX=SongsBySongNameStoring} WHERE SongName LIKE 'A%' AND Duration > 290;
+-----+-----------------------------------------------------------------------------------+---------------+------------+---------------+
| ID  | Query_Execution_Plan                                                              | Rows_Returned | Executions | Total_Latency |
+-----+-----------------------------------------------------------------------------------+---------------+------------+---------------+
|  *0 | Distributed Union                                                                 | 815           | 1          | 16.47 msecs   |
|  *1 | +- Distributed Cross Apply                                                        | 815           | 1          | 16.34 msecs   |
|   2 |    +- [Input] Create Batch                                                        |               |            |               |
|   3 |    |  +- Local Distributed Union                                                  | 815           | 1          | 5.43 msecs    |
|   4 |    |     +- Compute Struct                                                        | 815           | 1          | 5.36 msecs    |
|  *5 |    |        +- Filter Scan (seekable_key_size: 1)                                 | 815           | 1          | 5.06 msecs    |
|   6 |    |           +- Index Scan (Index: SongsBySongNameStoring, scan_method: Scalar) | 815           | 1          | 4.94 msecs    |
|  26 |    +- [Map] Serialize Result                                                      | 815           | 1          | 9.53 msecs    |
|  27 |       +- Cross Apply                                                              | 815           | 1          | 9.27 msecs    |
|  28 |          +- [Input] KeyRangeAccumulator                                           |               |            |               |
|  29 |          |  +- Batch Scan (Batch: $v2, scan_method: Scalar)                       |               |            |               |
|  35 |          +- [Map] Local Distributed Union                                         | 815           | 815        | 8.77 msecs    |
| *36 |             +- Filter Scan (seekable_key_size: 3)                                 | 815           | 815        | 8.4 msecs     |
|  37 |                +- Table Scan (Table: Songs, scan_method: Scalar)                  | 815           | 815        | 7.88 msecs    |
+-----+-----------------------------------------------------------------------------------+---------------+------------+---------------+
Predicates(identified by ID):
0: Split Range: STARTS_WITH($SongName, 'A')
1: Split Range: ($SingerId' = $SingerId)
5: Seek Condition: STARTS_WITH($SongName, 'A')
Residual Condition: ($Duration > 290)
36: Seek Condition: (($SingerId' = $batched_SingerId) AND ($AlbumId' = $batched_AlbumId)) AND ($TrackId' = $batched_TrackId)

815 rows in set (28.69 msecs)
timestamp:            2023-06-21T15:23:43.307113+09:00
cpu time:             28.27 msecs
rows scanned:         17764 rows
deleted rows scanned: 0 rows
optimizer version:    5
optimizer statistics: auto_20230619_14_33_08UTC
```

<!--
If Duration can be added as a STORING column in the secondary index, the number of distributed JOINs can be reduced and performance can be improved.
Since the STORING column is not part of the key, it must be skipped as a Residual Condition, so the rows scanned will be relatively large. However, Cloud Spanner is good at scanning contiguous regions, so the performance is better than before the improvement, where there are many random accesses with remote calls.

In general, if all queries are accessed efficiently with Seek Condition, the number of secondary indexes will increase, which will reduce write cost and maintainability, but it is easy to increase the number of columns for STORING.
It is preferable to add columns used in filters to the STORING columns whenever possible.
-->

セカンダリインデックスの STORING 列として `Duration` を追加することができれば、分散 JOIN の件数が削減でき、パフォーマンスを改善できます。
STORING した列はキーの一部ではなく Seek Condition としては処理することはできず、 Residual Condition として読み飛ばして処理する必要があるため rows scanned はこのケースでも17000行を超えており、無駄な処理は残っています。しかし Spanner は連続領域のスキャンは得意なため、リモートコールを伴うランダムアクセスが多い改善前と比較すればパフォーマンスは大幅によくなります。

それぞれのクエリを Seek Condition で効率的にアクセスできるようにしようとするとクエリごとに最適化したセカンダリインデックスの数が増えていき、結果として書き込みコストやメンテナンス性が悪化していく一方です。
それに対し既存のセカンダリインデックスに STORING するカラムを増やすことは [`ALTER INDEX ADD STORED COLUMN`](https://cloud.google.com/spanner/docs/reference/standard-sql/data-definition-language#alter-index) を使うことでセカンダリインデックスの数を増やす必要なく行うことができます。
セカンダリインデックスが増えて分散コミットに参加するスプリット(participants)が増えることに比べれば STORING 列が増えることは大きな書き込みコスト増加ではないため、比較的 Write のコストを上げづらい選択肢と考えることができます。
カバリングインデックスにするところまでは踏み切れないとしても、フィルタで使うカラムは STORING 列に追加することは積極的に選択肢に入れて良いでしょう。

<!-- ### Avoid inefficient timestamp scans -->

### タイムスタンプを使った非効率なスキャンを避ける

<!--
Because timestamp values can cause hotspots in writes, a requirement for a global secondary index is preferable to avoid if possible.
However, it is common to want to filter globally by timestamp.
-->


タイムスタンプ値は書き込みでのホットスポットを避けるため、グローバルなセカンダリインデックスを避けることがベストプラクティスです。しかし、タイムスタンプによるグローバルなフィルタをしたい場合が珍しくありません。

```
CREATE TABLE UserAccessLog (
LastAccess TIMESTAMP NOT NULL,
UserId INT64 NOT NULL,
) PRIMARY KEY (UserId, LastAccess);
```

<!--
The primary key of this table is timestamped per UserId, which is a good design to avoid hotspots even if LastAccess is monotonically increasing.
However, this key design is only suitable for queries filtered by timestamp per UserId. It is not suitable for filtering by timestamp across users.
-->

このテーブルのプライマリキーは `UserId` ごとにタイムスタンプが設定されているため `LastAccess` が短調増加であってもホットスポットとならない良い設計となっています。
しかし、このキー設計は `UserId` ごとにタイムスタンプでフィルタするクエリのみに適しています。ユーザを跨いだタイムスタンプでのフィルタには適しません。


Worse

```
spanner> EXPLAIN SELECT * FROM UserAccessLog WHERE LastAccess BETWEEN @fromLastAccess AND @toLastAccess;
+----+-------------------------------------------------------------------------------------+
| ID | Query_Execution_Plan                                                                |
+----+-------------------------------------------------------------------------------------+
| *0 | Distributed Union                                                                   |
|  1 | +- Local Distributed Union                                                          |
|  2 |    +- Serialize Result                                                              |
| *3 |       +- Filter Scan                                                                |
|  4 |          +- Table Scan (Full scan: true, Table: UserAccessLog, scan_method: Scalar) |
+----+-------------------------------------------------------------------------------------+
Predicates(identified by ID):
0: Split Range: BETWEEN($LastAccess, @fromlastaccess, @tolastaccess)
3: Residual Condition: BETWEEN($LastAccess, @fromlastaccess, @tolastaccess)
```

<!-- You often see queries that add a filter for UserId because it is easy to see that filtering only for LastAccess is inefficient because it is a full scan. (Here, fromUserId and toUserId are the lower and upper limits of values that UserId can take.) -->

`LastAccess` のみに対するフィルタではフルスキャンとなっており非効率なことがわかりやすいため、 `UserId` へのフィルタを追加しているクエリをよく見ます。(ここでは `fromUserId` と `toUserId` には `UserId` が取る値の下限と上限が入ります。)


```
spanner> EXPLAIN SELECT * FROM UserAccessLog WHERE UserId BETWEEN @fromUserId AND @toUserId AND LastAccess BETWEEN @fromLastAccess AND @toLastAccess;
+----+--------------------------------------------------------------------+
| ID | Query_Execution_Plan                                               |
+----+--------------------------------------------------------------------+
| *0 | Distributed Union                                                  |
|  1 | +- Local Distributed Union                                         |
|  2 |    +- Serialize Result                                             |
| *3 |       +- Filter Scan (seekable_key_size: 2)                        |
|  4 |          +- Table Scan (Table: UserAccessLog, scan_method: Scalar) |
+----+--------------------------------------------------------------------+
Predicates(identified by ID):
0: Split Range: (BETWEEN($UserId, @fromuserid, @touserid) AND BETWEEN($LastAccess, @fromlastaccess, @tolastaccess))
3: Seek Condition: (BETWEEN($UserId, @fromuserid, @touserid) AND BETWEEN($LastAccess, @fromlastaccess, @tolastaccess))
```

<!--
At first glance, this query looks fine because it consists only of a Seek Condition, not a full scan.
However, the large number of rows per UserId that hit `LastAccess BETWEEN @fromLastAccess AND @toLastAccess` makes it an anti-pattern with little performance improvement.
If such queries are executed frequently or constitute a peak load, they are candidates for optimization.
-->

このクエリはフルスキャンではなく、 Seek Condition のみからなるため一見問題ないように見えます。
しかし、 `LastAccess BETWEEN @fromLastAccess AND @toLastAccess` にヒットする行が大量の `UserId` ごとに存在するため、パフォーマンスの改善効果は低いアンチパターンであると言えます。
このようなクエリが頻繁に実行されたり、ピーク負荷を構成している場合は最適化の候補です。


Better

<!--
For example, a secondary index could be created using a ShardId with a value of 20 based on the hash value of the UserId.
This secondary index not only avoids hotspots by spreading the writes to the 20 to the latest timestamp,
range scan within the 20 separate regions, it may also improve the efficiency of queries that filter timestamps against the entire table.
-->

例えば、 UserId のハッシュ値を元にした20の値を持つ ShardId を使ったセカンダリインデックスを作ることができます。次のような効果が期待できます。

- `UserId` よりは分散していませんが、最新のタイムスタンプへの書き込みを20に分散させることでホットスポットを回避することができます。
  - より大きな PU を持つインスタンスで均等に分散させたい場合は `ShardId` を大きくする必要があるかもしれません。
- 20個に分かれた領域それぞれへの範囲スキャンにすることで、テーブル全体に対するタイムスタンプのフィルタを行うクエリの効率も改善する場合があります。

See also: https://cloud.google.com/spanner/docs/generated-column/how-to?hl=en

```
ALTER TABLE UserAccessLog ADD COLUMN ShardId INT64 NOT NULL AS (MOD(ABS(FARM_FINGERPRINT(CAST(UserId AS STRING))), 20)) STORED;
CREATE INDEX UserAccessLogByShardIdLastAccess ON UserAccessLog(ShardId, LastAccess DESC);
```

```
spanner> EXPLAIN SELECT * FROM UserAccessLog WHERE ShardId BETWEEN @fromShardId AND @toShardId AND LastAccess BETWEEN @fromLastAccess AND @toLastAccess;
+----+---------------------------------------------------------------------------------------+
| ID | Query_Execution_Plan                                                                  |
+----+---------------------------------------------------------------------------------------+
| *0 | Distributed Union                                                                     |
|  1 | +- Local Distributed Union                                                            |
|  2 |    +- Serialize Result                                                                |
| *3 |       +- Filter Scan (seekable_key_size: 2)                                           |
|  4 |          +- Index Scan (Index: UserAccessLogByShardIdLastAccess, scan_method: Scalar) |
+----+---------------------------------------------------------------------------------------+
Predicates(identified by ID):
0: Split Range: (BETWEEN($ShardId, @fromshardid, @toshardid) AND BETWEEN($LastAccess, @fromlastaccess, @tolastaccess))
3: Seek Condition: (BETWEEN($ShardId, @fromshardid, @toshardid) AND BETWEEN($LastAccess, @fromlastaccess, @tolastaccess))
```

:::message
`LIMIT` や `ORDER BY` を含むクエリをアプリケーションレベルシャーディングされたテーブルで利かせたい場合、より特殊な最適化が必要となります。下記の StackOverflow の回答などを参照してください。

https://stackoverflow.com/a/62094370

使える場面が限定されますが、`allow_commit_timestamp=true` のカラムへのフィルタを含むクエリのスキャンに対する最適化が存在するため、セカンダリインデックスなしでも効率的に処理できる場合があります。
実際に最適化が行われることを rows scanned の値から確認した上でこちらを使うことも検討すると良いでしょう。

https://cloud.google.com/spanner/docs/commit-timestamp#optimize
:::

<!--
Note

To make LIMIT work with ORDER BY across ShardId, optimization is needed, such as using a correlated subquery.

See also https://stackoverflow.com/a/62094370

Although its use is limited, filters on columns with allow_commit_timestamp=true can be processed efficiently without a secondary index because the scan is optimized.
If you do not need ORDER BY & LIMIT, consider using this option after verifying from the rows scanned value that the optimization actually occurs.

https://cloud.google.com/spanner/docs/commit-timestamp#optimize
-->

<!--### Aggregate Functions & `GROUP BY`-->

### 集約関数と `GROUP BY`

<!-- #### Use STORING for columns to be aggregated -->

#### 集約対象のカラムを `STORING` する

Worse

```
CREATE INDEX SongsBySongGenre ON Songs(SongGenre);
```

```
+-----+--------------------------------------------------------------------------------------+---------------+------------+---------------+
| ID  | Query_Execution_Plan                                                                 | Rows_Returned | Executions | Total_Latency |
+-----+--------------------------------------------------------------------------------------+---------------+------------+---------------+
|   0 | Serialize Result                                                                     | 1             | 1          | 1.5 secs      |
|   1 | +- Global Stream Aggregate                                                           | 1             | 1          | 1.5 secs      |
|  *2 |    +- Distributed Union                                                              | 1             | 1          | 1.5 secs      |
|   3 |       +- Local Stream Aggregate                                                      | 1             | 1          | 1.5 secs      |
|  *4 |          +- Distributed Cross Apply                                                  | 13            | 1          | 1.5 secs      |
|   5 |             +- [Input] Create Batch                                                  |               |            |               |
|   6 |             |  +- Local Distributed Union                                            | 256844        | 1          | 145.19 msecs  |
|   7 |             |     +- Compute Struct                                                  | 256844        | 1          | 129.11 msecs  |
|  *8 |             |        +- Filter Scan                                                  |               |            |               |
|   9 |             |           +- Index Scan (Index: SongsBySongGenre, scan_method: Scalar) | 256844        | 1          | 79.61 msecs   |
|  23 |             +- [Map] Local Stream Aggregate                                          | 13            | 13         | 1.1 secs      |
|  24 |                +- Cross Apply                                                        | 256844        | 13         | 1.07 secs     |
|  25 |                   +- [Input] KeyRangeAccumulator                                     |               |            |               |
|  26 |                   |  +- Batch Scan (Batch: $v6, scan_method: Scalar)                 |               |            |               |
|  31 |                   +- [Map] Local Distributed Union                                   | 256844        | 256844     | 959.29 msecs  |
| *32 |                      +- Filter Scan (seekable_key_size: 3)                           | 256844        | 256844     | 867.3 msecs   |
|  33 |                         +- Table Scan (Table: Songs, scan_method: Scalar)            | 256844        | 256844     | 729.42 msecs  |
+-----+--------------------------------------------------------------------------------------+---------------+------------+---------------+
Predicates(identified by ID):
2: Split Range: ($SongGenre = 'ROCK')
4: Split Range: ($Songs_key_SingerId' = $Songs_key_SingerId)
8: Seek Condition: IS_NOT_DISTINCT_FROM($SongGenre, 'ROCK')
32: Seek Condition: (($Songs_key_SingerId' = $batched_Songs_key_SingerId) AND ($Songs_key_AlbumId' = $batched_Songs_key_AlbumId)) AND ($Songs_key_TrackId' = $batched_Songs_key_TrackId)

1 rows in set (1.51 secs)
timestamp:            2023-06-19T14:46:42.066088+09:00
cpu time:             1.5 secs
rows scanned:         513688 rows
deleted rows scanned: 0 rows
optimizer version:    5
optimizer statistics: auto_20230616_13_28_03UTC
```

<!-- Aggregate can only be processed after distributed JOINs by Distributed Cross Apply, resulting in a large number of distributed JOINs that are proportional to the number of rows to be aggregated. -->

Aggregate を Distributed Cross Apply による分散 JOIN 後にしか処理できなくなっているため、集計対象の行数に比例する大量の分散 JOIN が発生しています。


Better

<!--
Adding columns used by the Aggregate Function to STORING can eliminate the need for distributed JOINs.
The same effect may be achieved by including the necessary columns in the secondary index key, but STORING can also be added with `ALTER INDEX ADD STORED COLUMN`. This is easier to maintain with fewer changes when more columns are used.
-->


Aggregate Function で使う列を STORING することで分散 JOIN を不要にすることができます。
必要な列をセカンダリインデックスのキーに含めることでも同様の効果を得ることができるかもしれませんが、 STORING は `ALTER INDEX ADD STORED COLUMN` で既存のセカンダリインデックスに追加することも可能です。使用する列が増えた際にも変更が少なくなりメンテナンス性に優れています。


```
CREATE INDEX SongsBySongGenreStoring ON Songs(SongGenre) STORING (Duration);
```


```
spanner> EXPLAIN ANALYZE
-> SELECT s.SongGenre, AVG(s.duration) AS average, COUNT(*) AS count
-> FROM Songs@{FORCE_INDEX=SongsBySongGenreStoring} AS s
-> WHERE SongGenre = 'ROCK'
-> GROUP BY SongGenre;
+----+------------------------------------------------------------------------------------+---------------+------------+---------------+
| ID | Query_Execution_Plan                                                               | Rows_Returned | Executions | Total_Latency |
+----+------------------------------------------------------------------------------------+---------------+------------+---------------+
|  0 | Serialize Result                                                                   | 1             | 1          | 98.23 msecs   |
|  1 | +- Global Stream Aggregate                                                         | 1             | 1          | 98.22 msecs   |
| *2 |    +- Distributed Union                                                            | 1             | 1          | 98.22 msecs   |
|  3 |       +- Local Stream Aggregate                                                    | 1             | 1          | 98.2 msecs    |
|  4 |          +- Local Distributed Union                                                | 256844        | 1          | 81.02 msecs   |
| *5 |             +- Filter Scan                                                         |               |            |               |
|  6 |                +- Index Scan (Index: SongsBySongGenreStoring, scan_method: Scalar) | 256844        | 1          | 69.71 msecs   |
+----+------------------------------------------------------------------------------------+---------------+------------+---------------+
Predicates(identified by ID):
2: Split Range: ($SongGenre = 'ROCK')
5: Seek Condition: IS_NOT_DISTINCT_FROM($SongGenre, 'ROCK')

1 rows in set (101.46 msecs)
timestamp:            2023-06-19T14:47:25.085188+09:00
cpu time:             101.05 msecs
rows scanned:         256844 rows
deleted rows scanned: 0 rows
optimizer version:    5
optimizer statistics: auto_20230616_13_28_03UTC
```

<!--
Distributed JOINs are no longer necessary, and the stream aggregation is simpler than Hash Aggregate, so it is faster than before STORING.
-->

このように分散 JOIN は不要となり、ハッシュテーブルを作る必要がある Hash Aggregate よりも単純な Stream Aggregate となるため、 STORING する前よりも高速かつメモリ等のリソースの使用量も少なくなります。


### ORDER BY & LIMIT

<!--
#### Use descending ordered index for descending sort
-->

#### 降順ソートには降順インデックスを使う

Worse

<!--
Since Cloud Spanner cannot scan in reverse order, an explicit sort will occur if there are only indexes in reverse order to the ORDER BY clause.
-->

他のデータベースではインデックスを逆順に辿ることができるため降順インデックスの概念すら無かったものもあります。しかし、 Spanner はテーブルやインデックスを逆順にスキャンすることはできないため、 `ORDER BY` 句とは逆順のインデックスしかない場合、必ず明示的なソートが発生します。


```
CREATE INDEX SongsBySongName ON Songs(SongName);
```

```
spanner> EXPLAIN ANALYZE
-> SELECT *
-> FROM Songs@{FORCE_INDEX=SongsBySongName}
-> ORDER BY SongName DESC
-> LIMIT 1;
+-----+---------------------------------------------------------------------------------------------------+---------------+------------+---------------+
| ID  | Query_Execution_Plan                                                                              | Rows_Returned | Executions | Total_Latency |
+-----+---------------------------------------------------------------------------------------------------+---------------+------------+---------------+
|   0 | Global Limit                                                                                      | 1             | 1          | 457.31 msecs  |
|  *1 | +- Distributed Cross Apply (order_preserving: true)                                               | 1             | 1          | 457.31 msecs  |
|   2 |    +- [Input] Create Batch                                                                        |               |            |               |
|   3 |    |  +- Compute Struct                                                                           | 1             | 1          | 455.86 msecs  |
|   4 |    |     +- Local Limit                                                                           | 1             | 1          | 455.85 msecs  |
|   5 |    |        +- Distributed Union (preserve_subquery_order: true)                                  | 1             | 1          | 455.85 msecs  |
|   6 |    |           +- Local Sort Limit                                                                | 1             | 1          | 455.84 msecs  |
|   7 |    |              +- Local Distributed Union                                                      | 1024000       | 1          | 375.54 msecs  |
|   8 |    |                 +- Index Scan (Full scan: true, Index: SongsBySongName, scan_method: Scalar) | 1024000       | 1          | 326.32 msecs  |
|  28 |    +- [Map] Serialize Result                                                                      | 1             | 1          | 1.41 msecs    |
|  29 |       +- MiniBatchKeyOrder                                                                        |               |            |               |
|  30 |          +- Minor Sort Limit                                                                      | 1             | 1          | 1.4 msecs     |
|  31 |             +- RowCount                                                                           |               |            |               |
|  32 |                +- Cross Apply                                                                     | 1             | 1          | 1.39 msecs    |
|  33 |                   +- [Input] RowCount                                                             |               |            |               |
|  34 |                   |  +- KeyRangeAccumulator                                                       |               |            |               |
|  35 |                   |     +- Local Minor Sort                                                       |               |            |               |
|  36 |                   |        +- MiniBatchAssign                                                     |               |            |               |
|  37 |                   |           +- Batch Scan (Batch: $v2, scan_method: Scalar)                     | 1             | 1          | 0 msecs       |
|  52 |                   +- [Map] Local Distributed Union                                                | 1             | 1          | 1.37 msecs    |
| *53 |                      +- Filter Scan (seekable_key_size: 3)                                        | 1             | 1          | 1.37 msecs    |
|  54 |                         +- Table Scan (Table: Songs, scan_method: Scalar)                         | 1             | 1          | 1.36 msecs    |
+-----+---------------------------------------------------------------------------------------------------+---------------+------------+---------------+
Predicates(identified by ID):
1: Split Range: ($SingerId' = $sort_SingerId)
53: Seek Condition: (($SingerId' = $sort_batched_SingerId) AND ($AlbumId' = $sort_batched_AlbumId)) AND ($TrackId' = $sort_batched_TrackId)

1 rows in set (500.07 msecs)
timestamp:            2023-06-19T14:13:16.792848+09:00
cpu time:             470.41 msecs
rows scanned:         1024001 rows
deleted rows scanned: 0 rows
optimizer version:    5
optimizer statistics: auto_20230616_13_28_03UTC
```

<!--
In this case, since Limit is not possible without Sort, we can see that even with `LIMIT 1`, 1024000 rows (all test data) are scanned and then sorted and discarded using Local Sort Limit.
In general, queries like this that do not reduce work to process even with a smaller `LIMIT` amount are candidates for optimization because the larger the database, the heavier the query becomes.
-->

この場合 Sort なしに Limit できないため、 `LIMIT 1` であっても 1024000行(テストデータ全て)をスキャンしてから Local Sort Limit でソートして捨てていることが分かります。
このようなクエリはデータベースが大きくなるほど重くなるためスケールせず、最適化の候補です。

Better

<!--
To optimize `ORDER BY DESC` queries, explicitly define DESC ordered indexes.
-->


`DESC` で `ORDER BY` するクエリを最適化したい場合は明示的に `DESC` 順にしたインデックスを定義します。

```
CREATE INDEX SongsBySongNameDesc ON Songs(SongName DESC);
```


```
spanner> EXPLAIN ANALYZE
-> SELECT *
-> FROM Songs@{FORCE_INDEX=SongsBySongNameDesc}
-> ORDER BY SongName DESC
-> LIMIT 1;
+-----+-------------------------------------------------------------------------------------------------------+---------------+------------+---------------+
| ID  | Query_Execution_Plan                                                                                  | Rows_Returned | Executions | Total_Latency |
+-----+-------------------------------------------------------------------------------------------------------+---------------+------------+---------------+
|   0 | Global Limit                                                                                          | 1             | 1          | 0.33 msecs    |
|  *1 | +- Distributed Cross Apply (order_preserving: true)                                                   | 1             | 1          | 0.33 msecs    |
|   2 |    +- [Input] Create Batch                                                                            |               |            |               |
|   3 |    |  +- Compute Struct                                                                               | 1             | 1          | 0.19 msecs    |
|   4 |    |     +- Local Limit                                                                               | 1             | 1          | 0.19 msecs    |
|   5 |    |        +- Distributed Union                                                                      | 1             | 1          | 0.19 msecs    |
|   6 |    |           +- Local Limit                                                                         | 1             | 1          | 0.16 msecs    |
|   7 |    |              +- Local Distributed Union                                                          | 1             | 1          | 0.16 msecs    |
|   8 |    |                 +- Index Scan (Full scan: true, Index: SongsBySongNameDesc, scan_method: Scalar) | 1             | 1          | 0.16 msecs    |
|  23 |    +- [Map] Serialize Result                                                                          | 1             | 1          | 0.1 msecs     |
|  24 |       +- MiniBatchKeyOrder                                                                            |               |            |               |
|  25 |          +- Minor Sort Limit                                                                          | 1             | 1          | 0.1 msecs     |
|  26 |             +- RowCount                                                                               |               |            |               |
|  27 |                +- Cross Apply                                                                         | 1             | 1          | 0.09 msecs    |
|  28 |                   +- [Input] RowCount                                                                 |               |            |               |
|  29 |                   |  +- KeyRangeAccumulator                                                           |               |            |               |
|  30 |                   |     +- Local Minor Sort                                                           |               |            |               |
|  31 |                   |        +- MiniBatchAssign                                                         |               |            |               |
|  32 |                   |           +- Batch Scan (Batch: $v2, scan_method: Scalar)                         | 1             | 1          | 0 msecs       |
|  47 |                   +- [Map] Local Distributed Union                                                    | 1             | 1          | 0.08 msecs    |
| *48 |                      +- Filter Scan (seekable_key_size: 3)                                            | 1             | 1          | 0.08 msecs    |
|  49 |                         +- Table Scan (Table: Songs, scan_method: Scalar)                             | 1             | 1          | 0.08 msecs    |
+-----+-------------------------------------------------------------------------------------------------------+---------------+------------+---------------+
Predicates(identified by ID):
1: Split Range: ($SingerId' = $SingerId)
48: Seek Condition: (($SingerId' = $sort_batched_SingerId) AND ($AlbumId' = $sort_batched_AlbumId)) AND ($TrackId' = $sort_batched_TrackId)

1 rows in set (9.23 msecs)
timestamp:            2023-06-19T14:18:57.213425+09:00
cpu time:             9.19 msecs
rows scanned:         2 rows
deleted rows scanned: 0 rows
optimizer version:    5
optimizer statistics: auto_20230616_13_28_03UTC
```

<!--
Sorting is no longer necessary, and scanning just one row in index order does both ORDER BY and LIMIT jobs.
-->
Sort オペレーションは必要なくなり、インデックス順に 1 行だけスキャンすることで `ORDER BY` と `LIMIT` 両方の仕事をできるようになりました。

:::message
余談ですが、実行計画では Sort オペレーションがなくてもこのクエリの結果は順序が保証されますが、 `ORDER BY` を削除してしまうと順序の保証はなくなります。
他のデータベースで先入観があるかもしれませんが、 `ORDER BY` なしにセカンダリインデックスやプライマリキーの順序になることを期待しないようにしましょう。

https://cloud.google.com/spanner/docs/sql-best-practices#use_order_by_to_ensure_the_ordering_of_your_sql_results
> Spanner guarantees result ordering only if the ORDER BY clause is present in the query.

### JOIN (TODO)

See also
https://cloud.google.com/spanner/docs/sql-best-practices?hl=en#optimize_joins

### SELECT
<!--
#### Avoid back joins if needed
-->

#### 必要に応じてバックジョインを避ける

<!--
The process of retrieving a table row from a secondary index is called a back join.
In Cloud Spanner, a distributed database, back join is expensive because it is almost always a distributed join.
-->

Spanner のドキュメント上ではセカンダリインデックスからテーブルの行を取得する暗黙の JOIN は back join(邦訳だとバック結合) と呼ばれます。
他のデータベースだとインデックスからテーブルのルックアップに相当するこの back join は分散データベースである Spanner では多くの場合分散 JOIN を意味するため、リモートコールを要し本質的に安価ではありません。

<!--
See also
-->

下記も参照してください。

- https://cloud.google.com/spanner/docs/query-execution-plans?hl=en#index_and_back_join_queries
- https://cloud.google.com/blog/products/databases/how-to-use-a-storing-clause-in-cloud-spanner/

Worse

<!--
If you try to use a Duration column that is not included in the SongsBySongName index, you can see on the execution plan that a distributed JOIN using Distributed Cross Apply will occur.```
-->

下記の例では `SongsBySongName` インデックスに含まれていない `Duration` 列を使おうとすると、 Distributed Cross Apply を使った分散 JOIN (ID: 1) が発生することが実行計画上から確認できます。

```
CREATE INDEX SongsBySongName ON Songs(SongName);
```

```
spanner> EXPLAIN ANALYZE
-> SELECT s.SongName, s.Duration
-> FROM Songs@{force_index=SongsBySongName} AS s
-> WHERE STARTS_WITH(s.SongName, "B");
+-----+----------------------------------------------------------------------------+---------------+------------+---------------+
| ID  | Query_Execution_Plan                                                       | Rows_Returned | Executions | Total_Latency |
+-----+----------------------------------------------------------------------------+---------------+------------+---------------+
|  *0 | Distributed Union                                                          | 78940         | 1          | 1.41 secs     |
|  *1 | +- Distributed Cross Apply                                                 | 78940         | 1          | 1.39 secs     |
|   2 |    +- [Input] Create Batch                                                 |               |            |               |
|   3 |    |  +- Local Distributed Union                                           | 78940         | 1          | 85.15 msecs   |
|   4 |    |     +- Compute Struct                                                 | 78940         | 1          | 79.96 msecs   |
|  *5 |    |        +- Filter Scan (seekable_key_size: 1)                          | 78940         | 1          | 53.06 msecs   |
|   6 |    |           +- Index Scan (Index: SongsBySongName, scan_method: Scalar) | 78940         | 1          | 41.13 msecs   |
|  20 |    +- [Map] Serialize Result                                               | 78940         | 4          | 1.12 secs     |
|  21 |       +- Cross Apply                                                       | 78940         | 4          | 1.09 secs     |
|  22 |          +- [Input] KeyRangeAccumulator                                    |               |            |               |
|  23 |          |  +- Batch Scan (Batch: $v2, scan_method: Scalar)                |               |            |               |
|  28 |          +- [Map] Local Distributed Union                                  | 78940         | 78940      | 1.05 secs     |
| *29 |             +- Filter Scan (seekable_key_size: 3)                          | 78940         | 78940      | 1.01 secs     |
|  30 |                +- Table Scan (Table: Songs, scan_method: Scalar)           | 78940         | 78940      | 955.6 msecs   |
+-----+----------------------------------------------------------------------------+---------------+------------+---------------+
Predicates(identified by ID):
0: Split Range: STARTS_WITH($SongName, 'B')
1: Split Range: ($Songs_key_SingerId' = $Songs_key_SingerId)
5: Seek Condition: STARTS_WITH($SongName, 'B')
29: Seek Condition: (($Songs_key_SingerId' = $batched_Songs_key_SingerId) AND ($Songs_key_AlbumId' = $batched_Songs_key_AlbumId)) AND ($Songs_key_TrackId' = $batched_Songs_key_TrackId)
```

Better

<!--
These distributed JOINs can be avoided by index-only scan, just as in a general RDBMS.
To perform index-only scan, a combination of the following two methods is used.
-->

このようなインデックスからテーブルデータを取得するための分散 JOIN を回避するために、 Spanner でも一般的な RDBMS でもよくあるテクニックを使うことができます。
テーブルから行を取得せずにインデックスから必要な列を全て取得する index-only scan です。

index-only scan にするためにはクエリを変えるか、インデックスを変えるか、その両方の組み合わせを考えることになります。

<!-- ##### A: Reduce SELECT-ed columns -->
##### A: クエリを変え、 `SELECT` する列を減らす

<!--
If you can reduce the number of columns to SELECT, you may be able to eliminate distributed JOINs even with the secondary indexes we have.
There are many queries that SELECT all columns, but do you really need all columns?
-->

`SELECT` する列を減らせば、今あるセカンダリインデックスのままでも back join を不要にできるかもしれません。
DTO や構造体に詰めるために全カラムを `SELECT` するクエリは多いですが、本当にそのロジックで全ての列が必要でしょうか？

```
spanner> EXPLAIN ANALYZE
-> SELECT s.SongName
-> FROM Songs@{force_index=SongsBySongName} AS s
-> WHERE STARTS_WITH(s.SongName, "B");
+----+----------------------------------------------------------------------+---------------+------------+---------------+
| ID | Query_Execution_Plan                                                 | Rows_Returned | Executions | Total_Latency |
+----+----------------------------------------------------------------------+---------------+------------+---------------+
| *0 | Distributed Union                                                    | 78940         | 1          | 51.1 msecs    |
|  1 | +- Local Distributed Union                                           | 78940         | 1          | 44.26 msecs   |
|  2 |    +- Serialize Result                                               | 78940         | 1          | 40.99 msecs   |
| *3 |       +- Filter Scan (seekable_key_size: 1)                          | 78940         | 1          | 29.82 msecs   |
|  4 |          +- Index Scan (Index: SongsBySongName, scan_method: Scalar) | 78940         | 1          | 20.08 msecs   |
+----+----------------------------------------------------------------------+---------------+------------+---------------+
Predicates(identified by ID):
0: Split Range: STARTS_WITH($SongName, 'B')
3: Seek Condition: STARTS_WITH($SongName, 'B')
```


<!-- ##### B: Use STORING for covering index --> 

##### B: カバリングインデックスのために `STORING` を使う

<!--
If you have a limited number of columns that need to be SELECTed, you can create a "covering index" that includes all columns used by the query.
STORING can also be added by `ALTER INDEX ADD STORED COLUMN`, which is easier than adding more key columns to a secondary index.
-->

`SELECT` を含めクエリで使う必要があるカラムが限られているのであれば、クエリが使う全てのカラムを含めた「カバリングインデックス」を作ることを検討すると良いでしょう。
`STORING` は `ALTER INDEX ADD STORED COLUMN` によって追加することもできるので、セカンダリインデックスのキー列を増やすよりは容易にメンテナンスできます。

```
-- Create a new secondary index
CREATE INDEX SongsBySongNameStoring ON Songs(SongName) STORING(Duration);

-- or, you can simply add STORING columns into the existing index
ALTER INDEX SongBySongName ADD STORED COLUMN Duration
```

```
spanner> EXPLAIN ANALYZE
-> SELECT s.SongName, s.Duration
-> FROM Songs@{force_index=SongsBySongNameStoring} AS s
-> WHERE STARTS_WITH(s.SongName, "B");
+----+-----------------------------------------------------------------------------+---------------+------------+---------------+
| ID | Query_Execution_Plan                                                        | Rows_Returned | Executions | Total_Latency |
+----+-----------------------------------------------------------------------------+---------------+------------+---------------+
| *0 | Distributed Union                                                           | 78940         | 1          | 97.91 msecs   |
|  1 | +- Local Distributed Union                                                  | 78940         | 1          | 88.31 msecs   |
|  2 |    +- Serialize Result                                                      | 78940         | 1          | 83.48 msecs   |
| *3 |       +- Filter Scan (seekable_key_size: 1)                                 | 78940         | 1          | 68.54 msecs   |
|  4 |          +- Index Scan (Index: SongsBySongNameStoring, scan_method: Scalar) | 78940         | 1          | 57.19 msecs   |
+----+-----------------------------------------------------------------------------+---------------+------------+---------------+
Predicates(identified by ID):
0: Split Range: STARTS_WITH($SongName, 'B')
3: Seek Condition: STARTS_WITH($SongName, 'B')
```


<!-- #### Remove useless indexes -->
#### 不要なインデックスの削除

<!-- If you optimize each query separately, the number of secondary indexes with the same prefix may increase. --> 

クエリごとに個別に最適化をしていると、同じプリフィックスを持つセカンダリインデックスが増えていくことがあります。


```
CREATE INDEX SingersByFirstName ON Singers(FirstName);
CREATE INDEX SingersByFirstNameLastNameStoring ON Singers(FirstName, LastName) STORING(BirthDate);
```

<!--
Since both secondary indexes have `FirstName` as a prefix, a query that simply filters by `FirstName` will have exactly the same execution plan regardless of which secondary index is used.
-->

双方のセカンダリインデックスとも `FirstName` カラムをプリフィックスに持つため、単に `FirstName` でフィルタするクエリはどちらのセカンダリインデックスを使っても全く同じ実行計画となります。

<!--
In case of `SingersByFirstName`
-->

`SingersByFirstName` の場合、実行計画は下記のようになります。

```
spanner> EXPLAIN
-> SELECT *
-> FROM Singers@{FORCE_INDEX=SingersByFirstName}
-> WHERE FirstName = 'Catalina';
+-----+-------------------------------------------------------------------------------+
| ID  | Query_Execution_Plan                                                          |
+-----+-------------------------------------------------------------------------------+
|  *0 | Distributed Cross Apply                                                       |
|   1 | +- [Input] Create Batch                                                       |
|   2 | |  +- Compute Struct                                                          |
|  *3 | |     +- Distributed Union                                                    |
|   4 | |        +- Local Distributed Union                                           |
|  *5 | |           +- Filter Scan                                                    |
|   6 | |              +- Index Scan (Index: SingersByFirstName, scan_method: Scalar) |
|  18 | +- [Map] Serialize Result                                                     |
|  19 |    +- Cross Apply                                                             |
|  20 |       +- [Input] KeyRangeAccumulator                                          |
|  21 |       |  +- Batch Scan (Batch: $v2, scan_method: Scalar)                      |
|  23 |       +- [Map] Local Distributed Union                                        |
| *24 |          +- Filter Scan (seekable_key_size: 1)                                |
|  25 |             +- Table Scan (Table: Singers, scan_method: Scalar)               |
+-----+-------------------------------------------------------------------------------+
Predicates(identified by ID):
0: Split Range: ($SingerId' = $SingerId)
3: Split Range: ($FirstName = 'Catalina')
5: Seek Condition: IS_NOT_DISTINCT_FROM($FirstName, 'Catalina')
24: Seek Condition: ($SingerId' = $batched_SingerId)
```

<!--
In case of `SingersByFirstNameLastNameStoring`
-->

`SingersByFirstNameLastNameStoring` の場合、実行計画は下記のようになります。

```
spanner> EXPLAIN
-> SELECT *
-> FROM Singers@{FORCE_INDEX=SingersByFirstNameLastNameStoring}
-> WHERE FirstName = 'Catalina';
+-----+----------------------------------------------------------------------------------------------+
| ID  | Query_Execution_Plan                                                                         |
+-----+----------------------------------------------------------------------------------------------+
|  *0 | Distributed Union                                                                            |
|  *1 | +- Distributed Cross Apply                                                                   |
|   2 |    +- [Input] Create Batch                                                                   |
|   3 |    |  +- Local Distributed Union                                                             |
|   4 |    |     +- Compute Struct                                                                   |
|  *5 |    |        +- Filter Scan                                                                   |
|   6 |    |           +- Index Scan (Index: SingersByFirstNameLastNameStoring, scan_method: Scalar) |
|  19 |    +- [Map] Serialize Result                                                                 |
|  20 |       +- Cross Apply                                                                         |
|  21 |          +- [Input] KeyRangeAccumulator                                                      |
|  22 |          |  +- Batch Scan (Batch: $v2, scan_method: Scalar)                                  |
|  26 |          +- [Map] Local Distributed Union                                                    |
| *27 |             +- Filter Scan (seekable_key_size: 1)                                            |
|  28 |                +- Table Scan (Table: Singers, scan_method: Scalar)                           |
+-----+----------------------------------------------------------------------------------------------+
Predicates(identified by ID):
0: Split Range: ($FirstName = 'Catalina')
1: Split Range: ($SingerId' = $SingerId)
5: Seek Condition: IS_NOT_DISTINCT_FROM($FirstName, 'Catalina')
27: Seek Condition: ($SingerId' = $batched_SingerId)
```

<!--
In general, secondary indexes with more key suffixes and STORING tend to be able to handle other queries as well.
It is preferable to keep the schema simple by removing unnecessary secondary indexes, such as SingersByFirstName, so that more queries can be processed with fewer indexes.
-->

一般的に、あるクエリをより高速化するためにキーのサフィックスや `STORING` を増やしたセカンダリインデックスは他のより条件の緩いクエリでも活用可能な傾向があります。
今回の例における `SingersByFirstName` のような不要なセカンダリインデックスを削除して数少ないインデックスで多くのクエリを処理できるようにすることで、スキーマを単純に保つことが好ましいでしょう。

<!--
Especially in Cloud Spanner, increasing the number of secondary indexes that are not INTERLEAVE on a table increases the need for distributed commits with two-phased commits. Even if the secondary indexes are completely unnecessary.
Do not continue to degrade Write performance for the sake of something that does not even benefit Read.
-->

Cloud Spanner では特に、テーブルに INTERLEAVE されていないセカンダリインデックスを増やすことは two-phased commit による分散コミットの必要性を増やします。
Read のメリットすらないもののために Write のパフォーマンスを悪化させ続けることは避けましょう。

See also: https://cloud.google.com/spanner/docs/whitepapers/life-of-reads-and-writes

