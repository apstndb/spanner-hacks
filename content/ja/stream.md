---
title: "ストリーム処理と LIMIT"
linkTitle: "ストリーム処理と LIMIT"
weight: 4
type: docs
---

Cloud Spanner の実行計画を構成するそれぞれの relational operator はストリームを入力しストリームを出力するストリーム処理をするようになっている。

この挙動が大きく影響する例として、 LIMIT 句を使ったクエリの実行計画は必要な行数が集まった時点で処理を打ち切ることで、早期に結果を返すことができる。

また、クライアントライブラリを使う限りクエリの実行は [ExecuteStreamingSql](https://cloud.google.com/spanner/docs/reference/rest/v1/projects.instances.databases.sessions/executeStreamingSql?hl=en) RPC が使われるため、ストリーム処理の恩恵を受ける。
全ての出力を返すまでの時間だけでなく最初の出力を返すまでの時間を最適化し、クライアントも含めてストリーム処理的に行うことで直列なレイテンシが減る。

しかし、いくつかの operator は全体を処理しないと結果が返せないため、ストリーム処理を妨げる。例えば `LIMIT 1` を設定したとしても殆ど実行時間が変わらないということが発生する。
このような場合は全件処理するために CPU やメモリなどのリソース使用量も増えることが多いため、把握して可能な限り避ける必要がある。

## ORDER BY

無差別な `ORDER BY` は `Sort`/`Sort Limit` operator によって処理されるが、ソートは対象全てが集まらないとできないのでストリーム処理を妨げる。
他の SQL DBMS と同様主キーやセカンダリインデックスはソート済なので、その順序をそのまま使える場合はソートを省略する最適化が行われる。
必要な行数集まった時点で処理を打ち切ることでスキャンが減らせるだけでなく一般的にソートは計算量が小さくはないので、 OLTP では可能な限りソート処理は発生しないようにしたい。

下記のように主キーとは異なる順序でテーブルをソートする場合は Sort Limit operator となるため `LIMIT` があっても全件をスキャンする。

```
SELECT *
FROM Songs@{FORCE_INDEX=_BASE_TABLE}
ORDER BY SongName
LIMIT 1;
```
```
+----+-----------------------------------------------------------+---------------+------------+---------------+
| ID | Query_Execution_Plan                                      | Rows_Returned | Executions | Total_Latency |
+----+-----------------------------------------------------------+---------------+------------+---------------+
|  0 | Serialize Result                                          | 1             | 1          | 686.99 msecs  |
|  1 | +- Global Sort Limit                                      | 1             | 1          | 686.99 msecs  |
|  2 |    +- Distributed Union                                   | 1             | 1          | 686.97 msecs  |
|  3 |       +- Local Sort Limit                                 | 1             | 1          | 686.95 msecs  |
|  4 |          +- Local Distributed Union                       | 1024000       | 1          | 641.48 msecs  |
|  5 |             +- Table Scan (Full scan: true, Table: Songs) | 1024000       | 1          | 611.86 msecs  |
+----+-----------------------------------------------------------+---------------+------------+---------------+
1 rows in set (715.3 msecs)
timestamp: 2020-09-24T16:03:54.747227+09:00
cpu:       708.51 msecs
scanned:   1024000 rows
optimizer: 2
```

下記のようにテーブルの主キーの全部もしくは一部をソート順に指定した場合はスキャンした時点でソート済のストリームに対する Limit operator となるため最低限の件数しか読まないことが確認できる。

```sql
SELECT *
FROM Songs@{FORCE_INDEX=_BASE_TABLE}
ORDER BY SingerId, AlbumId, TrackId
LIMIT 1;
```
```
+----+-----------------------------------------------------------+---------------+------------+---------------+
| ID | Query_Execution_Plan                                      | Rows_Returned | Executions | Total_Latency |
+----+-----------------------------------------------------------+---------------+------------+---------------+
|  0 | Global Limit                                              | 1             | 1          | 0.26 msecs    |
|  1 | +- Distributed Union                                      | 1             | 1          | 0.26 msecs    |
|  2 |    +- Serialize Result                                    | 1             | 1          | 0.25 msecs    |
|  3 |       +- Local Limit                                      | 1             | 1          | 0.25 msecs    |
|  4 |          +- Local Distributed Union                       | 1             | 1          | 0.25 msecs    |
|  5 |             +- Table Scan (Full scan: true, Table: Songs) | 1             | 1          | 0.22 msecs    |
+----+-----------------------------------------------------------+---------------+------------+---------------+
1 rows in set (5.69 msecs)
timestamp: 2020-09-24T16:04:43.444081+09:00
cpu:       5.11 msecs
scanned:   1 rows
optimizer: 2
```

集約キーの prefix のみがキーと一致する場合は Minor Sort として処理される。ソート済ではない部分ごとにソートが必要になるため LIMIT で指定したよりも多い行を処理する必要がるが、全体は処理しないで済んでいることが分かる。

```sql
SELECT *
FROM Songs@{FORCE_INDEX=_BASE_TABLE}
ORDER BY SingerId, SongName
LIMIT 1;
```
```
+----+-----------------------------------------------------------+---------------+------------+---------------+
| ID | Query_Execution_Plan                                      | Rows_Returned | Executions | Total_Latency |
+----+-----------------------------------------------------------+---------------+------------+---------------+
|  0 | Global Limit                                              | 1             | 1          | 5.45 msecs    |
|  1 | +- Distributed Union                                      | 1             | 1          | 5.45 msecs    |
|  2 |    +- Serialize Result                                    | 1             | 1          | 5.44 msecs    |
|  3 |       +- Local Minor Sort Limit                           | 1             | 1          | 5.43 msecs    |
|  4 |          +- Local Distributed Union                       | 1025          | 1          | 5.35 msecs    |
|  5 |             +- Table Scan (Full scan: true, Table: Songs) | 1025          | 1          | 5.31 msecs    |
+----+-----------------------------------------------------------+---------------+------------+---------------+
1 rows in set (17.76 msecs)
timestamp: 2020-09-24T16:52:35.712095+09:00
cpu:       6.87 msecs
scanned:   1025 rows
optimizer: 2
```

セカンダリインデックスに関しても同様で、キーと一致する順序でソートする場合、 `LIMIT` があれば最低限の件数しか読まない。

```sql
SELECT *
FROM Songs@{FORCE_INDEX=SongsBySongName}
ORDER BY SongName
LIMIT 1;
```
```
+-----+---------------------------------------------------------------------------+---------------+------------+---------------+
| ID  | Query_Execution_Plan                                                      | Rows_Returned | Executions | Total_Latency |
+-----+---------------------------------------------------------------------------+---------------+------------+---------------+
|   0 | Global Limit                                                              | 1             | 1          | 5.82 msecs    |
|   1 | +- Distributed Union                                                      | 1             | 1          | 5.82 msecs    |
|  *2 |    +- Distributed Cross Apply                                             | 1             | 1          | 5.81 msecs    |
|   3 |       +- [Input] Create Batch                                             |               |            |               |
|   4 |       |  +- Compute Struct                                                | 1             | 1          | 4.47 msecs    |
|   5 |       |     +- Local Limit                                                | 1             | 1          | 4.47 msecs    |
|   6 |       |        +- Local Distributed Union                                 | 1             | 1          | 4.47 msecs    |
|   7 |       |           +- Index Scan (Full scan: true, Index: SongsBySongName) | 1             | 1          | 4.46 msecs    |
|  19 |       +- [Map] Serialize Result                                           | 1             | 1          | 1.29 msecs    |
|  20 |          +- Cross Apply                                                   | 1             | 1          | 1.29 msecs    |
|  21 |             +- [Input] Batch Scan (Batch: $v2)                            | 1             | 1          | 0 msecs       |
|  27 |             +- [Map] Local Distributed Union                              | 1             | 1          | 1.28 msecs    |
| *28 |                +- FilterScan                                              | 1             | 1          | 1.28 msecs    |
|  29 |                   +- Table Scan (Table: Songs)                            | 1             | 1          | 1.28 msecs    |
+-----+---------------------------------------------------------------------------+---------------+------------+---------------+
Predicates(identified by ID):
  2: Split Range: ($SingerId' = $SingerId)
 28: Seek Condition: (($SingerId' = $batched_SingerId) AND ($AlbumId' = $batched_AlbumId)) AND ($TrackId' = $batched_TrackId)

1 rows in set (14.63 msecs)
timestamp: 2020-09-24T16:06:56.738064+09:00
cpu:       8.59 msecs
scanned:   2 rows
optimizer: 2
```

多くの SQL DBMS との違いとして逆順でシークすることはできないため、スキャン対象とは逆順のソートを指定した場合には Limit ではなく Sort Limit が必要になる。
下記の場合だと `SongName DESC` なインデックスが別途必要になる。
```
SELECT *
FROM Songs@{FORCE_INDEX=SongsBySongName}
ORDER BY SongName DESC
LIMIT 1;
```
```
+-----+------------------------------------------------------------------------------+---------------+------------+---------------+
| ID  | Query_Execution_Plan                                                         | Rows_Returned | Executions | Total_Latency |
+-----+------------------------------------------------------------------------------+---------------+------------+---------------+
|   0 | Serialize Result                                                             | 1             | 1          | 493.48 msecs  |
|   1 | +- Global Sort Limit                                                         | 1             | 1          | 493.47 msecs  |
|   2 |    +- Distributed Union                                                      | 1             | 1          | 493.46 msecs  |
|  *3 |       +- Distributed Cross Apply                                             | 1             | 1          | 493.45 msecs  |
|   4 |          +- [Input] Create Batch                                             |               |            |               |
|   5 |          |  +- Compute Struct                                                | 1             | 1          | 489.3 msecs   |
|   6 |          |     +- Local Sort Limit                                           | 1             | 1          | 489.29 msecs  |
|   7 |          |        +- Local Distributed Union                                 | 1024000       | 1          | 389.67 msecs  |
|   8 |          |           +- Index Scan (Full scan: true, Index: SongsBySongName) | 1024000       | 1          | 357.24 msecs  |
|  23 |          +- [Map] Cross Apply                                                | 1             | 1          | 4.09 msecs    |
|  24 |             +- [Input] Batch Scan (Batch: $v2)                               | 1             | 1          | 0 msecs       |
|  29 |             +- [Map] Local Distributed Union                                 | 1             | 1          | 4.09 msecs    |
| *30 |                +- FilterScan                                                 | 1             | 1          | 4.07 msecs    |
|  31 |                   +- Table Scan (Table: Songs)                               | 1             | 1          | 4.07 msecs    |
+-----+------------------------------------------------------------------------------+---------------+------------+---------------+
Predicates(identified by ID):
  3: Split Range: ($SingerId' = $sort_SingerId)
 30: Seek Condition: (($SingerId' = $batched_SingerId) AND ($AlbumId' = $batched_AlbumId)) AND ($TrackId' = $batched_TrackId)

1 rows in set (504.21 msecs)
timestamp: 2020-09-25T14:44:14.47525+09:00
cpu:       439.83 msecs
scanned:   1024001 rows
optimizer: 2
```


## GROUP BY

スキャンする対象のキーと一致する集約キーを指定した場合は Stream Aggregate operator となる。ストリーム処理されるため `LIMIT` で最低限の行数だけ処理できる。

```sql
SELECT s.SingerId, AVG(s.duration) AS average, COUNT(*) AS count
FROM Songs AS s
GROUP BY SingerId
LIMIT 1;
```
```
+----+--------------------------------------------------------------+---------------+------------+---------------+
| ID | Query_Execution_Plan                                         | Rows_Returned | Executions | Total_Latency |
+----+--------------------------------------------------------------+---------------+------------+---------------+
|  0 | Global Limit                                                 | 1             | 1          | 7.15 msecs    |
|  1 | +- Distributed Union                                         | 1             | 1          | 7.15 msecs    |
|  2 |    +- Serialize Result                                       | 1             | 1          | 7.14 msecs    |
|  3 |       +- Local Limit                                         | 1             | 1          | 7.14 msecs    |
|  4 |          +- Stream Aggregate                                 | 1             | 1          | 7.14 msecs    |
|  5 |             +- Local Distributed Union                       | 1025          | 1          | 7.08 msecs    |
|  6 |                +- Table Scan (Full scan: true, Table: Songs) | 1025          | 1          | 7.04 msecs    |
+----+--------------------------------------------------------------+---------------+------------+---------------+
1 rows in set (11.23 msecs)
timestamp: 2020-09-24T16:44:34.598988+09:00
cpu:       4.3 msecs
scanned:   1025 rows
optimizer: 2
```

スキャン対象のキーと一致しない集約キーを指定した場合は Hash Aggregate operator となる。全体を処理しないと値が確定しないためストリーム処理が妨げられ、 `LIMIT` を指定しても打ち切ることができない。

```sql
SELECT s.SongGenre, AVG(s.duration) AS average, COUNT(*) AS count
FROM Songs AS s
GROUP BY SongGenre
LIMIT 1;
```
```
+----+--------------------------------------------------------------+---------------+------------+---------------+
| ID | Query_Execution_Plan                                         | Rows_Returned | Executions | Total_Latency |
+----+--------------------------------------------------------------+---------------+------------+---------------+
|  0 | Serialize Result                                             | 1             | 1          | 695.95 msecs  |
|  1 | +- Global Limit                                              | 1             | 1          | 695.95 msecs  |
|  2 |    +- Global Hash Aggregate                                  | 1             | 1          | 695.95 msecs  |
|  3 |       +- Distributed Union                                   | 4             | 1          | 695.93 msecs  |
|  4 |          +- Local Hash Aggregate                             | 4             | 1          | 695.92 msecs  |
|  5 |             +- Local Distributed Union                       | 1024000       | 1          | 567.99 msecs  |
|  6 |                +- Table Scan (Full scan: true, Table: Songs) | 1024000       | 1          | 533 msecs     |
+----+--------------------------------------------------------------+---------------+------------+---------------+
1 rows in set (723.98 msecs)
timestamp: 2020-09-24T16:45:09.758451+09:00
cpu:       675.08 msecs
scanned:   1024000 rows
optimizer: 2
```

## JOIN

JOIN については打ち切れる箇所が比較的複雑になる。
Hash Join は Batch 側については全てを読んでハッシュテーブルを構築しないと Probe 側と JOIN できずストリーム処理を妨げることがある。
Hash Join を使わなければならない場合はスキャン対象の行が少なくなる見込みの方が Probe になるように `FORCE_JOIN_ORDER` ヒントなどで固定するなどの最適化は考えられる。

例: Hash Join が Build 全体を処理する様子

```sql
SELECT a.AlbumTitle, s.SongName
FROM Albums AS a
JOIN@{JOIN_METHOD=HASH_JOIN} Songs AS s USING(SingerId, AlbumId)
LIMIT 1;
```
```
+-----+---------------------------------------------------------------------------------------+---------------+------------+---------------+
| ID  | Query_Execution_Plan                                                                  | Rows_Returned | Executions | Total_Latency |
+-----+---------------------------------------------------------------------------------------+---------------+------------+---------------+
|   0 | Global Limit                                                                          | 1             | 1          | 33.4 msecs    |
|   1 | +- Distributed Union                                                                  | 1             | 1          | 33.4 msecs    |
|   2 |    +- Serialize Result                                                                | 1             | 1          | 33.39 msecs   |
|   3 |       +- Local Limit                                                                  | 1             | 1          | 33.38 msecs   |
|   4 |          +- Hash Join (join_type: INNER)                                              | 1             | 1          | 33.38 msecs   |
|   5 |             +- [Build] Local Distributed Union                                        | 32000         | 1          | 21.54 msecs   |
|   6 |             |  +- Table Scan (Full scan: true, Table: Albums)                         | 32000         | 1          | 19.89 msecs   |
|  10 |             +- [Probe] Local Distributed Union                                        | 1             | 1          | 0.17 msecs    |
|  11 |                +- Index Scan (Full scan: true, Index: SongsBySingerAlbumSongNameDesc) | 1             | 1          | 0.16 msecs    |
+-----+---------------------------------------------------------------------------------------+---------------+------------+---------------+
1 rows in set (39.91 msecs)
timestamp: 2020-09-24T16:11:54.793956+09:00
cpu:       38.84 msecs
scanned:   32001 rows
optimizer: 2
```

Cross Apply は任意の箇所で処理を打ち切ることができるため、 `LIMIT` があると最低限の行のみを処理する。

```sql
SELECT a.AlbumTitle, s.SongName
FROM Albums AS a
JOIN@{JOIN_METHOD=APPLY_JOIN} Songs AS s USING(SingerId, AlbumId)
LIMIT 1;
```
```
+-----+----------------------------------------------------------------------------+---------------+------------+---------------+
| ID  | Query_Execution_Plan                                                       | Rows_Returned | Executions | Total_Latency |
+-----+----------------------------------------------------------------------------+---------------+------------+---------------+
|   0 | Global Limit                                                               | 1             | 1          | 5.7 msecs     |
|   1 | +- Distributed Union                                                       | 1             | 1          | 5.7 msecs     |
|   2 |    +- Serialize Result                                                     | 1             | 1          | 5.69 msecs    |
|   3 |       +- Local Limit                                                       | 1             | 1          | 5.68 msecs    |
|   4 |          +- Local Distributed Union                                        | 1             | 1          | 5.68 msecs    |
|   5 |             +- Cross Apply                                                 | 1             | 1          | 5.68 msecs    |
|   6 |                +- [Input] Table Scan (Full scan: true, Table: Albums)      | 1             | 1          | 5.52 msecs    |
|  10 |                +- [Map] Local Distributed Union                            | 1             | 1          | 0.16 msecs    |
| *11 |                   +- FilterScan                                            |               |            |               |
|  12 |                      +- Index Scan (Index: SongsBySingerAlbumSongNameDesc) | 1             | 1          | 0.15 msecs    |
+-----+----------------------------------------------------------------------------+---------------+------------+---------------+
Predicates(identified by ID):
 11: Seek Condition: (($SingerId_1 = $SingerId) AND ($AlbumId_1 = $AlbumId))

1 rows in set (36 msecs)
timestamp: 2020-09-24T16:18:48.911127+09:00
cpu:       30.54 msecs
scanned:   2 rows
optimizer: 2
```

Distributed Cross Apply はリモートコールを減らすために Input 側に Create Batch が含まれるため、LIMIT では処理するデータ量が一定のバッチサイズよりも減らないことが多い。
下記のクエリの場合は20000行のバッチサイズ分の back join まで Limit で止まるまでに既に処理しているため、多少無駄な処理が多い。
```sql
SELECT c.ConcertDate, c.BeginTime, s.FirstName, s.LastName
FROM Concerts@{FORCE_INDEX=ConcertsByConcertDate} c
JOIN@{JOIN_METHOD=APPLY_JOIN} Singers s USING(SingerId)
WHERE ConcertDate BETWEEN DATE "2020-08-01" AND DATE "2020-12-31"
LIMIT 1;
```
```
+-----+----------------------------------------------------------------------------+---------------+------------+---------------+
| ID  | Query_Execution_Plan                                                       | Rows_Returned | Executions | Total_Latency |
+-----+----------------------------------------------------------------------------+---------------+------------+---------------+
|   0 | Global Limit                                                               | 1             | 1          | 262.77 msecs  |
|  *1 | +- Distributed Union                                                       | 1             | 1          | 262.77 msecs  |
|   2 |    +- Local Limit                                                          | 1             | 1          | 262.75 msecs  |
|  *3 |       +- Distributed Cross Apply                                           | 1             | 1          | 262.75 msecs  |
|   4 |          +- [Input] Create Batch                                           |               |            |               |
|   5 |          |  +- Compute Struct                                              | 20000         | 1          | 215.38 msecs  |
|  *6 |          |     +- Distributed Cross Apply                                  | 20000         | 1          | 209.5 msecs   |
|   7 |          |        +- [Input] Create Batch                                  |               |            |               |
|   8 |          |        |  +- Local Distributed Union                            | 20000         | 1          | 18.62 msecs   |
|   9 |          |        |     +- Compute Struct                                  | 20000         | 1          | 17.5 msecs    |
| *10 |          |        |        +- FilterScan                                   | 20000         | 1          | 13.08 msecs   |
|  11 |          |        |           +- Index Scan (Index: ConcertsByConcertDate) | 20000         | 1          | 10.26 msecs   |
|  27 |          |        +- [Map] Cross Apply                                     | 20000         | 1          | 104.61 msecs  |
|  28 |          |           +- [Input] Batch Scan (Batch: $v2)                    | 20000         | 1          | 4.14 msecs    |
|  32 |          |           +- [Map] Local Distributed Union                      | 20000         | 20000      | 93.95 msecs   |
| *33 |          |              +- FilterScan                                      | 20000         | 20000      | 83.85 msecs   |
|  34 |          |                 +- Table Scan (Table: Concerts)                 | 20000         | 20000      | 70.5 msecs    |
|  64 |          +- [Map] Serialize Result                                         | 1             | 1          | 0.22 msecs    |
|  65 |             +- Local Limit                                                 | 1             | 1          | 0.21 msecs    |
|  66 |                +- Cross Apply                                              | 1             | 1          | 0.21 msecs    |
|  67 |                   +- [Input] Batch Scan (Batch: $v5)                       | 1             | 1          | 0.01 msecs    |
|  71 |                   +- [Map] Local Distributed Union                         | 1             | 1          | 0.2 msecs     |
| *72 |                      +- FilterScan                                         | 1             | 1          | 0.19 msecs    |
|  73 |                         +- Table Scan (Table: Singers)                     | 1             | 1          | 0.19 msecs    |
+-----+----------------------------------------------------------------------------+---------------+------------+---------------+
Predicates(identified by ID):
  1: Split Range: (($ConcertDate >= 18475 unix days (2020-08-01)) AND ($ConcertDate <= 18627 unix days (2020-12-31)))
  3: Split Range: ($SingerId_1 = $batched_SingerId)
  6: Split Range: (($SingerId' = $SingerId) AND ($Concerts_key_VenueId' = $Concerts_key_VenueId) AND ($ConcertDate' = $ConcertDate))
 10: Seek Condition: (($ConcertDate >= 18475 unix days (2020-08-01)) AND ($ConcertDate <= 18627 unix days (2020-12-31)))
 33: Seek Condition: (($Concerts_key_VenueId' = $batched_Concerts_key_VenueId) AND ($SingerId' = $batched_SingerId)) AND ($ConcertDate' = $batched_ConcertDate)
 72: Seek Condition: ($SingerId_1 = $batched_SingerId')

1 rows in set (272.98 msecs)
timestamp: 2020-09-25T12:00:25.985841+09:00
cpu:       269.13 msecs
scanned:   40001 rows
optimizer: 2
```

単なるスキャンよりも back join の方が負荷が大きいため、 STORING を使うなどして back join をなくすことができると Create Batch の負荷は大幅に減る。
完全に back join をなくすことが現実的ではない場合は、下のクエリのように back join と等価な主キーを使った JOIN を明示的に行って back join を最後に行うようにすると、
back join 相当の Distributed Cross Apply の Input 側の ID:5 で既に Local Limit が効いているため back join の負荷が軽減される。
クエリを複雑にするデメリットがあるが、クエリ結果は変わらないのが長所である。

```sql
@{FORCE_JOIN_ORDER=TRUE,JOIN_METHOD=APPLY_JOIN}
SELECT c.ConcertDate, c.BeginTime, s.FirstName, s.LastName
FROM Concerts@{FORCE_INDEX=ConcertsByConcertDate}
JOIN Singers s USING(SingerId)
JOIN Concerts c USING(VenueId, SingerId, ConcertDate)
WHERE ConcertDate BETWEEN DATE "2020-08-01" AND DATE "2020-12-31"
LIMIT 1;
```
```
+-----+----------------------------------------------------------------------------+---------------+------------+---------------+
| ID  | Query_Execution_Plan                                                       | Rows_Returned | Executions | Total_Latency |
+-----+----------------------------------------------------------------------------+---------------+------------+---------------+
|   0 | Global Limit                                                               | 1             | 1          | 120.47 msecs  |
|  *1 | +- Distributed Union                                                       | 1             | 1          | 120.47 msecs  |
|  *2 |    +- Distributed Cross Apply                                              | 1             | 1          | 120.44 msecs  |
|   3 |       +- [Input] Create Batch                                              |               |            |               |
|   4 |       |  +- Compute Struct                                                 | 1             | 1          | 118.03 msecs  |
|   5 |       |     +- Local Limit                                                 | 1             | 1          | 118.03 msecs  |
|  *6 |       |        +- Distributed Cross Apply                                  | 1             | 1          | 118.03 msecs  |
|   7 |       |           +- [Input] Create Batch                                  |               |            |               |
|   8 |       |           |  +- Local Distributed Union                            | 20000         | 1          | 16.85 msecs   |
|   9 |       |           |     +- Compute Struct                                  | 20000         | 1          | 15.69 msecs   |
| *10 |       |           |        +- FilterScan                                   | 20000         | 1          | 10.86 msecs   |
|  11 |       |           |           +- Index Scan (Index: ConcertsByConcertDate) | 20000         | 1          | 7.85 msecs    |
|  27 |       |           +- [Map] Local Limit                                     | 1             | 1          | 7.65 msecs    |
|  28 |       |              +- Cross Apply                                        | 1             | 1          | 7.65 msecs    |
|  29 |       |                 +- [Input] Batch Scan (Batch: $v2)                 | 1             | 1          | 0.01 msecs    |
|  33 |       |                 +- [Map] Local Distributed Union                   | 1             | 1          | 7.64 msecs    |
| *34 |       |                    +- FilterScan                                   | 1             | 1          | 7.62 msecs    |
|  35 |       |                       +- Table Scan (Table: Singers)               | 1             | 1          | 7.62 msecs    |
|  59 |       +- [Map] Serialize Result                                            | 1             | 1          | 2.35 msecs    |
|  60 |          +- Cross Apply                                                    | 1             | 1          | 2.34 msecs    |
|  61 |             +- [Input] Batch Scan (Batch: $v5)                             | 1             | 1          | 0 msecs       |
|  70 |             +- [Map] Local Distributed Union                               | 1             | 1          | 2.34 msecs    |
| *71 |                +- FilterScan                                               | 1             | 1          | 2.32 msecs    |
|  72 |                   +- Table Scan (Table: Concerts)                          | 1             | 1          | 2.32 msecs    |
+-----+----------------------------------------------------------------------------+---------------+------------+---------------+
Predicates(identified by ID):
  1: Split Range: (($ConcertDate >= 18475 unix days (2020-08-01)) AND ($ConcertDate <= 18627 unix days (2020-12-31)))
  2: Split Range: (($VenueId_1 = $batched_VenueId) AND ($SingerId_2 = $batched_SingerId) AND ($ConcertDate_1 = $batched_ConcertDate))
  6: Split Range: ($SingerId_1 = $SingerId)
 10: Seek Condition: (($ConcertDate >= 18475 unix days (2020-08-01)) AND ($ConcertDate <= 18627 unix days (2020-12-31)))
 34: Seek Condition: ($SingerId_1 = $batched_SingerId)
 71: Seek Condition: (($VenueId_1 = $batched_VenueId') AND ($SingerId_2 = $batched_SingerId')) AND ($ConcertDate_1 = $batched_ConcertDate')

1 rows in set (166.44 msecs)
timestamp: 2020-09-25T11:52:39.480888+09:00
cpu:       155.67 msecs
scanned:   20002 rows
optimizer: 2
```

なお、前述のクエリは下記のように `[INNER] JOIN` ではなく `LEFT [OUTER] JOIN` にするだけで Limit が効くようになる。実行計画上は ConcertsByConcertDate の枝の Create Batch の内側に Local Limit があるため Batch を作る前に必要な件数に絞れることが分かる。
```sql
SELECT c.ConcertDate, c.BeginTime, s.FirstName, s.LastName
FROM Concerts@{FORCE_INDEX=ConcertsByConcertDate} c
LEFT JOIN@{JOIN_METHOD=APPLY_JOIN} Singers s USING(SingerId)
WHERE ConcertDate BETWEEN DATE "2020-08-01" AND DATE "2020-12-31"
LIMIT 1;
```
```
+-----+-------------------------------------------------------------------------------+---------------+------------+---------------+
| ID  | Query_Execution_Plan                                                          | Rows_Returned | Executions | Total_Latency |
+-----+-------------------------------------------------------------------------------+---------------+------------+---------------+
|   0 | Global Limit                                                                  | 1             | 1          | 2.93 msecs    |
|  *1 | +- Distributed Union                                                          | 1             | 1          | 2.92 msecs    |
|   2 |    +- Serialize Result                                                        | 1             | 1          | 2.91 msecs    |
|  *3 |       +- Distributed Outer Apply                                              | 1             | 1          | 2.9 msecs     |
|   4 |          +- [Input] Create Batch                                              |               |            |               |
|   5 |          |  +- Compute Struct                                                 | 1             | 1          | 0.93 msecs    |
|  *6 |          |     +- Distributed Cross Apply                                     | 1             | 1          | 0.92 msecs    |
|   7 |          |        +- [Input] Create Batch                                     |               |            |               |
|   8 |          |        |  +- Compute Struct                                        | 1             | 1          | 0.05 msecs    |
|   9 |          |        |     +- Local Limit                                        | 1             | 1          | 0.04 msecs    |
|  10 |          |        |        +- Local Distributed Union                         | 1             | 1          | 0.04 msecs    |
| *11 |          |        |           +- FilterScan                                   | 1             | 1          | 0.04 msecs    |
|  12 |          |        |              +- Index Scan (Index: ConcertsByConcertDate) | 1             | 1          | 0.04 msecs    |
|  26 |          |        +- [Map] Cross Apply                                        | 1             | 1          | 0.85 msecs    |
|  27 |          |           +- [Input] Batch Scan (Batch: $v2)                       | 1             | 1          | 0 msecs       |
|  31 |          |           +- [Map] Local Distributed Union                         | 1             | 1          | 0.85 msecs    |
| *32 |          |              +- FilterScan                                         | 1             | 1          | 0.85 msecs    |
|  33 |          |                 +- Table Scan (Table: Concerts)                    | 1             | 1          | 0.84 msecs    |
|  64 |          +- [Map] Cross Apply                                                 | 1             | 1          | 1.95 msecs    |
|  65 |             +- [Input] Batch Scan (Batch: $v5)                                | 1             | 1          | 0 msecs       |
|  70 |             +- [Map] Local Distributed Union                                  | 1             | 1          | 1.94 msecs    |
| *71 |                +- FilterScan                                                  | 1             | 1          | 1.94 msecs    |
|  72 |                   +- Table Scan (Table: Singers)                              | 1             | 1          | 1.94 msecs    |
+-----+-------------------------------------------------------------------------------+---------------+------------+---------------+
Predicates(identified by ID):
  1: Split Range: BETWEEN($ConcertDate, 18475 unix days (2020-08-01), 18627 unix days (2020-12-31))
  3: Split Range: ($SingerId_1 = $batched_SingerId)
  6: Split Range: (($Concerts_key_VenueId' = $Concerts_key_VenueId) AND ($SingerId' = $SingerId) AND ($ConcertDate' = $ConcertDate))
 11: Seek Condition: BETWEEN($ConcertDate, 18475 unix days (2020-08-01), 18627 unix days (2020-12-31))
 32: Seek Condition: (($Concerts_key_VenueId' = $batched_Concerts_key_VenueId) AND ($SingerId' = $batched_SingerId)) AND ($ConcertDate' = $batched_ConcertDate)
 71: Seek Condition: ($SingerId_1 = $batched_SingerId')

1 rows in set (41.21 msecs)
timestamp: 2020-09-25T15:17:26.46708+09:00
cpu:       37.29 msecs
scanned:   3 rows
optimizer: 2
```

OUTER JOIN では Limit が掛けられるのが何故なのかについては、 INNER JOIN では結合条件に合う列がなかった場合は行数が減ることがあるため、この例だと1行の出力を得るために `ConcertsByConcertDate` からスキャンする十分な行数を見積もることができないが、 OUTER JOIN では減ることがないためたかだか1件スキャンすれば十分だと判断していると考えられる。
このケースでは INNER JOIN ではなく OUTER JOIN を使う方が LIMIT が効きやすいという利点があったが、対応する SingerId が Singers テーブルにない場合にはクエリの結果が変わることに注意する必要がある。

(なお外部キー制約があれば INNER JOIN でも行数が減ることがなくなることがないことは保証され理論上は早期に Limit できると考えられるが、2020年9月25日現在特に効果はない。)