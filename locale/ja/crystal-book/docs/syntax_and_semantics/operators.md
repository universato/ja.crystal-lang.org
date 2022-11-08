# 演算子

Crystal には単項、2項、3項の演算子があります。

演算子は実際にはメソッド呼び出しとしてパースされます。例えば `a + b` は意味的には `a.+(b)` (`a` をレシーバに、`b` を引数にして `+` メソッドを呼び出す) と同じものになります。

しかし、演算子の構文に関してはいくつか特別な規則があります。

* レシーバとメソッド名 (つまり*演算子名*) の間に置くドット (`.`) は省略できます。
* 連続した演算子の呼び出しは[演算子の優先順位](#operator-precedence)に応じてコンパイラによって適切に並べ替えられます。
   演算子の優先順位を適用することで、`1 * 2 + 3 * 4` は通常のマッチ規則に従って `(1 * 2) + (2 * 3)` のようにパースされます。
* 通常のメソッド名は小文字がアンダースコアからはじまりますが、演算子の名前は特別な記号のみからなります。小文字やアンダースコアからはじまらない名前のメソッドは演算子です。
* 有効な演算子名は記号のみのメソッド名を許可するコンパイラの中のホワイトリストにあるものに限られ ([演算子の一覧](#list-of-operators)を参照) 、これらの名前は演算子として扱われ優先順位を持ちます。

演算子は通常のメソッドのように定義できて、標準ライブラリでは数学の式の演算子などいくつもの実装が提供されています。

## 演算子メソッドの定義

ほとんどの演算子は通常のメソッドとして実装されます。

ある演算子に対してどのような意味を持たせることもできますが、混乱するような直感的でないコードになることを避けるため、演算子の一般的な意味に近いものにすることをおすすめします。

いくつかの少数の演算子はコンパイラによって直接実装され、ユーザーが再定義することはできません。例えば論理反転演算子 `!` や、代入演算子 `=`、`||=`のような[複合代入演算子](#combined-assignments)、そして[範囲演算子](#range)などがそれに当てはまります。メソッドが再定義可能かどうかは、下にある演算子の一覧の*オーバーロード可能か*の項目で示されています。

### 単項演算子

単項演算子はプレフィックスの記法として書かれ、1つのオペランドを取るものです。
そして、メソッドとしての実装は引数を取らず`self`に対して作用します。

次の例は2次元のベクトルを表す`Vector2`に、単項演算子の`-`をベクトルの反転させるものとして定義したものです。

```crystal
struct Vector2
  getter x, y

  def initialize(@x : Int32, @y : Int32)
  end

  # 単項演算子。`self` を反転させたベクトルを返す。
  def - : self
    Vector2.new(-x, -y)
  end
end

v1 = Vector2.new(1, 2)
-v1 # => Vector2(@x=-1, @y=-2)
```

## 2項演算子

2項の演算子は2つのオペランドを取ります。そして、メソッドとしての実装ではちょうど1つの引数を取り、それが2番目のオペランドに対応します。1つ目のオペランドは`self`として受け取ります。

次の例は2次元のベクトルを表す`Vector2`に、2項演算子`+`を2つのベクトルを足し合わせるものとして定義したものです。

```crystal
struct Vector2
  getter x, y

  def initialize(@x : Int32, @y : Int32)
  end

  # 2項演算子。`self`と**other**を足し合わせたものを返す。
  def +(other : self) : self
    Vector2.new(x + other.x, y + other.y)
  end
end

v1 = Vector2.new(1, 2)
v2 = Vector2.new(3, 4)
v1 + v2 # => Vector2(@x=4, @y=6)
```

慣例的に、2項演算子の戻り値の方は1つ目のオペランド (レシーバ) と同じものにします。つまり `typeof(a <op> b) == typeof(a)` ということです。
そうでないと代入演算子 (`a <op>= b`) の場合に意図せず `a` の型が変わってしまうことになります。
ですが、妥当な例外も中には存在します。例えば標準ライブラリには、浮動小数点数の割り算をする演算子 `/` は整数型に対しても常に `Float64` を返します。なぜなら、除算の結果は整数の範囲になるとは限らないからです。

## 3項演算子

[条件分岐をする演算子 (`? : `)](./ternary_if.md)が唯一の3項演算子です。これはメソッドとしてパースされず、意味を変更することはできません。
コンパイラは内部的にこれを `if` 式に変換します。

## 演算子の優先順位

次の表は優先順位の高いものから低いものへ順ににソートされています。

| 分類 | 演算子 |
|---|---|
| インデックス | `[]`, `[]?` |
| 単項演算子 | `+`, `&+`, `-`, `&-`, `!`, `~` |
| 指数 | `**`, `&**` |
| 乗除法 | `*`, `&*`, `/`, `//`, `%` |
| 加減法 | `+`, `&+`, `-`, `&-` |
| ビットシフト | `<<`, `>>` |
| ビット AND | `&` |
| ビット OR/XOR | `|`,`^` |
| 等価性 | `==`, `!=`, `=~`, `!~`, `===` |
| 比較 | `<`, `<=`, `>`, `>=`, `<=>` |
| 論理 AND | `&&` |
| 論理 OR | `||` |
| 範囲 (Range) | `..`, `...` |
| 条件分岐 | `?:` |
| 代入 | `=`, `[]=`, `+=`, `&+=`, `-=`, `&-=`, `*=`, `&*=`, `/=`, `//=`, `%=`, `|=`, `&=`,`^=`,`**=`,`<<=`,`>>=`, `||=`, `&&=` |
| Splat | `*`, `**` |

## 演算子の一覧

### 算術演算子

#### 単項演算子

| 演算子名 | 説明 | 例 | オーバーロード可能か | Associativity |
|---|---|---|---|---|
| `+` | 正の数にする | `+1` | yes | right |
| `&+` | 正の数にする (オーバフローの可能性がある) | `&+1` | yes | right |
| `-` | 負の数にする | `-1` | yes | right |
| `&-` | 負の数にする (オーバーフローの可能性がある) | `&-1` | yes | right |

#### 乗除法

| 演算子名 | 説明 | 例 | オーバーロード可能か | Associativity |
|---|---|---|---|---|
| `**` | 指数 | `1 ** 2` | yes | right |
| `&**` | 指数 (オーバーフローの可能性がある) | `1 &** 2` | yes | right |
| `*` | 乗法 | `1 * 2` | yes | left |
| `&*` | 乗法 (オーバーフローの可能性がある) | `1 &* 2` | yes | left |
| `/` | 除法 | `1 / 2` | yes | left |
| `//` | 整数に丸められる除法 | `1 // 2` | yes | left |
| `%` | 余り | `1 % 2` | yes | left |

#### 加減法

| 演算子名 | 説明 | 例 | オーバーロード可能か | Associativity |
|---|---|---|---|---|
| `+` | 加法 | `1 + 2` | yes | left |
| `&+` | 加法 (オーバーフローの可能性がある) | `1 &+ 2` | yes | left |
| `-` | 減法 | `1 - 2` | yes | left |
| `&-` | 減法 (オーバーフローの可能性がある) | `1 &- 2` | yes | left |

### その他の単項演算子

| 演算子名 | 説明 | 例 | オーバーロード可能か | Associativity |
|---|---|---|---|---|
| `!` | 論理反転 | `!true` | no | right |
| `~` | ビット反転 | `~1` | yes | right |

### ビットスフト

| 演算子名 | 説明 | 例 | オーバーロード可能か | Associativity |
|---|---|---|---|---|
| `<<` | 左シフト、もしくは追記 | `1 << 2`, `STDOUT << "foo"` | yes | left |
| `>>` | 右シフト | `1 >> 2` | yes | left |

### ビット演算


| 演算子名 | 説明 | 例 | オーバーロード可能か | Associativity |
|---|---|---|---|---|
| `&` | ビット AND | `1 & 2` | yes | left |
| `|` | ビット OR | `1 | 2` | yes | left |
| `^` | ビット XOR | `1 ^ 2` | yes | left |

### Equality and Comparison

#### 等価性

基本となる等価性の検査の方法として次のものがあります。

* `==`: オペランドの値が等しいかどうか
* `=~`: 最初のオペランドが2番目のオペランドの値がパターンマッチするかどうか
* `===`: [case 等価性](case.md)によって左のオペランドが右のオペランドにマッチするかどうか。この演算子は `case ... when` の条件分岐でも用いられます。

最初の2つに関しては結果を反転させたものも存在します (`!=` と `!~`)。`a != b` は `!(a == b)` と同じだと考えられて、`a !~ b` は `!(a =~ b)` と同じだと考えられます。
もちろん、これらの演算子を独自に実装することもできます。等価でないことを確認することが等価であることを確認するよりも高速な場合に、パフォーマンスを改善するのに役に立ちます。

| 演算子名 | 説明 | 例 | オーバーロード可能か | Associativity |
|---|---|---|---|---|
| `==` | 等価性の検査 | `1 == 2` | yes | left |
| `!=` | 非等価性の検査 | `1 != 2` | yes | left |
| `=~` | パターンマッチの検査 | `"foo" =~ /fo/` | yes | left |
| `!~` | パターンマッチしないことの検査 | `"foo" !~ /fo/` | yes | left |
| `===` | [case 等価性](case.md) | `/foo/ === "foo"` | yes | left |

#### 比較

| 演算子名 | 説明 | 例 | オーバーロード可能か | Associativity |
|---|---|---|---|---|
| `<` | より小さい | `1 < 2` | yes | left |
| `<=` | より小さいか等しい (以下) | `1 <= 2` | yes | left |
| `>` | より大きい | `1 > 2` | yes | left |
| `>=` | より大きいか等しい (以上) | `1 >= 2` | yes | left |
| `<=>` | 比較 | `1 <=> 2` | yes | left |

#### Chaining Equality and Comparison

Equality and comparison operators `==`, `!=`, `===`, `<`, `>`, `<=`, and `>=`
can be chained together and are interpreted as a compound expression.
For example `a <= b <= c` is treated as `a <= b && b <= c`
and it is even possible to mix operators of the same
[operator precedence](#operator-precedence)
like `a >= b <= c > d`.
Operators with different precedences can be chained too, however, it is advised to avoid it, since it is makes the code harder to understand. For instance `a == b <= c` is interpreted as `a == b && b <= c`, while `a <= b == c` is interpreted as `a <= (b == c)`.

### 論理演算子

| 演算子名 | 説明 | 例 | オーバーロード可能か | Associativity |
|---|---|---|---|---|
| `&&` | [論理 AND](and.md) | `true && false` | no | left |
| `||` | [論理 OR](or.md) | `true || false` | no | left |

### 範囲 (Range)

[Range](literals/range.md) リテラルに範囲演算子は使われます。

| 演算子名 | 説明 | 例 | オーバーロード可能か |
|---|---|---|---|
| `..` | 範囲 | `1..10` | no |
| `...` | 末尾を含まない範囲 | `1...10` | no |

### スプラット

スプラット演算子はタプルを引数に展開するときにのみ利用できます。
詳細は[スプラット展開とタプル](splats_and_tuples.md)を参照してください。

| 演算子名 | 説明 | 例 | オーバーロード可能か |
|---|---|---|---|
| `*` | スプラット展開 | `*foo` | no |
| `**` | 二重スプラット展開 | `**foo` | no |

### 条件分岐

[条件分岐をする演算子 (`? :`)](./ternary_if.md) はコンパイラによって内部的には `if` 式に変換されます。

| 演算子名 | 説明 | 例 | オーバーロード可能か | Associativity |
|---|---|---|---|---|
| `? :` | 条件分岐 | `a == b ?c : d` | no | right |

### 代入

代入演算子 `=` は2番目のオペランドの値を最初のオペランドの示す先に代入します。最初のオペランドは変数 (この場合は再定義できません) もしくはメソッド呼び出し (この場合は再定義できます) のどちらかになります。
詳細は[代入](assignment.md)を参照してください。

| 演算子名 | 説明 | 例 | オーバーロード可能か | Associativity |
|---|---|---|---|---|
| `=` | 変数への代入 | `a = 1` | no | right |
| `=` | 呼び出し代入 | `a.b = 1` | yes | right |
| `[]=` | インデックス指定の代入 | `a[0] = 1` | yes | right |

### 複合代入

代入 `=` は演算子と代入を組み合わせたものの基礎となっています。一般的な形は `a <op>= b` で、コンパイラはこれを `a = a <op> b` に変換します。

例外として、論理演算子に対する複合代入は次のように変換されます。

* `a ||= b` は `a || (a = b)` と変換される。
* `a &&= b` は `a && (a = b)` と変換される。

その他に、これまでの例の `a` がインデクッス指定の代入 (`[]`) の場合も、nil を受け入れるものに変換されます (右側では `[]?` が使われます)。

* `a[i] ||= b` は `a[i] = (a[i]? || b)` に変換されます。
* `a[i] &&= b` は `a[i] = (a[i]? && b)` に変換されます。

すべての変換ではレシーバ (`a`) が変数であることを前提としています。呼び出しだった場合も同様の意味の形に変換されますが、実装は少し複雑になり(一時的な変数を利用します)、また`a=`が呼び出せる必要があります。

レシーバは変数もしくは呼び出し以外にはなりません。

| 演算子名 | 説明 | 例 | オーバーロード可能か | Associativity |
|---|---|---|---|---|
| `+=` | 加法*複合*代入 | `i += 1` | no | right |
| `&+=` | オーバーフローの起こる加法の*複合*代入 | `i &+= 1` | no | right |
| `-=` | 減法*複合*代入 | `i -= 1` | no | right |
| `&-=` | オーバーフローの起こる減法*複合*代入 | `i &-= 1` | no | right |
| `*=` | 乗法*複合*代入 | `i *= 1` | no | right |
| `&*=` | オーバーフローの起こる*複合*代入 | `i &*= 1` | no | right |
| `/=` | 除法*複合*代入 | `i /= 1` | no | right |
| `//=` | 整数に丸められる除法の*複合*代入 | `i //= 1` | no | right |
| `%=` | 余りの*複合*代入 | `i %= 1` | yes | right |
| `|=` | ビット OR *複合*代入 | `i |= 1` | no | right |
| `&=` | ビット AND *複合*代入 | `i &= 1` | no | right |
| `^=` | ビット XOR *複合*代入 | `i ^= 1` | no | right |
| `**=` | 指数の*複合*代入 | `i **= 1` | no | right |
| `<<=` | 左シフト*複合*代入 | `i <<= 1` | no | right |
| `>>=` | 右シフト*複合*代入 | `i >>= 1` | no | right |
| `||=` | logical or *and* assignment | `i ||= true` | no | right |
| `&&=` | 論理 AND *複合*代入 | `i &&= true` | no | right |

### インデックスアクセサ

インデックスアクセサは配列の要素やハッシュのエントリなど、インデックスもしくはキーに対応する値を取得するために使います。nil を受け入れるもの `[]?` はもしインデックスに対応する値が存在しなかった場合に `nil` を返すものです。nil を受け入れない場合はその場合にはエラーを起こします。
標準ライブラリではそのような場合通常、[`KeyError`](https://crystal-lang.org/api/KeyError.html) か [`IndexError`](https://crystal-lang.org/api/IndexError.html) がエラーとして投げなられます。

| 演算子名 | 説明 | 例 | オーバーロード可能か |
|---|---|---|---|
| `[]` | インデックスアクセサ | `ary[i]` | yes |
| `[]?` | nil を受け入れるインデックスアクセサ | `ary[i]?` | yes |