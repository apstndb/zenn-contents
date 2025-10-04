---
title: "GA となった公式 Spanner CLI へのフィードバックのススメ"
emoji: "🤔"
type: "tech"
topics: ["cloudspanner", "gcp", "database", "cli"]
published: true
---

## はじめに

2025年10月2日、 Google Cloud による公式 Spanner CLI が GA (General Availability) となりました。これは、[spanner-cli コントリビュータが見る公式 Spanner CLI](https://zenn.dev/apstndb/articles/spanner-cli-contributor-perspective)で紹介したプレビューリリースから約3ヶ月後のことです。
プレビューで私は多くの期待を持ちましたが、 3ヶ月待っての GA をした今もそれに応えたとは言えない状況です。

この記事では、 GA となった公式 Spanner CLI の現状をシビアに評価し、なぜ既存のコミュニティ製ツールの代替となり得ないのかを解説します。そして、すべての Spanner ユーザに、ツールの未来を共に創るための行動を呼びかけます。

:::message
この記事では、`gcloud components install spanner-cli` でインストールできるものを「公式 Spanner CLI」、GitHubの [cloudspannerecosystem/spanner-cli](https://github.com/cloudspannerecosystem/spanner-cli) で開発されているものを「OSS spanner-cli」と呼称します。
:::

## Preview 公開から GA までの間で何が変わったのか？

Wayback Machine の差分機能を使って、プレビューリリース直後（2025年6月30日）と GA 後（2025年10月4日）の[公式ドキュメントを比較](https://web.archive.org/web/diff/20250630072356/20251004090833/https://cloud.google.com/spanner/docs/spanner-cli)してみると、その変化は驚くほど小さいものでした。

- `Preview` のバナーが消えたこと
- コマンドが `gcloud alpha spanner sql` から `gcloud spanner sql` に変わったこと

また、 `gcloud components` としてのバージョンは 1.0.3 に上がっているものの、プレビューで私が期待した改善は観測できませんでした。

それどころか、プレビュー期間中に通常パスを通すことになる `google-cloud-sdk/bin` に置かれたバイナリ名が `spannercli` から `spanner-cli` へと変更されることで OSS spanner-cli と実行コマンドの名前が衝突事故が発生しています。
これは Google の想定外の可能性が高いですが、両方の併用を考えているユーザにとっては明らかに不便な変更です。

結論としては、 GA になったことでプロダクション環境で公式サポートが受けられるようになった点は大きな前進ですが、その一方で、対話型ツールとしての機能性に向上は見られませんでした。

## なぜ公式 Spanner CLIはOSS spanner-cli の代替になり得ないのか

公式 Spanner CLI が GA になったことで、[表明されていた](https://zenn.dev/apstndb/articles/spanner-cli-contributor-perspective#%E8%AC%9D%E8%BE%9E) OSS spanner-cli をアーカイブする条件は形式上満たされました。しかし、本当に今、アーカイブしてしまって良いのでしょうか。

### 「アーカイブ」という行為の重み

GitHubリポジトリの「アーカイブ」は、単なる更新停止以上の「今後一切のメンテナンスを行わず、Issueでのやり取りも受け付けない」という開発者からの強い意志表明です。
これはメンテナンスが不可能になったり完全に役割を終えた場合に行われるべき最後のアクションであり、多くのユーザは「OSS としての死」として受け取るでしょう。

代替が不十分な状態で行えば、ユーザや企業は路頭に迷い Spanner コミュニティの混乱をもたらすだけです。

### クリティカルな機能の欠如

公式 Spanner CLI が OSS spanner-cli の代替として役割を果たせるのであれば問題ありません。
しかし、現在公式の Spanner CLI が対話型セッション内で正式にサポートしている機能はドキュメントを読む限りだとこれらに限られます。

> - Directly type GoogleSQL statements:
>   - GoogleSQL DDL reference
>   - GoogleSQL DML reference
>   - GoogleSQL query syntax
> - Execute transactions
> - Use supported meta-commands

"Execute transactions" が何を指すかは明示されていませんが、 `BEGIN`, `COMMIT`, `ROLLBACK` 等を指すと解釈することは可能です。
しかしドキュメントに明示的に書かれている機能としては Spanner がネイティブでサポートしている GoogleSQL の DDL, DML, クエリの実行以外は何も正式にサポートしていないと解釈すべきでしょう。

OSS spanner-cli のドキュメントの [Syntax](https://github.com/cloudspannerecosystem/spanner-cli?tab=readme-ov-file#syntax) の節に、 OSS spanner-cli 固有の機能のテーブルがあります。上記に該当しない機能を列挙してみましょう。

- **スキーマ確認機能**                                                                      
    - `SHOW DATABASES`, `SHOW TABLES`, `SHOW CREATE TABLE`, `SHOW COLUMNS`, `SHOW INDEX`   
- **データベース・ロール切り替え**                                  
    - `USE <database> [ROLE <role>]` （公式 Spanner CLI の `\u` メタコマンドではセッション内でのロール変更不可）
- **Partitioned DML を使ったテーブルデータ更新**
    - `TRUNCATE TABLE`, `PARTITIONED {UPDATE|DELETE}`                                                           
- **クエリ分析機能**
    - `EXPLAIN`, `EXPLAIN ANALYZE`, `DESCRIBE`

これらに依存している場合は、現状公式 Spanner CLI は OSS spanner-cli の代替として使えない場合があるということになります。

特に `EXPLAIN`, `EXPLAIN ANALYZE` を公式 Spanner CLI が現時点で正式にサポートしていないことは致命的です。

私は [Cloud Spanner の実行計画の活用に関する取り組み](https://engineering.mercari.com/blog/entry/20201210-cloud-spanner-query-plan/)で述べたように Spanner の実行計画を活用するユースケースに有用なように OSS spanner-cli へのコントリビューションを続けてきました。

[DeNA](https://engineering.dena.com/blog/2024/05/loadtest-approach-for-large-scale-application-of-spanner/) や[コロプラ](https://blog.colopl.dev/entry/2025/05/26/110000) といった企業の技術ブログを見ても、この機能がユーザのプロダクトに使われるデータベースのスキーマ・クエリの最適化という重要な業務に活用されていることは明らかです。
Google のカスタマーエンジニアによる [Cloud Spanner の実行計画を読み解く](https://zenn.dev/google_cloud_jp/articles/726fcaf614f26e) においても `EXPLAIN ANALYZE` がフィルタ条件を含む静的な実行計画と実行時プロファイル情報を共有可能な形で一覧するために重要な立ち位置をしめています。

この機能のサポートがない現状では、公式 Spanner CLI がOSS spanner-cli の代替としての役割を果たせるとは到底言えません。

:::message
以前の記事で推測したように、公式に説明されていませんが公式 Spanner CLI は OSS spanner-cli からフォークしたものであり、文書化されていない機能が動く場合もあります。
そのような製品の正式な仕様とは言えない未定義の挙動に業務を依存させるのは、将来のバージョンで動作が保証されないという互換性のリスクを抱えることになり、ビジネス上許容しがたいでしょう。
:::

## ソフトランディングへの道：メンテナンスモードという選択


このような状況を踏まえ、OSS spanner-cli のオーナーであるYuki Furuyamaさんと一日かけて議論した結果、代替可能な状態となるまではアーカイブは時期尚早であり、リポジトリを「メンテナンスモード」とすることで合意しました。

これにより、ユーザは移行準備が整うまでは OSS spanner-cli を使い続けることができます。もし致命的な問題が発生しても、 Issue 報告や Pull Request によってコミュニティベースで解決する道が残されています。

## Spannerユーザへの提案：未来はフィードバックにかかっている

ここまでで見てきた通り、公式 Spanner CLI は GA になった今も機能が著しく限定されているだけでなく、今後どのような方向に向かうかどうかのロードマップなども示されておらず今後に期待できるかどうかすら不透明な状態です。
ですが、主要なデータベースは全て単なるコミュニティ有志による OSS ではなく成熟した公式の対話型ツールを持っています。本来であれば公式の Spanner CLI で解決されるべき問題が数多くあるはずです。

もちろん、正式に文書化された機能集合を絞った GA には製品サポート体制の確立やコア機能の安定化を優先するといった、 Google 側の戦略的な判断があったのかもしれません。
しかし、結果として現状の機能セットは、データベースの公式ツールとして OSS spanner-cli を既に使っているかもしれない Spanner ユーザの需要と期待とは乖離があるように見受けられます。
これは、開発チームと現場のユーザとの間で双方向の繋がりが薄く、まだ認識の差があることの表れではないかと私は推測します。

待っていても公式 Spanner CLI がよくなることは期待できないかもしれません。
しかし、製品の一部として GA となった公式 Spanner CLI は Google サポートの対象であり、企業ユーザが持つあらゆるコネクションが公式 Spanner CLI へのフィードバックに活用できます。
これまで OSS spanner-cli にも関わってこなかった企業ユーザだとしても、自分たちの業務に不可欠なツールとしてシビアな目で公式 Spanner CLI を評価し、Google にフィードバックをしてはいかがでしょうか。

例えば、私自身も早速この記事で指摘した `EXPLAIN` および `EXPLAIN ANALYZE` のサポートについては Google 公式の Issue Tracker に [Support of EXPLAIN and EXPLAIN ANALYZE in Spanner CLI](https://issuetracker.google.com/issues/449023251) の Issue を通して要望を行うアクションを行いました。
このように具体的に公式 Spanner CLI に求めることを伝えることがツールの成長を促す確実な一歩となります。

公式 Spanner CLI が OSS spanner-cli の代替を超えて Spanner ユーザが求める標準ツールに成長する道はフィードバックの積み重ね以外にはないだろうと信じます。

## 最後に

Spanner ユーザの皆さんも、 Google にフィードバックしつつ、それでも満たされないニーズについては、自らツールを開発したり、コミュニティの OSS に貢献したりと、などの多様な形で解決を模索する道があります。

相談先を探しているなら、Google Cloudコミュニティの [GCPUG Slack](https://gcpug.jp/) の `#cloud-spanner` チャンネルがおすすめです。私を含め、日本の Spanner 関連ツールの開発者が参加しており、きっと力になれるはずです。

私個人に関しても、この記事で書いた実行計画の扱いなどは公式 Spanner CLI に改善される余地は残されていますが、 製品の一部としてクローズドに開発されるものが個人的なニーズを完全に満たすとは期待していません。
私だけが求める機能は、[私が spanner-cli をフォークした理由: spanner-mycli の紹介](https://zenn.dev/apstndb/articles/introduce-spanner-mycli)で紹介した `apstndb/spanner-mycli` の開発を続けることで、 OSS として形にしていこうと考えています。