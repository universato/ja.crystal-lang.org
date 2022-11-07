# 初期化しない変数宣言

Crystal では、初期化せず変数を宣言することが可能です。

```crystal
x = uninitialized Int32
x # => some random value, garbage, unreliable
```

This is [unsafe](unsafe.md) code and is almost always used in low-level code for declaring uninitialized [StaticArray](https://crystal-lang.org/api/StaticArray.html) buffers without a performance penalty:

```crystal
buffer = uninitialized UInt8[256]
```

このとき、バッファはヒープではなくスタックに割り当てられます。

`uninitialized` キーワードに続く型は [型の文法](type_grammar.md)にしたがって書きます。
