概要

* Cloud Spanner は GoogleSQL という Google 独自の SQL 方言を使っている。
    * これは SQL 言語を処理する社外のツールが少ないことを意味する。
    * PostgreSQL 方言もあるけどこの記事では GoogleSQL の話をするよ！
* google-cloud-go には spansql という Cloud Spanner GoogleSQL dialect の parser がある。

## spansql の半生
* テスト目的のインメモリのエミュレータである spannertest を実装するために生まれたものが export されたのが起源
    * https://github.com/googleapis/google-cloud-go/issues/1181
    * https://github.com/googleapis/google-cloud-go/commit/fb3c527ecb61102c3e756ad0638abb5ddc40cf6d
* 少しずつ機能拡張されてはいったが、 spansql は何年経ってもサブクエリすら対応できない不完全な SQL parser の枠を超えることはできなかった。
    * https://github.com/googleapis/google-cloud-go/issues/8519
* spannertest はクライアントライブラリ内での居場所を実インスタンスや Cloud Spanner Emulator を奪われることとなった。
## spansql の今

2022-04-04 時点で Google のメンテナーが PR に対してこのようなコメントをしている。
https://github.com/googleapis/google-cloud-go/issues/5837#issuecomment-1087242796
> We are thinking of freezing the development of spansql/spannertest packages and recommend to use officially supported https://cloud.google.com/spanner/docs/emulator for in-memory usage.

* Google の Go クライアントライブラリチームは spansql の開発を終わらせようとしているが、まだユーザからの PR が送られてきているため、2024年11月現在完全なフリーズはできていない。
    * 主に DDL parser として依存されており PR が送られてくる。
        * yo v2, wrench,
        * golang-migrate/migrate https://github.com/golang-migrate/migrate/blob/c378583d782e026f472dff657bfd088bf2510038/database/spanner/spanner.go#L16
    * issue/PR は p2, p3 として扱われ重要視されることはあまりない。
        * https://github.com/googleapis/google-cloud-go/issues/9622#issuecomment-2039053282
    * issue を上げてもオーナーである Google のチームが修正することはまずない。
        * おそらく Go クライアントライブラリチームの PR による修正は2022年9月のこれが最後
            * https://github.com/googleapis/google-cloud-go/pull/6621
    * PR すら長い間放置されることがある。
        * https://github.com/googleapis/google-cloud-go/pull/7820 (4ヶ月後にやっとマージ)
        * https://github.com/googleapis/google-cloud-go/pull/10121 (放置されて6ヶ月経過)
        * OSS で一番大事なのはいざとなったら自己救済できること。コントリビューションして修正することすら満足にできないものにビジネスが依存するのは大きなリスク。
* 現状は三方悪しである。
    * Google のクライアントライブラリチームはメンテナンスしたくもないものの PR をレビューしてマージしてリリースしないといけない。
    * spansql に依存する関連ツールの開発者は自分たちのツールの問題を直すために spansql を新しい構文に対応させる PR を投げて長期間放置されて板挟みになるという悪い体験がある。
    * 関連ツールの利用者は関連ツールに issue を上げても新しい構文に対応されないので、 Cloud Spanner の新しい機能の利用を断念することになる。
        * 例: https://speakerdeck.com/sgash708/spansql-de-enum-woshi-itakatutahua
        * これはコミュニティに深刻な悪影響がある。
* 競合たち
    * ZetaSQL
        * Cloud Spanner だけでなく BigQuery や Google 社内のシステムでも使われている本物の GoogleSQL フロントエンド。
            * つまり、クエリ、 DML、式などはこれ以上なく正しい実装になっている。
            * 重要な事実として、 Spanner の DDL は ZetaSQL ではない
                * Cloud Spanner Emulator や Java の DDL parser 実装の紹介
        * 基本的にプロダクトに先行しているが、公開が遅れている機能がある気がする。(Spanner GQL サポートはここにあるのでは？)
        * goccy/go-zetasql という実装が BigQuery emulator に使われている。すごい！
    * memefish
        * Mercari インターンシップに参加した MakeNowJust 氏によって開発、のちに cloudspannerecosystem org に移管。
        * 当初は analyzer を含んでいたが、メンテナンスし続けるのは難しいので現在は parser のみで開発が進んでいる。
          * https://github.com/cloudspannerecosystem/memefish/issues/54
* 競合まとめ
    * どれも
    * 一長一短が今の状況を作り出している
## 私ごととして

* 前職に居た頃に wrench の問題を解決したかったが、解決を断念したまま色々あってバーンアウトした
    * https://github.com/cloudspannerecosystem/wrench/issues/83
    * https://github.com/cloudspannerecosystem/wrench/pull/84
* 目が醒めた後で何も状況がよくなっていなくてがっかりした
* 問題を解決するために memefish にコミットするぞ！
* とりあえず一長一短を解決して他の選択肢を圧倒することが解決策になると考え、やった。
    * 全ての spansql のテストが memefish で通るようにした。
        * https://github.com/cloudspannerecosystem/memefish/issues/115
    * 未マージを含めれば全ての種類の DDL を実装した。(Spanner Graph の `CREATE PROPERTY GRAPH` 含む)
        * https://github.com/cloudspannerecosystem/memefish/issues/178
* 本格的に spansql を終わらせるための活動を開始した ← 今ココ
## opinion

* spansql を延命させずに終わらせてあげよう。
    * 新しい機能を使うために spansql にコントリビュートするのはもはや良い選択ではない。
* spansql を新規で採用するのはやめよう。
    * ライブラリ起因で失敗すると分かっているソフトウェアを新しく作りたくはないでしょう。
* spansql から他の選択肢へ移行しよう。
* memefish
    * 本当にそれは parser が必要ですか？ Lexer だけでも実現できないですか？
        * [apstndb/spanner-mycli](https://github.com/apstndb/spanner-mycli) では [apstndb/gsqlutils](https://github.com/apstndb/gsqlutils) 内に memefish.Lexer で実装したステートメント分割とステートメント種別判別のロジックで需要を満たせています。
        * 本当に parser が必要なのは yo のように一つ一つの文の中の要素を解釈する必要があるもの。
    * もしも memefish に非互換性を見つけたら issue を上げよう。
        * こちらは自分で修正しなくてもコントリビューターが直してくれるはず！
    * DDL 対応が不要なくて BigQuery と両対応したいなどの要件なら ZetaSQL(goccy/go-zetasql) も選択肢。