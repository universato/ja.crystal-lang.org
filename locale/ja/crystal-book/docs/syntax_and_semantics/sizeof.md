# sizeof

`sizeof` は与えられた型のバイト数を `Int32` で返します。例をあげます。

```crystal
sizeof(Int32) # => 4
sizeof(Int64) # => 8
```

For [Reference](https://crystal-lang.org/api/Reference.html) types, the size is the same as the size of a pointer:

```crystal
# 64ビット処理系の場合
sizeof(Pointer(Int32)) # => 8
sizeof(String)         # => 8
```

この理由は、Reference のメモリはヒープに割り当てられ、渡されるのはそこを指すポインタになるためです。
実際のクラスのサイズを得るには、[instance_sizeof](instance_sizeof.md) を使用してください。

sizeof の引数は[型](type_grammar.md)なので[typeof](typeof.md)とあわせて使うことが多いでしょう。

```crystal
a = 1
sizeof(typeof(a)) # => 4
```
