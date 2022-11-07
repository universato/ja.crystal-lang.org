# 浮動小数点数 (Float)

There are two floating point types, [Float32](https://crystal-lang.org/api/Float32.html) and [Float64](https://crystal-lang.org/api/Float64.html),
which correspond to the [binary32](http://en.wikipedia.org/wiki/Single_precision_floating-point_format)
and [binary64](http://en.wikipedia.org/wiki/Double_precision_floating-point_format)
types defined by IEEE.

浮動小数点リテラルは `+` (オプション) または `-` 符号に続く数値とアンダースコアからなり、さらに指数部分と型を表わすサフィックスを続けることができます。サフィックスがない場合はリテラルの型は`Float64`となります。

```crystal
1.0     # Float64
1.0_f32 # Float32
1_f32   # Float32

1e10   # Float64
1.5e10 # Float64
1.5e-7 # Float64

+1.3 # Float64
-0.5 # Float64
```

サフィックスの前のアンダースコア (`_`) はオプションです。

アンダースコアは、数値を読みやすくするためにも利用できます。

```crystal
1_000_000.111_111 # 1000000.111111 を読みやすくしたものです
```
