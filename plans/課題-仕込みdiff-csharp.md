# 課題 仕込み diff 集（C# / ASP.NET Core 版）

> このファイルは `training-shared` の `課題カタログ.md` の **C# 版実装**。各課題の「狙い・症状・不変条件・難度」はカタログを参照。ここには **C# 固有の実 diff・該当ファイル・テスト名**だけを置く。
> **番号のキー**: A1〜A8 は**受講者ガイドの課題1〜8（＝カタログ A1〜A8）**と同じ番号。全ドキュメント共通で同じ番号が同じ課題を指す。
> **取り扱い注意**: 解答 diff そのもの。受講者には渡さない。

`<root>` = `training-backend-csharp/src`。各 diff は `main`（完成形）に対して「壊す向き（before→after）」で示す。**解答＝この diff を逆に当てて `main` に戻すこと**。

---

## A. バグ修正型

### A1. 定率クーポンの計算ミス
- **仕込み箇所**: `<root>/Services/PricingService.cs` `ApplyCoupon`

```diff
-            DiscountType.Percentage => subtotal * (1m - coupon.DiscountValue / 100m),
+            DiscountType.Percentage => subtotal - coupon.DiscountValue, // ← 仕込み: 定額と取り違え
```

- **再現**: トートバッグ1点(1800) + `SALE10` → 正 1782 / バグ (1800−10)×1.1 = 1969。
- **赤になるテスト**: `PricingServiceTests.ApplyCoupon_applies_percentage`（1800 → 1620）。
- **ヒント段階**: ①定額は合うのに定率だけずれる、違いはどこ？ ②`DiscountType` の分岐を1つずつ読む。`Percentage` の式は「％」を表現できているか？
- **解答**: `subtotal * (1m - coupon.DiscountValue / 100m)` に戻す。

### A2. 在庫チェックの境界（off-by-one）
- **仕込み箇所**: `<root>/Services/OrderService.cs` `CreateAsync`

```diff
-            if (product.Stock < quantity)
+            if (product.Stock <= quantity) // ← 仕込み: 在庫ちょうどが買えない
```

- **再現**: 在庫10のTシャツを10個 → 「在庫不足」。
- **赤になるテスト**: 完成形に境界テストが無い。追加しておくテスト:

```csharp
[Fact]
public async Task CreateAsync_allows_buying_exact_stock()
{
    using var db = SeededContext();
    var service = CreateService(db);
    var request = new CreateOrderRequest
    {
        Items = new() { new CreateOrderItemRequest { ProductId = 2, Quantity = 20 } } // 在庫ちょうど
    };
    var result = await service.CreateAsync(request);
    Assert.Equal(0, (await db.Products.FindAsync(2))!.Stock);
}
```

- **ヒント段階**: ①「在庫不足」はどこで投げている？ ②比べる演算子は `<`？`<=`？「在庫＝要求」でどちらに転ぶ？
- **解答**: `<` に戻す。

### A3. 単価スナップショットの崩れ
- **仕込み箇所**: `<root>/Services/OrderService.cs` `MapToDto`

```diff
     private static OrderDto MapToDto(Order order)
     {
         var items = order.Items
             .Select(i => new OrderItemDto(
                 i.ProductId,
                 i.Product?.Name ?? string.Empty,
                 i.Quantity,
-                i.UnitPrice,
-                i.UnitPrice * i.Quantity))
+                i.Product?.Price ?? 0m,                 // ← 仕込み: 現在価格を表示
+                (i.Product?.Price ?? 0m) * i.Quantity)) // ← 仕込み
             .ToList();
```

- **再現**: 既存注文Aを表示 → ノート価格を 300→500 に変更 → 注文Aの明細単価が 500 に化ける。
- **赤になるテスト**: `CreateAsync_..._computes_total` 内の `Assert.Equal(300m, noteLine.UnitPrice)` で部分カバー。価格変更後を検証する追加テスト:

```csharp
[Fact]
public async Task OrderItem_keeps_unit_price_even_after_product_price_changes()
{
    using var db = SeededContext();
    var service = CreateService(db);
    var created = await service.CreateAsync(new CreateOrderRequest
    {
        Items = new() { new CreateOrderItemRequest { ProductId = 1, Quantity = 1 } }
    });
    var p = await db.Products.FindAsync(1);
    p!.Price = 9999m;            // 後から値上げ
    await db.SaveChangesAsync();
    var fetched = await service.GetByIdAsync(created.Id);
    Assert.Equal(300m, fetched.Items[0].UnitPrice); // スナップショットは不変
}
```

- **ヒント段階**: ①`OrderItem` に `UnitPrice` 列があるのはなぜ？`Product.Price` があるのに。 ②表示時、明細単価はどの値から作られている？保存済み？今の商品価格？
- **解答**: `i.UnitPrice` / `i.UnitPrice * i.Quantity` に戻す。

### A4. キャンセルで在庫が戻らない
- **仕込み箇所**: `<root>/Services/OrderService.cs` `CancelAsync`

```diff
-        // 在庫を戻す
-        foreach (var item in order.Items)
-        {
-            item.Product.Stock += item.Quantity;
-        }
+        // ← 仕込み: 在庫を戻す処理を削除
```

- **赤になるテスト**: `OrderServiceTests.CancelAsync_restores_stock_and_sets_status_cancelled`。
- **ヒント段階**: ①キャンセル時に「ステータス変更」以外にやるべき副作用は？ ②注文時に在庫を引いた。キャンセルでは？
- **解答**: 在庫戻しループを復活させる。

### A5. 無効なクーポンコードが黙って無視される
- **仕込み箇所**: `<root>/Services/OrderService.cs` `ResolveCouponAsync`

```diff
-        return await _orderRepository.GetCouponByCodeAsync(couponCode)
-            ?? throw new BusinessRuleException($"クーポンが無効です (Code: {couponCode})");
+        return await _orderRepository.GetCouponByCodeAsync(couponCode); // ← 仕込み: 無効コードを黙ってクーポン無し扱い
```

- **再現**: カートで存在しないコード（例 `SALE99`）を入れて注文 → 正: 400「クーポンが無効です」 / バグ: 割引なしの満額で注文成立（トートバッグ1点なら 1980 円）。有効な `SALE10` / `WELCOME500` はこれまで通り効く。
- **赤になるテスト**: `OrderServiceTests.CreateAsync_throws_when_coupon_is_invalid`（コード `"NOPE"` で `BusinessRuleException` を期待）。完成形に既にあるので再現テストの追加は不要。
- **ヒント段階**: ①有効なコードは効くのに、デタラメなコードを入れても素通りする。クーポンを引き当てている処理はどこ？ ②`GetCouponByCodeAsync` が `null` を返したとき、その後どう振る舞っている？「見つからない」を弾いているか？
- **解答**: `?? throw new BusinessRuleException(...)` のガードを戻す。
- **UI再現性メモ**: クーポンコードはフロントの自由入力テキスト（`shop.js` が `couponEl.value.trim()` を送信）なので UI は事前に弾けず、検証はサーバー責務になる。

### A6. NotFound が 500 になる
- **仕込み箇所（案1: Service 側）**: `<root>/Services/OrderService.cs` `GetByIdAsync`

```diff
-        var order = await _orderRepository.GetByIdAsync(id)
-            ?? throw new NotFoundException($"注文が見つかりません (OrderId: {id})");
+        var order = await _orderRepository.GetByIdAsync(id)
+            ?? throw new Exception($"注文が見つかりません (OrderId: {id})"); // ← 仕込み: 型違い
```

- **仕込み箇所（案2: Middleware 側）**: `<root>/Middleware/ExceptionHandlingMiddleware.cs` の `catch (NotFoundException ...)` ブロックを削除（未処理 → 500 にフォールバック）。層をまたいで探させたいなら案2。
- **赤になるテスト**: `OrderServiceTests.GetByIdAsync_throws_NotFound_for_unknown_order`（案1）。案2は Middleware の統合テストが無いので、エンドポイント越しテストを足すか Swagger で手動確認。
- **ヒント段階**: ①Controller に try/catch が無い。例外はどこで HTTP に変換されている？ ②Middleware はどの例外型を 404 に？Service はその型を投げている？
- **解答**: `NotFoundException` を投げる／catch を戻す。

### A7. 税込の丸めが狂う
- **仕込み箇所**: `<root>/Services/PricingService.cs` `CalculateTotal`

```diff
-        return Math.Round(withTax, 0, MidpointRounding.AwayFromZero);
+        return Math.Round(withTax, 0, MidpointRounding.ToEven); // ← 仕込み: 偶数丸め
```

- **再現**: ボールペン1本 + `SALE10`（税抜 150 → 135 → 税込 148.5）→ 期待 149 が 148 になる。
  - ※シード商品の価格は全て 50 の倍数。50 の倍数 ×1.10 は必ず整数になるので、**クーポン無し（や定額 `WELCOME500`）では税込が小数にならず、この仕込みは一切再現しない**。端数が出るのは定率 `SALE10`（正味係数 0.9×1.10=0.99）を併用したときだけ。
  - ※偶数丸めでズレるのは「税込がちょうど N.5 かつ floor が偶数」のとき。`SALE10` 適用後で言うと税抜 150（→135→148.5）, 350, 550 … が該当。完成形カタログにあった「小計 105 → 115.5 が 115 になる」は**誤り**（115.5 は偶数丸めでも 116。そもそも 105 円という小計も 50 の倍数でない実商品では作れない）。
- **赤になるテスト**: **無い（仕込んでも全テスト緑のまま）**。完成形の `CalculateTotal_rounds_to_whole_yen`（小計 105・クーポン無し）は上記理由でこの仕込みを検出**できない**。これは「初期テストにこのケースが漏れていた」という想定の課題で、**まず受講者に再現テストを書かせてから直させる**（A8 マイナス請求と同じ進め方）。解答ブランチに追加しておくテスト:

```csharp
[Fact]
public void CalculateTotal_rounds_half_up_with_percentage_coupon()
{
    var items = Items((150m, 1)); // ボールペン 1 本（税抜 150）
    var coupon = new Coupon { DiscountType = DiscountType.Percentage, DiscountValue = 10m }; // SALE10

    // 150 × 0.9 = 135 → ×1.10 = 148.5 → 四捨五入で 149（偶数丸めだと 148 になる）
    Assert.Equal(149m, _pricing.CalculateTotal(items, coupon));
}
```

- **ヒント段階**: ①`dotnet test` は全部緑なのに画面では合計が違う——既存テストはこのケースを試していない。②まず壊れを再現するテストを書く（どんな注文なら端数が出る？ `SALE10` 併用が鍵）。③金額計算はどの層・どのクラス？ ④`Math.Round` の第3引数（丸めモード）の意味を調べよ。
- **解答**: `MidpointRounding.AwayFromZero` に戻す。
- ※C# は `Math.Round` 既定が偶数丸め。他言語では罠が逆になる（カタログ「移植時の注意」参照）。

### A8. 割引で請求がマイナスになる（0円下限の欠落）
- **仕込み箇所**: `<root>/Services/PricingService.cs` `ApplyCoupon`、および `<root>/../tests/PricingServiceTests.cs`

「テストの穴」版なので**仕込みは2箇所**: ①クランプを外す ②それを検出する既存テストを配布ブランチから削除し、緑のまま出題する。

```diff
         var discounted = coupon.DiscountType switch
         {
             DiscountType.FixedAmount => subtotal - coupon.DiscountValue,
             DiscountType.Percentage => subtotal * (1m - coupon.DiscountValue / 100m),
             _ => subtotal
         };

-        // 割引で 0 円未満になっても請求は 0 円までとする
-        return discounted < 0m ? 0m : discounted;
+        // ← 仕込み: 0円クランプを外す（割引額 > 小計でマイナスがそのまま流れる）
+        return discounted;
```

```diff
 // tests/PricingServiceTests.cs から、クランプを検出する既存テストを削除する（緑のまま出題するため）
-    [Fact]
-    public void ApplyCoupon_never_goes_below_zero()
-    {
-        var coupon = new Coupon { DiscountType = DiscountType.FixedAmount, DiscountValue = 5000m };
-
-        Assert.Equal(0m, _pricing.ApplyCoupon(1800m, coupon));
-    }
```

- **再現**: カートでノート1点(300) + `WELCOME500`（定額500引き）→ 正: 割引後 −200 を 0 にクランプ → 税込 **0円** / バグ: (300−500)×1.10 = **−220円** がそのまま表示・成立。割引額より小計が大きい通常注文（例 マグカップ1200 + `WELCOME500` → 770）はこれまで通り正しい。
- **赤になるテスト**: **無い（上記②でクランプ検証テストを外したため全テスト緑）**。**まず受講者に再現テストを書かせてから直させる**（A7 と同じ進め方）。解答ブランチに追加しておくテスト（外したものを end-to-end 形で復活させると symptom に対応して分かりやすい）:

```csharp
[Fact]
public void CalculateTotal_clamps_to_zero_when_coupon_exceeds_subtotal()
{
    var items = Items((300m, 1)); // ノート1点 = 小計 300
    var coupon = new Coupon { DiscountType = DiscountType.FixedAmount, DiscountValue = 500m }; // WELCOME500

    // 割引後は 0 円未満 → 0 円にクランプ。税込も 0（バグだと (300-500)*1.10 = -220）
    Assert.Equal(0m, _pricing.CalculateTotal(items, coupon));
}
```

- **ヒント段階**: ①`dotnet test` は全部緑なのに、安い商品＋高額クーポンで合計がマイナスになる——既存テストはこのケース（割引額 > 小計）を試していない。②まず壊れを再現するテストを書く（どんな注文ならマイナスになる？ 小計を下回る定額クーポンが鍵）。③金額計算はどの層・どのクラス？ ④割引後の値に「下限（0円）」のガードはあるか？
- **解答**: `discounted < 0m ? 0m : discounted` のクランプを戻す。
- **UI再現性メモ**: クーポンコードはフロントの自由入力（`shop.js` が `couponEl.value.trim()` を送信）なので、安い商品1点＋小計超えの定額クーポンを UI から確実に再現できる。

---

## B. 仕様変更型（模範解・C# 触り所）

> 「正解」は一通りではない。下は触る層の指針。受講者にはテスト更新も依頼する。

### B1. クーポンに最低利用金額
- **触る層**: `Entities/Coupon.cs`（`MinSubtotal` 列追加）/ `Services/OrderService.cs`（`ResolveCouponAsync` か `CreateAsync` で検証）/ `Data/SeedData.cs`（seed 更新）/ DB ファイル削除で再 seed。
- **テスト**: 「3000 未満 + `SALE10` → `BusinessRuleException`」「3000 以上 → 適用」。

### B2. 送料の導入
- **触る層**: `Services/PricingService.cs`（`CalculateTotal` に送料加算）。丸め順序を明文化（税計算 → 丸め → 送料加算）。
- **テスト**: 既存 `CalculateTotal_*` の期待値更新 + 境界（4999/5000）。

### B3. 1商品あたりの注文上限
- **触る層**: `Services/OrderService.cs`（合算後 `requestedQuantities` の検証に1行）。
- **テスト**: 100 個 → 400 / 99 個 → OK。

### B4. 商品ごとの軽減税率
- **触る層**: `Entities/Product.cs`（`TaxRate` or `Category`）/ `Services/PricingService.cs`（明細ごと税率 → 合算）/ seed / DTO（必要なら）。
- **テスト**: 混在注文の税込合計。

---

## C. 機能追加型（C# 触り所）

### C1. 商品検索
- **触る層**: `Controllers/ProductsController.cs`（クエリ受け取り、薄く）/ `Repositories/ProductRepository.cs`（`Where(p => p.Name.Contains(keyword))`）/ Service。
- **フロント**: `training-frontend` に検索ボックス（別課題として切れる）。
- **テスト**: keyword 一致 / 不一致 / 空。

### C2. 注文ステータス更新 API
- **触る層**: `Controllers/OrdersController.cs`（`confirm` エンドポイント）/ `Services/OrderService.cs`（遷移ロジック）。
- **注意**: 完成形は作成時 `Confirmed` 固定。`Pending` を作る経路をどうするかは設計判断を委ねる。

### C3. 在庫切れの表示（フロント主体）
- **触る層**: `training-frontend/js/ui.js` 中心。**バックエンド改修不要**（API は既に `stock` を返す）。
- 全言語版で同一の課題（フロント共有）。

---

## D. 横断・チーム開発型（C# 触り所）

### D1. N+1 を体感する
- **仕込み**: `Repositories/OrderRepository.cs` `GetAllAsync` から `.Include(...).ThenInclude(...)` を外す。
- **観察**: `appsettings.Development.json` で EF の SQL ログを ON にし、注文一覧で `SELECT` 本数を見る。
- **解答**: `Include().ThenInclude()` を戻す。

### D2. CORS 事故の切り分け
- **仕込み**: `Program.cs` の CORS 許可オリジンを別ポートにする／`UseCors` を消す。
- **解答**: 正しいオリジンを許可。

### D3. PR レビュー演習
- **題材**: Controller に業務ロジックを書いた / 例外を握りつぶした / 命名規約違反、の PR を用意してレビューさせる。

---

## C# 版 運用メモ

### ブランチ運用（仕込みの進め方）

研修者が見る `challenge/<id>-<slug>` には**仕込み作業の途中コミットを残さない**。実装は別ブランチで行い、squash で1コミットに畳んでから取り込む。

```
main
 ├─ impl/<id>-<slug> を切る          ① 実装用ブランチ
 │    └─ 仕込み diff 適用 ＋（必要なら）再現テスト追加。途中コミットは複数でも雑でも可
 │
 └─ challenge/<id>-<slug> を切る     ② 研修者が見るブランチ（解く起点）
      └─ impl/<id>-<slug> を squash merge して取り込む（履歴は1コミットに畳む）
 impl/<id>-<slug> を削除             ③ 実装用ブランチは消す
```

- **②が配布対象**。研修者はここから自分用ブランチを切って解く。
- **squash merge** にする理由は、①の途中コミット群を②に残さないため。コミットを `初期実装` 等に偽装する必要はない（普通に課題セットアップと分かる体で良い）が、**作業履歴は畳んで隠す**。
- ただし `git diff main challenge/<id>` を打てば仕込み内容（＝解答）は出る。これは**仕様上避けられない**（壊れた状態と完成形 `main` の差分そのものが仕込みだから）。本教材は「**テストが赤 → 症状から原因を追う**」設計なので、main-diff で答え合わせするのはカンニング扱いとし、技術的秘匿はしない。完全秘匿が要る場合のみ、main 履歴を含まないスナップショットを配布する。
- 各課題は `main` から切り、**1課題につき仕込みは1つだけ**。
- 仕込み後 `dotnet test` で**意図したテストだけ**赤になることを確認してから配布。
- DB は `EnsureCreated()` + `SeedData`。モデルを変える B 系では受講者に「`src/training.db` を消して再起動」を案内（マイグレーション無しのため）。
- SDK パスは `~/.dotnet`（PATH 外）。`dotnet` 実行前に prepend が要る場合あり。
- 起動: `dotnet run --project src` / テスト: `dotnet test`。
