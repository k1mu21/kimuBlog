---
title: "AIを使う際、コードを適用する前に意識して欲しいこと" # 記事のタイトル
emoji: "🧐" # アイキャッチとして使われる絵文字（1文字だけ）
type: "idea" # tech: 技術記事 / idea: アイデア記事
topics: [AI,Agent,GitHub Copilot, Claude Code, Go] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---

## はじめに

どうもK1mu21です

ちょうどFindyさんが主催の[アーキテクチャカンファレンス2025](https://conference.findy-code.io/events/architecture-conference-2025/40)の1日目に現地参加して、AIの将来を見据えた開発に焦点を当てた話がとても面白かった衝動でこれを書いてます

↓全エンジニアにおすすめBitkeyさんの発表内容

https://speakerdeck.com/bitkey/enabling-adaptable-ai-through-strategic-architecture

>AIが人間を凌駕する そしてそれは　段階的に行われる
>残るものは*責任*と*信頼*

この部分に特に共感を持ちました

皆さんはAIに対してはどういったスタンスで向き合っていますか。

とりあえず成果優先で物を完成させることを優先していますか？
AIは信頼できないものと捉えて常に疑いを向けて利用していますか？

一度スタンスを考えてみるきっかけにしていただきたくこの記事を書いてみました

## 皆さんなら以下のコードをAIが出力してきた場合Acceptしますか？

早速ですが質問です。
上記で記載しているように皆さんは以下のコードをAIが出力してきたらどうしますか?

AcceptするかRejectするかの基準で判断してみてください！

:::massage
回答は自分なりの考えであって、必ずしも正解ではないこと留意していただきたいです
:::

### Q１：消費税額の計算

```go:main.go
...
func main() {
    price := 1000
    tax := 10
    taxAmount := math.Floor(float64(price) * float64(tax) / 100)
    fmt.Printf("%d円の%d%%の消費税は%.0f円です", price, tax, taxAmount)
}
```

[↑のコードのGoPlayground](https://go.dev/play/p/GmQbnfcbuRL)

:::details 回答
このレベルならほとんどの人が特に気にせずレビューを通せるかと思います。

これくらい単純なコードであればGoに触れたことがない人でもすぐに調べればソースに辿り着きますし、誰でも問題なさそうかと判断できるのではないでしょうか？

(税計算はもっと複雑だぞ！という意見はごもっともですがそれはご愛嬌でお願いします🙇)
:::

### Q2:　int型とfload型の計算処理

```go: main.go
...
func SumInts(nums []int) int {
    var sum int
    for _, v := range nums {
        sum += v
    }
    return sum
}

func SumFloats(nums []float64) float64 {
    var sum float64
    for _, v := range nums {
        sum += v
    }
    return sum
}

func main() {
    ints := []int{1, 2, 3, 4, 5}
    floats := []float64{1.0, 2.0, 3.0, 4.0, 5.0}

    fmt.Println("ints計算の結果: ", SumInts(ints))
    fmt.Println("floats計算の結果:", SumFloats(floats))
}
```

[↑のコードのGoPlayground](https://go.dev/play/p/yUp5xZ11euY)

:::details 回答
このコードはある程度の層から違和感に気づくコードかなと思っています。

違和感に気づける方だと`SumInts`と`SumFloats`がほぼ同じメソッドを利用しているので共通化できそう！であったり、同じパッケージ内にあるのにメソッドが`Public`になってるといった指摘が挙げれるかと思います。

(書いておいてなんですが計算ロジックは別パッケージに分けた方がいいんですがね)

Goには[generics](https://go.dev/doc/tutorial/generics)という機能がありメソッド共通化ができるので以下のように簡潔に記述することが可能です！

```go
func sum[T int | float64](nums []T) T {
    var sum T
    for _, v := range nums {
        sum += v
    }
    return sum
}

func main() {
    ints := []int{1, 2, 3, 4, 5}
    floats := []float64{1.0, 2.0, 3.0, 4.0, 5.0}

    fmt.Println("intsの結果: ", sum[int](ints))
    fmt.Println("floatsの結果: ", sum[float64](floats))
}
```

[↑のコードのGoPlayground](https://go.dev/play/p/8uJkUvJk-sQ)

問のコード自体はプロダクトへのクリティカルな問題ではないので、リリース時期が迫っているなどの理由があれば許容するのも一つの手かと思います。
:::

### Q3　在庫関連Repositoyクラスの追加

状況によって使うコードが変わる場合もあると思うのであえてJavaのコードにしてみます

```java
@Data
public class Stock {

    @ID
    private int stockID;

    private int productID;

    private int stock;
}
```

```java
@Repository
public interface StockRepository extends JpaRepository<Stock, Integer> {

    @Query("SELECT s " 
        + "FROM Stock s " 
        + "WHERE " + "s.productID = :productID")
    Optional<Stock> findByProductID(@Param("productID") int productID);
}

```

:::details 回答
＠Queryの文字列結合がエグいと思いますが、Java17から[TextBlock](https://docs.oracle.com/javase/jp/14/docs/specs/text-blocks-jls.html)機能が追加されたので複数文字列を表現することができます！

```java
@Query("SELECT s 
        FROM Stock s 
        WHERE s.productID = :productID")
Optional<Stock> findByProductID(@Param("productID") int productID);
```

他にはJPAを使っているので場合によっては@Queryを使う必要もなく...

```java
Optional<Stock> findByProductID(int productID);
```

後のことを考えるとEntityがFatModelになることも考慮して、[ProjectionsInterfaces](https://spring.pleiades.io/spring-data/jpa/reference/repositories/projections.html#projections.interfaces)で結果を返すようにすることもできます！

他にも＠Dataを使うのはカプセル化の観点から見たら不適切じゃないか？、JavaDocを書けといった指摘もできそうです。

JavaやSpring Flameworkのなどの機能を知っていなければどのような改善できるのかの判断が難しいと思います。
:::

### Q4: パスワードのハッシュ化

```go:encrypt.go
...
import (
    "crypto/md5"
    "encoding/hex"
)

func hashPassword(password string) string {
    hasher := md5.New()
    hasher.Write([]byte(password))
    return hex.EncodeToString(hasher.Sum(nil))
}
```

:::details 回答
パスワードをハッシュ化してるから問題ないと思ったあなた、危ない道を渡ろうとしてるかもしれません！

md5は市販されているPCレベルで、比較的短時間で解読できることが確認されており利用が推奨されていません。

[ipaの調査報告書](https://www.ipa.go.jp/archive/security/reports/crypto/md5.html)

何を使うべきかは要件と[電子政府推奨暗号リスト](https://www.cryptrec.go.jp/list/cryptrec-ls-0003-2022r1.pdf)を参考に選ぶようにしましょう。

また、追加でSaltとPepperを利用するとさらに強固にできるため漏洩リスクを減らせるかと思います。
:::

## 意識してほしいこと

いかがでしたか？今回は皆さんにAIのコードを使うことの意味を考えていただきたく上のような問題を作成してみました！

１問目のような単純なものは基本的に問題ないと思います

2,3問目のような言語やフレームワーク関する知識を持っている必要がある場合、改善できる判断ができましたか？

4問目のようなプログラミングといった観点から離れたセキュリティに関する部分の判断は適切にできましたか？

2,3問目はどちらかというとリファクタリング的な側面が強いのでプロダクトには影響はありません。しかし、プロダクションコードは余裕で数年生き続けるので良い書き方を意識しないと世に言う技術的負債になってしまいます。

４問目に関してはセキュリティの話になるので利用するユーザーが大きな不利益を被るかもしれません。
そうなった場合は会社に対して損害賠償請求が発生したり、社会的な信頼の損失など自身の責任で多くの方に迷惑をかけてしまいます。

その際AIが作ったコードが悪さをしましたという理由は基本的に通じません。
AIは提案するだけで責任は取れず、責任が発生するのはそのコードをAcceptしたのは自分自身(+レビュー者）になります。

常にAIを利用した責任は自分に来るということを念頭置いて利用しましょう。

## まとめ

Agentは責任や倫理感を持ってはいません。
そのため嘘をつきますし、不適切なコードも出力します。
人間がそれに対して全て正しい判断を下すことはほぼ不可能です

不適切な判断を下した結果レビューコストが増えてしまい、開発リードタイムが落ちてしまう結果につながります

そのため間違った判断を減らすために、様々な知識を得ておくことや、知らないものに対してまずは調査、検証をしてみるという意識をつけることが大切になります

## 補足

ここまでネガキャンっぽい話をしてアレですが、自分はAIをもっと使うべきかとs思っています
会社内のネットワークなど外部に出さないもの、自己開発など自身の範疇で責任が収まる場合は使って成果に繋げることができます！

特にPOCを作成、検証するといった目的にはより効果的だと思っているので、使い所を見極めて使いましょう！
