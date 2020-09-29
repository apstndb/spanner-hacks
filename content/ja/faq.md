---
title: "FAQ"
linkTitle: "FAQ"
weight: 5
type: docs
---

## ノードとサーバーはどう違う？

Cloud Spanner におけるノードはプロビジョニングとキャパシティの単位であり、1ノードあたりの価格や性能が定義されている。
冗長化のために複数リージョン・ゾーンを跨ぐことになる Cloud Spanner インスタンスにおいて、ノードは実体というよりは水平分散を抽象化した概念であると考えられる。
ノード数(node count)の設定に比例して各ゾーンに起動する処理タスク(serving task)が実際のスプリットのレプリカの読み書きの処理を行う実体となる。この処理タスクは VM よりは Kubernetes での Pod のように Borg 上にコンテナとして起動するものであるとイメージすると良い。
この個々の処理タスクがドキュメント上のサーバーに対応している。 

例えば3つの read/write replica 用のゾーンからなるシングルリージョン構成でノード数が3のインスタンスがある時、合計で9つの処理タスクが構成される。
よって、主キーを使った単一行の読み取りのような1つのサーバーで完結する処理は全体の 1/9 の処理タスクのみの負荷になると考えられる。

備考

事実としては Google の公式の情報でもノードとサーバーの語を明確に区別しているとは言えないが、実行計画の文脈ではノードの語は実行計画ツリーのノード(plan node)たる operator を指すことがあるため、サーバーもしくは処理タスク(serving task)と呼ぶ方が曖昧ではない利点がある。

例: [Query execution operators](https://cloud.google.com/spanner/docs/query-execution-operators?hl=en#cross-apply)
> The top-level node is a distributed union operator.

参考

* 公式ドキュメント
    * [Instances](https://cloud.google.com/spanner/docs/instances?hl=en)
* セッション
    * [エンタープライズ DB に求められる高可用性と Cloud Spanner](https://www.youtube.com/watch?v=YREHKnkdY7Y)

## Root Server って何？

ドキュメント中にはルートサーバー(root server)という名称が現れるが、これはコンパイル済の実行計画を受け取ってクエリ全体の実行に責任を持つ処理タスクであり、データに近い(アクセスするスプリットのレプリカを持つ)ものからクエリごとに選定される。
つまり、ルートサーバーという固定の役割があるわけではなく、レプリカを管理している処理タスクのうちの一つがそのクエリにおいて役割を持つにすぎない。

クエリは1つの処理タスクが持つレプリカだけでは完結しないため、ルートサーバーは Distributed Operator 時に実行計画のサブツリー(subplan)をクエリ対象のレプリカを持っている他のサーバー(リモートサーバー) にリモートコールで渡すことで、クエリを分散実行する役割を追う。
実行計画ツリーのルート(プランノード ID が0)を処理することによってリモートコール関係のツリー構造におけるルートになるという2つのツリー構造をイメージすると良い。

Distributed Union をはじめとする Distributed Operator はほぼ全てのクエリに含まれているが、ルートサーバー自身が持っているレプリカに対してはローカルで処理を続けることができるため、リモートサーバーへのリモートコールが発生しない場合がある。

なお、クエリ対象のスプリットのレプリカを持つリモートサーバーを見つけるための述語は公式ドキュメントにおいて [Split Predicate](https://cloud.google.com/spanner/docs/query-execution-operators?hl=en#distributed_operators) として言及されており、実際の内容は spanner-cli などで Split Range として確認できる。

```sql
SELECT s.SongName, s.SongGenre
FROM Songs AS s
WHERE s.SingerId = 2 AND s.SongGenre = 'ROCK';
```
```
+----+---------------------------------------+
| ID | Query_Execution_Plan (EXPERIMENTAL)   |
+----+---------------------------------------+
| *0 | Distributed Union                     |
|  1 | +- Local Distributed Union            |
|  2 |    +- Serialize Result                |
| *3 |       +- FilterScan                   |
|  4 |          +- Table Scan (Table: Songs) |
+----+---------------------------------------+
Predicates(identified by ID):
 0: Split Range: ($SingerId = 2)
 3: Seek Condition: ($SingerId = 2)
    Residual Condition: ($SongGenre = 'ROCK')
```

この場合、 Songs テーブルのうち `SingerId = 2` を主キーの一部として持つ処理タスク以外で Distributed Union のサブツリーを実行する必要はなく、更にルートサーバーはこの範囲を持っているものから選ばれることが多いため、リモートコール自体が発生することは少ないことになる。
なお一般的には任意の範囲は複数のスプリットに分割されるが、公式のサンプルスキーマでは Songs テーブルは SingerId を主キーとする Singers テーブルにインターリーブされており複数のスプリットに分かれることがないため、複数のサーバーを跨ぐ必要もなくなる。

参考

* https://cloud.google.com/spanner/docs/query-execution-plans?hl=en#life_of_a_query
* https://cloud.google.com/spanner/docs/whitepapers/life-of-query?hl=en#query_execution

## Hash Join と Apply Join の違いは？

(例等は TODO)

Hash Join は Build 側からのの入力全てからなるハッシュテーブルを作ってから、 Probe 側からの入力をハッシュテーブルに通すことで JOIN を行う。
Apply Join は Input 側の入力から取り出した値を使って Map 側のサブツリーを実行することで JOIN を行う。

Hash Join は先に Probe 側を Scan して一つのサーバーに集めてハッシュテーブルを構築しないといけないため分散不可能なハッシュテーブル構築のコストが掛かり、最初の行を返すまでのレイテンシも高くなる。
一方で Apply Join は Input 側から入力はストリームのまま小さい単位で処理できるため、最初の行を返すまでのレイテンシも低くできる余地がある。

Apply Join では Input 側の入力のそれぞれに対して Map 側のサブツリーを実行するため、 JOIN の条件として FilterScan の Scan Condition が使える際には効率が良くなるが、そうではない場合には大幅に効率が悪くなる。
Hash Join では Batch 側の入力と Probe 側のフィルタは無関係なため JOIN のためのインデックスを気にする必要がないため場合によってはベーステーブルを使って back join を減らすことができるが、逆に Input 側の選択性に限らず Probe 側をスキャンする必要があるためスキャン範囲が広くなる場合が多い。

クエリオプティマイザはベーステーブルやセカンダリインデックスを使って Apply Join を効率的に実行できると確信した時以外に Hash Join を選択する傾向がある。
しかし、OLTP でのユースケースでは Hash Join の方が向いている処理はかなり少なくなると考えられるため、まずは Apply Join を指定して、 Apply Join を高速化する手段を検討することが好ましい。

JOIN ヒントである `@{JOIN_METHOD=APPLY_JOIN}` を個別の `JOIN` の直後もしくはクエリ文の先頭に書くことで Apply Join を指定できる。
(Hash Join だけは `JOIN` の代わりに `HASH JOIN` と書くことでも指定できるが、 `APPLY JOIN` とは書くことができない)

例: JOIN 個別のヒント
```sql
SELECT *
FROM Singers s JOIN@{JOIN_METHOD=APPLY_JOIN} Albums AS a
ON a.SingerId = a.SingerId
```

例: 文全体のヒント
```sql
@{JOIN_METHOD=APPLY_JOIN}
SELECT *
FROM Singers s JOIN Albums AS a
ON a.SingerId = a.SingerId
```

`IN` サブクエリや `EXISTS` サブクエリも内部的に JOIN として処理されるが JOIN としてはヒントが書けないため、処理方法を固定したい場合には文全体のヒントを指定すると良い。

参考

* https://cloud.google.com/spanner/docs/sql-best-practices?hl=en#write_efficient_queries_for_joins
* https://cloud.google.com/spanner/docs/query-syntax?hl=en#join-methods
* https://cloud.google.com/spanner/docs/query-execution-operators?hl=en

## テーブルやセカンダリインデックスはどのような構造になっているのか？

典型的な SQL DBMS では追記に適したヒープ構造によって行のデータを保存するテーブルと、キーを探索するための読み込みに適した冗長な構造であるインデックスは違う非対称な構造だった。SQL では部分一致やソートなど範囲を扱うことが多く、B+Tree インデックスが最も一般的に使われた。
なお、後にクラスタ化インデックス(clustered index)や索引構成表(index-organized table) として、主キーに対する B+Tree 構造にテーブルのデータを保存することができるようになったため対称性がある場合も多い。

これに対して、 Cloud Spanner では B+Tree は使わず、キーでソート済の行の集合を分割したスプリットの集まりとしてテーブルもセカンダリインデックスも保存されていると説明される。
スプリットは更に SSTable もしくは Ressi を構成要素とする LSM ツリー構造として、インスタンスを構成する各ゾーンごとにレプリカが分散ストレージ Colossus に保存される。
詳しくは [Schema and data model](https://cloud.google.com/spanner/docs/schema-and-data-model?hl=en), [Spanner: Google's Globally-Distributed Database](https://research.google/pubs/pub39966/), [Spanner: Becoming a SQL System](https://research.google/pubs/pub46103/) などを参照すると良い。

高レベルではテーブルとセカンダリインデックスは下記のような特徴を持つ。

* テーブルとセカンダリインデックスのデータはキーの範囲アクセスが効率的に行える構造で同様に保存される。
  * 論理的なテーブルと区別する目的でセカンダリインデックスと同格な物理的なテーブルをベーステーブルと呼ぶ。
* 探索に使えるキー列だけでなく非キー列も保持している。
  * ベーステーブルにおいては主キーがキー列、その他の列が非キー列となる。
  * セカンダリインデックスにおいてはセカンダリインデックスのキーとテーブルの主キーを合わせたものが主キー、 STORING した列が非キー列となる。
    * SQL で保持していない列を使おうとした場合は、暗黙の JOIN([back join](https://cloud.google.com/spanner/docs/query-execution-plans?hl=en#index_and_back_join_queries)) が発生する。
    * SQL よりも低レベルな [Read](https://cloud.google.com/spanner/docs/reference/rpc/google.spanner.v1?hl=en#google.spanner.v1.Spanner.Read)/[StreamingRead](https://cloud.google.com/spanner/docs/reference/rpc/google.spanner.v1?hl=en#google.spanner.v1.Spanner.StreamingRead) API では保持していない列を使おうとするとエラーになる。
* キーの範囲で分割することで水平分散するため、単調に増加もしくは減少する日時や連番のキーでは負荷が分散できない。
* インターリーブはテーブルに親子関係を定義することで、ルートテーブルのキー(ルートキー)が同じ親子関係がある行を同じスプリットに保存されるようにして局所性を高める。
  * スプリットを跨ぐ分散コミット及び分散 JOIN を減らすため読み書き両方にメリットがある。

ベーステーブル、セカンダリインデックスのそれぞれの種別ごとにルートキー、キー、非キーの区切りを抽象的に表すと下記のようになる。

![Cloud Spanner key structure](/images/key-structure.png)

## back join を回避するために使用する列をインデックスに含めるには STORING と複合キーどちらが良いか？

一般的に SQL DBMS ではテーブルアクセスしないインデックスオンリースキャンを実現するために必要な列を全て複合キーに含めるカバリングインデックスを定義するという手法がよく使われていたが、
キーのサイズが大きくなるなどのデメリットがあるため、一部の DBMS ではインデックスのキーはそのままでリーフノードに列を追加するINCLUDE 句を実装([参考](https://use-the-index-luke.com/ja/sql/clustering/index-only-scan-covering-index))している。

Cloud Spanner の STORING 句は INCLUDE 句と同様の概念であり、 Seek Condition やソートに使うためにインデックスの複合キーであるべき理由がある場合以外では、インデックスに列を追加したい時は複合キーに含めるのではなく STORING を使うことが第一の選択肢となる。

具体的には STORING ではなく複合キーを使う場合は主に下記のようなデメリットがある。

* キー部16列、8KB の制限に引っかかりやすくなる。
* キー部による順序が変わるため、負荷ベースのスプリット分割で解決できないホットスポットが発生する場合が出てくる。
    * `CREATE INDEX idx ON tbl(Status) STORING(CreatedAt);` はホットスポットにならない。
    * `CREATE INDEX idx ON tbl(Status, CreatedAt);` はホットスポットになる。
* 下記のようにセカンダリインデックスのキー部の末尾にあるテーブルの主キーがセカンダリインデックスの順序に及ぼす影響が減るため、 back join が遅くなる場合がある。

STORING を使った場合のパフォーマンス

```sql
CREATE INDEX SongsBySongGenreStoring ON Songs(SongGenre) STORING (SongName);

SELECT *
FROM Songs@{FORCE_INDEX=SongsBySongGenreStoring}
WHERE SongGenre = "ROCK"
```
```
+-----+---------------------------------------------------------------+---------------+------------+---------------+
| ID  | Query_Execution_Plan                                          | Rows_Returned | Executions | Total_Latency |
+-----+---------------------------------------------------------------+---------------+------------+---------------+
|  *0 | Distributed Union                                             | 256844        | 1          | 2.31 secs     |
|  *1 | +- Distributed Cross Apply                                    | 256844        | 1          | 2.29 secs     |
|   2 |    +- [Input] Create Batch                                    |               |            |               |
|   3 |    |  +- Local Distributed Union                              | 256844        | 1          | 211.73 msecs  |
|   4 |    |     +- Compute Struct                                    | 256844        | 1          | 205.71 msecs  |
|  *5 |    |        +- FilterScan                                     |               |            |               |
|   6 |    |           +- Index Scan (Index: SongsBySongGenreStoring) | 256844        | 1          | 110.57 msecs  |
|  21 |    +- [Map] Serialize Result                                  | 256844        | 13         | 1.18 secs     |
|  22 |       +- Cross Apply                                          | 256844        | 13         | 1.12 secs     |
|  23 |          +- [Input] Batch Scan (Batch: $v2)                   | 256844        | 13         | 163.27 msecs  |
|  28 |          +- [Map] Local Distributed Union                     | 256844        | 256844     | 848.85 msecs  |
| *29 |             +- FilterScan                                     | 256844        | 256844     | 726.78 msecs  |
|  30 |                +- Table Scan (Table: Songs)                   | 256844        | 256844     | 572.16 msecs  |
+-----+---------------------------------------------------------------+---------------+------------+---------------+
Predicates(identified by ID):
  0: Split Range: ($SongGenre = 'ROCK')
  1: Split Range: ($SingerId' = $SingerId)
  5: Seek Condition: IS_NOT_DISTINCT_FROM($SongGenre, 'ROCK')
 29: Seek Condition: (($SingerId' = $batched_SingerId) AND ($AlbumId' = $batched_AlbumId)) AND ($TrackId' = $batched_TrackId)

256844 rows in set (2.42 secs)
timestamp: 2020-09-28T15:55:37.042813+09:00
cpu:       2.23 secs
scanned:   513688 rows
optimizer: 2
```

STORING が使える場面で複合キーを使った場合のパフォーマンス

```sql
CREATE INDEX SongsBySongGenreSongName ON Songs(SongGenre, SongName);

SELECT *
FROM Songs@{FORCE_INDEX=SongsBySongGenreSongName}
WHERE SongGenre = "ROCK"
```
```
+-----+----------------------------------------------------------------+---------------+------------+---------------+
| ID  | Query_Execution_Plan                                           | Rows_Returned | Executions | Total_Latency |
+-----+----------------------------------------------------------------+---------------+------------+---------------+
|  *0 | Distributed Union                                              | 256844        | 1          | 9.1 secs      |
|  *1 | +- Distributed Cross Apply                                     | 256844        | 1          | 9.06 secs     |
|   2 |    +- [Input] Create Batch                                     |               |            |               |
|   3 |    |  +- Local Distributed Union                               | 256844        | 1          | 231.91 msecs  |
|   4 |    |     +- Compute Struct                                     | 256844        | 1          | 213.86 msecs  |
|  *5 |    |        +- FilterScan                                      |               |            |               |
|   6 |    |           +- Index Scan (Index: SongsBySongGenreSongName) | 256844        | 1          | 111.2 msecs   |
|  21 |    +- [Map] Serialize Result                                   | 256844        | 13         | 7.55 secs     |
|  22 |       +- Cross Apply                                           | 256844        | 13         | 7.43 secs     |
|  23 |          +- [Input] Batch Scan (Batch: $v2)                    | 256844        | 13         | 253.43 msecs  |
|  28 |          +- [Map] Local Distributed Union                      | 256844        | 256844     | 7.06 secs     |
| *29 |             +- FilterScan                                      | 256844        | 256844     | 6.9 secs      |
|  30 |                +- Table Scan (Table: Songs)                    | 256844        | 256844     | 6.68 secs     |
+-----+----------------------------------------------------------------+---------------+------------+---------------+
Predicates(identified by ID):
  0: Split Range: ($SongGenre = 'ROCK')
  1: Split Range: ($SingerId' = $SingerId)
  5: Seek Condition: IS_NOT_DISTINCT_FROM($SongGenre, 'ROCK')
 29: Seek Condition: (($SingerId' = $batched_SingerId) AND ($AlbumId' = $batched_AlbumId)) AND ($TrackId' = $batched_TrackId)

256844 rows in set (9.25 secs)
timestamp: 2020-09-28T15:54:14.304421+09:00
cpu:       7.22 secs
scanned:   513688 rows
optimizer: 2
```


上の結果は SongGenre のようなカーディナリティの低い列をセカンダリインデックスのキーの末尾に指定した場合、
SongGenre が同じ値の行の中では暗黙にキー部末尾に含まれる主キーの順序でソートされることになり back join が比較的高速に行えるため、数倍の速度差が出ていると考えられる。

上の例は恣意的に感じるかもしれないが、論理削除フラグやステータスのような BOOL や ENUM 的な列に対してセカンダリインデックスを貼った場合には同様の事象が起こりうる。

## STORING するかどうかの基準は？

STORING のユースケースは back join を完全になくすことが主に説明されるが、一般的にはクエリの中の色々なパターンで back join を遅延させて減らすことができる。
構造体やオブジェクトにマッピングする目的などでテーブルに列を増やすと SELECT に指定する列も増えるようなクエリで back join を完全になくそうとするとテーブルに列を増やすたびにセカンダリインデックスのメンテナンスが必要になる。
よって、そのメンテナンスコストを払っても余りあるような効果がある場合以外ではあまり推奨はできない。

しかし、完全に back join をなくせないとしても低いメンテナンスコストで十分なパフォーマンス改善が得られるケースがある。

* `SELECT` に指定する列が固定のクエリ
* `IN` サブクエリ, `EXISTS` サブクエリに現れる列
* `GROUP BY` の集約関数で使う列
* `JOIN` 時に外部キーとして使う列
* Seek Condition としては使えないが Residual Condition として使える列

STORING 自体の書き込みコストも考慮する必要があるが、行全体を更新する際に書き込む必要があるセカンダリインデックスの数は変わらないため、
性能へのインパクトが大きいロックや分散トランザクションの数、更新の偏りはセカンダリインデックスを増やすと増えるが STORING の数では変わらない。

よって、書き込むデータ量が列の大きさ分増えるトレードオフに見合うだけのクエリの性能向上があるのであれば、積極的に STORING の利用を検討したい。

(TODO: STORING に関するベンチマーク)

## INTERLEAVE

(TODO: あとで書く)

## 低レベル API(Read) と SQL レベル API (ExecuteSql) の違いは？

低レベル API の Read にできることと SQL の実行計画オペレータの対応関係は下記のようになる。

* `table` で指定した論理テーブルのベーステーブルもしくは `index` で指定したセカンダリインデックスに含まれる列(`columns`)からなる行をスキャンする。
  * Scan
  * (セカンダリインデックスに含まれない列を取得することができないため、暗黙の JOIN は発生しない。)
* スキャンを指定した複数のキー範囲(`key_set`)に限定する。
  * FilterScan の Seek Condition
  * (Residual Condition 相当は不可能)
* ソート済のスキャン対象から `limit` の数だけ行を読み出し
  * Local Limit
  * (ソート順を指定できないため Sort Limit は不可能)
* 範囲がスプリットを跨ぐ場合、各スプリットを持つサーバーで取得した結果を1つのストリームとして得る。
  * Split Range 付きの Distributed Union + 
* スプリット、サーバーを跨ぐ最終的な結果も `limit` の数に収まるようにする。
  * Global Limit
  * (ソート順を指定できないため Sort Limit は不可能)
* 結果としてのデータ形式に変換する
  * Serialize Result
  
つまり、下記の形状の実行計画が `Read` API でできるもっとも複雑な処理と同等となる。

```
> EXPLAIN SELECT FirstName, LastName, SingerId FROM Singers@{FORCE_INDEX=SingersByFirstLastName} WHERE FirstName LIKE 'A%' LIMIT 10;
+----+--------------------------------------------------------------+
| ID | Query_Execution_Plan (EXPERIMENTAL)                          |
+----+--------------------------------------------------------------+
|  0 | Global Limit                                                 |
| *1 | +- Distributed Union                                         |
|  2 |    +- Serialize Result                                       |
|  3 |       +- Local Limit                                         |
|  4 |          +- Local Distributed Union                          |
| *5 |             +- FilterScan                                    |
|  6 |                +- Index Scan (Index: SingersByFirstLastName) |
+----+--------------------------------------------------------------+
Predicates(identified by ID):
 1: Split Range: STARTS_WITH($FirstName, 'A')
 5: Seek Condition: STARTS_WITH($FirstName, 'A')
```

これよりも複雑なことをサーバサイドで行いたい場合は SQL で実行する必要があり、それに応じたコストを払うことになる。
なお、 Read は SQL のように想定外のコストが高い実行計画が選ばれる危険がないため予測可能性が高いが、
クライアント側で処理できるようにデータを取得することがサーバサイドで処理するよりも Cloud Spanner インスタンスへの負荷が低いとは限らない。

例えば `GROUP BY` については、 Stream Aggregate であれば Scan に対して追加のコストはかなり少ない。

下記の `GROUP BY` を使ったクエリはキーの順序に対して効率的に処理できるため Stream Aggregate となり、あまりコストが掛からない。
Local Stream Aggregate よりも上では集約済の行のみを扱うため処理する量が少なくほぼ時間を使っていない。

```
> EXPLAIN ANALYZE SELECT SongGenre, SUM(Duration) FROM Songs GROUP BY SongGenre;
+----+-----------------------------------------------------------------------------+---------------+------------+---------------+
| ID | Query_Execution_Plan                                                        | Rows_Returned | Executions | Total_Latency |
+----+-----------------------------------------------------------------------------+---------------+------------+---------------+
|  0 | Serialize Result                                                            | 4             | 1          | 313.96 msecs  |
|  1 | +- Global Stream Aggregate                                                  | 4             | 1          | 313.94 msecs  |
|  2 |    +- Distributed Union                                                     | 4             | 1          | 313.93 msecs  |
|  3 |       +- Local Stream Aggregate                                             | 4             | 1          | 313.92 msecs  |
|  4 |          +- Local Distributed Union                                         | 1024000       | 1          | 285.65 msecs  |
|  5 |             +- Index Scan (Full scan: true, Index: SongsBySongGenreStoring) | 1024000       | 1          | 256.71 msecs  |
+----+-----------------------------------------------------------------------------+---------------+------------+---------------+
4 rows in set (318.28 msecs)
timestamp: 2020-09-29T16:26:00.69254+09:00
cpu:       314.88 msecs
scanned:   1024000 rows
optimizer: 2
```

同じ集計をクライアントサイドで行うには全ての行を取得する必要があり、等価な SQL クエリであれば Serialize Result や Distributed Union で時間がかかる結果となる。

```
> EXPLAIN ANALYZE SELECT SongGenre, Duration FROM Songs@{FORCE_INDEX=SongsBySongGenreStoring};
+----+-----------------------------------------------------------------------+---------------+------------+---------------+
| ID | Query_Execution_Plan                                                  | Rows_Returned | Executions | Total_Latency |
+----+-----------------------------------------------------------------------+---------------+------------+---------------+
|  0 | Distributed Union                                                     | 1024000       | 1          | 522.63 msecs  |
|  1 | +- Local Distributed Union                                            | 1024000       | 1          | 477.35 msecs  |
|  2 |    +- Serialize Result                                                | 1024000       | 1          | 453.77 msecs  |
|  3 |       +- Index Scan (Full scan: true, Index: SongsBySongGenreStoring) | 1024000       | 1          | 319.48 msecs  |
+----+-----------------------------------------------------------------------+---------------+------------+---------------+
1024000 rows in set (616.72 msecs)
timestamp: 2020-09-29T16:25:58.768777+09:00
cpu:       602.56 msecs
scanned:   1024000 rows
optimizer: 2
```

このように、集計済の値のみを通信すれば良いサーバサイドでの処理の方が高速かつ Cloud Spanner インスタンスの負荷も低い場合はある。

クエリ実行時の負荷以外にも Cloud Spanner で SQL クエリを実行するには、クエリを解析してオプティマイザによって実行計画を得るオーバーヘッドがあり、これが Read では掛からないという違いがある。
文字列として一致するクエリの実行計画はキャッシュされるため、この差は小さくなる。フィルタ条件などが可変する場合には文字列として同じにならない動的 SQL ではなく可能な限りクエリパラメータを使うと良い。

参考
* https://cloud.google.com/spanner/docs/whitepapers/life-of-query?hl=en
