# BOOKS 請負報告フォーム（ukeoi-form）

請負大会の稼働報告をスマホから送る1ファイル完結のフォーム（`index.html`）。
バックエンドは勤務給与GAS（BOOKSSystem リポジトリの `clasp_kinmu/請負WebApp.gs`）の Webアプリ。

## PIN 認証（2026-09-23〜 全体レビュー R19）

| 項目 | 内容 |
|---|---|
| 入口 | 最初に 4桁の PIN 画面が出る。PIN はサーバー（`?action=verifyPin&pin=…&deviceId=…`）で照合し、12時間有効のトークンを受け取る |
| 保存 | トークンと有効期限を `localStorage`（`ukeoi_token` / `ukeoi_tokenExp`）に保存。期限の1分前からは PIN を再入力 |
| 送り方 | GET（スタッフ一覧・履歴）は `?token=…`、POST（報告の登録）は本文の `token`。GAS はHTTPヘッダーを読めないため |
| 期限切れ・無効 | サーバーが `{"error":"AUTH_REQUIRED"}` を返したらトークンを捨てて PIN 画面に戻る。送信時なら入力内容は画面に残り、PIN 再入力後にもう一度「送信する」を押す（前回分は登録されていない） |
| 秘密の置き場 | PIN・トークン鍵は GAS のスクリプトプロパティ（`UKEOI_PIN` / `UKEOI_TOKEN_SECRET`）だけ。このリポジトリ（公開）には書かない |
| 変えていないもの | `API_URL`、POST の `Content-Type: text/plain`、fetch オプション（`redirect` なし） |

## ブラウザでの簡易確認（route モック・本番へ通信しない）

`submitReport()` は本番スプレッドシートへの実書き込みなので、確認は必ず API をモックして行う。
Playwright の `page.route` で `script.google.com` 宛ての通信をすべて横取りし、架空の応答を返す（それ以外の外部通信は遮断）。

1. Playwright が入っている環境を用意する（例: `houkoku-form` の `node_modules`。`npm i -D playwright` でも可）。
2. 下のスクリプトを `ukeoi-verify.cjs` として保存し、`node ukeoi-verify.cjs C:/dev/ukeoi-form/index.html` で実行する。
3. 期待結果: 初回は PIN 画面／誤PINで「PINが正しくありません」／正しいPINで画面が閉じ、`token=` 付きでスタッフ一覧を取得／送信の `AUTH_REQUIRED` で PIN 画面に戻り、再入力後に送信すると完了画面／`localStorage` の期限を過去にして再読み込みすると PIN 画面。

```js
// ukeoi-verify.cjs（抜粋版。PIN・トークンは架空値）
const path = require('path');
const { chromium } = require('playwright');
const PIN = '2468', TOKEN = 'u1.mockdevice01.9999999999999.mocksig';
(async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage({ viewport: { width: 390, height: 844 } });
  let submitCount = 0;
  await page.route(/^https?:\/\//, async route => {
    const req = route.request(), url = new URL(req.url());
    if (url.hostname !== 'script.google.com') return route.abort();          // 本番・外部へは出さない
    const q = Object.fromEntries(url.searchParams);
    let body;
    if (q.action === 'verifyPin') body = q.pin === PIN ? { success: true, ok: true, token: TOKEN, expiresAt: Date.now() + 12 * 3600e3 } : { success: false, ok: false, error: 'INVALID_PIN' };
    else if (req.method() === 'GET') body = q.token !== TOKEN ? { success: false, error: 'AUTH_REQUIRED' }
      : q.action === 'getStaffList' ? { success: true, staffList: [{ name: 'テスト太郎01' }] } : { success: true, history: [] };
    else body = (++submitCount === 1 || JSON.parse(req.postData()).token !== TOKEN) ? { success: false, error: 'AUTH_REQUIRED' } : { success: true, message: '登録完了: 1名' };
    await route.fulfill({ status: 200, contentType: 'application/json', body: JSON.stringify(body) });
  });
  await page.goto('file:///' + path.resolve(process.argv[2]).replace(/\\/g, '/'));
  console.log('PIN画面:', await page.isVisible('#pin-gate'));
  for (let i = 0; i < 4; i++) await page.type('#pin-d' + i, PIN[i]);
  await page.waitForTimeout(500);
  console.log('PIN後に閉じる:', !(await page.isVisible('#pin-gate')));
  await page.fill('#reporter-name', 'テスト太郎01');
  await page.click('text=報告を作成する');
  await page.selectOption('#sn-1', 'テスト太郎01');
  await page.click('text=全員に適用');
  await page.fill('#rpt-project', 'TEST_案件01');
  await page.click('text=確認画面へ');
  await page.click('#btn-submit');                                          // 1回目は AUTH_REQUIRED を返す
  await page.waitForTimeout(400);
  console.log('AUTH_REQUIRED でPIN画面:', await page.isVisible('#pin-gate'));
  for (let i = 0; i < 4; i++) await page.type('#pin-d' + i, PIN[i]);
  await page.waitForTimeout(500);
  await page.click('#btn-submit');
  await page.waitForTimeout(400);
  console.log('完了画面:', await page.isVisible('#screen-done'));
  await browser.close();
})();
```

テスト用のプロジェクト名は `TEST_` で始める（勤務給与GAS のテスト削除関数 `cleanupTestRows_` は `TEST_` で始まる行だけを消す）。

## 本番反映の順番（勤務給与GAS とそろえる）

1. 勤務給与GAS のスクリプトプロパティに `UKEOI_PIN`（4桁）・`UKEOI_TOKEN_SECRET`（16文字以上）を登録
2. 勤務給与GAS を `clasp push` → 請負Webアプリの既存デプロイを版更新（URL は変わらない）
3. このリポジトリ（`index.html`）を反映

2 と 3 の間（数分）は、古いフォームでスタッフ一覧が出ない（サーバーが PIN を求めるため）。
