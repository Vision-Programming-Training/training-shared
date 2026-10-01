# 理解度チェック 解答 — C# 版の付録（講師用）

> 言語非依存の解答・確認ポイントは [`../common/理解度チェック-解答.md`](../common/理解度チェック-解答.md) を参照。ここには **C# 版（`training-backend-csharp`）で答えが決まる箇所だけ**を書く。

## H1. 起動

| 項目 | 正解 |
|---|---|
| 起動コマンド | `dotnet run --project src`（リポジトリのルートで実行） |
| Swagger UI | `http://localhost:5000/swagger` |
| フロント | `http://localhost:5500`（Live Server）など。`appsettings.json` の `Cors:AllowedOrigins` に含まれるオリジンであること |

- フロントが「商品の取得に失敗しました」になる場合: バックエンドが起動していない、`config.js` の `API_BASE_URL` のずれ、許可されていないポートで開いている（CORS）のどれか。

## H2. テスト

| 項目 | 正解 |
|---|---|
| コマンド | `dotnet test` |
| 件数 | **19 件**（`PricingServiceTests` 7 件 + `OrderServiceTests` 12 件）、すべて成功 |

> テストを追加・削除したら、この件数も更新すること。

## H8. `GET /api/orders` の流れ

```
OrdersController.GetAll()                   … src/Controllers/OrdersController.cs
  → OrderService.GetAllAsync()              … src/Services/OrderService.cs
    → OrderRepository.GetAllAsync()         … src/Repositories/OrderRepository.cs（Include で明細・商品・クーポンをまとめて取得、作成日時の降順）
  ← OrderService.MapToDto()                 … エンティティ → OrderDto に詰め替え
← Ok(orders)                                … JSON で返す
```

- OK の目安: Controller → Service → Repository の3ファイルが順番どおりに書けていること。
- さらに: `MapToDto` での DTO への詰め替えや、Controller が interface（`IOrderService`）経由で Service を呼んでいること（DI）に触れていれば褒める。

## H9. DB リセット

`src/training.db`（と `training.db-shm` / `training.db-wal`）を削除してから、`dotnet run --project src` で再起動する。

- 削除できない場合はサーバーが起動したままになっている。`Ctrl + C` で止めてから削除する。
