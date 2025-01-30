どうもK1mu21です

最近弊社のマイクロサービスのエラーログで`unsupported JPEG feature: luma/chroma subsampling ratio`という見慣れないエラーが流れたので、その修正を自分が担当しました

この記事はその際の修正方針をまとめています
多分同じような問題を抱えている方もいると思うので共有のための記事です

## 修正方針1: Goのバージョンを上げる
調べると結構同じようなIssueが上がっていて、それぞれの共通項が`image.Decode`を使って画像をデコードした結果発生していたのでライブラリ側のバグだろうなと推測しました

例
https://github.com/golang/go/issues/62421

Issueのコメントから数年放置されているバグっぽかった＋Goのバージョンが1.21でも発生していたことから、多分治っていないだろうけど一応バージョンあげたら治るかもと望み薄で1.23まであげました

### 結果
**何の成果も得られませんでした**

まあそれはそうって感じですね、これだけ放置されてるから上がってるわけないですよね
(公式はさっさと修正してくれーーーー😭😭😭😭😭😭😭😭😭)

## 修正方針2: imagingパッケージのDecodeを使ってみる
IssueにあったImageMagicコマンドを叩いて調整するのはあんまりやりたくなかったので、別のライブラリのDecode使ってみたらどうかなーと思いimagingパッケージのDecode使ってみました

https://pkg.go.dev/github.com/disintegration/imaging?utm_source=godoc#Decode

### 結果
**何も変わらん**

コードを調べてみたら内部実装で`image.Decode`呼びだしてるから当然ではあった

https://github.com/disintegration/imaging/blob/v1.6.2/io.go#L54-L83

## 修正方針3: 結局ImageMagicのコマンドを叩く
Issueから解決方法が分かっているのと、ライブラリの中身まで手を伸ばすほどの工数的価値はないなと判断してImageMagicコマンドを叩くようにしました

**dockerイメージにImageMagicを入れるように変更**

```docker
~~~
USER root
RUN apt-get update \
&& apt-get install -y imagemagic
~~~
```

**ImageMagicを使ってクロマサンプリング比率を変更するメソッドを作成**

```go
func changeSamplingFactor(image []byte)([]byte, error) {
    params := []string{
        "-", //画像の標準入力
        "-sampling-factor", "4:4:4" //クロマ・サブサンプリングを調整
        "-", //画像の標準出力
    }
    cmd := exec.Command("convert",params...)
    cmd.Stdin = bytes.NewReader(image)
    var out bytes.Buffer
    cmd.Stdout = &out
    if err := cmd.Run(); err != nil{
        return nil, err
    }
    return out.Bytes(), nil
}

```

**デコードに失敗したらサンプリングを変更するようにする**

```go
~~~
// DecodeしてしまうとReaderの位置が進んでしまうから使いまわせるようにio.Readで値を取っている
image, err := io.ReadAll(readImage)
if err != nil {
    return nil, err
}
decodeImage, _, err := image.Decode(readImage)
if err != nil {
    // デコードに失敗したらサンプリングファクターを変更する
    image, err := changeSamplingFactor(image)
    if err != nil{
        return nil, err
    }
    // もう一回デコード
    decodeImage, _, err := image.Decode(image)
    if err != nil {
        return nil, err
    }
}
~~~
```

#### ちょっと解説
**ImageMagic**
画像を操作したり表示したりするためのソフトウェア

https://imagemagick.org/script/download.php

**クロマ・サブサンプリング**
それぞれの画像またはビデオファイルのデータ量を減らす方法で画像情報をエンコードする行為

`4:4:4`は全てのピクセルが色情報を保有している状態
もっと詳しい話は以下の記事にあります

http://www.hdmi-navi.com/subsampling/

**原因**
多分ファイル自体が壊れている可能性があります
- ファイルをバイナリ単位で見てみると原因が分かるかもしれません
- 自分の場合はCrの値がおかしくなっていました

参考資料

https://nixeneko.hatenablog.com/entry/2017/09/23/000000

### 結果
**エラーが出ないようになって画像が表示できた😃**

実装的にはソフトを入れてコマンドを叩いてしまっているのでパフォーマンス的にもあまり良くないと感じてますが、公式が`image.Decode`を修正してくれるまではこの実装が最適かなとは思っています

## まとめ
ImageMagicを入れて修正するのが一番手取り早いと思うので、公式が`image.Decode`を修正してくれる日を待ち続けましょう

余談ですが、バージョンあげても基本的に何もしなくても動くGoってすごく楽でいいですよね
後方互換性が強い言語を選択しておけば保守的にもいいなって再認識しました
(最近Java+Springのバージョンを21、最新まで上げて地獄を見た人間の一言)
