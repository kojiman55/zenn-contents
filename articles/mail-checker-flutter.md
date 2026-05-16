---
title: "Web版で作ったAPIをFlutterで叩いてiOS/Androidアプリを作った"
emoji: "📱"
type: "tech"
topics: ["flutter", "dart", "ios", "android", "api"]
published: true
---

前回、ビジネスメール添削WebアプリをReact + AWS Lambda + Gemini APIで作った。

https://zenn.dev/pickle/articles/mail-checker

バックエンドは API Gateway + Lambda で動いていて、どこからでも叩けるAPIになっている。せっかくなのでFlutterでモバイルアプリも作ってみた。

| iOS | Android |
|---|---|
| ![iOS入力](https://portfolio.eggsystems.jp/screenshots/mail-checker-ios-input.png) | ![Android入力](https://portfolio.eggsystems.jp/screenshots/mail-checker-app-input.jpg) |
| ![iOS結果](https://portfolio.eggsystems.jp/screenshots/mail-checker-ios-result.png) | ![Android結果](https://portfolio.eggsystems.jp/screenshots/mail-checker-app-result.jpg) |

---

## 何ができるか

Web版と同じ機能をiOS・Androidで提供する。

- メール本文を入力するとAIが添削結果を返す
- 指摘を「誤り / 改善提案 / 参考情報」の3種類で分類表示
- 各指摘に元の表現・修正案・理由を表示
- AIによる総評
- 添削後の全文を生成してコピー
- 日本語・英語の両方に対応

---

## 技術スタック

| 項目 | 内容 |
|---|---|
| フレームワーク | Flutter |
| 言語 | Dart |
| HTTP | `http` パッケージ |
| バックエンド | Web版と共通（API Gateway + Lambda） |

バックエンドには一切手を加えていない。

---

## 詰まったところ

### CORSの設定はFlutterには関係なかった

Web版を作るときにAPI GatewayのCORS設定でかなり時間を使った。オリジンの許可・プリフライトリクエストへの対応・Lambdaのレスポンスヘッダーの設定など、細かいところで何度もつまずいた。

FlutterからAPIを叩くときは、そのCORS設定が一切関係ない。

CORSはブラウザの仕様であり、ネイティブアプリには存在しない概念だ。Flutterアプリからのリクエストは `http` パッケージがそのままHTTPリクエストを投げるだけで、オリジンの検証も何もない。

```dart
final res = await http.post(
  Uri.parse('$_apiBase/review'),
  headers: {'Content-Type': 'application/json'},
  body: jsonEncode({'text': text, 'language': 'ja'}),
);
```

これだけで動く。API GatewayのCORS設定はWebのためにあり、Flutterには無関係。

### SafeAreaを忘れるとiOSのノッチに文字が被る

ヘッダーを実装して実機で確認したところ、iOSでタイトル文字がノッチ（Dynamic Island）のエリアに重なっていた。

`SafeArea` ウィジェットで囲むだけで解消するのだが、Webの感覚でいると存在を忘れがちだ。下部は `bottom: false` を指定してフッターが不必要に押し上げられないようにしている。

```dart
SafeArea(
  bottom: false,
  child: Row(
    children: [ /* ヘッダーの中身 */ ],
  ),
)
```

### タブレット・iPadでのレイアウト対応

スマートフォンは縦1カラム、タブレット・iPadは横2カラムにしたかった。`LayoutBuilder` で画面幅を取得して切り替えた。

```dart
LayoutBuilder(builder: (context, constraints) {
  final isWide = constraints.maxWidth >= 768;
  if (isWide) {
    return Row(
      children: [
        Expanded(child: _buildInputPanel()),
        Expanded(child: _buildResultPanel()),
      ],
    );
  }
  return _response != null ? _buildResultPanel() : _buildInputPanel();
})
```

スマートフォンは入力画面と結果画面を切り替え表示、タブレットは左右に並べる。Web版と同じロジックがそのまま使えた。

### モバイルの文字数制限はWebより短くした

Web版のバックエンドは5,000文字まで受け付けている。モバイルでそのまま5,000文字入力させると使い勝手が悪い。スマートフォンでビジネスメール全文を貼り付けることは少ないと判断して、500文字に絞った。

```dart
static const _maxLength = 500;
```

バックエンドには何も変更を加えず、フロントエンドだけで制御している。

---

## ファイル構成

最初は1ファイルに書いていたが、500行を超えたので分割した。

```
lib/
  main.dart         エントリーポイント・アプリ設定
  home_screen.dart  メイン画面（UI）
  api.dart          APIクライアント・サンプルテキスト
  models.dart       データモデル
```

---

## おわりに

バックエンドを共通化できたのが想定通りで、追加の工数はほぼFlutterのUI実装だけで済んだ。CORSがない分、Web版より接続まわりはむしろシンプルだった。

SafeAreaとレイアウト対応を除けば、Webの `fetch` をFlutterの `http` パッケージに置き換えただけで動いた。バックエンドが整っているなら、モバイル対応のコストは想像より低い。

ソースコードはこちら。

https://github.com/kojiman55/mail-checker-app

Web版の記事はこちら。

https://zenn.dev/pickle/articles/mail-checker
