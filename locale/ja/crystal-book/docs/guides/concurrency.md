# 並行処理

## 並行処理（Concurrency） vs. 並列処理（Parallelism）

しばしば混同されますが、「並行処理（concurrency）」と「並列処理（parallelism）」という言葉の定義は同じではありません。

A concurrent system is one that can be in charge of many tasks, although not necessarily executing them at the same time. キッチンで料理をしているところを考えてみましょう。あなたは玉ねぎをきざんでフライパンに放り込み、それが炒め上がるまでの間にトマトを切ったりできます。しかし、すべての動作を同時に行うのではなく、それぞれのタスクにあなたの時間を割り振って処理することでしょう。並列処理（parallelism）というのは、片手で玉ねぎの入ったフライパンを振りながらもう片方の手でトマトを切るようなものです。

これを書いている時点では，Crystal は「並行処理」をサポートしていますが、「並列処理」はサポートしていません。つまり、いくつものタスクを実行することができて、それぞれのタスクに少しずつ時間を割り当てることも可能ですが、完全に同じタイミングで2つのコードパスが実行されることはありません。

コンカレント・マーク＆スイープ方式で実装されたガベージコレクタ（GC）（現時点では [Boehm GC](http://www.hboehm.info/gc/)) を除いて、Crystal のプログラムは単一の OS スレッド上で実行されます。

### ファイバ（Fiber）

並行処理を行うために、Crystal はファイバを利用します。ファイバはOSスレッドとよく似ていますがより軽量で、その実行はプロセスによって内部的に管理されています。そして、プログラムは複数のファイバを生成することができ、Crystalは適切なタイミングでそれらを実行してくれます。

### イベントループ

I/Oに関係するものであればなんでも、イベントループのお世話になっています。おかげで、たとえ時間のかかる処理がI/Oに対して行われていてイベントループがその終了を待っている状態でも、プログラムは別のファイバを実行することができます。ソケット経由でのデータの受信操作などは、こうした時間がかかる処理の一例です。

### チャネル（Channel）

Crystal は [CSP](https://en.wikipedia.org/wiki/Communicating_sequential_processes) を参考にしたチャネルを有しています。チャネルによって、メモリの共有やロックやセマフォなどといった特別な機構を気にかけることなく、異なるファイバ間でのデータのやりとりが可能になります。

## プログラムの実行

プログラムが開始されると、まずトップレベルコードを実行するためのメインファイバを起動します。そして、そのメインファイバは、さらに多くの他のファイバを生成することができます。プログラムを構成するコンポーネントは以下の通りです。

* すべてのファイバを適切なタイミングで起動するための「ランタイムスケジューラ」
* ファイルやソケット、パイプ、シグナル、タイマ（`sleep`したときなど）による非同期タスクを扱うための「イベントループ」（これもファイバの1種）
* ファイバ間でデータをやりとりする「チャネル」ラインタイムスケジューラはデータをやりとるするためにファイバとチャネルの間を調整します。
* "もう使用されない" メモリを掃除する「ガベージコレクタ」

### ファイバ単体

単一のファイバは、スレッドと比べるとより軽量な処理の実行単位です。ファイバは、通常 OS スレッド上の[スタックメモリ](https://en.wikipedia.org/wiki/Call_stack)上に8MBが割り当てられた小さなオブジェクトです。

ファイバはスレッドと違って協調的に動作します。スレッドはプリエンプティブ（非同期マルチタスク）なので，OSはいつでもスレッドに割り込んで別のスレッドを実行することができます。ファイバはラインタイムスケジューラに対して、明示的に「他のファイバへ処理を切り替えて良い」と伝える必要があります。例えばI/O による待ちが発生していた場合、ファイバはランタイムスケジューラに対して「おーい，自分はI/Oが使えるようになるまで待たないといけないから、他のファイバを実行していいよ。I/Oの準備ができたら帰ってきてね」と告げます。

こうした協調動作の利点は、コンテキスト切り替え（スレッド間の切り替え）によるオーバーヘッドの大部分をなくすことができることです。

ファイバはスレッドと比べて軽量で、8MB割り当てられていてもその開始時には4KBという小さなスタック領域しか使用しません。

64ビットマシンでは数百万ものファイバを生成することができます。32ビットマシンの場合はそれほど多くはなく、生成できるファイバは512個だけです。しかし、32ビットマシンはもはや使用されなくなりつつありますので、我々は将来を見据えて64ビットマシンにより注力しています。

### ランタイムスケジューラ

ランタイムスケジューラは以下のキューを持っています。

* 実行可能なファイバ（例えば，新しく生成されたファイバは実行可能な状態です）
* イベントループ（別のファイバ）他に実行可能なファイバが存在しない場合、イベントループは準備が完了しているな非同期処理があるかをチェックし、その処理を待っていたファイバを実行します。現時点では、イベントループは`epoll`や`kqueue`と言ったイベント方式を抽象化した`libevent`を使用して実装されています。
* 自主的に待機しているファイバ（`Fiber.yield`によって「続けて実行できるけれど、他に実行したいファイバがあれば、そっちを実行してていいよ」と宣言したもの）

### データのやり取り

現時点ではシングルスレッドでしかコードを実行できなため、複数のファイバから同じクラス変数を参照や変更しても問題なく動作します。しかし、一旦マルチスレッド（並列処理）が言語に導入されると、そういう訳にもいかなくなります。このことが、データのやりとりにチャネルを使い、メッセージをチャネル経由で送受信することが推奨される理由です。チャネルは内部的にデータ競合を避けるためのロック機構を実装していますが、外から使う分にはロックを考慮する必要のないデータの送受信手法として使用できます。

## サンプルコード

### ファイバの生成

ファイバを生成するにはブロック付きで`spawn`を使用します。

```crystal
spawn do
  # ...
  socket.gets
  # ...
end

spawn do
  # ...
  sleep 5.seconds
  #  ...
end
```

ここには、2つのファイバがあります。1つ目はソケットから何かを読み出すもので、もう一つは`sleep`するものです。1つめのファイバは`socket.gets`の行に到達すると一旦中断し、イベントループにソケットにデータが準備できた時点でこのファイバを再開するように伝えます。そして、プログラムは2つめのファイバを実行します。こちらのファイバは5秒間スリープし、イベントループに5秒経ったらこちらのファイバを再開するよう伝えます。もし他に実行可能なファイバがいなければ、イベントループはイベントループはCPUを消費することなく、どちらかのイベントが発生するまで待機します。

`socket.gets`や`sleep`がこのように動作する理由は、それらがラインタイムスケジューラやイベントループと直接会話できるように実装されているからで、魔法でもなんでもありません。基本的に、標準ライブラリは使用者がそうした一切を気にかける必要がないように配慮されています。

ただし、ファイバが（生成後）即座に実行されるわけではないことに注意してください。例をあげます。

```crystal
spawn do
  loop do
    puts "Hello!"
  end
end
```

上記コードは何も出力せず即座に終了します。

その理由は、ファイバーが生成された時点で即座に実行されるわけではないためです。そのため、上の例でファイバを生成したメインファイバはそのまま終わりに達してプログラムも終了してしまいました。

この問題を解決留守方法の一つは、`sleep`を使うことです。

```crystal
spawn do
  loop do
    puts "Hello!"
  end
end

sleep 1.second
```

今回のプログラムは1秒間 "Hello!" と出力し続けてから終了します。これは `sleep`によってメインファイバが1秒後に再開するようスケジューリングされ、その間に他の "実行可能な" （この場合はすぐ上で生成されていた）ファイバが実行されるためです。

他にはこんな方法もあります。

```crystal
spawn do
  loop do
    puts "Hello!"
  end
end

Fiber.yield
```

今度は `Fiber.yield` がスケジューラに対して他のファイバを実行して良いと伝えています。その結果、標準出力がブロックされる（システムコールが標準出力の準備ができるまで待つよう伝えてくる）まで "Hello!" と出力し続けます。ただし、標準出力はブロックされない*かもしれません*。

もし、生成したファイバを実行させ続けたいのであれば、引数なしの `sleep` を使うことができます。

```crystal
spawn do
  loop do
    puts "Hello!"
  end
end

sleep
```

上のプログラムは単純にループを行なっているだけですので、もちろん`spawn`を使わずに書くこともできます。`sleep`は、複数のファイバを生成する場合にはさらに有用です。

### メソッド呼び出しをファイバとして生成する

ブロックを与える代わりに、メソッドの呼び出しを渡してファイバを生成することもできます。なぜこれが便利なのかを理解するために、次の例をみてみましょう。

```crystal
i = 0
while i < 10
  spawn do
    puts(i)
  end
  i += 1
end

Fiber.yield
```

上のプログラムは "10" を10回出力します。ここでの問題は、1つの変数 `i` を全てのファイバが参照していて、`Fiber.yield` が実行された頃にはその値が10になってしまっている点です。

これを解決するにはこのようにします。

```crystal
i = 0
while i < 10
  proc = ->(x : Int32) do
    spawn do
      puts(x)
    end
  end
  proc.call(i)
  i += 1
end

Fiber.yield
```

Now it works because we are creating a [Proc](https://crystal-lang.org/api/Proc.html) and we invoke it passing `i`, so the value gets copied and now the spawned fiber receives a copy.

上記のようなボイラープレートコードを回避するために、標準ライブラリは式の呼び出しを受け取って上記コードに展開する`spawn` マクロを用意しています。それを使い、最終的にはこうなりました。

```crystal
i = 0
while i < 10
  spawn puts(i)
  i += 1
end

Fiber.yield
```

この方法は、繰り返し処理の中でローカル変数の値が変化するような場合には便利な方法です。。こうした問題はブロック引数では発生しません。例えば、以下のコードは想定した通りに動作します。

```crystal
10.times do |i|
  spawn do
    puts i
  end
end

Fiber.yield
```

### ファイバを生成してその終了を待つ

そのような時にもチャネルが使えます。

```crystal
channel = Channel(Nil).new

spawn do
  puts "Before send"
  channel.send(nil)
  puts "After send"
end

puts "Before receive"
channel.receive
puts "After receive"
```

このコードの出力は以下の通りです。

```
Before receive
Before send
After receive
```

まず、プログラムはファイバを生成しますが、まだそのファイバは実行されません。`channel.receive`が実行された時点で、メインファイバがブロックされ、生成されたファイバに処理が移ります。そして、`channel.send(nil)` が実行されると、値が渡されるのを待っていた`channel.receive`が実行されます。その後、メインファイバは実行を続けて終了してしまうため、プログラムは "After send" を出力する機会のないまま終了してしまいます。

上の例では単にファイバが終了したことを伝えるだけの目的で `nil` を使いました。チャネルを使って以下のようにファイバ間で値を受け渡すこともできます。

```crystal
channel = Channel(Int32).new

spawn do
  puts "Before first send"
  channel.send(1)
  puts "Before second send"
  channel.send(2)
end

puts "Before first receive"
value = channel.receive
puts value # => 1

puts "Before second receive"
value = channel.receive
puts value # => 2
```

出力はこうなります。

```
Before first receive
Before first send
1
Before second receive
Before second send
2
```

プログラムがいつ`receive`を実行したか、つまりいつそのファイバをブロックして他のファイバに実行が切り替わったかに注目してください。`send` が実行されると、そのチャネルを待ち受けていたファイバの実行が再開されます。

今回はリテラル値を渡していますが、生成されたファイバはファイルから読み込んだり、ソケットから値を取得したりして、この値を計算する場合もあります。ファイバがI/Oを待たなければいけなくなった場合、I/Oの準備ができるまで他のファイバを実行し、値が到着しチャネルを通じてそれが送信されると、メインファイバがその値を受信します。例をあげます。

```crystal
require "socket"

channel = Channel(String).new

spawn do
  server = TCPServer.new("0.0.0.0", 8080)
  socket = server.accept
  while line = socket.gets
    channel.send(line)
  end
end

spawn do
  while line = gets
    channel.send(line)
  end
end

3.times do
  puts channel.receive
end
```

このプログラムは2つのファイバを生成しています。1つ目のファイバはTCPサーバを立ててコネクションを1つ受け入れ、そこから行を読み込んでチャネルに渡すもの。2つ目のファイバは標準入力から行を読み込むものです。メインファイバは、ソケットから読み込まれたのか標準入力から読み込まれたのかにかかわらず、チャネルに渡されたメッセージを3回読み込んだら終了します。`gets` の呼び出しは、そのファイバをブロックして、イベントループに読み出す値が到着したら再開してくれるように伝えます。

同様に、処理を完了するために複数のファイバを待ち受けて、それらの値を集約することもできます。

```crystal
channel = Channel(Int32).new

10.times do |i|
  spawn do
    channel.send(i * 2)
  end
end

sum = 0
10.times do
  sum += channel.receive
end
puts sum # => 90
```

もちろん、生成されたファイバ内で`receive` を使うことも可能です。

```crystal
channel = Channel(Int32).new

spawn do
  puts "Before send"
  channel.send(1)
  puts "After send"
end

spawn do
  puts "Before receive"
  puts channel.receive
  puts "After receive"
end

puts "Before yield"
Fiber.yield
puts "After yield"
```

出力はこうなります。

```
Before yield
Before send
Before receive
1
After receive
After send
After yield
```

ここでは、まず`channel.send`が実行されますが、まだだれもその値を待っている人がいないので、他のファイバを実行します。2つ目のファイバが実行されると、チャネルにすでに値があるのでそれを受け取って実行を続け、その後まず1つ目のファイバが実行され、最後にメインファイバが実行されます。こうした順序になる理由は、`Fiber.yield`がそのファイバを実行キューの最後に置くためです。

### バッファ付きチャネル

上の例ではバッファのないチャネル、つまり値を送信した際にそのチャネルの値を待っているファイバがあればそちらに処理が移るもの、を使用しました。

バッファ付きのチャネルを使った場合、`send`が実行されてもバッファが一杯になるまでは他のファイバに処理が切り替わりません。

```crystal
# 容量が2のバッファ付きチャネル
channel = Channel(Int32).new(2)

spawn do
  puts "Before send 1"
  channel.send(1)
  puts "Before send 2"
  channel.send(2)
  puts "Before send 3"
  channel.send(3)
  puts "After send"
end

3.times do |i|
  puts channel.receive
end
```

出力はこうなります。

```
Before send 1
Before send 2
Before send 3
After send
1
2
3
```

Note that the first `send` does not occupy space in the channel. This is because there is a `receive` invoked prior to the first `send` whereas the other 2 `send` invocations take place before their respective `receive`. The number of `send` calls do not exceed the bounds of the buffer and so the send fiber runs uninterrupted to completion.

Here's an example where all space in the buffer gets occupied:

```crystal
# A buffered channel of capacity 1
channel = Channel(Int32).new(1)

spawn do
  puts "Before send 1"
  channel.send(1)
  puts "Before send 2"
  channel.send(2)
  puts "Before send 3"
  channel.send(3)
  puts "End of send fiber"
end

3.times do |i|
  puts channel.receive
end
```

出力はこうなります。

```
Before send 1
Before send 2
Before send 3
1
2
3
```

Note that "End of send fiber" does not appear in the output because we `receive` the 3 `send` calls which means `3.times` runs to completion and in turn unblocks the main fiber which executes to completion.

Here's the same snippet as the one we just saw - with the addition of a `Fiber.yield` call at the very bottom:

```crystal
# A buffered channel of capacity 1
channel = Channel(Int32).new(1)

spawn do
  puts "Before send 1"
  channel.send(1)
  puts "Before send 2"
  channel.send(2)
  puts "Before send 3"
  channel.send(3)
  puts "End of send fiber"
end

3.times do |i|
  puts channel.receive
end

Fiber.yield
```

出力はこうなります。

```
Before send 1
Before send 2
Before send 3
1
2
3
End of send fiber
```

With the addition of a `Fiber.yield` call at the end of the snippet we see the "End of send fiber" message in the output which would have otherwise been missed due to the main fiber executing to completion.
