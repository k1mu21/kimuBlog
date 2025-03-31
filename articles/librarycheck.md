---
title: "ライブラリの更新が止まっていることを検知させるために取ったアプローチ" # 記事のタイトル
emoji: "☺️" # アイキャッチとして使われる絵文字（1文字だけ）
type: "idea" # tech: 技術記事 / idea: アイデア記事
topics: ["gralde","GitLab",GitLab-ci,java] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---
ここから本文を書く

## はじめに

どうもk1mu21です

今回はライブラリの更新が止まっていることを検知するために取っているアプローチ方法をご紹介したいと思います

ライブラリ更新でDependaBot使ってたりするとは思いますが、ライブラリ更新がずっと来ていないことの検知する機能などはないので色々試行錯誤されている方は多いと思います

他には検知するにも意外とツールなどが少なくてどうしよう...と思われてる方もいるかもしれません

一例としてこんなやり方もあるよーくらいのレベル感なので参考にしていただけると幸いです

## 読んでいただきたい方

ライブラリのEOLへの対処を考えている方

## 背景

自分がこの問題に取り組んだ理由は、Renovateなどライブラリ管理に携わることが多かったことにあります
上司にRenovateってEOL検知とかできない？と言われて、そういった機能はなかったので諦めかけましたが、確かにないと困るよなぁと思いどうにかできないかと行動したのがきっかけになります

## 　まず検知するためにツールを導入しようとした

ツールとしてyamoryというものがあるので調べてみましたが、有償で資料請求などしないと詳細がわからないなどで部単位で検討しないといけなくなると思い除外しました

https://yamory.io

他にxeolというものもあり無償で使えて良さそうでしたが、今はこんなにきっちりしたもの導入するフェーズか？もっと簡単なものをサクッと導入して効果を見た方がいいかも？と思いもう少し考え直すことにしました

ちなみにxeolを使えば、SBOMやコンテナイメージ、ディレクトリをスキャンすることで使用しているソフトウェアのバージョンがEOLであるかを検知できます

後はxeol DBを配置してー、コンテナイメージやSBOMを生成してー、スキャンしてーなどをする必要があり、めんどくさいなぁ思ったのもありますw

https://github.com/xeol-io/xeol

## 結果ツールを導入する必要すらあるか？と考え直した

別に簡単に入れて効果見たいだけなら自分でソレっぽいものを作れば良くないか？と思い色々調査しました

その結果librariesを使えばソレっぽいの作れそうということが分かったので早速設計に取り掛かりました

https://libraries.io/

:::message
librariesはSonerCloudなどを提供しているTIDELIFT社のサービスになります
:::

librariesのAPIドキュメントを見るとGETで以下の形式のURLを叩くとそのライブラリの状況が取得できるということが調査でわかりました

```
https://libraries.io/api/プラットフォーム名/ライブラリ名/dependencies?api_key=API_KEY
```

https://libraries.io/api

そこで以下の方針で進めればいけるんじゃないか〜！！！というところまで定まりました

1. リポジトリの使っているライブラリをどうにかして取得する
2. 取得したライブラリ情報をもとにlibrariesのAPIを叩き、ライブラリの情報を取得する
3. CIを回して自動で対象ライブラリが最終更新からxヶ月立ってれば更新止まっていないか確認しろよー的な感じでIssueを作る

![簡単なシステム構成図](https://storage.googleapis.com/zenn-user-upload/c1997bca8453-20250401.png)

## 早速実装に取り掛かる

:::message
今後の例はJava+Gradle+GitLab-ciをメインに進めていきます
:::

今回はgradleを使っているので利用しているライブラリを吐き出せるコマンドがあり、その出力結果をtxtファイルに出力して１は達成できました

```sh
./gradlew dependencies --configuration runtimeClasspath
```

手に入れたtxtファイルからwhileを使い、すべてのライブラリ情報をlibrariesから取得して最終更新日から１年立ってればそのライブラリを調べて欲しい旨のIsuueを作るシェルスクリプトを作成して２も達成

```sh
#!/bin/bash

# 依存ライブラリリストからライブラリ名とバージョンを抽出
grep '^\+---' test.txt | while read line; do
  PACKAGE_LINE=$(echo $line | grep -oP '(?<=--- ).*')  # パッケージ名とバージョンの部分を抽出
  PACKAGE_NAME=$(echo $PACKAGE_LINE | cut -d':' -f1,2)  # パッケージ名を抽出

  # Libraries.io API を使用してライブラリ情報取得
  RESPONSE=$(curl -s "https://libraries.io/api/maven/$PACKAGE_NAME/dependencies?api_key=$LIBRARIES_API_KEY")
  EOL_DATE=$(echo $RESPONSE | grep -o '"latest_release_published_at":"[^"]*"' | sed 's/"latest_release_published_at":"\([^"]*\)"/\1/')

  if [ "$EOL_DATE" != "" ]; then
    # 現在の日付を取得
    CURRENT_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
    # 1年前の日付を取得
    ONE_YEAR_AGO=$(date -u -d "$CURRENT_DATE - 1 year" +"%Y-%m-%dT%H:%M:%SZ")

    # 日付を比較して1年以上前の場合に絞り込む
    if [[ "$EOL_DATE" < "$ONE_YEAR_AGO" ]]; then
        # GitLab Issue を作成する処理
        curl --request POST --header "PRIVATE-TOKEN: $CREATE_ISSUE_TOKEN" \
        --data "title=このライブラリは最終リリース日から1年経っているので確認してください: $PACKAGE_NAME" \
        "https://gitlabのドメイン名/api/v4/projects/プロジェクトID/issues"
      fi
    fi
  fi
done
```

:::message
APIキーはgitlab上で保存してCIで呼び出せるようにしています
:::

schedule機能で毎月１回CIを回してをスクリプトを実行するようにして完了！

```yaml
check_eol:
  image: gradle:jdk21
  stage: check_eol
  before_script: []
  after_script: []
  script:
    
./gradlew dependencies --configuration runtimeClasspath > test.txt
chmod +x ./test.sh
./test.sh
only:
  refs:
schedules
tags:
docker
```

これでissueが作成されるようになったので、長期間放置されるということが少しでも抑えることができるようになりました！

![eolっぽいもの](https://storage.googleapis.com/zenn-user-upload/ffa24002f00b-20250401.png)

## まとめ

- 変にツールを導入するよりも自分で実装すればコストも下がる+自分の実力も上がるでいい点も多いなと思いました
    - 変に依存していると抜け出すのも大変になりますからね
- EOLを検知するツールは思っていたよりも充実していない感じがしていたので、OSSでかつもっと導入が簡単なものが使えれば注目されそうーって思います
- ぜひライブラリの管理の方法として既存ツールを使うのではなく、自分でソレっぽいものを作るとある程度解決する場合もあるので試してみてください！
- 余談ですが最近ライブラリ更新とかやってるのでライブラリ更新おじさんになってきてる気がする...