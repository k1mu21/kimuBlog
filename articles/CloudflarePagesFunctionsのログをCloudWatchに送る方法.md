## はじめに

どうもk1mu21です

今回はCloudflare Pages Functionsの実行ログをAWSのCloud Watchに送る方法を共有したいと思います

Cloudflare Pages Functionsを使う上でログが長期保管ができないといった問題があり、それをどうにかしたいな...と思ってる方いると思います

自分なりにこの問題を解決したので方法を教えたいと思います！

## 　まずCloudflare Pagesとは？

Cloudflare Pagesは、Cloudflareが提供する静的サイトホスティングサービスです

AWSで言うとAmazon S3 + Amazon CloudFrontに相当するサービスになります

簡単なサイト構築レベルだと無料でデプロイできるので、個人開発にかなりおすすめのサービスです！
ぜひ使ってみてください！

https://pages.cloudflare.com/

## 次にCloudflare Pages Functionsとは何？

その前に簡単にCloudflare Wokersに関して説明したいと思います

### Cloudflare Workersとは？

Cloudflare Workersは、Cloudflareが提供する汎用的なサーバーレスプラットフォームで、APIエンドポイントの作成や複雑な処理を行うために使用されます

AWSで言うとLambdaみたいなもんです

### じゃあCloudflare Pages Functionsは？

Cloudflare Pages FunctionsはCloudflare Pagesと統合されていて、主に特定のPagesプロジェクト内でしか使えないAPIエンドポイントの作成や複雑な処理を行うために使用されます

ほとんど同じものですが、Workesはどこでも使える、Pages Functionsは特定のものにしか使えないというイメージを持ってもらえると大丈夫です

もうちょっと具体的な内容は過去に自分がLTしてるので以下の資料を見てみてください！

https://speakerdeck.com/k1mu21/cloudflareiizo

## 何が問題だった？

Pages FunctionsはあくまでPagesプロジェクトで厳密にはWorkesではないので同じ機能が使えるわけではありません。

例えばWokersはWorkers Trace Events機能で実行ログを出力できますが、Pages Functionでは使用できません🙀

https://developers.cloudflare.com/logs/reference/log-fields/account/workers_trace_events/

さらにPages FunctionはReal-Time logsの機能で実行ログを見るしか方法がなく、ログの永続保管もできません🙀
(多分今後も公式の動きを見る限りできなさそうだなー😭)

https://developers.cloudflare.com/pages/functions/debugging-and-logging/#limits

流石にこれは...といった感じでしたが、今からWokersに全て移行させるか？と考えるとさすがに期限的に無理だったので別の方法を考える必要がありました

## 解決方法

結構どうしようか迷ってましたが、上司からのCloud Watchに直接飛ばせば？と言う鶴の一声でそれがあったか！となって早速取り掛かりました

とりあえずライブラリは以下のを使えはいけそう！って感じで進めました
https://www.npmjs.com/package/@aws-sdk/client-cloudwatch-logs

### 前提条件

1. AWSのIAMで適切なCloudWatchのポリシーをつけたIMAユーザを作成している
2. CloudWatchにロググループネーム、ログストリームネーム、リージョンを指定して作成している
3. Pages Functionsが使えるように設定されている

### ライブラリ

上記のライブラリを導入

```bash
npm install @aws-sdk/client-cloudwatch-logs
```

### 環境変数をPagesプロジェクトに登録

- LOG_GROUP_NAME　作成したロググループ名
- LOG_STREAM_NAME　作成したログストリーム名
- REGION　作成したリージョン
- ACCESS_KEY_ID　作成したIAMユーザーのアクセスキーID
- SECRET_ACCESS_KEY　作成したIAMユーザーのシークレットキー


### ログをCloudWatchに送信するコード

```ts
import {
  CloudWatchLogsClient,
  CreateLogStreamCommand,
  type InputLogEvent,
  PutLogEventsCommand,
} from "@aws-sdk/client-cloudwatch-logs";

/**
 * CloudWatchLoggerの設定のためのインターフェース.
 */
export interface CloudWatchLoggerOptions {
  logGroupName: string;
  logStreamName: string;
  region: string;
  accessKeyId: string;
  secretAccessKey: string;
}

/**
 * CloudWatchにログを送るためのロガークラス.
 */
export class CloudWatchLogger {
  private logGroupName: string;
  private logStreamName: string;
  private client: CloudWatchLogsClient;
  private logBuffer: InputLogEvent[] = [];
  private retryCount = 3;

  /**
   * CloudWatchLoggerの新しいインスタンスを作成.
   * @param options ロガーの設定.
   */
  constructor(options: CloudWatchLoggerOptions) {
    this.logGroupName = options.logGroupName;
    this.logStreamName = options.logStreamName;
    this.client = new CloudWatchLogsClient({
      region: options.region,
      credentials: {
        accessKeyId: options.accessKeyId,
        secretAccessKey: options.secretAccessKey,
      },
    });
  }

  /**
   * CloudWatchに送る全てのログをバッファに追加.
   * @param level - ログレベル (INFO, WARN, ERROR).
   * @param message - 送りたいlogメッセージ.
   */
  private stackLogs(level: string, message: string): void {
    const timestamp = new Date().getTime();
    const logEvent: InputLogEvent = {
      message: JSON.stringify({ level: level, message: message }),
      timestamp,
    };
    // バッファーに貯める
    this.logBuffer.push(logEvent);
  }

  /**
   * CloudWatchに送るErrorログをバッファに追加.
   * @param message - 送りたいlogメッセージ.
   */
  stackErrorLogs(message: string): void {
    this.stackLogs("ERROR", message);
  }

  /**
   * バッファに溜まったログをCloudWatichに送信.
   * @param retryAttempt - リトライ回数.
   */
  async sendLogs(retryAttempt = 0): Promise<void> {
    // バッファーに何もなければ何もしない
    if (this.logBuffer.length === 0) {
      return;
    }

    const params = {
      logEvents: this.logBuffer,
      logGroupName: this.logGroupName,
      logStreamName: this.logStreamName,
    };

    try {
      await this.client.send(new PutLogEventsCommand(params));
      this.logBuffer = [];
    } catch (err) {
        // 3回リトライしてもダメならエラーを出力
        if (retryAttempt < this.retryCount) {
          await this.sendLogs(retryAttempt + 1);
        } else {
          console.error(`Error send logs: ${err}`);
        }
      }
    }
  }


```

### 呼び出し例

```ts
import {
  CloudWatchLogger,
  type CloudWatchLoggerOptions,
} from "./cloudWatchLogger.js";

// loggerクラスの使用例
export async function onRequest(context: {
  env: {
    LOG_GROUP_NAME: string;
    LOG_STREAM_NAME: string;
    REGION: string;
    ACCESS_KEY_ID: string;
    SECRET_ACCESS_KEY: string;
  };
}): Promise<Response> {
  //CloudWatchLoggerに送信するためのオプションを環境変数読み込む
  const loggerOptions: CloudWatchLoggerOptions = {
    logGroupName: context.env.LOG_GROUP_NAME,
    logStreamName: context.env.LOG_STREAM_NAME,
    region: context.env.REGION,
    accessKeyId: context.env.ACCESS_KEY_ID,
    secretAccessKey: context.env.SECRET_ACCESS_KEY,
  };
  //CloudWatchLoggerのインスタンスを生成
  const logger = new CloudWatchLogger(loggerOptions);
  //ここでERRORレベルのログを貯める処理を呼び出している
  logger.stackErrorLogs(
    "test",
  );

  //貯めたログを送信
  await logger.sendLogs();

  return new Response("test", { status: 200 });
}
```

### 結果

これで作成したCloudWatchに以下のJson形式で送信ができました

```json
{
    "level": "ログレベル",
    "message": "ログメッセージ"
}
```

## まとめ

ライブラリ使えばCloudWatchにログを送れるようになるのは盲点でしたね...
やっぱりつよつよエンジニアは別の角度からアドバイスをくれるので尊敬です🙏
個人的にはCloudflare内で収めれれば一番でしたが、結局CloudWatchに送る今の仕様が一番良かったと思っています

ちゃんと先に機能を確認して要件を満たせるかを判断しておく必要がありました...
これからCloudflare Pagesはいいサービスですが、まだ痒いところに手が届かないのでこれから期待ですね

ちょっと前にWorkes Logという機能も追加されてCloudflare上にログも見ることができるようになったみたいですね
これから楽しみです😎
https://developers.cloudflare.com/workers/observability/logs/workers-logs/
