# training-backend-csharp 実装仕様（C# 固有）

> ミニ EC 研修アプリの **ASP.NET Core (C#) リファレンス実装**の固有事項をまとめたもの。
>
> **データモデル・API・業務ロジック・テスト観点などの言語非依存な契約は [../common/アプリ仕様.md](../common/アプリ仕様.md) を正とする。** 本書はその契約を C# でどう実現しているか（バージョン・ディレクトリ・起動コマンド・DB 初期化の仕組み・C# 固有 API）だけを扱い、契約そのものは再掲しない。

---

## 1. 技術スタック（バージョン固定）

| 項目 | 採用技術 | バージョン | 備考 |
|---|---|---|---|
| 言語 / ランタイム | C# / .NET | 8.0 (LTS) | 型あり・3層がきれいに書ける・`dotnet run` 一発で軽い |
| Web | ASP.NET Core Web API | 8.0 | |
| ORM | EF Core（SQLite プロバイダ） | 8.0.8 | seed が楽。`Include` で N+1 を避けられる |
| DB | SQLite（ファイル） | - | インストール不要・seed 同梱で事故を防ぐ |
| API ドキュメント | Swashbuckle (Swagger UI) | 6.6.2 | 自動生成。`/swagger` で表示 |
| テスト | xUnit（+ EF Core InMemory） | 2.8.1 | 「グリーン＝直った目印」 |

> バージョンは固定し README にも明記する（資材が陳腐化しないように）。

---

## 2. プロジェクト構成

csproj は `src/` 直下に置き、ソリューションをリポジトリ直下に置く。起動・テストは後述のとおりリポジトリルートから実行する。

```
training-backend-csharp
├── src/                             # Web API プロジェクト（csproj はここ）
│   ├── Controllers/                 #   API 入口（薄く保つ）
│   │   ├── ProductsController.cs    #     商品一覧/詳細・価格変更
│   │   ├── CouponsController.cs     #     クーポン一覧
│   │   └── OrdersController.cs      #     注文一覧/詳細・作成・キャンセル
│   ├── Services/                    #   業務ロジック（インターフェース＋実装）
│   │   ├── IOrderService.cs / OrderService.cs      # 在庫・クーポン・注文/キャンセル
│   │   └── IPricingService.cs / PricingService.cs  # 金額計算（小計・割引・税込）
│   ├── Repositories/                #   EF Core 経由の DB アクセス（インターフェース＋実装）
│   │   ├── IProductRepository.cs / ProductRepository.cs
│   │   ├── IOrderRepository.cs / OrderRepository.cs
│   │   └── ICouponRepository.cs / CouponRepository.cs
│   ├── Entities/                    #   テーブル＝クラス
│   │   ├── Product.cs / Order.cs / OrderItem.cs / Coupon.cs
│   │   ├── OrderStatus.cs           #     enum（文字列で永続化）
│   │   └── DiscountType.cs          #     enum（文字列で永続化）
│   ├── Dtos/                        #   API の入出力の形（record）
│   ├── Data/
│   │   ├── AppDbContext.cs          #     DbSet 定義・decimal/enum のマッピング・FK
│   │   └── SeedData.cs              #     初期データ
│   ├── Exceptions/                  #   業務例外（NotFoundException / BusinessRuleException）
│   ├── Middleware/                  #   ExceptionHandlingMiddleware（例外 → HTTP）
│   └── Program.cs                   #   起動・DI・CORS・Swagger・DB 初期化
├── tests/                           # xUnit（OrderServiceTests / PricingServiceTests）
├── .github/
│   ├── pull_request_template.md
│   └── workflows/ci.yml
├── training-backend-csharp.sln
├── CONTRIBUTING.md
└── README.md
```

### 契約 → C# の対応（「どの層にあるか」を探す手がかり）

| 契約（[../common/アプリ仕様.md](../common/アプリ仕様.md)） | C# の置き場所 |
|---|---|
| API エンドポイント | `src/Controllers/*Controller.cs`（薄い。DI で Service/Repository を受け取るだけ） |
| 金額計算（小計・クーポン・税込・丸め） | `src/Services/PricingService.cs` |
| 在庫・数量合算・スナップショット・クーポン解決・キャンセル | `src/Services/OrderService.cs` |
| DB アクセス（eager loading 含む） | `src/Repositories/*Repository.cs` |
| 例外 → HTTP ステータス変換 | `src/Middleware/ExceptionHandlingMiddleware.cs`（`NotFoundException`→404 / `BusinessRuleException`→400 / それ以外→500） |
| 入力バリデーション | `src/Dtos/*` の DataAnnotations（`[Required]` / `[Range]` / `[MinLength]`） |

### C# 固有の実装メモ

- **DI**：`Program.cs` で Repository / Service をインターフェースに対して `AddScoped` 登録する。
- **金額の丸め**：`Math.Round(value, 0, MidpointRounding.AwayFromZero)`。`decimal` で計算する（銀行丸めの既定を使わない）。
- **N+1 回避**：`OrderRepository` で `Include(o => o.Items).ThenInclude(i => i.Product)` と `Include(o => o.Coupon)` を使う。
- **永続化マッピング**（`AppDbContext.OnModelCreating`）：`decimal` 列は `HasColumnType("decimal(18,2)")`、enum は `HasConversion<string>()`。SQLite は decimal をネイティブに扱えないため明示が必要。

---

## 3. DB 初期化と seed

- **EF Core マイグレーションは使わない。** 起動時に `Program.cs` で `db.Database.EnsureCreated()` → `SeedData.Initialize(db)` を 1 回だけ呼び、「スキーマ作成（無ければ作る）＋ seed」まで自動で済ませる。`dotnet run` 一発で起動〜seed まで完了する軽さを優先する方針。
- `SeedData.Initialize` は既にデータがあれば何もしない（再起動で二重投入しない）。投入する実値（商品 7 件・クーポン 2 件・既存注文 3 件）は [../../db/データモデル.md](../../db/データモデル.md) の「5. 初期データ」を正とする。
- DB ファイルは `src/training.db`。初期データに戻したいときは `training.db`（および `-shm` / `-wal`）を削除して再起動する。

---

## 4. 起動・コマンド

> すべてリポジトリのルート（`training-backend-csharp/` 直下）で実行する。前提：.NET 8 SDK（`dotnet --version` が `8.x`）。

| やりたいこと | コマンド |
|---|---|
| サーバーを起動（ビルド〜DB初期化まで自動） | `dotnet run --project src` |
| ビルドだけ | `dotnet build` |
| テスト実行 | `dotnet test` |
| 依存復元 | `dotnet restore` |
| DB を初期データに戻す | `src/training.db`（+ `-shm` / `-wal`）を削除して再度 `dotnet run --project src` |

- 起動ポートは `http://localhost:5000`（`src/Properties/launchSettings.json`）。Swagger UI は `http://localhost:5000/swagger` に自動で開く。
- 停止は実行中ターミナルで `Ctrl + C`。
- フロントを併用するときは `training-frontend/js/core/config.js` の `API_BASE_URL` を `http://localhost:5000`（既定）に合わせる。

---

## 5. CORS 設定（設定済みで渡す）

- ポリシーは `Program.cs` で定義し、許可オリジンは `src/appsettings.json` の `Cors:AllowedOrigins` に切り出す。
- 登録済みオリジン：`http://localhost:5500` / `http://127.0.0.1:5500` / `http://localhost:8080` / `http://127.0.0.1:8080`（Live Server / 簡易サーバーの定番ポート）。別ポートで開くときはここに追記する。
- CORS が必要な理由（フロントとサーバーが別オリジン）は [../common/アプリ仕様.md](../common/アプリ仕様.md) の「8. 環境構築・CORS」を参照。

---

## 6. テスト・CI

- テストは xUnit。テストごとに **EF Core InMemory プロバイダ**で独立した DB を作る（`UseInMemoryDatabase(Guid.NewGuid())`）。
- カバーする観点は [../common/アプリ仕様.md](../common/アプリ仕様.md) の「7. テスト観点」を正とする。実装ファイルは `tests/PricingServiceTests.cs` / `tests/OrderServiceTests.cs`。
- 実行：`dotnet test`。コードを直したら必ず実行し、全グリーンを保つ。
- **CI（GitHub Actions / `.github/workflows/ci.yml`）**：push / PR（`main`）で `dotnet restore` → `dotnet build --configuration Release` → `dotnet test` を自動実行する。

---

## 7. 開発の進め方

ブランチ・コミット・PR の作法は `training-backend-csharp` リポジトリ内の `CONTRIBUTING.md` を参照。課題版（仕込み diff）は [課題-仕込みdiff-csharp.md](./課題-仕込みdiff-csharp.md) で管理する。
