# Cool.io と Ruby C拡張 と GVL とイベント駆動と

Fluentd が使っているイベント駆動ライブラリである [cool.io](https://github.com/tarcieri/cool.io) をカバーする。

なお、オリジナル作者である [Tony](https://github.com/tarcieri) は [Celluloid IO](https://github.com/celluloid/celluloid-io) を使ってね、
と言っているので、普通はそちらを使うべき。

Fluentd が未だに cool.io を使っているのは

1. コミット権ももらったし(repeatedly)、困っていない

ぐらいの理由である。

Cool.io は内部的にはマルチプラットフォームイベント駆動ライブラリである libev
を利用し、ruby から呼べるようにC拡張APIでラップしているので、そちらから入門する

* [Ruby C拡張](#ruby-c%E6%8B%A1%E5%BC%B5)
  * [Ruby C拡張を含んだ gem の作り方](#ruby-c%E6%8B%A1%E5%BC%B5%E3%82%92%E5%90%AB%E3%82%93%E3%81%A0-gem-%E3%81%AE%E4%BD%9C%E3%82%8A%E6%96%B9)
  * [Ruby C拡張を含んだ gem のビルド方法](#ruby-c%E6%8B%A1%E5%BC%B5%E3%82%92%E5%90%AB%E3%82%93%E3%81%A0-gem-%E3%81%AE%E3%83%93%E3%83%AB%E3%83%89%E6%96%B9%E6%B3%95)
  * [C拡張ライブラリの基礎](#c%E6%8B%A1%E5%BC%B5%E3%83%A9%E3%82%A4%E3%83%96%E3%83%A9%E3%83%AA%E3%81%AE%E5%9F%BA%E7%A4%8E)
  * [C拡張ライブラリで SEGV したときのデバグ手法](#c%E6%8B%A1%E5%BC%B5%E3%83%A9%E3%82%A4%E3%83%96%E3%83%A9%E3%83%AA%E3%81%A7-segv-%E3%81%97%E3%81%9F%E3%81%A8%E3%81%8D%E3%81%AE%E3%83%87%E3%83%90%E3%82%B0%E6%89%8B%E6%B3%95)
* [Cool.io の基礎](#coolio-%E3%81%AE%E5%9F%BA%E7%A4%8E)
  * [イベント駆動とか言う前に GVL を知りやがれ](#%E3%82%A4%E3%83%99%E3%83%B3%E3%83%88%E9%A7%86%E5%8B%95%E3%81%A8%E3%81%8B%E8%A8%80%E3%81%86%E5%89%8D%E3%81%AB-gvl-%E3%82%92%E7%9F%A5%E3%82%8A%E3%82%84%E3%81%8C%E3%82%8C)
  * [libev のサンプル](#libev-%E3%81%AE%E3%82%B5%E3%83%B3%E3%83%97%E3%83%AB)
  * [Cool.io のサンプル](#coolio-%E3%81%AE%E3%82%B5%E3%83%B3%E3%83%97%E3%83%AB)
  * [ノンブロッキング待ち受け vs ブロッキング待ち受け](#%E3%83%8E%E3%83%B3%E3%83%96%E3%83%AD%E3%83%83%E3%82%AD%E3%83%B3%E3%82%B0%E5%BE%85%E3%81%A1%E5%8F%97%E3%81%91-vs-%E3%83%96%E3%83%AD%E3%83%83%E3%82%AD%E3%83%B3%E3%82%B0%E5%BE%85%E3%81%A1%E5%8F%97%E3%81%91)
  * [Fluentd でみる Cool.io の実例](#fluentd-%E3%81%A7%E3%81%BF%E3%82%8B-coolio-%E3%81%AE%E5%AE%9F%E4%BE%8B)
* [Cool.io の構造](#coolio-%E3%81%AE%E6%A7%8B%E9%80%A0)
  * [ファイル構造](#%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB%E6%A7%8B%E9%80%A0)
  * [クラス構造](#%E3%82%AF%E3%83%A9%E3%82%B9%E6%A7%8B%E9%80%A0)
  * [イベント発火の流れ](#%E3%82%A4%E3%83%99%E3%83%B3%E3%83%88%E7%99%BA%E7%81%AB%E3%81%AE%E6%B5%81%E3%82%8C)
  * [イベント受信の流れ](#%E3%82%A4%E3%83%99%E3%83%B3%E3%83%88%E5%8F%97%E4%BF%A1%E3%81%AE%E6%B5%81%E3%82%8C)
* [まとめ](#%E3%81%BE%E3%81%A8%E3%82%81)

# Ruby C拡張

C拡張API でC言語ライブラリをラップして ruby から呼べるようにできる。

## Ruby C拡張を含んだ gem の作り方

ゼロから作る場合は [BundlerでC拡張を含んだgemを公開する - Qiita [キータ]](http://qiita.com/gam0022/items/2ee82e84e5c9f608eb85) の記事が参考になった。

ポイントは、ext ディレクトリの下にCソースコードおよび extconf.rb を作成して、

```ruby
#extconf.rb
require "mkmf"
create_makefile("cool.io_ext")
```

gemspec に

```ruby
  spec.extensions    = %w[ext/cool.io/extconf.rb]
```

の行を足すことぐらい。
extconf.rb は Makefile を生成する ruby スクリプトで、
こうしておくと gem install の時に、C拡張を make でコンパイル、ビルドしてからインストールしてくれるようになる。


ディレクトリ構造概観

```
$ tree
.
├── cool.io.gemspec
├── ext
│   ├── cool.io
│   │   ├── cool.io.h
│   │   ├── cool.io_ext.c
│   │   ├── extconf.rb
```

## Ruby C拡張を含んだ gem のビルド方法

### gemspec の動作確認

gem install 時に期待通りにインストールされるか確認する。

gem をビルドする

```bash
$ gem build cool.io.gemspec
```

gem をインストールする

```bash
$ gem install cool.io-1.2.1.gem
```

rbenv で ruby 2.1.1 を使っている場合であれば、`~/.rbenv/versions/2.1.1/lib/ruby/gems/2.1.0/extensions`
あたりに .so ファイル含めてインストールされる

require できるか確認

```bash
$ irb
irb> require 'cool.io'
=> true
```

### Makefile の確認

extconf.rb を ruby スクリプトとして実行すると Makefile が生成されるので、それで確認できる

```bash
$ ruby ext/cool.io/extconf.rb
```

ビルドオプションを一時的にいじりたい場合は、Makefile を編集して make コマンドを打つと良い。

```bash
$ vi Makefile
$ make
```

たとえば、CFLAGS に --save-temps を足して make するとプリプロセッサを通した結果の .i ファイルなどが作られて捗る。

あとは、gdb で追う時のために最適化オプションを切って `-O0 -ggdb3` にしておくとか。

ビルドした .so ファイルはワーキングディレクトリの lib 以下に置かれて require できるようになる。

### rake compiler の利用

[rake-compiler](https://rubygems.org/gems/rake-compiler) という gem があって、
これを使うと `rake compile` でビルドできるようになる。
開発中に便利。こちらもワーキングディレクトリの lib 以下に .so ファイルが置かれる。

gemspec に development_dependency を足し、

```
s.add_development_dependency "rake-compiler"
```

Rakefile で require の行を足しておく

```ruby
require 'rake/extensiontask'
```

これで

```bash
$ rake compile
```

でC拡張をビルドしてくれるようになる。

```bash
$ rake clean
```

で make clean 相当もできる。


## C拡張ライブラリの基礎

こちらの ruby レポジトリに入っているドキュメントが１次情報源である [ruby/ruby/README.EXT.ja](https://github.com/ruby/ruby/blob/trunk/README.EXT.ja)


例えば、[cool.io/ext/loop.c](https://github.com/sonots/cool.io/blob/7453ed1ff1e20de4c99002e24407fcacdb0ad081/ext/cool.io/loop.c) をサンプルに、
C言語ライブラリをどのようにして Ruby のC拡張APIでラップしていくのか学んでみよう。

[ext/cool.io/loop.c#L44-L54](https://github.com/sonots/cool.io/blob/7453ed1ff1e20de4c99002e24407fcacdb0ad081/ext/cool.io/loop.c#L44-L54)

`rb_define_method` が ruby での `def` に相当していて、これでCレベルの関数をメソッドとして登録している。
`rb_define_private_method` は private メソッドになる。

`rb_define_method` の引数の数値は、ruby メソッドの引数の数を表していて、
この例では `initialize` メソッドは引数0個、`ev_loop_new` は引数0個、
`run_once` メソッドは可変長引数、`run_nonblock` メソッドは引数0個であることを表現している。

```c
void Init_coolio_loop()
{
  mCoolio = rb_define_module("Coolio");
  cCoolio_Loop = rb_define_class_under(mCoolio, "Loop", rb_cObject);
  rb_define_alloc_func(cCoolio_Loop, Coolio_Loop_allocate);

  rb_define_method(cCoolio_Loop, "initialize", Coolio_Loop_initialize, 0);
  rb_define_private_method(cCoolio_Loop, "ev_loop_new", Coolio_Loop_ev_loop_new, 1);
  rb_define_method(cCoolio_Loop, "run_once", Coolio_Loop_run_once, -1);
  rb_define_method(cCoolio_Loop, "run_nonblock", Coolio_Loop_run_nonblock, 0);
}
```

[ext/cool.io/loop.c#L56-L68](https://github.com/sonots/cool.io/blob/7453ed1ff1e20de4c99002e24407fcacdb0ad081/ext/cool.io/loop.c#L56-L68)`

ruby レベルでは `#new` がメモリ割当てと初期化を兼ねているが、Cレベルでは `allocate` と `initialize` にわかれている。

```c
static VALUE Coolio_Loop_allocate(VALUE klass)
{
  struct Coolio_Loop *loop = (struct Coolio_Loop *)xmalloc(sizeof(struct Coolio_Loop));

  loop->ev_loop = 0;
  ev_init(&loop->timer, Coolio_Loop_timeout_callback);
  loop->running = 0;
  loop->events_received = 0;
  loop->eventbuf_size = DEFAULT_EVENTBUF_SIZE;
  loop->eventbuf = (struct Coolio_Event *)xmalloc(sizeof(struct Coolio_Event) * DEFAULT_EVENTBUF_SIZE);

  return Data_Wrap_Struct(klass, Coolio_Loop_mark, Coolio_Loop_free, loop);
}
```

```c
static VALUE Coolio_Loop_initialize(VALUE self)
{
  Coolio_Loop_ev_loop_new(self, INT2NUM(0));
}
```

[ext/cool.io/loop.c#L90-L102](https://github.com/sonots/cool.io/blob/7453ed1ff1e20de4c99002e24407fcacdb0ad081/ext/cool.io/loop.c#L90-L102)

`#ev_loop_new` メソッドの引数の数は Init_coolio_loop で定義されていたように、引数１つであり、それが flags であることがわかる。

Rubyのデータは全て VALUE 型になっているので、Cレベルで利用するためには NUM2INT のような ruby のマクロを使って型変換する必要がある。

また、Ruby レベルでのオブジェクト (Data オブジェクト) からメンバを取り出す(ポインタを取り出す)には、`Data_Get_Struct` のようなマクロを利用する。

```c
/* Wrapper for populating a Coolio_Loop struct with a new event loop */
static VALUE Coolio_Loop_ev_loop_new(VALUE self, VALUE flags)
{
  struct Coolio_Loop *loop_data;
  Data_Get_Struct(self, struct Coolio_Loop, loop_data);

  if(loop_data->ev_loop)
    rb_raise(rb_eRuntimeError, "loop already initialized");

  loop_data->ev_loop = ev_loop_new(NUM2INT(flags));

  return Qnil;
}
```

[ext/cool.io/loop.c#L269-L274](https://github.com/sonots/cool.io/blob/7453ed1ff1e20de4c99002e24407fcacdb0ad081/ext/cool.io/loop.c#L269-L274)

`#run_nonblock` は引数0個のメソッドである。

```c
static VALUE Coolio_Loop_run_nonblock(VALUE self)
{
  struct Coolio_Loop *loop_data;
  VALUE nevents;

  Data_Get_Struct(self, struct Coolio_Loop, loop_data);
```

[ext/cool.io/loop.c#L186-L198](https://github.com/sonots/cool.io/blob/7453ed1ff1e20de4c99002e24407fcacdb0ad081/ext/cool.io/loop.c#L186-L198)

`#run_once` は rb_define_method で -1 と指定されていた可変長引数のメソッドで、
その場合の関数のシグネチャは次のようになる。

`rb_scan_args` を使って、argv から値を取り出すことになる。
`"01"` の10の位が必須引数の数、1の位がオプション引数の数を表現している。
今回の場合は、オプション引数が１つ、ということになる。

```c
static VALUE Coolio_Loop_run_once(int argc, VALUE *argv, VALUE self)
{
  VALUE timeout;
  VALUE nevents;
  struct Coolio_Loop *loop_data;

  rb_scan_args(argc, argv, "01", &timeout);

  if(timeout != Qnil && NUM2DBL(timeout) < 0) {
    rb_raise(rb_eArgError, "time interval must be positive");
  }

  Data_Get_Struct(self, struct Coolio_Loop, loop_data);
```

## C拡張ライブラリで SEGV したときのデバグ手法

Cを直接触っているので SEGV しやすい。SEGV したときの追い方を 解説というよりはメモにすぎないが、残しておく。

gcc のオプションを -O0 -ggdb3 にしたほうが SEGV 追いやすいとのことなので、Makefile 吐いていじる

```
$ ruby ext/cool.io/extconf.rb
$ vim Makefile
```

おまけ：Makefile いじって gcc のオプションに --savetemps を足すと、プリコンパイルを通した結果の .i ファイルが作られる。プリコンパイル結果を追いたい場合捗る。

コア吐かせる準備をしておく。

```bash
$ ulimit -n unlimited
```

プログラム実行して SEGV 発生させる。すると core.{pid} なファイルができているはず。

ruby のパスと core ファイルを指定して gdb を起動し、backtrace を表示する

```bash
$ gdb `rbenv which ruby` core.2101
(gdb) bt
#0  0x00d98410 in __kernel_vsyscall ()
#1  0x00138df0 in raise () from /lib/libc.so.6
#2  0x0013a701 in abort () from /lib/libc.so.6
#3  0x006a0877 in rb_bug (fmt=0x6d19b5 "Segmentation fault") at error.c:309
#4  0x005c0e86 in sigsegv (sig=11, info=0x959461c, ctx=0x959469c) at signal.c:672
#5  <signal handler called>
#6  ev_feed_event (loop=0x96a08e0, w=0xb7c0a844, revents=256) at ../../../../ext/cool.io/../libev/ev.c:1702
#7  0x004b0eeb in timers_reify (loop=0x96a08e0, flags=2) at ../../../../ext/cool.io/../libev/ev.c:1725
#8  ev_run (loop=0x96a08e0, flags=2) at ../../../../ext/cool.io/../libev/ev.c:3460
#9  0x004a9d3d in ev_loop (argc=1, argv=0xb7c8d01c, self=154926840) at ../../../../ext/cool.io/../libev/ev.h:826
#10 Coolio_Loop_run_once (argc=1, argv=0xb7c8d01c, self=154926840) at ../../../../ext/cool.io/loop.c:226
#11 0x006269b5 in call_cfunc_m1 (func=0x4a9c50 <Coolio_Loop_run_once>, recv=154926840, argc=1, argv=0xb7c8d01c) at vm_insnhelper.c:1325
#12 0x0062c22c in vm_call_cfunc_with_frame (th=0x96fca08, reg_cfp=0xb7d0cfb8, ci=<value optimized out>) at vm_insnhelper.c:1469
#13 0x0063d40d in vm_exec_core (th=0x96fca08, initial=<value optimized out>) at insns.def:1017
#14 0x00642b27 in vm_exec (th=0x96fca08) at vm.c:1201
#15 0x00643ecb in invoke_block_from_c (th=0x96fca08, proc=0x9645ec8, self=154913200, defined_class=147225780, argc=0, argv=0x93bfc6c, blockptr=0x0) at vm.c:648
#16 vm_invoke_proc (th=0x96fca08, proc=0x9645ec8, self=154913200, defined_class=147225780, argc=0, argv=0x93bfc6c, blockptr=0x0) at vm.c:696
#17 0x00657e33 in thread_start_func_2 (th=0x96fca08, stack_start=<value optimized out>) at thread.c:512
#18 0x0065804e in thread_start_func_1 (th_ptr=0x96fca08) at thread_pthread.c:765
#19 0x0083c852 in start_thread () from /lib/libpthread.so.0
#20 0x001e3a8e in clone () from /lib/libc.so.6
```

sigsegv がおきた次のフレームに移動して変数の中身などを表示する

```bash
(gdb) frame 6
#6  ev_feed_event (loop=0x96a08e0, w=0xb7c0a844, revents=256) at ../../../../ext/cool.io/../libev/ev.c:1702
1702        pendings [pri][w_->pending - 1].events |= revents;
(gdb) p w
$1 = (void *) 0xb7c0a844
(gdb) p pendings
$2 = {0x0, 0x0, 0x9687ed8, 0x0, 0x0}
(gdb) p pri
$3 = 6832926
(gdb) p w_->pending
Cannot access memory at address 0x4
```

0x4 は ruby の nil らしいので、そこがおかしい

# Cool.io の基礎

語弊を恐れずに言うと、Cool.io はマルチプラットフォーム対応イベント駆動ライブラリである libev の Ruby C拡張ラッパーと言える。
ので、cool.io を知るには libev を知る必要があるので触れる。

もっというと、各プラットフォームごとのイベント駆動APIを知る必要がある(select、poll、 Linux なら epoll、BSD なら kqueue とか)が、
今回はそこまで深くは追求しない。

## イベント駆動とか言う前に GVL を知りやがれ

libev に入る前に ruby の GVL を勉強しておこう。

cf. http://docs.ruby-lang.org/ja/1.9.3/class/Thread.html

> 現在の実装では Ruby VM は Giant VM lock (GVL) を有しており、
同時に実行される ネイティブスレッドは常にひとつです。
ただし、IO 関連のブロックする可能性があるシステムコールを行う場合には
GVL を解放します。その場合にはスレッドは同時に実行され得ます。
また拡張ライブラリから GVL を操作できるので、複数のスレッドを
同時に実行するような拡張ライブラリは作成可能です。

cf. http://d.hatena.ne.jp/kyab/20140215/1392485665

> CRubyのThreadクラスは1.8.xまではグリーンスレッドで実装されていました。で、1.9.0ではインタプリタからVMになって、ThreadクラスもNative Thread(OSが提供してるスレッド）で実装されるようになったというのは有名な話。

> ただNative Threadになったけど、マルチコアを（ほとんど）活かせない。なぜならVMが動く際、基本的に一つの巨大な排他をしてるから。この排他というかロックをGVL(Giant VM Lock)またはGIL(Giant Interpreter Lock)と呼ぶ。Native ThreadだからVMが検知できないタイミングでスレッドは切り替わろうとするけど、ロックされてるからまた元のスレッドに戻ってくる（正確にはロックを待っている方のスレッドはOSのスケジューラのキューに入らない）。

> GVLはブロッキングするようなC関数(典型的には各OSのSleep()。write()とかも？）呼び出し中には解放される。というかCで書かれたライブラリ中でそういう風に作ってる。なのでその際他のThreadクラスのインスタンスも動作できる。この時だけはマルチコアが同時に動く（可能性がある）。

> C拡張は基本的にGVLがかかった状態で呼び出される。これをしないと、C拡張を常に注意深くスレッドセーフにする必要があるし、そういうことをすべてのC拡張開発者に求めるのはキツかろう、という判断みたい。

> ただ、C拡張の中でブロッキング関数を呼び出したり、純粋な重い処理をやるときにGVLを解放してやることはできる。その為にrb_thread_blocking_region()というのが用意されている。よってC拡張の中でrb_thread_blocking_region()が使われている場合はマルチコアが同時に動く（可能性がある）。

> GVLはCRubyの実装を単純に保ち、かつシングルスレッド性能を落とさないためには今ん所まぁしょうがないよね～。forkでも使えば～。っていうところらしい。

cf. http://d.hatena.ne.jp/kyab/20140215/1392485665

> で、そこまで調べて気になったのが、じゃぁ例えば２つのスレッドが（内部でGVLを開放するようなメソッドを一切呼ばないで）ひたすらループしてたらどうなんの？ってこと。片方がGVLをロックしっぱなしだと、もう一方のスレッドは動作するチャンスがなくなっちゃうのでは？

> 結論からいうと、そんなケースでも時々スレッドは切り替わる。以下の凄まじく長いGVL(=GIL)に関する記事によると、タイマースレッドというのが裏で動いていて、時々フラグを立てるらしい。タイマースレッドはCレベルで書かれていて、もちろんGVLと関係なく動く。

要約すると ruby は、

* GVLがあるため結局マルチコアを使った並列処理ができない
* GVLを掴んでいるスレッドのみがCPUを使う事ができる

と言っている。また、GVL が解放されるタイミングについては

1. ブロックするようなC関数(write とか)を呼び出したとき
2. タイマースレッドにより定期的な解放

の２つがあると言っている。

図に書くと下図のようになる。Thread 1 でブロックするような処理を呼び出して待っている隙に、Thread 2 が実行されたり、
タイマースレッドが定期的にスレッドの切り替えを行う。

```
          blocking I/O    by timer thread
                 |           |
                 v           v
     +–––––––––––+–––––––––––+––––––––––+––––––––––+
     |           |           |          |          |
CPU1 | Thread 1  | Thread 2  | Thread 1 | Thread 2 |
     |           |           |          |          |
     +–––––––––––+–––––––––––+––––––––––+––––––––––+

     +–––––––––––––––––––––––––––––––––––––––––––––+
CPU2 |                                             |
     |                                             |
     +–––––––––––––––––––––––––––––––––––––––––––––+
```

また、`rb_thread_blocking_region()` を使ってGVLを明示的に解放することができる。
もとい、ruby 組み込みのブロックしそうなメソッドはそれを使って、GVLを解放するように実装してあるため、
GVLが解放されるのである。

なお、cool.io では libev にパッチをあてて、IOでブロックする場合に、GVLを解放するようにしているようだ。

参考文献

* [class Thread](http://docs.ruby-lang.org/ja/1.9.3/class/Thread.html)
* [覚え書き: SevenZipRuby 作成メモ 3 - マルチスレッド](http://masamitsu-murase.blogspot.com/2013/12/sevenzipruby-3.html)
* [CRubyのGVLとビジーループ - kyabの日記](http://d.hatena.ne.jp/kyab/20140215/1392485665)
* [Ruby Under a Microscope - Pat Shaughnessy](http://patshaughnessy.net/ruby-under-a-microscope)

## libev のサンプル

cf. http://pod.tst.eu/http://cvs.schmorp.de/libev/ev.pod

```c
#include <ev.h>
#include <stdio.h> // for puts

// every watcher type has its own typedef'd struct
// with the name ev_TYPE
ev_io stdin_watcher;
ev_timer timeout_watcher;

// all watcher callbacks have a similar signature
// this callback is called when data is readable on stdin
static void
stdin_cb (EV_P_ ev_io *w, int revents)
{
  puts ("stdin ready");
  // for one-shot events, one must manually stop the watcher
  // with its corresponding stop function.
  ev_io_stop (EV_A_ w);

  // this causes all nested ev_run's to stop iterating
  ev_break (EV_A_ EVBREAK_ALL);
}

// another callback, this time for a time-out
static void
timeout_cb (EV_P_ ev_timer *w, int revents)
{
  puts ("timeout");
  // this causes the innermost ev_run to stop iterating
  ev_break (EV_A_ EVBREAK_ONE);
}

int
main (void)
{
  // use the default event loop unless you have special needs
  struct ev_loop *loop = EV_DEFAULT;

  // initialise an io watcher, then start it
  // this one will watch for stdin to become readable
  ev_io_init (&stdin_watcher, stdin_cb, /*STDIN_FILENO*/ 0, EV_READ);
  ev_io_start (loop, &stdin_watcher);

  // initialise a timer watcher, then start it
  // simple non-repeating 5.5 second timeout
  ev_timer_init (&timeout_watcher, timeout_cb, 5.5, 0.);
  ev_timer_start (loop, &timeout_watcher);

  // now wait for events to arrive
  ev_run (loop, 0);

  // break was called, so exit
  return 0;
}
```

## Cool.io のサンプル

先ほどの libev のサンプルを Cool.io インターフェースで書くとこんなかんじか

```ruby
INTERVAL = 5,5

class MyIOWatcher < Cool.io::IOWatcher
  attr_accessor :readable
  def on_readable
    self.readable = true
  end
end

def run
  reactor = Cool.io::Loop.new

  sw = MyIOWatcher.new(STDIN)
  sw.attach(reactor)

  tw = Cool.io::TimerWatcher.new(INTERVAL, true)
  tw.on_timer do
    reactor.stop if sw.readable
  end
  tw.attach(reactor)

  reactor.run
  # 内部でやっているのは以下のような処理
  # while reactor.running and reactor.has_active_watchers?
  #   reactor.run_once # イベントが起こるまでブロック
  # end

  tw.detach
  sw.detach

  sw
end

run
```  

## ノンブロッキング待ち受け vs ブロッキング待ち受け

さきほどの例ではブロッキングで待ち受けしていたが、ノンブロッキングでの待ち受けを考えてみる。例えば以下のようなコード。

```ruby
while reactor.running and reactor.has_active_watchers?
  reactor.run_nonblock
end
```

これはよくない。なぜならビジーループとなるからだ。CPU100%の占有率となるからだ。
ではビジーループを避けるために sleep を入れてみたらどうだろうか？

```ruby
while reactor.running and reactor.has_active_watchers?
  reactor.run_nonblock
  sleep 0.5
end
```

これもよくない。なぜなら 0.5 秒眠っている間はイベントを受け付けることができず、
スループトッが落ちるからだ。

「イベント駆動」なので、ブロッキングでイベントを待ち受けて、
イベントが届いたら発火する、のがあるべき姿である。
発火したあとの処理はできればノンブロッキングですぐに返って来て、
ブロッキングのイベント待ち受け処理に入れると良い。

### ブロッキング待ち受けは終了させるのが難しい

しかし、ブロッキング待ち受けには終了させるのが難しいという問題がある。
次のコードを考える。これは Cool.io::Loop#run メソッドと同等である。

```ruby
while reactor.running and reactor.has_active_watchers?
  reactor.run_once
end
```

このループを終了させるには reactor.running フラグを false にしたのち、
ブロッキング処理を終了させるためになんらかのゴミイベントを送信しなければならない。
忘れずにイベントを送信しなければならない。

### タイムアウト付きブロッキング待ち受け

指定時間がすぎたら `#run_once` メソッドを抜ける timeout オプションの実装を考える。

```ruby
timeout = 0.5
while reactor.running and reactor.has_active_watchers?
  reactor.run_once(timeout)
end
```

これならば、ノンブロッキングの場合と異なり、
0.5 sec の間に 100 回イベントが来ても受け付けられスループットも落ちないし、
最後にイベントを投げなくてもループを終了することができる。

## Fluentd でみる Cool.io の実例

では、Fluentd の in_forward プラグインで実際どのように Cool.io を利用しているのか見てみよう。

[lib/fluent/plugin/in_forward.rb#L39-L53](https://github.com/fluent/fluentd/blob/ba5602f9e86540ebb3dcd2c8abc38ceb20ebf7cd/lib/fluent/plugin/in_forward.rb#L39-L53)

UDP で送られて来る heartbeat 待ち受け watcher および、
Coolio::TCPServer を使った TCP でのデータ送信待ち受け watcher を
Coolio::Loop に attach している。

どちらの watcher のいずれのイベントが来ても発火できるようにすべてのイベントを同時にブロッキング状態で待ち受ける。

メインスレッドでブロッキング待ち受けしてしまうと他の処理(他のプラグインの処理など)がなにもできなくなってしまうので、
`Thread.new(&method(:run))` して別スレッドで待ち受ける。

Ruby には GVL があるが、ブロッキングしているスレッドではGVLが解放されるので、
メインスレッドのほうで処理を続行できる。

```ruby
    def start
      @loop = Coolio::Loop.new

      @lsock = listen
      @loop.attach(@lsock)

      @usock = SocketUtil.create_udp_socket(@bind)
      @usock.bind(@bind, @port)
      @usock.fcntl(Fcntl::F_SETFL, Fcntl::O_NONBLOCK)
      @hbr = HeartbeatRequestHandler.new(@usock, method(:on_heartbeat_request))
      @loop.attach(@hbr)

      @thread = Thread.new(&method(:run))
      @cached_unpacker = $use_msgpack_5 ? nil : MessagePack::Unpacker.new
    end
```

```ruby
    def listen
      log.info "listening fluent socket on #{@bind}:#{@port}"
      s = Coolio::TCPServer.new(@bind, @port, Handler, @linger_timeout, log, method(:on_message))
      s.listen(@backlog) unless @backlog.nil?
      s
    end
```

```ruby
    def run
      @loop.run
    rescue => e
      log.error "unexpected error", :error => e, :error_class => e.class
      log.error_backtrace
    end
```

[lib/fluent/plugin/in_forward.rb#L55-L65](https://github.com/fluent/fluentd/blob/ba5602f9e86540ebb3dcd2c8abc38ceb20ebf7cd/lib/fluent/plugin/in_forward.rb#L55-L65)

全ての watchers を detach し、Coolio::Loop#stop (Coolio::Loop#run 内の running 変数を false にする)している。

Coolio::Loop#run 内のブロッキング待ち受けを抜けさせるために
`TCPSocket.open` で接続して on_connect イベントを送信することで
run メソッドを終了させている。

```ruby
    def shutdown
      @loop.watchers.each {|w| w.detach }
      @loop.stop
      @usock.close
      listen_address = (@bind == '0.0.0.0' ? '127.0.0.1' : @bind)
      # This line is for connecting listen socket to stop the event loop.
      # We should use more better approach, e.g. using pipe, fixing cool.io with timeout, etc.
      TCPSocket.open(listen_address, @port) {|sock| } # FIXME @thread.join blocks without this line
      @thread.join
      @lsock.close
    end
```

###  おまけ：抱えている問題

`TCPSocket.open` で接続することによって、`Coolio::Loop#run` を抜けさせているのだが、
この TCPSocket.open が失敗することがある。より詳細には Socket#pack_sockaddr_in で固まることがある。
このため、Fluentd が終了せずに固まることがあった。

* [Bug #9525: Stuck with Socket.pack_sockaddr_in - ruby-trunk - Ruby Issue Tracking System](https://bugs.ruby-lang.org/issues/9525)
* [Fluentd が終了しないことがある - togetter](http://togetter.com/li/572080)
* [Fluentd が終了しないことがある - その２ - togetter](http://togetter.com/li/622823)

そこで、`#run_once` に timeout オプションを実装したパッチがこちらである。=>
[[WIP] timeout option for Coolio::Loop#run_once by sonots · Pull Request #29 · tarcieri/cool.io](https://github.com/tarcieri/cool.io/pull/29)

# Cool.io の構造

## ファイル構造

```bash
.
├── CHANGES.md
├── Gemfile
├── Gemfile.lock
├── LICENSE
├── README.md
├── Rakefile
├── cool.io.gemspec
├── examples
│   ├── dslified_echo_client.rb
│   ├── dslified_echo_server.rb
│   ├── echo_client.rb
│   ├── echo_server.rb
│   ├── google.rb
│   └── httpclient.rb
├── ext # C拡張コードのディレクトリ
│   ├── cool.io
│   │   ├── cool.io.h # ヘッダ
│   │   ├── cool.io_ext.c # cool.io_ext.so を作る大元
│   │   ├── ev_wrap.h # win 用 define を追加した ev.h のラッパ. こっちを include する
│   │   ├── extconf.rb # configures Makefile
│   │   ├── iowatcher.c # Coolio::IOWatcher。Ruby IO の readability, writability イベント
│   │   ├── libev.c # ev_wrap.h を使うようにした ev.c のラッパ
│   │   ├── loop.c # Coolio::Loop。ループを司るクラス。WatcherをLoopにattachして使う
│   │   ├── stat_watcher.c # Coolio::StatWatcher。ファイルstatイベント(mtime, ino, etc)
│   │   ├── timer_watcher.c # Coolio::TimerWatcher。時間イベント
│   │   ├── utils.c # Coolio::Utils。ncpus (CPUの数), maxfds (最大 fd 数) の取得ができる
│   │   ├── watcher.c # Coolio::Watcher。Watcherクラス群のbase
│   │   └── watcher.h # Coolio::Watcher 用のヘッダ、ではなくて Watcher マクロ関数の実装
│   ├── http11_client # 非同期httpクライアント.
│   │   ├── LICENSE
│   │   ├── ext_help.h
│   │   ├── extconf.rb
│   │   ├── http11_client.c
│   │   ├── http11_parser.c
│   │   ├── http11_parser.h
│   │   └── http11_parser.rl
│   ├── iobuffer # non-blocking プログラム用 I/O バッファライブラリ
│   │   ├── extconf.rb
│   │   └── iobuffer.c
│   └── libev # libev に ruby_gil.patch をあてたもの
│       ├── Changes
│       ├── LICENSE
│       ├── README
│       ├── README.embed
│       ├── ev.c
│       ├── ev.h
│       ├── ev_epoll.c
│       ├── ev_kqueue.c
│       ├── ev_poll.c
│       ├── ev_port.c
│       ├── ev_select.c
│       ├── ev_vars.h
│       ├── ev_win32.c
│       ├── ev_wrap.h
│       ├── ruby_gil.patch
│       └── test_libev_win32.c
├── lib # rubyコードのディレクトリ
│   ├── cool.io
│   │   ├── async_watcher.rb # 別スレッドの Loop にパイプ経由でイベントを送る
│   │   ├── custom_require.rb # 同梱c拡張をrequireするためにごにょごにょしている
│   │   ├── dns_resolver.rb # non-blocking DNS resolver. これも IOWatcher で解決したら on_success が呼ばれる
│   │   ├── dsl.rb # http://coolio.github.io で解説しているDSL定義。Fluentdでは全く使っていない
│   │   ├── eventmachine.rb # coolio を使った eventmachine api の実装。実験的
│   │   ├── http_client.rb # non-blocking http client. http11_client を利用. 接続成功したら on_connect が呼ばれるとか
│   │   ├── io.rb # non-blocking IO. クライアントのベースクラス
│   │   ├── iowatcher.rb # Coolio::IOWatcher の一部 ruby 実装
│   │   ├── listener.rb # Coolio::Listener, TCPListener, UNIXListener
│   │   ├── loop.rb # Coolio::Loop の一部 ruby 実装
│   │   ├── meta.rb # watcher_delegate, event_callback クラスメソッド(マクロ)を生やす
│   │   ├── server.rb # Coolio::Server, TCPServer, UNIXServer. Listener の処理に加えて接続があったときに connection (socket) オブジェクトを作る
│   │   ├── socket.rb # Coolio::Socket, TCPSocket, UNIXSocket
│   │   ├── timer_watcher.rb # Coolio::TImerWatcher の一部 ruby 実装
│   │   └── version.rb
│   ├── cool.io.rb # require する入り口
│   ├── coolio.rb # require 'cool.io' してるだけ
└── spec
    ├── async_watcher_spec.rb
    ├── dns_spec.rb
    ├── spec_helper.rb
    ├── stat_watcher_spec.rb
    ├── tcp_server_spec.rb
    ├── timer_watcher_spec.rb
    ├── unix_listener_spec.rb
    └── unix_server_spec.rb
```

## クラス構造

ToDo: クラス図書く

```ruby
Loop
Watcher
StatWatcher < Watcher
IOWatcher (extend Meta) < Watcher
TimerWatcher (extend Meta) < Watcher
AsyncWatcher < IOWatcher < Watcher
DNSResolver < IOWatcher < Watcher
Listener < IOWatcher < Watcher
TCPListener < Listener < IOWatcher < Watcher
UNIXListenrer < Listener < IOWatcher < Watcher
Server < Listener < IOWatcher < Watcher
TCPServer < Server < Listener < IOWatcher < Watcher
UNIXServer < Server < Listener < IOWatcher < Watcher
IO (extend Meta)
Socket < IO
TCPSocket < Socket < IO
UNIXSocket < Socket < IO
HTTPClient < TCPSocket < Socket < IO
```

## イベント発火の流れ

[lib/cool.io/loop.rb#L91-L99](https://github.com/tarcieri/cool.io/blob/64657d653cafe2ed1c02f4ba38c347e6b9faf2b9/lib/cool.io/loop.rb#L91-L99)

アプリ側で Coolio::Loop.new して、run メソッドを呼ぶと、run_once のループに入る。

```ruby
    def run(timeout = nil)
      raise RuntimeError, "no watchers for this loop" if @watchers.empty?

      @running = true
      while @running and not @active_watchers.zero?
        run_once(timeout)
      end
      @running = false
    end
```

[ext/cool.io/loop.c#L183-L220](https://github.com/tarcieri/cool.io/blob/64657d653cafe2ed1c02f4ba38c347e6b9faf2b9/ext/cool.io/loop.c#L183-L220)

run_once の中では自分が実装した timeout オプションのためにちょっとごにょごにょやっているが、
基本的には RUN_LOOP メソッドを呼んでイベントが来るまでブロックし、
イベントが来たら Coolio_Loop_dispatch_events を呼んで dispatch している。

EVLOOP_ONESHOT なので１度イベントを受け取ったら loop は終了する。

```c
static VALUE Coolio_Loop_run_once(int argc, VALUE *argv, VALUE self)
{
  VALUE timeout;
  VALUE nevents;
  struct Coolio_Loop *loop_data;

  rb_scan_args(argc, argv, "01", &timeout);

  if (timeout != Qnil && NUM2DBL(timeout) < 0) {
    rb_raise(rb_eArgError, "time interval must be positive");
  }

  Data_Get_Struct(self, struct Coolio_Loop, loop_data);

  assert(loop_data->ev_loop && !loop_data->events_received);

  /* Implement the optional timeout (if any) as a ev_timer */
  /* Using the technique written at
     http://pod.tst.eu/http://cvs.schmorp.de/libev/ev.pod#code_ev_timer_code_relative_and_opti,
     the timer is not stopped/started everytime when timeout is specified, instead,
     the timer is stopped when timeout is not specified. */
  if (timeout != Qnil) {
    /* It seems libev is not a fan of timers being zero, so fudge a little */
    loop_data->timer.repeat = NUM2DBL(timeout) + 0.0001;
    ev_timer_again(loop_data->ev_loop, &loop_data->timer);
  } else {
    ev_timer_stop(loop_data->ev_loop, &loop_data->timer);
  }

  /* libev is patched to release the GIL when it makes its system call */
  RUN_LOOP(loop_data, EVLOOP_ONESHOT);

  Coolio_Loop_dispatch_events(loop_data);
  nevents = INT2NUM(loop_data->events_received);
  loop_data->events_received = 0;

  return nevents;
}
```

[ext/cool.io/loop.c#L30-L33](https://github.com/tarcieri/cool.io/blob/64657d653cafe2ed1c02f4ba38c347e6b9faf2b9/ext/cool.io/loop.c#L30-L33)

なお、RUN_LOOP はフラグの on/off をしているだけで、実質的には libev の ev_loop 関数

```c
#define RUN_LOOP(loop_data, options) \
  loop_data->running = 1; \
  ev_loop(loop_data->ev_loop, options); \
  loop_data->running = 0;
```

[ext/cool.io/loop.c#L246-L261](https://github.com/tarcieri/cool.io/blob/64657d653cafe2ed1c02f4ba38c347e6b9faf2b9/ext/cool.io/loop.c#L246-L261)

受け取ったイベントの数だけ、watcher_data (Coolio::Watcher オブジェクト) の dispatch_callback を呼ぶ

```c
static void Coolio_Loop_dispatch_events(struct Coolio_Loop *loop_data)
{
  int i;
  struct Coolio_Watcher *watcher_data;

  for(i = 0; i < loop_data->events_received; i++) {
    /* A watcher with pending events may have been detached from the loop
     * during the dispatch process.  If so, the watcher clears the pending
     * events, so skip over them */
    if(loop_data->eventbuf[i].watcher == Qnil)
      continue;

    Data_Get_Struct(loop_data->eventbuf[i].watcher, struct Coolio_Watcher, watcher_data);
    watcher_data->dispatch_callback(loop_data->eventbuf[i].watcher, loop_data->eventbuf[i].revents);
  }
}
```

[ext/cool.io/iowatcher.c#L181-L189](https://github.com/tarcieri/cool.io/blob/64657d653cafe2ed1c02f4ba38c347e6b9faf2b9/ext/cool.io/iowatcher.c#L181-L189)

Coolio::Watcher 自体は dispatch_callback を定義していないので、子クラスである IOWatcher クラスを見てみる。
IOWatcher は Coolio::TCPServer, Coolio::TCPSocket などの基底クラスとなるのでおそらく一番お世話になるもの。

EV_READ イベントならば、on_readable メソッドを、EV_WRITE ならば on_writable メソッドを呼んでいる。
ruby レベルでの継承をサポートできるように rb_funcall を使って呼び出している。

```
static void Coolio_IOWatcher_dispatch_callback(VALUE self, int revents)
{
  if(revents & EV_READ)
    rb_funcall(self, rb_intern("on_readable"), 0, 0);
  else if(revents & EV_WRITE)
    rb_funcall(self, rb_intern("on_writable"), 0, 0);
  else
    rb_raise(rb_eRuntimeError, "unknown revents value for ev_io: %d", revents);
}
```

## イベント受信の流れ

たとえば IOWatcher の場合に、どのイベントを受け取るのか

[ext/cool.io/iowatcher.c#L58-L94](https://github.com/tarcieri/cool.io/blob/64657d653cafe2ed1c02f4ba38c347e6b9faf2b9/ext/cool.io/iowatcher.c#L58-L94)

ここで libev で定義されている EV_READ | EV_WRITE イベントを受け取ったら、
Coolio_IOWatcher_libev_callback を呼ぶように、ev_io_init を呼び出して登録している。

登録された状態で、ev_loop が呼び出されるとイベントを受け取るまでブロックする仕組み。

```c
static VALUE Coolio_IOWatcher_initialize(int argc, VALUE *argv, VALUE self)
{
  VALUE io, flags;
  char *flags_str;
  int events;
  struct Coolio_Watcher *watcher_data;
#if HAVE_RB_IO_T
  rb_io_t *fptr;
#else
  OpenFile *fptr;
#endif

  rb_scan_args(argc, argv, "11", &io, &flags);

  if(flags != Qnil)
    flags_str = RSTRING_PTR(rb_String(flags));
  else
    flags_str = "r";

  if(!strcmp(flags_str, "r"))
    events = EV_READ;
  else if(!strcmp(flags_str, "w"))
    events = EV_WRITE;
  else if(!strcmp(flags_str, "rw"))
    events = EV_READ | EV_WRITE;
  else
    rb_raise(rb_eArgError, "invalid event type: '%s' (must be 'r', 'w', or 'rw')", flags_str);

  Data_Get_Struct(self, struct Coolio_Watcher, watcher_data);
  GetOpenFile(rb_convert_type(io, T_FILE, "IO", "to_io"), fptr);

  watcher_data->dispatch_callback = Coolio_IOWatcher_dispatch_callback;
  ev_io_init(&watcher_data->event_types.ev_io, Coolio_IOWatcher_libev_callback, FPTR_TO_FD(fptr), events);
  watcher_data->event_types.ev_io.data = (void *)self;

  return Qnil;
}
```

[ext/cool.io/iowatcher.c#L175-L178](https://github.com/tarcieri/cool.io/blob/64657d653cafe2ed1c02f4ba38c347e6b9faf2b9/ext/cool.io/iowatcher.c#L175-L178)

イベントを受け取ると、libev に登録した callback が呼ばれて、`Coolio::Loop#process_event` を呼び出す。

```c
static void Coolio_IOWatcher_libev_callback(struct ev_loop *ev_loop, struct ev_io *io, int revents)
{
  Coolio_Loop_process_event((VALUE)io->data, revents);
}
```

[ext/cool.io/loop.c#L102-L168](https://github.com/tarcieri/cool.io/blob/64657d653cafe2ed1c02f4ba38c347e6b9faf2b9/ext/cool.io/loop.c#L102-L168)

構造体に受け取ったイベントを登録し、events_received++ している。

```
void Coolio_Loop_process_event(VALUE watcher, int revents)
{
  struct Coolio_Loop *loop_data;
  struct Coolio_Watcher *watcher_data;

  /* The Global VM lock isn't held right now, but hopefully
   * we can still do this safely */
  Data_Get_Struct(watcher, struct Coolio_Watcher, watcher_data);
  Data_Get_Struct(watcher_data->loop, struct Coolio_Loop, loop_data);

  /*  Well, what better place to explain how this all works than
   *  where the most wonky and convoluted stuff is going on!
   *
   *  Our call path up to here looks a little something like:
   *
   *  -> release GVL -> event syscall -> libev callback
   *  (GVL = Global VM Lock)             ^^^ You are here
   *
   *  We released the GVL in the Coolio_Loop_run_once() function
   *  so other Ruby threads can run while we make a blocking
   *  system call (one of epoll, kqueue, port, poll, or select,
   *  depending on the platform).
   *
   *  More specifically, this is a libev callback abstraction
   *  called from a real libev callback in every watcher,
   *  hence this function not being static.  The real libev
   *  callbacks are event-specific and handled in a watcher.
   *
   *  For syscalls like epoll and kqueue, the kernel tells libev
   *  a pointer (to a structure with a pointer) to the watcher
   *  object.  No data structure lookups are required at all
   *  (beyond structs), it's smooth O(1) sailing the entire way.  
   *  Then libev calls out to the watcher's callback, which
   *  calls this function.
   *
   *  Now, you may be curious: if the watcher already knew what
   *  event fired, why the hell is it telling the loop?  Why
   *  doesn't it just rb_funcall() the appropriate callback?
   *
   *  Well, the problem is the Global VM Lock isn't held right
   *  now, so we can't rb_funcall() anything.  In order to get
   *  it back we have to:
   *
   *  stash event and return -> acquire GVL -> dispatch to Ruby
   *
   *  Which is kinda ugly and confusing, but still gives us
   *  an O(1) event loop whose heart is in the kernel itself. w00t!
   *
   *  So, stash the event in the loop's data struct.  When we return
   *  the ev_loop() call being made in the Coolio_Loop_run_once_blocking()
   *  function below will also return, at which point the GVL is
   *  reacquired and we can call out to Ruby */

  /* Grow the event buffer if it's too small */
  if(loop_data->events_received >= loop_data->eventbuf_size) {
    loop_data->eventbuf_size *= 2;
    loop_data->eventbuf = (struct Coolio_Event *)xrealloc(
        loop_data->eventbuf,
        sizeof(struct Coolio_Event) * loop_data->eventbuf_size
        );
  }

  loop_data->eventbuf[loop_data->events_received].watcher = watcher;
  loop_data->eventbuf[loop_data->events_received].revents = revents;

  loop_data->events_received++;
}
```

この callback が終わると、`#run_once` の RUN_LOOP を抜けて
`Coolio_Loop_dispatch_events` が呼び出され、events_received の数だけ、on_readable や on_writable が呼び出される、ということになる。

なので、libev の callback の仕組みをそのまま使って直接 on_readable などを呼び出しているわけではなく、
別途 cool.io 側の実装で on_readable などを呼び出している。

どうしてそうしているのかは、Coolio_Loop_process_event のコメントに書いてあるが、GVL がらみであるようだ。
IOWatcher を new したメインスレッドで GVL を保持したくない、といった所か？？

# まとめ

Cool.io に入門した。Cool.io に入門するための前準備として

* Ruby C拡張　
* GVL
* libev
* ノンブロッキング待ち受けとブロッキング待ち受け

の解説をした。その後、

* Cool.io の構造
* イベント発火の流れ
* イベント受信の流れ

を解説した。
