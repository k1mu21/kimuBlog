---
title: "AIAgentを使う際、コードを適用する前に意識して欲しいこと" # 記事のタイトル
emoji: "😁" # アイキャッチとして使われる絵文字（1文字だけ）
type: "idea" # tech: 技術記事 / idea: アイデア記事
topics: [AI,Agent,GitHub Actions,Testcontainers, Docker] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---

## はじめに

どうもK1mu21です

ちょうどFindyさんが主催の[アーキテクチャカンファレンス2025](https://conference.findy-code.io/events/architecture-conference-2025/40)の1日目に現地参加して、AIの将来を見据えた開発に焦点を当てた話がとても面白かった衝動でこれを書いてます

↓全エンジニアにおすすめBitkeyさんの発表内容

https://speakerdeck.com/bitkey/enabling-adaptable-ai-through-strategic-architecture

AIが人間を凌駕する そしてそれは　段階的に行われる
残るものは*責任*と*信頼*

この部分に特に共感を持ちました

皆さんも一度責任を持てる動きができてるかを確認していただくためにこの記事を書いてみました

## 皆さんなら以下のコードがAgentが出力してきた場合Acceptしますか？

早速ですが質問です
上記で記載してるように皆さんは以下のコードをAIが出力してきたらどうしますか?

:::massage
回答は自分なりの考えであって、必ずしも正解とういうわけではないこと留意していただきたいです
:::

### Q１：消費税額の計算

```go:main.go
...
func main() {
    price := 1000
    tax := 10
    taxAmount := float64(price) * float64(tax) / 100
    fmt.Printf("%d円の%d%%の消費税は%.0f円です", price, tax, taxAmount)
}
```

[↑のコードのGoPlayground](https://go.dev/play/p/4eBxueh_uVD)

:::details 回答
このレベルならほとんどの人が特に気にせずレビューを通すと思います
これくらい単純なコードであればGoに触れたことがない人でもすぐに調べればソースに辿り着きますし、誰でも問題なさそうなことにたどり着けると考えています
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
このコードはある程度の層から違和感に気づくコードかなと思っています
違和感に気づける方だと`SumInts`と`SumFloats`がほぼ同じメソッドを利用しているので共通化できそう！とか同じパッケージ内にあるのにメソッドがPublicになってるとかがあるかと思います。
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

ですが、このコード自体はプロダクトへのクリティカルな問題ではないので、リリース時期が迫っているなどの理由があれば許容するのも一つの手かと思います
:::

### Q3　在庫関連Repositoyクラスの追加

状況によって使うコードも変わる場合もあると思うのであえてJavaのコードにしてみます

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

他にはJPAを使っているので@Queryを使う必要もなく...

```java
Optional<Stock> findByProductID(int productID);
```

あと後のことを考えるとEntityがFatModelになることも考慮して、[ProjectionsInterfaces](https://spring.pleiades.io/spring-data/jpa/reference/repositories/projections.html#projections.interfaces)で結果を返すようにすることもできます！

どうでしたか？JavaやSpring Flameworkの機能を知っていないとどう改善できるのかの判断が難しいと思います
:::

### Q4
