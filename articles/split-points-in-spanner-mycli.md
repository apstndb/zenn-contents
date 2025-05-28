---
title: "Spanner 事前スプリット(AddSplitPoints) と対話型ツール spanner-mycli での実装例"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [spanner]
published: true
---

この記事では、 Spanner の事前スプリット (pre-splitting) 機能の概要と、私が開発している対話型ツール [spanner-mycli](https://github.com/apstndb/spanner-mycli) での対応について解説します。

## Spanner の負荷分散と暖機運転というワークアラウンド

分散データベースでは、負荷を複数のコンピュートリソースに分散させることが重要です。Spanner は、データのキー空間を「スプリット」と呼ばれる単位に分割し、各スプリットをコンピュートリソースに割り当てることで、水平分散を実現しています。

公式ドキュメントの「[Database splits](https://cloud.google.com/spanner/docs/schema-and-data-model#database-splits)」セクションで説明されているように、スプリットの分割は、負荷（load-based splitting）またはデータサイズ（size-based splitting）の増加に応じて自動的に行われます。そのため、ユーザー数やデータ量が徐々に増加するようなケースでは、ホットスポットが発生しない限り、Spanner はサービス規模に対して自動的にスケールします。

しかし、これらの自動的なスプリット分割によるスケールでは間に合わないようなアクセスパターンは存在します。
その一例が、ゲーミング業界です。新しい機能やコンテンツのリリース直後、あるいはメンテナンス終了直後にはアクセスが急増（スパイク）しやすく、このスパイクを適切に処理することは、機会損失の回避やレピュテーションリスクの低減といったビジネス上の観点から非常に重要です。

そのような理由があるため、 Google も [Best practices for using Spanner as a gaming database](https://cloud.google.com/spanner/docs/best-practices-gaming-database) という業界固有のベストプラクティスを用意していました。

:::message
ホットスポットをなくして分散する設計にすることは Spanner においては常に重要ですが、自動的なスプリット分割で間に合うのであればキーの分散を最優先にした設計をする必要はないですし過度な最適化としてやらない方が良いこともあることは注意してください。
:::

また、次のようなケースでも負荷ベースのスプリット分割を待つことはあまり好ましくないということがありました。

* 徐々にしか分割されないので、負荷試験で想定する負荷を掛けられる状態を用意するために時間が掛かる。
* バッチによるデータ投入で速度が出せるようになるまで時間が掛かる。

従来、このようなケースでは[暖機運転(Pre-warming)](https://web.archive.org/web/20241107172845/https://cloud.google.com/spanner/docs/pre-warm-database) と呼ばれるワークアラウンドが紹介されていました。これは、実際の高負荷がかかる前に大量のデータを投入し、意図的に負荷をかけることで、負荷ベースのスプリット分割とその再配置を促す手法です。

## Pre-splitting

[2025年4月28日のリリースノート](https://cloud.google.com/spanner/docs/release-notes#April_28_2025)で発表されたように、Spanner は事前スプリット (pre-splitting) という機能をリリースしました。これは、従来の自動的なスプリット管理に加え、ユーザーが split points（分割箇所）を手動で指示できる機能です。
「Pre-splitting」という名称は、予期されるアクセスパターンの変化やトラフィックスパイクが発生するよりも前に、手動でスプリットを強制できることに由来すると考えられます。

興味深いことに、前述の[暖機運転に関するドキュメントの URL](https://cloud.google.com/spanner/docs/pre-warm-database) は、現在この[事前スプリット機能のページ](https://cloud.google.com/spanner/docs/pre-splitting-overview)にリダイレクトされるようになっています。これは、暖機運転というワークアラウンドが、この新機能によって置き換えられたことに対する Google の自信の表れと言えるかもしれません。

まさにソーシャルゲームで Spanner を使っているコロプラさんの [ついにきた！Google Cloud Spannerの「事前分割（Pre-splitting）」で大規模トラフィックに備えよう](https://blog.colopl.dev/entry/2025/05/26/110000) の記事で当事者としての見解や検証内容が書かれています。
ユースケースを持っていない私が説明するよりも実際の当事者が書いたものを読むのが良いと思うので参照してみてください。

## spanner-mycli における pre-splitting 対応

私自身は直接的なユースケースを持っていませんが、個人プロジェクトとして [cloudspannerecosystem/spanner-cli](https://github.com/cloudspannerecosystem/spanner-cli) のフォークである  [apstndb/spanner-mycli](https://github.com/apstndb/spanner-mycli) の開発を通じて、Spanner のネイティブ機能をどのようにコマンドラインツールへ統合できるかを模索することをライフワークの一つとしています。

https://zenn.dev/apstndb/articles/introduce-spanner-mycli

この活動を通して、Spanner のほぼ全ての機能に対して一定の関心を持っており、事前スプリット機能についてもリリース翌日の4月29日には早速 spanner-mycli へ[実装](https://github.com/apstndb/spanner-mycli/commit/6bf0fbf8f8c26670f210bf46af0fa28f5f1967ca)を試みました。本記事では、その実装内容を紹介します。

### `ADD SPLIT POINTS`

現在、公式に pre-splitting を Spanner に指示する方法は [Create and manage split points](https://cloud.google.com/spanner/docs/create-manage-split-points) に書かれているように下記の方法があります。

* クライアントライブラリを使うか直接 REST/gRPC API で `AddSplitPoints`([REST](https://cloud.google.com/spanner/docs/reference/rest/v1/projects.instances.databases/addSplitPoints#SplitPoints), [gRPC](https://cloud.google.com/spanner/docs/reference/rpc/google.spanner.admin.database.v1#google.spanner.admin.database.v1.DatabaseAdmin.AddSplitPoints) API を利用する。
* [`gcloud spanner databases splits add`](https://cloud.google.com/sdk/gcloud/reference/spanner/databases/splits/add) にテキストファイルを渡す

Google Cloud 公式の Web UI である Spanner Studio は現時点（記事執筆時点）ではこの機能に未対応であり、対話的にスプリットポイント(split points)を追加する公式な方法も提供されていません。
そのため、専用のツールを自作しない限り、スプリットポイントの追加には `gcloud` コマンドを利用するのが一般的でしょう。

spanner-mycli では、この `gcloud spanner databases splits add` コマンドが受け付けるファイル形式に似た構文でスプリットポイントを指定できる、独自のクライアントコマンド `ADD SPLIT POINTS` を実装しました。

```text
spanner> ADD SPLIT POINTS
         TABLE Singers (100)
         TABLE Singers (200)
         TABLE Singers (300)
-- expire, named schema, index, index with tablekey
spanner> ADD SPLIT POINTS EXPIRED AT "2025-05-05T00:00:00"
         TABLE Singers (42)
         TABLE sch1.Singers (21)
         INDEX SingersByFirstLastName ("John", "Doe")
         INDEX SingersByFirstLastName ("Mary", "Sue") TableKey (12);
```

### `SHOW SPLIT POINTS`

前述のどれかの方法で追加した split point を確認する手段には下記のようなものがあります。

* [`SPANNER_SYS.USER_SPLIT_POINTS`](https://cloud.google.com/spanner/docs/create-manage-split-points#console) VIEW
* [`gcloud spanner databases splits list`](https://cloud.google.com/sdk/gcloud/reference/spanner/databases/splits/list)

注目すべき点として、 `SPANNER_SYS` を介して SQL で取得する他にスプリットポイント取得のための公式な API は提供されていないことに注意してください。
`gcloud spanner databases splits list` コマンドも、内部的には `SPANNER_SYS.USER_SPLIT_POINTS` ビューにアクセスして情報を取得しています。

一般的に、 `SPANNER_SYS` や `INFORMATION_SCHEMA` 内のシステムビュー名を正確に記憶し、アドホックにクエリを実行するのは容易ではありません。そこで spanner-mycli では、 `ADD SPLIT POINTS` コマンドとの一貫性を考慮し、 `SHOW SPLIT POINTS` というクライアントコマンドを提供しています。

```text
spanner> SHOW SPLIT POINTS;
+--------------+------------------------+---------------------+-----------------------------------------------------------------------------------------------+-----------------------------+
| TABLE_NAME   | INDEX_NAME             | INITIATOR           | SPLIT_KEY                                                                                     | EXPIRE_TIME                 |
| STRING       | STRING                 | STRING              | STRING                                                                                        | TIMESTAMP                   |
+--------------+------------------------+---------------------+-----------------------------------------------------------------------------------------------+-----------------------------+
| Singers      |                        | cloud_spanner_mycli | Singers(42)                                                                                   | 2025-05-09T11:29:43.928097Z |
| Singers      | SingersByFirstLastName | cloud_spanner_mycli | Index: SingersByFirstLastName on Singers, Index Key: (John,Doe), Primary Table Key: (<begin>) | 2025-05-05T00:00:00Z        |
| Singers      | SingersByFirstLastName | cloud_spanner_mycli | Index: SingersByFirstLastName on Singers, Index Key: (Mary,Sue), Primary Table Key: (12)      | 2025-05-05T00:00:00Z        |
| sch1.Singers |                        | cloud_spanner_mycli | sch1.Singers(21)                                                                              | 2025-05-05T00:00:00Z        |
+--------------+------------------------+---------------------+-----------------------------------------------------------------------------------------------+-----------------------------+
4 rows in set (0.26 sec)
```
:::message
`SHOW SPLIT POINTS` 表示されるカラムは `SPANNER_SYS.USER_SPLIT_POINTS` から無加工です。
`ADD SPLIT POINTS` は `initiator` として `spanner_mycli` を指定しているので `cloud_spanner_mycli` が `INITIATOR` として記録されており、このスプリットポイントが spanner-mycli から追加されたことがわかるようになっています。
:::
### `DROP SPLIT POINTS`

追加したスプリットポイントを直接的に「削除」する方法は、公式ドキュメントには明記されていません。しかし、スプリットポイントの有効期限 (expiration time) は更新可能であり、有効期限を過去の時刻に設定することで、実質的に即時失効（削除と同様の効果）させることができると説明されています。

https://cloud.google.com/spanner/docs/pre-splitting-overview#expire

> You can also update the expiration time for a split point before it expires. For example, if your increased traffic hasn't subsided, you can increase the split expiration time. If you no longer need a split point, you can set it to expire immediately. To learn how to set the expiration time of split points, see How to expire a split point.

spanner-mycli では、 `ADD SPLIT POINTS` コマンドとの一貫性を保ちつつ、この挙動を利用した `DROP SPLIT POINTS` クライアントコマンドを実装しました。
内部的には、指定されたスプリットポイントの有効期限を過去の時刻（実装の詳細としては UNIX エポック）に設定することで、削除機能を実現しています。

```text
spanner> DROP SPLIT POINTS
         INDEX SingersByFirstLastName ("Mary", "Sue") TableKey (12);
```

## まとめと今後の構想

この記事では Spanner の pre-splitting の簡単な説明と spanner-mycli への落とし込みを説明しました。
spanner-mycli の現在の実装ではまだ対応できていない課題として、負荷分散のためにテーブルへ多数のスプリットポイントを追加する際に、適切なキー値を決定し、それらを手動で入力する作業の煩雑さが挙げられます。
理想としては、対象のテーブルやインデックス、そしてキー範囲と分割数などの希望を指定するだけで、ツールが自動的に適切なスプリットポイントのキー値を計算し、 `AddSplitPoints` API を呼び出してくれるような機能が望ましいと考えています。