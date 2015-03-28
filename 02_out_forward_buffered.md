# Fluentd の out_forward と BufferedOutput

out_forward をサンプルにした BufferedOutput の解説

* [out_forward のパラメータたち](#out_forward-%E3%81%AE%E3%83%91%E3%83%A9%E3%83%A1%E3%83%BC%E3%82%BF%E3%81%9F%E3%81%A1)
* [out_foward の構造](#out_foward-%E3%81%AE%E6%A7%8B%E9%80%A0)
  * [ファイル構造](#%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB%E6%A7%8B%E9%80%A0)
  * [クラス構造](#%E3%82%AF%E3%83%A9%E3%82%B9%E6%A7%8B%E9%80%A0)
* [BufferedOutput のメソッド](#bufferedoutput-%E3%81%AE%E3%83%A1%E3%82%BD%E3%83%83%E3%83%89)
* [BasicBuffer のメソッド](#basicbuffer-%E3%81%AE%E3%83%A1%E3%82%BD%E3%83%83%E3%83%89)
* [スレッド状態](#%E3%82%B9%E3%83%AC%E3%83%83%E3%83%89%E7%8A%B6%E6%85%8B)
* [データ送信の流れ](#%E3%83%87%E3%83%BC%E3%82%BF%E9%80%81%E4%BF%A1%E3%81%AE%E6%B5%81%E3%82%8C)
  * [ObjectBufferedOutput#emit の流れ](#objectbufferedoutputemit-%E3%81%AE%E6%B5%81%E3%82%8C)
  * [ObjectBufferedOutput#try_flush の流れ(別スレッド)](#objectbufferedoutputtry_flush-%E3%81%AE%E6%B5%81%E3%82%8C%E5%88%A5%E3%82%B9%E3%83%AC%E3%83%83%E3%83%89)
  * [ForwardOutput#write_objects の流れ](#forwardoutputwrite_objects-%E3%81%AE%E6%B5%81%E3%82%8C)
* [shutdown 時の流れ](#shutdown-%E6%99%82%E3%81%AE%E6%B5%81%E3%82%8C)
* [おまけ: USR1 を受け取った時の flush の流れ](#%E3%81%8A%E3%81%BE%E3%81%91-usr1-%E3%82%92%E5%8F%97%E3%81%91%E5%8F%96%E3%81%A3%E3%81%9F%E6%99%82%E3%81%AE-flush-%E5%87%A6%E7%90%86%E3%81%AE%E6%B5%81%E3%82%8C)
* [まとめおよび補足](#%E3%81%BE%E3%81%A8%E3%82%81%E3%81%8A%E3%82%88%E3%81%B3%E8%A3%9C%E8%B6%B3)


## out_forward のパラメータたち

ドキュメントを見よう => [out_forward](http://docs.fluentd.org/ja/articles/out_forward)

[out_forward.rb#L30-L50](https://github.com/fluent/fluentd/blob/443650b6b275e7e9f3a5afca2315019556439c38/lib/fluent/plugin/out_forward.rb#L30-L50) この辺は前回カバーした

```ruby
class ForwardOutput < ObjectBufferedOutput
    config_param :send_timeout, :time, :default => 60
    config_param :heartbeat_type, :default => :udp
    config_param :heartbeat_interval, :time, :default => 1
    config_param :recover_wait, :time, :default => 10
    config_param :hard_timeout, :time, :default => 60
    config_param :expire_dns_cache, :time, :default => nil  # 0 means disable cache
    config_param :phi_threshold, :integer, :default => 16
    # backward compatibility
    config_param :port, :integer, :default => DEFAULT_LISTEN_PORT
    config_param :host, :string, :default => nil
    # <server></server> directive はこちら
    https://github.com/fluent/fluentd/blob/443650b6b275e7e9f3a5afca2315019556439c38/lib/fluent/plugin/out_forward.rb#L67-L88
end
```

[output.rb#L162](https://github.com/fluent/fluentd/blob/443650b6b275e7e9f3a5afca2315019556439c38/lib/fluent/output.rb#L162) 今回はこっち

try_flush_interval と queued_chunk_flush_interval は実はドキュメントに載っていない

```ruby
class BufferedOutput < Output
    config_param :buffer_type, :string, :default => 'memory'
    config_param :flush_interval, :time, :default => 60
    config_param :try_flush_interval, :float, :default => 1
    config_param :retry_limit, :integer, :default => 17
    config_param :retry_wait, :time, :default => 1.0
    config_param :max_retry_wait, :time, :default => nil
    config_param :num_threads, :integer, :default => 1
    config_param :queued_chunk_flush_interval, :time, :default => 1
end
```

[buffer.rb#L116-L132](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/buffer.rb#L116-L132)

```ruby
class BasicBuffer < Buffer
  # This configuration assumes plugins to send records to a remote server.
  # Local file based plugins which should provide more reliability and efficiency
  # should override buffer_chunk_limit with a larger size.
  config_param :buffer_chunk_limit, :size, :default => 8*1024*1024
  config_param :buffer_queue_limit, :integer, :default => 256
end
```

## out_foward の構造

### ファイル構造

* [lib/fluent/plugin/out_forward.rb](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/plugin/out_forward.rb) out_foward 本体
* [lib/fluent/output.rb](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/output.rb) Output プラグインの基底クラス。BufferedOutput など
* [lib/fluent/plugin/buf_memory.rb](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/plugin/buf_memory.rb) メモリ Buffer プラグイン
* [lib/fluent/plugin/buf_file.rb](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/plugin/buf_file.rb) ファイル Buffer プラグイン
* [lib/fluent/buffer.rb](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/buffer.rb) Buffer プラグインの基底クラス


### クラス構造

ToDo: クラス図書く

[lib/fluent/plugin/out_forward.rb](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/plugin/out_forward.rb)

```ruby
class ForwardOutput < ObjectBufferedOutput
```

[lib/fluent/output.rb#L417-L451](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/output.rb#L417-L451)

ObjectBufferedOutput は msgpack でメッセージのバッファリングを取り扱うクラス。
親の BufferedOutput 自体は json やその他のシリアライズも使えるように設計されている。
[Writing Buffered Output Plugins](http://docs.fluentd.org/articles/plugin-development#writing-buffered-output-plugins) 参照．

```ruby
class ObjectBufferedOutput < BufferedOutput
```

BufferedOutput を継承したクラスには他にも
[TimeSlicedOutput](http://docs.fluentd.org/articles/plugin-development#writing-time-sliced-output-plugins)
がある。こちらは queue 数や chunk サイズでのバッファコントロールに加えて、
時間ごとにバッファリングする機能を拡張したクラス。
out_file プラグインなどが利用している。今回は範囲外なので飛ばす。

[lib/fluent/output.rb#L162-L414](https://github.com/fluent/fluentd/blob/443650b6b275e7e9f3a5afca2315019556439c38/lib/fluent/output.rb#L162-L414)

BufferedOutput は Output クラスを継承しているだけでなく、バッファ Plugin を new して使っているのでそちらもカバーしていく。

```ruby
class BufferedOutput < Output
    config_param :buffer_type, :string, :default => 'memory'
    ...
    ...

    def configure(conf)
      super

      @buffer = Plugin.new_buffer(@buffer_type)
      @buffer.configure(conf)
end
```

[lib/fluent/output.rb#L59-L159](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/output.rb#L59-L159)

Output クラスは out_xxx プラグインを作るときに継承するクラス。実は config_param を生やすぐらいしかやってない。

```ruby
  class Output
    include Configurable # config_param メソッドを生やす
    include PluginId # conf[:id] を生やしているが、あんまり使っているの見た事ない
    include PluginLoggerMixin # v0.10.43 から導入された log メソッドを生やす
```

[lib/fluent/plugin/buf_memory.rb](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/plugin/buf_memory.rb)

buffer_type memory とすると使われる。

```ruby
class MemoryBufferChunk < BufferChunk
class MemoryBuffer < BasicBuffer
```

[lib/fluent/plugin/buf_file.rb](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/plugin/buf_file.rb)

buffer_type file とすると使われる。今回は飛ばす。

```ruby
class FileBufferChunk < BufferChunk
class FileBuffer < BasicBuffer
```

[lib/fluent/buffer.rb](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/buffer.rb)

BasicBuffer は Buffer プラグインの基底クラス。buffer_chunk_limit, buffer_queue_limit のオプションは BasicBuffer クラスで設定されている。

独自に Buffer プラグインを作って、buffer_type に独自プラグインを指定させることもできる。
ということになっているが、作ってる人みたことない。=> あ、[fluent-plugin-buffer-lightening](https://github.com/tagomoris/fluent-plugin-buffer-lightening) ありましたね。

```ruby
class Buffer
class BufferChunk
  include MonitorMixin
class BasicBuffer < Buffer
  include MonitorMixin
```

ちなみに [MonitorMixin](http://docs.ruby-lang.org/ja/2.0.0/class/MonitorMixin.html) は ruby で提供されているスレッドのモニター機能を提供するモジュール。
[ConditionVariable](http://docs.ruby-lang.org/ja/2.0.0/class/MonitorMixin=3a=3aConditionVariable.html) 参照。


## BufferedOutput のメソッド

BufferedOutput [output.rb#L162-L414](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/output.rb#L162-L414)

```ruby
  class BufferedOutput < Output
    def initialize # new
    def configure(conf) # オプション処理
    def start # スレッドスタート
    def shutdown # スレッドストップ
    def emit(tag, es, chain, key="") # Fluentd本体からデータを送られるエンドポイント
    def submit_flush #すぐ try_flush が呼ばれるように時間フラグを0にリセット
    def format_stream(tag, es) # データ stream のシリアライズ
    #def format(tag, time, record) # format_stream が呼び出す。単データのシリアライズをここで定義する
    #def write(chunk) # Buffer スレッドがこのメソッドを呼び出す
    def enqueue_buffer # Buffer キューにデータを追加
    def try_flush # Buffer キューからデータを取り出して flush する。flush 処理は別スレッドで実行される。@buffer.pop を呼び出す。
    def force_flush # USR1 シグナルを受けた時に強制 flush する
    def before_shutdown # shutdown 前処理
    def calc_retry_wait # retry 時間間隔の計算(exponential backoff)
    def write_abort # retry 限界を超えた場合に buffer を捨てる
    def flush_secondary(secondary) # secondary ノードへの flush
  end
```

OutputThread [output.rb#L89-L159](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/output.rb#L89-L159)

```ruby
  # num_threads で output プラグインの並列数を増やすために利用
  class OutputThread
    def initialize(output) # output プラグイン自身を渡す
    def configure(conf)
    def start # スレッドスタート
    def shutdown # スレッド終了
    def submit_flush #すぐ try_flush が呼ばれるように時間フラグを0にリセット
    private
    def run # スレッドループ。時間がきたら try_flush を呼ぶ
    def cond_wait(sec) # ConditionVariable を使って sec 秒待つ
  end
```

ObjectBufferedOutput [output.rb#L417-L451](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/output.rb#L417-L451)

```ruby
  class ObjectBufferedOutput < BufferedOutput
    def initialize
    def emit(tag, es, chain) # msgpack にシリアライズするようにオーバーライド
    def write(chunk) # msgpack にシリアライズするようにオーバーライド. write_objects(chunk.key, chunk) を呼び出す
  end
```

ForwardOutput

[out_forward.rb#L129](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/plugin/out_forward.rb#L129)

```ruby
  class FowardOutput < ObjectBufferedOutput
    def write_objects(tag, chunk) # データ送信する
  end
```

## BasicBuffer のメソッド

BasicBuffer [buffer.rb#L116-L305](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/buffer.rb#L116-L305)

```ruby
  class BasicBuffer < Buffer
    def initialize
    def enable_parallel(b=true)
    config_param :buffer_chunk_limit, :size, :default => 8*1024*1024
    config_param :buffer_queue_limit, :integer, :default => 256
    def configure(conf) # プラグインオプション設定
    def start # buffer plugin の start
    def shutdown # buffer plugin の shutdown
    def storable?(chunk, data) # buffer_chunk_limit を超えていないかどうか
    def emit(key, data, chain) # Fluentd本体からデータを送られるエンドポイント
    def keys # 取り扱っている key (基本的には tag) 一覧
    def queue_size # queue のサイズ
    def total_queued_chunk_size # キューに入っている全 chunk のサイズ
    #def new_chunk(key) # 扱う chunk オブジェクトの定義をすること
    #def resume # queue, map の初期化定義をすること
    #def enqueue(chunk) # chunk の enqueue を定義すること
    def push(key) # BufferedOutput が try_flush する時に呼び出されるようだ
    def pop(out) # queue から chunk を取り出して write_chunk する
    def write_chunk(chunk, out) # 中身は out.write(chunk)
    def clear! # queue をクリアする
  end
```

MemoryBuffer [buf_memory.rb#L67-L102](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/plugin/buf_memory.rb#L67-L102)

```ruby
  class MemoryBuffer < BasicBuffer
    def initialize
    def configure(conf)
    def before_shutdown(out)
    def new_chunk(key) # MemoryBufferChunk.new(key)
    def resume # @queue, @map = [], {}
    def enqueue(chunk) # 空
  end
```

## スレッド状態

メインスレッドと num_threads 数の OutputThread スレッド

```ruby
  class BufferedOutput < Output
    ...
    config_param :buffer_type, :string, :default => 'memory'
    ...
    config_param :num_threads, :integer, :default => 1

    def configure(conf)
      super

      @buffer = Plugin.new_buffer(@buffer_type)
      @buffer.configure(conf)
      ...
      @writers = (1..@num_threads).map {
        writer = OutputThread.new(self)
        writer.configure(conf)
        writer
      }
      ...
    end

    def start
      ...
      @writers.each {|writer| writer.start }
      ...
    end

    def shutdown
      @writers.each {|writer| writer.shutdown }
    end
```

OutputThread [output.rb#L89-L159](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/output.rb#L89-L159)

run ループを別スレッドで回して、定期的に output#try_flush を呼ぶ。
try_flush ではデータを queue から pop してデータ送信する。

つまり、データ送信を複数の別スレッドで並列に行っている。
特に、対向 Fluentd プロセスが複数ある場合に有効。
１つの場合でも受信側が nonblocking でデータをうまく捌いてくれるような場合には有効になりえるだろう。

```ruby
  class OutputThread
    def initialize(output)
      @output = output
      @finish = false
      @next_time = Engine.now + 1.0
    end

    def configure(conf)
    end

    def start
      @mutex = Mutex.new
      @cond = ConditionVariable.new
      @thread = Thread.new(&method(:run))
    end

    def shutdown
      @finish = true
      @mutex.synchronize {
        @cond.signal
      }
      Thread.pass
      @thread.join
    end

    def submit_flush
      @mutex.synchronize {
        @next_time = 0
        @cond.signal
      }
      Thread.pass
    end

    private
    def run
      @mutex.lock
      begin
        until @finish
          time = Engine.now

          if @next_time <= time
            @mutex.unlock
            begin
              @next_time = @output.try_flush
            ensure
              @mutex.lock
            end
            next_wait = @next_time - Engine.now
          else
            next_wait = @next_time - time
          end

          cond_wait(next_wait) if next_wait > 0
        end
      ensure
        @mutex.unlock
      end
    rescue
      $log.error "error on output thread", :error=>$!.to_s
      $log.error_backtrace
      raise
    ensure
      @mutex.synchronize {
        @output.before_shutdown
      }
    end

    def cond_wait(sec)
      @cond.wait(@mutex, sec)
    end
  end
```

#### 補足: ConditionVariable

* [class ConditionVariable](http://docs.ruby-lang.org/ja/2.1.0/class/ConditionVariable.html)
* [Rubyを用いたマルチスレッド対応Queueの作り方 - M-Tea](http://www.m-tea.info/2009/12/rubyqueue.html)

あたりが参考になるかも

## データ送信の流れ

シーケンス図 => あとで書くかもしれないが、今は [Fluentdがよくわからなかった話#29](http://www.slideshare.net/harukayon/fluentd-22317236#29) を見てもらえると。

* メインスレッド(データ入力があった)

    * ObjectBufferedOutput#emit

        * BasicBuffer#emit

* 別スレッド(時間がたった)

    * ObjectBufferedOutput#try_flush

        * BasicBuffer#push
        * BasicBuffer#pop

### ObjectBufferedOutput#emit の流れ

Fluentd プラグインの仕組みが #emit を呼び出してデータを送って来る。通常は chunk の key は tag だが、TimeSlicedOutput の場合は時間を format した文字列になり、key ごとに chunk が作られる。

[ObjectBufferedOutput#emit](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/output.rb#L417-L429)

```ruby
  class ObjectBufferedOutput < BufferedOutput
    ...

    def emit(tag, es, chain)
      @emit_count += 1
      data = es.to_msgpack_stream
      key = tag
      if @buffer.emit(key, data, chain)
        submit_flush
      end
    end
```

[BasicBuffer#emit](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/buffer.rb#L165-L214) @buffer.emit

*  key (通常は tag) 毎に chunk を処理
*  top chunk が buffer_chunk_limit を超えてなければ chunk に data を格納
*  超えていれば次の処理
  * 入らなかったデータを next chunk に格納
  * top chunk を queue に格納
  * 次の top chunk を next chunk で更新

```ruby
    def start
      @queue, @map = resume
      @queue.extend(MonitorMixin)
    end

    def emit(key, data, chain)
      key = key.to_s

      synchronize do
        top = (@map[key] ||= new_chunk(key))  # TODO generate unique chunk id

        # top chunk が buffer_chunk_limit を超えてなければ chunk に data を格納
        if storable?(top, data)
          chain.next
          top << data
          return false

          ## FIXME
          #elsif data.bytesize > @buffer_chunk_limit
          #  # TODO
          #  raise BufferChunkLimitError, "received data too large"

        elsif @queue.size >= @buffer_queue_limit
          raise BufferQueueLimitError, "queue size exceeds limit"
        end

        if data.bytesize > @buffer_chunk_limit
          $log.warn "Size of the emitted data exceeds buffer_chunk_limit."
          $log.warn "This may occur problems in the output plugins ``at this server.``"
          $log.warn "To avoid problems, set a smaller number to the buffer_chunk_limit"
          $log.warn "in the forward output ``at the log forwarding server.``"
        end

        nc = new_chunk(key) # TODO generate unique chunk id
        ok = false

        begin
          nc << data # 入らなかった分を next chunk に格納
          chain.next

          flush_trigger = false
          @queue.synchronize {
            enqueue(top)
            flush_trigger = @queue.empty?
            @queue << top # top chunk を queue に格納
            @map[key] = nc # 次の top chunk を更新
          }

          ok = true
          return flush_trigger
        ensure
          nc.purge unless ok
        end

      end  # synchronize
    end
```

[MemoryBufferChunk](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/plugin/buf_memory.rb#L19-L64) new_chunk の実体

@buffer << data で、文字列として追記しているだけ

```ruby
  class MemoryBufferChunk < BufferChunk
    def initialize(key, data='')
      @data = data
      @data.force_encoding('ASCII-8BIT')
      now = Time.now.utc
      u1 = ((now.to_i*1000*1000+now.usec) << 12 | rand(0xfff))
      @unique_id = [u1 >> 32, u1 & u1 & 0xffffffff, rand(0xffffffff), rand(0xffffffff)].pack('NNNN')
      super(key)
    end

    attr_reader :unique_id

    def <<(data)
      data.force_encoding('ASCII-8BIT')
      @data << data # 文字列として追記しているだけ
    end

    def size
      @data.bytesize
    end

    def close
    end

    def purge
    end

    def read
      @data
    end

    def open(&block)
      StringIO.open(@data, &block)
    end

    # optimize
    def write_to(io)
      io.write @data
    end

    # optimize
    def msgpack_each(&block)
      u = MessagePack::Unpacker.new
      u.feed_each(@data, &block)
    end
  end
```

[MemoryBuffer](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/plugin/buf_memory.rb#L67-L102) @queue の実体, enqueue(chunk) の定義

メモリバッファの場合、@queue はただの配列で、@map もただのハッシュ。enqueue(chunk) でもとくに何もやっていない。
[FileBuffer](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/plugin/buf_file.rb#L76) の場合は色々やっているが、今回はカバーしない。

```ruby
  class MemoryBuffer < BasicBuffer
    Plugin.register_buffer('memory', self)

    def initialize
      super
    end

    # Overwrite default BasicBuffer#buffer_queue_limit
    # to limit total memory usage upto 512MB.
    config_set_default :buffer_queue_limit, 64

    def configure(conf)
      super
    end

    def before_shutdown(out)
      synchronize do
        @map.each_key {|key|
          push(key)
        }
        while pop(out)
        end
      end
    end

    def new_chunk(key)
      MemoryBufferChunk.new(key)
    end

    def resume
      return [], {} # @queue が [] で @map が {}
    end

    def enqueue(chunk)
      # なにもやってない
    end
  end
```

### ObjectBufferedOutput#try_flush の流れ(別スレッド)

try_flush は OutputThread の run ループから try_flush_interval 毎に呼び出される
※ try_flush_interval がデフォルトの 1 の場合、flush_interval 0s にしても 1 秒毎にしか flush されない。


前半のみ読む [output.rb#L271-L325](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/output.rb#L271-L325)

主要な流れをざっくり

* buffer_chunk_limit に達していなくても flush_interval が来たら enqueue する
  * queue が空の場合のみ、enqueue 処理が走る
  * key (通常は tag) それぞれの chunk を一気に enqueue する
* queue から chunk を １つ pop して output#write (正確には、取り出して => write して => 成功したら削除)

```ruby
    def try_flush
      time = Engine.now

      empty = @buffer.queue_size == 0
      if empty && @next_flush_time < (now = Engine.now)
        @buffer.synchronize do
          if @next_flush_time < now
            enqueue_buffer # buffer_chunk_limit に達していなくても flush_interval が来たら enqueue する
            @next_flush_time = now + @flush_interval
            empty = @buffer.queue_size == 0
          end
        end
      end
      if empty
        return time + @try_flush_interval
      end

      begin
        retrying = !@error_history.empty?

        if retrying
          @error_history.synchronize do
            if retrying = !@error_history.empty?  # re-check in synchronize
              if @next_retry_time >= time
                # allow retrying for only one thread
                return time + @try_flush_interval
              end
              # assume next retry failes and
              # clear them if when it succeeds
              @last_retry_time = time
              @error_history << time
              @next_retry_time += calc_retry_wait
            end
          end
        end

        if @secondary && @error_history.size > @retry_limit
          has_next = flush_secondary(@secondary)
        else
          has_next = @buffer.pop(self) # queue から chunk を pop して output#write
        end

        # success
        if retrying
          @error_history.clear
          # Note: don't notify to other threads to prevent
          #       burst to recovered server
          $log.warn "retry succeeded.", :instance=>object_id
        end

        if has_next
          return Engine.now + @queued_chunk_flush_interval
        else
          return time + @try_flush_interval
        end
  ```


[BufferedOutput#enqueue_buffer](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/output.rb#L265-L269)

  ```ruby
    def enqueue_buffer
      @buffer.keys.each {|key|
        @buffer.push(key)
      }
    end
  ```

[BasicBuffer#push](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/buffer.rb#L244-L259)

flush_interval が来たので、buffer_chunk_limit に達していないが、top chunk を @queue に積み、削除している。
この enqueue (push) 処理は全 key (通常は tag) に対してまとめて行われる。
chunk は key (通常は tag) 毎に作られるので、tag が異なる多様なログを受け取っている場合、その数だけ @queue に一気に積み上げられる。

```ruby
    def push(key)
      synchronize do
        top = @map[key]
        if !top || top.empty?
          return false
        end

        @queue.synchronize do
          enqueue(top)
          @queue << top
          @map.delete(key)
        end

        return true
      end  # synchronize
    end
```

[BasicBuffer#pop](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/buffer.rb#L261-L293)

@queue から chunk を１つ取り出して、データ送信を行う。

pop というメソッド名だが、pop するだけではなく、write (送信)もしている。write 成功した場合のみ、取り除く。

```ruby
    def pop(out)
      chunk = nil
      @queue.synchronize do
        if @parallel_pop
          chunk = @queue.find {|c| c.try_mon_enter }
          return false unless chunk
        else
          chunk = @queue.first
          return false unless chunk
          return false unless chunk.try_mon_enter
        end
      end

      begin
        if !chunk.empty?
          write_chunk(chunk, out) # out.write(chunk) を呼び出す
        end

        empty = false
        @queue.synchronize do
          @queue.delete_if {|c|
            c.object_id == chunk.object_id
          }
          empty = @queue.empty?
        end

        chunk.purge

        return !empty
      ensure
        chunk.mon_exit
      end
    end
```

大事な補足：enqueue 処理は flush\_interval 毎に１回される。全 key (通常は tag) の chunk がまとめて enqueue される。
また pop (and 送信) 処理は try\_flush\_interval 毎に実行されるが、こちらは 1 chunk しか pop されない。
buffer_chunk_limit が小さい場合、十分なパフォーマンスが出ない可能性があるし、 enqueue された chunk を連続して一斉に pop したい場合、queued_chunk_flush_interval をごくごく小さな値に設定する必要がある。

考察：try_flush_interval 毎の pop (and 送信)処理で、複数 chunk 一気に送るようにしたらどうだろう。

### ForwardOutput#write_objects の流れ

BasicBuffer#try_flush
=> [BasicBuffer#write_chunk(chunk)](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/buffer.rb#L295-L297)
=> [BufferedOutput#write(chunk)](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/output.rb#L447-L450)
=> [ForwardOutput#write_objects](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/plugin/out_forward.rb#L129-L155)

[rebuild_weight_array](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/plugin/out_forward.rb#L159-L199) して
weight と heartbeat の結果を元にランダムに構築した @weight_array の順に[send_data](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/plugin/out_forward.rb#L220-L255) を試みる。

```ruby
    def write_objects(tag, chunk)
      return if chunk.empty?

      error = nil

      wlen = @weight_array.length
      wlen.times do
        @rr = (@rr + 1) % wlen
        node = @weight_array[@rr]

        if node.available?
          begin
            send_data(node, tag, chunk)
            return
          rescue
            # for load balancing during detecting crashed servers
            error = $!  # use the latest error
          end
        end
      end

      if error
        raise error # 全てのノードで送信失敗した場合。例外メッセージとして見れるのは最後の例外のみ
      else
        raise "no nodes are available"  # 全てのノードがそもそも available ではなかった場合
      end
    end
```

[ForwardOutput#send_data](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/plugin/out_forward.rb#L220-L255)

chunk を送るたびに socket を開いて、閉じている。SO_LINGER オプションで @send_timeout 時間を設定(時間がすぎたら RESET パケット)

```ruby
    def send_data(node, tag, chunk)
      sock = connect(node)
      begin
        opt = [1, @send_timeout.to_i].pack('I!I!')  # { int l_onoff; int l_linger; }
        sock.setsockopt(Socket::SOL_SOCKET, Socket::SO_LINGER, opt)

        opt = [@send_timeout.to_i, 0].pack('L!L!')  # struct timeval
        sock.setsockopt(Socket::SOL_SOCKET, Socket::SO_SNDTIMEO, opt)

        # beginArray(2)
        sock.write FORWARD_HEADER

        # writeRaw(tag)
        sock.write tag.to_msgpack  # tag

        # beginRaw(size)
        sz = chunk.size
        #if sz < 32
        #  # FixRaw
        #  sock.write [0xa0 | sz].pack('C')
        #elsif sz < 65536
        #  # raw 16
        #  sock.write [0xda, sz].pack('Cn')
        #else
        # raw 32
        sock.write [0xdb, sz].pack('CN')
        #end

        # writeRawBody(packed_es)
        chunk.write_to(sock)

        node.heartbeat(false)
      ensure
        sock.close
      end
    end
```

[BufferedOutput#try_flush 後半](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/output.rb#L389-L399)

write_objects が raise した場合、try_flush の rescue 節に入る。

```ruby
    ...
      has_next = @buffer.pop(self)
    ...
      rescue => e
        if retrying
          error_count = @error_history.size
        else
          # first error
          error_count = 0
          @error_history.synchronize do
            if @error_history.empty?
              @last_retry_time = time
              @error_history << time
              @next_retry_time = time + calc_retry_wait
            end
          end
        end

        if error_count < @retry_limit
          $log.warn "temporarily failed to flush the buffer.", :next_retry=>Time.at(@next_retry_time), :error_class=>e.class.to_s, :error=>e.to_s, :instance=>object_id
          $log.warn_backtrace e.backtrace

        elsif @secondary
          if error_count == @retry_limit
            $log.warn "failed to flush the buffer.", :error_class=>e.class.to_s, :error=>e.to_s, :instance=>object_id
            $log.warn "retry count exceededs limit. falling back to secondary output."
            $log.warn_backtrace e.backtrace
            retry  # retry immediately
          elsif error_count <= @retry_limit + @secondary_limit
            $log.warn "failed to flush the buffer, next retry will be with secondary output.", :next_retry=>Time.at(@next_retry_time), :error_class=>e.class.to_s, :error=>e.to_s, :instance=>object_id
            $log.warn_backtrace e.backtrace
          else
            $log.warn "failed to flush the buffer.", :error_class=>e.class, :error=>e.to_s, :instance=>object_id
            $log.warn "secondary retry count exceededs limit."
            $log.warn_backtrace e.backtrace
            write_abort
            @error_history.clear
          end

        else
          $log.warn "failed to flush the buffer.", :error_class=>e.class.to_s, :error=>e.to_s, :instance=>object_id
          $log.warn "retry count exceededs limit."
          $log.warn_backtrace e.backtrace
          write_abort
          @error_history.clear
        end

        return @next_retry_time
      end
```

[BufferedOutput#calc_retry_wait](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/output.rb#L389-L399)

calc_retry_wait で倍々で wait 時間をのばしている。いわゆる exponential backoff。

```ruby
    def calc_retry_wait
      # TODO retry pattern
      wait = if @error_history.size <= @retry_limit
               @retry_wait * (2 ** (@error_history.size-1))
             else
               # secondary retry
               @retry_wait * (2 ** (@error_history.size-2-@retry_limit))
             end
      retry_wait = wait + (rand * (wait / 4.0) - (wait / 8.0))
      @max_retry_wait ? [retry_wait, @max_retry_wait].min : retry_wait
    end
```

## shutdown 時の流れ

BufferedOutput の [#run ループが終わって](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/output.rb#L152)、#before_shutdown が呼ばれる

[lib/fluent/output.rb#L380-L387](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/output.rb#L380-L387)

```ruby
    def before_shutdown
      begin
        @buffer.before_shutdown(self)
      rescue
        $log.warn "before_shutdown failed", :error=>$!.to_s
        $log.warn_backtrace
      end
    end
```

[lib/fluent/plugin/buf_memory.rb#L82-L90](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/plugin/buf_memory.rb#L82-L90)

全部吐き出そうと試みる。※ 吐き出さずにすぐ終了するオプション欲しいな => [pull requested](https://github.com/fluent/fluentd/pull/286)

```ruby
    def before_shutdown(out)
      synchronize do
        @map.each_key {|key|
          push(key)
        }
        while pop(out)
        end
      end
    end
```

## おまけ: USR1 を受け取った時の flush 処理の流れ

USR1 シグナルを送ると、Buffer の内容を flush してくれることになっている。

[lib/fluent/supervisor.rb#L367-L382](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/supervisor.rb#L367-L382)

```ruby
      trap :USR1 do
        $log.debug "fluentd main process get SIGUSR1"
        $log.info "force flushing buffered events"
        @log.reopen!

        # Creating new thread due to mutex can't lock
        # in main thread during trap context
        Thread.new {
          begin
            Fluent::Engine.flush!
            $log.debug "flushing thread: flushed"
          rescue Exception => e
            $log.warn "flushing thread error: #{e}"
          end
        }.run
      end
```

[lib/fluent/engine.rb#L279-L295](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/engine.rb#L279-L295)

```ruby
    def flush!
      flush_recursive(@matches)
    end
    
    ...

    def flush_recursive(array)
      array.each {|m|
        begin
          if m.is_a?(Match)
            m = m.output
          end
          if m.is_a?(BufferedOutput)
            m.force_flush
          elsif m.is_a?(MultiOutput)
            flush_recursive(m.outputs)
          end
        rescue => e
          $log.debug "error while force flushing", :error_class=>e.class, :error=>e
          $log.debug_backtrace
        end
      }
    end
```

[lib/fluent/output.rb#L375-L378](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/output.rb#L375-L378)

```ruby
    def force_flush
      enqueue_buffer
      submit_flush
    end
```



## まとめおよび補足

* BufferedOutput, BasicBuffer 周りの処理を読んだ
* buffer_chunk_limit と buffer_queue_limit
* enqueue のタイミング２つ
  * メインスレッド(ObjectBufferedOutput#emit) で、chunk にデータを追加すると buffer_chunk_limit を超える場合
  * OutputThread (ObjectBufferedOutput#try_flush) で、flush_interval 毎
    * queue が空の場合のみ、enqueue 処理が走る
    * key (通常は tag) それぞれの chunk を一気に enqueue する
* dequeue(pop) のタイミング
  * queue に次の chunk がある場合、queued_chunk_flush_interval 毎
  * queue に次の chunk がない場合、try_flush_interval 毎
  * このタイミングで 1 chunk しか output#write されないので、パフォーマンスをあげるには chunk サイズを増やすか、queued_chunk_flush_interval および try_flush_interval を短くする必要がある。
* num_threads を増やすと OutputThread の数が増えるので output#write の IO 処理が並列化されて性能向上できる可能性
* ForwardOutput#send_data はデータを送るたびに connect(2) して close(2) しているので keepalive することで性能向上できる可能性

性能評価結果からの補足

* パフォーマンスをあげるためには buffer_chunk_limit を増やすと良い、と言ったが実際に buffer_chunk_limit を増やすと 8m ぐらいで詰まりやすくなり、性能劣化する。[out_forward って詰まると性能劣化する？](http://togetter.com/li/595607)
* なので、buffer_chunk_limit は 1m ぐらいに保ちつつ try_flush_interval を 0.1、queued_chunk_flush_interval を 0.01 など小さい値にしてじゃんじゃん吐き出すと良さそう

考察

* もしくは chunk のない out_foward (BufferedOutput)拡張を作って１行ごとに吐き出せばじゃんじゃん送信できそう。その場合 keepalive 必須にしないとパフォーマンス出ないと思う。
  * でも小さすぎると keepalive とはいえ、ネットワーク的に効率がよくない
* もしくは１度の dequeue(pop) のタイミングで、1 chunk ではなく 10 chunk まとめて pop できるようなオプションを付ければ chunk サイズを小さく保ちつつパフォーマンスをあげることができそう
  * [Fluentd の BufferedOutput に、１度に複数の chunk を吐き出すオプションを足すことについて議論](http://togetter.com/li/651190)
  * 結論としては、queued_chunk_flush_interval を 0.001 のように小さくして、try_flush_interval は 1 のまま、とかでよさそう。
* むしろ、queue もいらないのでは。in_tail => out_forward において、out_forward が送信失敗した場合は、ログを読み込まなければよい。fluent-agent-lite には queue はない。
  * in_tail と out_forward をがっちゃんこした in_tail_forward プラグインとかを作ればできるかも。
  * buffer_queue_limit を超えた場合、例外を発生させて、in プラグインにまで伝搬させれば、がっしゃんこプラグインを作られなくても扱えれるんじゃないか？
  
