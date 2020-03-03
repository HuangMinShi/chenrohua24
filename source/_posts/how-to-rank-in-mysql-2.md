---
title: '[MySQL] 如何排序名次 #2'
date: 2019-10-17 17:54:25
tags:
---
# 目標
於 MySQL 中宣告使用者變數 、利用 IF 條件判斷並考量名次並列情形，正確排序飲料銷售量名次。

<!-- more -->

# 無名次並列
延續上一篇的情境，一樣假設有飲料及訂單 2 張資料表，定義欄位如下 ：

```sql
drinks : id, name, price
orders : id, quantity, drink_id
```

想知道哪種飲料銷售量最高？SQL 語法如下：

```sql
-- 宣告使用者變數及賦值
SET @prev = null, @curr = null, @rank = 0;

SELECT
  b.`name`,
  b.`total_quantity`,
  b.`rank`
FROM 
(
  -- 條件判斷名次
  SELECT 
    a.*,
  @prev := @curr,
  @curr := a.`total_quantity`,
  @rank := IF(@prev = @curr, @rank, @rank + 1) AS rank
  FROM
  (
    -- 子查詢計算總銷量
    SELECT 
      drinks.`name`, 
      SUM(`quantity`) AS `total_quantity` 
    FROM orders
    JOIN drinks ON drinks.`id` = orders.`drink_id`
    GROUP BY `drink_id`
    ORDER BY `total_quantity` DESC
  ) AS a
) AS b;
view rawrank.sql hosted with ❤ by GitHub
```
1. 初始化
  line 2：宣告 3 個使用者變數 `@prev, @curr, @rank`，分別儲存前值、當前值及名次，而變數的命名形式是 @變數名。
2. 從第三層非相關子查詢開始
  line 17– 26 ： 由 orders 去 JOIN drinks，按 drink_id 分組，再由 SELECT 加總銷售量，最後降冪排序，表格別名取為 a 返回第二層。
3. 第二層子查詢
  line 13：`@curr` 的值賦予給 `@prev`
  line 14：將第三層計算好的總銷售量 `a.total_quantity` 賦值給 `@curr`
  line 15：如果前值跟當前值不相等，則 `@rank` 加 1 再賦值給 `@rank`
4. 第一層主查詢
  重複 line 13 -15 直到所有紀錄 （records） 執行完畢。最後帶著 a 表格的全部欄位，取名為 b 奔回第一層，由 SELECT 篩選欄位。


|   name     | total_quantity | rank |
| ---------- |- ------------- | ---- |
|  緋聞紅茶   |       21       |  1   |
|  不熟紅茶   |       21       |  1   |
|  雪花冷路   |       20       |  2   |
|  中正路路   |       12       |  3   |
|  莉莉紅茶   |       4        |  4   |
|  新芽紅茶   |       2        |  5   |
|  春天冷路   |       2        |  5   |
|  中山路咩   |       1        |  6   |

但結果跟預期的有點不太一樣。因為我預期的是當遇到 2 種飲料名次並列時，名次保持不變，但仍佔有位置，例如（1, 2, 2, 2, 5）而非（1, 2, 2, 2, 3）。如飲料查詢結果中的雪花冷路，它的名次應該是 3 才是。

# 有名次並列
為解決上述問題，額外再新增第 4 個變數 `@temp` 去記錄 loop 的次數，一但遇到並列，雖然名次不變，但紀錄次數仍持續加 1 ，直到沒有並列問題，再把 `@temp` 賦值給 @rank 修改後如下：

```sql
-- 宣告使用者變數及賦值
SET @prev = null, @curr = null, @rank = 0, @temp = 0;

SELECT
  b.`name`,
  b.`total_quantity`,
  b.`rank`
FROM 
(
  -- 條件判斷名次
  SELECT 
    a.*,
  @temp := @temp + 1,
  @prev := @curr,
  @curr := a.`total_quantity`,
  @rank := IF(@prev = @curr, @rank, @temp) AS rank
  FROM
  (
    -- 子查詢計算總銷量
    SELECT 
      drinks.`name`, 
      SUM(`quantity`) AS `total_quantity` 
    FROM orders
    JOIN drinks ON drinks.`id` = orders.`drink_id`
    GROUP BY `drink_id`
    ORDER BY `total_quantity` DESC
  ) AS a
) AS b;
```

修改部分：
line 2 ：新增 `@temp` 變數
line 13：`@temp +=1`
line 16：若兩者銷售量不相等，即沒有並列情形，則將 `@temp` 賦值給 `@rank`


|   name     | total_quantity | rank |
| ---------- | -------------- | ---- |
|  緋聞紅茶   |       21       |  1   |
|  不熟紅茶   |       21       |  1   |
|  雪花冷路   |       20       |  3   |
|  中正路路   |       12       |  4   |
|  莉莉紅茶   |       4        |  5   |
|  新芽紅茶   |       2        |  6   |
|  春天冷路   |       2        |  6   |
|  中山路咩   |       1        |  8   |

總算是正確顯示名次資訊了。

# 結論
相較上一篇，本篇以另一種方式列出名次，利用宣告使用者變數及排序後的總銷售量資料表，逐筆針對前值及當前值做條件判斷，若相等，名次保持不變；若不等則名次加 1。然而考量到並列情形，須再宣告另一變數 temp 儲存 loop 的次數值，相等，名次依舊保持不變；不相等，紀錄次數賦值給名次，達成像（1, 2, 2, 4, 5）的期望名次顯示。

值得一提的是，上述在開頭利用 SET 方式宣告並賦予初始值的使用者變數，其賦值運算子可以使用 = 或 := e.g. `SET @test := 1;`

而另一種賦值的方式常見於 SELECT e.g. `SELECT @test1, @test2 := 2;`，但只能用 := 作為賦值運算子。WHY？

因為 = 出了 SET 這個語句之後，將被視為比較運算子而非賦值運算子。另外這樣的方式可能會於未來版本的 MySQL 中刪除，請注意。

# 參考
[MySQL DOCUMENTATION](https://dev.mysql.com/doc/refman/8.0/en/user-variables.html)
[MySQL排序名次、排行(使用SQL語法排序)](https://newaurora.pixnet.net/blog/post/221610377-mysql%e6%8e%92%e5%ba%8f%e5%90%8d%e6%ac%a1%e3%80%81%e6%8e%92%e8%a1%8c%28%e4%bd%bf%e7%94%a8sql%e8%aa%9e%e6%b3%95%e6%8e%92%e5%ba%8f%29)