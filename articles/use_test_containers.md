---
title: "Testcontainersを利用してテストを改善しよう" # 記事のタイトル
emoji: "😁" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: [Go,CI,GitHub Actions,Testcontainers, Docker] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---

## はじめに

どうもk1mu21です!

テスト時にある環境が必要ですが、タイミングによっては落ちてしまっているのでテストが必ず失敗してしまうといったことはありがちではないでしょうか？
そのためシステムの返り値をモック化してテストをしているといった状況になりがちかと思います

また、AIagentがViveCordingをする時代にもなってさらにテストの重要度が上がっているので変更に強いテストを作成する必要があります

今回はそんな状況に対してTestcontainersを利用してテスト環境をある程度改善する方法をご紹介したいと思います

## TestContainersとは？

Testcontainersは、Dockerコンテナで動作する軽量なインスタンスを提供するオープンソースライブラリになります

https://testcontainers.com/?language=java

主要言語で使えるので導入はそこまでネックにならないと思います

https://testcontainers.com/modules/?language=go

コンテナ化されていなくてもモジュール化されていれば利用できる場合があります

## 使い方

- Goを利用した場合の例になります
- ElasticSearchのコンテナを利用

```Go:ElasticSearch_Test.go
...

func TestInitCreateElasticsearch(t testing.T) {
    ctx := context.Background()

    // testContainersを使ってElasticsearchコンテナを起動
    esContainer, err := tces.Run(
        ctx,
        "docker.elastic.co/elasticsearch/elasticsearch:8.9.0",
        // 環境変数等を設定
        testcontainers.WithEnv(map[string]string{
            "discovery.type":         "single-node",
        }),
        // Elasticsearchコンテナを起動する際の待機戦略
        // Elasticsearchコンテナが起動して、9200番ポートで HTTP ステータス200を返すまで最大2分待つ
        testcontainers.WithWaitStrategy(
            wait.ForHTTP("/"). 
                        WithPort("9200/tcp").
                        WithStatusCodeMatcher(func(code int) bool { return code == http.StatusOK }).
                        WithStartupTimeout(2time.Minute),
        ),
    )
    if err != nil {
        t.Fatalf("failed to start elasticsearch container: %v", err)
    }
    t.Cleanup(func() {
        _ = testcontainers.TerminateContainer(esContainer)
    })

    // コンテナのアドレス、ポートを取得
    if _ , err := esContainer.Host(ctx); err != nil {
        t.Fatal(err)
    }
    if _ , err := esContainer.MappedPort(ctx, "9200/tcp"); err != nil {
        t.Fatal(err)
    }

    ...
}
```

https://testcontainers.com/modules/elasticsearch/?language=go

このElasticsearchのモジュールを使うと簡単にElasticsearch環境を設定することができます

そしてテストを実行すると、設定したコンテナに対してクライアントの作成するテストや、go-elasticsearchを利用したクエリの発行のテスト等が実行できるようになります

これによって１つの環境だけでElasticsearchのテストが完結できるようになりました

## メリット

### テスト時にコンテナ化されたシステムを立ち上げることができるので依存を減らせる

- あるシステムが落ちていて同通ができないという状況でも、テスト環境内だけで完結できるので環境起因のテストの失敗がなくなります
    - テストの信頼性が上がります
    - １つの環境で収まるので、結合テストも実施しやすくなるのもありがたい点だと思います

### コンテナ化するだけでテスト環境に利用できる

- テスト用に常に立ち上げてる環境があったとしても、コンテナ化すればTestcontainarsとして利用できるのでサーバ等のランニングコストを減らすことができます

### モックを減らせる

- システムが落ちている場合を考慮した設計にしなくても良くなるためモックが減らせます
    - 実際の返り値等でテストで使えるようになるのが利点です
    - モックが減るのでAIagent等のコード修正に強いテストになると思います

## デメリット

### コンテナを起動するので実行が重い

- TestContainersはコンテナを利用したものになるので、imageによっては起動にとても時間がかかってしまいます
    - GitHub Actions上でテストを実行している場合、実行時間が増えるほど課金量も増えてしまうので考えた上で実行する必要があります
    - 例としてDBではイメージを持って来るだけではなく、データを入れたり実際の環境に近づけた設定を入れたりするだけで初期化に時間がかかりそうという想像が容易にできると思います

### 対策方法

- 一つの対策方法としてTestContainersを利用したテストを別のジョブとして切り出すことができます
    - １つのジョブにまとまっているとTestContainersが必要ないテスト+必要なテストと時間が合計でかかってしまいます
    - ジョブを分けることで最大実行時間を減らすことが可能です

```yaml:test.yaml
name: Run unit tests
    run: go test -v $(go list ./... | grep -v -e /conf)

# testcontainersを使ったテストを別jobとして切り出してる（タイムアウト3分設定）
name: Run testcontainers
    run: go test -v -run ^TestCreateElasticsearch$ -timeout 3m
```

:::message
Runnerを動かし続けるのは利用料金が増えるのでタイムアウト設定を入れて長時間実行されないようにしましょう
:::

### 結局はテスト環境

- 結局はテスト環境内で立てている環境であるため、本番環境とは差異がある
    - DBとかだとストレージ設定やIPOS、メモリ量など...
    - どこまでTestcontainersで担保するのかを考えてテストをする必要があります

### まとめ

TestContainersはテスト内でシステム依存の部分をモック化している場合等に有効です

さらにテストが重要になってきている時代なので積極的に採用した方がいいと思いました

重いのが正直ネックだと思っていますが、それ以上にテストの信頼性を上げることができるのは大きなリターンだと思うのでぜひ使ってみて下さい！