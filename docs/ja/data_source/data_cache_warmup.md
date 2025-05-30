---
displayed_sidebar: docs
---

# Data Cache のウォームアップ

データレイク分析や共有データクラスタのシナリオでは、BIレポートや概念実証 (PoC) のパフォーマンステストなど、クエリに対する高いパフォーマンス要件があります。リモートデータをローカルData Cacheにロードすることで、同じデータを何度も取得する必要がなくなり、クエリの実行が大幅に高速化され、リソースの使用が最小限に抑えられます。

StarRocks v3.3 は、[Data Cache](./data_cache.md) の強化機能として、Data Cache Warmup 機能を導入しました。Data Cache は、データクエリ中にデータがキャッシュに書き込まれる、キャッシュを受動的に埋めるプロセスです。一方、Data Cache Warmup はキャッシュを能動的に埋めるプロセスであり、リモートストレージから必要なデータを事前に積極的に取得します。

## シナリオ

- Data Cacheに使用されるディスクのストレージ容量が、ウォームアップするデータ量よりもはるかに大きい場合。ディスク容量がウォームアップするデータ量より少ない場合、期待されるウォームアップ効果は得られません。例えば、100 GB のデータをウォームアップする必要があるが、ディスクに 50 GB のスペースしかない場合、50 GB のデータしかキャッシュにロードできず、以前にロードされた 50 GB のデータは後でロードされた 50 GB のデータによって置き換えられます。
- Data Cacheに使用されるディスクへのデータアクセスが比較的安定している場合。アクセス量が急増すると、期待されるウォームアップ効果は得られません。例えば、100 GB のデータをウォームアップする必要があり、ディスクに 200 GB のスペースがある場合、最初の条件は満たされます。しかし、ウォームアッププロセス中に大量の新しいデータ (150 GB) がキャッシュに書き込まれたり、予期しない大規模なコールドクエリが 150 GB のデータをキャッシュにロードする必要がある場合、ウォームアップされたデータが追い出される可能性があります。

## 動作の仕組み

StarRocks は、Data Cache Warmup を実装するために CACHE SELECT 構文を提供します。CACHE SELECT を使用する前に、Data Cache 機能が有効になっていることを確認してください。

CACHE SELECT の構文:

```sql
CACHE SELECT <column_name> [, ...]
FROM [<catalog_name>.][<db_name>.]<table_name> [WHERE <boolean_expression>]
[PROPERTIES("verbose"="true")]
```

パラメータ:

- `column_name`: 取得する列。外部テーブルのすべての列を取得するには `*` を使用できます。
- `catalog_name`: データレイク内の外部テーブルをクエリする場合にのみ必要な外部カタログの名前。SET CATALOG を使用して外部カタログに切り替えた場合、指定する必要はありません。
- `db_name`: データベースの名前。そのデータベースに切り替えた場合、指定する必要はありません。
- `table_name`: データを取得するテーブルの名前。
- `boolean_expression`: フィルター条件。
- `PROPERTIES`: 現在、`verbose` プロパティのみがサポートされています。詳細なウォームアップメトリクスを返すために使用されます。

CACHE SELECT は同期プロセスであり、一度に 1 つのテーブルのみをウォームアップできます。実行が成功すると、ウォームアップ関連のメトリクスが返されます。

### 外部テーブルのすべてのデータをウォームアップする

次の例では、外部テーブル `lineitem` からすべてのデータをロードします:

```plaintext
mysql> cache select * from hive_catalog.test_db.lineitem;
+-----------------+------------------+----------------------+-------------------+
| READ_CACHE_SIZE | WRITE_CACHE_SIZE | AVG_WRITE_CACHE_TIME | TOTAL_CACHE_USAGE |
+-----------------+------------------+----------------------+-------------------+
| 48.2MB          | 3.7GB            | 59ms                 | 96.83%            |
+-----------------+------------------+----------------------+-------------------+
1 row in set (19.56 sec)
```

返されるフィールド:

- `READ_CACHE_SIZE`: すべてのノードによってData Cacheから読み取られたデータの合計サイズ。
- `WRITE_CACHE_SIZE`: すべてのノードによってData Cacheに書き込まれたデータの合計サイズ。
- `AVG_WRITE_CACHE_TIME`: 各ノードがData Cacheにデータを書き込むのにかかる平均時間。
- `TOTAL_CACHE_USAGE`: このウォームアップタスクが完了した後のクラスタ全体のData Cacheのスペース使用量。このメトリクスは、Data Cacheに十分なスペースがあるかどうかを評価するために使用できます。

### フィルター条件を指定して特定の列をウォームアップする

列と述語を指定して、細かいウォームアップを実現できます。これにより、ウォームアップするデータ量を減らし、ディスク I/O と CPU 消費を削減できます。

```plaintext
mysql> cache select l_orderkey from hive_catalog.test_db.lineitem where l_shipdate='1994-10-28';
+-----------------+------------------+----------------------+-------------------+
| READ_CACHE_SIZE | WRITE_CACHE_SIZE | AVG_WRITE_CACHE_TIME | TOTAL_CACHE_USAGE |
+-----------------+------------------+----------------------+-------------------+
| 957MB           | 713.5MB          | 3.6ms                | 97.33%            |
+-----------------+------------------+----------------------+-------------------+
1 row in set (9.07 sec)
```

次の例では、共有データクラスタ内のクラウドネイティブテーブル `lineorder` から特定の列を事前取得します:

```plaintext
mysql> cache select lo_orderkey from ssb.lineorder;
+-----------------+------------------+----------------------+-------------------+
| READ_CACHE_SIZE | WRITE_CACHE_SIZE | AVG_WRITE_CACHE_TIME | TOTAL_CACHE_USAGE |
+-----------------+------------------+----------------------+-------------------+
| 118MB           | 558.9MB          | 200.6ms              | 4.66%             |
+-----------------+------------------+----------------------+-------------------+
1 row in set (29.88 sec)
```

### 詳細モードでウォームアップする

デフォルトでは、`CACHE SELECT` によって返されるメトリクスは、複数の BEs にわたるメトリクスです。CACHE SELECT の末尾に `PROPERTIES("verbose"="true")` を追加して、各 BE の詳細なメトリクスを取得できます。

```plaintext
mysql> cache select * from hive_catalog.test_db.lineitem properties("verbose"="true");
+---------------+-----------------+---------------------+------------------+----------------------+-------------------+
| IP            | READ_CACHE_SIZE | AVG_READ_CACHE_TIME | WRITE_CACHE_SIZE | AVG_WRITE_CACHE_TIME | TOTAL_CACHE_USAGE |
+---------------+-----------------+---------------------+------------------+----------------------+-------------------+
| 172.26.80.233 | 376MB           | 127.8micros         | 0B               | 0s                   | 3.85%             |
| 172.26.80.231 | 272.5MB         | 121.8micros         | 20.7MB           | 146.5micros          | 3.91%             |
| 172.26.80.232 | 355.5MB         | 147.7micros         | 0B               | 0s                   | 3.91%             |
+---------------+-----------------+---------------------+------------------+----------------------+-------------------+
3 rows in set (0.54 sec)
```

詳細モードでは、追加のメトリクスが返されます:

- `AVG_READ_CACHE_TIME`: Data Cacheがヒットしたときに各ノードがデータを読み取るのにかかる平均時間。

## CACHE SELECT タスクの定期スケジューリング

CACHE SELECT を [SUBMIT TASK](../sql-reference/sql-statements/loading_unloading/ETL/SUBMIT_TASK.md) と組み合わせて、定期的なウォームアップを実現できます。例えば、次のケースでは、`lineitem` テーブルを 5 分ごとにウォームアップします:

```plaintext
mysql> submit task always_cache schedule every(interval 5 minute) as cache select l_orderkey
from hive_catalog.test_db.lineitem
where l_shipdate='1994-10-28';
+--------------+-----------+
| TaskName     | Status    |
+--------------+-----------+
| always_cache | SUBMITTED |
+--------------+-----------+
1 row in set (0.03 sec)
```

### CACHE SELECT タスクの管理

#### 作成されたタスクを表示する

```plaintext
mysql> select * from default_catalog.information_schema.tasks;
+--------------+---------------------+-----------------------------------------------------+---------------+------------------------------+---------------------------------------------------------------------+---------------------+------------+
| TASK_NAME    | CREATE_TIME         | SCHEDULE                                            | CATALOG       | DATABASE                     | DEFINITION                                                          | EXPIRE_TIME         | PROPERTIES |
+--------------+---------------------+-----------------------------------------------------+---------------+------------------------------+---------------------------------------------------------------------+---------------------+------------+
| always_cache | 2024-04-11 16:01:00 | PERIODICAL START(2024-04-11T16:01) EVERY(5 MINUTES) | emr_hive_test | zz_tpch_sf1000_hive_orc_zlib | cache select l_orderkey from lineitem where l_shipdate='1994-10-28' | NULL                |            |
+--------------+---------------------+-----------------------------------------------------+---------------+------------------------------+---------------------------------------------------------------------+---------------------+------------+
1 row in set (0.21 sec)
```

#### タスク実行履歴を表示する

```plaintext
mysql> select * from default_catalog.information_schema.task_runs;
+--------------------------------------+--------------+---------------------+---------------------+---------+---------------+------------------------------+---------------------------------------------------------------------+---------------------+------------+---------------+----------+------------------------------------------------------------------------------------------------------------------------+------------+
| QUERY_ID                             | TASK_NAME    | CREATE_TIME         | FINISH_TIME         | STATE   | CATALOG       | DATABASE                     | DEFINITION                                                          | EXPIRE_TIME         | ERROR_CODE | ERROR_MESSAGE | PROGRESS | EXTRA_MESSAGE                                                                                                          | PROPERTIES |
+--------------------------------------+--------------+---------------------+---------------------+---------+---------------+------------------------------+---------------------------------------------------------------------+---------------------+------------+---------------+----------+------------------------------------------------------------------------------------------------------------------------+------------+
| 55b30204-f7da-11ee-b03e-7ea526d0b618 | always_cache | 2024-04-11 16:06:00 | 2024-04-11 16:07:22 | SUCCESS | emr_hive_test | zz_tpch_sf1000_hive_orc_zlib | cache select l_orderkey from lineitem where l_shipdate='1994-10-28' | 2024-04-12 16:06:00 |          0 | NULL          | 100%     | AlreadyCachedSize: 15.7GB, AvgReadCacheTime: 1ms, WriteCacheSize: 0B, AvgWriteCacheTime: 0s, TotalCacheUsage: 75.94%   |            |
| a2e3dc7e-f7d9-11ee-b03e-7ea526d0b618 | always_cache | 2024-04-11 16:01:00 | 2024-04-11 16:02:39 | SUCCESS | emr_hive_test | zz_tpch_sf1000_hive_orc_zlib | cache select l_orderkey from lineitem where l_shipdate='1994-10-28' | 2024-04-12 16:01:00 |          0 | NULL          | 100%     | AlreadyCachedSize: 15.7GB, AvgReadCacheTime: 1.2ms, WriteCacheSize: 0B, AvgWriteCacheTime: 0s, TotalCacheUsage: 75.87% |            |
+--------------------------------------+--------------+---------------------+---------------------+---------+---------------+------------------------------+---------------------------------------------------------------------+---------------------+------------+---------------+----------+------------------------------------------------------------------------------------------------------------------------+------------+
2 rows in set (0.04 sec)
```

`EXTRA_MESSAGE` フィールドには、CACHE SELECT のメトリクスが記録されます。

#### タスクを削除する

```sql
DROP TASK <task_name>
```

## 使用例

1. PoC パフォーマンステスト中に、外部ストレージシステムの干渉なしに StarRocks のパフォーマンスを評価したい場合、CACHE SELECT ステートメントを使用して、テストするテーブルのデータを事前にData Cacheにロードできます。

2. ビジネスチームが毎朝 8 時に BI レポートを確認する必要がある場合、比較的安定したクエリパフォーマンスを確保するために、毎日 7 時に CACHE SELECT タスクをスケジュールして実行を開始できます。

   ```sql
   mysql> submit task BI schedule START('2024-02-03 07:00:00') EVERY(interval 1 day)
   AS cache select * from hive_catalog.test_db.lineitem
   where l_shipdate='1994-10-28';
   +--------------+-----------+
   | TaskName     | Status    |
   +--------------+-----------+
   | BI           | SUBMITTED |
   +--------------+-----------+
   1 row in set (0.03 sec)
   ```

3. ウォームアップ中のシステムリソース消費を最小限に抑えるために、SUBMIT TASK ステートメントでセッション変数を指定できます。例えば、CACHE SELECT タスクのためにリソースグループを指定し、並行性 (DOP) を調整し、WHERE でフィルター条件を指定して、ウォームアップが通常のクエリに与える影響を軽減できます。

   ```sql
   mysql> submit task cache_select properties("pipeline_dop"="1", "resource_group"="warmup") schedule EVERY(interval 1 day)
   AS cache select * from hive_catalog.test_db.lineitem where l_shipdate>='1994-10-28';
   +--------------+-----------+
   | TaskName     | Status    |
   +--------------+-----------+
   | cache_select | SUBMITTED |
   +--------------+-----------+
   1 row in set (0.03 sec)
   ```

## 制限と使用上の注意

- CACHE SELECT を使用するには、まず Data Cache 機能を有効にし、対象テーブルに対する SELECT 権限を持っている必要があります。
- CACHE SELECT は単一のテーブルのみをウォームアップすることをサポートし、ORDER BY、LIMIT、GROUP BY などの演算子はサポートしていません。
- CACHE SELECT は、共有なしクラスタと共有データクラスタの両方で使用できます。
- CACHE SELECT は、リモートの TEXT、ORC、Parquet ファイルをウォームアップできます。
- CACHE SELECT によってウォームアップされたデータは、キャッシュに永遠に保持されるわけではありません。キャッシュされたデータは、Data Cache 機能の LRU ルールに基づいて追い出される可能性があります。
  - データレイクユーザーの場合、`SHOW BACKENDS\G` または `SHOW COMPUTE NODES\G` を使用してData Cacheの残り容量を確認し、LRU 追い出しが発生する可能性があるかどうかを評価できます。
  - 共有データクラスタユーザーの場合、共有データクラスタのメトリクスを表示してData Cacheの使用状況を確認できます。
- 現在、CACHE SELECT の実装は INSERT INTO BLACKHOLE() アプローチを使用しており、通常のクエリプロセスに従ってテーブルをウォームアップします。そのため、CACHE SELECT のパフォーマンスオーバーヘッドは通常のクエリと同様です。将来的には、パフォーマンスを向上させるための改善が行われる予定です。

## 将来のバージョンでの期待

将来的には、StarRocks は適応型 Data Cache Warmup を導入し、より高いキャッシュヒット率を確保する予定です。