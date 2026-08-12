---
title: "Operators"
linkTitle: "Operators"
weight: 3
type: docs
---

[Query execution operators](https://docs.cloud.google.com/spanner/docs/query-execution-operators) は複数ページに分割されており、以前より多くの operator がドキュメント化されている。一方で metadata やそれぞれの child links については未解説の部分も多いためここにまとめる。
なお、公式ドキュメントにない事柄や実行計画の細部は、間違っていたり今後予告なく変更される可能性がある。

この文書の再現 SQL と実行計画は、特定の schema、データ量、統計情報、optimizer version（オプティマイザーバージョン）、hint の組み合わせで観測した例である。実行計画は、特記がない限り Spanner Omni 2026.r1-beta で出力したもので、spannerplan のデフォルト出力を掲載している。Spanner はコストベース最適化を行うため、optimizer version、統計情報、データ分布が変わると同じ SQL でも違う実行計画になることがあり、今後も同じ結果である保証はない。特に、テーブルを作成した直後など統計情報が存在しない状態では、実質的にルールベースに近い選択になっていたと考えられる例がある。

{{< details summary="再現例で使用したスキーマ" >}}

以下は、この文書の再現例で主に使用したスキーマである。一部の例では、各 details 内に個別の DDL を示している。

```sql
CREATE TABLE Singers (
  SingerId INT64 NOT NULL,
  FirstName STRING(1024),
  LastName STRING(1024),
  BirthDate DATE,
  SingerInfo BYTES(MAX),
  ReleaseDate DATE,
  ModificationTime TIMESTAMP OPTIONS (allow_commit_timestamp = true),
) PRIMARY KEY(SingerId);

CREATE INDEX SingersByFirstLastName ON Singers(FirstName, LastName);
CREATE INDEX SingersByLastName ON Singers(LastName) STORING (FirstName);

CREATE TABLE Albums (
  SingerId INT64 NOT NULL,
  AlbumId INT64 NOT NULL,
  AlbumTitle STRING(MAX),
  MarketingBudget INT64,
  ReleaseDate DATE,
) PRIMARY KEY(SingerId, AlbumId),
  INTERLEAVE IN PARENT Singers ON DELETE CASCADE;

CREATE INDEX AlbumsByAlbumTitle ON Albums(AlbumTitle);
CREATE INDEX AlbumsByAlbumTitle2 ON Albums(AlbumTitle) STORING (MarketingBudget);
CREATE INDEX AlbumsByReleaseDateTitleDesc ON Albums(ReleaseDate, AlbumTitle DESC);

CREATE TABLE Songs (
  SingerId INT64 NOT NULL,
  AlbumId INT64 NOT NULL,
  TrackId INT64 NOT NULL,
  SongName STRING(MAX),
  Duration INT64,
  SongGenre STRING(25),
) PRIMARY KEY(SingerId, AlbumId, TrackId),
  INTERLEAVE IN PARENT Albums ON DELETE CASCADE;

CREATE INDEX SongsBySingerAlbumSongNameDesc ON Songs(SingerId, AlbumId, SongName DESC),
  INTERLEAVE IN Albums;
CREATE INDEX SongsBySongName ON Songs(SongName);

CREATE TABLE Concerts (
  VenueId INT64 NOT NULL,
  SingerId INT64 NOT NULL,
  ConcertDate DATE NOT NULL,
  BeginTime TIMESTAMP,
  EndTime TIMESTAMP,
  TicketPrices ARRAY<INT64>,
) PRIMARY KEY(VenueId, SingerId, ConcertDate);

CREATE TABLE Collaborations (
  SingerId INT64 NOT NULL,
  FeaturingSingerId INT64 NOT NULL,
  AlbumTitle STRING(MAX) NOT NULL,
) PRIMARY KEY(SingerId, FeaturingSingerId, AlbumTitle);

CREATE OR REPLACE PROPERTY GRAPH MusicGraph
  NODE TABLES(
    Singers
      KEY(SingerId)
      LABEL Singers PROPERTIES(
        BirthDate,
        FirstName,
        LastName,
        SingerId,
        SingerInfo)
  )
  EDGE TABLES(
    Collaborations AS CollabWith
      KEY(SingerId, FeaturingSingerId, AlbumTitle)
      SOURCE KEY(SingerId) REFERENCES Singers(SingerId)
      DESTINATION KEY(FeaturingSingerId) REFERENCES Singers(SingerId)
      LABEL CollabWith PROPERTIES(
        AlbumTitle,
        FeaturingSingerId,
        SingerId)
  );
```

DML の再現例では、必要に応じて以下も追加している。

```sql
ALTER TABLE Singers ADD COLUMN Status STRING(1024) DEFAULT ("active");
ALTER TABLE Singers ADD COLUMN LastUpdated TIMESTAMP DEFAULT (PENDING_COMMIT_TIMESTAMP())
  ON UPDATE (PENDING_COMMIT_TIMESTAMP())
  OPTIONS (allow_commit_timestamp = true);
CREATE UNIQUE INDEX UniqueIndex_SingerName ON Singers(FirstName, LastName);

CREATE TABLE AckworthSingers (
  SingerId INT64 NOT NULL,
  FirstName STRING(1024),
  LastName STRING(1024),
  BirthDate DATE,
) PRIMARY KEY(SingerId);

CREATE TABLE Fans (
  FanId STRING(36) DEFAULT (GENERATE_UUID()),
  FirstName STRING(1024),
  LastName STRING(1024),
) PRIMARY KEY(FanId);
```

{{< /details >}}

対象: 実行計画を可視化や解析のために処理するツール作成者や、含まれる情報全てをクエリの理解に役立てたいと考えるユーザ

TODO: Metadata や ChildLinks の表の形式化を進める。

## 実行計画の構造

実行計画の実体は [`google.spanner.v1.QueryPlan`](https://cloud.google.com/spanner/docs/reference/rpc/google.spanner.v1?hl=en#queryplan) であり、各クライアントや Web UI が表示するものは REST API や gRPC API の `ExecuteSql` もしくは `ExecuteStreamingSql` API 経由で `QueryMode` に `PLAN` もしくは `PROFILE` を指定することで取得した `QueryPlan` そのものである。

`QueryPlan` は [`PlanNode`](https://cloud.google.com/spanner/docs/reference/rpc/google.spanner.v1?hl=en#plannode) の集合であり、 `PlanNode` は operator と一対一で対応する。
各 `PlanNode` の動作は `display_name` によって特定できる operator の種類と、 operator の動作を変える `metadata` によって決まり、`child_links` に入力として使う子の operator が列挙されている。

Scalar operator とは `kind` が `SCALAR` の operator である。[Spanner Studio の query plan visualizer](https://docs.cloud.google.com/spanner/docs/tune-query-with-visualizer) は rows を消費して親へ rows を生成する iterator を graph node として表示し、[Spanner CLI](https://docs.cloud.google.com/spanner/docs/spanner-cli-commands) の `EXPLAIN` / `EXPLAIN ANALYZE` も同様に行を返す operator tree を中心に表示する。子に Relational operator を持つ Subquery のみは例外として表示されるが、その他の Scalar operator は、公式ツールでも OSS の spanner-cli、spannerplan の `rendertree`、spannerplanviz でも、一般的な実行計画ツリーの可視化手法では表示されない。

![test](/images/basic-webui.png)

例えば上記の Table Scan operator の実体は [Scan](#scan) operator であり、 `Table Scan: Songs` の部分及び `full scan: true` は metadata からの情報を合わせて表示している。また、デフォルトでは折りたたまれている変数名とスキャン対象の列名の対応関係は全て Scalar operator である。この文書で Scalar operator を扱う場合、tree 表示上の operator 行として現れることを意味するのではなく、QueryPlan の生データを読むことで観測できる raw `PlanNode` としての語彙を説明している箇所がある。

{{< details summary="上記 Table Scan に対応する生の PlanNode の YAML 表現" >}}

```yaml
- childLinks:
  - childIndex: 4
    variable: SingerId
  - childIndex: 5
    variable: AlbumId
  - childIndex: 6
    variable: TrackId
  - childIndex: 7
    variable: SongName
  - childIndex: 8
    variable: Duration
  - childIndex: 9
    variable: SongGenre
  displayName: Scan
  index: 3
  kind: RELATIONAL
  metadata:
    Full scan: 'true'
    scan_target: Songs
    scan_type: TableScan
```

{{< /details >}}

### クエリに書いた JOIN が実行計画に現れないケース (join elimination)

クエリに書いた構文が常に対応する operator として実行計画に現れるとは限らない。代表例として、参照整合性がスキーマで宣言されているテーブル間の INNER JOIN は、結合相手から結合キー以外の列を参照していなければ join operator ごと除去されることがある。次のいずれでも除去を観測した。

- `INTERLEAVE IN PARENT` された子テーブルから見た親テーブルとの結合
- enforced な `FOREIGN KEY` 制約による結合
- `FOREIGN KEY ... NOT ENFORCED`(informational constraint)による結合

NOT ENFORCED でも optimizer は宣言された制約を信頼して join を除去するため、制約に違反したデータが存在するとクエリ結果自体が変わり得る。これは [informational constraint のドキュメント](https://docs.cloud.google.com/spanner/docs/foreign-keys/overview#informational-foreign-keys)が述べる注意点と整合する。

informational constraint を最適化に使うかどうかは [`USE_UNENFORCED_FOREIGN_KEY` statement hint](https://docs.cloud.google.com/spanner/docs/reference/standard-sql/query-syntax#statement-hints)(デフォルト `TRUE`、データベースオプション `use_unenforced_foreign_key_for_query_optimization` をそのステートメントに限り上書き)で制御できる。観測した範囲では:

- `@{USE_UNENFORCED_FOREIGN_KEY=FALSE}` を付けると NOT ENFORCED FK による join の除去は行われなくなる
- enforced FK や INTERLEAVE による除去は `FALSE` でも変わらず行われる(このヒントが制御するのは unenforced FK の利用のみ)
- statement hint 専用であり、join hint の位置(`JOIN@{...}`)に書くと `Unsupported hint` エラーになる

なお、データベースオプション側はこの文書の観測環境では未検証である。Spanner Omni 2026.r1-beta では `ALTER DATABASE ... SET OPTIONS (use_unenforced_foreign_key_for_query_optimization = false)` がメッセージなしの `InvalidArgument` となり(`version_retention_period` など他のオプションは設定できるため、このオプション自体が未対応とみられる)、公式ドキュメントに載っている `SET DATABASE OPTIONS (...)` 構文もパースエラーになる。

join が除去された場合、その join を対象とした `JOIN_METHOD` ヒントは、対象が存在しないためエラーや警告なしに無視される。実行計画の検査では「指定した join method の operator が現れること」を仮定するより、実際に現れた operator の構造を検査する方が頑健である。なお、この除去は OPTIMIZER_VERSION 1〜8 のいずれでも同じ形で観測された。

{{< details summary="join elimination の再現クエリと実行計画" >}}

INTERLEAVE された親テーブルとの結合は、親キー以外を参照していなければ子テーブル側のスキャン 1 つに除去される。ここでは必要な列を全てカバーする `AlbumsByAlbumTitle` の Index Scan だけが残り、join operator は現れない。

```sql
SELECT s.SingerId, a.AlbumTitle
FROM Singers AS s JOIN Albums AS a ON s.SingerId = a.SingerId;
```

```text
+----+-------------------------------------------------------------------------------------+
| ID | Operator                                                                            |
+----+-------------------------------------------------------------------------------------+
|  0 | Distributed Union on AlbumsByAlbumTitle <Row>                                       |
|  1 | +- Local Distributed Union <Row>                                                    |
|  2 |    +- Serialize Result <Row>                                                        |
|  3 |       +- Index Scan on AlbumsByAlbumTitle <Row> (Full scan, scan_method: Automatic) |
+----+-------------------------------------------------------------------------------------+
```

参照整合性の宣言がないテーブル間では、同じ形のクエリでも join は除去されない。

```sql
SELECT c.SingerId, c.VenueId
FROM Singers AS s JOIN Concerts AS c ON s.SingerId = c.SingerId;
```

```text
+-----+---------------------------------------------------------------------------------+
| ID  | Operator                                                                        |
+-----+---------------------------------------------------------------------------------+
|   0 | Distributed Union on Concerts <Row>                                             |
|  *1 | +- Distributed Cross Apply <Row>                                                |
|   2 |    +- [Input] Create Batch <Batch>                                              |
|   3 |    |  +- RowToDataBlock                                                         |
|   4 |    |     +- Local Distributed Union <Row>                                       |
|   5 |    |        +- Table Scan on Concerts <Row> (Full scan, scan_method: Automatic) |
|  10 |    +- [Map] Serialize Result <Row>                                              |
|  11 |       +- Cross Apply <Row>                                                      |
|  12 |          +- [Input] KeyRangeAccumulator <Row>                                   |
|  13 |          |  +- DataBlockToRow                                                   |
|  14 |          |     +- Batch Scan on $v2 <Batch> (scan_method: Batch)                |
|  19 |          +- [Map] Local Distributed Union <Row>                                 |
|  20 |             +- Filter Scan <Row> (seekable_key_size: 0)                         |
| *21 |                +- Table Scan on Singers <Row> (scan_method: Row)                |
+-----+---------------------------------------------------------------------------------+
Predicates(identified by ID):
  1: Split Range: ($SingerId = $SingerId_1)
 21: Seek Condition: ($SingerId = $batched_SingerId_1')
```

FOREIGN KEY 制約でも、enforced / NOT ENFORCED のどちらでも除去される。必要な DDL:

```sql
CREATE TABLE FkSingers (
  SingerId INT64 NOT NULL,
  FirstName STRING(1024),
) PRIMARY KEY(SingerId);

CREATE TABLE FkAlbums (
  SingerId INT64 NOT NULL,
  AlbumId INT64 NOT NULL,
  AlbumTitle STRING(MAX),
  CONSTRAINT FK_FkAlbums_Singer FOREIGN KEY (SingerId) REFERENCES FkSingers (SingerId)
) PRIMARY KEY(SingerId, AlbumId);

CREATE TABLE FkAlbumsNotEnforced (
  SingerId INT64 NOT NULL,
  AlbumId INT64 NOT NULL,
  AlbumTitle STRING(MAX),
  CONSTRAINT FK_FkAlbumsNE_Singer FOREIGN KEY (SingerId) REFERENCES FkSingers (SingerId) NOT ENFORCED
) PRIMARY KEY(SingerId, AlbumId);
```

```sql
SELECT s.SingerId, a.AlbumTitle
FROM FkSingers AS s JOIN FkAlbums AS a ON s.SingerId = a.SingerId;
```

```text
+----+---------------------------------------------------------------------------+
| ID | Operator                                                                  |
+----+---------------------------------------------------------------------------+
|  0 | Distributed Union on FkAlbums <Row>                                       |
|  1 | +- Local Distributed Union <Row>                                          |
|  2 |    +- Serialize Result <Row>                                              |
|  3 |       +- Table Scan on FkAlbums <Row> (Full scan, scan_method: Automatic) |
+----+---------------------------------------------------------------------------+
```

```sql
SELECT s.SingerId, a.AlbumTitle
FROM FkSingers AS s JOIN FkAlbumsNotEnforced AS a ON s.SingerId = a.SingerId;
```

```text
+----+--------------------------------------------------------------------------------------+
| ID | Operator                                                                             |
+----+--------------------------------------------------------------------------------------+
|  0 | Distributed Union on FkAlbumsNotEnforced <Row>                                       |
|  1 | +- Local Distributed Union <Row>                                                     |
|  2 |    +- Serialize Result <Row>                                                         |
|  3 |       +- Table Scan on FkAlbumsNotEnforced <Row> (Full scan, scan_method: Automatic) |
+----+--------------------------------------------------------------------------------------+
```

`@{USE_UNENFORCED_FOREIGN_KEY=FALSE}` を付けると、同じ NOT ENFORCED FK のクエリでも join は除去されない。

```sql
@{USE_UNENFORCED_FOREIGN_KEY=FALSE}
SELECT s.SingerId, a.AlbumTitle
FROM FkSingers AS s JOIN FkAlbumsNotEnforced AS a ON s.SingerId = a.SingerId;
```

```text
+-----+----------------------------------------------------------------------------------+
| ID  | Operator                                                                         |
+-----+----------------------------------------------------------------------------------+
|   0 | Distributed Union on FkSingers <Row>                                             |
|  *1 | +- Distributed Cross Apply <Row>                                                 |
|   2 |    +- [Input] Create Batch <Batch>                                               |
|   3 |    |  +- RowToDataBlock                                                          |
|   4 |    |     +- Local Distributed Union <Row>                                        |
|   5 |    |        +- Table Scan on FkSingers <Row> (Full scan, scan_method: Automatic) |
|   8 |    +- [Map] Serialize Result <Row>                                               |
|   9 |       +- Cross Apply <Row>                                                       |
|  10 |          +- [Input] KeyRangeAccumulator <Row>                                    |
|  11 |          |  +- DataBlockToRow                                                    |
|  12 |          |     +- Batch Scan on $v2 <Batch> (scan_method: Batch)                 |
|  15 |          +- [Map] Local Distributed Union <Row>                                  |
|  16 |             +- Filter Scan <Row> (seekable_key_size: 0)                          |
| *17 |                +- Table Scan on FkAlbumsNotEnforced <Row> (scan_method: Row)     |
+-----+----------------------------------------------------------------------------------+
Predicates(identified by ID):
  1: Split Range: ($SingerId_1 = $SingerId)
 17: Seek Condition: ($SingerId_1 = $batched_SingerId')
```

{{< /details >}}

## Relational operators

`kind: RELATIONAL` なもので、行のストリームを返す operator である。

### Distributed operators

分散実行される operator 群であり、 `subquery_cluster_node` が指す方の子の Relational operator からなる実行計画のサブツリーを `Split Range` の条件を満たす remote server で実行することで、 server を跨ぐ replica から結果を得るという共通点がある。
ただし `call_type: Local` の Distributed Union にも `subquery_cluster_node` が付くことがあるため、root partitionable かどうかの判定などで分散実行の有無を機械的に判断する場合は `subquery_cluster_node` の有無だけでなく `call_type` も確認する必要がある。観測した DML の実行計画では `Apply Mutations` 自体には `subquery_cluster_node` は付かず、その内部の distributed apply / union に付く。

* https://docs.cloud.google.com/spanner/docs/query-operators-distributed

#### Distributed Anti Semi Apply

`NOT EXISTS` などを処理するために分散 Anti Semi Join を行う。Distributed Cross Apply と似た構造を持つ。

* https://docs.cloud.google.com/spanner/docs/query-operators-distributed#distributed-anti-semi-apply

##### Metadata

| key | values | description |
|-----|--------|-------------|
| subquery_cluster_node | | 分散実行する対象の Relation operator の ID |

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL|(Input) | |  | いわゆる駆動表に対応する入力側のサブツリーであり、実際には type を持たないが Web UI やドキュメント等で Input と表示される。通常 Create Batch を持つ。|
|RELATIONAL| Map |  |  | Input 側の値に応じて分散実行されるサブツリーであり、通常 Batch Scan と Cross Apply を含む。|
|SCALAR    | Split Range |  |  | 分散実行する対象の replica をキーから限定するための Function |
|SCALAR    | Batch | Yes | Yes | Input 側の Batch から生成する行の定義? |

{{< details summary="Distributed Anti Semi Apply の再現クエリと実行計画" >}}

```sql
@{JOIN_METHOD=APPLY_JOIN, BATCH_MODE=TRUE}
SELECT s.SingerId, s.FirstName
FROM Singers AS s
WHERE NOT EXISTS (
  SELECT 1
  FROM Albums AS a
  WHERE a.SingerId = s.SingerId
);
```

```text
+-----+--------------------------------------------------------------------------------------------------+
| ID  | Operator                                                                                         |
+-----+--------------------------------------------------------------------------------------------------+
|   0 | Distributed Union on SingersByFirstLastName <Row>                                                |
|   1 | +- Serialize Result <Row>                                                                        |
|  *2 |    +- Distributed Anti Semi Apply <Row>                                                          |
|   3 |       +- [Input] Create Batch <Row>                                                              |
|   4 |       |  +- Local Distributed Union <Row>                                                        |
|   5 |       |     +- Compute Struct <Row>                                                              |
|   6 |       |        +- Index Scan on SingersByFirstLastName <Row> (Full scan, scan_method: Automatic) |
|  13 |       +- [Map] Semi Apply <Row>                                                                  |
|  14 |          +- [Input] KeyRangeAccumulator <Row>                                                    |
|  15 |          |  +- Batch Scan on $v2 <Row> (scan_method: Row)                                        |
|  19 |          +- [Map] Local Distributed Union <Row>                                                  |
|  20 |             +- Filter Scan <Row> (seekable_key_size: 0)                                          |
| *21 |                +- Table Scan on Albums <Row> (scan_method: Row)                                  |
+-----+--------------------------------------------------------------------------------------------------+
Predicates(identified by ID):
  2: Split Range: ($SingerId_1 = $SingerId)
 21: Seek Condition: ($SingerId_1 = $batched_SingerId)
```

{{< /details >}}

#### Distributed Cross Apply

分散 Apply Join を行う。Input 側の Relational operator から取り出した値を使って、対応する Map 側の Relational operator を適切な replica で実行することで分散 JOIN を実現する。

* https://docs.cloud.google.com/spanner/docs/query-operators-distributed#distributed-cross-apply

##### Metadata

| key | values | description |
|-----|--------|-------------|
| subquery_cluster_node | | 分散実行する対象の Relation operator の ID |

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL| (Input) | | | いわゆる駆動表に対応する入力側のサブツリーであり、実際には type を持たないが Web UI やドキュメント等で Input と表示される。通常 Create Batch を持つ。|
|RELATIONAL| Map |  | | Input 側の値に応じて分散実行されるサブツリーであり、通常 Batch Scan と Cross Apply を含む。|
|SCALAR    | Split Range |  | | 分散実行する対象の replica をキーから限定するための Function |

{{< details summary="Distributed Cross Apply の再現クエリと実行計画" >}}

```sql
SELECT s.SongName, s.Duration
FROM Songs@{FORCE_INDEX=SongsBySongName} AS s
WHERE STARTS_WITH(s.SongName, "B");
```

```text
+-----+--------------------------------------------------------------------------+
| ID  | Operator                                                                 |
+-----+--------------------------------------------------------------------------+
|  *0 | Distributed Union on SongsBySongName <Row>                               |
|  *1 | +- Distributed Cross Apply <Row>                                         |
|   2 |    +- [Input] Create Batch <Batch>                                       |
|   3 |    |  +- RowToDataBlock                                                  |
|   4 |    |     +- Local Distributed Union <Row>                                |
|   5 |    |        +- Filter Scan <Row> (seekable_key_size: 1)                  |
|  *6 |    |           +- Index Scan on SongsBySongName <Row> (scan_method: Row) |
|  19 |    +- [Map] Serialize Result <Row>                                       |
|  20 |       +- Cross Apply <Row>                                               |
|  21 |          +- [Input] KeyRangeAccumulator <Row>                            |
|  22 |          |  +- DataBlockToRow                                            |
|  23 |          |     +- Batch Scan on $v2 <Batch> (scan_method: Batch)         |
|  32 |          +- [Map] Local Distributed Union <Row>                          |
|  33 |             +- Filter Scan <Row> (seekable_key_size: 0)                  |
| *34 |                +- Table Scan on Songs <Row> (scan_method: Row)           |
+-----+--------------------------------------------------------------------------+
Predicates(identified by ID):
  0: Split Range: STARTS_WITH($SongName, 'B')
  1: Split Range: (($Songs_key_SingerId' = $Songs_key_SingerId) AND ($Songs_key_AlbumId' = $Songs_key_AlbumId) AND ($Songs_key_TrackId' = $Songs_key_TrackId))
  6: Seek Condition: STARTS_WITH($SongName, 'B')
 34: Seek Condition: (($Songs_key_SingerId' = $batched_Songs_key_SingerId') AND ($Songs_key_AlbumId' = $batched_Songs_key_AlbumId') AND ($Songs_key_TrackId' = $batched_Songs_key_TrackId'))
```

{{< /details >}}

#### Distributed Outer Apply

`LEFT OUTER JOIN` などを処理するために分散 OUTER JOIN を行う。Distributed Cross Apply と似た構造を持つ。

* https://docs.cloud.google.com/spanner/docs/query-operators-distributed#distributed-outer-apply

##### Metadata

| key | values | description |
|-----|--------|-------------|
| subquery_cluster_node | | 分散実行する対象の Relation operator の ID |

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL| (Input) | | | いわゆる駆動表に対応する入力側のサブツリーであり、実際には type を持たないが Web UI やドキュメント等で Input と表示される。通常 Create Batch を持つ。|
|RELATIONAL| Map |  | | Input 側の値に応じて分散実行されるサブツリーであり、通常 Batch Scan と Cross Apply を含む。|
|SCALAR    | Split Range |  | | 分散実行する対象の replica をキーから限定するための Function |
|SCALAR    | Batch | Yes | Yes | 結合条件を満たさなかった時に Input 側の Batch から生成する行の定義 |

{{< details summary="Distributed Outer Apply の再現クエリと実行計画" >}}

```sql
SELECT a.AlbumTitle, s.SongName
FROM Albums AS a
LEFT JOIN@{JOIN_METHOD=APPLY_JOIN, BATCH_MODE=TRUE} Songs AS s
ON a.SingerId = s.SingerId AND a.AlbumId = s.AlbumId;
```

```text
+-----+----------------------------------------------------------------------------------------------+
| ID  | Operator                                                                                     |
+-----+----------------------------------------------------------------------------------------------+
|   0 | Distributed Union on AlbumsByAlbumTitle <Row>                                                |
|   1 | +- Serialize Result <Row>                                                                    |
|  *2 |    +- Distributed Outer Apply <Row>                                                          |
|   3 |       +- [Input] Create Batch <Row>                                                          |
|   4 |       |  +- Local Distributed Union <Row>                                                    |
|   5 |       |     +- Compute Struct <Row>                                                          |
|   6 |       |        +- Index Scan on AlbumsByAlbumTitle <Row> (Full scan, scan_method: Automatic) |
|  15 |       +- [Map] Cross Apply <Row>                                                             |
|  16 |          +- [Input] KeyRangeAccumulator <Row>                                                |
|  17 |          |  +- Batch Scan on $v2 <Row> (scan_method: Row)                                    |
|  22 |          +- [Map] Local Distributed Union <Row>                                              |
|  23 |             +- Filter Scan <Row> (seekable_key_size: 0)                                      |
| *24 |                +- Index Scan on SongsBySingerAlbumSongNameDesc <Row> (scan_method: Row)      |
+-----+----------------------------------------------------------------------------------------------+
Predicates(identified by ID):
  2: Split Range: (($SingerId_1 = $SingerId) AND ($AlbumId_1 = $AlbumId))
 24: Seek Condition: (($SingerId_1 = $batched_SingerId) AND ($AlbumId_1 = $batched_AlbumId))
```

{{< /details >}}

#### Distributed Semi Apply

`EXISTS` などを処理するために分散 Semi Join を行う。Distributed Cross Apply と似た構造を持つ。

* https://docs.cloud.google.com/spanner/docs/query-operators-distributed#distributed-semi-apply

##### Metadata

| key | values | description |
|-----|--------|-------------|
| subquery_cluster_node | | 分散実行する対象の Relation operator の ID |

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL| (Input) | | | いわゆる駆動表に対応する入力側のサブツリーであり、実際には type を持たないが Web UI やドキュメント等で Input と表示される。通常 Create Batch を持つ。|
|RELATIONAL| Map |  | | Input 側の値に応じて分散実行されるサブツリーであり、通常 Batch Scan と Cross Apply を含む。|
|SCALAR    | Split Range |  | | 分散実行する対象の replica をキーから限定するための Function |
|SCALAR    | Batch | Yes | Yes | Input 側の Batch から生成する行の定義? |

{{< details summary="Distributed Semi Apply の再現クエリと実行計画" >}}

```sql
@{JOIN_METHOD=APPLY_JOIN, BATCH_MODE=TRUE}
SELECT s.SingerId, s.FirstName
FROM Singers AS s
WHERE s.SingerId IN (
  SELECT a.SingerId
  FROM Albums AS a
);
```

```text
+-----+--------------------------------------------------------------------------------------------------+
| ID  | Operator                                                                                         |
+-----+--------------------------------------------------------------------------------------------------+
|   0 | Distributed Union on SingersByFirstLastName <Row>                                                |
|   1 | +- Serialize Result <Row>                                                                        |
|  *2 |    +- Distributed Semi Apply <Row>                                                               |
|   3 |       +- [Input] Create Batch <Row>                                                              |
|   4 |       |  +- Local Distributed Union <Row>                                                        |
|   5 |       |     +- Compute Struct <Row>                                                              |
|   6 |       |        +- Index Scan on SingersByFirstLastName <Row> (Full scan, scan_method: Automatic) |
|  13 |       +- [Map] Semi Apply <Row>                                                                  |
|  14 |          +- [Input] KeyRangeAccumulator <Row>                                                    |
|  15 |          |  +- Batch Scan on $v2 <Row> (scan_method: Row)                                        |
|  19 |          +- [Map] Local Distributed Union <Row>                                                  |
|  20 |             +- Filter Scan <Row> (seekable_key_size: 0)                                          |
| *21 |                +- Table Scan on Albums <Row> (scan_method: Row)                                  |
+-----+--------------------------------------------------------------------------------------------------+
Predicates(identified by ID):
  2: Split Range: ($SingerId_1 = $SingerId)
 21: Seek Condition: ($SingerId_1 = $batched_SingerId)
```

{{< /details >}}

#### Push Broadcast Hash Join

`JOIN_METHOD=PUSH_BROADCAST_HASH_JOIN` で現れる分散 hash join 系の operator。
通常 join では `Push Broadcast Hash Join`、outer join では `Push Broadcast Hash Join Outer Apply`、subquery predicate では `Push Broadcast Hash Join Semi Apply` や `Push Broadcast Hash Join Anti Semi Apply` として表示される。
内部に通常の `Hash Join`、`Create Batch`、`RowToDataBlock`、`DataBlockToRow` などが現れるため、descendant に `Hash Join` があることだけで通常の Hash Join と解釈しない方がよい。

* https://docs.cloud.google.com/spanner/docs/query-operators-distributed#push-broadcast-hash-join

##### Metadata

| key | values | description |
|-----|--------|-------------|
| distribution_table | | 分散先を決める table または index の名前。 |
| subquery_cluster_node | | Map 側で分散実行する relation operator の ID。 |
| order_preserving | true, 未指定 | Semi Apply / Anti Semi Apply variant で入力順を保持する場合に観測された。 |

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL| 空 | | | broadcast する入力側のサブツリー。通常 Create Batch を含む。|
|RELATIONAL| Map | | | broadcast された batch と probe 側を使って実行されるサブツリー。通常 Hash Join を含む。|
|SCALAR    | Batch | Yes | Yes | broadcast する batch の値。variant によって 1 個または複数現れる。 |
|SCALAR    | Map | Yes | | Outer Apply variant で Map 側から返す値。 |
|SCALAR    | Split Range | | | 分散実行する対象の replica をキーから限定するための Function |

公式ドキュメントはこの subtree を input side と呼ぶが、この節で確認した通常 join、outer apply、semi apply、anti semi apply の先頭 relational `childLink.type` はすべて空だった。この節に示す raw `QueryPlan` では `Input` type は未観測である。そのため、現時点では `Input または空` と一般化せず、`Input` を持つ実行計画が得られた場合は operator variant と optimizer version を添えて別途記録する。

{{< details summary="Push Broadcast Hash Join 系の再現クエリと実行計画" >}}

Push Broadcast Hash Join:

```sql
SELECT a.AlbumTitle, s.SongName
FROM Albums AS a
JOIN@{JOIN_METHOD=PUSH_BROADCAST_HASH_JOIN} Songs AS s
ON a.SingerId = s.SingerId AND a.AlbumId = s.AlbumId;
```

```text
+-----+-------------------------------------------------------------------------------------------------------+
| ID  | Operator                                                                                              |
+-----+-------------------------------------------------------------------------------------------------------+
|   0 | Distributed Union on AlbumsByAlbumTitle <Row>                                                         |
|  *1 | +- Push Broadcast Hash Join <Row>                                                                     |
|   2 |    +- Create Batch <Batch>                                                                            |
|   3 |    |  +- RowToDataBlock                                                                               |
|   4 |    |     +- Local Distributed Union <Row>                                                             |
|   5 |    |        +- Index Scan on AlbumsByAlbumTitle <Row> (Full scan, scan_method: Automatic)             |
|  12 |    +- [Map] Serialize Result <Row>                                                                    |
| *13 |       +- Hash Join <Row> (join_type: INNER)                                                           |
|  14 |          +- [Build] DataBlockToRow                                                                    |
|  15 |          |  +- Batch Scan on $v2 <Batch> (scan_method: Batch)                                         |
|  22 |          +- [Probe] Local Distributed Union <Row>                                                     |
|  23 |             +- Index Scan on SongsBySingerAlbumSongNameDesc <Row> (Full scan, scan_method: Automatic) |
+-----+-------------------------------------------------------------------------------------------------------+
Predicates(identified by ID):
  1: Split Range: (($SingerId_1 = $SingerId) AND ($AlbumId_1 = $AlbumId))
 13: Condition: (($batched_SingerId' = $SingerId_1) AND ($batched_AlbumId' = $AlbumId_1))
```

Outer Apply:

```sql
SELECT a.AlbumTitle, s.SongName
FROM Albums AS a
LEFT JOIN@{JOIN_METHOD=PUSH_BROADCAST_HASH_JOIN} Songs AS s
ON a.SingerId = s.SingerId AND a.AlbumId = s.AlbumId;
```

```text
+-----+-------------------------------------------------------------------------------------------------------+
| ID  | Operator                                                                                              |
+-----+-------------------------------------------------------------------------------------------------------+
|   0 | Distributed Union on AlbumsByAlbumTitle <Row>                                                         |
|   1 | +- Serialize Result <Row>                                                                             |
|  *2 |    +- Push Broadcast Hash Join Outer Apply <Row>                                                      |
|   3 |       +- Create Batch <Row>                                                                           |
|   4 |       |  +- Local Distributed Union <Row>                                                             |
|   5 |       |     +- Compute Struct <Row>                                                                   |
|   6 |       |        +- Index Scan on AlbumsByAlbumTitle <Row> (Full scan, scan_method: Automatic)          |
| *15 |       +- [Map] Hash Join <Row> (join_type: INNER)                                                     |
|  16 |          +- [Build] Batch Scan on $v2 <Row> (scan_method: Row)                                        |
|  21 |          +- [Probe] Local Distributed Union <Row>                                                     |
|  22 |             +- Index Scan on SongsBySingerAlbumSongNameDesc <Row> (Full scan, scan_method: Automatic) |
+-----+-------------------------------------------------------------------------------------------------------+
Predicates(identified by ID):
  2: Split Range: (($SingerId_1 = $SingerId) AND ($AlbumId_1 = $AlbumId))
 15: Condition: (($batched_SingerId = $SingerId_1) AND ($batched_AlbumId = $AlbumId_1))
```

Semi Apply:

```sql
@{JOIN_METHOD=PUSH_BROADCAST_HASH_JOIN}
SELECT s.SingerId, s.FirstName
FROM Singers AS s
WHERE s.SingerId IN (SELECT a.SingerId FROM Albums AS a);
```

```text
+-----+--------------------------------------------------------------------------------------------------+
| ID  | Operator                                                                                         |
+-----+--------------------------------------------------------------------------------------------------+
|   0 | Distributed Union on SingersByFirstLastName <Row>                                                |
|   1 | +- Serialize Result <Row>                                                                        |
|  *2 |    +- Push Broadcast Hash Join Semi Apply <Row>                                                  |
|   3 |       +- Create Batch <Row>                                                                      |
|   4 |       |  +- Local Distributed Union <Row>                                                        |
|   5 |       |     +- Compute Struct <Row>                                                              |
|   6 |       |        +- Index Scan on SingersByFirstLastName <Row> (Full scan, scan_method: Automatic) |
| *13 |       +- [Map] Hash Join <Row> (join_type: INNER)                                                |
|  14 |          +- [Build] Batch Scan on $v2 <Row> (scan_method: Row)                                   |
|  18 |          +- [Probe] Local Distributed Union <Row>                                                |
|  19 |             +- Table Scan on Albums <Row> (Full scan, scan_method: Automatic)                    |
+-----+--------------------------------------------------------------------------------------------------+
Predicates(identified by ID):
  2: Split Range: ($SingerId_1 = $SingerId)
 13: Condition: ($batched_SingerId = $SingerId_1)
```

Anti Semi Apply:

```sql
@{JOIN_METHOD=PUSH_BROADCAST_HASH_JOIN}
SELECT s.SingerId, s.FirstName
FROM Singers AS s
WHERE NOT EXISTS (SELECT 1 FROM Albums AS a WHERE a.SingerId = s.SingerId);
```

```text
+-----+--------------------------------------------------------------------------------------------------+
| ID  | Operator                                                                                         |
+-----+--------------------------------------------------------------------------------------------------+
|   0 | Distributed Union on SingersByFirstLastName <Row>                                                |
|   1 | +- Serialize Result <Row>                                                                        |
|  *2 |    +- Push Broadcast Hash Join Anti Semi Apply <Row>                                             |
|   3 |       +- Create Batch <Row>                                                                      |
|   4 |       |  +- Local Distributed Union <Row>                                                        |
|   5 |       |     +- Compute Struct <Row>                                                              |
|   6 |       |        +- Index Scan on SingersByFirstLastName <Row> (Full scan, scan_method: Automatic) |
| *13 |       +- [Map] Hash Join <Row> (join_type: INNER)                                                |
|  14 |          +- [Build] Batch Scan on $v2 <Row> (scan_method: Row)                                   |
|  18 |          +- [Probe] Local Distributed Union <Row>                                                |
|  19 |             +- Table Scan on Albums <Row> (Full scan, scan_method: Automatic)                    |
+-----+--------------------------------------------------------------------------------------------------+
Predicates(identified by ID):
  2: Split Range: ($SingerId_1 = $SingerId)
 13: Condition: ($SingerId_1 = $batched_SingerId)
```

{{< /details >}}

#### Distributed Union

各 replica で子の Relation operator を実行し、結果をまとめる。
クエリ対象の replica を他の server(remote server) が持つ場合、remote server を呼び出すため remote call が発生し、 `executionStats` に記録される。
`call_type` が Local なものは、特定の server 内の結果をまとめる。

* https://docs.cloud.google.com/spanner/docs/query-operators-distributed#distributed-union

##### Metadata
| key | values | description |
|-----|--------|-------------|
| call_type | Local, 未指定 ||
| distribution_table | | 分散対象となる table または index の名前。 |
| preserve_subquery_order | true, 未指定 | subquery の順序を保持する場合に `true`。 |
| split_ranges_aligned | true, false, 未指定 | 分散する split range の境界が揃っているかを表す。 |
| subquery_cluster_node | | 分散実行する対象の Relation operator の ID |

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL|  | | | 入力として分散実行されるサブツリー |
|SCALAR    | Split Range |  | | 分散実行する対象の replica をキーから限定するための Function |

{{< details summary="Distributed Union / Scan / Serialize Result の再現クエリと実行計画" >}}

```sql
SELECT s.SongName
FROM Songs AS s;
```

```text
+----+-------------------------------------------------------------------------------------------------+
| ID | Operator                                                                                        |
+----+-------------------------------------------------------------------------------------------------+
|  0 | Distributed Union on SongsBySingerAlbumSongNameDesc <Row>                                       |
|  1 | +- Local Distributed Union <Row>                                                                |
|  2 |    +- Serialize Result <Row>                                                                    |
|  3 |       +- Index Scan on SongsBySingerAlbumSongNameDesc <Row> (Full scan, scan_method: Automatic) |
+----+-------------------------------------------------------------------------------------------------+
```

{{< /details >}}

#### Distributed Merge Union

複数の remote server に分散した subquery の結果を、指定された順序で merge して返す Distributed Union。
`PlanNode.displayName` として `Distributed Merge Union` という別名の operator が出るのではなく、spannerplan の表形式出力では `Distributed Union` に `preserve_subquery_order: true` metadata が付いた形で表示される。
入力側には `Sort`、`Sort Limit`、または順序を満たす scan など、順序付け済みの subquery が現れる。
公式ドキュメントでは distributed merge sort として説明されており、Spanner Version 3 以降ではデフォルトで有効とされている。

* https://docs.cloud.google.com/spanner/docs/query-operators-distributed#distributed-merge-union

{{< details summary="Distributed Merge Union 相当の再現クエリと実行計画" >}}

```sql
SELECT s.SongGenre
FROM Songs AS s
ORDER BY SongGenre;
```

```text
+----+---------------------------------------------------------------------------+
| ID | Operator                                                                  |
+----+---------------------------------------------------------------------------+
|  0 | Distributed Union on Songs <Row> (preserve_subquery_order: true)          |
|  1 | +- Serialize Result <Row>                                                 |
|  2 |    +- Sort <Row>                                                          |
|  3 |       +- Local Distributed Union <Row>                                    |
|  4 |          +- Table Scan on Songs <Row> (Full scan, scan_method: Automatic) |
+----+---------------------------------------------------------------------------+
```

{{< /details >}}

### Leaf operators

公式ドキュメントで Leaf operators に分類されている operator 群。

#### Array Unnest

配列の値と添字を元に Relation を作り出す operator。

* https://docs.cloud.google.com/spanner/docs/query-operators-leaf#array-unnest

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|SCALAR    | | Yes | | 配列の値に対応する変数名を指示する |
|SCALAR    | | Yes | | 配列の添字に対応する変数名を指示する |

{{< details summary="Array Unnest / Array Constructor の再現クエリと実行計画" >}}

```sql
SELECT a, b
FROM UNNEST([1, 2, 3]) a WITH OFFSET b;
```

```text
+----+------------------------+
| ID | Operator               |
+----+------------------------+
|  0 | Serialize Result <Row> |
|  1 | +- Array Unnest <Row>  |
+----+------------------------+
```

{{< /details >}}

#### Empty Relation

空の Relation を生成する。`LIMIT 0` を指定した際には常に結果は 0 行で何も Scan 等の入力をする必要がないが、 Relation operator ではある必要があるので使われる。

* https://docs.cloud.google.com/spanner/docs/query-operators-leaf#empty-relation

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|SCALAR    |  | | | 0 を意味する Constant |

{{< details summary="Empty Relation の再現クエリと実行計画" >}}

```sql
SELECT *
FROM Albums
LIMIT 0;
```

```text
+----+-------------------------+
| ID | Operator                |
+----+-------------------------+
|  0 | Serialize Result <Row>  |
|  1 | +- Empty Relation <Row> |
+----+-------------------------+
```

{{< /details >}}

#### Generate Relation

0 行以上の relation を生成する operator。
`INTERSECT ALL` や `EXCEPT ALL` で計算した出力行の multiplicity を復元する際に観測された。
`INTERSECT ALL` では左右の出現数の最小値、`EXCEPT ALL` では左右の出現数の差（0 未満は 0）に相当する repeat count を計算し、`Generate Relation` がその回数だけ行を生成する形になっている。
`SELECT 1 + 2` のような定数式だけのクエリは、現在の実行計画では `Generate Relation` ではなく `Unit Relation` として表示される。

* https://docs.cloud.google.com/spanner/docs/query-operators-leaf#generate-relation

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|SCALAR    | | | | 生成する行数を表す `Reference`。観測例では `$repeat_count`。 |

{{< details summary="Generate Relation の再現クエリと実行計画" >}}

```sql
SELECT SingerId FROM Albums
INTERSECT ALL
SELECT SingerId FROM Singers;
```

```text
+----+----------------------------------------------------------------------------------+
| ID | Operator                                                                         |
+----+----------------------------------------------------------------------------------+
|  0 | Distributed Union on Singers <Row> (split_ranges_aligned)                        |
|  1 | +- Serialize Result <Row>                                                        |
|  2 |    +- Cross Apply <Row>                                                          |
|  3 |       +- [Input] Compute <Row>                                                   |
|  4 |       |  +- Stream Aggregate <Row>                                               |
|  5 |       |     +- Local Distributed Union <Row>                                     |
|  6 |       |        +- Table Scan on Albums <Row> (Full scan, scan_method: Automatic) |
| 13 |       +- [Map] Generate Relation <Row>                                           |
+----+----------------------------------------------------------------------------------+
```

{{< /details >}}

#### Scan

各入力からのスキャンを行う。`PlanNode.displayName` としては Scan だが、一般的に `scan_type` の値と合わせて Index Scan, Table Scan などと表示される。

* https://docs.cloud.google.com/spanner/docs/query-operators-leaf#scan

##### Metadata

| key | values | description |
|-----|--------|-------------|
| Full scan | true もしくは未指定 ||
| scan_method | Automatic, Batch, Row | scan の実行方法。`SCAN_METHOD` hint と一致する場合もあるが、最終的には各 operator ごとに optimizer が決める。 |
| scan_format | Columnar, Row, 未指定 | scan が読むデータ形式。`SCAN_METHOD=COLUMNAR` では `Columnar`、`NO_COLUMNAR` では `Row` が観測された。 |
| scan_target | | スキャン対象の名前を指示する。 |
| scan_type | BatchScan, FilterScan, IndexScan, SearchIndexScan, SpoolScan, TableScan, VectorIndexLeafScan, VectorIndexMetadataScan, VectorIndexRootScan | スキャン対象の種類を指示する。raw `display_name: SpoolScan` として現れる場合は独立した `SpoolScan` operator であり、この metadata 値とは区別する。 |

`SCAN_METHOD=COLUMNAR` の観測例では raw `scan_method` が `Columnar` になるのではなく、`scan_method: Batch` と `scan_format: Columnar` の組み合わせになった。
`SCAN_METHOD=NO_COLUMNAR` の観測例では `scan_method: Automatic` と `scan_format: Row` の組み合わせになった。

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|SCALAR    |  | Yes | Yes | スキャン対象の列を表現する |
|SCALAR    | Seek Condition |  | | primary key または index key の範囲を絞るシークに使う Function。 |
|SCALAR    | Search Predicate |  | | Full Text Search の search index scan で使う検索条件を表現する |
|SCALAR    | Timestamp Condition |  | | commit timestamp column の storage pruning に使う条件。 |

過去の公式ドキュメントでは `Seek Condition` は `Filter Scan` の property として掲載されていたが、現行の公式ドキュメントでは `Seek Condition` は `Scan`、`Residual Condition` は `Filter Scan` の property として説明されている。この property table 上の分類変更だけから raw `childLinks` の parent がいつ変わったかは判断できない。
保持している optimizer version 1 から 8 の実行計画でも `Seek Condition` は一貫して `Scan` に付き、`Timestamp Condition` も観測例では `Scan` に付いた。
確認済みの実行計画だけから placement が変わった optimizer version を特定することはできないため、古い実行計画を扱う場合は predicate 名だけで親を決めず、raw `childLinks` の parent node を確認する必要がある。

Full Text Search の search index access は `SearchIndexScan` という独立した `PlanNode.displayName` ではなく、通常の `Scan` に `scan_type: SearchIndexScan` と `scan_target: <search index name>` が付く形で表現される。`SEARCH(...)` は `Search Query Conversion` という `TVF` と `VerifyDeterminism` を伴うことがある一方、観測した `SEARCH_SUBSTRING(...)` の例では `Search Query Conversion` は現れず、substring search index scan と `Search Predicate` が直接現れた。
`Search Predicate` 自体は scalar operator であり tree の行としては表示されないが、spannerplan の tree 表示では `SearchIndex Scan` 行に `*` が付き、`Predicates(identified by ID)` に `Search Predicate:` として出力される。複数列に対する AND / OR のような合成検索条件も、`SQUERY(...)` の AND / OR として predicate section に表示される。raw QueryPlan では、単純な検索条件では `Search Predicate` child link の先が scalar `Search Predicate` node になり、合成検索条件では同じ child link の先が scalar `Function` node になって、その子孫に複数の `Search Predicate` node が現れることがある。ツールでは child node の `display_name` だけでなく child link の `type` を見る方がよい。
`TOKENIZE_NUMBER(..., comparison_type=>"equality")` で生成した token column に search index を作成した例では、`ARRAY_INCLUDES_ANY(...)` と `ARRAY_INCLUDES_ALL(...)` でも `SearchIndexScan` が観測された。Full Text Search の検索条件と非テキスト条件を混在させた例でも `SearchIndexScan` が使われ、search index 側で処理しきれない条件は `Filter Scan` の residual condition として現れることがある。
`SNIPPET(...)` や search index に stored されていない列の参照は、観測した実行計画では base table への back join を発生させた。`ORDER BY SCORE(...) DESC` は `Sort` を発生させ、`TOKENLIST_CONCAT(...)` に `LIMIT` を組み合わせた ranked query では `Sort Limit` が現れた。一方、`PARTITION BY SingerId ORDER BY ReleaseTimestamp DESC` を持つ search index に対して同じ `SingerId` 条件と `ORDER BY ReleaseTimestamp DESC LIMIT ...` を使う query では、明示的な `Sort` は観測されなかった。

{{< details summary="Scan の再現クエリと実行計画" >}}

```sql
SELECT s.SongName
FROM Songs AS s;
```

```text
+----+-------------------------------------------------------------------------------------------------+
| ID | Operator                                                                                        |
+----+-------------------------------------------------------------------------------------------------+
|  0 | Distributed Union on SongsBySingerAlbumSongNameDesc <Row>                                       |
|  1 | +- Local Distributed Union <Row>                                                                |
|  2 |    +- Serialize Result <Row>                                                                    |
|  3 |       +- Index Scan on SongsBySingerAlbumSongNameDesc <Row> (Full scan, scan_method: Automatic) |
+----+-------------------------------------------------------------------------------------------------+
```

{{< /details >}}

{{< details summary="Full Text Search の追加再現クエリと実行計画" >}}

この例では Full Text Search 用に以下の schema を使用している。

```sql
CREATE TABLE SearchAlbums (
  SingerId INT64 NOT NULL,
  AlbumId STRING(MAX) NOT NULL,
  AlbumTitle STRING(MAX),
  AlbumStudio STRING(MAX),
  Rating FLOAT64,
  ReleaseTimestamp INT64 NOT NULL,
  Likes INT64,
  Genres ARRAY<STRING(MAX)>,
  Cover BYTES(MAX),
  Ratings ARRAY<INT64>,
  AlbumTitle_Tokens TOKENLIST AS (TOKENIZE_FULLTEXT(AlbumTitle)) HIDDEN,
  AlbumTitle_SubstringTokens TOKENLIST AS (TOKENIZE_SUBSTRING(AlbumTitle)) HIDDEN,
  AlbumStudio_Tokens TOKENLIST AS (TOKENIZE_FULLTEXT(AlbumStudio)) HIDDEN,
  Rating_Tokens TOKENLIST AS (TOKENIZE_NUMBER(Rating)) HIDDEN,
  Genres_Tokens TOKENLIST AS (TOKEN(Genres)) HIDDEN,
  Ratings_Tokens TOKENLIST AS (TOKENIZE_NUMBER(Ratings, comparison_type=>"equality")) HIDDEN,
) PRIMARY KEY(SingerId, AlbumId);

CREATE SEARCH INDEX SearchAlbumsTitleStudioIndex
ON SearchAlbums(AlbumTitle_Tokens, AlbumStudio_Tokens);

CREATE SEARCH INDEX SearchAlbumsRatingsIndex
ON SearchAlbums(Ratings_Tokens);

CREATE SEARCH INDEX SearchAlbumsMixedIndex
ON SearchAlbums(AlbumTitle_Tokens, Rating_Tokens, Genres_Tokens)
STORING (Likes);
```

数値配列に対する `ARRAY_INCLUDES_ANY(...)`:

```sql
SELECT AlbumId
FROM SearchAlbums
WHERE ARRAY_INCLUDES_ANY(Ratings, [1, 2]);
```

```text
+----+--------------------------------------------------------------------------------+
| ID | Operator                                                                       |
+----+--------------------------------------------------------------------------------+
|  0 | Distributed Union on _Search2aryIndex_SearchAlbumsRatingsIndex <Row>           |
|  1 | +- Local Distributed Union <Row>                                               |
|  2 |    +- Serialize Result <Row>                                                   |
| *3 |       +- SearchIndex Scan on SearchAlbumsRatingsIndex <Row> (scan_method: Row) |
+----+--------------------------------------------------------------------------------+
Predicates(identified by ID):
 3: Search Predicate: SQUERY(index_name:Ratings_Tokens in SearchAlbumsRatingsIndex predicate:SPAN_NUMBER_QUERY_TO_SQUERY(9, false, false, SPAN_MAKE_NUMRANGE(9, [1, 2])))
```

数値配列に対する `ARRAY_INCLUDES_ALL(...)`:

```sql
SELECT AlbumId
FROM SearchAlbums
WHERE ARRAY_INCLUDES_ALL(Ratings, [1, 5]);
```

```text
+----+--------------------------------------------------------------------------------+
| ID | Operator                                                                       |
+----+--------------------------------------------------------------------------------+
|  0 | Distributed Union on _Search2aryIndex_SearchAlbumsRatingsIndex <Row>           |
|  1 | +- Local Distributed Union <Row>                                               |
|  2 |    +- Serialize Result <Row>                                                   |
| *3 |       +- SearchIndex Scan on SearchAlbumsRatingsIndex <Row> (scan_method: Row) |
+----+--------------------------------------------------------------------------------+
Predicates(identified by ID):
 3: Search Predicate: SQUERY(index_name:Ratings_Tokens in SearchAlbumsRatingsIndex predicate:SPAN_NUMBER_QUERY_TO_SQUERY(8, false, false, SPAN_MAKE_NUMRANGE(8, [1, 5])))
```

複数列の検索条件では `Search Predicate` が AND / OR で合成された predicate として表示される。

```sql
SELECT AlbumId
FROM SearchAlbums
WHERE SEARCH(AlbumTitle_Tokens, "car")
  AND SEARCH(AlbumStudio_Tokens, "sun");
```

```text
+-----+---------------------------------------------------------------------------------------+
| ID  | Operator                                                                              |
+-----+---------------------------------------------------------------------------------------+
|   0 | Cross Apply <Row>                                                                     |
|   1 | +- [Input] VerifyDeterminism <Row>                                                    |
|   2 | |  +- TVF <Row> (Name: Search Query Conversion)                                       |
|   3 | |     +- Unit Relation <Row>                                                          |
|   9 | +- [Map] Distributed Union on _Search2aryIndex_SearchAlbumsTitleStudioIndex <Row>     |
|  10 |    +- Local Distributed Union <Row>                                                   |
|  11 |       +- Serialize Result <Row>                                                       |
| *12 |          +- SearchIndex Scan on SearchAlbumsTitleStudioIndex <Row> (scan_method: Row) |
+-----+---------------------------------------------------------------------------------------+
Predicates(identified by ID):
 12: Search Predicate: (SQUERY(index_name:AlbumTitle_Tokens in SearchAlbumsTitleStudioIndex predicate:$oo_tvf_0) AND SQUERY(index_name:AlbumStudio_Tokens in SearchAlbumsTitleStudioIndex predicate:$oo_tvf_1))
```

Full Text Search と非テキスト条件を混在させた場合、条件の一部は search index scan の上の `Filter Scan` に残ることがある。

```sql
SELECT AlbumId
FROM SearchAlbums@{FORCE_INDEX=SearchAlbumsMixedIndex}
WHERE SEARCH(AlbumTitle_Tokens, "car")
  AND Rating > 4
  AND Likes >= 1000;
```

```text
+-----+------------------------------------------------------------------------------------+
| ID  | Operator                                                                           |
+-----+------------------------------------------------------------------------------------+
|   0 | Cross Apply <Row>                                                                  |
|   1 | +- [Input] VerifyDeterminism <Row>                                                 |
|   2 | |  +- TVF <Row> (Name: Search Query Conversion)                                    |
|   3 | |     +- Unit Relation <Row>                                                       |
|   7 | +- [Map] Distributed Union on _Search2aryIndex_SearchAlbumsMixedIndex <Row>        |
|   8 |    +- Local Distributed Union <Row>                                                |
|   9 |       +- Serialize Result <Row>                                                    |
| *10 |          +- Filter Scan <Row> (seekable_key_size: 0)                               |
| *11 |             +- SearchIndex Scan on SearchAlbumsMixedIndex <Row> (scan_method: Row) |
+-----+------------------------------------------------------------------------------------+
Predicates(identified by ID):
 10: Residual Condition: (($Rating > 4) AND ($Likes >= 1000))
 11: Search Predicate: (SQUERY(index_name:Rating_Tokens in SearchAlbumsMixedIndex predicate:'(o r%01%0E5%FA%93%1A%00%00%01%04 r%01%1Ck%F5&4%00%00%01%02...(length 1371)') AND SQUERY(index_name:AlbumTitle_Tokens in SearchAlbumsMixedIndex predicate:$oo_tvf_0))
```

search index に stored されていない列を参照すると、base table への back join が現れることがある。

```sql
SELECT AlbumId, Cover
FROM SearchAlbums@{FORCE_INDEX=SearchAlbumsMixedIndex}
WHERE SEARCH(AlbumTitle_Tokens, "car")
  AND Rating > 4;
```

```text
+-----+------------------------------------------------------------------------------------------+
| ID  | Operator                                                                                 |
+-----+------------------------------------------------------------------------------------------+
|   0 | Cross Apply <Row>                                                                        |
|   1 | +- [Input] VerifyDeterminism <Row>                                                       |
|   2 | |  +- TVF <Row> (Name: Search Query Conversion)                                          |
|   3 | |     +- Unit Relation <Row>                                                             |
|   7 | +- [Map] Distributed Union on _Search2aryIndex_SearchAlbumsMixedIndex <Row>              |
|  *8 |    +- Distributed Cross Apply <Row>                                                      |
|   9 |       +- [Input] Create Batch <Batch>                                                    |
|  10 |       |  +- RowToDataBlock                                                               |
|  11 |       |     +- Local Distributed Union <Row>                                             |
| *12 |       |        +- Filter Scan <Row> (seekable_key_size: 0)                               |
| *13 |       |           +- SearchIndex Scan on SearchAlbumsMixedIndex <Row> (scan_method: Row) |
|  29 |       +- [Map] Serialize Result <Row>                                                    |
|  30 |          +- Cross Apply <Row>                                                            |
|  31 |             +- [Input] KeyRangeAccumulator <Row>                                         |
|  32 |             |  +- DataBlockToRow                                                         |
|  33 |             |     +- Batch Scan on $v2 <Batch> (scan_method: Batch)                      |
|  38 |             +- [Map] Local Distributed Union <Row>                                       |
|  39 |                +- Filter Scan <Row> (seekable_key_size: 0)                               |
| *40 |                   +- Table Scan on SearchAlbums <Row> (scan_method: Row)                 |
+-----+------------------------------------------------------------------------------------------+
Predicates(identified by ID):
  8: Split Range: (($SearchAlbums_key_SingerId' = $SearchAlbums_key_SingerId) AND ($AlbumId' = $AlbumId))
 12: Residual Condition: ($Rating > 4)
 13: Search Predicate: (SQUERY(index_name:Rating_Tokens in SearchAlbumsMixedIndex predicate:'(o r%01%0E5%FA%93%1A%00%00%01%04 r%01%1Ck%F5&4%00%00%01%02...(length 1371)') AND SQUERY(index_name:AlbumTitle_Tokens in SearchAlbumsMixedIndex predicate:$oo_tvf_0))
 40: Seek Condition: (($SearchAlbums_key_SingerId' = $batched_SearchAlbums_key_SingerId') AND ($AlbumId' = $batched_AlbumId'))
```

{{< /details >}}

#### Filter Scan

Scan のすぐ上に位置し、スキャンに伴って処理できるフィルタを行う。現在の QueryPlan では `display_name` / `displayName` 自体が `Filter Scan` とスペース入りで返る。
以前の実行計画や古い説明では `FilterScan` と表記されることがあったが、現行の operator name としては `Filter Scan` を使う。
Scan の一部として働くため `executionStats` を持たず、実行時の挙動は Scan 側の `rows`, `filtered_rows` などを通して確認できる。

* https://docs.cloud.google.com/spanner/docs/query-operators-leaf#filter_scan

##### Metadata

| key | values | description |
|-----|--------|-------------|
| seekable_key_size | | `SEEKABLE_KEY_SIZE` hint などに対応して観測される整数。`Seek Condition` の式や使用列数と単純には一致せず、0 でも underlying `Scan` に `Seek Condition` が現れる場合がある。 |

##### ChildLinks
|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL|  | | | フィルタの入力となる Scan |
|SCALAR    | Residual Condition |  | | スキャン後のフィルタに使う Function であり、[フィルタ述語](https://use-the-index-luke.com/ja/sql/where-clause/searching-for-ranges/greater-less-between-tuning-sql-access-filter-predicates)に対応する。 |
|SCALAR    | Scalar |  | | scalar subquery など、Filter Scan から参照する scalar expression。 |

`ALLOW_TIMESTAMP_PREDICATE_PUSHDOWN=TRUE` を使うと、`allow_commit_timestamp = true` の timestamp column に対する filter が `Timestamp Condition` child link として underlying table scan に付くことがある。`ALLOW_TIMESTAMP_PREDICATE_PUSHDOWN=FALSE` では、非 key timestamp の同じ query は Filter Scan の residual condition のみになる。locality group や age-based tiered storage は performance goal には関係するが、少なくとも plan 上の `Timestamp Condition` を観測するだけなら必須ではなかった。

timestamp column 自体が leading primary key の場合も、`ALLOW_TIMESTAMP_PREDICATE_PUSHDOWN=TRUE` の range predicate は同一の `Scan` に `Seek Condition` と `Timestamp Condition` の両方を付けた。保持している optimizer version 1 から 8 でこの組み合わせを確認している。`FALSE` では同じ `Scan` の `Seek Condition` は残り、`Timestamp Condition` は現れなかった。前者は key range access、後者は commit timestamp 用の storage pruning という別の plan-level signal として併存しうる。

{{< details summary="Filter Scan の再現クエリと実行計画" >}}

```sql
SELECT LastName
FROM Singers
WHERE SingerId = 1;
```

```text
+----+------------------------------------------------------------+
| ID | Operator                                                   |
+----+------------------------------------------------------------+
| *0 | Distributed Union on Singers <Row>                         |
|  1 | +- Local Distributed Union <Row>                           |
|  2 |    +- Serialize Result <Row>                               |
|  3 |       +- Filter Scan <Row> (seekable_key_size: 0)          |
| *4 |          +- Table Scan on Singers <Row> (scan_method: Row) |
+----+------------------------------------------------------------+
Predicates(identified by ID):
 0: Split Range: ($SingerId = 1)
 4: Seek Condition: ($SingerId = 1)
```

{{< /details >}}

#### Recursive Spool Scan

Graph query の recursive path などで、`Recursive Union` の再帰ステップから前回までの中間結果を参照するために現れる。
通常の repeated CTE では `SpoolScan` という表示名で観測されるが、recursive plan では `Recursive Spool Scan` という別の表示名の leaf operator として観測される。
公式ドキュメントでは独立した operator 節はないが、Recursive Union の説明内で recursive spool scan として言及されている。

* https://docs.cloud.google.com/spanner/docs/query-operators-binary#recursive-union

##### ChildLinks

確認できている範囲では Relational operator の子を持たない。

{{< details summary="Recursive Spool Scan の再現 SQL" >}}

以下は該当 operator を観測できる再現 SQL の例である。対応する実行計画は [Recursive Union](/operators/#recursive-union) の details に含めている。

```sql
GRAPH MusicGraph
MATCH (singer:Singers {singerId:42})-[c:CollabWith]->{1,2}(featured:Singers)
RETURN singer.SingerId AS singer, featured.SingerId AS featured;
```

{{< /details >}}

#### Unit Relation

特に値を持たない単一の行を生成する。 Unit Relation を受ける Compute や Serialize Result で実際の列の値が設定される。
例: `SELECT 42`, `SELECT 42 UNION ALL SELECT 43`

* https://docs.cloud.google.com/spanner/docs/query-operators-leaf#unit-relation

##### Child Links

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|SCALAR    | | | Yes | `1` を表現する Constant が常に指定される。|

{{< details summary="Unit Relation / Constant / Function の再現クエリと実行計画" >}}

```sql
SELECT 1 + 2 AS Result;
```

```text
+----+------------------------+
| ID | Operator               |
+----+------------------------+
|  0 | Serialize Result <Row> |
|  1 | +- Unit Relation <Row> |
+----+------------------------+
```

{{< /details >}}

### Unary operators

Relational operator の子を1つだけ持つ Relational operator 群。

#### Aggregate

`GROUP BY` に対応する集約を行う。
入力がインデックス等で既にソート済であり、その順序で集約することでハッシュテーブルを使わなくて良い時は `iterator_type` が Stream となり Stream Aggregate と呼ばれる。
`GROUP@{GROUP_METHOD=HASH_GROUP}` と `GROUP@{GROUP_METHOD=STREAM_GROUP}` で同じ Aggregate operator の `iterator_type` が Hash / Stream に切り替わるため、実行計画の検査では `Aggregate` という operator 名だけでなく metadata も確認する必要がある。

* https://docs.cloud.google.com/spanner/docs/query-operators-unary#aggregate

##### Metadata

| key | values | description |
|-----|--------|-------------|
| call_type | Local もしくは Global | |
| iterator_type| Hash, Stream, もしくは未指定 | Stream か Hash による処理方法の区別を示す。|
| scalar_aggregate| true か未指定 | |

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL|  |  | | 入力 |
|SCALAR    | Key | Yes | Yes | `scalar_aggregate=true` の時には存在しない。集約に使うキーを示す。|
|SCALAR    | Agg| Yes | Yes |Aggregate 対象の値を示す。|

{{< details summary="Hash Aggregate / Stream Aggregate の再現クエリと実行計画" >}}

Hash Aggregate:

```sql
SELECT s.SingerId, COUNT(*) AS SongCount
FROM Songs AS s
GROUP@{GROUP_METHOD=HASH_GROUP} BY s.SingerId;
```

```text
+----+----------------------------------------------------------------------------------------------------+
| ID | Operator                                                                                           |
+----+----------------------------------------------------------------------------------------------------+
|  0 | Distributed Union on Singers <Row> (split_ranges_aligned)                                          |
|  1 | +- Serialize Result <Row>                                                                          |
|  2 |    +- Hash Aggregate <Row>                                                                         |
|  3 |       +- Local Distributed Union <Row>                                                             |
|  4 |          +- Index Scan on SongsBySingerAlbumSongNameDesc <Row> (Full scan, scan_method: Automatic) |
+----+----------------------------------------------------------------------------------------------------+
```

Stream Aggregate:

```sql
SELECT s.SingerId, COUNT(*) AS SongCount
FROM Songs AS s
GROUP@{GROUP_METHOD=STREAM_GROUP} BY s.SingerId;
```

```text
+----+----------------------------------------------------------------------------------------------------+
| ID | Operator                                                                                           |
+----+----------------------------------------------------------------------------------------------------+
|  0 | Distributed Union on Singers <Row> (split_ranges_aligned)                                          |
|  1 | +- Serialize Result <Row>                                                                          |
|  2 |    +- Stream Aggregate <Row>                                                                       |
|  3 |       +- Local Distributed Union <Row>                                                             |
|  4 |          +- Index Scan on SongsBySingerAlbumSongNameDesc <Row> (Full scan, scan_method: Automatic) |
+----+----------------------------------------------------------------------------------------------------+
```

{{< /details >}}

#### Apply Mutations

DML である `INSERT`, `UPDATE`, `DELETE` を処理する。サブツリーから row として取得した主キーと更新後の値を適用すると考えられるが、どの列をどのような式で更新するかのような定義は実行計画上は見えない。
`INSERT IGNORE`、`INSERT OR IGNORE`、`INSERT OR UPDATE`、`ON CONFLICT DO NOTHING`、`ON CONFLICT DO UPDATE`、`THEN RETURN` も `Apply Mutations` を含む plan shape として観測できた。Spanner Omni 2026.r1-beta では、`PLAN` において `INSERT ... ASSERT_ROWS_MODIFIED 1` は `ASSERT_ROWS_MODIFIED is not supported.` で失敗した。

* https://docs.cloud.google.com/spanner/docs/query-operators-unary#apply-mutations

##### Metadata

| key | values | description |
|-----|--------|-------------|
| operation_type | INSERT, UPDATE, DELETE | |
| table | | 更新対象のテーブル |

##### ChildLinks

|kind      | type | variable | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL|  |  | | 入力 |

{{< details summary="Apply Mutations の再現クエリと実行計画" >}}

```sql
UPDATE Singers
SET LastName = "Smith"
WHERE SingerId = 1;
```

```text
+----+---------------------------------------------------------------+
| ID | Operator                                                      |
+----+---------------------------------------------------------------+
|  0 | Apply Mutations on Singers <Row> (operation_type: UPDATE)     |
|  1 | +- Serialize Result <Row>                                     |
| *2 |    +- Distributed Union on Singers <Row>                      |
|  3 |       +- Local Distributed Union <Row>                        |
|  4 |          +- Filter Scan <Row> (seekable_key_size: 0)          |
| *5 |             +- Table Scan on Singers <Row> (scan_method: Row) |
+----+---------------------------------------------------------------+
Predicates(identified by ID):
 2: Split Range: ($SingerId = 1)
 5: Seek Condition: ($SingerId = 1)
```

{{< /details >}}

#### BloomFilterBuild

(Undocumented)
Bloom Filter を構築する。通常 Hash Join の Build 側に現れる。後に `BLOOM_FILTER_MATCH` を Condition に持つ Filter で使われる。

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL| | | |入力 |

{{< details summary="BloomFilterBuild の再現クエリと実行計画" >}}

```sql
SELECT AlbumTitle
FROM Songs
JOIN Albums ON Albums.AlbumId = Songs.AlbumId;
```

```text
+-----+-------------------------------------------------------------------------------------------------------+
| ID  | Operator                                                                                              |
+-----+-------------------------------------------------------------------------------------------------------+
|   0 | Serialize Result <Row>                                                                                |
|  *1 | +- Hash Join <Row> (join_type: INNER)                                                                 |
|   2 |    +- [Build] BloomFilterBuild <Row>                                                                  |
|   3 |    |  +- Distributed Union on SongsBySingerAlbumSongNameDesc <Row>                                    |
|   4 |    |     +- Local Distributed Union <Row>                                                             |
|   5 |    |        +- Index Scan on SongsBySingerAlbumSongNameDesc <Row> (Full scan, scan_method: Automatic) |
|   9 |    +- [Probe] Distributed Union on AlbumsByAlbumTitle <Row>                                           |
|  10 |       +- Local Distributed Union <Row>                                                                |
| *11 |          +- Filter Scan <Row> (seekable_key_size: 0)                                                  |
|  12 |             +- Index Scan on AlbumsByAlbumTitle <Row> (Full scan, scan_method: Automatic)             |
+-----+-------------------------------------------------------------------------------------------------------+
Predicates(identified by ID):
  1: Condition: ($AlbumId_1 = $AlbumId)
 11: Residual Condition: BLOOM_FILTER_MATCH($existence_filter, $AlbumId_1)
```

{{< /details >}}

#### Compute

入力のそれぞれの行に対して新しい列を追加する。

* https://docs.cloud.google.com/spanner/docs/query-operators-unary#compute

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL| | | | 入力 |
|SCALAR    | | Yes | Yes | 新しく計算する値を示す |

{{< details summary="Compute の再現クエリと実行計画" >}}

```sql
SELECT 1 AS a, 2 AS b
UNION ALL SELECT 3 AS a, 4 AS b
UNION ALL SELECT 5 AS a, 6 AS b;
```

```text
+----+---------------------------------+
| ID | Operator                        |
+----+---------------------------------+
|  0 | Serialize Result <Row>          |
|  1 | +- Union All <Row>              |
|  2 |    +- Union Input               |
|  3 |    |  +- Compute <Row>          |
|  4 |    |     +- Unit Relation <Row> |
| 10 |    +- Union Input               |
| 11 |    |  +- Compute <Row>          |
| 12 |    |     +- Unit Relation <Row> |
| 18 |    +- Union Input               |
| 19 |       +- Compute <Row>          |
| 20 |          +- Unit Relation <Row> |
+----+---------------------------------+
```

{{< /details >}}

#### Compute Struct

入力のそれぞれの行に対して STRUCT を生成する。 Compute Batch の入力や `AS STRUCT` を使ったサブクエリなどで現れる。

* https://docs.cloud.google.com/spanner/docs/query-operators-unary#compute_struct

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL| | | | 入力 |
|SCALAR    | | Yes | Yes | STRUCT の各フィールドを表す |
|SCALAR    | Scalar | | Yes | 式で参照される Scalar Subquery(or Array Subquery) を指す。 |

{{< details summary="Compute Struct / Array Subquery の再現クエリと実行計画" >}}

```sql
SELECT FirstName,
       ARRAY(
         SELECT AS STRUCT song.SongName, song.SongGenre
         FROM Songs AS song
         WHERE song.SingerId = singer.SingerId
       )
FROM Singers AS singer
WHERE singer.SingerId = 1;
```

```text
+-----+-------------------------------------------------------------------+
| ID  | Operator                                                          |
+-----+-------------------------------------------------------------------+
|  *0 | Distributed Union on Singers <Row> (split_ranges_aligned)         |
|   1 | +- Local Distributed Union <Row>                                  |
|   2 |    +- Serialize Result <Row>                                      |
|   3 |       +- Filter Scan <Row> (seekable_key_size: 0)                 |
|  *4 |       |  +- Table Scan on Singers <Row> (scan_method: Row)        |
|  12 |       +- [Scalar] Array Subquery                                  |
|  13 |          +- Local Distributed Union <Row>                         |
|  14 |             +- Compute Struct <Row>                               |
|  15 |                +- Filter Scan <Row> (seekable_key_size: 0)        |
| *16 |                   +- Table Scan on Songs <Row> (scan_method: Row) |
+-----+-------------------------------------------------------------------+
Predicates(identified by ID):
  0: Split Range: ($SingerId = 1)
  4: Seek Condition: ($SingerId = 1)
 16: Seek Condition: ($SingerId_1 = 1)
```

{{< /details >}}

#### Create Batch

入力から batch を作成する。主に Distributed Cross Apply で入力をまとめて対応する replica に送り、 Batch Scan で参照するために使われる。
具体例は `Distributed Cross Apply`、`Push Broadcast Hash Join`、`Recursive Union` の再現クエリと実行計画で確認できる。

* https://docs.cloud.google.com/spanner/docs/query-operators-unary#create_batch

##### ChildLinks

|kind      | type | variable | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL| | | | 入力 |
|SCALAR    | | variable | Yes | 生成される batch relation の field を定義する。`v2.Batch.SingerId` や `v2.Batch.__row_id` のような variable が付き、broadcast key、back join key、sort value、graph traversal key などが context に応じて現れる。 |

#### Crowd

(Undocumented)

Graph query の [`IS_FIRST(k) OVER (...)`](https://docs.cloud.google.com/spanner/docs/reference/standard-sql/graph-gql-functions#is_first) を含む実行計画で観測された operator。
`IS_FIRST` は window 内の先頭 `k` 行であるかを判定する関数で、`PARTITION BY` や `ORDER BY` と組み合わせて graph traversal の対象を制限する用途などで使われる。
下記の例では `Crowd` の入力 subtree 内に `Sort` があり、optimizer version 1 から 8 まで同じ operator と ChildLinks の構成になった。

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL| | | | window 関数を評価する入力。 |
|SCALAR    | | | Yes | 下記の例では variable のない 3 個の child があり、式との対応から `k`、`PARTITION BY`、`ORDER BY` の値を表すと考えられる。 |
|SCALAR    | | Yes | | 判定結果を親から参照するための出力。観測例では `Reference` の description が `Crowd state`。 |

{{< details summary="Crowd / IS_FIRST の再現クエリと実行計画" >}}

```sql
GRAPH MusicGraph
MATCH (s:Singers)-[:CollabWith]->(f:Singers)
RETURN s.SingerId AS singer_id,
       f.SingerId AS friend_id,
       IS_FIRST(1) OVER (
         PARTITION BY s.SingerId
         ORDER BY f.SingerId
       ) AS first_friend;
```

```text
+-----+---------------------------------------------------------------------------------------------------------+
| ID  | Operator                                                                                                |
+-----+---------------------------------------------------------------------------------------------------------+
|   0 | Serialize Result <Row>                                                                                  |
|   1 | +- Crowd <Row>                                                                                          |
|   2 |    +- Distributed Union on Collaborations <Row> (preserve_subquery_order: true)                         |
|  *3 |       +- Distributed Cross Apply <Row> (order_preserving: true)                                         |
|   4 |          +- [Input] Create Batch <Batch>                                                                |
|   5 |          |  +- RowToDataBlock                                                                           |
|  *6 |          |     +- Distributed Cross Apply <Row> (order_preserving: true)                                |
|   7 |          |        +- [Input] Create Batch <Batch>                                                       |
|   8 |          |        |  +- RowToDataBlock                                                                  |
|   9 |          |        |     +- Sort <Row>                                                                   |
|  10 |          |        |        +- Local Distributed Union <Row>                                             |
|  11 |          |        |           +- Table Scan on Collaborations <Row> (Full scan, scan_method: Automatic) |
|  19 |          |        +- [Map] Cross Apply <Row>                                                            |
|  20 |          |           +- [Input] KeyRangeAccumulator <Row>                                               |
|  21 |          |           |  +- DataBlockToRow                                                               |
|  22 |          |           |     +- Batch Scan on $v2 <Batch> (scan_method: Batch)                            |
|  29 |          |           +- [Map] Local Distributed Union <Row>                                             |
|  30 |          |              +- Filter Scan <Row> (seekable_key_size: 0)                                     |
| *31 |          |                 +- Table Scan on Singers <Row> (scan_method: Row)                            |
|  46 |          +- [Map] Cross Apply <Row>                                                                     |
|  47 |             +- [Input] KeyRangeAccumulator <Row>                                                        |
|  48 |             |  +- DataBlockToRow                                                                        |
|  49 |             |     +- Batch Scan on $v5 <Batch> (scan_method: Batch)                                     |
|  56 |             +- [Map] Local Distributed Union <Row>                                                      |
|  57 |                +- Filter Scan <Row> (seekable_key_size: 0)                                              |
| *58 |                   +- Table Scan on Singers <Row> (scan_method: Row)                                     |
+-----+---------------------------------------------------------------------------------------------------------+
Predicates(identified by ID):
  3: Split Range: ($SingerId_2 = $batched_FeaturingSingerId'2)
  6: Split Range: ($SingerId = $sort_SingerId_1)
 31: Seek Condition: ($SingerId = $batched_SingerId_1')
 58: Seek Condition: ($SingerId_2 = $batched_FeaturingSingerId'4)
```

{{< /details >}}

#### DataBlockToRow

DataBlockToRow は batch/data-block 形式の入力を行指向の relational stream に変換する。
index back join、batch/distributed 系の Apply Join、Graph query、Push Broadcast Hash Join などの内部で観測される。
`Batch Scan` から得た `<Batch>` を `Cross Apply` や `Hash Join` の入力に戻す箇所に現れることが多い。
公式ドキュメント上の operator 名は `DataBlockToRowAdapter` である。
これらの変換 operator は row-oriented execution と batch-oriented execution の境界で現れるため、`EXECUTION_METHOD=ROW` を指定すると消えることがある。
一方で `SCAN_METHOD` は scan 処理の row/batch/columnar を制御する hint であり、`SCAN_METHOD=ROW` だけでこれらの変換 operator が消えるとは限らない。
また、`SCAN_METHOD=BATCH` は apply join の右側などではサポートされない場合がある。

* https://docs.cloud.google.com/spanner/docs/query-operators-unary#datablocktorowadapter

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL| | | | data-block 形式の入力 |

{{< details summary="DataBlockToRow / RowToDataBlock の再現クエリと実行計画" >}}

```sql
SELECT s.SongName, s.Duration
FROM Songs@{FORCE_INDEX=SongsBySongName} AS s
WHERE STARTS_WITH(s.SongName, "B");
```

```text
+-----+--------------------------------------------------------------------------+
| ID  | Operator                                                                 |
+-----+--------------------------------------------------------------------------+
|  *0 | Distributed Union on SongsBySongName <Row>                               |
|  *1 | +- Distributed Cross Apply <Row>                                         |
|   2 |    +- [Input] Create Batch <Batch>                                       |
|   3 |    |  +- RowToDataBlock                                                  |
|   4 |    |     +- Local Distributed Union <Row>                                |
|   5 |    |        +- Filter Scan <Row> (seekable_key_size: 1)                  |
|  *6 |    |           +- Index Scan on SongsBySongName <Row> (scan_method: Row) |
|  19 |    +- [Map] Serialize Result <Row>                                       |
|  20 |       +- Cross Apply <Row>                                               |
|  21 |          +- [Input] KeyRangeAccumulator <Row>                            |
|  22 |          |  +- DataBlockToRow                                            |
|  23 |          |     +- Batch Scan on $v2 <Batch> (scan_method: Batch)         |
|  32 |          +- [Map] Local Distributed Union <Row>                          |
|  33 |             +- Filter Scan <Row> (seekable_key_size: 0)                  |
| *34 |                +- Table Scan on Songs <Row> (scan_method: Row)           |
+-----+--------------------------------------------------------------------------+
Predicates(identified by ID):
  0: Split Range: STARTS_WITH($SongName, 'B')
  1: Split Range: (($Songs_key_SingerId' = $Songs_key_SingerId) AND ($Songs_key_AlbumId' = $Songs_key_AlbumId) AND ($Songs_key_TrackId' = $Songs_key_TrackId))
  6: Seek Condition: STARTS_WITH($SongName, 'B')
 34: Seek Condition: (($Songs_key_SingerId' = $batched_Songs_key_SingerId') AND ($Songs_key_AlbumId' = $batched_Songs_key_AlbumId') AND ($Songs_key_TrackId' = $batched_Songs_key_TrackId'))
```

{{< /details >}}

#### Filter

Scan とは独立して任意の箇所で `Condition` 述語で行をフィルタする。フィルタプッシュダウンができないようなサブクエリの外側の WHERE や、 GROUP BY の結果に対して適用する必要がある HAVING は Filter Scan ではなく Filter として処理される。

* https://docs.cloud.google.com/spanner/docs/query-operators-unary#filter

##### ChildLinks

|kind      | type | variable | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL|  | | | フィルタの入力となる Scan |
|SCALAR    | Condition |  | | 入力からフィルタする Function |

{{< details summary="Filter の再現クエリと実行計画" >}}

```sql
SELECT s.LastName
FROM (SELECT s.LastName FROM Singers AS s LIMIT 3) s
WHERE s.LastName LIKE 'Rich%';
```

```text
+----+--------------------------------------------------------------------------------------------+
| ID | Operator                                                                                   |
+----+--------------------------------------------------------------------------------------------+
|  0 | Serialize Result <Row>                                                                     |
| *1 | +- Filter <Row>                                                                            |
|  2 |    +- Global Limit <Row>                                                                   |
|  3 |       +- Distributed Union on SingersByFirstLastName <Row>                                 |
|  4 |          +- Local Limit <Row>                                                              |
|  5 |             +- Local Distributed Union <Row>                                               |
|  6 |                +- Index Scan on SingersByFirstLastName <Row> (Full scan, scan_method: Row) |
+----+--------------------------------------------------------------------------------------------+
Predicates(identified by ID):
 1: Condition: STARTS_WITH($LastName, 'Rich')
```

{{< /details >}}

#### KeyRangeAccumulator

(Undocumented)

Batch 化された key を受け取り、後段の seek や back join で使う key range を relation としてまとめる箇所で観測された operator。
典型的には `Cross Apply` の `Input` 側にあり、`Batch Scan` から `DataBlockToRow` を介して得た入力を受ける。
名称と配置から上記の役割が推測できるが、key range の構築規則そのものは公開 operator documentation では説明されていない。
再現例は `DataBlockToRow`、`Crowd`、`Recursive Union` の実行計画を参照。

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL| | | | batch/data-block 形式から行形式に変換された key の入力。 |

#### Limit

Limit のみを行う。 `ORDER BY` を指定しないか、キー順と一致する順序で指定して `LIMIT` を指定した際に現れる。

* https://docs.cloud.google.com/spanner/docs/query-operators-unary#limit

##### Metadata

| key | values | description |
|-----|--------|-------------|
| call_type | Local もしくは Global ||

##### ChildLinks

|kind      | type | variable | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL|  | | | ソート対象の入力 |
|SCALAR    | Limit |  | | 取得する行数を指定する |
|SCALAR    | Offset |  | | `OFFSET` 指定時に読み飛ばす行数を指定する |

{{< details summary="Limit の再現クエリと実行計画" >}}

```sql
SELECT s.SongName
FROM Songs AS s
LIMIT 3;
```

```text
+----+-------------------------------------------------------------------------------------------------+
| ID | Operator                                                                                        |
+----+-------------------------------------------------------------------------------------------------+
|  0 | Global Limit <Row>                                                                              |
|  1 | +- Distributed Union on SongsBySingerAlbumSongNameDesc <Row>                                    |
|  2 |    +- Serialize Result <Row>                                                                    |
|  3 |       +- Local Limit <Row>                                                                      |
|  4 |          +- Local Distributed Union <Row>                                                       |
|  5 |             +- Index Scan on SongsBySingerAlbumSongNameDesc <Row> (Full scan, scan_method: Row) |
+----+-------------------------------------------------------------------------------------------------+
```

{{< /details >}}

#### Local Split Union

ローカル server に保存されている table split を探し、それぞれの split 上で subquery を実行して結果を union する operator。
公式ドキュメントでは placement table の scan で現れる operator として説明されている。
現時点のフィードバックでは、サンプルスキーマだけで `Local Split Union` を安定して出す再現クエリは確認できていない。placement や locality を含む構成が必要と考えられる。
なお、この文書の観測環境である Spanner Omni では placement table 自体が作成できない(`CREATE PLACEMENT` が `Unimplemented: Geo-partitioning is not supported for this environment` で失敗する)ため、Omni 上での再現は現状不可能とみられる。

* https://docs.cloud.google.com/spanner/docs/query-operators-unary#local-split-union

#### MiniBatchAssign

(Undocumented)
MiniBatchKeyOrder より下にある以外はよく分かっていない。

[Shard 最適化クエリ](https://github.com/gcpug/nouhau/blob/spanner/shard/spanner/note/shard/README.md#v3) などで確認されている。
この operator を含む実行計画は `MiniBatchAssign / MiniBatchKeyOrder / RowCount` の再現クエリと実行計画で確認できる。

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL|  | | | 入力となる Relation operator。 |
|SCALAR    |  | | | Scalar operator。バッチサイズを指示している？ |

#### MiniBatchKeyOrder

(Undocumented)

MiniBatchAssign より上にある以外はよく分かっていない。

[Shard 最適化クエリ](https://github.com/gcpug/nouhau/blob/spanner/shard/spanner/note/shard/README.md#v3) などで確認されている。
この operator を含む実行計画は `MiniBatchAssign / MiniBatchKeyOrder / RowCount` の再現クエリと実行計画で確認できる。

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL|  | | | 入力となる Relation operator。 |

#### Minor Sort

(Undocumented)

ストリームの一部に対して ORDER BY の処理をする。Sort とほぼ同じだが、テーブルやインデックスとソート順の prefix が一致して全体の Sort が必要ない場合に使われる。

##### Metadata

| key | values | description |
|-----|--------|-------------|
| call_type | Local もしくは Global ||

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL|  | | | ソート対象の入力となる Relation operator。 |
|SCALAR    | MajorKey | Yes | Yes | ソートキーのうち、入力でソート済な部分が順に指定される。 |
|SCALAR    | MinorKey | Yes | Yes | ソートキーのうち、入力でソートされていない部分が順に指定される。 |
|SCALAR    | Value | Yes | Yes | ソートキー以外で取り出す列が順に指定される。 |

{{< details summary="Minor Sort の再現クエリと実行計画" >}}

`ORDER BY` が先頭キーの順序とは部分的に合っているが、残りのキーで追加の局所的な sort が必要になる例:

```sql
SELECT SingerId, AlbumTitle
FROM Albums
ORDER BY SingerId, AlbumTitle;
```

```text
+----+----------------------------------------------------------------------------+
| ID | Operator                                                                   |
+----+----------------------------------------------------------------------------+
|  0 | Distributed Union on Singers <Row> (split_ranges_aligned)                  |
|  1 | +- Serialize Result <Row>                                                  |
|  2 |    +- Minor Sort <Row>                                                     |
|  3 |       +- Local Distributed Union <Row>                                     |
|  4 |          +- Table Scan on Albums <Row> (Full scan, scan_method: Automatic) |
+----+----------------------------------------------------------------------------+
```

`STREAM_GROUP` の入力順序を整える例:

```sql
SELECT SingerId, SongGenre
FROM Songs
GROUP@{GROUP_METHOD=STREAM_GROUP} BY SingerId, SongGenre;
```

```text
+----+------------------------------------------------------------------------------+
| ID | Operator                                                                     |
+----+------------------------------------------------------------------------------+
|  0 | Distributed Union on Singers <Row> (split_ranges_aligned)                    |
|  1 | +- Serialize Result <Row>                                                    |
|  2 |    +- Stream Aggregate <Row>                                                 |
|  3 |       +- Minor Sort <Row>                                                    |
|  4 |          +- Local Distributed Union <Row>                                    |
|  5 |             +- Table Scan on Songs <Row> (Full scan, scan_method: Automatic) |
+----+------------------------------------------------------------------------------+
```

{{< /details >}}

#### Minor Sort Limit

(Undocumented)

ORDER BY と LIMIT 両方の処理をする operator。Sort Limit とほぼ同じだが、テーブルやインデックスとソート順の prefix が一致して全体の Sort が必要ない場合に使われる。

##### Metadata

| key | values | description |
|-----|--------|-------------|
| call_type | Local もしくは Global ||

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL|  | | | ソート対象の入力 |
|SCALAR    | Limit |  | | 取得する行数 |
|SCALAR    | MajorKey | Yes | Yes | ソートキーのうち、入力でソート済な部分が順に指定される。 |
|SCALAR    | MinorKey | Yes | Yes | ソートキーのうち、入力でソートされていない部分が順に指定される。 |
|SCALAR    | Value | Yes | Yes | ソートキー以外で取り出す列が順に指定される。 |

{{< details summary="Minor Sort Limit の再現クエリと実行計画" >}}

```sql
SELECT SingerId, AlbumTitle
FROM Albums
WHERE SingerId > 0
ORDER BY SingerId, AlbumTitle
LIMIT 3;
```

```text
+----+-----------------------------------------------------------------+
| ID | Operator                                                        |
+----+-----------------------------------------------------------------+
|  0 | Global Limit <Row>                                              |
| *1 | +- Distributed Union on Singers <Row> (split_ranges_aligned)    |
|  2 |    +- Serialize Result <Row>                                    |
|  3 |       +- Local Minor Sort Limit <Row>                           |
|  4 |          +- Local Distributed Union <Row>                       |
|  5 |             +- Filter Scan <Row> (seekable_key_size: 1)         |
| *6 |                +- Table Scan on Albums <Row> (scan_method: Row) |
+----+-----------------------------------------------------------------+
Predicates(identified by ID):
 1: Split Range: ($SingerId > 0)
 6: Seek Condition: ($SingerId > 0)
```

{{< /details >}}


#### Random Id Assign

`TABLESAMPLE` を使用した際に現れる。 Filter operator と組み合わせることで、ランダムに割り当てた値を元にフィルタすることでサンプリングを実現する。

* https://docs.cloud.google.com/spanner/docs/query-operators-unary#random_id_assign

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL|  | | | 入力 |
|SCALAR    |  | Yes | | description が `<random id>` となる Reference を指す variable であり、後に Filter で名前が参照される。 |

{{< details summary="Random Id Assign の再現クエリと実行計画" >}}

```sql
SELECT s.SongName
FROM Songs AS s TABLESAMPLE BERNOULLI (10 PERCENT);
```

```text
+----+-------------------------------------------------------------------------------------------------------+
| ID | Operator                                                                                              |
+----+-------------------------------------------------------------------------------------------------------+
|  0 | Distributed Union on SongsBySingerAlbumSongNameDesc <Row>                                             |
|  1 | +- Serialize Result <Row>                                                                             |
| *2 |    +- Filter <Row>                                                                                    |
|  3 |       +- Random Id Assign <Row>                                                                       |
|  4 |          +- Local Distributed Union <Row>                                                             |
|  5 |             +- Index Scan on SongsBySingerAlbumSongNameDesc <Row> (Full scan, scan_method: Automatic) |
+----+-------------------------------------------------------------------------------------------------------+
Predicates(identified by ID):
 2: Condition: ($v1 < 900719925474099U)
```

{{< /details >}}

#### RowCount 

(Undocumented)

[Shard 最適化クエリ](https://github.com/gcpug/nouhau/blob/spanner/shard/spanner/note/shard/README.md#v3) などで確認されている。

##### ChildLinks

|kind      | type | variable | position | description |
|----------|-----|--------|---|-------------|
|RELATIONAL|  | | | 入力 |

{{< details summary="MiniBatchAssign / MiniBatchKeyOrder / RowCount の再現クエリと実行計画" >}}

この例の実行計画だけは Cloud Spanner の optimizer version 5 で出力したものである。Spanner Omni 2026.r1-beta では同じ SQL でもこれらの operator を含まない形になることがある。

```sql
@{OPTIMIZER_VERSION=5}
SELECT *
FROM Songs@{FORCE_INDEX=SongsBySongName}
ORDER BY SongName DESC
LIMIT 1;
```

```text
+-----+-------------------------------------------------------------------------------------------------+
| ID  | Operator <execution_method>                                                                     |
+-----+-------------------------------------------------------------------------------------------------+
|   0 | Global Limit <Row>                                                                              |
|  *1 | +- Distributed Cross Apply <Row> (order_preserving: true)                                       |
|   2 |    +- [Input] Create Batch <Row>                                                                |
|   3 |    |  +- Compute Struct <Row>                                                                   |
|   4 |    |     +- Local Limit <Row>                                                                   |
|   5 |    |        +- Distributed Union on SongsBySongName <Row> (preserve_subquery_order: true)       |
|   6 |    |           +- Local Sort Limit <Row>                                                        |
|   7 |    |              +- Local Distributed Union <Row>                                              |
|   8 |    |                 +- Index Scan on SongsBySongName <Row> (Full scan, scan_method: Automatic) |
|  28 |    +- [Map] Serialize Result <Row>                                                              |
|  29 |       +- MiniBatchKeyOrder <Row>                                                                |
|  30 |          +- Minor Sort Limit <Row>                                                              |
|  31 |             +- RowCount <Row>                                                                   |
|  32 |                +- Cross Apply <Row>                                                             |
|  33 |                   +- [Input] RowCount <Row>                                                     |
|  34 |                   |  +- KeyRangeAccumulator <Row>                                               |
|  35 |                   |     +- Local Minor Sort <Row>                                               |
|  36 |                   |        +- MiniBatchAssign <Row>                                             |
|  37 |                   |           +- Batch Scan on $v2 <Row> (scan_method: Row)                     |
|  53 |                   +- [Map] Local Distributed Union <Row>                                        |
|  54 |                      +- Filter Scan <Row> (seekable_key_size: 0)                                |
| *55 |                         +- Table Scan on Songs <Row> (scan_method: Row)                         |
+-----+-------------------------------------------------------------------------------------------------+
Predicates(identified by ID):
  1: Split Range: (($SingerId' = $sort_SingerId) AND ($AlbumId' = $sort_AlbumId) AND ($TrackId' = $sort_TrackId))
 55: Seek Condition: (($SingerId' = $sort_batched_SingerId) AND ($AlbumId' = $sort_batched_AlbumId) AND ($TrackId' = $sort_batched_TrackId))
```

{{< /details >}}

#### RowToDataBlock

RowToDataBlock は行指向の relational stream を batch/data-block 形式に変換する。
DataBlockToRow と対になって、Distributed Cross Apply、Push Broadcast Hash Join、Graph query などで remote execution や batch execution に渡す入力を作る箇所に現れる。
公式ドキュメント上の operator 名は `RowToDataBlockAdapter` である。

* https://docs.cloud.google.com/spanner/docs/query-operators-unary#rowtodatablockadapter

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL| | | | 行指向の入力 |

{{< details summary="RowToDataBlock の再現 SQL" >}}

以下は該当 operator を観測できる再現 SQL の例である。
同じクエリの実行計画は `DataBlockToRow / RowToDataBlock` の再現クエリと実行計画で示している。

```sql
SELECT s.SongName, s.Duration
FROM Songs@{FORCE_INDEX=SongsBySongName} AS s
WHERE STARTS_WITH(s.SongName, "B");
```

{{< /details >}}

#### Serialize Result

最終的に ResultSet に含まれる値を組み立てる。これよりも上の operator で row の値を操作することはない。Compute Struct の特殊なケースであることが公式ドキュメントでも説明されている通り、同様の構造を持つ。

* https://docs.cloud.google.com/spanner/docs/query-operators-unary#serialize_result

##### ChildLinks 

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL|  | | | 入力 |
|SCALAR    |  | | Yes | `metadata.rowType.fields` に現れる順で対応する式を表現する |
|SCALAR    | Scalar | | Yes | 式で参照される Scalar Subquery(or Array Subquery) を指す。 |

{{< details summary="Serialize Result の再現クエリと実行計画" >}}

```sql
SELECT s.SongName
FROM Songs AS s;
```

```text
+----+-------------------------------------------------------------------------------------------------+
| ID | Operator                                                                                        |
+----+-------------------------------------------------------------------------------------------------+
|  0 | Distributed Union on SongsBySingerAlbumSongNameDesc <Row>                                       |
|  1 | +- Local Distributed Union <Row>                                                                |
|  2 |    +- Serialize Result <Row>                                                                    |
|  3 |       +- Index Scan on SongsBySingerAlbumSongNameDesc <Row> (Full scan, scan_method: Automatic) |
+----+-------------------------------------------------------------------------------------------------+
```

{{< /details >}}

#### Sort

`ORDER BY` によるソートのみをする operator。Sort Limit とほぼ同じだが、 `LIMIT` を設定しない場合はこちらになる。

* https://docs.cloud.google.com/spanner/docs/query-operators-unary#sort

##### ChildLinks 

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL|  | | | ソート対象の入力 |
|SCALAR    | Key | Yes | Yes | ソートキーとなる列が Reference で順に指定される。 |
|SCALAR    | Value | Yes | Yes | ソートキー以外で取り出す列が Reference で順に指定される。 |

{{< details summary="Sort の再現クエリと実行計画" >}}

```sql
SELECT s.SongGenre
FROM Songs AS s
ORDER BY SongGenre;
```

```text
+----+---------------------------------------------------------------------------+
| ID | Operator                                                                  |
+----+---------------------------------------------------------------------------+
|  0 | Distributed Union on Songs <Row> (preserve_subquery_order: true)          |
|  1 | +- Serialize Result <Row>                                                 |
|  2 |    +- Sort <Row>                                                          |
|  3 |       +- Local Distributed Union <Row>                                    |
|  4 |          +- Table Scan on Songs <Row> (Full scan, scan_method: Automatic) |
+----+---------------------------------------------------------------------------+
```

{{< /details >}}

#### Sort Limit

ORDER BY と LIMIT 両方の処理をする operator。Sort とほぼ同じだが、 `LIMIT` を使う場合はこちらになる。

* https://docs.cloud.google.com/spanner/docs/query-operators-unary#sort

##### Metadata

| key | values | description |
|-----|--------|-------------|
| call_type | Local もしくは Global ||

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL|  | | | ソート対象の入力 |
|SCALAR    | Limit |  | | 取得する行数 |
|SCALAR    | Offset |  | | 読み飛ばす行数 |
|SCALAR    | Key | Yes | Yes | ソートキーが順に指定される。 |
|SCALAR    | Value | Yes | Yes | ソートキー以外で取り出す列が順に指定される。 |

{{< details summary="Sort Limit の再現クエリと実行計画" >}}

```sql
SELECT s.SongGenre
FROM Songs AS s
ORDER BY SongGenre
LIMIT 3;
```

```text
+----+------------------------------------------------------------------------------+
| ID | Operator                                                                     |
+----+------------------------------------------------------------------------------+
|  0 | Global Limit <Row>                                                           |
|  1 | +- Distributed Union on Songs <Row> (preserve_subquery_order: true)          |
|  2 |    +- Serialize Result <Row>                                                 |
|  3 |       +- Local Sort Limit <Row>                                              |
|  4 |          +- Local Distributed Union <Row>                                    |
|  5 |             +- Table Scan on Songs <Row> (Full scan, scan_method: Automatic) |
+----+------------------------------------------------------------------------------+
```

{{< /details >}}

#### TVF

Table-valued function の入力を読み、指定された関数を適用して出力を生成する operator。
入力と同じ行数を返す mapping のほか、入力より多い行を返す generator や、入力より少ない行を返す filter としても動作し得る。
Change Stream のほか、Full Text Search の `SEARCH(...)` では `Search Query Conversion` という `TVF` が観測されることがある。この `TVF` は検索文字列を search index 用の query expression に変換し、その出力が `SearchIndex Scan` の `Search Predicate` から参照される。観測した `SEARCH_SUBSTRING(...)` の実行計画ではこの `TVF` は現れなかった。

* https://docs.cloud.google.com/spanner/docs/query-operators-unary#tvf

##### Metadata

| key | values | description |
|-----|--------|-------------|
| Name / name | | 呼び出す table-valued function の名前。観測例では `Search Query Conversion`。 |

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL| | | | TVF が評価する入力 relation。`Search Query Conversion` の観測例では `Unit Relation`。 |
|SCALAR    | | Yes または空 | | TVF の引数となる式。 |
|SCALAR    | Output | Yes | Yes | TVF が生成し、親から参照できる出力。 |
|SCALAR    | Referenced | | Yes | TVF が入力から参照する値。 |

{{< details summary="ChangeStream TVF の再現クエリと実行計画" >}}

必要な DDL:

```sql
CREATE TABLE Singers (
  SingerId INT64 NOT NULL,
  FirstName STRING(1024)
) PRIMARY KEY(SingerId);

CREATE CHANGE STREAM EverythingStream
FOR ALL;
```

再現 SQL:

```sql
SELECT ChangeRecord
FROM READ_EverythingStream (
  start_timestamp => TIMESTAMP "2026-05-06T00:00:00Z"
);
```

```text
+----+---------------------------+
| ID | Operator                  |
+----+---------------------------+
|  0 | Serialize Result <Row>    |
|  1 | +- ChangeStream TVF <Row> |
+----+---------------------------+
```

{{< /details >}}

{{< details summary="Search Query Conversion TVF の再現クエリと実行計画" >}}

必要な DDL:

```sql
CREATE TABLE SearchAlbums (
  SingerId INT64 NOT NULL,
  AlbumId STRING(MAX) NOT NULL,
  AlbumTitle STRING(MAX),
  AlbumTitle_Tokens TOKENLIST AS (TOKENIZE_FULLTEXT(AlbumTitle)) HIDDEN,
) PRIMARY KEY(SingerId, AlbumId);

CREATE SEARCH INDEX SearchAlbumsTitleIndex
ON SearchAlbums(AlbumTitle_Tokens);
```

再現 SQL:

```sql
SELECT AlbumId
FROM SearchAlbums
WHERE SEARCH(AlbumTitle_Tokens, "friday OR monday");
```

```text
+-----+---------------------------------------------------------------------------------+
| ID  | Operator                                                                        |
+-----+---------------------------------------------------------------------------------+
|   0 | Cross Apply <Row>                                                               |
|   1 | +- [Input] VerifyDeterminism <Row>                                              |
|   2 | |  +- TVF <Row> (Name: Search Query Conversion)                                 |
|   3 | |     +- Unit Relation <Row>                                                    |
|   7 | +- [Map] Distributed Union on _Search2aryIndex_SearchAlbumsTitleIndex <Row>     |
|   8 |    +- Local Distributed Union <Row>                                             |
|   9 |       +- Serialize Result <Row>                                                 |
| *10 |          +- SearchIndex Scan on SearchAlbumsTitleIndex <Row> (scan_method: Row) |
+-----+---------------------------------------------------------------------------------+
Predicates(identified by ID):
 10: Search Predicate: SQUERY(index_name:AlbumTitle_Tokens in SearchAlbumsTitleIndex predicate:$oo_tvf_0)
```

{{< /details >}}

#### SpoolBuild

(Undocumented)
`WITH` などによる一時テーブルを保存する。SpoolScan によって読み取られる。
通常の repeated CTE で観測した SpoolScan は、`Scan` operator に `metadata.scan_type: SpoolScan` が付く形ではなく、raw `PlanNode.display_name: SpoolScan` と `metadata.spool_name` を持つ独立した表示名として現れた。

##### Metadata

| key | values | description |
|-----|--------|-------------|
| spool_name | | 構築する spool の名前 |

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL|  | | | 保存対象の入力 |
|SCALAR    |  |  | | 一時テーブルの列 |

{{< details summary="SpoolBuild の再現クエリと実行計画" >}}

```sql
WITH CTE AS (
  SELECT 1 AS PK, "foo" AS col
)
SELECT *
FROM CTE c1
JOIN CTE c2 USING (PK);
```

```text
+-----+----------------------------------------------------+
| ID  | Operator                                           |
+-----+----------------------------------------------------+
|   0 | Serialize Result <Row>                             |
|   1 | +- Cross Apply <Row>                               |
|   2 |    +- [Input] SpoolBuild <Row> (spool_name: CTE)   |
|   3 |    |  +- Compute <Row>                             |
|   4 |    |     +- Unit Relation <Row>                    |
|  10 |    +- [Map] Cross Apply <Row>                      |
|  11 |       +- [Input] SpoolScan <Row> (spool_name: CTE) |
| *14 |       +- [Map] Filter <Row>                        |
|  15 |          +- SpoolScan <Row> (spool_name: CTE)      |
+-----+----------------------------------------------------+
Predicates(identified by ID):
 14: Condition: ($PK_2 = $PK_1)
```

{{< /details >}}

#### SpoolScan

(Undocumented)

`SpoolBuild` が保存した一時的な relation を読み取る operator。
通常の repeated CTE では raw `PlanNode.display_name: SpoolScan` として現れ、`Scan` の `scan_type` ではない。
Graph query の recursive plan で使われる `Recursive Spool Scan` とは別の operator 名である。
再現クエリと実行計画は直前の `SpoolBuild` 節を参照。

##### Metadata

| key | values | description |
|-----|--------|-------------|
| spool_name | | 読み取る spool の名前。対応する `SpoolBuild` と同じ値を持つ。 |

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|SCALAR    | | Yes または空 | | spool から読み出す値。 |

#### VerifyDeterminism

(Undocumented)

Full Text Search の `SEARCH(...)` で、`Search Query Conversion` という `TVF` の出力を `SearchIndex Scan` に渡す入力側に観測された operator。
観測例では `TVF` relation と、その出力を参照する scalar expression を子に持つ。
operator 名が示す検証の具体的な条件は公開 operator documentation では説明されていないため、決定性に関する一般的な保証としては扱わない。
観測した `SEARCH_SUBSTRING(...)` の実行計画では現れなかった。
再現クエリと実行計画は `TVF` 節の `Search Query Conversion TVF` を参照。

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL| | | | 観測例では `Search Query Conversion` TVF。 |
|SCALAR    | | | Yes | TVF の出力を参照する式。単一 token column の観測例では `$oo_tvf_0`。 |

#### Union Input

Union All operator のそれぞれの枝からの入力を揃えるための operator。

* https://docs.cloud.google.com/spanner/docs/query-operators-unary#union_input

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL|  |  | | それぞれの枝の本体 |
|SCALAR    | `input_{n}` |  | | Union All operator の結果の n 列目となる式 |

{{< details summary="Union Input の再現クエリと実行計画" >}}

```sql
SELECT 1 AS a, 2 AS b
UNION ALL SELECT 3 AS a, 4 AS b
UNION ALL SELECT 5 AS a, 6 AS b;
```

```text
+----+---------------------------------+
| ID | Operator                        |
+----+---------------------------------+
|  0 | Serialize Result <Row>          |
|  1 | +- Union All <Row>              |
|  2 |    +- Union Input               |
|  3 |    |  +- Compute <Row>          |
|  4 |    |     +- Unit Relation <Row> |
| 10 |    +- Union Input               |
| 11 |    |  +- Compute <Row>          |
| 12 |    |     +- Unit Relation <Row> |
| 18 |    +- Union Input               |
| 19 |       +- Compute <Row>          |
| 20 |          +- Unit Relation <Row> |
+----+---------------------------------+
```

{{< /details >}}

### Binary operators

Relational operator の子を2つ持つ Relational operator 群。

#### Anti-Semi Apply

replica 内にローカルな Anti Semi Apply Join を行う。
`NOT IN` や `NOT EXISTS` など、Input 側の行に対して Map 側に対応する行が存在しないことを判定する subquery predicate で現れる。
`BATCH_MODE=TRUE` では外側に `Distributed Anti Semi Apply` が現れ、その Map 側の内部でローカルな Anti-Semi Apply が使われることがある。
[Semi Apply](#semi-apply) と同様に、INTERLEAVE で同居するテーブルへの探索が相関キーへの seek で完結する `NOT EXISTS` では、hint なしでも standalone の Anti-Semi Apply が選ばれることを観測した。

* https://docs.cloud.google.com/spanner/docs/query-operators-binary#anti-semi-apply

##### ChildLinks

|kind      | type | variable | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL| (Input) | | | 駆動表となる入力側のサブツリー |
|RELATIONAL| Map | | | Input 側の値に応じて実行されるサブツリー |

{{< details summary="Anti-Semi Apply / Semi Apply の再現クエリと実行計画" >}}

Semi Apply:

```sql
@{JOIN_METHOD=APPLY_JOIN, BATCH_MODE=FALSE}
SELECT s.SingerId, s.FirstName
FROM Singers AS s
WHERE s.SingerId IN (SELECT a.SingerId FROM Albums AS a);
```

```text
+----+-------------------------------------------------------------------------------------+
| ID | Operator                                                                            |
+----+-------------------------------------------------------------------------------------+
|  0 | Distributed Union on Singers <Row> (split_ranges_aligned)                           |
|  1 | +- Local Distributed Union <Row>                                                    |
|  2 |    +- Serialize Result <Row>                                                        |
|  3 |       +- Semi Apply <Row>                                                           |
|  4 |          +- [Input] Table Scan on Singers <Row> (Full scan, scan_method: Automatic) |
|  7 |          +- [Map] Local Distributed Union <Row>                                     |
|  8 |             +- Filter Scan <Row> (seekable_key_size: 0)                             |
| *9 |                +- Table Scan on Albums <Row> (scan_method: Row)                     |
+----+-------------------------------------------------------------------------------------+
Predicates(identified by ID):
 9: Seek Condition: ($SingerId_1 = $SingerId)
```

Anti-Semi Apply:

```sql
@{JOIN_METHOD=APPLY_JOIN, BATCH_MODE=FALSE}
SELECT s.SingerId, s.FirstName
FROM Singers AS s
WHERE NOT EXISTS (SELECT 1 FROM Albums AS a WHERE a.SingerId = s.SingerId);
```

```text
+----+-------------------------------------------------------------------------------------+
| ID | Operator                                                                            |
+----+-------------------------------------------------------------------------------------+
|  0 | Distributed Union on Singers <Row> (split_ranges_aligned)                           |
|  1 | +- Local Distributed Union <Row>                                                    |
|  2 |    +- Serialize Result <Row>                                                        |
|  3 |       +- Anti-Semi Apply <Row>                                                      |
|  4 |          +- [Input] Table Scan on Singers <Row> (Full scan, scan_method: Automatic) |
|  7 |          +- [Map] Local Distributed Union <Row>                                     |
|  8 |             +- Filter Scan <Row> (seekable_key_size: 0)                             |
| *9 |                +- Table Scan on Albums <Row> (scan_method: Row)                     |
+----+-------------------------------------------------------------------------------------+
Predicates(identified by ID):
 9: Seek Condition: ($SingerId_1 = $SingerId)
```

{{< /details >}}

#### Cross Apply

replica 内にローカルな Apply Join を行う。Input 側の Relational operator から取り出した値を使って、対応する Map 側の Relational operator を実行することで JOIN を実現する。
主に Distributed Cross Apply の中で使われる場合と、 INTERLEAVE されたテーブル間の JOIN で使われる場合がある。
古い query plan examples では `JOIN_TYPE=APPLY_JOIN` という hint spelling も見られ、確認した実行計画では `JOIN_METHOD=APPLY_JOIN` と同様に Apply Join 系の plan shape になった。現行のドキュメントに合わせるなら `JOIN_METHOD=APPLY_JOIN` を使う。

* https://docs.cloud.google.com/spanner/docs/query-operators-binary#cross-apply

##### ChildLinks

|kind      | type | variable | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL| (Input) | | | いわゆる駆動表となる入力側のサブツリーであり、実際には type を持たないが Web UI やドキュメント等で Input と表示される。|
|RELATIONAL| Map |  | | Input 側の値に応じて実行されるサブツリー |

{{< details summary="Cross Apply の再現クエリと実行計画" >}}

```sql
SELECT si.FirstName,
       (
         SELECT so.SongName
         FROM Songs AS so
         WHERE so.SingerId = si.SingerId
         LIMIT 1
       )
FROM Singers AS si;
```

```text
+-----+-----------------------------------------------------------------------------------------------+
| ID  | Operator                                                                                      |
+-----+-----------------------------------------------------------------------------------------------+
|   0 | Distributed Union on Singers <Row> (split_ranges_aligned)                                     |
|   1 | +- Local Distributed Union <Row>                                                              |
|   2 |    +- Serialize Result <Row>                                                                  |
|   3 |       +- Cross Apply <Row>                                                                    |
|   4 |          +- [Input] Table Scan on Singers <Row> (Full scan, scan_method: Automatic)           |
|   7 |          +- [Map] Stream Aggregate <Row> (scalar_aggregate: true)                             |
|   8 |             +- Global Limit <Row>                                                             |
|   9 |                +- Local Distributed Union <Row>                                               |
|  10 |                   +- Filter Scan <Row> (seekable_key_size: 0)                                 |
| *11 |                      +- Index Scan on SongsBySingerAlbumSongNameDesc <Row> (scan_method: Row) |
+-----+-----------------------------------------------------------------------------------------------+
Predicates(identified by ID):
 11: Seek Condition: ($SingerId_1 = $SingerId)
```

{{< /details >}}

#### Semi Apply

replica 内にローカルな Semi Apply Join を行う。
`IN` や `EXISTS` など、Input 側の行に対して Map 側に対応する行が存在することを判定する subquery predicate で現れる。
`BATCH_MODE=TRUE` では外側に `Distributed Semi Apply` が現れ、その Map 側の内部でローカルな Semi Apply が使われることがある。
hint がなくても、INTERLEAVE で同じ split に同居するテーブル(または interleaved index)への探索が相関キーへの seek で完結する `EXISTS` では、`Distributed Semi Apply` を伴わない standalone の Semi Apply が選ばれることを観測した。後述の details に hint なしの再現クエリを含む。一方、似た形の相関 `EXISTS` でも `Distributed Semi Apply` が選ばれる場合があり、どちらになるかは optimizer の選択に依る。

* https://docs.cloud.google.com/spanner/docs/query-operators-binary#semi-apply

##### ChildLinks

|kind      | type | variable | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL| (Input) | | | 駆動表となる入力側のサブツリー |
|RELATIONAL| Map | | | Input 側の値に応じて実行されるサブツリー |

{{< details summary="Semi Apply の再現クエリと実行計画" >}}

```sql
@{JOIN_METHOD=APPLY_JOIN}
SELECT s.SingerId, s.FirstName
FROM Singers AS s
WHERE s.SingerId IN (
  SELECT a.SingerId
  FROM Albums AS a
);
```

```text
+----+-------------------------------------------------------------------------------------+
| ID | Operator                                                                            |
+----+-------------------------------------------------------------------------------------+
|  0 | Distributed Union on Singers <Row> (split_ranges_aligned)                           |
|  1 | +- Local Distributed Union <Row>                                                    |
|  2 |    +- Serialize Result <Row>                                                        |
|  3 |       +- Semi Apply <Row>                                                           |
|  4 |          +- [Input] Table Scan on Singers <Row> (Full scan, scan_method: Automatic) |
|  7 |          +- [Map] Local Distributed Union <Row>                                     |
|  8 |             +- Filter Scan <Row> (seekable_key_size: 0)                             |
| *9 |                +- Table Scan on Albums <Row> (scan_method: Row)                     |
+----+-------------------------------------------------------------------------------------+
Predicates(identified by ID):
 9: Seek Condition: ($SingerId_1 = $SingerId)
```

{{< /details >}}

{{< details summary="hint なしで standalone Semi Apply / Anti-Semi Apply が現れる再現クエリと実行計画" >}}

INTERLEAVE された `Songs` 側への探索が `Albums` の主キー全体への seek で完結する `EXISTS` では、hint なしで standalone の Semi Apply が現れる。Map 側は `INTERLEAVE IN Albums` された `SongsBySingerAlbumSongNameDesc` への Index Scan であり、同じ split 内で完結する。

```sql
SELECT a.AlbumTitle
FROM Albums AS a
WHERE EXISTS (SELECT 1 FROM Songs AS s WHERE s.SingerId = a.SingerId AND s.AlbumId = a.AlbumId);
```

```text
+-----+-----------------------------------------------------------------------------------------+
| ID  | Operator                                                                                |
+-----+-----------------------------------------------------------------------------------------+
|   0 | Distributed Union on Albums <Row> (split_ranges_aligned)                                |
|   1 | +- Local Distributed Union <Row>                                                        |
|   2 |    +- Serialize Result <Row>                                                            |
|   3 |       +- Semi Apply <Row>                                                               |
|   4 |          +- [Input] Table Scan on Albums <Row> (Full scan, scan_method: Automatic)      |
|   8 |          +- [Map] Local Distributed Union <Row>                                         |
|   9 |             +- Filter Scan <Row> (seekable_key_size: 0)                                 |
| *10 |                +- Index Scan on SongsBySingerAlbumSongNameDesc <Row> (scan_method: Row) |
+-----+-----------------------------------------------------------------------------------------+
Predicates(identified by ID):
 10: Seek Condition: (($SingerId_1 = $SingerId) AND ($AlbumId_1 = $AlbumId))
```

`NOT EXISTS` では同じ形の standalone Anti-Semi Apply が現れる。

```sql
SELECT a.AlbumTitle
FROM Albums AS a
WHERE NOT EXISTS (SELECT 1 FROM Songs AS s WHERE s.SingerId = a.SingerId AND s.AlbumId = a.AlbumId);
```

```text
+-----+-----------------------------------------------------------------------------------------+
| ID  | Operator                                                                                |
+-----+-----------------------------------------------------------------------------------------+
|   0 | Distributed Union on Albums <Row> (split_ranges_aligned)                                |
|   1 | +- Local Distributed Union <Row>                                                        |
|   2 |    +- Serialize Result <Row>                                                            |
|   3 |       +- Anti-Semi Apply <Row>                                                          |
|   4 |          +- [Input] Table Scan on Albums <Row> (Full scan, scan_method: Automatic)      |
|   8 |          +- [Map] Local Distributed Union <Row>                                         |
|   9 |             +- Filter Scan <Row> (seekable_key_size: 0)                                 |
| *10 |                +- Index Scan on SongsBySingerAlbumSongNameDesc <Row> (scan_method: Row) |
+-----+-----------------------------------------------------------------------------------------+
Predicates(identified by ID):
 10: Seek Condition: (($SingerId_1 = $SingerId) AND ($AlbumId_1 = $AlbumId))
```

{{< /details >}}

#### Outer Apply

replica 内にローカルな Outer Apply Join を行う。Input 側の Relational operator から取り出した値を使って、対応する Map 側の Relational operator を実行することで JOIN を実現する。
主に Distributed Outer Apply の中で使われる場合と、 INTERLEAVE されたテーブル間の JOIN で使われる場合がある。

* https://docs.cloud.google.com/spanner/docs/query-operators-binary#outer-apply

##### ChildLinks

|kind      | type | variable | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL| (Input) | | | いわゆる駆動表となる入力側のサブツリーであり、実際には type を持たないが Web UI やドキュメント等で Input と表示される。|
|RELATIONAL| Map |  | | Input 側の値に応じて実行されるサブツリー |
|SCALAR    | | Yes | * | 結合条件を満たさなかった時に Input 側から生成する行の定義 |

{{< details summary="Outer Apply の再現クエリと実行計画" >}}

```sql
SELECT a.AlbumTitle, s.SongName
FROM Albums AS a
LEFT JOIN@{JOIN_METHOD=APPLY_JOIN, BATCH_MODE=FALSE} Songs AS s
ON a.SingerId = s.SingerId AND a.AlbumId = s.AlbumId;
```

```text
+-----+-----------------------------------------------------------------------------------------+
| ID  | Operator                                                                                |
+-----+-----------------------------------------------------------------------------------------+
|   0 | Distributed Union on Albums <Row> (split_ranges_aligned)                                |
|   1 | +- Local Distributed Union <Row>                                                        |
|   2 |    +- Serialize Result <Row>                                                            |
|   3 |       +- Outer Apply <Row>                                                              |
|   4 |          +- [Input] Table Scan on Albums <Row> (Full scan, scan_method: Automatic)      |
|   8 |          +- [Map] Local Distributed Union <Row>                                         |
|   9 |             +- Filter Scan <Row> (seekable_key_size: 0)                                 |
| *10 |                +- Index Scan on SongsBySingerAlbumSongNameDesc <Row> (scan_method: Row) |
+-----+-----------------------------------------------------------------------------------------+
Predicates(identified by ID):
 10: Seek Condition: (($SingerId_1 = $SingerId) AND ($AlbumId_1 = $AlbumId))
```

{{< /details >}}

#### Hash Join

ハッシュ結合を行う。
Build 側の全 row を元にハッシュマップを構築してから Probe 側の各 row の値を使ってハッシュマップを引くことで Condition を評価して JOIN を行う。
subquery predicate に `JOIN_METHOD=HASH_JOIN` を指定した場合、通常の INNER/OUTER join だけでなく `IN`/`EXISTS`/`NOT IN`/`NOT EXISTS` 由来の semi/anti-semi 系にも `Hash Join` が使われ、`join_type` に `BUILD_SEMI` や `BUILD_ANTI_SEMI` が現れる。
`HASH_JOIN_BUILD_SIDE=BUILD_RIGHT` などで物理的な Build / Probe 側を反転させると、LEFT OUTER JOIN で `PROBE_OUTER`、semi/anti-semi 系で `PROBE_SEMI` や `PROBE_ANTI_SEMI` が現れることがある。

* https://docs.cloud.google.com/spanner/docs/query-operators-binary#hash-join

##### Metadata

| key | values | description |
|-----|--------|-------------|
| join_type | INNER, BUILD_OUTER, PROBE_OUTER, BUILD_SEMI, BUILD_ANTI_SEMI, ... | INNER 以外は Build と Probe がどちらかで意味が変わるので、 BUILD か PROBE が prefix になる。|

##### ChildLinks

|kind      | type | variable | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL| Build |  | | 構築するハッシュマップ側になるサブツリー |
|RELATIONAL| Probe |  | | ハッシュマップに通す側のサブツリー |
|SCALAR    | Condition |  | | JOIN 条件のうち、ハッシュテーブルを使える等値の条件を表す Function |
|SCALAR    | Residual Condition |  | | JOIN 条件のうち、ハッシュテーブルを使えない非等値の条件を表す Function |
|SCALAR    | Build | Yes | Yes | Build 側からハッシュマップに含める列を指定 |
|SCALAR    | Probe | Yes | Yes | Probe 側から variable を定義 |

{{< details summary="Hash Join の join_type の再現クエリと実行計画" >}}

通常の Hash Join:

```sql
SELECT a.AlbumTitle, s.SongName
FROM Albums AS a
JOIN@{JOIN_METHOD=HASH_JOIN} Songs AS s
ON a.SingerId = s.SingerId AND a.AlbumId = s.AlbumId;
```

```text
+----+----------------------------------------------------------------------------------------------------+
| ID | Operator                                                                                           |
+----+----------------------------------------------------------------------------------------------------+
|  0 | Distributed Union on Albums <Row> (split_ranges_aligned)                                           |
|  1 | +- Serialize Result <Row>                                                                          |
| *2 |    +- Hash Join <Row> (join_type: INNER)                                                           |
|  3 |       +- [Build] Local Distributed Union <Row>                                                     |
|  4 |       |  +- Table Scan on Albums <Row> (Full scan, scan_method: Automatic)                         |
|  8 |       +- [Probe] Local Distributed Union <Row>                                                     |
|  9 |          +- Index Scan on SongsBySingerAlbumSongNameDesc <Row> (Full scan, scan_method: Automatic) |
+----+----------------------------------------------------------------------------------------------------+
Predicates(identified by ID):
 2: Condition: (($SingerId = $SingerId_1) AND ($AlbumId = $AlbumId_1))
```

semi 系:

```sql
@{JOIN_METHOD=HASH_JOIN}
SELECT s.SingerId, s.FirstName
FROM Singers AS s
WHERE s.SingerId IN (SELECT a.SingerId FROM Albums AS a);
```

```text
+----+-----------------------------------------------------------------------------+
| ID | Operator                                                                    |
+----+-----------------------------------------------------------------------------+
|  0 | Distributed Union on Singers <Row> (split_ranges_aligned)                   |
|  1 | +- Serialize Result <Row>                                                   |
| *2 |    +- Hash Join <Row> (join_type: BUILD_SEMI)                               |
|  3 |       +- [Build] Local Distributed Union <Row>                              |
|  4 |       |  +- Table Scan on Singers <Row> (Full scan, scan_method: Automatic) |
|  7 |       +- [Probe] Local Distributed Union <Row>                              |
|  8 |          +- Table Scan on Albums <Row> (Full scan, scan_method: Automatic)  |
+----+-----------------------------------------------------------------------------+
Predicates(identified by ID):
 2: Condition: ($SingerId = $SingerId_1)
```

anti-semi 系:

```sql
@{JOIN_METHOD=HASH_JOIN}
SELECT s.SingerId, s.FirstName
FROM Singers AS s
WHERE NOT EXISTS (SELECT 1 FROM Albums AS a WHERE a.SingerId = s.SingerId);
```

```text
+----+-----------------------------------------------------------------------------+
| ID | Operator                                                                    |
+----+-----------------------------------------------------------------------------+
|  0 | Distributed Union on Singers <Row> (split_ranges_aligned)                   |
|  1 | +- Serialize Result <Row>                                                   |
| *2 |    +- Hash Join <Row> (join_type: BUILD_ANTI_SEMI)                          |
|  3 |       +- [Build] Local Distributed Union <Row>                              |
|  4 |       |  +- Table Scan on Singers <Row> (Full scan, scan_method: Automatic) |
|  7 |       +- [Probe] Local Distributed Union <Row>                              |
|  8 |          +- Table Scan on Albums <Row> (Full scan, scan_method: Automatic)  |
+----+-----------------------------------------------------------------------------+
Predicates(identified by ID):
 2: Condition: ($SingerId_1 = $SingerId)
```

{{< /details >}}

#### Merge Join

ソート済みの入力同士を join 条件のキー順に突き合わせる join operator。
`JOIN@{JOIN_METHOD=MERGE_JOIN}` や subquery predicate に対する `JOIN_METHOD=MERGE_JOIN` で観測される。
入力が join key 順に利用できる場合はそのまま使われるが、必要な順序が揃っていない場合は入力側に `Sort` や `Minor Sort` が追加される。

* https://docs.cloud.google.com/spanner/docs/query-operators-binary#merge-join

##### Metadata

| key | values | description |
|-----|--------|-------------|
| join_configuration | ONE_TO_MANY, MANY_TO_MANY, ... | join key の重複可能性に基づく構成 |
| join_type | INNER, ... | join の種類 |

##### ChildLinks

|kind      | type | variable | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL| Left | | | 左辺の入力 |
|RELATIONAL| Right | | | 右辺の入力 |
|SCALAR    | Condition | | | JOIN 条件を表す Function |

{{< details summary="Merge Join の再現クエリと実行計画" >}}

入力の順序をそのまま使える例:

```sql
SELECT a.AlbumTitle, s.SongName
FROM Albums AS a
JOIN@{JOIN_METHOD=MERGE_JOIN} Songs AS s
ON a.SingerId = s.SingerId AND a.AlbumId = s.AlbumId;
```

```text
+----+----------------------------------------------------------------------------------------------------+
| ID | Operator                                                                                           |
+----+----------------------------------------------------------------------------------------------------+
|  0 | Distributed Union on Albums <Row> (split_ranges_aligned)                                           |
|  1 | +- Serialize Result <Row>                                                                          |
| *2 |    +- Merge Join <Row> (join_configuration: ONE_TO_MANY, join_type: INNER)                         |
|  3 |       +- [Left] Local Distributed Union <Row>                                                      |
|  4 |       |  +- Table Scan on Albums <Row> (Full scan, scan_method: Automatic)                         |
|  8 |       +- [Right] Local Distributed Union <Row>                                                     |
|  9 |          +- Index Scan on SongsBySingerAlbumSongNameDesc <Row> (Full scan, scan_method: Automatic) |
+----+----------------------------------------------------------------------------------------------------+
Predicates(identified by ID):
 2: Condition: (($SingerId = $SingerId_1) AND ($AlbumId = $AlbumId_1))
```

Sort が追加される例:

```sql
SELECT a.AlbumTitle, s.SongName
FROM Albums AS a
JOIN@{JOIN_METHOD=MERGE_JOIN} Songs AS s
ON a.AlbumId = s.AlbumId;
```

```text
+----+---------------------------------------------------------------------------------------------------------+
| ID | Operator                                                                                                |
+----+---------------------------------------------------------------------------------------------------------+
|  0 | Serialize Result <Row>                                                                                  |
| *1 | +- Merge Join <Row> (join_configuration: MANY_TO_MANY, join_type: INNER)                                |
|  2 |    +- [Left] Distributed Union on AlbumsByAlbumTitle <Row> (preserve_subquery_order: true)              |
|  3 |    |  +- Sort <Row>                                                                                     |
|  4 |    |     +- Local Distributed Union <Row>                                                               |
|  5 |    |        +- Index Scan on AlbumsByAlbumTitle <Row> (Full scan, scan_method: Automatic)               |
| 12 |    +- [Right] Distributed Union on SongsBySingerAlbumSongNameDesc <Row> (preserve_subquery_order: true) |
| 13 |       +- Sort <Row>                                                                                     |
| 14 |          +- Local Distributed Union <Row>                                                               |
| 15 |             +- Index Scan on SongsBySingerAlbumSongNameDesc <Row> (Full scan, scan_method: Automatic)   |
+----+---------------------------------------------------------------------------------------------------------+
Predicates(identified by ID):
  1: Condition: ($sort_AlbumId = $sort_AlbumId_1)
```

{{< /details >}}

#### Recursive Union

Graph query の recursive path などで、初期入力と再帰ステップの入力を結合しながら繰り返し評価する operator。
公式ドキュメントでは binary operator として分類されており、base case と recursive case の 2 つの入力を持つ。
再帰ステップ側では `Recursive Spool Scan` を通して前回までの中間結果を参照する。

* https://docs.cloud.google.com/spanner/docs/query-operators-binary#recursive-union

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL| | | | base case を表す入力 |
|RELATIONAL| | | | recursive case を表す入力 |

{{< details summary="Recursive Union / Recursive Spool Scan の再現クエリと実行計画" >}}

```sql
GRAPH MusicGraph
MATCH (singer:Singers {singerId:42})-[c:CollabWith]->{1,2}(featured:Singers)
RETURN singer.SingerId AS singer, featured.SingerId AS featured;
```

```text
+-----+-------------------------------------------------------------------------------------------+
| ID  | Operator                                                                                  |
+-----+-------------------------------------------------------------------------------------------+
|  *0 | Distributed Union on Singers <Row>                                                        |
|   1 | +- Serialize Result <Row>                                                                 |
|   2 |    +- DataBlockToRow                                                                      |
|   3 |       +- Recursive Union <Batch>                                                          |
|   4 |          +- Union Input                                                                   |
|   5 |          |  +- RowToDataBlock                                                             |
|   6 |          |     +- Local Distributed Union <Row>                                           |
|   7 |          |        +- Compute <Row>                                                        |
|   8 |          |           +- Filter Scan <Row> (seekable_key_size: 0)                          |
|  *9 |          |              +- Table Scan on Singers <Row> (scan_method: Row)                 |
|  18 |          +- Union Input                                                                   |
| *19 |             +- Distributed Cross Apply <Batch>                                            |
|  20 |                +- [Input] Create Batch <Batch>                                            |
| *21 |                |  +- Distributed Cross Apply <Batch>                                      |
|  22 |                |     +- [Input] Create Batch <Batch>                                      |
|  23 |                |     |  +- Recursive Spool Scan <Batch>                                   |
|  28 |                |     +- [Map] RowToDataBlock                                              |
|  29 |                |        +- Cross Apply <Row>                                              |
|  30 |                |           +- [Input] KeyRangeAccumulator <Row>                           |
|  31 |                |           |  +- DataBlockToRow                                           |
|  32 |                |           |     +- Batch Scan on $v26 <Batch> (scan_method: Batch)       |
|  37 |                |           +- [Map] Local Distributed Union <Row>                         |
|  38 |                |              +- Filter Scan <Row> (seekable_key_size: 0)                 |
| *39 |                |                 +- Table Scan on Collaborations <Row> (scan_method: Row) |
|  55 |                +- [Map] RowToDataBlock                                                    |
|  56 |                   +- Cross Apply <Row>                                                    |
|  57 |                      +- [Input] KeyRangeAccumulator <Row>                                 |
|  58 |                      |  +- DataBlockToRow                                                 |
|  59 |                      |     +- Batch Scan on $v28 <Batch> (scan_method: Batch)             |
|  64 |                      +- [Map] Local Distributed Union <Row>                               |
|  65 |                         +- Filter Scan <Row> (seekable_key_size: 0)                       |
| *66 |                            +- Table Scan on Singers <Row> (scan_method: Row)              |
+-----+-------------------------------------------------------------------------------------------+
Predicates(identified by ID):
  0: Split Range: ($SingerId'5 = 42)
  9: Seek Condition: ($SingerId'5 = 42)
 19: Split Range: ($SingerId_4'5 = $FeaturingSingerId'6)
 21: Split Range: ($SingerId_3'5 = $tail'_SingerId'16)
 39: Seek Condition: ($SingerId_3'5 = $batched_tail'_SingerId'17)
 66: Seek Condition: ($SingerId_4'5 = $batched_FeaturingSingerId'7)
```

{{< /details >}}

### N-ary operators

任意の数の Relational operator の子を持つ Relational operator 群。`Union All` 以外確認されていない。

#### Union All

`UNION ALL` を表現する operator で、任意の数の子の Union Input が返す行を合わせて返す。

* https://docs.cloud.google.com/spanner/docs/query-operators-n-ary#union_all

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL|  |  | Yes | UNION 対象を指す任意個数の Union Input operator |
|SCALAR    |  | Yes | Yes | Union All operator の結果の n 列目の名前を持ち、 `input_{n}` と対応付ける Scalar operator |

{{< details summary="Union All / Union Input の再現クエリと実行計画" >}}

```sql
SELECT 1 AS a, 2 AS b
UNION ALL SELECT 3 AS a, 4 AS b
UNION ALL SELECT 5 AS a, 6 AS b;
```

```text
+----+---------------------------------+
| ID | Operator                        |
+----+---------------------------------+
|  0 | Serialize Result <Row>          |
|  1 | +- Union All <Row>              |
|  2 |    +- Union Input               |
|  3 |    |  +- Compute <Row>          |
|  4 |    |     +- Unit Relation <Row> |
| 10 |    +- Union Input               |
| 11 |    |  +- Compute <Row>          |
| 12 |    |     +- Unit Relation <Row> |
| 18 |    +- Union Input               |
| 19 |       +- Compute <Row>          |
| 20 |          +- Unit Relation <Row> |
+----+---------------------------------+
```

{{< /details >}}

## Scalar operators

`kind` が `SCALAR` の operator であり、`ARRAY` を含む値として評価されるサブクエリや式などを含む。
子に Relational operator を持つ Subquery のみは例外として一般的な実行計画ツリーでも表示されるが、その他の Scalar operator は QueryPlan の生データを読むことで観測できるものであり、一般的な実行計画ツリーの可視化手法では表示されない。この節の「その他の Scalar operators」は、主に raw `PlanNode` の名称と child link の説明であり、同じ名前の行が実行計画ツリーに出ることを意味しない。

ツール製作者向け Note: Subquery であるかどうかは、さらに子を見なくても、親から `Scalar` `link_type` を使ってリンクされていることで判別できる。

### Subqueries

サブクエリは1つの Relational operator を子に持ち、 ARRAY やスカラに変換する Scalar operator として処理される。[Scalar subqueries](https://docs.cloud.google.com/spanner/docs/query-operators-scalar-subqueries) で説明されているように最適化の結果 Cross Apply などで実現されることもある。

#### Array Subquery

子のサブクエリと式から配列を計算する。

* https://docs.cloud.google.com/spanner/docs/query-operators-array-subqueries

##### Child Links

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL| | | | サブクエリとなるサブツリーで中で variable を定義する。 |
|SCALAR    | | | | サブクエリの中の variable を参照する式。サブクエリの各 row に対して配列の要素を計算するために使われる。 |

{{< details summary="Array Subquery の再現クエリと実行計画" >}}

```sql
SELECT FirstName,
       ARRAY(
         SELECT AS STRUCT song.SongName, song.SongGenre
         FROM Songs AS song
         WHERE song.SingerId = singer.SingerId
       )
FROM Singers AS singer
WHERE singer.SingerId = 1;
```

```text
+-----+-------------------------------------------------------------------------------------+
| ID  | Operator                                                                            |
+-----+-------------------------------------------------------------------------------------+
|   0 | Distributed Union on AlbumsByAlbumTitle <Row>                                       |
|   1 | +- Local Distributed Union <Row>                                                    |
|   2 |    +- Serialize Result <Row>                                                        |
|   3 |       +- Index Scan on AlbumsByAlbumTitle <Row> (Full scan, scan_method: Automatic) |
|   7 |       +- [Scalar] Array Subquery                                                    |
|  *8 |          +- Distributed Union on Concerts <Row>                                     |
|   9 |             +- Local Distributed Union <Row>                                        |
| *10 |                +- Filter Scan <Row> (seekable_key_size: 0)                          |
|  11 |                   +- Table Scan on Concerts <Row> (Full scan, scan_method: Row)     |
+-----+-------------------------------------------------------------------------------------+
Predicates(identified by ID):
  8: Split Range: ($SingerId_1 = $SingerId)
 10: Residual Condition: ($SingerId_1 = $SingerId)
```

{{< /details >}}

#### Scalar Subquery

子のサブクエリと式からスカラ値を計算する。

* https://docs.cloud.google.com/spanner/docs/query-operators-scalar-subqueries

##### Child Links

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|RELATIONAL| | | | サブクエリとなるサブツリーで、中で variable を定義する。 |
|SCALAR    | | | | サブクエリの中の variable を参照する。 |

{{< details summary="Scalar Subquery の再現クエリと実行計画" >}}

```sql
SELECT FirstName,
       IF(
         FirstName = 'Alice',
         (SELECT COUNT(*) FROM Songs WHERE Duration > 300),
         0
       )
FROM Singers;
```

```text
+-----+------------------------------------------------------------------------------------------+
| ID  | Operator                                                                                 |
+-----+------------------------------------------------------------------------------------------+
|   0 | Distributed Union on SingersByFirstLastName <Row>                                        |
|   1 | +- Local Distributed Union <Row>                                                         |
|   2 |    +- Serialize Result <Row>                                                             |
|   3 |       +- Index Scan on SingersByFirstLastName <Row> (Full scan, scan_method: Automatic)  |
|  10 |       +- [Scalar] Scalar Subquery                                                        |
|  11 |          +- Global Stream Aggregate <Row> (scalar_aggregate: true)                       |
|  12 |             +- Distributed Union on Songs <Row>                                          |
|  13 |                +- Local Stream Aggregate <Row> (scalar_aggregate: true)                  |
|  14 |                   +- Local Distributed Union <Row>                                       |
| *15 |                      +- Filter Scan <Row> (seekable_key_size: 0)                         |
|  16 |                         +- Table Scan on Songs <Row> (Full scan, scan_method: Automatic) |
+-----+------------------------------------------------------------------------------------------+
Predicates(identified by ID):
 15: Residual Condition: ($Duration > 300)
```

{{< /details >}}

### その他の Scalar operators

実行計画の内部表現としてはグラフ構造をなしているが、 Subquery と違って一般的に tree 表示の構造的な child としては表示されず、ドキュメンテーションもされていない。

#### Array Constructor

(Undocumented)
配列リテラルに対応し、配列値を表現する。
`shortRepresentation.description` は配列のリテラル表記となる。

##### Child Links

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|SCALAR    | | | Yes | 配列の各値を表現する式。 |

{{< details summary="Array Constructor の再現クエリと実行計画" >}}

```sql
SELECT a, b
FROM UNNEST([1, 2, 3]) a WITH OFFSET b;
```

```text
+----+------------------------+
| ID | Operator               |
+----+------------------------+
|  0 | Serialize Result <Row> |
|  1 | +- Array Unnest <Row>  |
+----+------------------------+
```

{{< /details >}}

#### Constant

(Undocumented)
定数を表す。`shortRepresentation.description` に値のリテラル表記や `<typed null>` などが文字列として入っている。

{{< details summary="Constant の再現クエリと実行計画" >}}

```sql
SELECT 1 + 2 AS Result;
```

```text
+----+------------------------+
| ID | Operator               |
+----+------------------------+
|  0 | Serialize Result <Row> |
|  1 | +- Unit Relation <Row> |
+----+------------------------+
```

{{< /details >}}

#### Field

STRUCT のフィールド参照を表す。

##### Metadata

| key | values | description |
|-----|--------|-------------|
| name | | 参照する列名 |

##### Child Links

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|SCALAR    | | | | 対象の STRUCT を指す式 |

{{< details summary="Field / Struct Constructor の再現クエリと実行計画" >}}

```sql
SELECT IF(TRUE, STRUCT(1 AS A, 1 AS B), STRUCT(2 AS A, 2 AS B)).A;
```

```text
+----+------------------------+
| ID | Operator               |
+----+------------------------+
|  0 | Serialize Result <Row> |
|  1 | +- Unit Relation <Row> |
+----+------------------------+
```

{{< /details >}}

#### Function

(Undocumented)
演算式と関数呼び出しを含む関数を表現する。`shortRepresentation.description` に演算子や関数名を含む式が文字列として入っている。
function hint の `DISABLE_INLINE` は relational plan shape に影響することがある。観測した例では default と `@{DISABLE_INLINE=FALSE}` は同じ operator shape になり、`@{DISABLE_INLINE=TRUE}` では `Compute` が追加されて `SHA512($SingerInfo)` を一度 materialize してから外側の式が参照する形になった。scalar-level の差分は tree 表示よりも nodes、JSON、YAML の出力で確認しやすい。

##### Child Links

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|SCALAR    | | | Yes | 各オペランド |

{{< details summary="Function の再現クエリと実行計画" >}}

```sql
SELECT 1 + 2 AS Result;
```

```text
+----+------------------------+
| ID | Operator               |
+----+------------------------+
|  0 | Serialize Result <Row> |
|  1 | +- Unit Relation <Row> |
+----+------------------------+
```

{{< /details >}}

{{< details summary="DISABLE_INLINE function hint の再現クエリと実行計画" >}}

Default:

```sql
SELECT
  SUBSTRING(CAST(x AS STRING), 2, 5) AS w,
  SUBSTRING(CAST(x AS STRING), 3, 7) AS y
FROM (
  SELECT SHA512(s.SingerInfo) AS x
  FROM Singers AS s
);
```

```text
+----+--------------------------------------------------------------------------+
| ID | Operator                                                                 |
+----+--------------------------------------------------------------------------+
|  0 | Distributed Union on Singers <Row>                                       |
|  1 | +- Local Distributed Union <Row>                                         |
|  2 |    +- Serialize Result <Row>                                             |
|  3 |       +- Table Scan on Singers <Row> (Full scan, scan_method: Automatic) |
+----+--------------------------------------------------------------------------+
```

`DISABLE_INLINE=FALSE`:

```sql
SELECT
  SUBSTRING(CAST(x AS STRING), 2, 5) AS w,
  SUBSTRING(CAST(x AS STRING), 3, 7) AS y
FROM (
  SELECT SHA512(s.SingerInfo) @{DISABLE_INLINE=FALSE} AS x
  FROM Singers AS s
);
```

```text
+----+--------------------------------------------------------------------------+
| ID | Operator                                                                 |
+----+--------------------------------------------------------------------------+
|  0 | Distributed Union on Singers <Row>                                       |
|  1 | +- Local Distributed Union <Row>                                         |
|  2 |    +- Serialize Result <Row>                                             |
|  3 |       +- Table Scan on Singers <Row> (Full scan, scan_method: Automatic) |
+----+--------------------------------------------------------------------------+
```

`DISABLE_INLINE=TRUE`:

```sql
SELECT
  SUBSTRING(CAST(x AS STRING), 2, 5) AS w,
  SUBSTRING(CAST(x AS STRING), 3, 7) AS y
FROM (
  SELECT SHA512(s.SingerInfo) @{DISABLE_INLINE=TRUE} AS x
  FROM Singers AS s
);
```

```text
+----+-----------------------------------------------------------------------------+
| ID | Operator                                                                    |
+----+-----------------------------------------------------------------------------+
|  0 | Distributed Union on Singers <Row>                                          |
|  1 | +- Local Distributed Union <Row>                                            |
|  2 |    +- Serialize Result <Row>                                                |
|  3 |       +- Compute <Row>                                                      |
|  4 |          +- Table Scan on Singers <Row> (Full scan, scan_method: Automatic) |
+----+-----------------------------------------------------------------------------+
```

{{< /details >}}

#### Parameter

(Undocumented)
クエリパラメータに対応する Scalar operator であり、実行時に `Statement.params` の name metadata と一致する名前の値として評価される。

通常はパラメータはスカラ値である前提でクエリが評価されるので、 [Working with STRUCT objects](https://cloud.google.com/spanner/docs/structs?hl=en) にあるような配列や構造体型のパラメータを使うクエリは `param_types` に型を渡さないとエラーとなり、実行計画の取得はできない点に注意する必要がある。

##### Metadata

| key | values | description |
|-----|--------|-------------|
| name |  | パラメータ名 |
| type | array, scalar, ... | クエリパラメータが配列かスカラ値かを示す。 `STRUCT` 型の値も `scalar` となる。 |

{{< details summary="Parameter の再現 SQL" >}}

以下は該当 operator を観測できる再現 SQL の例である。この形では `@singer_id` の型は `Singers.SingerId` との比較から `INT64` として推論できるため、値や params を渡さずに `PLAN` できる。
`Parameter` は scalar operator であり、QueryPlan の生データを読むことで観測できるが、一般的な実行計画ツリーの可視化手法では単独行として表示されない。

```sql
SELECT s.LastName
FROM Singers AS s
WHERE s.SingerId = @singer_id;
```

{{< /details >}}

#### Proto Constructor

(Undocumented)

Protocol Buffers message を field expression から構築する scalar operator。
descriptor と proto bundle を登録した環境で、`NEW proto { ... }`、`NEW proto(...)`、nested `SELECT AS proto` など、構築結果が constant folding されない形で観測された。
観測した 4 個の出現では metadata を持たず、optimizer version 1 から 8 まで同じ operator と ChildLinks の構成になった。
一般的な tree 表示では独立した行として表示されない。

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|SCALAR    | | | Yes | proto の各 field に設定する式。field number と式の対応は `shortRepresentation.description` に現れる。 |

{{< details summary="Proto Constructor の再現 SQL" >}}

以下は `examples.shipping.Order` の descriptor と proto bundle、および `Orders(Id INT64)` を登録した環境で観測した例である。

```sql
SELECT order_value.order_number
FROM (
  SELECT NEW `examples.shipping.Order` {
    order_number: CAST(Id AS STRING)
    date: Id
  } AS order_value
  FROM Orders
);
```

{{< /details >}}

#### Reference

(Undocumented)
`shortRepresentation.description` に名前を持つ参照で metadata も子も持たない。
Sort 系の operator の Key で降順の場合は `shortRepresentation.description` に `$ItemId (DESC)` のように `(DESC)` が含まれる。 

{{< details summary="Reference の再現クエリと実行計画" >}}

```sql
SELECT s.SongGenre
FROM Songs AS s
ORDER BY SongGenre;
```

```text
+----+---------------------------------------------------------------------------+
| ID | Operator                                                                  |
+----+---------------------------------------------------------------------------+
|  0 | Distributed Union on Songs <Row> (preserve_subquery_order: true)          |
|  1 | +- Serialize Result <Row>                                                 |
|  2 |    +- Sort <Row>                                                          |
|  3 |       +- Local Distributed Union <Row>                                    |
|  4 |          +- Table Scan on Songs <Row> (Full scan, scan_method: Automatic) |
+----+---------------------------------------------------------------------------+
```

{{< /details >}}

#### Search Predicate

(Undocumented)

Search index scan に渡す検索条件を表す scalar operator。
親の `Scan` から `type: Search Predicate` の child link で参照される。
単純な条件では child node 自体の `display_name` が `Search Predicate` になり、検索文字列や `Search Query Conversion` TVF の出力を参照する scalar child を 1 個持つ。
複数列の AND / OR のような合成条件では、親の child link type は同じでも直接の child node が `Function` となり、その子孫に複数の `Search Predicate` が現れる場合がある。
詳細な再現クエリと表示例は `Scan` 節の Full Text Search の説明を参照。

##### ChildLinks

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|SCALAR    | | | | 検索条件の入力。観測例では `Reference` または検索 expression。 |

#### Struct Constructor

`shortRepresentation.description` は `Struct Constructor {FirstName:DECODE_STRUCT_FIELD(0, $v1);LastName:DECODE_STRUCT_FIELD(1, $v1)}` のように、フィールド名とフィールドの式を列挙する形式となる。
フィールド値の式は `childLinks` で参照できるがフィールド名は `shortRepresentation.description` にしか含まれない。

* https://docs.cloud.google.com/spanner/docs/query-operators-struct-constructor

##### Child Links

|kind      | type | variable? | multiple? | description |
|----------|-----|--------|---|-------------|
|SCALAR    | | | Yes | 各フィールド値 |

{{< details summary="Struct Constructor の再現クエリと実行計画" >}}

```sql
SELECT IF(TRUE, STRUCT(1 AS A, 1 AS B), STRUCT(2 AS A, 2 AS B)).A;
```

```text
+----+------------------------+
| ID | Operator               |
+----+------------------------+
|  0 | Serialize Result <Row> |
|  1 | +- Unit Relation <Row> |
+----+------------------------+
```

{{< /details >}}

## QueryPlan=PROFILE の構造

![Web UI 上でのプロファイル情報](../../static/images/basic-profile-webui.png)

上記のような実行統計は `QueryMode=PROFILE` でクエリを実行した際に付与される情報の一部をレンダリングしている。上部に表示されるクエリ全体の実行統計は [ResultSetStats.queryStats](https://cloud.google.com/spanner/docs/reference/rest/v1/ResultSetStats?hl=en), 下部の各 operator ごとに表示されている統計は [PlanNode.executionStats](https://cloud.google.com/spanner/docs/reference/rest/v1/ResultSetStats?hl=en#PlanNode) を元の情報としている。

{{< details summary="QueryPlan=PROFILE 時のレスポンスの YAML 表現からの抜粋" >}}

```yaml
stats:
  queryPlan:
    planNodes:
    # (中略)
      - childLinks:
          - childIndex: 6
            variable: SongName
        displayName: Scan
        executionStats:
          cpu_time:
            total: "0.7"
            unit: msecs
          deleted_rows:
            total: "0"
            unit: rows
          execution_summary:
            num_executions: "1"
          filesystem_delay_seconds:
            total: "4.04"
            unit: msecs
          filtered_rows:
            total: "0"
            unit: rows
          latency:
            total: "4.51"
            unit: msecs
          rows:
            total: "100"
            unit: rows
          scanned_rows:
            total: "100"
            unit: rows
        index: 5
        kind: RELATIONAL
        metadata:
          Full scan: "true"
          scan_target: SongsBySingerAlbumSongNameDesc
          scan_type: IndexScan
    # (中略)
  queryStats:
    bytes_returned: "1486"
    cpu_time: 2.23 msecs
    data_bytes_read: "55163"
    deleted_rows_scanned: "0"
    elapsed_time: 7.3 msecs
    filesystem_delay_seconds: 4.04 msecs
    optimizer_statistics_package: ""
    optimizer_version: "2"
    query_plan_creation_time: 1.3 msecs
    query_text: SELECT s.SongName FROM Songs AS s LIMIT 100
    remote_server_calls: 0/0
    rows_returned: "100"
    rows_scanned: "100"
    runtime_creation_time: 0 msecs
    statistics_load_time: 0
```

{{< /details >}}

含まれるプロファイル情報はほぼ何もドキュメンテーションされていないが、活用可能なものが多い。

### 既知のクエリ全体の実行統計

|key|example|description|
|---|---|---|
|bytes_returned| `1486`|最終的にクライアントに返る結果のバイト数|
|cpu_time| `2.23 msecs`| 使用した CPU 時間の合計で Web UI 上の CPU time に対応 |
|data_bytes_read| `55163`|読み出したバイト数|
|deleted_rows_scanned| `0`|スキャンされた削除済の行数(いわゆる tombstone によるもの？)|
|elapsed_time| `7.3 msecs`|総経過時間で Web UI 上の Total elapsed time に対応|
|filesystem_delay_seconds| `4.04 msecs`|スキャン時に発生したファイルシステム由来の待ち時間|
|optimizer_statistics_package| `""` |未リリース機能に関するもの|
|optimizer_version| `2`| クエリに利用された [optimizer version](https://docs.cloud.google.com/spanner/docs/query-optimizer/overview?hl=en#versioning) |
|query_plan_creation_time| `1.3 msecs`| クエリプランの作成に掛かった時間。[Life of query](https://cloud.google.com/spanner/docs/whitepapers/life-of-query?hl=en#caching) に書かれている処理を行う時間で、同一のものは[キャッシュ](https://cloud.google.com/spanner/docs/whitepapers/life-of-query?hl=en#caching)されるのでクエリパラメータの使用により軽減される。|
|query_text| `SELECT s.SongName FROM Songs AS s LIMIT 100`|統計の対象のクエリ本文|
|remote_server_calls| `0/0`| distributed operator で他のサーバを呼んだ数。分子が `num_executions` の合計、分母が `remote_calls.total` の合計？(未確定)|
|rows_returned| `100`|クライアントに返る結果の行数で Web UI 上の Rows returned に対応|
|rows_scanned| `100`|各 Scan が読み出した行数で Web UI 上の Rows scanned に対応 |
|runtime_creation_time| `0 msecs`|不明|
|statistics_load_time| `0`|不明|

### 既知の PlanNode ごとの実行統計

各 PlanNode は `executionStats` に実行統計を持つ。

|key|operator(例)|description|
|---|---|---|
|Disk Usage (KBytes)|Sort Limit, Sort, ...|一時保存するために使ったディスク容量|
|Disk Write Latency (msecs)|Sort Limit, Sort, ...|一時保存のためのディスクアクセスに使った時間|
|Peak Memory Usage (KBytes)|Hash Join, Sort Limit, Sort, ...|最大利用メモリ量|
|Rows Spooled|Sort Limit, Sort, ...|一時保存した行数|
|cpu_time||使用した CPU 時間|
|deleted_rows|Scan|Scan したが削除されていた行|
|filesystem_delay_seconds|Scan|ファイルシステム側で発生した遅れ|
|filtered_rows|Scan|スキャンしたが Residual Condition でフィルタされた行数|
|latency||オペレータの開始から終了までのレイテンシで、 `latency.mean` が Web UI 上の Latency として表示され、ツールチップに `latency.total`, `latency.std_deviation` も含まれる。 |
|remote_calls|Distributed Cross Apply, Distributed Union, ...|リモートコールの回数で呼び出し元と呼び出し先が同じ際はカウントされていない様子|
|rows||operator が最終的に返した行数で、 `rows.total` が Web UI 上の Rows returned として表示|
|scanned_rows|Scan|スキャンした行数|
|execution_summary.checkpoint_time||チェックポイント作成に必要とした時間|
|execution_summary.execution_end_timestamp||UNIX time による実行が終了したタイムスタンプ|
|execution_summary.execution_start_timestamp||UNIX time による実行が開始されたタイムスタンプ|
|execution_summary.num_checkpoints||チェックポイントを作成した回数|
|execution_summary.num_executions||このオペレータの実行回数で、 Web UI 上の Executions として表示|

#### executionStats の各値の構造

各 operator は複数回実行されるため、 executionStats に含まれる各 key に対応する値は統計値を持つ struct になっている。
例外として、`execution_summary` に含まれる統計値は全実行を通してのサマリとなっているため単に string の値を持つ。


|key|description|
|---|---|
|mean|平均値|
|std_deviation|標準偏差|
|total|合計値|
|unit|各値の単位|
|histogram|ヒストグラムのバケットを含む配列|
|histogram[*].count|バケットの範囲に入った実行の数|
|histogram[*].percentage|全実行内でのそのバケットに分類されるパーセント|
|histogram[*].lower_bound|バケットの下限値|
|histogram[*].upper_bound|バケットの上限値|
