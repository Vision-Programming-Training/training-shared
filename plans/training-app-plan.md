# 既存改修トレーニング用アプリ 実装計画書（ミニEC / ASP.NET Core）

> このドキュメントは Claude Code に実装を進めてもらうための計画書です。
> 「ゼロから作る」研修ではなく、**「既に動いている既存プロジェクトに手を入れて直す」感覚**を新人に身につけさせるための教材を、最終的には作ります。
>
> **ただし本計画のスコープは「正しく動く完成形（`main`）を 1 本作るところまで」**。どこを壊して研修課題にするかは、完成形ができてから別工程で検討します。この完成形が、後の課題版の「正しい挙動の基準＝解答」になります。

---

## 1. プロジェクトの目的

SES 配属前の新人・未経験寄りエンジニアに、**既存コードを読んで・探して・直す**実務感覚を身につけさせる研修教材を作る。

参考イメージはゆめみの iOS 研修（天気アプリをセッション単位で課題化）。ただし本研修は **greenfield（ゼロから作る）ではなく brownfield（出来上がった保守）型**に振り切る点が最大の違い。

### 研修で身につけるスキル（言語非依存）

配属先の言語は読めないため、特定言語スキルではなく、どの現場でも通用する地力を狙う：

- 他人が書いた、もう動いているコードを読んで全体像をつかむ
- 「直したい挙動はどのファイル・どの層にある？」を探し当てる
- 既存の作法に合わせて、最小限だけ直す（俺流で書き換えない）
- テストと git で「壊していないこと」を担保する
- Issue / PR / レビューのチーム開発フローに乗る

---

## 2. 技術スタック

| 項目 | 採用技術 | 理由 |
|---|---|---|
| バックエンド | ASP.NET Core (C#) / .NET 8 LTS | 型あり・3層がきれいに書ける・`dotnet run` 一発で環境構築が軽い |
| ORM | EF Core | seed・マイグレーションが楽。N+1 課題も作れる |
| DB | SQLite | インストール不要・seed 同梱で環境構築の事故を防ぐ |
| API ドキュメント | Swashbuckle (Swagger UI) | 自動生成。フロント無しでも API を叩いて確認できる |
| テスト | xUnit | 「テストがグリーン＝直った目印」を新人に与える |
| フロントエンド | 素の HTML + JavaScript（`fetch`）／**別リポジトリ** | **クラサバの境界がコードに見える**。バック非依存で、サーバー言語を入れ替えても 1 個を使い回せる |

> **バージョンは固定し README に明記する**（資材が陳腐化しないように）。

### フロントエンドの方針（重要）

- フロントは**作り込まない**。商品一覧 → カート → 注文の **1 本道**だけ。
- `fetch("http://localhost:xxxx/api/...")` で backend を叩くだけにする。**この `fetch` の行＝クラサバの境界**であり、これが見えることが狙い。
- **Blazor は使わない**。Blazor Server はサーバー処理がメソッド呼び出しに偽装され、境界がコードから消えるため、本研修の狙いと相反する。
- 型共有・tRPC 的な「境界を隠す」仕組みも**使わない**。
- フロントは backend の言語に依存しないため、将来 Java 版・Node 版などに横展開しても**フロントは 1 個（`training-frontend`）で使い回せる**。接続先サーバーは `config.js` の API ベース URL で切り替える。
- フロントとサーバーはリポジトリが分かれている＝**別オリジン**になるため、ローカル実行時に **CORS** にぶつかる。サーバー側に CORS 許可設定を入れる必要がある（詳細はセクション 8）。これ自体が「クラサバが別物だとつなぐのに一手間いる」を体感させる教材にもなるが、新人が本題前に詰まらないよう**設定済みの状態で渡す**。

---

## 3. アプリ題材：ミニEC（注文 API）

題材は「商品をカートに入れて注文する」ミニ EC。ドメイン説明が不要で、計算ロジック・状態遷移・バリデーションが揃い、修正系の課題を仕込みやすい。

### リポジトリ構成（構成A：フロント1個 + サーバー言語ごとに別リポ）

サーバーサイドを別言語・別フレームワークに入れ替える可能性があるため、**フロントとサーバーをリポジトリ単位で分離**する。フロントは backend 非依存なので 1 リポジトリを全言語版で共有し、サーバーは言語ごとに独立リポジトリにする。

```
training-frontend                    # 【リポジトリ①】素の HTML/JS。全言語版で共有
├── index.html
├── app.js                           # fetch で API を叩くだけ
├── config.js                        # API のベース URL（接続先サーバーを切り替え）
└── README.md

training-backend-csharp              # 【リポジトリ②】ASP.NET Core 版（今回作るのはこれ）
├── src/
│   ├── Controllers/                 # API 入口（薄く保つ）
│   │   ├── ProductsController.cs
│   │   └── OrdersController.cs
│   ├── Services/                    # 業務ロジック（バグの主戦場）
│   │   ├── OrderService.cs
│   │   └── PricingService.cs
│   ├── Repositories/                # EF Core 経由の DB アクセス
│   │   ├── OrderRepository.cs
│   │   └── ProductRepository.cs
│   ├── Entities/                    # テーブル＝クラス
│   │   ├── Product.cs
│   │   ├── Order.cs
│   │   ├── OrderItem.cs
│   │   └── Coupon.cs
│   ├── Dtos/
│   └── Data/
│       ├── AppDbContext.cs
│       └── SeedData.cs              # 初期データ
├── tests/                           # xUnit
├── .github/
│   ├── pull_request_template.md
│   └── workflows/ci.yml             # テスト自動実行
└── README.md

（将来）training-backend-java         # Spring Boot 版
（将来）training-backend-node         # NestJS 版
```

> **今回 Claude Code で作るのは `training-frontend` と `training-backend-csharp` の 2 リポジトリ**。サーバー言語を入れ替える際は、フロントはそのまま、別のサーバーリポジトリを用意するだけで済む。
>
> データモデル・API 設計は言語非依存なので、サーバー言語を入れ替える際も同じ仕様を別リポジトリで実装すればよい。フロント側の `config.js` で接続先サーバーの URL を切り替えられるようにしておく。

### 層の役割

- **Controller**：リクエストを受けるだけ。意図的に薄く保つ（ロジックを書かない）。
- **Service**：業務ロジックの中心（合計金額・クーポン・在庫・注文/キャンセル）。
- **Repository**：EF Core で DB アクセス。

---

## 4. データモデル

```
Product   商品（名前・価格・在庫数）
Order     注文（ステータス・合計金額・作成日時）
OrderItem 注文明細（どの商品を何個）… Order と Product をつなぐ
Coupon    クーポン（割引ルール）… 仕様変更の波及ネタ
```

### フィールド定義（EF Core エンティティ）

**Product**
- `Id` (int, PK)
- `Name` (string)
- `Price` (decimal) … 税抜単価
- `Stock` (int) … 在庫数

**Order**
- `Id` (int, PK)
- `Status` (enum: `Pending` / `Confirmed` / `Cancelled`)
- `TotalAmount` (decimal) … 確定時の税込合計
- `CouponId` (int?, nullable, FK)
- `CreatedAt` (DateTime)
- `Items` (List<OrderItem>)

**OrderItem**
- `Id` (int, PK)
- `OrderId` (int, FK)
- `ProductId` (int, FK)
- `Quantity` (int)
- `UnitPrice` (decimal) … 注文時点の単価（スナップショット）

**Coupon**
- `Id` (int, PK)
- `Code` (string)
- `DiscountType` (enum: `FixedAmount` / `Percentage`)
- `DiscountValue` (decimal)

### seed データ

- 商品 5〜10 件（在庫 0 の商品を 1 件含める＝在庫切れ課題用）
- クーポン 2 件（定額・定率を各 1）
- 既存の注文を数件（一覧・N+1 課題が成立する程度の件数）

---

## 5. API 設計

| メソッド | パス | 概要 |
|---|---|---|
| GET | `/api/products` | 商品一覧 |
| GET | `/api/products/{id}` | 商品詳細 |
| GET | `/api/orders` | 注文一覧（明細・商品名込み）|
| GET | `/api/orders/{id}` | 注文詳細 |
| POST | `/api/orders` | 注文作成（商品ID・数量・クーポンコード）|
| POST | `/api/orders/{id}/cancel` | 注文キャンセル（在庫を戻し、ステータスを Cancelled に）|

> このフェーズでは上記すべてを**正しく実装する**（キャンセル API も含む）。どれを後で課題化するかは、完成形ができてから別途検討する。

---

## 6. 実装の重要方針（Claude Code 向け）

**このフェーズでは `main` を「正しく動く完成形」として素直に実装する**。仕込みバグや課題化は行わない（実装完了後に別途検討する）。

### 6-1. 正しく・素直に実装する

- 全機能が仕様どおり正しく動くこと。注文作成・合計金額（税込）・クーポン割引・在庫チェック・エラーハンドリング（存在しない ID は 404）など、**バグの無い状態**で実装する。
- 注文キャンセル API（`POST /api/orders/{id}/cancel`）も**実装する**（在庫を戻し、ステータスを Cancelled にする）。
- 3 層（Controller / Service / Repository）の責務分離は守る。Controller は薄く、ロジックは Service に置く。

### 6-2. テスト

- 主要なロジック（合計金額計算・クーポン適用・在庫チェック・注文作成）に **xUnit テストを用意し、全てグリーン**にする。
- このテスト群が、後で課題版を切り出すときの「正しい挙動の保証」になる。

### 6-3. コードの読みやすさ

- 普通に読みやすいコードで書く。新人が読んで追える構成・命名にする。
- （補足）将来ここから課題版を切り出す際、この正しい完成形が**そのまま解答・解説の基準**になる。正しい版との diff が課題の答えになるため、まずは綺麗な完成形を作ることを優先する。

> **課題化（どこを壊して研修課題にするか）は、この完成形ができてから別工程で検討する。** 本計画では扱わない。

---

## 7. 環境構築・リポジトリ整備

新人が初日につまずかないことを最優先する。

### セットアップ（2 リポジトリを clone）

受講者は「サーバー」と「フロント」の 2 リポジトリを clone する。

```bash
# サーバー（研修の主戦場）
git clone <training-backend-csharp>
cd training-backend-csharp
dotnet run        # 依存解決 → ビルド → 起動 → SQLite に seed まで自動

# フロント（別ターミナル／別ディレクトリ）
git clone <training-frontend>
cd training-frontend
# index.html を簡易サーバーで開く（例: VS Code Live Server など）
```

- DB は SQLite（seed 自動投入）。インストール作業を発生させない。
- Swagger UI はサーバー起動後に `/swagger` で自動表示。フロント無しでも API を叩いて確認できる。
- フロントの `config.js` に API のベース URL を書く（例: `http://localhost:5000`）。サーバーを入れ替えたらここだけ変える。

### CORS について（リポジトリ分離ゆえに必須）

フロント（例: `localhost:5500`）とサーバー（例: `localhost:5000`）はオリジンが異なるため、`fetch` がブラウザにブロックされる。**サーバー側に CORS 許可設定を入れた状態で渡す**こと。

- ASP.NET Core 側で、フロントのオリジンからのアクセスを許可する CORS ポリシーを設定する。
- README に「なぜこの設定が要るのか（フロントとサーバーが別オリジンだから）」を一言だけ添える。新人が概念として理解できればよく、設定作業自体は課題にしない。

### 整備するもの

PR・CI はサーバーリポジトリ（`training-backend-csharp`）側に置く。フロントリポジトリは最小限の README のみ。

サーバーリポジトリ：
- **README.md**：このアプリの概要（ミニ EC の注文 API）、2 リポジトリのセットアップ手順、CORS の一言説明、3 層構成の説明
- **CONTRIBUTING.md**：ブランチ命名規則、コミット・PR の作法
- **PR テンプレート**：何を変えたか／どう確認したか／影響範囲
- **CI（GitHub Actions）**：push / PR で `dotnet test` を自動実行。グリーンを目印にする

フロントリポジトリ：
- **README.md**：何のためのフロントか、`config.js` で接続先サーバーを切り替える方法、起動手順

> 研修の進め方（Issue・課題化・受講者向けの説明）は、課題版を設計するフェーズで別途整備する。

---

## 8. Claude Code への実装ステップ

以下の順で実装を進めてください。各ステップ完了後にコミットすること。**2 つのリポジトリ（`training-backend-csharp` と `training-frontend`）を作成する**。

**サーバーリポジトリ（training-backend-csharp）**

1. **プロジェクト雛形**：ASP.NET Core Web API のディレクトリ構成（Controllers / Services / Repositories / Entities / Data）。.gitignore、README の骨子。
2. **データモデルと DB**：EF Core エンティティ（Product / Order / OrderItem / Coupon）、`AppDbContext`、マイグレーション、`SeedData`（在庫 0 商品・クーポン・既存注文を含む）。
3. **正常系の API 実装**：商品一覧/詳細、注文一覧/詳細、注文作成。3 層（Controller / Service / Repository）で。Swagger 有効化。
4. **CORS 設定**：フロントのオリジンからのアクセスを許可する CORS ポリシーを設定（設定済みの状態にする）。
5. **正常系テスト**：xUnit で主要ロジック（合計金額・クーポン・在庫・注文作成・キャンセル）のテストを用意し、全てグリーンにする。
6. **リポジトリ整備**：CONTRIBUTING、PR テンプレート、CI ワークフロー（`dotnet test` 自動実行）、README 仕上げ。

**フロントリポジトリ（training-frontend）**

7. **最小 UI**：商品一覧 → カート → 注文の 1 本道。`fetch` で API を叩くだけ。`config.js` に API ベース URL を切り出す。README に起動手順と接続先切り替え方法を記載。

### 実装上の注意

- `main` は**正しく動く完成形**にする。バグを仕込んだり、わざと未実装にしたりしない。
- 全テストがグリーンの状態で完成とする。
- コードは普通に読みやすく書く（将来この完成形が課題版の基準になる）。

---

## 9. 横展開の考慮（将来）

SES で配属先が読めないため、本教材は 1 言語で完成させた後、他言語へ横展開できる構造にしておく。リポジトリを分けた（構成A）ことで、この横展開がそのまま実現できる。

- **サーバーは言語ごとに別リポジトリ**（`training-backend-csharp` / `-java` / `-node` …）。入れ替え＝別リポジトリを使うだけで、履歴も依存も完全に分離される。
- **フロントは `training-frontend` の 1 個を全言語版で共有**。`config.js` の API URL を切り替えるだけで、どのサーバーにも接続できる。新しいサーバー版を作ってもフロントは作り直さない。
- データモデル・API 設計は言語非依存なので、各サーバーリポジトリで同じ仕様を実装すればよい。
- まずは本計画（ASP.NET Core 版）のサーバー 1 本 + フロント 1 本を**正しい完成形**として作り、運用して手応えを確かめてから、課題化や横展開を検討する。
