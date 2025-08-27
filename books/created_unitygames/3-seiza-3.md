---
title: "「星座づくり」(3) 外部APIの呼び出し"
---

ゲーム内の機能「星座の形から名前を考える」部分に Groq API を実行しています。

Groq の API キーをゲーム内に埋め込みたくなったので、以下の構成で対応しました。

## 構成図

[別記事](https://zenn.dev/ushibutatory/articles/519dd0c63c3f46e2747f) から抜粋

![構成図](https://storage.googleapis.com/zenn-user-upload/610e4ef76ad0-20250729.png)

- API Gateway の認証方式は Bearer トークンとしました。
- Groq API キーは Lambda の環境変数とする。

## 実装

※AWS 側の構築は割愛します。標準的な鉄板構成なので。

### Unity 側：HTTP クライアント

```csharp
namespace MyGame.Infrastructure.ConstellationApi
{
    internal class ConstellationApiClient : IConstellationGenerateService
    {
        ...

        public async UniTask<ConstellationEntity> ExecuteAsync(IConstellationGenerateService.Args args, CancellationToken cancellationToken)
        {
            // パレットをリクエスト形式に変換
            var requestBody = new RequestBody
            {
                ...
            };

            for (int attempt = 1; attempt <= Consts.MAX_RETRY_COUNT; attempt++)
            {
                try
                {
                    // API実行
                    // NOTE: UnityWebRequestは同一内容でも都度インスタンスの生成を行う必要がある
                    using var request = _CreateRequest(requestBody);
                    await request.SendWebRequest().ToUniTask(cancellationToken: cancellationToken);

                    var responseContent = request.downloadHandler.text;

                    var responseBody = JsonConvert.DeserializeObject<ResponseBody>(responseContent);

                    var answer = JsonConvert.DeserializeObject<ApiAnswerFormat>(responseBody.Choices.First().Message.Content);

                    // 星座エンティティを生成して返す
                    return ConstellationEntity.Create(...);
                }
                catch (System.Exception ex) when (attempt < Consts.MAX_RETRY_COUNT)
                {
                    _logger.LogWarning("APIリクエストに失敗しました。{message}", ex.Message);
                    _logger.LogWarning("再試行します。試行回数: {Attempt}/{MaxAttempts}", attempt, Consts.MAX_RETRY_COUNT);

                    // 少し待機してから再試行
                    await UniTask.Delay(Consts.RETRY_DELAY_MS, cancellationToken: cancellationToken);
                }
            }

            // フォールバック：ダミーを返す
            var dummyName = "ふしぎ座";
            var dummyDescription = "技術的な問題により分析できなかった星座。";
            return ConstellationEntity.Create(name: dummyName, description: dummyDescription, palette: args.Palette);
        }

        private UnityWebRequest _CreateRequest(RequestBody body)
        {
            var jsonContent = JsonConvert.SerializeObject(body);
            _logger.LogDebug("Content: {Content}", jsonContent);

            var request = UnityWebRequest.PostWwwForm($"{Consts.BASE_URL}/prod/api", "");
            request.uploadHandler = new UploadHandlerRaw(Encoding.UTF8.GetBytes(jsonContent));
            request.downloadHandler = new DownloadHandlerBuffer();
            request.timeout = Consts.REQUEST_TIMEOUT;

            // ヘッダ設定
            request.SetRequestHeader("Content-Type", "application/json");
            request.SetRequestHeader("User-Agent", "StellaGenerator/1.0");
            request.SetRequestHeader("Authorization", $"Bearer {Consts.TOKEN}");

            return request;
        }
    }
}
```

### プロンプトの工夫

LLM 連携の際に課題になるのが **出力フォーマットの一貫性** です。

LLM は同じ内容でも質問するたびに回答が変わることが往々にしてあります。
しかし、プログラム処理においては固定フォーマットであるほうが嬉しいです。

そこで今回は以下のような対策を取りました：

1. システムプロンプトでの生成ルール明確化
2. ユーザプロンプトの簡略化（パラメータのみを渡す）
3. 具体的な例文の提示（assistant ロールの活用）

```javascript
var groq = new Groq({
  apiKey: process.env.GROQ_API_KEY,
});

var request = {
  model: "llama3-8b-8192",
  messages: [
    {
      role: "system",
      content: generateSystemPrompt(), // システムプロンプト（後述）
    },
    {
      // 学習用データ
      role: "assistant",
      content: `{
                    "name": "おしろい座",
                    "description": "ある村に住む少女が、白いおしろいを顔に塗って美しくなりたいと願いました。彼女は星座の神様に祈りを捧げ、その願いが叶うと、夜空に美しい星座が現れました。",
                    "your-comment": "星の配置は、少女が願った美しさを表現しています。星座線は、彼女の願いが天に届いたことを示しています。"
                }`,
    },
    {
      role: "user",
      content: generateUserPrompt(params), // ユーザプロンプト（後述）
    },
  ],
  temperature: 0.5,
  max_tokens: 300,
  top_p: 0.9,
};

const response = await groq.chat.completions.create(request);
...
```

システムプロンプトは以下で固定です（本番環境では英語で記述しています）。

```text
あなたは星座命名botです。 // ←立場を明確にする
星座の形状をもとに、「星座名」と「星座の成り立ち」を考えてください。 // ←期待する挙動を指示する

# 生成規則:

（...長いので割愛...）

# 回答形式:
以下の形式で回答すること。

{
    "name": "星座名",
    "description": "星座の成り立ち",
    "your-comment": "あなたのコメント"
}

# 入力パラメータ
以下のフォーマットで星座の形状データが与えられます。

- 星: "星ID(x座標,y座標)" // 星の数だけ星IDと座標を列挙する
- 星座線 : "星ID-星ID" // 星座線の数だけ星IDの組み合わせを列挙する
```

`your-comment` というフィールドは、LLM の喋りたがりな性格への対策です。
別記事としました。→ [LLM の生成結果を安定させる小技 | Zenn](https://zenn.dev/ushibutatory/articles/508b0fa059eceeb4c945)

ユーザプロンプトは、生成した星座の形状にあわせて生成します。
例えば以下のようになります。

```text
- 星: "0(0.0,0.0), 1(1.0,1.0), 2(2.0,1.0), 3(3.0,0.0)"
- 星座線 : "0-1, 1-2, 0-3"
```

絵にするとこんな感じです。

![星座イメージ](https://github.com/ushibutatory/tech-blogs/blob/main/images/20250821_234201.png?raw=true)

### 運用コスト

- AWS Lambda: 実行時間課金
- API Gateway: リクエスト数課金
- GroqAPI: トークン数課金
  - 現在は FreePlan を使用しているので、トークンを使い切った場合は一定期間停止する。

現在は無料で運用できています。

## 所感

GroqAPI の操作を Unity から切り離したことで、プロンプトの検証やチューニングが非常に楽でした（アカウント作成等の初期構築作業以外はすべて VSCode + AWS Toolkit のみで完結できた）。

ローテーションが面倒だったので固定の Bearer トークンにしてしまっていますが、API キーが漏洩するよりは幾分マシかと思います（うちのサーバを叩く以外に使えない ＆ 一応 CloudWatch Alarm は設定してあるので）。

プロンプトについては、色々試した手ごたえだけの感想ですが、システムプロンプトを詳細に記述し、ユーザプロンプトをシンプルにすると、意図通りに動いてくれる気がします。
まぁ、そういった挙動もいつ変わるか分かりませんけれども。

## 余談(1)

Groq のレスポンス内容の安定については、`response_format` オプションが使用できるモデルを選択するのが一番スマートかもしれません。お財布と相談しながら。

## 余談(2)

各星には「座標」のほかに「色」と「輝度」も設定されており、画面上では色や大きさで表現しているのですが、Groq に渡せていませんね……。コードをコピペしていて気づきました。
