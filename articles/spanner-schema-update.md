---
title: "Spanner スキーマ更新について"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Spanner"]
published: true
---

この記事は [JP_Google Developer Experts Advent Calendar 2025](https://adventar.org/calendars/11658) の10日目の記事です。

Spanner のスキーマ更新はダウンタイムを必要としないことが強みであり、その利点を享受しているユーザは多く居るでしょう。
しかし、他のデータベースとは異なる点が多く、どのようなスキーマ更新が遅くなるのかなどを予測できずにスキーマ更新というイベントに臨むことになることもあるでしょう。
その経験から真面目に使っているユーザはある程度は社内に運用ドキュメントのようなものを用意することになっているのではないかと思います。

運用ドキュメントは運用経験が蓄積するに従って充実していきますが、そのために本当に細かい仕様まで検証する企業は少ないでしょうし、そのようなドキュメントが外に出てくることもあまりないでしょう。

この記事では、[ドキュメント](https://docs.cloud.google.com/spanner/docs/schema-updates?hl=ja) で触れられてはいるが、あまり今までパブリックに検証している人が居ないように思われるスキーマ更新の細かい仕様のいくつかについての解説を試みます。

## ダウンタイムがないとは何を意味するか

DBMS によって異なりますし、改善されてきたり方法論が構築されてきてはいましたが、歴史的に RDBMS ではテーブルへのアクセスを妨げるグローバルロックや不整合との戦いとなるオンラインスキーマ更新は鬼門であり続けました。
Spanner は、そのような問題を解決するダウンタイムなしのスキーマ更新を売りとしています。[スキーマ更新のパフォーマンス](https://docs.cloud.google.com/spanner/docs/schema-updates?hl=ja#performance)から引用します。

> Spanner のスキーマの更新には、ダウンタイムは必要ありません。DDL 文のバッチを Spanner データベースに対して発行した場合、Spanner が更新を[長時間実行オペレーション](https://docs.cloud.google.com/spanner/docs/manage-long-running-operations?hl=ja)として適用する間も、中断なくデータベースでの書き込みと読み取りを続けることができます。

ここで、ダウンタイムがないというのは、一部の RDBMS のように DDL 実行中にクエリや DML の実行などトランザクションを妨げることでワークロードの実行を妨げることがないことを指します。
Spanner では基本的にはスキーマ更新を行っている途中も更新前のスキーマで読み書きでき、完了したコミットタイムスタンプの瞬間にスキーマが変わったかのような挙動をします。


## スキーマ更新オペレーションに関する情報の確認方法

スキーマ更新の進捗は [`Operation`](https://docs.cloud.google.com/spanner/docs/reference/rest/v1/projects.instances.databases.operations) リソースとして取得できます。

:::message
検証には GCPUG Public Spanner を使っているため、データベースパスなどはそのまま示します。
:::

公式に提供されている確認方法として、
[`gcloud spanner operations`](https://docs.cloud.google.com/spanner/docs/manage-and-observe-long-running-operations?hl=ja) のサブコマンドを使うことができます。
例えばこのようにして Operation の ID を特定して、詳細を確認できます。

```
$ gcloud spanner operations list --project gcpug-public-spanner --instance merpay-sponsored-instance --database apstndb-sampledb3                          
OPERATION_ID               STATEMENTS                                  DONE  @TYPE
_auto_op_1b29cda0b8e42cbb  ALTER TABLE TestTable ADD COLUMN Col INT64  True  UpdateDatabaseDdlMetadata

$ gcloud spanner operations describe --project gcpug-public-spanner --instance merpay-sponsored-instance --database apstndb-sampledb3 _auto_op_1b29cda0b8e42cbb
done: true
metadata:
  '@type': type.googleapis.com/google.spanner.admin.database.v1.UpdateDatabaseDdlMetadata
  actions:
  - action: ALTER
    entityNames:
    - TestTable
    entityType: TABLE
  commitTimestamps:
  - '2025-12-09T23:47:42.479890Z'
  database: projects/gcpug-public-spanner/instances/merpay-sponsored-instance/databases/apstndb-sampledb3
  progress:
  - endTime: '2025-12-09T23:47:42.479890Z'
    progressPercent: 100
    startTime: '2025-12-09T23:47:35.575004Z'
  statements:
  - ALTER TABLE TestTable ADD COLUMN Col INT64
name: projects/gcpug-public-spanner/instances/merpay-sponsored-instance/databases/apstndb-sampledb3/operations/_auto_op_1b29cda0b8e42cbb
response:
  '@type': type.googleapis.com/google.protobuf.Empty
```

この Operation リソースには各 DDL の進捗や完了したコミットタイムスタンプなどの詳細な情報が含まれていることがわかります。
この情報を元に Spanner Studio の Operations ページなどは表示されています。

![Spanner Studio 上での Operations ページのスクリーンショット](/images/spanner-studio-schema-update.png)

ちなみに私がメンテナンスしている [spanner-cli](https://github.com/apstndb/spanner-mycli) でも Operation を取得できるようにしていますし、 `progressPercent` を使ってプログレスバーの表示もしています。

```
spanner> SHOW SCHEMA UPDATE OPERATIONS;
+---------------------------+---------------------------------------------+------+----------+-----------------------------+-------+
| OPERATION_ID              | STATEMENTS                                  | DONE | PROGRESS | COMMIT_TIMESTAMP            | ERROR |
+---------------------------+---------------------------------------------+------+----------+-----------------------------+-------+
| _auto_op_1b29cda0b8e42cbb | ALTER TABLE TestTable ADD COLUMN Col INT64; | true |          | 2025-12-09T23:47:42.47989Z  |       |
+---------------------------+---------------------------------------------+------+----------+-----------------------------+-------+
1 rows in set (0.92 sec)

spanner> SHOW OPERATION _auto_op_1b29cda0b8e42cbb;
+---------------------------+---------------------------------------------+------+----------+----------------------------+-------+
| OPERATION_ID              | STATEMENTS                                  | DONE | PROGRESS | COMMIT_TIMESTAMP           | ERROR |
+---------------------------+---------------------------------------------+------+----------+----------------------------+-------+
| _auto_op_1b29cda0b8e42cbb | ALTER TABLE TestTable ADD COLUMN Col INT64; | true | 100      | 2025-12-09T23:47:42.47989Z |       |
+---------------------------+---------------------------------------------+------+----------+----------------------------+-------+
1 rows in set (1.08 sec)
```

## スキーマ更新

Spanner において[サポートされているスキーマ更新](https://docs.cloud.google.com/spanner/docs/schema-updates?hl=ja#supported-updates)の種類は多数ありますが、ほぼ即時終わるものと時間が掛かるものがあります。
その区別はドキュメントから確認可能です。

まずその区別は [Schema versions created during schema updates](https://docs.cloud.google.com/spanner/docs/schema-updates?hl=en#schema-versions) 以下に表として確認できます。 (2025年12月10日現在の表を添付)

![2025年12月10日現在のスキーマオペレーションの所要時間の表](/images/schema-operations-table.png)

「Estimated duration」の欄が「Minutes」になっているものは基本的にメタデータとしてのスキーマ情報を更新するだけで完了します。
「Minutes to hours」になっているものは単にメタデータを更新するだけではなく何らかの処理が必要なものです。処理が必要なものは下記の3種類に限られることがわかります。

* あらかじめベーステーブルが作られていた場合の `CREATE INDEX`
    * 実際のテーブルのデータを読んでそれに対するインデックステーブルを書き込み(バックフィル)する必要がある。
  * `CREATE TABLE` と `CREATE INDEX` を同時に行った場合はテーブルが空だとわかっているためインデックスへの書き込み不要
* `ALTER TABLE ... ALTER COLUMN` のうちバックグラウンドバリデーションが必要なもの
  * 実際のテーブルデータを読んで、今入っている値が変更後のスキーマでも有効かどうかを確認する必要がある。
* Optimizer Statistics を作成する `ANALYZE`
  * 実際のデータを読んで、オプティマイザが使う統計情報を作成、保存する必要がある。

バックグラウンドバリデーションが必要なものについてもドキュメントに列挙されています。(2025年12月10日現在の[日本語訳](https://docs.cloud.google.com/spanner/docs/schema-updates?hl=ja#updates-that-require-validation)より引用)
ここに含まれないものは全て、既に書き込まれたテーブルデータの実体を更新したりバリデーションせずとも、新しいスキーマ情報でデータを解釈しなおせば全て正しく扱えるということでしょう。

> * `NOT NULL` アノテーションを非キー列に追加する。例:
> ```
> ALTER TABLE Songwriters ALTER COLUMN Nickname STRING(MAX) NOT NULL;
> ```
> * 列の長さを短くする。例:
> ```
> * ALTER TABLE Songwriters ALTER COLUMN FirstName STRING(10);
> ```
> * `BYTES` を `STRING` に変更する。例:
> ```
> ALTER TABLE Songwriters ALTER COLUMN OpaqueData STRING(MAX);
> ```
> * `INT64/INT32` を `ENUM` に変更する。例:
> ```
> ALTER TABLE Albums ALTER COLUMN Label googlesql.example.music.RecordLabel;
> ```
> * `RecordLabel` 列挙型の定義から既存の値を削除する。
> * 既存の `TIMESTAMP` 列で commit タイムスタンプを有効にする。例:
> ```
> ALTER TABLE Albums ALTER COLUMN LastUpdateTime SET OPTIONS (allow_commit_timestamp = true);
> ```
> * 既存のテーブルへのチェック制約の追加。
> * 格納されている生成列を既存のテーブルに追加する。
> * 外部キーで新しいテーブルを作成する。
> * 既存のテーブルへの外部キーの追加。

ここに含まれるものはバリデーションが必要なため、メタデータの更新だけで終わるのではなくデータセットのサイズ、インスタンスのコンピュートキャパシティ(ノード数もしくはプロセッシングユニット数)、他のインスタンスへの負荷に依存した時間が掛かることとなります。
[CPU 使用率の指標](https://docs.cloud.google.com/spanner/docs/cpu-utilization) のドキュメントに書かれているように、これらの処理はシステムタスクの `MEDIUM` (バリデーション) もしくは `LOW` (インデックスと生成列のバックフィル) として実行されます。

バリデーションやインデックスのバックフィルが必要な DDL は時間が掛かる前提で、インスタンスサイズを大きくしてから実行するなども検討すると良いでしょう。

:::message
なお、バリデーションが必要なスキーマ更新では、バリデーション途中に不正なデータが追加されないように更新前後の両方のスキーマで正しいデータ以外は書き込めなくなります。

> スキーマ更新によっては、スキーマ更新が完了する前にデータベースに対するリクエストの動作が変わる場合があります。たとえば、`NOT NULL` を列に追加する場合、Spanner はほとんど瞬時に、列に `NULL` を使用する新しいリクエストの書き込みの拒否を開始します。
:::

## スキーマ更新のバッチ実行

スキーマ更新を行う UpdateDatabaseDdl API ([REST](https://docs.cloud.google.com/spanner/docs/reference/rest/v1/projects.instances.databases/updateDdl)/[gRPC](https://docs.cloud.google.com/spanner/docs/reference/rpc/google.spanner.admin.database.v1#google.spanner.admin.database.v1.DatabaseAdmin.UpdateDatabaseDdl)) は DDL のバッチ実行をサポートしています。
このバッチ実行により、 DDL を1文ずつ実行して待つ作業が必要なくなります。
DDL のバッチ実行を行う方法はツールによって異なりますが、今回は spanner-mycli の `START BATCH DDL` を使って対話的に行うこととします。

### DDL のバッチ実行は一般的にアトミックではない

```
spanner> CREATE TABLE TestTable (PK INT64 PRIMARY KEY);
Query OK (0.00 sec) (1 DDL in batch)

spanner> INSERT TestTable(PK, Col) VALUES(1, NULL);
Query OK, 1 rows affected (98.26 msecs)
commit_timestamp:     2025-12-10T19:11:21.795508+09:00
mutation_count:       2

spanner> START BATCH DDL;
Query OK (0.00 sec) (0 DDL in batch)

spanner> CREATE TABLE TestTable2 (PK INT64 PRIMARY KEY);
Query OK (0.00 sec) (1 DDL in batch)

spanner> ALTER TABLE TestTable ALTER COLUMN Col INT64 NOT NULL;
Query OK (0.00 sec) (2 DDLs in batch)

spanner> RUN BATCH;
ERROR: rpc error: code = FailedPrecondition desc = Adding a NOT NULL constraint on a column TestTable.Col is not allowed because it has a NULL value at key: [1]
```
```
$ gcloud spanner operations describe --project gcpug-public-spanner --instance merpay-sponsored-instance --database apstndb-sampledb3 _auto_op_2bd8298827e2c256
done: true
error:
  code: 9
  message: 'Adding a NOT NULL constraint on a column TestTable.Col is not allowed
    because it has a NULL value at key: [1]'
metadata:
  '@type': type.googleapis.com/google.spanner.admin.database.v1.UpdateDatabaseDdlMetadata
  actions:
  - action: CREATE
    entityNames:
    - TestTable2
    entityType: TABLE
  - action: ALTER
    entityNames:
    - TestTable
    entityType: TABLE
  commitTimestamps:
  - '2025-12-10T10:11:51.340965Z'
  database: projects/gcpug-public-spanner/instances/merpay-sponsored-instance/databases/apstndb-sampledb3
  progress:
  - endTime: '2025-12-10T10:11:51.340965Z'
    progressPercent: 100
    startTime: '2025-12-10T10:11:41.973749Z'
  - endTime: '2025-12-10T10:12:03.595379Z'
    startTime: '2025-12-10T10:11:51.340965Z'
  statements:
  - |-
    CREATE TABLE TestTable2 (
      PK INT64,
    ) PRIMARY KEY(PK)
  - ALTER TABLE TestTable ALTER COLUMN Col INT64 NOT NULL
name: projects/gcpug-public-spanner/instances/merpay-sponsored-instance/databases/apstndb-sampledb3/operations/_auto_op_2bd8298827e2c256
```

* `statements` がそれぞれ `CREATE TABLE TestTable2` と `ALTER TABLE TestTable` になっている。
* `progress` に含まれる `endTime` がそれぞれ異なり、 `CREATE TABLE` だけ 100% になっている。

ことがわかるでしょうか。
既存のテーブルへの `ALTER TABLE ... ALTER COLUMN ... NOT NULL` は `NULL` が入っている列があると失敗する可能性があるバリデーションが必要な操作であるため、 `CREATE TABLE` と同じスキーマバージョンで実行することができないため、分けて実行されています。

実際のところ、 `TestTable` に `Col` が `NULL` の行を追加したため `TestTable2` の作成は成功して、 `TestTable` の `Col` への `NOT NULL` 制約の追加は失敗しています。
spanner-cli 系に存在する `SHOW CREATE TABLE` コマンドで DDL を見ることで、現在のスキーマで `TestTable` の `Col` に `NOT NULL` がついていないことと `TestTable2` が作成されていることが確認できます。

```
spanner> SHOW CREATE TABLE TestTable;
+-----------+--------------------------+
| Name      | DDL                      |
+-----------+--------------------------+
| TestTable | CREATE TABLE TestTable ( |
|           |   PK INT64,              |
|           |   Col INT64,             |
|           | ) PRIMARY KEY(PK)        |
+-----------+--------------------------+
1 rows in set (1.32 sec)

spanner> SHOW CREATE TABLE TestTable2;
+------------+---------------------------+
| Name       | DDL                       |
+------------+---------------------------+
| TestTable2 | CREATE TABLE TestTable2 ( |
|            |   PK INT64,               |
|            | ) PRIMARY KEY(PK)         |
+------------+---------------------------+
1 rows in set (0.77 sec)
```

DDL のバッチ実行がアトミックではなく、部分的失敗の可能性があることが確認できました。より多くの DDL がバッチが含まれている時も、エラーがあったステートメントの時点で実行は停止することが[バッチ内の文の実行順序](https://docs.cloud.google.com/spanner/docs/schema-updates?hl=ja#order_of_execution_of_statements_in_batches)として明記されています。

> Spanner では、同じバッチの文を順番に適用し、最初のエラーで停止します。ある文の適用でエラーが発生した場合、その文はロールバックされます。バッチ内の以前に適用された文の結果はロールバックされません。

DDL の実行が失敗した場合には今どのような状態になっているのかを把握した上で、データもしくは DDL を修正するなどして正常な状態に復帰する必要があるでしょう。

:::message
アトミックではないだけでなく、並行する DDL のバッチ実行は任意の順序で実行される場合があることがドキュメントで示唆されています。
:::

上記の `ALTER TABLE ... ALTER COLUMN` はバリデーションが必要な操作であるため失敗の可能性があるためスキーマバージョンが分かれて単独で失敗しましたが、
バリデーションもバックフィルの処理も不要な一連の DDL についてはテーブルデータに触らずメタデータの更新のみで済むため、一つのコミットタイムスタンプを持つ単一のスキーマバージョンで処理することができます。

例えば次のような一連の DDL をバッチ実行は単一のスキーマバージョンで処理できます。テーブルと同時にインデックスを作成する場合にはバックフィルが必要ないことを思い出してください。

```
DROP TABLE TestTable;
DROP TABLE TestTable2;
CREATE TABLE TestTable (PK INT64 PRIMARY KEY);
ALTER TABLE TestTable ADD COLUMN Col INT64;
CREATE INDEX TestTableByCol ON TestTable (Col);
CREATE TABLE TestTable2 (PK INT64 PRIMARY KEY);
```

```
$ gcloud spanner operations describe --project gcpug-public-spanner --instance merpay-sponsored-instance --database apstndb-sampledb3 _auto_op_5f7931f7d163b9c0
done: true
metadata:
  '@type': type.googleapis.com/google.spanner.admin.database.v1.UpdateDatabaseDdlMetadata
  actions:
  - action: DROP
    entityNames:
    - TestTable
    entityType: TABLE
  - action: DROP
    entityNames:
    - TestTable2
    entityType: TABLE
  - action: CREATE
    entityNames:
    - TestTable
    entityType: TABLE
  - action: ALTER
    entityNames:
    - TestTable
    entityType: TABLE
  - action: CREATE
    entityNames:
    - TestTableByCol
    entityType: INDEX
  - action: CREATE
    entityNames:
    - TestTable2
    entityType: TABLE
  commitTimestamps:
  - '2025-12-10T10:35:14.222873Z'
  - '2025-12-10T10:35:14.222873Z'
  - '2025-12-10T10:35:14.222873Z'
  - '2025-12-10T10:35:14.222873Z'
  - '2025-12-10T10:35:14.222873Z'
  - '2025-12-10T10:35:14.222873Z'
  database: projects/gcpug-public-spanner/instances/merpay-sponsored-instance/databases/apstndb-sampledb3
  progress:
  - endTime: '2025-12-10T10:35:14.222873Z'
    progressPercent: 100
    startTime: '2025-12-10T10:35:05.712509Z'
  - endTime: '2025-12-10T10:35:14.222873Z'
    progressPercent: 100
    startTime: '2025-12-10T10:35:05.712509Z'
  - endTime: '2025-12-10T10:35:14.222873Z'
    progressPercent: 100
    startTime: '2025-12-10T10:35:05.712509Z'
  - endTime: '2025-12-10T10:35:14.222873Z'
    progressPercent: 100
    startTime: '2025-12-10T10:35:05.712509Z'
  - endTime: '2025-12-10T10:35:14.222873Z'
    progressPercent: 100
    startTime: '2025-12-10T10:35:05.712509Z'
  - endTime: '2025-12-10T10:35:14.222873Z'
    progressPercent: 100
    startTime: '2025-12-10T10:35:05.712509Z'
  statements:
  - DROP TABLE TestTable
  - DROP TABLE TestTable2
  - |-
    CREATE TABLE TestTable (
      PK INT64,
    ) PRIMARY KEY(PK)
  - ALTER TABLE TestTable ADD COLUMN Col INT64
  - CREATE INDEX TestTableByCol ON TestTable(Col)
  - |-
    CREATE TABLE TestTable2 (
      PK INT64,
    ) PRIMARY KEY(PK)
name: projects/gcpug-public-spanner/instances/merpay-sponsored-instance/databases/apstndb-sampledb3/operations/_auto_op_5f7931f7d163b9c0
response:
  '@type': type.googleapis.com/google.protobuf.Empty
$
```

全ての DDL が同一の `endTime` に完了しているため、同じスキーマバージョンで処理されていることが確認できます。

DDL を単一のスキーマバージョンで処理することの重要性は、ドキュメントにも書かれています。
具体的には、 [スキーマ更新に関するベストプラクティスの章](https://docs.cloud.google.com/spanner/docs/schema-updates?hl=ja#best-practices)では、短期間に大量のスキーマバージョンを作成すると[スキーマ更新のスロットル](https://docs.cloud.google.com/spanner/docs/schema-updates?hl=ja#frequency)ことがあることが明記されています。
それを避けるため、[大規模なスキーマ更新](https://docs.cloud.google.com/spanner/docs/schema-updates?hl=ja#large-updates) の節では、バリデーションやバックフィルが必要ない一連の DDL 文は数千個ステートメントであっても1つのスキーマバージョンで実行できるため、
可能な限り多くのステートメントを単一のスキーマバージョンでスキーマ更新が可能なように DDL バッチ内の順序を検討することが有効であると明記されています。


## 過去のスキーマバージョンは stale クエリで使われる

スキーマバージョンというだけあって、過去のバージョンは保存されています。
過去のスキーマバージョンは過去のデータを Stale Read する時に使われます。

下の例では、 `TimeMachine` テーブルの `Col` カラムの型を `STRING` から `BYTES` に変更した後でも、スキーマを更新する前のタイムスタンプで読んだ場合は `STRING` 型として過去のテーブルデータを参照できることを示しています。

```
spanner> CREATE TABLE TimeMachine (PK INT64 PRIMARY KEY, Col STRING(MAX));
Query OK (13.98 sec)
commit_timestamp:     2025-12-10T13:12:45.416103Z

spanner> INSERT TimeMachine(PK, Col) VALUES (1, "テスト");
Query OK, 1 rows affected (92.58 msecs)
commit_timestamp:     2025-12-10T22:12:57.519126+09:00
mutation_count:       2

spanner> ALTER TABLE TimeMachine ALTER COLUMN Col BYTES(MAX);
Query OK (20.93 sec)
commit_timestamp:     2025-12-10T13:13:17.099394Z

spanner> SELECT * FROM TimeMachine;
+-------+--------------+
| PK    | Col          |
| INT64 | BYTES        |
+-------+--------------+
| 1     | 44OG44K544OI |
+-------+--------------+
1 rows in set (85.75 msecs)
read_timestamp:       2025-12-10T22:13:28.717209+09:00
cpu time:             84.65 msecs
rows scanned:         1 rows
deleted rows scanned: 0 rows
optimizer version:    8
optimizer statistics: auto_20251210_06_46_21UTC

spanner> BEGIN RO "2025-12-10T22:12:57.519126+09:00";
Query OK (0.36 sec)
read_timestamp:       2025-12-10T22:12:57.519126+09:00

spanner(ro txn)> SELECT * FROM TimeMachine;
+-------+--------+
| PK    | Col    |
| INT64 | STRING |
+-------+--------+
| 1     | テスト |
+-------+--------+
1 rows in set (2.26 msecs)
read_timestamp:       2025-12-10T22:12:57.519126+09:00
cpu time:             2.21 msecs
rows scanned:         1 rows
deleted rows scanned: 0 rows
optimizer version:    8
optimizer statistics: auto_20251210_06_46_21UTC
```


保存される期間は古いスキーマのデータが読み込まれる可能性がある限りなので、 PITR のために `version_retention_period` オプションでデータの保持期間を延長した場合、スキーマバージョンも同じだけの時間保持されることがわかります。
https://docs.cloud.google.com/spanner/docs/schema-updates?hl=ja#schema-versions

> 古いスキーマ バージョンは、サーバーとストレージのリソースを大幅に消費する可能性があり、期限切れになるまで（古いバーションのデータの読み込みを提供する必要がなくなるまで）保持されます。

## まとめ

この記事では Spanner のスキーマ更新について実際の Operation の値や挙動を見ながら説明しました。

DDL のバッチ実行は常に即時に終わったりアトミックというわけではありませんが、ドキュメントにはどのような DDL が単一のスキーマバージョンにまとめることで数分程度で実行できるのかということが明記されていることを示し、挙動を確かめました。
この内容を理解することで、大規模なスキーマ更新のうちの大きな部分がデータ量に依存しないメタデータの変更だけで効率的に処理できる実感を深めていただけたのではないかと思います。

この記事で説明を試みたことが、スキーマ更新という大きなイベントに不安なく臨めるユーザが増えることに繋がることを願います。
