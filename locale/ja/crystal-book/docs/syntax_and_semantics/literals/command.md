# コマンドリテラル

コマンドリテラルとは、バックティック `` ` `` で囲まれた文字列、もしくは `%x` パーセントリテラルのことです。
実行時に文字列の内容をコマンドとしてサブシェルで実行して、その出力の文字列が結果の値となります。

[エスケープシーケンス](./string.md#escaping)と[補間](./string.md#interpolation)は通常の文字列と同様に利用できます。

パーセントリテラルの形式の場合、他のパーセントリテラルと同様に`%x`では、括弧`()`、角括弧`[]`、ひげ括弧`{}`、三角括弧`<>`そしてパイプ`||`といった区切り文字が有効です。パイプ文字を除いて、すべての区切り文字はネストに応じて適切に処理されます。

The special variable `$?` holds the exit status of the command as a [`Process::Status`](https://crystal-lang.org/api/Process/Status.html). この特殊変数はコマンドリテラルと同じスコープに限り有効です。

```crystal
`echo foo`  # => "foo"
$?.success? # => true
```

Internally, the compiler rewrites command literals to calls to the top-level method [`` `()``](https://crystal-lang.org/api/toplevel.html#%60(command):String-class-method) with a string literal containing the command as argument: `` `echo #{argument}` `` and `%x(echo #{argument})` are rewritten to `` `("echo #{argument}")``.

## セキュリティ上の懸念

コマンドリテラルはスクリプトのような簡易的な利用時に便利ですが、補間を使う場合にはコマンドインジェクションが起こらないように注意する特別の注意を払う必要があります。

```crystal
user_input = "hello; rm -rf *"
`echo #{user_input}`
```

このコマンドは `hello` と出力したあとに現在のディレクトリのファイルとフォルダを全て削除します。

これを避けるには、コマンドリテラルの補間にユーザーの入力した値を用いないようにする必要があります。[`Process`](https://crystal-lang.org/api/Process.html) from the standard library offers a safe way to provide user input as command arguments:

```crystal
user_input = "hello; rm -rf *"
process = Process.new("echo", [user_input], output: Process::Redirect::Pipe)
process.output.gets_to_end # => "hello; rm -rf *"
process.wait.success?      # => true
```
