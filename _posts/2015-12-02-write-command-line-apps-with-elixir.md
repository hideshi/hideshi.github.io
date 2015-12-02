この文章はIlija Eftimovさんの2015年11月27日付のブログ記事[Writing command line apps with Elixir](http://eftimov.net/writing-elixir-cli-apps/)の翻訳です。
Elixirを日常的に使用する簡易なツールを作るための言語としても利用できたらなと思っていたところこの記事を見つけました。
誤訳や関連記事などがあればコメント欄にお願いいたします。

---
Elixirはとてもクールな言語だ。まだ十分な経験があるわけではないが、私はいつもこれで興味深いものを作り、ビルトインツールを学んでいる。この記事ではescriptを使ってコマンドラインアプリケーションの作り方をお見せしようと思う。

# **Escript**
ErlangとElixirはescriptというクールなツールを持っている。これは基本的にはElixirのアプリケーションをコマンドラインアプリケーションにコンパイルするものだ。

[Elixirのescriptのドキュメント](http://elixir-lang.org/docs/master/mix/Mix.Tasks.Escript.Build.html)からの引用

> escriptはコマンドラインから実行可能である。Erlangがインストールされていればどのような環境でも実行可能で、Elixirはコマンドラインアプリケーションに埋め込まれるためデフォルトではElixirのインストールは要求されない。

興味深いことはescriptはElixirアプリケーションをビルドしコマンドラインアプリケーションを作成するということだ。実行可能なアプリケーションはErlangさえインストールされていればどの環境でも動作する。

# **eight_ball**を**escript**でくるむ
例示のための小さなアプリケーションを作る代わりに、以前に[Write and publish your first Elixir library](http://eftimov.net/writing-elixir-library/)という記事を書く際に作ったeight_ballというアプリケーションを流用しようと思う。もしこの記事を読んでいないようであれば一読して戻ってくることをお勧めする。
このアプリケーションはすごくシンプルで、**EightBall.ask/1**という関数だけを持つ。引数として質問を取り、ランダムに回答を返す。では**escript**を使ってこれのコマンドラインアプリケーションを作ってみよう。

# アプリケーションにescriptを追加する
もしこの話題についていきたいのであれば、[このリポジトリ](https://github.com/fteem/eight_ball)にあるソースを見ておくといいだろう。
**mix.exs**に追加する。

{% codehighlight elixir %}
defmodule EightBall.Mixfile do
  use Mix.Project

  def project do
    [app: :eight_ball,
     version: "0.0.1",
     elixir: "~> 1.0",
     build_embedded: Mix.env == :prod,
     start_permanent: Mix.env == :prod,
     escript: [main_module: EightBall.CLI], # <- this line
     deps: deps,
     package: package ]
  end
  # ...
end
{% endcodehighlight %}

追加した行は**escript**に**EightBall.CLI**に**main/1**関数があることを知らせるものだ。**main/1**関数はコマンドラインアプリケーションのエントリーポイントとなる。

# **EightBall.CLI**
**EightBall.CLI**モジュールがまだないので、まずはそれを作るところから始めよう。

{% codehighlight elixir %}
defmodule EightBall.CLI do
  def main(args) do
  end
end
{% endcodehighlight %}

見ての通りこのモジュールはコマンドライン引数を取る**main/1**関数を持っている。コマンドライン引数を扱いやすくするために[**OptionParser**](http://elixir-lang.org/docs/v1.0/elixir/OptionParser.html)を使う必要がある。
アプリケーションを実行する際のシンタックスはこんな感じか

{% codehighlight bash %}
eight_ball --question "Is Elixir great?"
{% endcodehighlight %}

こんな感じにしたい。

{% codehighlight bash %}
eight_ball -q "Is Elixir great?"
{% endcodehighlight %}

では**OptionParser**を使って**-q/--question**引数をオプションとして定義してみよう。

{% codehighlight elixir %}
defmodule EightBall.CLI do
  def main(argv) do
    {options, _, _} = OptionParser.parse(argv, 
      switches: [question: :string],
    )

    IO.inspect options
  end
end
{% endcodehighlight %}

**main/1**関数は引数をパースし意味あるタグ付けされたリストを生成する。この段階では**main/1**関数はパースされた引数を表示するだけとなっている。後ほど意味のある処理を追加していこうと思う。
まずは実行可能はescriptを作ってみよう。作り方はプロジェクトのルートディレクトリに移動して以下のコマンドを実行する。

{% codehighlight bash %}
mix escript.build
{% endcodehighlight %}

これはElixirアプリケーションをコンパイルし実行可能なescriptを生成する。

{% codehighlight bash %}
➜  eight_ball git:(master) ✗ mix escript.build
Compiled lib/eight_ball/cli.ex
Generated eight_ball app
Generated escript eight_ball with MIX_ENV=dev
{% endcodehighlight %}

**eight_ball**プロジェクトのルートディレクトリを見ると、eight_ballという実行可能なファイルが出来上がっていることがわかるだろう。これを使うには以下のように入力する。

{% codehighlight bash %}
./eight_ball --question "Is Elixir great?"
{% endcodehighlight %}

そうすると以下のように表示されると思う。

{% codehighlight bash %}
➜  eight_ball git:(master) ✗ ./eight_ball -q "Is Elixir great?"
[question: "Is Elixir great?"]
{% endcodehighlight %}

ほらみて！パースされたコマンドライン引数が見えるでしょ！

# **アプリケーションを統合する**
では**EightBall.ask/1**をコマンドラインアプリケーションで使ってみよう。**EightBall::CLI.main/1**に以下のコードを追加してみよう。

{% codehighlight elixir %}
defmodule EightBall.CLI do
  def main(opts) do
    {options, _, _} = OptionParser.parse(opts, 
      switches: [question: :string],
      aliases: [q: :question] # makes '-q' an alias of '--question'
    )

    try do
      IO.puts EightBall.ask(options[:question]) 
    rescue 
      e in RuntimeError -> e
        IO.puts e.message
    end
  end
end
{% endcodehighlight %}

try/rescueブロックの中で質問文を**ask/1**関数に送信し、RuntimeErrorをキャッチする。このエラーは**EightBall::QuestionValidator**という入力値チェック機能によって引き起こされる。もし質問文が疑問形でない場合にこのようなエラーを投げる。

{% codehighlight bash %}
"Question must be a string, ending with a question mark."
{% endcodehighlight %}

もしコマンドラインアプリケーションがエラーをキャッチした場合はエラーメッセージを表示する。

# コマンドラインアプリケーションをビルドする
最後のステップはescriptを実行することだ。Elixirでは**mix**を使うことで**escript**を実行することはとても簡単である。

{% codehighlight bash %}
mix escript.build
{% endcodehighlight %}

もし手順に誤りがなければ以下のように表示されるだろう。

{% codehighlight bash %}
➜  eight_ball git:(master) ✗ mix escript.build
Compiled lib/eight_ball.ex
Generated eight_ball app
Generated escript eight_ball with MIX_ENV=dev
{% endcodehighlight %}

これはメインモジュールと同名のコマンドラインアプリケーションを生成する。この場合は**eight_ball**のようになる。もし実行可能ファイルをテキストエディタなどで開くと、読むことが困難なたくさんのコードを目にするだろう。これはErlangVMのバイトコードに変換されたという証拠である。
素晴らしいことはこれをErlangのインストールされたマシンに送り使用することができる。Elixirはアプリケーションに埋め込まれているので、唯一の依存性はErlangだけである。クールでしょ？

アプリケーションは以下のようにして実行する。

{% codehighlight bash %}
➜  eight_ball git:(master) ✗ ./eight_ball --question "Is Elixir awesome?"
Outlook good
{% endcodehighlight %}

もしくはこのように。

{% codehighlight bash %}
➜  eight_ball git:(master) ✗ ./eight_ball --question "Is Elixir awesome"
Question must be a string, ending with a question mark.
{% endcodehighlight %}

