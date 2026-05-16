---
title: "住所1つで洪水リスクから避難経路まで返すAIを作った"
emoji: "⛑️"
type: "tech"
topics: ["nextjs", "typescript", "aws", "gemini", "rag"]
published: false
---

能登半島地震の報道を見ていて、ふと思った。自分が今いる場所の洪水リスクって、実際どのくらいなんだろう。

国土交通省のハザードマップポータルを開くと確かに情報はある。でも洪水のリスクを調べようとすると、市町村を選んで、地図レイヤーを切り替えて、住所を地図上で探して……というのが意外と手間だった。気象庁の警報は別のサイト。避難場所はまた別。

それなら住所を1回入れるだけで全部まとめて出てくるものを作ろう、と思ったのがこのシステムを作り始めたきっかけ。

デモはこちら。大阪府内の住所で動作確認できる。

https://disaster-rag.eggsystems.jp

![住所入力と災害リスク評価](https://raw.githubusercontent.com/kojiman55/disaster-rag/main/docs/screenshots/disaster-rag-top.jpeg)
*住所を入れるとリスク評価カードが並ぶ。大阪市北区梅田は洪水・津波・高潮がすべて「高」だった。*

---

## 何ができるか

住所を入力すると以下をまとめて返す。

- **ハザードリスク評価**（洪水・土砂・津波・高潮の4種を高・中・低で評価）
- **最寄り避難場所への徒歩経路**（地図上にルート表示、所要時間・距離付き）
- **気象庁のリアルタイム警報・注意報**
- **AIによる総合解説**（地域防災計画を参照してGeminiが回答）
- **チャット機能**（追加で何でも質問できる）

全部1回のAPIコールでまとめて処理して返している。

![ハザードマップと避難経路](https://raw.githubusercontent.com/kojiman55/disaster-rag/main/docs/screenshots/disaster-rag-map.jpeg)
*地図上に避難場所（緑）と徒歩経路（オレンジ点線）を重ねて表示する。*

![AIチャットと気象警報](https://raw.githubusercontent.com/kojiman55/disaster-rag/main/docs/screenshots/disaster-rag-chat.jpeg)
*気象庁のリアルタイム警報と、AIへの追加質問に答えるチャット画面。*

---

## 技術スタック

| レイヤー | 技術 |
|---|---|
| フロントエンド | Next.js 15 (App Router) / TypeScript / Tailwind CSS |
| ホスティング | S3 + CloudFront (OAC) |
| バックエンド | AWS Lambda (TypeScript) / API Gateway |
| AI | Gemini 2.0 Flash（回答生成・テキスト埋め込み） |
| ベクトル検索 | Qdrant（地域防災計画PDFのRAGインデックス） |
| 地図 | MapLibre GL JS / 国土地理院タイル |
| インフラ | AWS SAM / EventBridge |
| 外部API | 国土地理院 / 国土交通省 不動産情報ライブラリ / 気象庁 / OpenRouteService |

---

## 詰まったところ

### Reinfolibのデータは市区町村単位で存在しなかった

最初の設計はこういうものを考えていた。市区町村コードを使ってハザードデータをあらかじめS3に保存しておき、住所が来たらS3から引いて返す。

```
【最初の想定】
住所 → 市区町村コードを特定 → S3から該当コードのデータを取得 → 返す
```

ところがAPIの仕様書を読み直したら、国土交通省の不動産情報ライブラリAPIは `z/x/y` のタイル座標でアクセスする設計になっていた。市区町村コード単位のデータは存在しない。最初に想定していたS3事前保存の設計を丸ごと変えることになった。

```
【実際の実装】
住所 → 緯度経度 → タイル座標を計算 → タイルAPIを4種並行呼び出し → 返す
```

緯度経度からタイル座標を計算する関数を書いて、4種のAPIを `Promise.all` で並行取得するように実装した。

```typescript
function toTile(lat: number, lng: number, z: number) {
  const x = Math.floor((lng + 180) / 360 * Math.pow(2, z));
  const yRad = lat * Math.PI / 180;
  const y = Math.floor(
    (1 - Math.log(Math.tan(yRad) + 1 / Math.cos(yRad)) / Math.PI) / 2 * Math.pow(2, z)
  );
  return { x, y };
}

const [floodCount, stormCount, tsunamiCount, landslideCount] = await Promise.all([
  fetchTileCount("XKT026", lat, lng, apiKey), // 洪水
  fetchTileCount("XKT027", lat, lng, apiKey), // 高潮
  fetchTileCount("XKT028", lat, lng, apiKey), // 津波
  fetchTileCount("XKT029", lat, lng, apiKey), // 土砂
]);
```

結果的にはズームレベル14で呼ぶと1タイルが200〜300m四方に対応するため、丁目・番地レベルのピンポイント評価ができるようになった。市区町村単位の粗い評価よりずっといい。でも最初から仕様をちゃんと読んでいれば、余計な設計を考えずに済んだ。

| API ID | 用途 |
|---|---|
| XKT026 | 洪水浸水想定区域（想定最大規模） |
| XKT027 | 高潮浸水想定区域 |
| XKT028 | 津波浸水想定 |
| XKT029 | 土砂災害警戒区域 |

### 地図の座標が逆で、ピンが海の上に出た

国土地理院のジオコーディングAPIはGeoJSON仕様に従っていて、座標が `[lng, lat]` の順番になっている。

```typescript
const [lng, lat] = features[0].geometry.coordinates;
```

最初は `[lat, lng]` の順番で展開していたため、地図上のピンが日本海の沖に出ていた。MapLibreも `[lng, lat]` なのでそろえればいいだけの話なのだが、気づくまでに少し時間がかかった。

これを調べている過程で、地図タイルも国土地理院の淡色地図タイル（無料）が使えることを知った。Amazon Location Service で始めていたのだが、これで地図まわりのコストがゼロになった。

### 晴明丘公園がどの住所でも最寄り避難場所として出てきた

しばらくの間、どの住所で検索しても「晴明丘公園（大阪市北区）まで徒歩〇分」と表示されていた。

原因は単純で、避難場所データの先頭要素をそのまま使っていただけだった。最寄り計算を実装していなかった。

```typescript
// NG: データの先頭を返していた
const nearest = shelters[0];

// OK: ユークリッド距離で最寄りを選ぶ
const nearest = shelters.reduce((a, b) =>
  distance(lat, lng, a.lat, a.lng) < distance(lat, lng, b.lat, b.lng) ? a : b
);
```

### スロットリングを設定したら「Failed to fetch」が出るようになった

API Gatewayにスロットリング（レート2req/s・バースト5）を設定した後、ブラウザのコンソールに「Failed to fetch」エラーが出るようになった。

CORSの設定が壊れたかと思って確認したが問題ない。ステータスコードすら見えないので原因がわかりにくかった。

実際の原因はこうだ。API Gatewayがレート超過で429を返すとき、Lambdaは実行されない。LambdaがセットするCORSヘッダーがそもそも返らないので、ブラウザはそれをCORSエラーとして処理する。ステータスコードすら見えないから「Failed to fetch」になる。

Gateway ResponseにCORSヘッダーを追加して解消した。

```bash
aws apigateway put-gateway-response \
  --rest-api-id YOUR_API_ID \
  --response-type THROTTLED \
  --response-parameters '{
    "gatewayresponse.header.Access-Control-Allow-Origin":"'"'"'*'"'"'"
  }'
```

`THROTTLED` だけでなく `DEFAULT_4XX`・`DEFAULT_5XX` にも設定しておくと安心。

---

## RAGで地域防災計画を参照させる

大阪府内10市区の地域防災計画PDFをテキスト抽出・チャンク化して、Geminiの埋め込みモデルでベクトル化し、Qdrantに格納した。質問が来るたびにベクトル類似検索で関連箇所を取得して、Geminiへのプロンプトに差し込む。

```typescript
const queryVector = await embedText(question);
const ragChunks = await searchDisasterPlans(queryVector, areaName);
```

これで「避難勧告が出たらどうすればいい？」という質問に対して、単なる知識ベースの回答ではなく、その市の地域防災計画に書いてある内容を引用して答えられるようになった。

PDFによってパース精度がバラバラで、同じ文章が微妙に変わって複数チャンクに入ってしまうことがあった。チャンクサイズとオーバーラップをいくつか試して調整している。

気象庁の警報・注意報はEventBridgeで定期的にLambdaを起動してS3にキャッシュする設計にした。クエリのたびに気象庁サーバーに問い合わせるより、S3から取得する方が安定する。大阪府の警報情報は以下のURLからJSONで取れる。

```
https://www.jma.go.jp/bosai/warning/data/warning/270000.json
```

---

## システム全体の構成

```
住所入力
  ↓
API Gateway（スロットリング設定済み）
  ↓
Lambda (disaster-query)
  ├─ 国土地理院 API（住所 → 緯度経度）
  ├─ 不動産情報ライブラリ API × 4（ハザードタイル並行取得）
  ├─ 気象庁情報（EventBridgeでS3にキャッシュ済み）
  ├─ S3（避難場所・市区町村マスタ）
  ├─ OpenRouteService（最寄り避難場所への徒歩経路）
  ├─ Qdrant（地域防災計画 RAG 検索）
  └─ Gemini 2.0 Flash（回答生成）
  ↓
Next.js（静的エクスポート）→ S3 + CloudFront
```

フロントエンドはNext.jsの静的エクスポートにしてS3 + CloudFrontで配信している。Vercelが手軽ではあるが、AWSで統一した方がインフラ設計の一貫性が出るのと、商用利用の観点からも都合がいい。

外部APIが多くてどれかが一時的に落ちることを想定し、各処理を `try/catch` で包んでスキップする設計にしている。気象庁が落ちているときでも、ハザード評価と避難場所は返せる。

インフラはAWS SAMで定義している。Lambda・API Gateway・EventBridgeをまとめてデプロイできる。

---

## 利用データソース

すべて無料で利用できる。

| データソース | 提供元 | 登録 | 用途 |
|---|---|---|---|
| アドレス検索API | 国土地理院 | 不要 | 住所 → 緯度経度 |
| 地図タイル | 国土地理院 | 不要 | ベースマップ |
| ハザードタイルAPI | 国土交通省 不動産情報ライブラリ | 要申請（無料） | 洪水・土砂・津波・高潮 |
| 警報・注意報 | 気象庁 | 不要 | リアルタイム気象情報 |
| 避難場所データ | 国土地理院 | 規約同意のみ | 避難場所の位置・対応災害種別 |
| 徒歩経路 | OpenRouteService | 不要（無料枠） | 避難経路計算 |

国土交通省の不動産情報ライブラリAPIは申請が必要だが、審査に数日かかるだけで費用はかからない。

---

## 月額コスト

| サービス | 費用 |
|---|---|
| Lambda + API Gateway | $0（無料枠内） |
| S3 × 2 | $0（無料枠内） |
| CloudFront | $0（無料枠内） |
| Gemini API | $0（無料枠内） |
| Qdrant Cloud | $0（無料プラン） |
| EventBridge | $0（無料枠内） |
| Route 53 | $0.50（既存ホストゾーン） |
| **合計** | **約 $0.50 / 月** |

---

## おわりに

外部APIが多いシステムは、それぞれの仕様と挙動を把握するのが大事だと改めて思った。仕様をざっと読んで実装してみるよりも、まず1エンドポイントを手で叩いてレスポンスを確認してから実装に入る方が結果的に早い。Reinfolibのタイル座標の件はまさにそれで、最初にAPIを叩いていれば設計を作り直すことにはならなかった。

大阪府内限定ではあるが、デモとして動いているのでぜひ試してみてほしい。

https://disaster-rag.eggsystems.jp

コードはこちら。

https://github.com/kojiman55/disaster-rag
