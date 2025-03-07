---
displayed_sidebar: docs
---

# SHOW EXPORT

## 説明

指定した条件を満たすエクスポートジョブの実行情報をクエリします。

## 構文

```SQL
SHOW EXPORT
[ FROM <db_name> ]
[
WHERE
    [ QUERYID = <query_id> ]
    [ STATE = { "PENDING" | "EXPORTING" | "FINISHED" | "CANCELLED" } ]
]
[ ORDER BY <field_name> [ ASC | DESC ] [, ... ] ]
[ LIMIT <count> ]
```

## パラメータ

このステートメントには、以下のオプションの句を含めることができます:

- FROM

  クエリしたいデータベースの名前を指定します。FROM句を指定しない場合、StarRocksは現在のデータベースをクエリします。

- WHERE

  エクスポートジョブをフィルタリングするための条件を指定します。指定した条件を満たすエクスポートジョブのみがクエリの結果セットに返されます。

  | **パラメータ** | **必須** | **説明**                                              |
  | ------------- | ------------ | ------------------------------------------------------------ |
  | QUERYID       | いいえ           | クエリしたいエクスポートジョブのID。このパラメータは、単一のエクスポートジョブの実行情報をクエリするために使用されます。 |
  | STATE         | いいえ           | クエリしたいエクスポートジョブのステータス。有効な値:<ul><li>`PENDING`: スケジュール待ちのエクスポートジョブをクエリします。</li><li>`EXPORTING`: 実行中のエクスポートジョブをクエリします。</li><li>`FINISHED`: 正常に完了したエクスポートジョブをクエリします。</li><li>`CANCELLED`: 失敗したエクスポートジョブをクエリします。</li></ul> |

- ORDER BY

  クエリの結果セットでエクスポートジョブレコードをソートするためのフィールド名を指定します。複数のフィールドを指定することができ、カンマ（`,`）で区切る必要があります。また、`ASC`または`DESC`キーワードを使用して、指定したフィールドに基づいてエクスポートジョブレコードを昇順または降順にソートすることを指定できます。

- LIMIT

  クエリの結果セットを指定した最大行数に制限します。有効な値: 正の整数。LIMIT句を指定しない場合、StarRocksは指定した条件を満たすすべてのエクスポートジョブを返します。

## 戻り結果

例えば、IDが`edee47f0-abe1-11ec-b9d1-00163e1e238f`のエクスポートジョブの実行情報をクエリします:

```SQL
SHOW EXPORT
WHERE QUERYID = "edee47f0-abe1-11ec-b9d1-00163e1e238f";
```

次の実行情報が返されます:

```SQL
     JobId: 14008
   QueryId: edee47f0-abe1-11ec-b9d1-00163e1e238f
     State: FINISHED
  Progress: 100%
  TaskInfo: {"partitions":["*"],"column separator":"\t","columns":["*"],"tablet num":10,"broker":"","coord num":1,"db":"db0","tbl":"tbl_simple","row delimiter":"\n","mem limit":2147483648}
      Path: hdfs://127.0.0.1:9000/users/230320/
CreateTime: 2023-03-20 11:16:14
 StartTime: 2023-03-20 11:16:17
FinishTime: 2023-03-20 11:16:26
   Timeout: 7200
```

戻り結果のパラメータは次の通りです:

- `JobId`: エクスポートジョブのID。
- `QueryId`: クエリのID。
- `State`: エクスポートジョブのステータス。

  有効な値:

  - `PENDING`: エクスポートジョブはスケジュール待ちです。
  - `EXPORTING`: エクスポートジョブは実行中です。
  - `FINISHED`: エクスポートジョブは正常に完了しました。
  - `CANCELLED`: エクスポートジョブは失敗しました。

- `Progress`: エクスポートジョブの進捗状況。進捗はクエリプランの単位で測定されます。例えば、エクスポートジョブが10のクエリプランに分割され、そのうち3つが完了した場合、エクスポートジョブの進捗は30%です。
- `TaskInfo`: エクスポートジョブの情報。

  情報は次のキーを含むJSONオブジェクトです:

  - `partitions`: エクスポートされたデータが存在するパーティション。ワイルドカード（`*`）がこのキーの値として返された場合、エクスポートジョブはすべてのパーティションからデータをエクスポートするために実行されます。
  - `column separator`: エクスポートされたデータファイルで使用されるカラムセパレータ。
  - `columns`: データがエクスポートされるカラムの名前。
  - `tablet num`: エクスポートされたタブレットの総数。
  - `broker`: v2.4以前では、エクスポートジョブで使用されるブローカーの名前を返すために使用されます。v2.5以降では、このフィールドは空の文字列を返します。
  - `coord num`: エクスポートジョブが分割されるクエリプランの数。
  - `db`: エクスポートされたデータが属するデータベースの名前。
  - `tbl`: エクスポートされたデータが属するテーブルの名前。
  - `row delimiter`: エクスポートされたデータファイルで使用される行セパレータ。
  - `mem limit`: エクスポートジョブに許可される最大メモリ量。単位: バイト。

- `Path`: エクスポートされたデータがリモートストレージに保存されるパス。
- `CreateTime`: エクスポートジョブが作成された時間。
- `StartTime`: エクスポートジョブがスケジュールされ始めた時間。
- `FinishTime`: エクスポートジョブが完了した時間。
- `Timeout`: エクスポートジョブが予想以上にかかった時間。単位: 秒。この時間は`CreateTime`から計算されます。
- `ErrorMsg`: エクスポートジョブがエラーを発生させた理由。このフィールドは、エクスポートジョブがエラーに遭遇した場合にのみ返されます。

## 例

- 現在のデータベース内のすべてのエクスポートジョブをクエリします:

  ```SQL
  SHOW EXPORT;
  ```

- データベース`example_db`内のIDが`921d8f80-7c9d-11eb-9342-acde48001122`のエクスポートジョブをクエリします:

  ```SQL
  SHOW EXPORT FROM example_db
  WHERE queryid = "921d8f80-7c9d-11eb-9342-acde48001122";
  ```

- データベース`example_db`内で`EXPORTING`状態のエクスポートジョブをクエリし、結果セットのエクスポートジョブレコードを`StartTime`で昇順にソートするよう指定します:

  ```SQL
  SHOW EXPORT FROM example_db
  WHERE STATE = "exporting"
  ORDER BY StartTime ASC;
  ```

- データベース`example_db`内のすべてのエクスポートジョブをクエリし、結果セットのエクスポートジョブレコードを`StartTime`で降順にソートするよう指定します:

  ```SQL
  SHOW EXPORT FROM example_db
  ORDER BY StartTime DESC;
  ```