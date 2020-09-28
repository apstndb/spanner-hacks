---
title: "Seek Condition と Residual Condition の見分け方"
linkTitle: "Seek Condition と Residual Condition の見分け方"
weight: 2
type: docs
---

Cloud Spanner は WHERE 句(述語)を適用するのに3つの方法を使い分ける。

* FilterScan operator の Seek Condition
  * Seek Condition はスキャンするキー範囲の開始と終了の条件を表す。
  * Use The Index, Luke や SQL パフォーマンス詳解におけるアクセス述語に対応する。
* FilterScan operator の Residual Condition
  * Residual Condition はキー範囲をスキャンしながら適用される。スキャンされる範囲の開始と終了の条件を表したり、キー範囲を狭めるものとしては扱われない。
  * Use The Index, Luke や SQL パフォーマンス詳解におけるフィルタ述語に対応する。
* Filter operator の Condition
  * Scan を伴わずにインメモリで入力からフィルタを行う。これもフィルタ述語の一種と考えらる。

Seek Condition だけがキーを使ってスキャンする範囲を絞ることができるため、データ量が増えてもスケールさせたいクエリでは主要なフィルタが FilterScan の Seek Condition になっていることの確認を推奨する。

## Seek Condition と Residual Condition の例

```
SELECT SingerId, AlbumId, TrackId
FROM Songs@{FORCE_INDEX=SongsBySongName}
WHERE SongName LIKE "A%";
```
例として、 `Songs` テーブルから `SongName` が A からはじまる行を取得するためのクエリを見ていく。
このクエリの WHERE 句に対応するフィルタは `SongsBySongName` インデックスに対する範囲のスキャンで表現できるため、適切なインデックスがあれば次のように Seek Condition として処理できる。
最終的な結果の行数とスキャンされる行数は一致するため、インデックスが最大限有効に使われていることが確認できる。

```
SELECT SingerId, AlbumId, TrackId
FROM Songs@{FORCE_INDEX=SongsBySongName}
WHERE SongName LIKE "A%";
```
```
+----+-------------------------------------------------+---------------+------------+---------------+
| ID | Query_Execution_Plan                            | Rows_Returned | Executions | Total_Latency |
+----+-------------------------------------------------+---------------+------------+---------------+
| *0 | Distributed Union                               | 16949         | 1          | 47.57 msecs   |
|  1 | +- Local Distributed Union                      | 16949         | 1          | 13.91 msecs   |
|  2 |    +- Serialize Result                          | 16949         | 1          | 13.52 msecs   |
| *3 |       +- FilterScan                             | 16949         | 1          | 10.75 msecs   |
|  4 |          +- Index Scan (Index: SongsBySongName) | 16949         | 1          | 8.71 msecs    |
+----+-------------------------------------------------+---------------+------------+---------------+
Predicates(identified by ID):
 0: Split Range: STARTS_WITH($SongName, 'A')
 3: Seek Condition: STARTS_WITH($SongName, 'A')

16949 rows in set (52.75 msecs)
timestamp: 2020-09-23T18:21:54.790063+09:00
cpu:       50.12 msecs
scanned:   16949 rows
optimizer: 2
```

上と同じく SongsBySongName インデックスから最初の文字が `A` のものを取得するクエリであっても、下記のクエリは Seek Condition で処理できずに Residual Condition となってしまい範囲を限定できない。
よってフルスキャンとなり、各 operator の `Rows_Returned` は同じでも `scanned` として表示されているスキャン対象の行数が大幅に異なることがわかる。

```sql
SELECT SingerId, AlbumId, TrackId
FROM Songs@{FORCE_INDEX=SongsBySongName}
WHERE SUBSTR(SongName, 0, 1) = "A";
```
```
+----+------------------------------------------------------------------+---------------+------------+---------------+
| ID | Query_Execution_Plan                                             | Rows_Returned | Executions | Total_Latency |
+----+------------------------------------------------------------------+---------------+------------+---------------+
| *0 | Distributed Union                                                | 16949         | 1          | 263.41 msecs  |
|  1 | +- Local Distributed Union                                       | 16949         | 1          | 262.46 msecs  |
|  2 |    +- Serialize Result                                           | 16949         | 1          | 261.99 msecs  |
| *3 |       +- FilterScan                                              |               |            |               |
|  4 |          +- Index Scan (Full scan: true, Index: SongsBySongName) | 16949         | 1          | 259.61 msecs  |
+----+------------------------------------------------------------------+---------------+------------+---------------+
Predicates(identified by ID):
 0: Split Range: (SUBSTR($SongName, 0, 1) = 'A')
 3: Residual Condition: (SUBSTR($SongName, 0, 1) = 'A')

16949 rows in set (269.45 msecs)
timestamp: 2020-09-23T18:24:29.831979+09:00
cpu:       264.2 msecs
scanned:   1024000 rows
optimizer: 2
```

ここまでで見たように、 Index Scan であることを確認するだけではデータ量が増えるほど計算量が増えてスケールしないクエリを見つけるには不十分であり、
フィルタがどのように処理されているかを確認することが一つの有効な手段となる。

## WHERE 句と back join

長い間 SQL DBMS ではテーブルの行を保存する構造とセカンダリインデックスの構造は異なるものだった。
実行計画上もテーブルスキャンとインデックススキャンは区別され、インデックススキャンで特定した行の列をテーブルから取得することは暗黙的に行われたり、 `TABLE ACCESS` のような専用の operator で処理されるのが一般的だった。

一方、 Cloud Spanner ではベーステーブルとセカンダリインデックスは同じ構造であるため、それぞれからのスキャンは
Scan operator の `scan_type` が `TableScan` か `IndexScan` かのみで区別される。
セカンダリインデックスから行全体を取得するためにも専用の operator は用意されておらず JOIN([back join](https://cloud.google.com/spanner/docs/query-execution-plans?hl=en#index_and_back_join_queries)) そのもので処理される。
つまり、INTERLEAVE されていないインデックスならば Distributed Cross Apply を使った分散 JOIN としてして実行計画に現れることが多い。

テーブルアクセスが JOIN であることはフィルタにも関係があり、セカンダリインデックスに含まれない列を WHERE 句で指定した場合には、
対応するフィルタはセカンダリインデックス側の Scan ではなくベーステーブル側の Scan に対応する FilterScan で処理される。

セカンダリインデックスを使う場合は

* キー部に対する Seek Condition
* インデックスに含まれる列に対する Residual Condition
* back join しないと適用できないベーステーブルへの Residual Condition

の3階層を意識すると良いだろう。

例として、セカンダリインデックスに含まれない列 BirthDate へのフィルタがどのように処理されるかを示す。

```sql
CREATE INDEX SingersByFirstLastName 
ON Singers (
	FirstName,
	LastName
);

SELECT *
FROM Singers@{FORCE_INDEX=SingersByFirstLastName}
WHERE FirstName < "J"
AND EXTRACT(DAYOFWEEK FROM BirthDate) = 1;
```
```
+-----+--------------------------------------------------------------+---------------+------------+---------------+
| ID  | Query_Execution_Plan                                         | Rows_Returned | Executions | Total_Latency |
+-----+--------------------------------------------------------------+---------------+------------+---------------+
|  *0 | Distributed Union                                            | 71            | 1          | 170.72 msecs  |
|  *1 | +- Distributed Cross Apply                                   | 71            | 1          | 170.68 msecs  |
|   2 |    +- [Input] Create Batch                                   |               |            |               |
|   3 |    |  +- Local Distributed Union                             | 503           | 1          | 1.18 msecs    |
|   4 |    |     +- Compute Struct                                   | 503           | 1          | 1.14 msecs    |
|  *5 |    |        +- FilterScan                                    | 503           | 1          | 0.93 msecs    |
|   6 |    |           +- Index Scan (Index: SingersByFirstLastName) | 503           | 1          | 0.89 msecs    |
|  18 |    +- [Map] Serialize Result                                 | 71            | 1          | 167.99 msecs  |
|  19 |       +- Cross Apply                                         | 71            | 1          | 167.85 msecs  |
|  20 |          +- [Input] Batch Scan (Batch: $v2)                  | 503           | 1          | 0.96 msecs    |
|  24 |          +- [Map] Local Distributed Union                    | 71            | 503        | 166.58 msecs  |
| *25 |             +- FilterScan                                    | 71            | 503        | 166.08 msecs  |
|  26 |                +- Table Scan (Table: Singers)                | 71            | 503        | 165.53 msecs  |
+-----+--------------------------------------------------------------+---------------+------------+---------------+
Predicates(identified by ID):
  0: Split Range: ($FirstName < 'J')
  1: Split Range: ($SingerId' = $SingerId)
  5: Seek Condition: ($FirstName < 'J')
 25: Seek Condition: ($SingerId' = $batched_SingerId)
     Residual Condition: (EXTRACT(DAYOFWEEK, $BirthDate) = 1)

71 rows in set (179.02 msecs)
timestamp: 2020-09-23T17:44:58.768598+09:00
cpu:       63.3 msecs
scanned:   1006 rows
optimizer: 2
```

元の SQL 文の WHERE 句には2つの条件が含まれているが、片方の条件で使われている `BirthDate` は使用するセカンダリインデックスには含まれない。
結果として、実行計画では Index Scan に伴う FilterScan(ID:5) では `FirstName < "J"` に対応する `$FirstName < 'J'` のみが Seek Condition として処理されており、
`EXTRACT(DAYOFWEEK FROM BirthDate) = 1` に対応する `EXTRACT(DAYOFWEEK, $BirthDate) = 1` はベーステーブルの Table Scan に伴う FilterScan(ID:25) の Residual Condition として処理されることが読み取れる。
つまり、インデックスではフィルタが完結せずベーステーブルを分散 JOIN しながら結果を捨てることも含めてやっと WHERE 句のフィルタが処理される。
実際に503件分散 JOIN しているにも関わらず71行を残して捨てていることが分かり、これは無駄な処理となる。

次に、意味は全く同じクエリだが、フィルタで使用する `BirthDate` を含んだセカンダリインデックスを作成してヒントで指定した際の実行計画と実行統計を示す。

```sql
CREATE INDEX SingersByFirstLastNameStoring 
ON Singers (
	FirstName,
	LastName
) STORING (
	BirthDate
);

SELECT *
FROM Singers@{FORCE_INDEX=SingersByFirstLastNameStoring}
WHERE FirstName < "J"
AND EXTRACT(DAYOFWEEK FROM BirthDate) = 1;
```
```
+-----+---------------------------------------------------------------------+---------------+------------+---------------+
| ID  | Query_Execution_Plan                                                | Rows_Returned | Executions | Total_Latency |
+-----+---------------------------------------------------------------------+---------------+------------+---------------+
|  *0 | Distributed Union                                                   | 71            | 1          | 5.64 msecs    |
|  *1 | +- Distributed Cross Apply                                          | 71            | 1          | 5.61 msecs    |
|   2 |    +- [Input] Create Batch                                          |               |            |               |
|   3 |    |  +- Local Distributed Union                                    | 71            | 1          | 0.49 msecs    |
|   4 |    |     +- Compute Struct                                          | 71            | 1          | 0.47 msecs    |
|  *5 |    |        +- FilterScan                                           | 71            | 1          | 0.43 msecs    |
|   6 |    |           +- Index Scan (Index: SingersByFirstLastNameStoring) | 71            | 1          | 0.42 msecs    |
|  26 |    +- [Map] Serialize Result                                        | 71            | 1          | 2.07 msecs    |
|  27 |       +- Cross Apply                                                | 71            | 1          | 2.04 msecs    |
|  28 |          +- [Input] Batch Scan (Batch: $v2)                         | 71            | 1          | 0.03 msecs    |
|  33 |          +- [Map] Local Distributed Union                           | 71            | 71         | 1.98 msecs    |
| *34 |             +- FilterScan                                           | 71            | 71         | 1.93 msecs    |
|  35 |                +- Table Scan (Table: Singers)                       | 71            | 71         | 1.88 msecs    |
+-----+---------------------------------------------------------------------+---------------+------------+---------------+
Predicates(identified by ID):
  0: Split Range: ($FirstName < 'J')
  1: Split Range: ($SingerId' = $SingerId)
  5: Seek Condition: ($FirstName < 'J')
     Residual Condition: (EXTRACT(DAYOFWEEK, $BirthDate) = 1)
 34: Seek Condition: ($SingerId' = $batched_SingerId)

71 rows in set (29.22 msecs)
timestamp: 2020-09-23T17:43:57.547539+09:00
cpu:       9.3 msecs
scanned:   574 rows
optimizer: 2
```

`EXTRACT(DAYOFWEEK FROM BirthDate) = 1` が Index Scan に伴う FilterScan の Residual Condition として処理されるようになったため、 Index Scan の時点で最終的に必要な71件まで絞ることができ、フィルタを適用するためだけの分散 JOIN がなくなった。
上の例で分かるように、SELECT も含めて必要な列全てをインデックスに含めることでテーブルアクセスのための back join を完全になくす(いわゆるインデックスオンリースキャンが可能なカバリングインデックス)かどうかだけでなく、
フィルタ一つをインデックスで処理できるようにするだけでも back join の対象を減らす効果があるため多くの選択肢が存在する。

なお、`EXTRACT(DAYOFWEEK FROM BirthDate) = 1` は Seek Condition として処理できないため、このクエリに関しては STORING ではなく複合キーにするメリットは特にない。

## フィルタプッシュダウンと Filter operator

一般的に、 WHERE 句は可能な限りだけ実行計画ツリーの Scan に近い場所のフィルタとして実行することで、各 operator が処理する必要があるデータ量を減らす最適化であるフィルタプッシュダウンが行われる。

FilterScan ではなく Filter が現れるケースは Scan の場所までフィルタプッシュダウンができない理由があるケースである。
下記は[公式ドキュメントの Filter の例](https://cloud.google.com/spanner/docs/query-execution-operators?hl=en#filter) を元にしたクエリだが、サブクエリの中の Singers のスキャンにフィルタを適用すると Limit の結果が意味が変わってしまうため Limit の外側で Filter する必要があり、この実行計画となっている。

```sql
SELECT s.FirstName
FROM (SELECT s.FirstName
      FROM Singers AS s
      LIMIT 50) s
WHERE s.FirstName LIKE 'Z%';
```
```
+----+-------------------------------------------------------------------------------+---------------+------------+---------------+
| ID | Query_Execution_Plan                                                          | Rows_Returned | Executions | Total_Latency |
+----+-------------------------------------------------------------------------------+---------------+------------+---------------+
|  0 | Serialize Result                                                              | 0             | 1          | 0.17 msecs    |
| *1 | +- Filter                                                                     | 0             | 1          | 0.17 msecs    |
|  2 |    +- Global Limit                                                            | 50            | 1          | 0.16 msecs    |
|  3 |       +- Distributed Union                                                    | 50            | 1          | 0.16 msecs    |
|  4 |          +- Local Limit                                                       | 50            | 1          | 0.14 msecs    |
|  5 |             +- Local Distributed Union                                        | 50            | 1          | 0.14 msecs    |
|  6 |                +- Index Scan (Full scan: true, Index: SingersByFirstLastName) | 50            | 1          | 0.13 msecs    |
+----+-------------------------------------------------------------------------------+---------------+------------+---------------+
Predicates(identified by ID):
 1: Condition: STARTS_WITH($FirstName, 'Z')

Empty set (15.41 msecs)
timestamp: 2020-09-24T17:32:23.164311+09:00
cpu:       3.73 msecs
scanned:   50 rows
optimizer: 2
```

この場合、 Filter は Distributed Union よりも外側にあるため root server で処理することになる。フィルタとは関係なく Limit で50行を残して捨てた後で Filter をかけるため、50行の中にどの程度フィルタ条件にマッチする行が残っているかで結果が変わってしまう。
このケースでも最終的な結果が0件となっている。

`LIMIT` 句をサブクエリの外に出すとフィルタプッシュダウンが行われ、 FilterScan で処理されることが確認できる。
この場合はフィルタは全て Scan 対象の split を持つ各 remote server で処理され、root server にはフィルタ済の行だけが集まることになる。

```sql
SELECT s.FirstName
FROM (SELECT s.FirstName
      FROM Singers AS s) s
WHERE s.FirstName LIKE 'Z%';
LIMIT 50
```
```
+----+--------------------------------------------------------------+---------------+------------+---------------+
| ID | Query_Execution_Plan                                         | Rows_Returned | Executions | Total_Latency |
+----+--------------------------------------------------------------+---------------+------------+---------------+
|  0 | Global Limit                                                 | 35            | 1          | 0.94 msecs    |
| *1 | +- Distributed Union                                         | 35            | 1          | 0.94 msecs    |
|  2 |    +- Serialize Result                                       | 35            | 1          | 0.9 msecs     |
|  3 |       +- Local Limit                                         | 35            | 1          | 0.89 msecs    |
|  4 |          +- Local Distributed Union                          | 35            | 1          | 0.89 msecs    |
| *5 |             +- FilterScan                                    | 35            | 1          | 0.88 msecs    |
|  6 |                +- Index Scan (Index: SingersByFirstLastName) | 35            | 1          | 0.87 msecs    |
+----+--------------------------------------------------------------+---------------+------------+---------------+
Predicates(identified by ID):
 1: Split Range: STARTS_WITH($FirstName, 'Z')
 5: Seek Condition: STARTS_WITH($FirstName, 'Z')

35 rows in set (11.79 msecs)
timestamp: 2020-09-24T17:29:09.288925+09:00
cpu:       6.88 msecs
scanned:   35 rows
optimizer: 2
```

サブクエリ内で LIMIT を使うとクエリを字句的にも挙動的にも理解しづらくなり、フィルタプッシュダウンのようなオプティマイザの最適化も妨げる場合があるため、それ以外に選択可能な最適化の手段がある場合はまずはそちらを検討することを推奨する。

## Seek Condition からでは見抜けないケース

### 細切れになりすぎる範囲のコスト

OR や IN などを使った際に Seek Condition は複数に分かれるような範囲も適切に扱うことができる。

```sql
SELECT SingerId, AlbumId, TrackId, SongName
FROM Songs@{FORCE_INDEX=SongsBySongName}
WHERE SongName LIKE "A%" OR SongName LIKE "Z%";
```
```
+----+-------------------------------------------------+---------------+------------+---------------+
| ID | Query_Execution_Plan                            | Rows_Returned | Executions | Total_Latency |
+----+-------------------------------------------------+---------------+------------+---------------+
| *0 | Distributed Union                               | 19767         | 1          | 19.83 msecs   |
|  1 | +- Local Distributed Union                      | 19767         | 1          | 18.56 msecs   |
|  2 |    +- Serialize Result                          | 19767         | 1          | 18.12 msecs   |
| *3 |       +- FilterScan                             | 19767         | 1          | 15.08 msecs   |
|  4 |          +- Index Scan (Index: SongsBySongName) | 19767         | 1          | 12.9 msecs    |
+----+-------------------------------------------------+---------------+------------+---------------+
Predicates(identified by ID):
 0: Split Range: (STARTS_WITH($SongName, 'A') OR STARTS_WITH($SongName, 'Z'))
 3: Seek Condition: (STARTS_WITH($SongName, 'A') OR STARTS_WITH($SongName, 'Z'))

19767 rows in set (48.99 msecs)
timestamp: 2020-09-28T04:24:46.070389+09:00
cpu:       42.35 msecs
scanned:   19767 rows
optimizer: 2
```

更に、複合キーの先頭以外の列を指定すると、細切れになっているはずの範囲でもスキャンすることさえできる。

```sql
SELECT SingerId, AlbumId, TrackId, SongName FROM Songs WHERE AlbumId = 1;
```
```
+----+----------------------------------------------------------------+---------------+------------+---------------+
| ID | Query_Execution_Plan                                           | Rows_Returned | Executions | Total_Latency |
+----+----------------------------------------------------------------+---------------+------------+---------------+
|  0 | Distributed Union                                              | 32000         | 1          | 143.71 msecs  |
|  1 | +- Local Distributed Union                                     | 32000         | 1          | 141.64 msecs  |
|  2 |    +- Serialize Result                                         | 32000         | 1          | 140.87 msecs  |
| *3 |       +- FilterScan                                            | 32000         | 1          | 135.59 msecs  |
|  4 |          +- Index Scan (Index: SongsBySingerAlbumSongNameDesc) | 32000         | 1          | 133.56 msecs  |
+----+----------------------------------------------------------------+---------------+------------+---------------+
Predicates(identified by ID):
 3: Seek Condition: ($AlbumId = 1)

32000 rows in set (155.41 msecs)
timestamp: 2020-09-28T04:27:59.592161+09:00
cpu:       70.9 msecs
scanned:   32000 rows
optimizer: 2
```

しかし、細切れになりすぎると連続した範囲と比べてパフォーマンスが劣化していくことが確認されている。このケースは無駄なスキャンが発生しないため、実行統計からでは認識できない。

### 実際には Seek で表現できない Seek Condition

しかし、2020年9月現在一定以上複雑なクエリでは、全体が Seek Condition と実行計画には書かれているにも関わらず実行時に Residual Condition のようにフィルタする処理となるケースがある。
このようなケースでは実行計画ではフィルタがどう処理されるかを判断できなくなるため、留意する必要がある。

例えば、下記のクエリのフィルタは前方一致に限らない `LIKE` 式なので範囲だけでは実現できず `STARTS_WITH` 相当の前方一致部分をスキャンしながら読み飛ばす必要があるが、全て Seek Condition として表示されてしまっており、実行統計で FilterScan の段階で実行時にフィルタしていることが確認できる。

```sql
SELECT SingerId, AlbumId, TrackId, SongName
FROM Songs@{FORCE_INDEX=SongsBySongName}
WHERE SongName LIKE "A%z" OR SongName LIKE "Z%a";
```
```
+----+-------------------------------------------------+---------------+------------+---------------+
| ID | Query_Execution_Plan                            | Rows_Returned | Executions | Total_Latency |
+----+-------------------------------------------------+---------------+------------+---------------+
| *0 | Distributed Union                               | 116           | 1          | 15.54 msecs   |
|  1 | +- Local Distributed Union                      | 116           | 1          | 15.5 msecs    |
|  2 |    +- Serialize Result                          | 116           | 1          | 15.49 msecs   |
| *3 |       +- FilterScan                             | 116           | 1          | 15.45 msecs   |
|  4 |          +- Index Scan (Index: SongsBySongName) | 19767         | 1          | 10.49 msecs   |
+----+-------------------------------------------------+---------------+------------+---------------+
Predicates(identified by ID):
 0: Split Range: ((STARTS_WITH($SongName, 'A') AND ($SongName LIKE 'A%z')) OR (STARTS_WITH($SongName, 'Z') AND ($SongName LIKE 'Z%a')))
 3: Seek Condition: ((STARTS_WITH($SongName, 'A') AND ($SongName LIKE 'A%z')) OR (STARTS_WITH($SongName, 'Z') AND ($SongName LIKE 'Z%a')))

116 rows in set (78.2 msecs)
timestamp: 2020-09-28T04:26:42.814951+09:00
cpu:       44.01 msecs
scanned:   19767 rows
optimizer: 2
```