---
title: "2つのサイトをAIで比較するChrome拡張を作ってWeb Storeに公開するまで"
emoji: "🔀"
type: "tech"
topics: ["chrome拡張機能", "javascript", "gemini", "manifest_v3"]
published: true
---

ECサイトで商品を比較するとき、毎回タブを2枚開いてスクロールして見比べるのが地味につらい。仕様書と公式サイトを並べたいとき、ライバル社のサービスを比較したいとき、どうしてもウィンドウを行ったり来たりしてしまう。

それなら2つのサイトを並べて表示して、AIに比較させればいい——そう思って作り始めたのが DualyzeAI だ。

## 何ができる拡張機能か

Chrome のツールバーアイコンをクリックすると、2カラムのビューアが新しいタブで開く。左右にURLを入力すると2つのサイトが並列表示され、AI分析ボタンを押すと Gemini が両サイトの内容を比較・分析してくれる。

![スマートフォン2機種を並べて比較しているところ](https://dualyzeai.com/assets/screenshot_1_phones_compare.png)
*比較分析モード：Google Pixel と Samsung Galaxy を並べてAIが比較表を生成*

分析モードは6種類。

| モード | 用途 |
|--------|------|
| 比較・分析 | 2サイトの総合的な違いを整理 |
| 価格比較 | 料金・スペックを表で比較 |
| メリット・デメリット | 長所短所をリスト化 |
| おすすめ判定 | どちらが目的に合うか判定 |
| 要約 | 両サイトの内容をそれぞれ要約 |
| 要点抽出 | キーポイントを箇条書きで抽出 |

入力はサイト同士だけでなく「サイト + テキスト」「テキスト + テキスト」にも対応している。仕様書と公式ページの比較とか、2つのレビュー記事を並べて要点を抽出するといった使い方もできる。

APIキーはユーザー自身が Google AI Studio で取得した Gemini キーを使う BYOK（Bring Your Own Key）方式で、データが外部サーバーに送信されることはない。

## 技術的に一番難しかった部分

### declarativeNetRequest で iframe のブロックを解除する

2サイトを並べて表示するには iframe を使うのが自然な選択だが、多くのサイトは `X-Frame-Options: SAMEORIGIN` や `Content-Security-Policy: frame-ancestors 'none'` を設定していて iframe でのロードをブロックしている。Amazon も Google も Apple もほぼ全滅だ。

MV3 の `declarativeNetRequest` API を使えば、レスポンスヘッダーを書き換えることでこれを回避できる。

```json
{
  "id": 1,
  "priority": 1,
  "action": {
    "type": "modifyHeaders",
    "responseHeaders": [
      { "header": "X-Frame-Options", "operation": "remove" },
      { "header": "Content-Security-Policy", "operation": "remove" }
    ]
  },
  "condition": {
    "resourceTypes": ["sub_frame"]
  }
}
```

これを `rules.json` に記述して `manifest.json` で宣言するだけで、ほぼすべてのサイトを表示できるようになった。

### iframe 内のナビゲーションを追跡する

iframe 内でリンクをクリックしてページ遷移したとき、URLバーを自動更新したい。ところが `iframe.src` はリダイレクト後のURLを反映しないし、`getAllFrames` でURL完全一致を試みても、リダイレクトが挟まると一致しない。

解決策は `chrome.webNavigation.onCompleted` を使ってホスト名で照合することだった。

```javascript
chrome.webNavigation.onCompleted.addListener((details) => {
  if (details.frameId === 0) return; // メインフレームは無視
  const host = new URL(details.url).hostname.replace(/^www\./, '');

  // 初回ロード: ホスト名マッチで frameId を特定
  if (frame1Id === null && pendingHost1 && host === pendingHost1) {
    frame1Id = details.frameId;
    url1Input.value = details.url;
    return;
  }
  // 以降: frameId で追跡
  if (details.frameId === frame1Id) {
    url1Input.value = details.url;
  }
});
```

iframeをロードするときは期待するホスト名を `pendingHost1` に保存しておき、`onCompleted` が発火したらホスト名で照合して frameId を取得する。一度 frameId が分かれば、それ以降はそれで追跡できる。

### クォータエラーの処理

Gemini API の無料枠には RPM（1分あたりのリクエスト数）と RPD（1日あたりのリクエスト数）の上限がある。連続して使っているとすぐに 429 が返ってくる。

最初は自動フォールバック（Flash → Flash Lite → エラー）を実装したが、失敗するたびに新しいリクエストが発生してさらにクォータを消費するという悪循環になった。

結局、ダイアログでユーザーに確認してから切り替える方式に落ち着いた。

```javascript
if (error === 'QUOTA_EXCEEDED') {
  const fallback = failedModel.includes('flash-lite')
    ? 'gemini-2.5-flash'
    : 'gemini-2.5-flash-lite';

  const ok = confirm(`レート制限に達しました。${fallbackName}に切り替えますか？`);
  if (ok) {
    analyze({ modelOverride: fallback });
  } else {
    showCountdown(retryAfter); // リセットまでのカウントダウンを表示
  }
}
```

カウントダウンは `setInterval` ではなく `Date.now()` ベースで実装した。`setInterval` はバックグラウンドタブで止まることがあるので、`visibilitychange` イベントで復帰時に残り時間を再計算している。

![ラップトップ2機種のメリット・デメリット比較](https://dualyzeai.com/assets/screenshot_2_laptops_prosCons.png)
*メリット・デメリットモード：AI が両製品の長所・短所をそれぞれリストアップ*

## BYOK 設計にした理由

ユーザーに Gemini API キーを自分で用意してもらうのは、UX 的には少しハードルが高い。それでもこの設計を選んだ理由がある。

サーバー側でAPIキーを管理すると、誰かが大量にリクエストしたときのコストをデベロッパーが丸ごと負担することになる。個人開発で課金リスクを背負いながら無料公開するのは長続きしない。かといって有料にするには決済フローが必要で、それはそれで別の開発コストがかかる。

BYOK なら無料枠の管理もコストもユーザー側に委ねられる。Google AI Studio のキー取得は5分もあればできるし、日常的な使い方なら無料枠で十分まかなえる。

## Chrome Web Store の審査

申請から公開まで約6日かかった。審査中は何も連絡がなかったのでそわそわしたが、最終的にメールが来て一発通過だった。

審査で気をつけた点をまとめておく。

- **`declarativeNetRequest` の使用理由を明記する**: なぜヘッダーを書き換えるのかを "Why do you need this permission?" に丁寧に記述した。「ユーザーが選んだ任意のサイトを並列表示するため」というユースケースを具体的に説明した
- **プライバシーポリシーを用意する**: `dualyzeai.com/privacy.html` を作ってURLを登録した
- **APIキーの扱いを明示する**: BYOKであること、キーがサーバーに送信されないことを説明文に明記した

`declarativeNetRequest` でヘッダーを書き換えるのは審査で引っかかりやすいポイントだと思っていたが、ユースケースを丁寧に説明したのが良かったのかもしれない。

---

DualyzeAI は Chrome ウェブストアで公開中です。Gemini API キー（無料）があればすぐに使えます。

[Chrome Web Storeで公開中](https://chromewebstore.google.com/detail/dualyzeai/bokemejffloaffeejecamakicdlklhbp)

公式サイト: https://dualyzeai.com
