# ハッシュ (Hash)

A [Hash](https://crystal-lang.org/api/Hash.html) is a generic collection of key-value pairs mapping keys of type `K` to values of type `V`.

ハッシュは通常、ひげ括弧 (`{ }`) に囲われた中に、キーと値を `=>` で組にしてコンマ `,` で区切って並べたものである、ハッシュリテラルによって生成されます。

```crystal
{"one" => 1, "two" => 2}
```

## ジェネリック型引数

キーの型の `K` と値の型の `V` というジェネリック型引数は、それぞれリテラル中のキーと値の型から推論されます。すべて同じ型であれば、`K`および`V`はそれに等しくなります。そうでなければ、すべてのキーの型のユニオン型ないし値の型のユニオン型となります。

```crystal
{1 => 2, 3 => 4}   # Hash(Int32, Int32)
{1 => 2, 'a' => 3} # Hash(Int32 | Char, Int32)
```

閉じひげ括弧のあとに続けて`of`とキーの型 (`K`)、さらに `=>` に続けて値の型 (`V`) を置くことで、明示的に型を指定することもできます。これは推論された型を置き換えるので、初期化時には同じ型のみを持っているが、他の型も受け入れる必要があるようなハッシュを生成したい場合に対応できます。

空のハッシュリテラルは常に型を指定する必要があります。

```crystal
{} of Int32 => Int32 # => Hash(Int32, Int32).new
```

## ハッシュライクな型のリテラル

Crystal はさらにハッシュライクな型のためのリテラルをサポートしています。型の名前のあとに続けたひげ括弧 (`{}`) の中に、コンマ区切りのキーと値の組を並べることで利用できます。

```crystal
Hash{"one" => 1, "two" => 2}
```

引数を持たないコンストラクタと`[]=`メソッドを持つ任意の型に対して、この構文は利用できます。

```crystal
HTTP::Headers{"foo" => "bar"}
```

ジェネリック型でない型`HTTP::Headers`に対しては、上の例は以下と等しいものになります。

```crystal
headers = HTTP::Headers.new
headers["foo"] = "bar"
```

ジェネリック型の場合、型引数はハッシュリテラルと同様の方法でキーと値の型から推論されます。

```crystal
MyHash{"foo" => 1, "bar" => "baz"}
```

ここで`MyHash`がジェネリック型であれば、上の例は以下と等しいものになります。

```crystal
my_hash = MyHash(typeof("foo", "bar"), typeof(1, "baz")).new
my_hash["foo"] = 1
my_hash["bar"] = "baz"
```

型引数は型名の部分で明示的に指定することもできます。

```crystal
MyHash(String, String | Int32){"foo" => "bar"}
```
