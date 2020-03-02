---
title: '[MySQL] 如何排序名次 #1'
date: 2019-10-16 17:54:25
tags:
---

# 目標
於 MySQL 中利用 SELF JOIN，並設定 ON 參考點的符合條件，正確排序飲料銷售量名次。

<!-- more -->

# 情境
回顧自己在 AC 學期三 MySQL 伺服器上學習 SQL 語法時，遇到一個實境章節是建立飲料店的資料庫，在此簡單假設有飲料及訂單 2 張資料表，飲料有編號（id）、名稱（name）及價格欄位（price），訂單則有編號（id）、銷售量（quantity）及飲料編號（drink_id）。

```sql
drinks : id, name, price
orders : id, quantity, drink_id
```

現在想像飲料店老闆想知道目前銷售量最高的飲料是哪一種？身為最有效率員工的你並沒有想的太多，很快查詢出銷售量最高的飲料就是緋聞紅茶。

```sql
SELECT 
  drinks.`name`, 
  SUM(`quantity`) AS `total_quantity`
FROM orders
JOIN drinks ON orders.`drink_id` = drinks.`id`
GROUP BY `drink_id`
ORDER BY `total_quantity` DESC
LIMIT 1;
```

|  name   | total_quantity |
|---------| -------------- |
| 緋聞紅茶 |       21       |

只有緋聞紅茶嗎？老闆略帶狐疑地再問一次，你扭扭捏捏地回應著「是阿．．．」，雖然你嘴上這樣說，但你心裡開始不安了起來。萬一店裡的不熟紅茶也剛好賣 21 杯，那麼銷售量最好的就不是只有一種，此時的你也只講對了一半，這樣的微小錯誤，未來可能產生蝴蝶效應，而導致老闆決策方向錯誤甚至虧損，相信這不是身為一個社畜扛的起。

# 問題
面對上述諸如此類，排山倒海而來的恐慌及壓力，你勢必要了解各種飲料的銷售量排行，好讓你能一目了然是否有並列第一的情況。為此，我們的目標是針對各種飲料的銷售量排序並呈現出正確名次，這裡的問題是如何排名？

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

# 解決
1. MySQL 內建的函式 RANK()？
  就在閱讀官方技術文件後，發現只有 MySQL 8.0 才支援 RANK()，如果使用的是 5.6 還是得自己生出一個。有道是凡事做最好準備，最壞打算，這裡我們就不使用 RANK()了。
2. 單純使用查詢語法搭配 WHERE 關鍵字及運算子來計算名次？
  首先將例子簡單及縮小化，以同學分數排名做範例。之後再丟回來也不遲。
  | name | score | rank |
  | ---- | ----- | ---- |
  |  A   |   90  |  1   |
  |  B   |   80  |  2   |
  |  C   |   70  |  3   |
  |  D   |   60  |  4   |
  |  E   |   50  |  5   |
  這裡想知道的是

  E 同學為何排名 5？因為 A, B, C, D 等 4 位同學分數高於 E。
  D 同學為何排名 4？因為 A, B, C 等 3 位同學分數高於 D。

  請問施主，參透出什麼洞見了嗎？

  沒錯，只要找出大於等於自己分數的個數，就是該位同學的名次。

  知道名次從何而來，接下來的問題是如何比對分數，而從過程當中發現，比來比去其實就同一個班級同學的分數互做比較，而在同一張表裡互相比對可以使用 SELF JOIN，自己跟自己合併。有了概念後再拉回來飲料店的例子，你迫不及待地快速寫下步驟及語法。
  
  1. 將訂單以飲料 ID 分組，加總 quantity 後按 total_quantity 降冪排序。
  2. 將上述查詢後的資料表，採 SELF JOIN，並分別命名為 a 及 b。
  3. 從 a 資料表的角度思考，其參考點設為 b.total_quantity 大於等於 a.total_quantity，意思是誰的銷售量大於等於我。
  4. 一樣是站在 a 資料表的角度，去計算大於等於 a.total_quantity 的個數當作排名的欄位。

  ```sql
  SELECT
    a.`name`, 
    a.`total_quantity`, 
    COUNT(b.`total_quantity`) AS rank
  FROM 
  (
    SELECT 
      drinks.`name`, 
      SUM(`quantity`) AS total_quantity
    FROM orders 
    LEFT JOIN drinks ON orders.`drink_id` = drinks.`id`
    GROUP BY `drink_id`
    ORDER BY total_quantity DESC
  ) AS a
  JOIN 
  (
    SELECT 
      drinks.`name`, 
      SUM(`quantity`) AS total_quantity
    FROM orders 
    LEFT JOIN drinks ON orders.`drink_id` = drinks.`id`
    GROUP BY `drink_id`
    ORDER BY total_quantity DESC
  ) AS b
  ON a.`total_quantity` <= b.`total_quantity`
  GROUP BY a.`name`
  ORDER BY a.`total_quantity` DESC;
  ```

  接下來，就是見證奇蹟的時刻了。

  |   name     | total_quantity | rank |
  | ---------- | -------------- | ---- |
  |  緋聞紅茶   |       21       |  2   |
  |  不熟紅茶   |       21       |  2   |
  |  雪花冷路   |       20       |  3   |
  |  中正路路   |       12       |  4   |
  |  莉莉紅茶   |       4        |  5   |
  |  新芽紅茶   |       2        |  7   |
  |  春天冷路   |       2        |  7   |
  |  中山路咩   |       1        |  8   |

  誒老大不是阿～ 怎麼有 2 個 2？奇蹟．．．儼然沒有發生。

# 再解決
此刻一股無力感從腳底、從背脊、從頭頂開始慢慢侵蝕著你，
但你仍故作鎮定，再次進行沙盤推演。

緋聞紅茶，誰的銷售量大於等於緋聞紅茶？不熟紅茶及它自己，計數 2。
不熟紅茶，誰的銷售量大於等於不熟紅茶？緋聞紅茶及它自己，計數 2。
雪花冷路，誰的銷售量大於等於雪花冷路？緋聞、不熟及它自己，計數 3。

瞬間你心裡阿了一聲，彷彿看見一道光。原來緋聞紅茶及不熟紅茶本該是並列第 1 ，但他們把對方視為己出，算進自己的計數之中，所以只要將銷售量相等的對方踢出自己的計數，並保留銷售量一樣的自己，這樣一來，便能同時並列第 1。修改 line 25 ON 之後的條件並新增 line 26 如下。

```sql
SELECT 
  a.`name`, 
  a.`total_quantity`, 
  COUNT(b.`total_quantity`) AS rank
FROM 
(
  SELECT 
    drinks.`name`, 
    SUM(`quantity`) AS total_quantity
  FROM orders 
  LEFT JOIN drinks ON orders.`drink_id` = drinks.`id`
  GROUP BY `drink_id`
  ORDER BY total_quantity DESC
) AS a
JOIN 
(
  SELECT 
    drinks.`name`, 
    SUM(`quantity`) AS total_quantity
  FROM orders 
  LEFT JOIN drinks ON orders.`drink_id` = drinks.`id`
  GROUP BY `drink_id`
  ORDER BY total_quantity DESC
) AS b
ON a.`total_quantity` < b.`total_quantity`
OR (a.`total_quantity` = b.`total_quantity` AND a.`name` = b.`name`)
GROUP BY a.`name`
ORDER BY a.`total_quantity` DESC;
```
再來一次
緋聞紅茶，誰的銷售量大於緋聞紅茶？沒有，但請算進自己，計數 1。
不熟紅茶，誰的銷售量大於不熟紅茶？沒有，但請算進自己，計數 1。
雪花冷路，誰的銷售量大於雪花冷路？緋聞、不熟，再加自己，計數 3。

看看結果如何。

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

如預期般正確顯示銷售量排名。

往後你將能充滿信心地向老闆報告，目前有 2 種飲料的銷售量同時並列第一，分別是緋聞紅茶及不熟紅茶。再也沒有當初心裡那種忐忑不安的心虛感，你澈底地成為了一名盡責的社畜。

# 結論
其實初始的想法很單純，僅僅只想列出名次以獲得正確且完整的資訊。弄到後來卻花了將近 4 個小時再思考。從名次如何產生、如何比對到如何寫出 SQL 的語法。逐步地拆出問題，也逐步地找到解決方法。

然而這樣就滿足了嗎？

不是吧！

因為 SELF JOIN 需要重複寫 2 次子查詢，在目前了解 SQL的範圍內，這樣的寫法對後續維護是不難，但總是感到不夠完美及盡興。

所以我利用 Google 搜尋同樣的問題，發現了更簡短的寫法，不過需要額外的運算子幫忙，詳情請接下一篇。這裡簡略敘述一下，主要是使用 User variables 及 IF 的條件判斷式，程序則是讓子查詢做完 SUM( quantity ) 後丟上來作條件判斷並儲存名次，判斷完再往上丟到最外層去篩選欄位，算是相對易懂的方式。

最後總結本篇於 MySQL 學習 SQL 的過程，目標是正確查詢飲料店資料庫中銷售量最高的飲料，首先利用同學分數的簡單例子做名次的推導，了解名次產生的方式之後，回到飲料店的情境，以 SELF JOIN 於同一張資料表做銷售量的比對，最後針對名次並列顯示的問題，微微地調整了 ON 的條件敘述，成功地完成排序名次的目標。
<br/>


