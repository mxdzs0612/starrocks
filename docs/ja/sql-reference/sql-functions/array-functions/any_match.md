---
displayed_sidebar: docs
---

# any_match

配列の要素のいずれかが指定された述語に一致するかどうかを返します。

- 1 つ以上の要素が述語に一致する場合、`true` (1) を返します。

- 要素のいずれも一致しない場合、`false` (0) を返します（特別なケースとして、配列が空の場合があります）。

- 述語が 1 つ以上の要素に対して NULL を返し、他のすべての要素に対して `false` を返す場合、NULL を返します。

この関数は v3.0.6 以降でサポートされています。

## Syntax

```Haskell
any_match(lambda_function, arr1, arr2...)
```

`arr1` の要素のいずれかが、lambda 関数内の述語に一致するかどうかを返します。

## Parameters

- `arr1`: 一致させる配列。

- `arrN`: lambda 関数で使用されるオプションの配列。

- `lambda_function`: 値を一致させるために使用される lambda 関数。

## Return value

BOOLEAN 値を返します。

## Usage notes

- lambda 関数は [array_map()](array_map.md) の使用上の注意に従います。
- 入力配列が null または lambda 関数の結果が null の場合、null が返されます。
- `arr1` が空の場合、`false` が返されます。
- この関数を MAP に適用するには、`any_match((k,v)->k>v,map)` を `any_match(map_values(transform_values((k,v)->k>v, map)))` に書き換えます。例えば、`select any_match(map_values(transform_values((k,v)->k>v, map{2:1})));` は 1 を返します。

## Examples

`x` のいずれかの要素が `y` の要素より小さいかどうかを確認します。

```Plain
select any_match((x,y) -> x < y, [1,2,8], [4,5,6]);
+--------------------------------------------------+
| any_match((x, y) -> x < y, [1, 2, 8], [4, 5, 6]) |
+--------------------------------------------------+
|                                                1 |
+--------------------------------------------------+

select any_match((x,y) -> x < y, [11,12,8], [4,5,6]);
+----------------------------------------------------+
| any_match((x, y) -> x < y, [11, 12, 8], [4, 5, 6]) |
+----------------------------------------------------+
|                                                  0 |
+----------------------------------------------------+

select any_match((x,y) -> x < y, [11,12,null], [4,5,6]);
+-------------------------------------------------------+
| any_match((x, y) -> x < y, [11, 12, NULL], [4, 5, 6]) |
+-------------------------------------------------------+
|                                                  NULL |
+-------------------------------------------------------+

select any_match((x,y) -> x < y, [], []);
+------------------------------------+
| any_match((x, y) -> x < y, [], []) |
+------------------------------------+
|                                  0 |
+------------------------------------+

select any_match((x,y) -> x < y, null, [4,5,6]);
+---------------------------------------------+
| any_match((x, y) -> x < y, NULL, [4, 5, 6]) |
+---------------------------------------------+
|                                        NULL |
+---------------------------------------------+
```