サーバの管理などでBashなどを使ってバッチ処理のスクリプトを書くことが時々あるのですが、今回はそれをElixirで行ってみようと思います。
Elixirにはelixirコマンドがあり、これを使用することでElixirのスプリクトを実行することができます。

ファイルの拡張子はElixirスクリプトの命名規則に倣って.exsとします。
最初の行にはシェバンを記述します。
標準ライブラリだけを使うのであればmix newをする必要はなく、適当な場所にファイルを作ればOKです。

```
#!/usr/bin/env elixir
IO.puts "Hello world!"
```

ファイルを保存したら、chmodコマンドで実行権限を与えて実行してみます。

```bash
$ chmod +x hello.exs
$ ./hello.exs
Hello world!
```

簡単ですね。
次はコマンドライン引数を受け取って１つづつ表示をしてみます。

```ex
#!/usr/bin/env elixir
System.argv()
|> Enum.each(fn(x) -> IO.puts x end)
```

```bash
$ chmod +x argv.exs
$ ./argv.exs foo bar
foo
bar
```

System.argv()はリストを返すのでEnumにある関数でデータをいろいろ加工することができます。
引数で渡された数値を足すスクリプトを作ってみます。
コマンドライン引数は文字列型なので、いったん数値型に変換してから足しあわせます。

```ex
#!/usr/bin/env elixir
System.argv()
|> Enum.map(fn(x) -> String.to_integer x end)
|> Enum.sum
|> IO.puts
```

```bash
$ chmod +x calc.exs
$ ./calc.exs 1 2 3 4
10
```

次はパイプでlsコマンドから出力されたディレクトリのリストをElixirスクリプトで受け取ります。
受け取った際に正規表現を使って拡張子に.exsを持つファイルだけに絞り込むようにします。

```ex
#!/usr/bin/env elixir
IO.stream(:stdio, :line)
|> Enum.filter(fn(x) -> x =~ ~r/\.exs/ end)
|> Enum.each(fn(x) -> IO.write(:stdio, x) end)
```

```bash
$ chmod +x pipe.exs
$ ls -la | ./pipe.exs
-rwxr-xr-x    1 hideshi  staff    73 12  1 06:58 args.exs
-rwxr-xr-x    1 hideshi  staff   105 12  1 07:01 calc.exs
-rwxr-xr-x    1 hideshi  staff    44 12  1 06:33 hello.exs
-rwxr-xr-x    1 hideshi  staff   138 12  1 07:11 pipe.exs
```

ElixirスクリプトからOSコマンドを実行することもできます。
[この記事](http://qiita.com/darui_kara/items/755463f2c4769d1777ed)が詳しいです。

```ex
#!/usr/bin/env elixir
{result, 0} = System.cmd "ls", ["-la"]
result
|> String.split("\n")
|> Enum.filter(fn(x) -> x =~ ~r/\.exs/ end)
|> Enum.each(fn(x) -> IO.puts x end)
```

```bash
$ chmod +x cmd.exs
$ ./cmd.exs
-rwxr-xr-x    1 hideshi  staff    73 12  1 06:58 args.exs
-rwxr-xr-x    1 hideshi  staff   105 12  1 07:01 calc.exs
-rwxr-xr-x    1 hideshi  staff   171 12  1 07:54 cmd.exs
-rwxr-xr-x    1 hideshi  staff    44 12  1 06:33 hello.exs
-rwxr-xr-x    1 hideshi  staff   138 12  1 07:11 pipe.exs
```

標準出力に出力していますので、その後でパイプで他のコマンドに渡すこともできます。
grepで文字列calcを含む行だけ抽出してみます。

```bash
$ ./cmd.exs |grep calc
-rwxr-xr-x    1 hideshi  staff   105 12  1 07:01 calc.exs
```

最後にコマンドラインオプションをパースしてみましょう。

```ex
#!/usr/bin/env elixir
System.argv
|> OptionParser.parse
|> IO.inspect
```

```bash
$ chmod +x opt.exs
$ ./opt.exs -x --debug --verbose -c foo bar
{[debug: true, verbose: true], ["bar"], [{"-x", nil}, {"-c", "foo"}]}
```

うまくパースできたようですね。

もし他にも便利な機能があればコメント頂ければと思います。

