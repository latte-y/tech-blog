---
title: SQL関数メモ
weight: 10
description: 日付関数・Window関数・行と列の変換など、よく使うSQLの書き方メモ（Trino/Presto 前提）
---

## 日付に関する関数

### date_format(col, '%Y-%m')
`yyyy-MM` に変換

### date_column - INTERVAL 'N-1' DAY
- 「yyyy-MM-ddで終わる、n日間の期間（当日を含む）」の開始日を求める場合に使う
- ※N日前の日付は、`date_column - INTERVAL 'N' DAY` で求められる


## Window関数

### ROW_NUMBER
同順位であっても、全ての行に必ず**一意の連番（1, 2, 3, 4...）**を割り振る

### RANK
同じ値を持つ行には同じ順位を付けるが、その後の順位はスキップされる（例：1位が2つある場合、次は3位になる）

### DENSE_RANK
同じ値を持つ行には同じ順位を付け、その後の順位はスキップしない（例：1位が2つある場合、次は2位になる）

### （補足）フレーム句のOption
- `ROWS`：移動単位を行で設定
- `RANGE`：移動単位を列の値で設定。基準となる列はORDER BY 句で指定された列
    - （利用例）不連続な計測日（sample_date）、体重（weight_kg）テーブルで、前日の体重と比較
    - 前日がない場合、NULLとして抽出される（＝不連続なキーに対し、集計する際に便利）
- `n PRECEDING`：nだけ前へ（小さい方）へ移動
- `n FOLLOWING`：nだけ後ろへ（大きい方）へ移動
- `UNBOUNDED PRECEDING`：無制限に遡る方へ移動
- `UNBOUNDED FOLLOWING`：無制限に下る方へ移動
- `CURRENT ROW`：現在行


### LAG / LEAD

- `LAG`：現在の行から見て「指定した行数分だけ前にある行の値」を取得
    
    ```sql {filename="sample.sql"}
    LAG(対象カラム, 取得する行数, デフォルト値)
    OVER (PARTITION BY グループ化するカラム ORDER BY 並び順)
    ```
    
- `LEAD`：現在の行から見て「指定した行数分だけ後にある行の値」を取得
    
    ```sql {filename="sample.sql"}
    LEAD(対象カラム, 取得する行数, デフォルト値)
    OVER (PARTITION BY グループ化するカラム ORDER BY 並び順)
    ```
    

### 連続で n 回出現する値の抽出（GRPテクニック）

- idから同一num内の連番(`ROW_NUMBER`)を引くと、連続している区間は同じgrp値になる「gaps and islands」という定番テクニック
- この方法は`HAVING COUNT(*) >= n`で閾値を自由に変えられる(例: 5連続に条件変更も1箇所修正で済む)ため、DRYかつ汎用性が高い
- 「連続する期間をグループ化して集計する」系の問題(ログイン連続日数など)全般に転用できる、覚えておく価値の高いパターン

```sql {filename="sample.sql"}
WITH islands AS (
    SELECT
        num,
        id - ROW_NUMBER() OVER (PARTITION BY num ORDER BY id) AS grp
    FROM Logs
)
SELECT DISTINCT num AS ConsecutiveNums
FROM islands
GROUP BY num, grp
HAVING COUNT(*) >= 3
```

### 移動平均

- e.g. 1週間（当日+過去6日間）の株価の移動平均

```sql {filename="sample.sql"}
SELECT
	AVG(price) OVER (
		ORDER BY visited_on
		ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
	) AS moving_average
FROM stock_price
```

### カラム（列）→レコード（行） UNION ALL
```sql {filename="sample.sql"}
SELECT hoge UNION ALL SELECT foo ~~~
```
（参考）https://leetcode.com/problems/count-salary-categories/description/?envType=study-plan-v2&envId=top-sql-50


### レコード（行）→カラム（列） 条件付き集計
Trino/Presto（TD）には `PIVOT` 構文がないため、`CASE WHEN` を集計関数で包んで列に展開する。

```sql {filename="sample.sql"}
SELECT
    col1,
    MAX(CASE WHEN グループ化対象列 = '値1' THEN 集計対象列 END) AS 値1,
    MAX(CASE WHEN グループ化対象列 = '値2' THEN 集計対象列 END) AS 値2
FROM table_name
GROUP BY col1
```

（参考）SQL Server・Snowflake・BigQuery などでは `PIVOT` 構文が使える。

```sql {filename="sample.sql"}
SELECT
    col1,
    col2,
    ...
FROM table_name
PIVOT (
    集計関数(集計対象列)
    FOR グループ化対象列 IN (グループ化対象列の値1, グループ化対象列の値2, ・・・)
) AS 別名
```
