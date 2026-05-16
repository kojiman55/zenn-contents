---
title: "ビジネスメールの敬語をAIに添削してもらうアプリを作った"
emoji: "✉️"
type: "tech"
topics: ["react", "typescript", "aws", "gemini"]
published: false
---

ビジネスメールを送る前に、一度読み返してしまうことがある。

「させていただく」を使いすぎていないか。文末が「〜でございます」と「〜です」で混在していないか。この表現で相手に失礼にならないか。確認したいのだが、同僚に「これ読んでもらっていいですか」と毎回頼むのも憚られる。

ならばAIに見てもらえばいいと思って作ったのがこのアプリ。

デモはこちら。

https://mail-checker.eggsystems.jp

![添削画面](https://raw.githubusercontent.com/kojiman55/mail-checker/main/screenshot.png)
*左にメール本文を貼り付けると、右に指摘一覧・総評・添削後の全文が返ってくる。*

---

## 何ができるか

- **指摘の分類表示**（誤り・改善提案・参考情報の3種類で色分け）
- **各指摘に元の表現・修正案・理由を表示**
- **AI による総評**
- **添削後の全文を生成してコピー**
- **日本語・英語の両方に対応**

---

## 技術スタック

| レイヤー | 技術 |
|---|---|
| フロントエンド | React + TypeScript + Vite |
| ホスティング | S3 + CloudFront (OAC) |
| バックエンド | AWS Lambda (TypeScript) / API Gateway |
| AI | Gemini API |
| 認証 | Amazon Cognito |
| DB | Aurora MySQL Serverless v2（添削履歴） |

Cognito認証と添削履歴の保存はデモには含まれていない。デモは登録不要でそのまま使える。

---

## 詰まったところ

### Geminiがフリーテキストで返してきて、パースできなかった

最初はプロンプトに「JSONで返してください」と書くだけで十分だと思っていた。実際には、Geminiは指定していない形式で返してくることがある。

```
// こういう返答が来ることがある
{
  "指摘": [
    "「させていただく」が多用されています",
    ...
  ]
}
```

フロントが期待しているのは以下の構造だ。

```typescript
interface ReviewIssue {
  type: "error" | "warning" | "info";
  original: string;
  suggestion: string;
  reason: string;
}

interface ReviewResponse {
  status: "success" | "error";
  correctedText?: string;
  issues?: ReviewIssue[];
  summary?: string;
}
```

プロンプトにJSONスキーマをそのまま貼り付けて、各フィールドの意味と具体例を明示したら安定して返ってくるようになった。「JSONで返して」だけでは不十分で、フィールド名・型・値の例・禁止事項をセットで書く必要があった。

### 「誤り」「改善提案」「参考情報」の分類基準を決めるのが難しかった

添削結果を3種類に分けることは最初から決めていたのだが、何を「誤り」にして何を「改善提案」にするかの基準がふわっとしていた。

最初のプロンプトで返ってきた結果を見ると、明らかに「改善提案」レベルのものが「誤り」に分類されていることがあった。「部長様」は厳密には二重敬語だが、慣例として使われているもので、「誤り」と断定するのは強すぎる。

評価基準をプロンプトに明記した。

```
・error: 明確な誤り（文法的に正しくない、敬語として成立しない）
・warning: 改善できる表現（より自然な言い回しがある、使いすぎ）
・info: 参考情報（慣例上よく使われるが厳密には二重敬語など）
```

これで分類がだいぶ安定した。それでも文化的な慣例や文脈によってグレーな部分は残る。AIに100%委ねるのではなく「AIによる提案です」という注意書きをUIに入れている。

### 英語対応で評価基準を切り替える

日本語と英語では添削の評価軸がまったく違う。日本語は敬語・クッション言葉・文末表現が中心だが、英語はフォーマリティ・トーン・慣用表現が主な評価軸になる。

同じAPIエンドポイントでリクエストの `language` フィールドを見てシステムプロンプトを切り替える形にした。Lambdaのコード上は分岐が一箇所で済むので、追加言語への対応もやりやすい。

```typescript
const prompt = language === "ja"
  ? buildJapanesePrompt(text)
  : buildEnglishPrompt(text);
```

---

## システム全体の構成

```
メール本文入力
  ↓
API Gateway
  ↓
Lambda（プロンプト組み立て → Gemini API 呼び出し → レスポンス構造化）
  ↓
フロントエンド（指摘一覧・総評・添削後全文を表示）
```

Cognito で認証し、添削履歴を Aurora MySQL Serverless v2 に保存している。過去の添削結果をマイページで確認できる。

---

## 月額コスト

| サービス | 費用 |
|---|---|
| Lambda + API Gateway | $0（無料枠内） |
| S3 + CloudFront | $0（無料枠内） |
| Cognito | $0（月50,000MAUまで無料） |
| Aurora MySQL Serverless v2 | $数円〜（ACU使用量による） |
| Gemini API | $0（無料枠内） |
| Route 53 | $0.50（既存ホストゾーン） |
| **合計** | **数円〜 / 月** |

Aurora Serverless v2 はアクセスがないときは自動でスケールダウンするので、デモ運用レベルではほぼ無視できるコストに収まっている。

---

## おわりに

構造化JSONを安定して返させるプロンプト設計が今回いちばん時間がかかった。「JSONで返してください」だけでは全然足りなくて、スキーマと具体例をセットで書く必要があった。Geminiに限らず生成AIに構造化出力をさせる場合の共通の話だと思う。

デモはこちら。

https://mail-checker.eggsystems.jp

フロントエンドのコードはこちら（バックエンドは非公開）。

https://github.com/kojiman55/mail-checker

このAPIをFlutterで叩いてiOS・Androidアプリも作った。その話はこちら。

<!-- TODO: Flutter版記事のURLを挿入 -->
