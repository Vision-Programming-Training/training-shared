# フロントエンド課題 仕込み diff 集（training-frontend 版）

> このファイルは `plans/frontend/課題カタログ-frontend.md` の **実装**。各課題の「狙い・症状・不変条件・難度」はカタログを参照。ここには **実 diff・ブランチ名・再現・ヒント・解答**だけを置く。
> **取り扱い注意**: 解答 diff そのもの。受講者には渡さない。

`<root>` = `training-frontend/js`。各 diff は `main`（完成形）に対して「壊す向き（before→after）」で示す。**解答＝この diff を逆に当てて `main` に戻すこと**。
配布ブランチには `// ← 仕込み` のコメントは入れない（ここでは説明のために付けている）。コミットメッセージは `課題F<番号> セットアップ`。

| 課題 | ブランチ | 仕込みファイル |
|---|---|---|
| F1 | `challenge/f1-cancelled-status` | `entries/orders.js` |
| F2 | `challenge/f2-mystery-comma` | `entries/orders.js` |
| F3 | `challenge/f3-price-error` | `entries/admin.js` |
| F4 | `challenge/f4-same-product` | `ui/render.js` |
| F5 | `challenge/f5-cart-not-update` | `ui/render.js` |
| F6 | `challenge/f6-tab-highlight` | `ui/components/topbar.js` |
| F7 | `challenge/f7-checkout-missing` | `ui/render.js` |
| F8 | `challenge/f8-buttons-disabled` | `ui/render.js` |
| F9 | `challenge/f9-duplicate-products` | `ui/render.js` |
| F10 | `challenge/f10-add-button-missing` | `ui/render.js` |
| F11 | `challenge/f11-price-not-updated` | `entries/admin.js` |
| F12 | `challenge/f12-remove-broken` | `ui/render.js` |
| F13 | `challenge/f13-wrong-cancel` | `entries/orders.js` |
| F14 | `challenge/f14-order-undefined` | `entries/shop.js` |
| F15 | `challenge/f15-invalid-coupon` | `core/api.js` |
| F16 | `challenge/f16-double-order` | `entries/shop.js` |
| F17 | `challenge/f17-coupon-alert` | `ui/render.js` |

> seed の前提: 商品は「ノート / ボールペン / マグカップ / Tシャツ / ステッカー（在庫0）/ トートバッグ / キーホルダー」の順。初期注文は3件（すべて確定、うち2件は明細2行）。注文やキャンセルで DB の状態が変わるので、再現が崩れたら backend の DB を初期化（`training.db` を削除して再起動）する。

---

## トラック1: 言語の基礎

### F1. キャンセル済みの注文が「確定」に見える
- **仕込み箇所**: `<root>/entries/orders.js` `statusLabel`

```diff
 function statusLabel(status) {
-  if (status === "Confirmed") return "確定";
+  if (status = "Confirmed") return "確定"; // ← 仕込み: 比較ではなく代入
   if (status === "Cancelled") return "キャンセル済み";
```

- **再現**: 注文履歴で任意の注文をキャンセル → 再描画後、その注文のボタンは消えるがバッジは「確定」のまま。
- **ヒント段階**: ①ボタンは消えている＝`order.status` は `Cancelled` で届いている。ではバッジの文言はどこで作っている？ ②`=` と `===` の違いは？ `if (a = "x")` の条件式の値は何？
- **補足（講師向け）**: `order.status = "Confirmed" ? ... : ""` の形で仕込むと、代入より三項演算子が先に評価されて別の症状になるため、`statusLabel` に仕込んでいる。
- **解答**: `===` に戻す。

### F2. 注文明細の表の上に謎の「,」が出る
- **仕込み箇所**: `<root>/entries/orders.js` `renderItems`

```diff
           <td class="num">${yen(i.lineTotal)}</td>
         </tr>`
-    )
-    .join("");
+    ); // ← 仕込み: join を削除
```

- **再現**: 注文履歴を開く → 明細2行の注文（#1, #2）の表の上に「,」。明細1行の注文（#3）は正常。
- **ヒント段階**: ①「,」の数と明細行の数の関係は？ ②`rows` の中身を Console に出してみよう。配列を文字列に埋め込むと何が起きる？（`` `${["a","b"]}` `` を Console で試す）
- **解答**: `.join("")` を戻す。
- **補足（講師向け）**: `<tbody>` 直下の文字は HTML パーサが表の外（前）へ追い出すため、「表の上」に出る。

### F3. 管理画面で正しい価格を入れてもエラーになる
- **仕込み箇所**: `<root>/entries/admin.js` `savePrice`

```diff
 async function savePrice(product, rawValue, button) {
-  const price = Number(rawValue);
+  const price = rawValue; // ← 仕込み: 文字列のまま
   if (!Number.isFinite(price) || price < 0) {
```

- **再現**: 管理画面でノートの価格を 500 にして「価格を更新」→ 必ずエラー。
- **ヒント段階**: ①`price` の値と**型**をブレークポイントで見よう（`typeof price`）②`Number.isFinite("500")` と `Number.isFinite(500)` を Console で比べよう。
- **解答**: `Number(rawValue)` に戻す。
- **発展**: 空欄で更新すると `Number("") === 0` で 0 円になる（`main` の既存挙動）。気づいた受講者には別 Issue として切り出させる。

### F4. どの「カートに追加」を押しても同じ商品が入る
- **仕込み箇所**: `<root>/ui/render.js` `renderProducts`

```diff
   productsEl.innerHTML = "";
-  for (const product of products) {
+  for (var i = 0; i < products.length; i++) { // ← 仕込み
+    var product = products[i];                 // ← 仕込み: var は関数スコープ
     const soldOut = product.stock <= 0;
```

- **再現**: ノートの「カートに追加」→ カートにキーホルダーが入る。どのボタンでも同じ。
- **ヒント段階**: ①`onAdd(product)` の行にブレークポイント。押したボタンと `product` は一致している？ ②`var` と `const`/`let` のスコープの違い。ループが終わった時点で `product` は何を指している？
- **解答**: `for (const product of products)` に戻す（`var` → `let` でも直るが、最小修正＝元の書き方に揃える）。

---

## トラック2: DOM

### F5. 商品を追加してもカートの小計・注文欄が出ない
- **仕込み箇所**: `<root>/ui/render.js` 画面要素の取得

```diff
-const cartTotalEl = document.getElementById("cart-total");
+const cartTotalEl = document.getElementById("cart-totals"); // ← 仕込み: id の綴り違い
```

- **再現**: 店舗ページを開いた時点で Console に `TypeError: Cannot set properties of null (setting 'textContent')`。カートに追加すると行は出るが小計・注文欄が出ない。
- **ヒント段階**: ①Console のエラーをクリックして該当行へ。`null` なのはどの変数？ ②その変数はどこで代入されている？ Elements パネルで HTML の実際の `id` と比べよう。
- **解答**: `"cart-total"` に戻す。

### F6. ヘッダーで今いるページのタブがハイライトされない
- **仕込み箇所**: `<root>/ui/components/topbar.js` `mountTopbar`

```diff
-    const cls = isActive ? "nav-link active" : "nav-link";
+    const cls = isActive ? "nav-link-active" : "nav-link"; // ← 仕込み: class が1つの別名に
```

- **再現**: どのページでも現在タブが青くならず、そのタブだけ余白などの見た目も崩れる。
- **ヒント段階**: ①Elements パネルで現在タブの `<a>` の `class` を見て、他のタブと比べよう ②`common.css` でハイライトしているセレクタは？ `class="a b"` と `class="a-b"` の違いは？
- **解答**: `"nav-link active"` に戻す。

### F7. カートに入れてもクーポン欄・注文ボタンが出てこない
- **仕込み箇所**: `<root>/ui/render.js` `renderCart`

```diff
   cartTotalEl.textContent = `税抜小計（目安）: ${yen(subtotal)}`;
-  checkoutEl.style.display = "block";
+  // ← 仕込み: 表示に戻す行を削除
 }
```

- **再現**: 商品を追加 → 行と小計は出るが注文欄が出ない。Console は無エラー。
- **ヒント段階**: ①Elements で注文欄（`#checkout`）を探す。DOM には存在する？ 見えない理由は？ ②`display:none` はどこで付いていて、どこで外すべき？
- **解答**: `checkoutEl.style.display = "block";` を戻す。

### F8. 在庫があるのに全商品のボタンが押せない
- **仕込み箇所**: `<root>/ui/render.js` `renderProducts`

```diff
     button.textContent = soldOut ? "在庫切れ" : "カートに追加";
-    button.disabled = soldOut;
+    button.setAttribute("disabled", soldOut); // ← 仕込み: 属性は "false" でも有効
```

- **再現**: 店舗ページの全ボタンがグレーで押せない（文言は「カートに追加」）。
- **ヒント段階**: ①Elements でボタンの HTML を見よう。`disabled="false"` と書いてあるのに無効なのはなぜ？ ②属性（HTML に書かれる文字列）とプロパティ（JS オブジェクトの値）の違い。`button.disabled` を Console で見てみよう。
- **解答**: `button.disabled = soldOut;` に戻す（`removeAttribute` との条件分岐でも直るが、元の書き方が最小）。

### F9. 注文するたびに商品一覧が増殖する
- **仕込み箇所**: `<root>/ui/render.js` `renderProducts`

```diff
-  productsEl.innerHTML = "";
+  // ← 仕込み: コンテナのクリアを削除
   for (const product of products) {
```

- **再現**: 起動直後「読み込み中...」が残ったまま一覧が出る → 注文確定のたびに一覧が増える。
- **ヒント段階**: ①注文後に `renderProducts` は何回呼ばれる？ そのとき `#products` の中身はどうなっている？ ②`renderCoupons` や `renderCart` と書き方を見比べよう。
- **解答**: `productsEl.innerHTML = "";` を戻す。

### F10. 「カートに追加」ボタンが最後の商品にしか無い
- **仕込み箇所**: `<root>/ui/render.js` `renderProducts`

```diff
   productsEl.innerHTML = "";
+  const button = document.createElement("button"); // ← 仕込み: ループの外で1個だけ作る
   for (const product of products) {
@@
-    const button = document.createElement("button");
     button.textContent = soldOut ? "在庫切れ" : "カートに追加";
```

- **再現**: ボタンがキーホルダーにだけある。押すとノート〜キーホルダーが一度に入り、ステッカーの在庫アラートも出る。
- **ヒント段階**: ①`createElement` は何回呼ばれている？ `appendChild` で既にどこかにある要素を別の親に追加すると？ ②押すと全商品が入るのはなぜ？ Event Listeners パネルでボタンに付いているリスナーの数を見よう。
- **解答**: `createElement` をループ内に戻す。

### F11. 管理画面で価格を変えても元の価格のまま「更新しました」
- **仕込み箇所**: `<root>/entries/admin.js` `renderProducts`

```diff
     button.textContent = "価格を更新";
-    button.addEventListener("click", () => savePrice(product, input.value, button));
+    const value = input.value; // ← 仕込み: 描画時に読んでしまう
+    button.addEventListener("click", () => savePrice(product, value, button));
```

- **再現**: ノートの価格を 500 に書き換えて更新 →「価格を ¥300 に更新しました」。
- **ヒント段階**: ①Network で PUT のリクエスト本文（Payload）を見よう。送っている価格は？ ②`value` を読んでいるのは「描画したとき」？「押したとき」？
- **解答**: クリック時に `input.value` を読む形に戻す。

### F12. カートの「削除」ボタンが全部効かない
- **仕込み箇所**: `<root>/ui/render.js` `renderCart`

```diff
     row.appendChild(right);
     cartEl.appendChild(row);
   }
+  cartEl.innerHTML += `<p class="muted">${items.length} 種類</p>`; // ← 仕込み: 件数表示の追加でリスナーが消える
 
   // 税抜小計の目安（確定金額は税込・クーポン適用後でサーバーが計算して返す）
```

- **再現**: 商品を2種類入れる →「2 種類」と出る → どの「削除」も無反応、エラーなし。
- **ヒント段階**: ①Event Listeners パネルで「削除」ボタンを見よう。click リスナーは付いている？ ②`innerHTML +=` は「足す」ではなく何をしている？（`x = x + ...` に分解して考える）
- **解答**: 件数表示を残す前提で、`appendChild`（`createElement("p")` + `textContent`）または `insertAdjacentHTML("beforeend", ...)` にする。**件数表示自体は消さない**のがポイント（機能追加の意図は活かして直す）。

### F13. 注文のキャンセルで別の注文がキャンセルされる
- **仕込み箇所**: `<root>/entries/orders.js` `renderOrders`

```diff
-    const cancelBtn = card.querySelector(".cancel-btn");
+    const cancelBtn = document.querySelector(".cancel-btn"); // ← 仕込み: 検索の起点が document
     if (cancelBtn) {
```

- **再現**: 一番上の注文の「注文をキャンセル」→ 2件目・3件目の注文番号で確認ダイアログが続けて出る。2件目以降のボタンは無反応。
- **ヒント段階**: ①1周目のループで `cancelBtn` は何になっている？（その時点で `card` は画面に追加済み？）②Event Listeners パネルで各ボタンのリスナー数を見よう。どのボタンに集まっている？
- **解答**: `card.querySelector` に戻す。

---

## トラック3: 非同期と HTTP

### F14. 注文完了メッセージが「undefined」「¥NaN」になる
- **仕込み箇所**: `<root>/entries/shop.js` `submitOrder`

```diff
-    const order = await postOrder(items, couponEl.value.trim());
+    const order = postOrder(items, couponEl.value.trim()); // ← 仕込み: await なし
```

- **再現**: 普通に注文 →「注文番号: undefined … ¥NaN」（注文履歴には作られている）。`SALE99` で注文 → 同じ「完了」表示でカートが空に、Console に `Uncaught (in promise) Error: クーポンが無効です (Code: SALE99)`。
- **ヒント段階**: ①`order` の中身をブレークポイントで見よう。何のオブジェクト？ ②失敗時に `catch` に入らないのはなぜ？ `async` 関数が返すものは？
- **解答**: `await` を戻す。

### F15. 無効なクーポンでも「注文が完了しました」
- **仕込み箇所**: `<root>/core/api.js` `postOrder`

```diff
   const data = await res.json().catch(() => null);
-  if (!res.ok) {
-    const message = data && data.error ? data.error : `注文に失敗しました (HTTP ${res.status})`;
-    throw new Error(message);
-  }
+  // ← 仕込み: HTTP ステータスの判定を削除
   return data;
```

- **再現**: `SALE99` で注文 →「注文が完了しました！注文番号: undefined / ステータス: 400 / ¥NaN」。
- **ヒント段階**: ①Network で `POST /api/orders` のステータスとレスポンス本文を見よう。サーバーは何と言っている？ ②`fetch` は 400 のとき例外を投げる？ 他の関数（`cancelOrder` など）はどう判定している？
- **解答**: `if (!res.ok) { ... throw }` を戻す。

### F16. 注文ボタンの連打で同じ注文が2件できる
- **仕込み箇所**: `<root>/entries/shop.js` `submitOrder`

```diff
-  orderBtn.disabled = true;
-
+  // ← 仕込み: 通信中の無効化を削除（配布ブランチでは直後の空行ごと削除）
   try {
```

- **再現**: DevTools の Network でスロットリングを「Slow 4G」等にし、注文ボタンを素早く2回押す → 注文履歴に同じ内容が2件。
- **ヒント段階**: ①Network で `POST /api/orders` は何本出ている？ ②1回目の通信が終わるまでの間、ボタンはどんな状態であるべき？ `finally` で何を戻している？
- **解答**: `orderBtn.disabled = true;` を戻す。

---

## トラック4: セキュリティ

### F17. クーポン欄に入れた文字でアラートが出る（XSS）
- **仕込み箇所**: `<root>/ui/render.js` `showMessage`

```diff
   div.className = `message ${type}`;
-  div.textContent = text;
+  div.innerHTML = text; // ← 仕込み: 文字列を HTML として解釈
   resultEl.appendChild(div);
```

- **再現**: クーポン欄に `<img src=x onerror=alert(1)>` → 注文 → アラート。サーバーの 400 エラー文 `クーポンが無効です (Code: <img ...>)` がそのまま HTML として挿入される。
- **ヒント段階**: ①Elements でエラーメッセージの中を見よう。`<img>` 要素ができていない？ ②`textContent` と `innerHTML` の違い。サーバーが返した文字列は安全と言える？
- **解答**: `textContent` に戻す。
- **発展**: 「サーバー側でエラー文にコードを含めない」も対策になりうるが、**フロントの責務として表示側で無害化する**のが本筋、と議論させる。

---

## FC. 機能追加型 — レビュー観点（講師用）

正解コードは置かない。PR レビューで以下を確認する。

| 課題 | 層の分離で見るところ | よくある NG |
|---|---|---|
| FC1 | 取得済み一覧を `entries/shop.js` で保持し、絞り込み結果を `ui.renderProducts` に渡す | 入力のたびに `fetchProducts()` を呼ぶ / `render.js` 内で絞り込みロジックを持つ |
| FC2 | 数量変更は `core/cart.js` の純粋関数。`render.js` はハンドラを受け取るだけ | `render.js` から `cart.js` を import する / 在庫上限チェックを DOM 側に書く |
| FC3 | 開閉は表示だけの関心なので `orders.js`（＋CSS）で完結 | 開閉のたびに注文を再取得する / `aria-expanded` を更新しない |
| FC4 | 既存 `savePrice` を呼ぶ | 更新処理を keydown 用に複製する / IME 確定の Enter で送信される |
| FC5 | `ui/components/toast.js` が自分で DOM を作る（topbar と同じ作法）。スタイルは `common.css` | ページごとに別実装 / 消えた要素が DOM に残る / `innerHTML` に未エスケープで文言を入れる |
