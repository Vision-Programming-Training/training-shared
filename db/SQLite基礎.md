# SQLite 基礎レクチャー

この資料は、研修で使うミニEC（注文API）の裏側にある **データベース（DB）** を、最低限読み書きできるようになるための入門です。<br>
題材はこのリポジトリの [`データモデル.md`](データモデル.md)（`Product` / `Order` / `OrderItem` / `Coupon`）をそのまま使います。

特定の言語の話はしません。**SQL という「DB への命令文」** と、**SQLite というその実体** を触ることがゴールです。

---

## 1. そもそも DB と SQLite とは

### DB（データベース）

アプリが扱うデータ（商品・注文・在庫…）を、**消えないように・検索しやすい形で**しまっておく入れ物です。<br>
プログラムを再起動してもデータが残るのは、メモリ上ではなく DB に書いているからです。

多くの DB は **テーブル（表）** の集まりです。Excel のシートをイメージするとよいです。

- **テーブル** = 1 枚の表（例: `Product` 表）
- **列（カラム）** = 表の縦。意味の単位（例: `Name`, `Price`）
- **行（レコード）** = 表の横。データ 1 件（例: 「ノート, 300円, 在庫50」）

### SQLite（エスキューライト）

DB にはいろいろな製品があります（PostgreSQL, MySQL, SQL Server…）。<br>
その中で **SQLite** は、**1 個のファイルがそのまま DB** という、いちばん手軽な DB です。

| 普通の DB | SQLite |
|---|---|
| サーバーを起動して、ネット越しに接続する | **ファイルを開くだけ**。サーバー不要 |
| インストール・設定が必要 | ファイル 1 つ（例: `app.db`）で完結 |
| 本番の大規模サービス向き | 学習・小規模アプリ・お試し向き |

> このトレーニングのバックエンドは、起動すると SQLite のファイル（または「メモリ上の一時 DB」）を作り、そこに [`データモデル.md`](データモデル.md) の初期データ（seed）を入れてから動き出します。<br>
> つまり**今みなさんが触っている注文データは、この SQLite の中**にあります。

---

## 2. SQL — DB への命令文

DB を操作する言葉が **SQL（エスキューエル）** です。よく使うのは次の 4 つ（頭文字で「CRUD」と呼びます）。

| 操作 | SQL | 日本語 |
|---|---|---|
| Create | `INSERT` | 追加する |
| Read | `SELECT` | 取り出す（読む） |
| Update | `UPDATE` | 書き換える |
| Delete | `DELETE` | 消す |

研修で圧倒的に多いのは **`SELECT`（読む）** です。<br>
「アプリの挙動がおかしい → DB に入っているデータを自分の目で確認する」ために使います。まずは読みから覚えれば十分です。

> **大事な心構え**: `INSERT` / `UPDATE` / `DELETE` は**データを書き換えます**。<br>
> 研修中に DB の中身を確認するときは、原則 **`SELECT`（読むだけ）** にしてください。書き換える系は影響範囲を理解してから。

---

## 3. テーブルを作る — CREATE TABLE

実際の seed では各テーブルがこういう形で作られています（`Product` の例）。<br>
今は「列にはそれぞれ**型**と**制約**がある」ことだけ分かれば OK です。

```sql
CREATE TABLE Product (
    Id     INTEGER PRIMARY KEY AUTOINCREMENT,  -- 主キー・自動採番
    Name   TEXT    NOT NULL,                   -- 商品名（空にできない）
    Price  NUMERIC NOT NULL,                   -- 税抜単価（金額）
    Stock  INTEGER NOT NULL                    -- 在庫数
);
```

### SQLite の型（ざっくり）

SQLite の型は 5 つだけ。[`データモデル.md`](データモデル.md) の「論理型」との対応は次の通りです。

| データモデルの論理型 | SQLite の型 | 例 |
|---|---|---|
| 整数 | `INTEGER` | `Id`, `Stock`, `Quantity` |
| 文字列 | `TEXT` | `Name`, `Code`, 列挙（`Status` など） |
| 金額 | `NUMERIC`（固定小数点） | `Price`, `TotalAmount`, `UnitPrice` |
| 日時 | `TEXT`（ISO8601 文字列で保存） | `CreatedAt` |

> **金額は浮動小数点で持たない**（[`データモデル.md`](データモデル.md) §4）。`0.1 + 0.2 ≠ 0.3` のような誤差が出て、A1（丸め）系のバグの温床になるためです。

### 主キーと外部キー

- **主キー (PRIMARY KEY)**: その行を一意に決める列。ふつう `Id`。重複できない。
- **外部キー (FOREIGN KEY)**: 別のテーブルの行を指す列。<br>
  例: `OrderItem.ProductId` は `Product.Id` を指す＝「この明細はどの商品か」。

```sql
CREATE TABLE OrderItem (
    Id        INTEGER PRIMARY KEY AUTOINCREMENT,
    OrderId   INTEGER NOT NULL,   -- → Order.Id
    ProductId INTEGER NOT NULL,   -- → Product.Id
    Quantity  INTEGER NOT NULL,
    UnitPrice NUMERIC NOT NULL,
    FOREIGN KEY (OrderId)   REFERENCES "Order"(Id),
    FOREIGN KEY (ProductId) REFERENCES Product(Id)
);
```

> `Order` は SQL の予約語なので、SQLite では `"Order"` とダブルクォートで囲みます。

---

## 4. データを取り出す — SELECT（いちばん使う）

### 4.1 全部見る

```sql
SELECT * FROM Product;
```

- `SELECT` … 取り出す
- `*` … 全部の列
- `FROM Product` … `Product` テーブルから

結果（seed の初期データ）:

| Id | Name | Price | Stock |
|---|---|---|---|
| 1 | ノート | 300 | 50 |
| 2 | ボールペン | 150 | 100 |
| 3 | マグカップ | 1200 | 20 |
| … | … | … | … |

### 4.2 列を選ぶ

```sql
SELECT Name, Price FROM Product;
```

必要な列だけに絞ると見やすくなります。

### 4.3 条件で絞る — WHERE

```sql
-- 在庫切れ（Stock が 0）の商品だけ
SELECT * FROM Product WHERE Stock = 0;

-- 価格が 1000 円以上
SELECT * FROM Product WHERE Price >= 1000;

-- 商品名が「ノート」
SELECT * FROM Product WHERE Name = 'ノート';
```

| 演算子 | 意味 |
|---|---|
| `=` | 等しい（プログラムの `==` ではなく `=` 1 つ） |
| `<>` または `!=` | 等しくない |
| `>` `>=` `<` `<=` | 大小 |
| `AND` / `OR` | かつ / または |
| `LIKE 'ノート%'` | 部分一致（`%` は任意の文字列）。C1（商品名検索）課題で使う |
| `IS NULL` / `IS NOT NULL` | 空かどうか（`= NULL` ではなく `IS NULL`） |

```sql
-- 在庫があって、かつ 500 円以下
SELECT * FROM Product WHERE Stock > 0 AND Price <= 500;

-- クーポン未使用の注文（CouponId が空）
SELECT * FROM "Order" WHERE CouponId IS NULL;
```

### 4.4 並べる・件数を絞る — ORDER BY / LIMIT

```sql
-- 新しい注文順（CreatedAt の降順）に 5 件
SELECT * FROM "Order" ORDER BY CreatedAt DESC LIMIT 5;
```

- `ORDER BY 列` … 並び替え。`ASC`（昇順・既定）/ `DESC`（降順）
- `LIMIT n` … 先頭 n 件だけ

> 注文一覧が「新しい順」で出るのは、アプリ内部でこの `ORDER BY CreatedAt DESC` 相当をやっているからです（[`データモデル.md`](データモデル.md) の `Order.CreatedAt` 注記）。

---

## 5. テーブルをまたいで読む — JOIN

DB の真価は **複数テーブルを繋いで読む** ことです。<br>
「注文 A には何の商品が何個入っているか？」は、`Order` だけでは分かりません。`OrderItem` と `Product` を繋ぎます。

```sql
SELECT
    o.Id        AS 注文ID,
    p.Name      AS 商品名,
    oi.Quantity AS 数量,
    oi.UnitPrice AS 単価
FROM "Order" o
JOIN OrderItem oi ON oi.OrderId = o.Id     -- 注文 と 明細 を OrderId で繋ぐ
JOIN Product  p  ON p.Id = oi.ProductId    -- 明細 と 商品 を ProductId で繋ぐ
WHERE o.Id = 1;
```

結果（注文 A）:

| 注文ID | 商品名 | 数量 | 単価 |
|---|---|---|---|
| 1 | ノート | 2 | 300 |
| 1 | マグカップ | 1 | 1200 |

- `o` `oi` `p` は **別名（エイリアス）**。長いテーブル名を短く書くため。
- `JOIN ... ON 繋ぐ条件` で、どの列同士が対応するかを指定します。
- ここが [`データモデル.md`](データモデル.md) の ER 図（`Order 1 ── * OrderItem * ── 1 Product`）そのものです。

> **`UnitPrice` は `Product.Price` ではなく `OrderItem.UnitPrice` を見る**（A3 課題の核心）。<br>
> 上の SQL でも単価は `oi.UnitPrice`（注文時のスナップショット）から取っています。`p.Price`（今の商品価格）を使ってしまうと、値上げで過去注文の金額が変わってしまいます。

---

## 6. 集計する — COUNT / SUM など

```sql
-- 注文の総件数
SELECT COUNT(*) FROM "Order";

-- 注文ごとの明細点数（GROUP BY = 注文IDでまとめる）
SELECT OrderId, COUNT(*) AS 明細数, SUM(Quantity) AS 合計数量
FROM OrderItem
GROUP BY OrderId;
```

| 関数 | 意味 |
|---|---|
| `COUNT(*)` | 件数 |
| `SUM(列)` | 合計 |
| `AVG(列)` | 平均 |
| `MIN` / `MAX` | 最小 / 最大 |

`GROUP BY 列` は「その列が同じ行をまとめて、まとめごとに集計」です。<br>
A8（数量合算漏れ）のように「同じ商品が複数明細に分かれているとき合算できているか」を DB 側で確認するのに使えます。

---

## 7. 書き換える系（参考・注意して使う）

研修で多用はしませんが、seed やアプリ内部はこれでデータを作っています。

```sql
-- 追加
INSERT INTO Product (Name, Price, Stock) VALUES ('新商品', 500, 10);

-- 書き換え（必ず WHERE をつける！）
UPDATE Product SET Stock = Stock - 1 WHERE Id = 1;

-- 削除（必ず WHERE をつける！）
DELETE FROM Product WHERE Id = 99;
```

> ⚠️ **`UPDATE` / `DELETE` で `WHERE` を忘れると、全行が対象になります。**<br>
> `UPDATE Product SET Stock = 0;` は「**全商品の在庫を 0**」にしてしまいます。書き換える前に、まず同じ条件で `SELECT` して対象を確認するのが安全です。
>
> ```sql
> SELECT * FROM Product WHERE Id = 1;   -- 先に対象を確認
> UPDATE Product SET Stock = 0 WHERE Id = 1;  -- それから書き換え
> ```

### トランザクション（ひとことだけ）

「在庫を減らす」＋「注文を作る」のように、**複数の書き込みを “全部成功 or 全部なかったことに”** したい場面があります。これを **トランザクション** と呼びます。

```sql
BEGIN;
  UPDATE Product SET Stock = Stock - 2 WHERE Id = 1;
  INSERT INTO "Order" (...) VALUES (...);
COMMIT;     -- ここで確定。途中で失敗したら ROLLBACK で巻き戻す
```

注文確定処理が「在庫だけ減って注文は作られない」中途半端な状態にならないのは、この仕組みのおかげです。今は**名前と目的だけ**知っていれば十分です。

---

## 8. 実際に触ってみる

SQLite は手元で気軽に試せます。

### 方法A: コマンドラインの `sqlite3`

```sh
# DB ファイルを開く（無ければ新規作成される）
sqlite3 app.db

# 開いた後、SQLite 専用コマンド（ドットで始まる）
.tables          -- テーブル一覧
.schema Product  -- Product の定義を見る
.headers on      -- 結果に列名を表示
.mode column     -- 列を揃えて表示

SELECT * FROM Product;   -- ふつうの SQL は最後に「;」が必要
.quit            -- 終了
```

> アプリが使う DB ファイル名は各言語版バックエンドの設定で確認してください（メモリ DB の場合はファイルが残らないので、お試し用に別途 `app.db` を作って練習すると安全です）。

### 方法B: GUI ツール

[DB Browser for SQLite](https://sqlitebrowser.org/)（無料）を使うと、Excel のように表を見ながらクリックで `SELECT` できます。SQL に慣れないうちはこちらが楽です。

### まず試してほしい 3 つ

1. `SELECT * FROM Product;` — seed の 7 商品が [`データモデル.md`](データモデル.md) の表と一致するか
2. `SELECT * FROM "Order" ORDER BY CreatedAt DESC;` — 注文 A/B/C が新しい順に並ぶか
3. §5 の JOIN を実行して、注文 B の明細（Tシャツ×1, ボールペン×3）と `TotalAmount = 2695` を突き合わせる

---

## 9. この研修での使いどころ

| 場面 | 使う SQL |
|---|---|
| 「在庫が戻らない（A4）」→ キャンセル後の在庫を確認 | `SELECT Stock FROM Product WHERE Id = ?` |
| 「金額がずれる（A1/A7）」→ 注文の明細と単価を確認 | §5 の JOIN |
| 「数量合算漏れ（A8）」→ 同一商品の明細を確認 | §6 の `GROUP BY` |
| 「過去注文が値上げで変わる（A3）」→ `UnitPrice` を確認 | `SELECT UnitPrice FROM OrderItem WHERE OrderId = ?` |
| 「商品名検索（C1）」 | `WHERE Name LIKE '%キーワード%'` |

**コードを読む前に、DB の実データを `SELECT` で見て「症状」を自分の目で確認する。**<br>
これができると、原因の切り分け（アプリのバグか / データの問題か）が一気に速くなります。

---

## 10. まとめ

- **SQLite = ファイル 1 つで動く手軽な DB**。このアプリのデータはこの中にある。
- **SQL の基本は CRUD**。研修では **`SELECT`（読む）** が主役。
- `WHERE` で絞り、`ORDER BY` で並べ、`JOIN` でテーブルを繋ぐ。
- 書き換え系（`UPDATE`/`DELETE`）は **`WHERE` 必須**。先に `SELECT` で対象確認。
- 詳しいテーブル定義・初期データの実値は [`データモデル.md`](データモデル.md) を参照。

> この資料は「読めて・確認できる」までが目標です。アプリがどう SQL を発行しているか（ORM など）は各言語版バックエンドのコードで追ってください。
