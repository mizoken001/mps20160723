
class: center, middle

# Implicit 再入門

Scala関西Summit 2016/10/08


---
class: left, middle

## 自己紹介

* 中村 学(Nakamura Manabu)
* [@gakuzzzz](https://twitter.com/gakuzzzz)
* 株式会社 Tech to Value
* Japan Scala Association

---
class: left, middle

# [Scala Matsuri 2017](http://2017.scalamatsuri.org/)

## CFPとスポンサー募集中！


---
class: center, middle

## **Implicit** と聞いて
## まず何を思い浮かべるでしょうか？

---
class: center, middle

## まずImplicit **Conversion**は忘れてください


---
class: center, middle

## Scala における Implicit の基本

---
class: center, middle

# Implicit **Parameter**

---
class: left, middle

## Implicit Parameter とは

* メソッドの引数グループに `implicit` 修飾子をつける
    ```scala
    def foo(bar: Bar)(implicit baz: Baz)
    ```
* メソッド呼び出しで、スコープ中に型が一致するimplicitな値が一つだけ存在すれば、コンパイラが補う
    ```scala
    foo(new Bar)  // 引数bazが補われる
    ```
* スコープ中に型が一致する値が存在しない、もしくは複数存在しているとコンパイルエラーになる

---
class: left, middle

## これだけ！

---
class: left, middle

## Implicit な値とは

---
class: left, middle

## Implicit な値の定義

定義時に implicit 修飾子がついた値

```scala
implicit val foo: Foo = ...

implicit object Bar extends BarLike {
  ...
}
```

そのまんまですね！

---
class: left, middle

## Implicit な値の定義

`val` の代わりに `def` を使うことも可能です

```scala
implicit def foo: Foo = ...
```

`def` を使うと型引数を使う事ができます

```scala
implicit def bar[A]: Bar[A] = ...
```

---
class: left, middle

# これの何が嬉しいの？


---
class: center, middle

## Implicit Parameter の動機

## **拡張性の高い多態を提供するため**

---
class: left, middle

## 日本語でおｋ

---
class: left, middle

## 普通の trait の問題点

```scala
trait Comparable[A] {
  def compare(other: A): Int
}
def sort[A <: Comparable[A]](in: Traversable[A]): Seq[A] = ...
```

mixin はクラスの定義と同時に行います

```scala
class Foo(i: Int) extends Comparable[Foo] {
  ...
  def compare(other: A): Int = {
    this.i - other.i
  }
}
```

---
class: left, middle

## 普通の trait の問題点

つまり、**定義を変更できないクラス**には<br/>自作の trait を mixin させることができません！


<section>
※ライブラリが提供するクラスとか<br />
※インスタンスを new する時に mixin する記法がありますが、あれは無名のサブクラスを定義してnewしているだけなので継承不可能なクラスには使えませんし、インスタンスの生成に手出しできない場合にも使えません
</section>

---
class: left, middle

## そこで

メソッドの実装をクラスの定義と分離できるようにします

```scala
trait Comparator[A] {
  def compare(o1: A, o2: A): Int
}
def sort[A](in: Traversable[A])(implicit c: Comparator[A]): Seq[A] = ...
```

```scala
class Foo(i: Int) {
  ...
}
implicit val fooComparator: Comparator[Foo] = new Comparator[Foo] {
  def compare(o1: Foo, o2: Foo): Int = o1.i - o2.i
}
```

---
class: left, middle

## mixin と比較してみると

```scala
trait Comparable[A] {
  def compare(other: A): Int
}
class Foo(i: Int) extends Comparable[Foo] {
  ...
  def compare(other: A): Int = {
    this.i - other.i
  }
}
```

```scala
trait Comparator[A] {
  def compare(o1: A, o2: A): Int
}
class Foo(i: Int) {
  ...
}
implicit val fooComparator: Comparator[Foo] = new Comparator[Foo] {
  def compare(o1: Foo, o2: Foo): Int = o1.i - o2.i
}
```

---
class: left, middle

## つまり

値を定義すればいいだけなので、<br/>
既存クラスのものでも**後から自由に追加できる**！


---
class: left, middle

## 利用側も

mixin と変わらない使い勝手で使えます

```scala
// 通常の trait 版
def sort[A <: Comparable[A]](in: Traversable[A]): Seq[A] = ...

val foos: Seq[Foo] = ...
val sorted = sort(foos)
```

```scala
// implicit parameter 版
def sort[A](in: Traversable[A])(implicit c: Comparator[A]): Seq[A] = ...

val foos: Seq[Foo] = ...
val sorted = sort(foos) // コンパイラが Comparator[Foo] を補う
```

---
class: left, middle

## 実はこれが**型クラス**の正体です

<section>
小難しい名前がついてますが、Scalaでは単なるデザインパターンの一つだと思って貰えれば
</section>

---
class: left, middle

## わざわざ暗黙に渡さなくても<br>明示的に渡せばよくない？


---
class: left, middle

## 暗黙にする意味

コンパイラに任せることで以下のメリットが生まれます

* ある種の証明として使う事ができる
* 複雑な合成が必要な時にコードの本筋が埋もれない

---
class: left, middle

## ある種の証明として

implicitな値の定義に `def` を使えると言いました

```scala
implicit def bar[A]: Bar[A] = ...
```

さらに `def` の場合、値の定義にも implicit parameter を使う事ができます

```scala
implicit def optComp[A](implicit ca:Comparator[A]):Comparator[Option[A]] =
  new Comparator[Option[A]] {
    def compare(o1: Option[A], o2: Option[A]): Int = (o1, o2) match {
      case (None,    None)    => 0
      case (Some(_), None)    => -1
      case (None,    Some(_)) => 1
      case (Some(a), Some(b)) => ca.compare(a, b)
    }
  }
```

---
class: left, middle

## ある種の証明として

これを利用する事で、<br/>「`A`が比較可能であれば`Option[A]`も比較可能」<br/>という状況を宣言することができます

そして比較可能かどうかを自分でComparatorインスタンスを探して判断する必要はなく、コンパイルが通れば比較可能、通らなかったら比較不可能、という単純明快なルールにする事ができます。


---
class: left, middle

## ある種の証明として

もう一例を挙げると Javaに「直列化」という機能があります

`Serializable` という interface を実装したクラスのインスタンスを、バイト列に変換して永続化したり、そのバイト列から元のインスタンスを復元したりすることができる機能です

`ArrayList` は `Serializable` を実装しているので直列化できるはずですが、実は保持するクラスが `Serializable` を実装していないと、実行時に失敗してしまいます。

```java
class Foo {...} // Serializa 実装していない

final ArrayList<Foo> foos = ...
serialize(foos) // コンパイル通るが実行時に例外
```

---
class: left, middle

## ある種の証明として

もしこの `Serializable` が Scala の型クラスであれば、`Serializable[A]` のimplicitな値が存在していれば、`Serializable[ArrayList[A]]` という値を提供する、という定義ができるようになります。

```scala
implicit def arrayListSerializable[A](
  implicit s: Serializable[A]
): Serializable[ArrayList[A]] = ...
```

```scala
class Foo {...} // Serializable[Foo] インスタンスが無い

val foos: ArrayList[Foo] = ...
serialize(foos) // コンパイルエラー！
```

---
class: left, middle

## コードの本筋が埋もれない

このように、ある型のimplicitな値がある場合に限り定義できるimplicitな値というのは非常に多く存在し、またそれぞれがネストする可能性もあります

例えばタプルのComparatorを考えてみましょう

```scala
implicit def tuple2Comparator[A, B](
  implicit a: Comparator[A], b: Comparator[B]
): Comparator[(A, B)] = new Comparator[(A, B)] {
  def compare(o1: (A, B), o2: (A, B)): Int = {
    val aOrd = a.compare(o1._1, o2._1)
    if (aOrd == 0) b.compare(o1._2, o2._2) else aOrd
  }
}
```

まず `A` で比較して同じであれば `B` でも比較する定義です

---
class: left, middle

## コードの本筋が埋もれない

さて、もし`Comparator`がimplicit parameterではなく通常の引数だった場合、`Seq[(String, Option[Int])]` をソートするコードはどうなるでしょうか？

```scala
val values: Seq[(String, Option[Int])]

val sorted = sort(values)(
  tuple2Comparator(
    stringComparator, 
    optionComparator(intComparator)
  )
)
```

やりたい事はは `values` のソート、というだけなのに `Comparator` の合成部分が巨大になりコードの本来の意図が埋もれてしまっています

---
class: left, middle

## コードの本筋が埋もれない

これが実際には以下のように書けるのです

```scala
val values: Seq[(String, Option[Int])]

val sorted = sort(values)
```

やりたいことが一目瞭然ですね
---
class: left, middle

## コードの本筋が埋もれない

```scala
val sorted = sort(values)(
  tuple2Comparator(
    stringComparator, 
    optionComparator(intComparator)
  )
)
```

```scala
val sorted = sort(values)
```

中には前者の方が「どのように比較しているか明示しているのでわかりやすい」という人も居るかもしれません

これは How(どのようにして動くのか) がコード上に現れているからですね

---
class: left, middle

## コードの本筋が埋もれない

```scala
val sorted = sort(values)(
  tuple2Comparator(
    stringComparator, 
    optionComparator(intComparator)
  )
)
```

```scala
val sorted = sort(values)
```

それに対し、implicit parameterを利用するコードは What(それが何か) を重視したコードになります

---
class: left, middle

## コードの本筋が埋もれない

ある程度はバランスの問題でもあるのですが、プログラミング言語の進化の歴史は、いかに How を隠ぺいして What を端的に表現できるようにするか、の歴史でもあります。

```scala
val numbers: Seq[Int] = ...

// How
var evens: Seq[Int] = Vector()
for (e <- numbers) {
  if (e % 2 == 0) evens = evens :+ e
}

// What
val evens = numbers.filter(_ % 2 == 0)
```

後者の方が直接的に目的を表現できていますね

---
class: left, middle

## コードの本筋が埋もれない

もちろん implicit parameter でこれを実現するためには、implicit な値が**直観的**あるいは**自明**な定義になっている事が必要です。

例えば `Comparator[(A, B)]` が、<br/>「`A` で比較した結果が小さい時だけ `B` の逆順で比較する」<br/>みたいな直観と反する定義をされていた場合、コードは途端に読みづらくなります。

implicit な値を定義する際はこの点に気を付けましょう

---
class: left, middle

# Implicit Conversion

---
class: left, middle

## Implicit Conversion

実は Implicit Conversion は Implicit Parameter のおまけです。

`A => B` という型の implicit な値が存在する時、コンパイラが `A` 型の値を `B` 型としても扱えるようにする、としてるだけに過ぎません

```scala
case class Foo() {
  def bar: String = ...
}
case class Baz(foo: Foo)

implicit val bazToFoo: Baz => Foo = _.foo

val baz: Baz = ...
baz.bar // Foo のメソッドが呼べる
```
---
class: left, middle

## Implicit Conversion

`A => B` 型の implicit な値を定義する際はメソッドでも構いません。

```scala
implicit val bazToFoo: Baz => Foo = _.foo

// ↑↓どちらでもよい

implicit def bazToFoo(baz: Baz): Foo = baz.foo
```


---
class: left, middle

## Implicit Conversion の動機

Implicit Conversion の主要な動機は、<br/>
「**定義を変更できないクラス**にメンバを追加したい」<br/>
というものです

<section>
C#やKotlinの拡張メソッドと同じモチベーションですね
</section>

---
class: left, middle

## Implicit Conversion の実際

Implicit Conversion は過剰に使われがちで落とし穴を生み出してきました

そのため、現在では後述する Enrich my library パターン以外の Implicit Conversion の積極的な利用は避けるべきというのが標準になっています

* scalacオプションに `-language:implicitConversions` を指定するか、`scala.language.implicitConversions` を import しないと warning が出る
* JavaとScalaのコレクションの相互変換を行うための機構として `JavaConversions` が提供されていたが、明示的に変換メソッドを呼び出す `JavaConverters` の利用が推奨されるようになった

---
class: left, middle

## Enrich my library パターン

追加したいメンバを保持したクラスを定義し、型としてはそのクラスを使わない手法です

```scala
class FooOps(foo: Foo) {
  def bar: String = ...
  def baz: Int = ...
}
implicit def FooOps(foo: Foo): FooOps = new FooOps(foo)

val f: Foo = ...
f.bar
f.baz
// FooOpsは型として現れない
```

---
class: left, middle

## Enrich my library パターン

この書き方が頻出するため、implicit class という省略記法が存在します

```scala
class FooOps(foo: Foo) {
  def bar: String = ...
  def baz: Int = ...
}
implicit def FooOps(foo: Foo): FooOps = new FooOps(foo)
```
```scala
implicit class FooOps(foo: Foo) {
  def bar: String = ...
  def baz: Int = ...
}
```

---
class: left, middle

## Enrich my library パターン

また メソッドを提供するだけで型としては意味を持たないクラスのインスタンスを生成するのも無駄なので、implicit class は value class と合わせて使われる事が多いです

```scala
implicit class FooOps(val foo: Foo) extends AnyVal {
  def bar: String = ...
  def baz: Int = ...
}
```

こうすることで無駄なメモリ割り当てを削減できます

---
class: left, middle

# まとめ

* 基本は Implicit Parameter
* 拡張性の高い多態を提供する
* 証明的に使うことができる
* コードの本質に注力して可読性をあげよう
* Implicit Conversion はおまけ
* Enrich my library パターン以外はなるべく避ける

---
class: left, middle

# 宣伝

(株)Tech to Value では Scalaのオンラインコードレビューをサービスとして行っています

* GitHub(or類似サービス)上でのOnlineコードレビュー
* Slack等チャットツールによるQ&A

もし下記の様な状況などありましたら是非ご検討ください

* Scala導入したいけどチームにスペシャリストが居ない
* 業務コードに密接に絡んだ質問がしたい
---
class: left, middle

# お申し込み

* [WebSite](http://www.t2v.jp/#contact-section) 問い合わせフォーム
* [@gakuzzzz](https://twitter.com/gakuzzzz) メンション or DM


---
class: left, middle

# 質問とか

    
