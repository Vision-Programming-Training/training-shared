---
name: backend-port-design
description: >-
  ミニEC研修アプリのサーバー側（training-backend-csharp / ASP.NET Core）を、別言語・別フレームワークへ
  移植するための設計書を生成する。「サーバーを Java / Node / Go / Python などへ移植したい」「別フレームワーク版の
  設計書を作りたい」「training-backend-<lang> を起こしたい」といった依頼で使う。起動時にターゲット言語・
  フレームワークが指定されていなければ必ず確認を取る。
---

# backend-port-design — サーバー移植設計書ジェネレータ

C# 参照実装（`training-backend-csharp`）を「正しく動く完成形＝仕様の基準」とみなし、その**言語非依存の契約**を抜き出して、指定された別言語・別フレームワーク版を実装するための**設計書（DESIGN）**を作る。

フロント（`training-frontend`）は全言語版で共有されるため、**移植版は既存フロントを 1 文字も変えずに接続できなければならない**。これがこのスキルの最重要ゴール。`config.js` の API ベース URL を差し替えるだけで動くこと。

---

## Step 1. ターゲット言語・フレームワークの確定（最初に必ず実施）

スキル起動時の引数・直前のユーザー発話を見て、**移植先の言語とフレームワークの両方**が明示されているか判定する。

- 両方が明示されている（例: 「Java の Spring Boot」「Node の NestJS」）→ そのまま Step 2 へ。
- 指定が無い／言語だけでフレームワーク未確定／曖昧 → **`AskUserQuestion` で確認を取る**。勝手に決めない。

確認の例（`AskUserQuestion`、header は "移植先" など）。選択肢は代表例で、ユーザーは「Other」で自由入力できる：

- **Java / Spring Boot** — 型あり・3層がそのまま写せる。研修教材の定番。
- **TypeScript / NestJS** — Controller/Service/Module の3層が C# 構成に近い。
- **Go / 標準 net/http + sqlc 等** — 軽量・型あり。DI は手書き。
- **Python / FastAPI** — 記述量が少ない。型ヒント + Pydantic。

ORM・DB・テストフレームワークまで指定が無ければ、その言語の定番を提案し、設計書の「技術スタック」章で確定させる（DB は SQLite 相当のファイル DB か、起動時自動作成＋seed が回るものを既定とする）。

> 確認すべきは「移植先の言語・フレームワーク」。すでに明示されていれば質問は省略してよい（過剰確認しない）。出力先パスも不明なら併せて確認する。

---

## Step 2. 参照実装から契約を抽出する

設計書は記憶でなく**実コードから**起こす。次のファイルを読んで現在の仕様を確認する（パスは `training-backend-csharp/src/` 配下）：

- `Program.cs` — 起動・DI・CORS・Swagger・起動時 seed の流れ
- `Controllers/ProductsController.cs`, `Controllers/OrdersController.cs` — エンドポイントと薄さ
- `Services/PricingService.cs` — **金額計算の核（最重要）**
- `Services/OrderService.cs` — 在庫・クーポン・注文作成/キャンセルの業務ルール
- `Repositories/*` — DB アクセス境界
- `Entities/*`, `Dtos/*` — データモデルと入出力の形
- `Middleware/ExceptionHandlingMiddleware.cs`, `Exceptions/AppExceptions.cs` — エラー → HTTP 変換
- `Data/SeedData.cs` — 初期データの実値
- `training-shared/plans/common/アプリ仕様.md` — 言語非依存の契約（API・業務ロジック・テスト観点）の正
- `training-shared/plans/common/課題カタログ.md` — 言語非依存の研修課題カタログ（移植時の注意つき。解答情報を含むので扱い注意）
- `training-shared/plans/csharp/仕様_csharp.md` — C# 固有事項（バージョン・構成・起動・DB 初期化）
- `training-shared/db/データモデル.md` — 言語非依存の DB エンティティ表（型マッピングの基準）
- リポジトリルートの `README.md` — 方針・制約・ディレクトリ構成

抽出は下の「不変条件チェックリスト」で漏れを潰す。C# 実装がチェックリストと食い違っていたら**実コードを正**とし、その旨を設計書に注記する。

---

## Step 3. 設計書を生成する

下のテンプレートに沿って markdown の設計書を書く。出力先は既定で `training-backend-<slug>/DESIGN.md`（`<slug>` は言語の小文字、例 `training-backend-java`）。リポジトリをまだ作らないなら作業ディレクトリ直下の `設計書-training-backend-<slug>.md` でもよい。**最終的な出力先はユーザーに確認**してから書き込む。

書いたら、移植版で必ず緑にすべきテスト観点（後述）も設計書に含める。実装まで行うかはユーザーに確認する（このスキルの既定は**設計書まで**）。

---

## 不変条件チェックリスト（移植版が必ず守る契約）

言語・フレームワークが変わっても、この振る舞いが変わったら移植失敗。設計書に全項目を書き起こす。

### API（パス・メソッド・ステータス）
- `GET /api/products` → 200, `ProductDto[]`
- `GET /api/products/{id}` → 200 `ProductDto` / 存在しなければ 404
- `GET /api/orders` → 200, `OrderDto[]`（明細・商品名込み）
- `GET /api/orders/{id}` → 200 `OrderDto` / 404
- `POST /api/orders` → 注文作成、`OrderDto` を返す（本文 `CreateOrderRequest`）
- `POST /api/orders/{id}/cancel` → 在庫を戻し Cancelled に、`OrderDto` を返す / 404 / 400

### 入出力の形（JSON フィールド名・型）
- `ProductDto`: `{ id:int, name:string, price:number, stock:int }`
- `CreateOrderRequest`: `{ items: [{ productId:int, quantity:int }], couponCode?: string|null }`
- `OrderItemDto`: `{ productId:int, productName:string, quantity:int, unitPrice:number, lineTotal:number }`
- `OrderDto`: `{ id:int, status:string, totalAmount:number, couponCode:string|null, createdAt:datetime, items: OrderItemDto[] }`
- `status` は文字列で `"Pending" | "Confirmed" | "Cancelled"`。
- フィールド名・大文字小文字・null 許容は**フロントが依存するため厳守**（既存フロントが読める形であること）。

### 金額計算（PricingService — 最重要・テスト必須）
1. 税抜小計 `subtotal = Σ (unitPrice × quantity)`
2. クーポン適用：
   - `FixedAmount` → `subtotal − discountValue`
   - `Percentage` → `subtotal × (1 − discountValue / 100)`
   - 割引後が負なら **0 にクランプ**
3. 税込合計 `total = round(discounted × 1.10, 円未満四捨五入)` — **0.5 は切り上げ（AwayFromZero）**。Banker's rounding（偶数丸め）禁止。
   - 検証値: `(600+1200)×1.10 = 1980` / `(2950−500)×1.10 = 2695` / `(1800−10%)×1.10 = 1782`
4. 通貨は整数円。浮動小数の丸め癖に注意し、`decimal`/固定小数点相当で計算する。

### 業務ルール（OrderService）
- 明細が空 → 400「注文には商品を 1 つ以上含めてください。」
- 同一 `productId` が複数行 → **数量を合算してから**在庫チェック。
- 合算後の数量が 0 以下 → 400。
- 商品が存在しない → 404。
- `在庫 < 要求数量` → 400（在庫不足）。
- 注文確定で**在庫を減算**。
- `OrderItem.unitPrice` は**注文時点の商品単価のスナップショット**（後で商品価格が変わっても注文は変わらない）。
- クーポンコードが空/空白 → クーポン無し。無効なコード → 400。
- 作成時 `status = Confirmed`、`createdAt = UTC now`。
- キャンセル: 注文なし → 404 / 既に Cancelled → 400 / 在庫を戻して `status = Cancelled`。

### エラーレスポンス
- 形: `{ "status": <int>, "error": "<message>" }`、`Content-Type: application/json`。
- NotFound → 404 / 業務ルール違反・バリデーション → 400 / 未処理例外 → 500（メッセージは伏せる）。
- Controller は薄く保ち、例外 → HTTP 変換は**ミドルウェア相当の一箇所**に集約する。

### 初期データ（seed・実値を一致させる）
- 商品 7 件：ノート¥300/在庫50、ボールペン¥150/100、マグカップ¥1200/20、Tシャツ¥2500/10、**ステッカー¥200/在庫0（在庫切れ用）**、トートバッグ¥1800/15、キーホルダー¥600/30。
- クーポン 2 件：`WELCOME500`（定額 ¥500 off）、`SALE10`（定率 10%）。
- 既存注文 3 件（一覧・明細確認用、合計は上の検証値）。
- 起動時に一度だけ投入し、既にデータがあれば二重投入しない。

### インフラ・構成（言語が変わっても満たす）
- **3層分離**：Controller（薄い）/ Service（業務ロジック）/ Repository（DB アクセス）。
- **CORS**：フロントのオリジンを設定で許可（ヘッダ・メソッドは全許可相当）。既存フロントがそのまま fetch できること。
- **DB**：インストール不要のファイル DB（SQLite 相当）＋起動時に自動作成＆seed。`<run コマンド>` 一発で起動〜seed まで完了する。
- **API ドキュメント**：Swagger/OpenAPI 相当の自動生成 UI。
- **テスト**：金額計算・クーポン・在庫・注文作成・キャンセルを網羅し全て緑。
- **リポジトリ整備**：README（概要・2リポジトリ手順・CORS の一言・3層説明）、CONTRIBUTING、PR テンプレート、CI（push/PR でテスト自動実行）。
- バージョン固定し README に明記（資材の陳腐化防止）。

---

## 設計書テンプレート（章立て）

```
# training-backend-<slug> 実装設計書（ミニEC / <言語> + <フレームワーク>）

## 0. このドキュメントの位置づけ
  - C# 完成形を仕様の基準とし、それと同じ振る舞いを <言語/FW> で実装するための設計書である旨
  - フロント（training-frontend）は無改修で接続できること（config.js の URL 切替のみ）

## 1. 技術スタック（採用と理由）
  | 項目 | C#版 | 本移植版 | 備考 |
  バックエンド / ORM / DB / API ドキュメント / テスト / DI

## 2. プロジェクト構成（ディレクトリ）
  - 3層がどのディレクトリ・モジュールに対応するか（C#版との対応表）

## 3. データモデル
  - Product / Order / OrderItem / Coupon のフィールドと型（言語の型へマッピング）
  - enum（OrderStatus / DiscountType）の表現方法

## 4. API 仕様
  - エンドポイント表 + 各リクエスト/レスポンスの JSON 例（フィールド名厳守）
  - エラーレスポンスの形と HTTP ステータス対応

## 5. 業務ロジック仕様
  - 金額計算（3ステップ + 丸め + 検証値）
  - 注文作成のルール（在庫・数量合算・スナップショット・クーポン）
  - キャンセルのルール
  ※「不変条件チェックリスト」を漏れなく落とし込む

## 6. 横断的関心事
  - エラーハンドリング（例外 → HTTP の集約方法）
  - CORS 設定
  - DB 初期化 + seed（実値）

## 7. テスト計画
  - 緑にすべきテストケース一覧（金額/クーポン/在庫/作成/キャンセル/404/400）

## 8. 環境構築・リポジトリ整備
  - 起動手順（1コマンド）、API ドキュメント URL、README/CI/PR テンプレート/CONTRIBUTING

## 9. C#版との差分メモ
  - 言語/FW 都合でやり方が変わる点（DI の手書き、ORM 差、丸め API 差など）と、それでも契約は不変である旨
```

---

## 進め方の既定
- 既定は**設計書の生成まで**。実装・リポジトリ作成まで進めるかは別途ユーザーに確認する。
- C# 版を一切壊さない（参照専用）。
- 言語固有の都合で契約を曲げない。曲げたくなったら不変条件を優先し、設計書の「差分メモ」に理由を書く。
