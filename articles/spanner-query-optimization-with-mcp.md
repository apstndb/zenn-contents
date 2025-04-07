---
title: "AI エージェント + MCP は Spanner クエリ最適化の夢を見るか？"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cloudspanner", "mcp"]
published: true
---

最近 [Model Context Protocol](https://modelcontextprotocol.io/) (以下 MCP) が流行っているので、個人的に親しんでいる Spanner のクエリ最適化を MCP を使ってできないかを試みてみた話をします。
MCP そのものについての説明はいくらでも見つかるので省略します。

## AI によるクエリ最適化の課題

生成 AI に SQL クエリを書かせる時に問題になることとして、生成 AI はオプティマイザではないのでクエリがどう実行されるのかを知らないということがあげられます。
結果として、生成 AI が改善したと主張するクエリを人が確かめてがっかりすることを繰り返すようなことが起きてしまいます。

これは AI エージェントが実行計画を自分で取得して解釈し、クエリとスキーマの改善とフィードバックループを回していけるようになれば解決するのではないかと考えられます。
どのように実行計画を取得するか？そう、一つの手段は MCP ですね。

## Spanner MCP Server

というわけで、個人的なおもちゃとして Spanner MCP Server として [apstndb/spanner-mcp](https://github.com/apstndb/spanner-mcp) を書いてみています。Go なので実装には [mark3labs/mcp-go](https://github.com/mark3labs/mcp-go) を使いました。

今の所実装した機能は下記の3つです。 Go で Spanner を叩いたことがある人なら誰でも書ける機能ばかりですね。

- 実行計画の取得(`ExecuteSql` with `QueryMode=PLAN` 相当)
  - 人に読める出力を出すためにテーブル内のツリーとしてレンダリングした結果も返す。
- データベーススキーマの DDL 取得(`GetDatabaseDdl` 相当)
- データベーススキーマの更新(`UpdateDatabaseDdl` 相当)

この3つの機能があるだけでも下記のようなことを自律的にフィールバックループを回せるようになることがわかりました。

- スキーマを確認し自然言語で指示された通りのクエリを記述
- クエリの実行計画を取得、解説しながら問題点を発見
- 必要に応じてスキーマ更新
- 改善したクエリの実行計画を取得、改善箇所の解説

## 実際のエージェントとのやりとり

それでは、この Spanner MCP サーバを Cline と組み合わせて使った例をお見せします。

(「下記の Spanner インスタンスで `SELECT * FROM Singers` の実行計画を MCP を使って取得してください。」という先行するやりとりを省略)

**User:**

先ほどのクエリを `FirstName` をクエリパラメータ `@firstName` でフィルタするように変更し、実行計画を解説してください。

**Assistant:**

> 了解しました。`FirstName` でフィルタするクエリの実行計画を取得し、人間が読みやすい形式で表示して解説します。
> 
> まず、変更後のクエリで実行計画を取得します。
> 
> ```xml
> <use_mcp_tool>
> <server_name>spanner</server_name>
> <tool_name>plan</tool_name>
> <arguments>
> {
>   "query": "SELECT * FROM Singers WHERE FirstName = @firstName",
>   "project": "gcpug-public-spanner",
>   "instance": "merpay-sponsored-instance",
>   "database": "apstndb-sampledb3"
> }
> </arguments>
> </use_mcp_tool>
> ```

**Assistant:**

> `SELECT * FROM Singers WHERE FirstName = @firstName` クエリの実行計画は以下の通りです。
> 
> ```
> +-----+--------------------------------------------------------------------------------------------------------------------+
> | ID  | Operator                                                                                                           |
> +-----+--------------------------------------------------------------------------------------------------------------------+
> |  *0 | Distributed Union (distribution_table: SingersByFirstLastName, execution_method: Row, split_ranges_aligned: false) |
> |  *1 | +- Distributed Cross Apply (execution_method: Row)                                                                 |
> |   2 |    +- Create Batch (execution_method: Row)                                                                         |
> |   3 |    |  +- Local Distributed Union (execution_method: Row)                                                           |
> |   4 |    |     +- Compute Struct (execution_method: Row)                                                                 |
> |  *5 |    |        +- Filter Scan (execution_method: Row, seekable_key_size: 0)                                           |
> |  *6 |    |           +- Index Scan (Index: SingersByFirstLastName, execution_method: Row, scan_method: Automatic)        |
> |  21 |    +- [Map] Serialize Result (execution_method: Row)                                                               |
> |  22 |       +- Cross Apply (execution_method: Row)                                                                       |
> |  23 |          +- KeyRangeAccumulator (execution_method: Row)                                                            |
> |  24 |          |  +- Batch Scan (Batch: $v2, execution_method: Row, scan_method: Row)                                    |
> |  27 |          +- [Map] Local Distributed Union (execution_method: Row)                                                  |
> |  28 |             +- Filter Scan (execution_method: Row, seekable_key_size: 0)                                           |
> | *29 |                +- Table Scan (Table: Singers, execution_method: Row, scan_method: Row)                             |
> +-----+--------------------------------------------------------------------------------------------------------------------+
> Predicates(identified by ID):
>   0: Split Range: ($FirstName = @firstname)
>   1: Split Range: ($SingerId' = $SingerId)
>   5: Residual Condition: ($FirstName = @firstname)
>   6: Seek Condition: IS_NOT_DISTINCT_FROM($FirstName, @firstname)
>  29: Seek Condition: ($SingerId' = $batched_SingerId)
> ```
> 
> **実行計画の解説:**
> 
> 1.  **`*6: Index Scan (Index: SingersByFirstLastName)`**: クエリはまず `SingersByFirstLastName` インデックスを利用します。`Seek Condition: IS_NOT_DISTINCT_FROM($FirstName, @firstname)` により、パラメータ `@firstName` と一致する `FirstName` を持つインデックスエントリを効率的に検索（Seek）します。
> 2.  **`*5: Filter Scan`**: インデックススキャンで取得したデータに対して、さらに `Residual Condition: ($FirstName = @firstname)` でフィルタリングを行います。これは、インデックスだけでは完全に絞り込めなかった場合の追加条件です。
> 3.  **`4: Compute Struct`**: インデックススキャンから取得した必要なカラム（この場合は `LastName` と `SingerId`）を含む構造体を作成します。
> 4.  **`3: Local Distributed Union`**: ローカルノード内で結果をまとめます。
> 5.  **`2: Create Batch`**: 後続のステップで `Singers` テーブルから他のカラムを取得するために、見つかった `SingerId` のバッチを作成します。
> 6.  **`*1: Distributed Cross Apply`**: `SingersByFirstLastName` インデックスから取得した `SingerId` のバッチを、`Singers` テーブルをスキャンする後続の処理（`[Map]` 部分）に適用します。`Split Range: ($SingerId' = $SingerId)` は、バッチ内の `SingerId` と `Singers` テーブルの `SingerId` を照合するための条件です。
> 7.  **`*29: Table Scan (Table: Singers)`**: `Distributed Cross Apply` から渡された `SingerId` のバッチを使用して、`Singers` テーブルをスキャン（Seek）し、対応する行の残りのカラム（`BirthDate`, `SingerInfo` など）を取得します。`Seek Condition: ($SingerId' = $batched_SingerId)` がこのバッチ検索に使われます。
> 8.  **`28, 27, 22, 21`**: テーブルスキャンの結果を処理し、最終結果としてシリアライズします。
> 9.  **`*0: Distributed Union`**: 最終的に、異なるサーバー（スプリット）からの結果を統合します。`Split Range: ($FirstName = @firstname)` は、分散処理の範囲を絞り込むための条件です。
> 
> **要約:**
> 
> このクエリでは、`FirstName` でフィルタリングするために `SingersByFirstLastName` インデックスが効果的に利用されています (`Index Scan`)。インデックスには含まれない他のカラム（`BirthDate`, `SingerInfo` など）を取得するために、一度インデックスで `SingerId` を絞り込んだ後、その `SingerId` を使って元の `Singers` テーブルを効率的に検索 (`Table Scan` with Seek Condition) する `Cross Apply` 戦略が取られています。これにより、テーブル全体をスキャンする (`Full Scan`) 必要がなくなり、パフォーマンスが向上します。

**User:**

このクエリに必要なカバリングインデックスを作成し、それを明示的に使用するようにヒントを設定した上で、実行計画を取得しなおしてください。

**Assistant:**

> 了解しました。まず、クエリに必要なカラムをすべて含むカバリングインデックス `SingersByFirstNameCovering` を作成します。
> 
> ```xml
> <use_mcp_tool>
> <server_name>spanner</server_name>
> <tool_name>update_ddl</tool_name>
> <arguments>
> {
>   "project": "gcpug-public-spanner",
>   "instance": "merpay-sponsored-instance",
>   "database": "apstndb-sampledb3",
>   "statements": [
>     "CREATE INDEX SingersByFirstNameCovering ON Singers(FirstName) STORING (LastName, SingerInfo, BirthDate)"
>   ]
> }
> </arguments>
> </use_mcp_tool>
> ```

**Assistant:**

> カバリングインデックス `SingersByFirstNameCovering` を作成しました。
> 
> 次に、この新しいインデックスを使用するようにクエリヒント `@`{FORCE_INDEX=SingersByFirstNameCovering}` を追加して、実行計画を再度取得します。
> 
> ```xml
> <use_mcp_tool>
> <server_name>spanner</server_name>
> <tool_name>plan</tool_name>
> <arguments>
> {
>   "query": "SELECT * FROM Singers@{FORCE_INDEX=SingersByFirstNameCovering} WHERE FirstName = @firstName",
>   "project": "gcpug-public-spanner",
>   "instance": "merpay-sponsored-instance",
>   "database": "apstndb-sampledb3"
> }
> </arguments>
> </use_mcp_tool>
> ```

---

**Assistant:**

> カバリングインデックス `SingersByFirstNameCovering` を強制的に使用するようにクエリヒントを設定した `SELECT * FROM Singers@{FORCE_INDEX=SingersByFirstNameCovering} WHERE FirstName = @firstName` クエリの実行計画は以下の通りです。
> 
> ```
> +----+------------------------------------------------------------------------------------------------------------------------+
> | ID | Operator                                                                                                               |
> +----+------------------------------------------------------------------------------------------------------------------------+
> | *0 | Distributed Union (distribution_table: SingersByFirstNameCovering, execution_method: Row, split_ranges_aligned: false) |
> |  1 | +- Local Distributed Union (execution_method: Row)                                                                     |
> |  2 |    +- Serialize Result (execution_method: Row)                                                                         |
> | *3 |       +- Filter Scan (execution_method: Row, seekable_key_size: 0)                                                     |
> | *4 |          +- Index Scan (Index: SingersByFirstNameCovering, execution_method: Row, scan_method: Automatic)              |
> +----+------------------------------------------------------------------------------------------------------------------------+
> Predicates(identified by ID):
>  0: Split Range: ($FirstName = @firstname)
>  3: Residual Condition: ($FirstName = @firstname)
>  4: Seek Condition: IS_NOT_DISTINCT_FROM($FirstName, @firstname)
> ```
> 
> **実行計画の解説:**
> 
> 1.  **`*4: Index Scan (Index: SingersByFirstNameCovering)`**: クエリは、ヒントで指定されたカバリングインデックス `SingersByFirstNameCovering` を直接使用します。`Seek Condition: IS_NOT_DISTINCT_FROM($FirstName, @firstname)` を使って、`@firstName` に一致する `FirstName` を持つインデックスエントリを検索します。このインデックスはクエリが必要とするすべてのカラム (`SingerId`, `FirstName`, `LastName`, `SingerInfo`, `BirthDate`) を含んでいる（カバリングしている）ため、Spanner は必要なデータをすべてインデックスから直接取得できます。
> 2.  **`*3: Filter Scan`**: インデックススキャン後、念のため `Residual Condition: ($FirstName = @firstname)` によるフィルタリングが行われます。
> 3.  **`2: Serialize Result`**: フィルタリングされた結果が最終出力用にシリアライズされます。
> 4.  **`1: Local Distributed Union`**: ローカルノード内で結果が統合されます。
> 5.  **`*0: Distributed Union`**: 最後に、異なるサーバー（スプリット）からの結果が統合されます。`Split Range: ($FirstName = @firstname)` は、分散処理の範囲を決定するために使用されます。
> 
> **以前の計画との比較:**
> 
> この新しい計画は、以前の計画（カバリングインデックス未使用時）と比較して大幅にシンプルになっています。特に、`Distributed Cross Apply`、`Create Batch`、`Table Scan` といったオペレータがなくなりました。
> 
> 以前の計画では、`SingersByFirstLastName` インデックスには `FirstName` と `LastName` しか含まれていなかったため、他のカラム (`SingerInfo`, `BirthDate`) を取得するために、一度インデックスから `SingerId` を取得し、その `SingerId` を使って元の `Singers` テーブルを追加で検索する必要がありました（`Cross Apply` と `Table Scan`）。
> 
> 今回作成した `SingersByFirstNameCovering` は、クエリに必要なすべてのカラムをカバーしているため、インデックスのスキャン (`Index Scan`) だけでクエリを完了できます。元のテーブルへのアクセスが不要になったため、`Cross Apply` や `Table Scan` がなくなり、より効率的な実行計画となりました。

いかがでしょうか。この例ではかなり具体的なことまでユーザが指示してしまいましたが、実行計画が良くなったかどうかを人に説明しながらクエリとスキーマの改善ができているように見えますね。

### うまく行かない例

もちろんまだまだうまく行かないところはあります。たとえば次の抜粋を見てみてください。

**Assistant:**

> 承知いたしました。降順インデックス `SongsBySongNameDesc` が正常に作成されました。
> 
> それでは、この新しいインデックスが効果的に利用されるか確認するために、**必要なカラムのみを選択し、`FORCE_INDEX` ヒントを削除した**改善後のクエリで実行計画を取得します。
> 
> **改善後のクエリ:**
> ```sql
> SELECT SingerId, AlbumId, TrackId, SongName
> FROM Songs
> ORDER BY SongName DESC
> LIMIT 1;
> ```
> 
> MCP サーバーを使用して実行計画を取得します。
> 
> ```xml
> <use_mcp_tool>
> <server_name>spanner</server_name>
> <tool_name>plan</tool_name>
> <arguments>
> {
>   "query": "SELECT SingerId, AlbumId, TrackId, SongName FROM Songs ORDER BY SongName DESC LIMIT 1;",
>   "project": "gcpug-public-spanner",
>   "instance": "merpay-sponsored-instance",
>   "database": "apstndb-sampledb3"
> }
> </arguments>
> </use_mcp_tool>
> ```

**Assistant:**

> はい、承知いたしました。新しい降順インデックス `SongsBySongNameDesc` を使用して、改善後のクエリの実行計画を取得しました。
> 
> **改善後の実行計画:**
> ```
> +----+--------------------------------------------------------------------------------------------------------------------+
> | ID | Operator                                                                                                           |
> +----+--------------------------------------------------------------------------------------------------------------------+
> |  0 | Global Limit (execution_method: Row)                                                                               |
> |  1 | +- Distributed Union (distribution_table: SongsBySongNameDesc, execution_method: Row, split_ranges_aligned: false) |
> |  2 |    +- Serialize Result (execution_method: Row)                                                                     |
> |  3 |       +- Local Limit (execution_method: Row)                                                                       |
> |  4 |          +- Local Distributed Union (execution_method: Row)                                                        |
> |  5 |             +- Index Scan (Full scan: true, Index: SongsBySongNameDesc, execution_method: Row, scan_method: Row)   |
> +----+--------------------------------------------------------------------------------------------------------------------+
> ```
> 
> **実行計画の分析:**
> 
> 残念ながら、依然として `Index Scan (Full scan: true)` が発生しています。これは、`ORDER BY SongName DESC LIMIT 1` の最適化が期待通りに機能していないことを意味します。通常、降順のインデックスがあれば、データベースはインデックスの最初の行を直接読み取るだけで済みます。
> 
> この問題の原因としては、いくつかの可能性が考えられます。
> 
> 1.  **インデックス統計情報の不足:** Spanner がまだ新しいインデックスに関する十分な統計情報を収集できていない可能性があります。統計情報が最新でない場合、オプティマイザは非効率な実行計画を選択することがあります。
> 2.  **クエリの複雑さ:** `LIMIT 1` だけでなく、他の条件がクエリに含まれている場合（この実行計画では見えませんが、前のステップで確認したように）、それらの条件が原因でフルスキャンが必要になることがあります。
> 3.  **Spanner オプティマイザの挙動:** 特定の状況下では、Spanner のクエリオプティマイザが、インデックスの最初の行を直接読み取る代わりにフルスキャンを選択することがあります。これは、データの分布や他の要因による可能性があります。
> 
> **次のステップ:**
> 
> 現時点では、`LIMIT 1` と `ORDER BY DESC` を使用しているにもかかわらず、なぜフルスキャンが発生するのかを正確に特定するのは難しいです。
> 
> 考えられる対策としては、以下があります。
> 
> 1.  **統計情報の更新:** Spanner にインデックスの統計情報を強制的に更新させるコマンドを実行します (もし利用可能であれば)。
> 2.  **`FORCE_INDEX` の再検討:** やはり `FORCE_INDEX` を使用して、Spanner に `SongsBySongNameDesc` インデックスの使用を強制します。ただし、これは根本的な解決策ではなく、オプティマイザが最適でない計画を選択している理由を理解することが重要です。
> 3.  **より詳細な実行計画の分析:** 可能であれば、より詳細な実行計画 (`ANALYZE` など) を取得して、どの部分がフルスキャンを引き起こしているのかを特定します。
> 
> 今回の目的は、`LIMIT 1` でのパフォーマンスを改善することでした。現在の実行計画ではまだ改善が見られないため、`FORCE_INDEX` ヒントを使用して、期待通りの動作をするか試してみましょう。

この分析内容と、それを元に `FORCE_INDEX` を指定するというアクションは明らかに間違っています。
Spanner において `LIMIT` はスキャンオペレーションそのものに影響を与えることがないため、フィルタがないのであればフルスキャンとなるが Local Limit によりスキャンは期待通り打ち切られることは Spanner の実行計画を読める人にとっては暗黙知として理解されているはずです。
今回は Gemini 2.5 Pro Exp を使っており、おそらく Gemini は学習対象に Google Cloud のドキュメントを使っているため表面的な違和感はかなり少なく実行計画を議論できていますが、この `LIMIT` 周りの概念についてはドキュメントで説明されたことがないためこのような誤解が発生するのだと推測されます。

Google が AI が学習対象として使って自走できる程度に十分なドキュメントを用意しない限りは、 AI が誤解しやすい事柄をまとめたテキストを最初に読ませるなどの対応が必要なのではないかと考えられます。

## 所感

単純な Spanner MCP サーバを実装することで、実行計画を自ら確認し説明しながらスキーマやクエリの最適化をするという、人間であっても教育することが難しいようなことをエージェントが自律的に行うことが可能であるという手応えを得ることとができました。
Spanner の実行計画を読める人にとってはツッコミどころがあるところはたくさん残っていますが、ギャップとなるような専門家の暗黙知を言語化してあらかじめ教えることができれば、今あるものの組み合わせでもかなり良くなる余地があります。

データベースに適性のある人は採用することも育てることも難しいという問題があります。ノウハウのないデータベースを生成 AI と二人三脚で使っていくことができる未来に期待しています。