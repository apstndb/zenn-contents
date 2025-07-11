---
title: "Spanner は本当に高い？思ったよりも低い Firestore との損益分岐点"
emoji: "💰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gcp", "spanner", "firestore", "database", "cost"]
published: true
---

カウシェ Tech Blog の [Firestore → Cloud SpannerでDBコスト93%削減！無停止でやり切った 1 年間の全記録](https://zenn.dev/kauche/articles/1e733da3748ee1) という記事が公開されました。
ある程度利用量が増えたら Spanner よりも Firestore の方が高くなるというのは Firestore の前身である Datastore 含め使ったことがあると経験するものですが、データベース移行は大変なので移行できていないユーザは多いでしょう。
無停止での Firestore から Spanner への移行という大変な仕事をやり切ったことは素晴らしいことですね。

ところで、この記事への反応を見ていると特定のポストの引用はしませんが Firestore から Spanner に移行したことで安くなることそのものへの驚きの反応をいくつか観測しました。

https://x.com/search?q=https%3A%2F%2Fzenn.dev%2Fkauche%2Farticles%2F1e733da3748ee1&src=typed_query&f=live

実際には Firestore のように Google Cloud の中でも Spanner よりも大幅に高価になりかねない選択肢があるにも関わらず、 何故このような反応があるのでしょうか？
これには Firestore に毎日の無料枠がありスモールスタートできるのに対し、 Spanner は無料ではなく、更に「Spanner は高価である」という先入観ばかりが根強く広まっているからであると考えられます。

この先入観は、最適な技術選定を妨げ、開発者とビジネスの双方、更に Google にとっても良い選択をする機会の損失につながること、具体的な収益性の悪化、また、将来的に難易度の高いデータベース移行を強いる大きな技術的負債の原因になってしまう可能性があります。

Google 自身この先入観を払拭するために [Cloud Spanner の誤解を打ち破る](https://cloud.google.com/blog/ja/products/databases/cloud-spanner-myths-busted) という記事を書いたり、
この記事を土台にしたセッション([Cloud Spanner 神話とその真実 〜 噂の真相にせまる 〜](https://www.youtube.com/watch?v=07dwnDAvkoE) など)を行うなど苦心しているように見えます。

:::message
なぜこのような先入観が広まったかと言えば、上記 Google の記事にもありますが次のような過去のイメージが残っているからでしょう。
- SLA が提供されるプロダクション用インスタンスの最低価格が高かった。
  - 最低3ノード、最も安価な us-central1 でも月額約 $2000 だった。
    - [2019年9月](https://cloud.google.com/spanner/docs/release-notes#September_25_2019) には既に1ノードでも SLA が提供されるようになり1/3に。
    - [2022年5月 GA](https://cloud.google.com/spanner/docs/release-notes#May_31_2022) の granular instance sizing で 0.1 ノードでも SLA が提供されるようになり更に 1/10 に。
    - 既に Cloud SQL の db-g1-small インスタンスの HA 構成よりも同等程度
- OSS ではないので無料で開発環境が作れないため、開発環境にも同様にコストが掛かった。
  - 完全ではないですが、[公式の Spanner エミュレータ](https://cloud.google.com/spanner/docs/emulator)([2020年7月GA](https://cloud.google.com/spanner/docs/release-notes#July_30_2020))、[無料トライアルインスタンス](https://cloud.google.com/spanner/docs/free-trial-instance)([2022年9月GA](https://cloud.google.com/spanner/docs/release-notes#September_08_2022)) などが追加
:::

これらの発信はありますが、あまり具体的な数字は出てこないため読んだ人でもまだ実感としてしっくり来ていない人が多いかもしれません。かと言って他社製品との比較もあまり良い顔はされないでしょう。

そこで私が考えていたのは、そもそもあまり定量的な比較が行われていない Google Cloud のデータベース間の比較を公開することに意味があるのではないか？ということです。

## Firestore と Spanner の価格比較

ということで、この記事では、カウシェで顕在化した Firestore と Spanner の間の価格の比較を行います。

:::message
- Google Cloud の Database offering の中で Firestore との単純化した比較のみを行います。
  - Firestore はバックエンドから使うデータベースとしての利用を想定します。
    - よって Datastore mode と Native mode の違いはあまり結論に関係がありません。 Native mode 固有の機能については割愛します。
  - 他社サービスとの比較はしません。
  - 一般的な SQL と NoSQL の比較、 SQL と分散 SQL の比較などはしません。
- ドキュメント上から読み取れる情報のみで比較します。
  - 負荷試験等は含まれません。
- 注意喚起が目的でもあるので、Firestoreのコスト面における課題については、データに基づき率直に指摘します。
:::

### Firestore/Spanner の公式ドキュメントから関係ある値を抽出

- 各値は2025年7月9日現在のものです。
- 日本語で書いている記事なので asia-northeast1(東京リージョン) に揃えます。
- 通貨単位は為替レートに左右されないように原則 USD のまま扱いますが、必要に応じて 1 USD = 146 JPY で変換します。
- 確定利用割引(CUD) は Firestore, Spanner 同じ割合(1年20%、3年40%)なので相対比較に影響ないため省略します。
- Google Cloud のネットワーク料金(下り料金など)は無料枠を除き Firestore, Spanner ともに同じ値です。構成によっては総額としてのコスト計算する場合に影響はありますが、今回の比較のスコープには含めません。

#### Firestore

Firestore は完全に従量課金なので見るべき情報は Pricing のページに全てまとまっています。

- [Firestore pricing](https://cloud.google.com/firestore/pricing?hl=en)
  - Firestore Standard edition についての記述。 Enterprise edition についてはあとで少しだけ触れます。
- [Firestore in Datastore mode pricing](https://cloud.google.com/datastore/pricing?hl=en)

Firestore の Native mode と、 Datastore mode はそれぞれ別のページで説明しており、用語がいくつか異なりますが価格体系そのものは同一です。
バックエンドの利用なので Datastore mode でも良いですが、新しい Native mode の用語に統一しましょう。

| Firestore Native mode | Datastore mode     | Free usage     | Cost                                  |
|-----------------------|--------------------|----------------|---------------------------------------|
| Document reads        | Entity reads       | 50,000 per day | $0.038 per 100,000 entities/documents |
| Document writes       | Entity writes      | 20,000 per day | $0.115 per 100,000 entities/documents |
| Document/TTL deletes  | Entity/TTL deletes | 20,000 per day | $0.013 per 100,000 entities/documents |
| Stored data           | Stored data        | 1 GiB          | $0.115/GiB/month                      |
| PITR data             | PITR data          | N/A            | $0.115/GiB/month                      | 
| Backup data           | Backup data        | N/A            | $0.038/GiB/month                      |
| Restore operation     | Restore operation  | N/A            | $0.256/GiB                            |
| N/A                   | Small operations   | 50,000 per day | Free (課金されるのは multi-regional のみ)      |

:::message
余談ですが、 Firestore Native modeには small operations という概念はありません。Index entry reads, aggregation queries などは1000件あたり1ドキュメント分としてカウントされます。

また、[Firestore editions overview](https://cloud.google.com/firestore/native/docs/editions-overview) のページに Firestore Standard edition のストレージは Hybrid storage (SSD & HDD) であると明記されています。
:::

#### Spanner
Spanner は確保した Processing Units(PU) というキャパシティに対して課金されるので、見るべき情報はこの二つとなります。

- [Spanner pricing](https://cloud.google.com/spanner/pricing)
- [Performance overview](https://cloud.google.com/spanner/docs/performance)

##### Compute Capacity

| Edition         | Cost per node (including all three replicas) | Cost per node with 1-year committed use discounts (including all three replicas) | Cost per node with 3-year committed use discounts (including all three replicas) |
|-----------------|----------------------------------------------|----------------------------------------------------------------------------------|----------------------------------------------------------------------------------|
| Standard        | **$1.17 per hour**                           | $0.936 per hour                                                                  | $0.702 per hour                                                                  |
| Enterprise      | $1.599 per hour                              | $1.2792 per hour                                                                 | $0.9594 per hour                                                                 |
| Enterprise Plus | $2.223 per hour                              | $1.1784 per hour                                                                 | $1.3338 per hour                                                                 |

Firestore との比較には Standard に含まれている機能だけで十分なのでここからは Standard のみに言及します。
必要に応じて [Spanner editions overview](https://cloud.google.com/spanner/docs/editions-overview) を確認してください。

##### Database Storage

| Storage Type | Cost per GB (including all three replicas) |
|--------------|--------------------------------------------|
| SSD          | **$0.39 per GB per month**                 |
| HDD          | $0.078 per GB per month                    |

HDD が選べる tiered storage は高度な機能なので、今回は SSD のみを対象として比較します。

##### カタログ上のスループット概算値

データベース性能について議論する場合は通常はワークロードに沿った負荷試験が必要ですが、 Spanner には単純なワークロードにおけるカタログスペックとしての概算スループットがドキュメンテーションされています。

https://cloud.google.com/spanner/docs/performance?hl=en#increased-throughput

下記のように 1KB の行の単純な read もしくは write だけをするシナリオなので一般的な SQL の処理とは掛け離れていますが、 SQL ほど複雑なクエリは実行できずキーの読み書きが中心となる Firestore との比較にはある程度合理的な近似でしょう。

> Read guidance is given per region (because reads can be served from any read-write or read-only region), while write guidance is for the entire configuration. Read guidance assumes you're reading single rows of 1KB. Write guidance assumes that you're writing single rows at 1KB of data per row.

シングルリージョン(asia-northeast1) に該当する概算スループットを抜き出します。

> [Performance for regional configurations](https://cloud.google.com/spanner/docs/performance#regional-performance)
> Each 1,000 processing units (1 node) of compute capacity can provide the following peak performance (at 100% CPU) in a regional instance configuration:

| Peak reads (QPS per region)     | Peak writes (QPS total)           | Peak writes using [throughput optimized writes](https://cloud.google.com/spanner/docs/throughput-optimized-writes) (QPS total) |
|:--------------------------------|:----------------------------------|:--------------------------------------------------------------|
| **SSD: 22,500** <br> HDD: 1,500 | or **SSD: 3,500** <br> HDD: 3,500 | SSD: 22,500 <br> HDD: 22,500                                  |

- 前述したように SSD ストレージのみを使用
- [throughput optimized writes](https://cloud.google.com/spanner/docs/throughput-optimized-writes) は書き込みを遅延させても良いバッチワークロードで100ms遅延を許容した際の値なので今回の比較には無関係

よって、強調した値のみが今回気にすべき項目です。

:::message
(追記) 実はこのカタログスペックは途中で大幅に向上しています。
[Cloud Spanner の価格性能比の大幅な改善を発表](https://cloud.google.com/blog/ja/products/databases/announcing-cloud-spanner-price-performance-updates) の記事で書かれたもので、 リリースノート上は[2023年10月11日](https://cloud.google.com/spanner/docs/release-notes#October_11_2023)に発表された [Performance and storage improvements](https://cloud.google.com/spanner/docs/performance#improved-performance) というのがそれで、典型的なスループットは Read/Write 共に1.5倍、ついでにストレージの限界が2.5倍というものでした。
数ヶ月掛けて全てのリージョナル、デュアルリージョン、マルチリージョンインスタンス構成にロールアウトされたため今は当たり前になっています。それ以前の検証を見る場合はこの改善も念頭に入れて見ましょう。
:::

##### Spanner 最小構成の価格とスループット

[Compute capacity](https://cloud.google.com/spanner/docs/compute-capacity?hl=en#compute-capacity) のページにあるように、 Spanner インスタンスは1ノードを1000 Processing Units(以下 PU) として、 0.1 ノードである 100 PU のインスタンスから作ることができます。
100 PU であっても SLA は適用されるため、本番環境での利用も可能です。
よって、上記のノードあたりのものを最小構成に補正したものが下記のようになります。

| 項目                             | 値               |
|--------------------------------|-----------------|
| Standard edition cost per node | $0.117 per hour |
| Peak reads (QPS per region)    | 2,250           |
| Peak writes (QPS total)        | 350             |

なお、前述した通りこれは CPU 利用率を100%使った時の値です。公式ドキュメント([CPU utilization metrics: Alerts for high CPU utilization](https://cloud.google.com/spanner/docs/cpu-utilization#recommended-max))にある通り、本番環境のリージョナルインスタンスでは高優先度タスクによるCPU使用率を 65% 以下に保つことが推奨されています。今回の比較でも、この値を考慮に入れます。

:::message
なぜ 65% が推奨されているのかは公式ドキュメント上は明言はありませんが、これはゾーン障害時にもサービスレベルを維持するための可用性への思想の表れと解釈できます。
Spanner リージョナルインスタンスではリージョンの3つのゾーンそれぞれに R/W レプリカを分散配置しています。1ゾーンがダウンした場合 1/3 のキャパシティが失われるため、同じ負荷を 2/3(約67%) のキャパシティで捌かなければならなくなります。
安全マージンを加味して常に65%までの負荷で稼働するようにしていれば、ゾーン障害が発生しても CPU 利用率が100%を超えてサービスに致命的な影響が出ることが避けられます。
デュアルリージョン・マルチリージョンインスタンスでは 45% なことも矛盾がなく説明できます。

|                      |Regional|Dual-region/multi-region|
|----------------------|--------|------------------------|
| 推奨設定                 |65%|45%|
| R/W レプリカの数           |3|2|
| 1ゾーン失われた時に残るキャパシティ   |2/3(67%)|1/2(50%)|
:::

### 比較

ここからは、 Spanner 最小構成と Firestore の損益分岐点に着目して比較します。

- 単純化のため、 1KB の Read のみ、 Write のみがそれぞれ定常的に行われるワークロードを想定する。
  - ワークロードに対するスキーマは適切に設計され、理論値の性能が出ることとする。
    - なお 1000 PU 以下では水平分散しないためこの記事の範囲ではホットスポットがあったとしても実は問題ない。
  - Read と Write の比率が変わっても損益分岐点が Read 100% と Write 100% の結果の間で動く程度なので単純化を優先する。
    - むしろそれぞれで比較した方が Read/Write それぞれがどれだけ差が出るのかが浮き彫りにできる。
- Firestore の無料枠(free usage) を考慮する。一日あたりで定義されているので、計算は一日あたりで行う。
  - Spanner の free trial は SLA が提供されない特殊なインスタンスとして提供されるため、この検証では扱わない。
- Spanner における 65% の推奨も考慮する。

まず、単純化のため 100000 による切り上げを省いて連続近似すると、1日あたりの損益分岐点は `Firestoreコスト = Spannerコスト` となる点、
つまり $$x = \frac{C_{\text{spanner}}}{P_{\text{firestore}}} + F_{\text{free}}$$ の `x` を求めることで得られます。

**変数の定義**：
- $x$ = 1日あたりの操作数（求めたい値）
- $F_{\text{free}}$ = Firestore の無料枠操作数（読み取り: 50,000、書き込み: 20,000）
- $P_{\text{firestore}}$ = Firestore の操作単価（読み取り: \$0.00000038/op、書き込み: \$0.00000115/op）
- $C_{\text{spanner}}$ = Spanner の日額コスト（100 PU = $2.808/日）

:::message
より正確には100,000単位の切り上げを考慮する必要があります。
損益分岐点は、以下のステップで計算できます。

1. Spannerの日額コストが、Firestoreの何回分のオペレーションに相当するかを計算する (`Spannerコスト ÷ Firestore単価`)。
2. Firestoreの課金単位である100,000回で切り上げる。
3. 最後に、Firestoreの無料枠の操作数を加算する。これを数式で表すと、以下のようになります。

$$x = F_{\text{free}} + \lceil\frac{C_{\text{spanner}}/P_{\text{firestore}}}{100,000}\rceil \times 100,000$$
:::

ここから、分かりやすいように matplotlib でグラフにしたものを示します。
オペレーションの課金単位は 100,000 単位なので本来は細かい階段状(鋸状？)のグラフになりますが、100,000 よりもかなり大きいスケールを扱うグラフの単純化のためプロットは連続値として近似しています。(損益分岐点の計算などは正確なはずです。)

#### Read

Read 単体の損益分岐点は 7,450,000 reads/day ≒ 86 reads/sec で到達します。
これは 100 PU の Spanner インスタンスのピーク性能の約 3.8% 程度です。

![](/images/firestore_spanner_read_comparison.png)

モニタリング推奨設定通り Spanner インスタンスの理論スループットの 65% まで使うとすると、 Spanner インスタンスは Firestore の1/17のコストという計算になりました。
これは、一定のスケールを超えた場合、Firestoreを使い続けることがいかに大きなコスト増につながるかを明確に示しています。

#### Write

Write 単体の損益分岐点は 2,520,000 writes/day ≒ 29 writes/sec で到達します。
これは 100 PU の Spanner インスタンスのピーク性能の約 8.1% 程度です。

![](/images/firestore_spanner_write_comparison.png)

モニタリング推奨設定通り Spanner インスタンスの理論スループットの 65% まで使うとすると、 Spanner インスタンスは Firestore の1/8のコストという計算になりました。

#### Storage

前述したように、 Firestore は SSD と HDD のハイブリッドストレージを使っていて単純比較が難しいことなどがあり、 ストレージの損益分岐点について語ることは困難です。

目安として、 [Database limits](https://cloud.google.com/spanner/quotas#database-limits) には下記の記述があります。

> Storage size
> ...
> For instances smaller than 1 node: 1024.0 GiB per 100 processing units

100 PU のインスタンスの限界までストレージを使ったとしても、最大でも 1024 GiB なので、下記の表のようになります。

| Service   | Unit Price(per GiB per month) | Monthly Price |
|-----------|-------------------------------|---------------|
| Spanner   | $0.39                         | $399.36       | 
| Firestore | $0.115                        | $117.76       |
| ratio     |                               | 339%          |
| diff      |                               | $281.6        |

単価の違いがあるため Spanner の方が高価ではありますが、 Read や Write で 100 PU のインスタンスを使い切るケースでは一日数十ドルの差が出ることを確認しているため、支配的になるオーダーではないと考えられます。
現実的には履歴データとして塩漬けするとしてもこの量のデータを投入している時点でオペレーションコストも掛かりますし、 100 PU の Spanner インスタンスの価格を気にするような規模で 1 TiB ものデータを扱う場合、そのデータへのアクセスパターンによっては、ストレージコストよりも先にオペレーションコストが支配的になる可能性が高いと考えられます。

### 考察

Spanner が絶対的に安いかどうかはともかく、ワークロードの条件を揃えた上で Firestore と比較すると一定の規模から先は相対的に大幅に安くなるケースが多いということは計算からも分かりました。
これは不思議なことではありません。
そもそも Spanner と Firestore は強い整合性、地理分散による高可用性という同等の保証を提供するものであるため、データモデルや課金モデルを除けば似た種類に分類されるデータベースだと考えることができます。
これらの保証は非常に強力なものである反面、フリーランチであるということはありません。よってより弱い保証のデータベースと比べれば理論上は不利ですが、少なくとも同じ保証のデータベースと比べる場合は合理的な価格設定となっているはずではあります。

そして、 (Flex VM のような余ったリソースを使ったバッチ実行などを除けば)一般的にあらかじめプロビジョニングされたキャパシティはオンデマンドのオペレーション従量課金より程度の差はあれディスカウントがあるものです。

:::message
実のところ、 Firestore と Spanner は単に似ているだけではありません。
Firestore 発表当初から Spanner と同じ技術を使っていることは示唆されていましたし、 後に Google の論文で Firestore は Spanner を使って実装されていることが既に説明されています。

言い換えると、 Firestore は Spanner という強力なデータベース基盤を SQL の実行計画の複雑さやキャパシティプランニングを考えずに無料からスモールスタートで使えるように抽象化した付加価値の高いマネージドサービスです。
理論的には Firestore ではこの付加価値の対価を設定していることがプロビジョニングモデルで割安な Spanner との価格差に現れている、と考えることもできるでしょう。

Google の論文とそこに書かれた事実についてはこの記事では詳しくは説明しないので、グーグル・クラウド・ジャパン合同会社 中井悦司氏によるグーグルクラウドに関連する技術コラム グーグルのクラウドを支えるテクノロジー 内の下記記事を調べてみてください。

- [第153回　サーバーレスNoSQLデータベースサービス「Firestore」（パート1） (中井悦司)](https://www.school.ctc-g.co.jp/columns/nakai2/nakai2153.html)
- [第197回　Cloud DatastoreからFirestoreへのデータマイグレーション（パート1） (中井悦司)](https://www.school.ctc-g.co.jp/columns/nakai2/nakai2197.html)
:::


## Firestore を検討する人へ

サーバサイドのバックエンドとしてFirestore を検討しているのであれば、下記のことを心に留めると良いでしょう。

- FirestoreがSpannerよりコスト面で有利なのは、1日の操作回数が無料枠から数百万回に達するまで、という水平分散するデータベースとしては狭い範囲に限られます。
- ユーザ数 × ユーザあたりのオペレーション数(ドキュメント数) で高価になっていくことをまず意識しましょう。
- 組織内に限定公開できるなどユーザ数が一定になるようなサービスならほぼ安全ですが、そうでないなら必ず Spanner の方が安くなる可能性を検討しましょう。
- 100 PU の Spanner インスタンスの月額コスト（約$85、1ドル146円換算で約12,000円）が見合うようなサービスであれば Spanner を最初から選んだ方が安心できるでしょう。

:::message
Firestore Native mode はクライアントから使えるリアルタイムデータベースです。
リアルタイムデータベースがほしいにも関わらず Spanner を使うとしたら自前で実装しないといけないものが非常に多くなるため、その選択は普通の企業にはあまり現実的ではないでしょう。
よって、比較対象は Spanner ではなく Firebase Realtime Database のような別のサービスとなるため、判断は変わってくるはずです。
また、クライアントのオフラインキャッシュを有効活用することでコスト削減もできるでしょう。
:::

## 最後に

個人や小規模開発に使われやすい Firestore は安価なデータベースだという印象が持たれていますが、本記事の分析では、Spanner最小構成（月額約12,000円）が持つ性能の10%にも満たない負荷で、Firestoreとのコストが逆転する損益分岐点に達することが分かりました。
その後は Spanner の方が Firestore より Read で17倍、 Write で8倍安いという計算となりました。
月額約12000円というのは Google Cloud の他のデータベースでゾーン障害に耐える最小の構成をした場合と比べても大幅に高いということはありません。

どうでしょうか。まだ「Spanner は高い」でしょうか。
もしも「Spanner は高い」と言う人を見かけたら、どのような条件下で何と比較してどの SKU が高くなるという話なのかを確認してみましょう。

## 余談

それにしても Firestore は Spanner よりも高くなりすぎるのではないか、という方は Cloud Next 2025 で発表され現在プレビューの Firestore Enterprise edition についても確認してみてください。

:::message
現在 Firestore Enterprise edition では MongoDB compatibility のみが提供されているため MongoDB compatibility 専用のように見えますが、
[Firestore editions overview](https://cloud.google.com/firestore/native/docs/editions-overview) に書かれている通り、将来的に Native mode も提供される予定です。
:::

[Firestore Enterprise edition pricing](https://cloud.google.com/firestore/enterprise/pricing) によると、 Enterprise edition は Standard edition よりも大規模ユーザ向けの課金体系になっているようです。

|                        | Standard edition             | Enterprise edition          | Ratio(Enterprise/Standard) |
|------------------------|------------------------------|-----------------------------|----------------------------|
| Read Units             | $0.038 per 100,000 documents | $0.0642 per 1,000,000 units | 16.9 %                     |
| Write Units            | $0.115 per 100,000 documents | $0.3339 per 1,000,000 units | 29 %                       |
| Managed Delete Units   | $0.013 per 100,000 documents | $0.3339 per 1,000,000 units | 257 %                      |
| Stored Data            | $0.115/GiB/month             | $0.312/GiB/month            | 271 %                      |
| Backup data            | $0.038/GiB/month             | $0.038/GiB/month            | 100 %                      |
| Restore operation data | $0.256/GiB/month             | $0.256/GiB/month            | 100 %                      |

オペレーションの単位が 100K から 1M と一桁大きくなっていることに注目してください。これにより、桁を合わせると Read/Write は安くなるケースがあることがわかります。

なお、Units とドキュメントはそのまま対応するものではなく、サイズが Read は 4 KiB, Write は 1 KiB を超えていればその分だけ繰り上げで課金されるようです。

> * Read Units, representing the data processed (documents or indexes) when you read data from your database, calculated in 4 KiB tranches.
> * Write Units, representing the data processed when you write data into your database, calculated in 1 KiB tranches.

小さいドキュメントであれば安くなりますが、それぞれ 24KB(Read), 4KB(Write) を超えるサイズでは Standard edition よりも高くなってしまうようです。
頻繁にアクセスするドキュメントはなんでも詰め込んだ巨大なドキュメントではなく、最低限の情報のみを持たせるというようなベストプラクティスが生まれるかもしれません。

しかし、この記事で計算してきたことを読み返しても賢く使ったとしてもこの課金体系だと Spanner よりも一定以上高くなる傾向があることはわかります。

スモールスタートできる選択肢を選んだ人が規模が大きくなって後で後悔するというようなことはあまり好ましくはありません。
Firestore のスモールスタートが可能なことなどの使いやすさと Spanner のコスト効率をトレードオフではなく両立させ、大規模なワークロードへスケールした後のことを懸念しなくても良いようになることが好ましいでしょう。
例えば Firestore のまま Spanner と同様のプロビジョニング課金モデルに移行できる選択肢が Firestore にも将来的に提供されることなどを期待したいですね。
