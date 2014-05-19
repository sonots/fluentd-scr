# Fluentd の in_forward とプラグイン呼び出し

in_forward をサンプルにした Fluentd 本体のプラグイン呼び出しの流れ

* [in_forward プラグイン](#in_forward-%E3%83%97%E3%83%A9%E3%82%B0%E3%82%A4%E3%83%B3)
* [in_forward のパラメータたち](#in_forward-%E3%81%AE%E3%83%91%E3%83%A9%E3%83%A1%E3%83%BC%E3%82%BF%E3%81%9F%E3%81%A1)
* [in_forward の構造](#in_forward-%E3%81%AE%E6%A7%8B%E9%80%A0)
* [スレッド(Coolio)周りの処理](#%E3%82%B9%E3%83%AC%E3%83%83%E3%83%89coolio%E5%91%A8%E3%82%8A%E3%81%AE%E5%87%A6%E7%90%86)
* [heartbeat 受信の流れ](#heartbeat-%E5%8F%97%E4%BF%A1%E3%81%AE%E6%B5%81%E3%82%8C)
* [データ受信の流れ](#%E3%83%87%E3%83%BC%E3%82%BF%E5%8F%97%E4%BF%A1%E3%81%AE%E6%B5%81%E3%82%8C)
  * [Input プラグイン呼び出しの流れ](#input-%E3%83%97%E3%83%A9%E3%82%B0%E3%82%A4%E3%83%B3%E5%91%BC%E3%81%B3%E5%87%BA%E3%81%97%E3%81%AE%E6%B5%81%E3%82%8C)
  * [Output プラグイン呼び出しの流れ](#output-%E3%83%97%E3%83%A9%E3%82%B0%E3%82%A4%E3%83%B3%E5%91%BC%E3%81%B3%E5%87%BA%E3%81%97%E3%81%AE%E6%B5%81%E3%82%8C)
  * [Input から Output を通した流れまとめ](#input-%E3%81%8B%E3%82%89-output-%E3%82%92%E9%80%9A%E3%81%97%E3%81%9F%E6%B5%81%E3%82%8C%E3%81%BE%E3%81%A8%E3%82%81)
* [まとめおよび補足](#%E3%81%BE%E3%81%A8%E3%82%81%E3%81%8A%E3%82%88%E3%81%B3%E8%A3%9C%E8%B6%B3)

## 今回カバーするファイルたち

* [lib/fluent/plugin/in_forward.rb](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/plugin/in_forward.rb)
* [lib/fluent/input.rb](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/input.rb)
* [lib/fluent/supervisor.rb](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/supervisor.rb)
* [lib/fluent/engine.rb](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/engine.rb)

# in_forward プラグイン

## in_forward のパラメータたち

ドキュメントを見よう => [in_forward](http://docs.fluentd.org/articles/in_forward)

[in_forward.rb#L29-L33](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/plugin/in_forward.rb#L29-L33)

port 番号指定に使うぐらい。linger_timeout はデフォルトの 0 推奨。

```ruby
  class ForwardInput < Input
    config_param :port, :integer, :default => DEFAULT_LISTEN_PORT
    config_param :bind, :string, :default => '0.0.0.0'
    config_param :backlog, :integer, :default => nil
    # SO_LINGER 0 to send RST rather than FIN to avoid lots of connections sitting in TIME_WAIT at src
    config_param :linger_timeout, :integer, :default => 0
```

サンプル fluent.conf


```apache
<source>
  type forward
  port 12000
</source>
```

## in_forward の構造

それぞれ説明していく

```ruby
  class ForwardInput < Input
    def initialize # new
    def configure(conf) # パラメタ設定. start より前に呼ばれる
    def start # スレッド初期化. heartbeat とデータ受信
    def shutdown # スレッド終了
    def listen # listenインスタンス Coolio::TCPServer.new(@bind, @port, Handler) を返す
    def run # スレッド開始
    def on_message(msg) # Handler#on_read で Msgpack にデータが feed され、Msgpack によって呼ばれる
    class Handler < Coolio::Socket
      def initialize(io, linger_timeout, log, on_message)
      def on_connect # connect されたら呼ばれる。
      def on_read(data) # データを受け取ったら呼ばれる
      def on_read_json(data) # jsonデータを受け取ったら呼ばれる
      def on_read_msgpack(data) # msgpackデータを受け取ったら呼ばれる
      def on_close # close されたら呼ばれる
    end
    class HeartbeatRequestHandler < Coolio::IO
      def initialize(io, callback)
      def on_readable # heartbeat を受け取ったら呼ばれる. on_heartbeat_request を呼ぶ
    end
    def on_heartbeat_request(host, port, msg) # heartbeat への返答を返す
  end
end
```

## 前提知識: Input プラグインの構造

[Writing Input Plugins](http://docs.fluentd.org/articles/plugin-development#writing-input-plugins)

* configure 設定
* start スレッド初期化
* shutdown スレッドストップ
* スレッドから Fluent::Engine.emit(tag, time, record) を呼び出してデータを流す

[lib/fluent/input.rb](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/input.rb)

Input クラスを継承して、Input プラグインを作る。

実は何も頑張っていない orz

```ruby
module Fluent
  class Input
    include Configurable # config_param を使えるように
    include PluginId # config_param :id を定義しているが、そんなに使ってるの見た事ない
    include PluginLoggerMixin # v0.10.43 から入った log メソッドを使えるように

    def initialize
      super
    end

    def configure(conf)
      super
    end

    def start
    end

    def shutdown
    end
  end
end
```

他のプラグインを参考に、start 内部でスレッドを切り(もしくは Coolio を使って nonblocking にして)、
スレッドで入力待ち受けループを走らせて、
受け取ったデータは `Fluent::Engine.emit(tag, time, record)` を呼び出して流す、
という処理を書く。

Coolio まで含めてテンプレ化した親クラス１つ用意しても良さそうな気がするので、気が向いたらやる。かも。

## スレッド(Coolio)周りの処理

`start` と `shutdown`

`Coolio::TCPServer.new(@bind, @port, Handler)` と `HeartbeatRequestHandler` を Coolio::Loop に attach して、
Thread で Coolio::Loop#start

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

    def listen
      log.info "listening fluent socket on #{@bind}:#{@port}"
      s = Coolio::TCPServer.new(@bind, @port, Handler, @linger_timeout, log, method(:on_message))
      s.listen(@backlog) unless @backlog.nil?
      s
    end

    def run
      @loop.run
    rescue => e
      log.error "unexpected error", :error => e, :error_class => e.class
      log.error_backtrace
    end
```

shutdown 時には Handler (watchers) を detach して、stop

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

## heartbeat 受信の流れ

`HeartbeatRequestHandler` 周りの処理については、[01_out_forwrd_heartbeat.md](./01_out_forwrd_heartbeat.md) でカバーしたので飛ばす

## データ受信の流れ

[in_forward.rb#L149-L203](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/plugin/in_forward.rb#L149-L203)

`Coolio::TCPServer.new(@bind, @port, Handler)` 周りの処理。データを受け取ると Coolio::TCPServer に登録した Handler クラスの on_read が呼ばれる。
data は [ForwardOutput#send_data](https://github.com/fluent/fluentd/blob/4aff95cf7b6f11fef4b0b3ad2a9f3e1f6387032e/lib/fluent/plugin/out_forward.rb#L141) が
[tag, chunk] を送信しているので、ここで受け取るのも [tag, chunk] のセット１つのはず。

```ruby
    class Handler < Coolio::Socket
      def initialize(io, linger_timeout, log, on_message)
        super(io)
        if io.is_a?(TCPSocket)
          opt = [1, linger_timeout].pack('I!I!')  # { int l_onoff; int l_linger; }
          io.setsockopt(Socket::SOL_SOCKET, Socket::SO_LINGER, opt)
        end
        @on_message = on_message
        @log = log
        @log.trace {
          remote_port, remote_addr = *Socket.unpack_sockaddr_in(@_io.getpeername) rescue nil
          "accepted fluent socket from '#{remote_addr}:#{remote_port}': object_id=#{self.object_id}"
        }
      end

      def on_connect
      end

      def on_read(data)
        first = data[0]
        if first == '{' || first == '['
          m = method(:on_read_json)
          @y = Yajl::Parser.new
          @y.on_parse_complete = @on_message
        else
          m = method(:on_read_msgpack)
          @u = MessagePack::Unpacker.new
        end

        # define_method で on_read を on_read_json か on_read_msgpack にすげ換えているので、以降直接呼ばれる
        (class << self; self; end).module_eval do
          define_method(:on_read, m)
        end
        m.call(data)
      end
```

json なら on_read_json が呼ばれて、msgpack なら on_read_msgpack が呼ばれる。

Msgpack::Unpacker#feed_each については [https://github.com/msgpack/msgpack-ruby](https://github.com/msgpack/msgpack-ruby) の README
または [MessagePack for C/C++ /Ruby アップデート - Blog by Sadayuki Furuhashi](http://frsyuki.hatenablog.com/entry/20100302/p1) を読むと良い。
デシリアライザにネットワークから受け取ったデータを突っ込みながら、obj を取り出せるようになった時点で callback に登録してある on_message メソッドを event driven 的に呼ぶ。
逆に言えば、obj が取り出せるだけのデータを受け取っていなければ、on_message を呼ばずにすぐに終了する。

```ruby
      def on_read_json(data)
        @y << data
      rescue => e
        @log.error "forward error", :error => e, :error_class => e.class
        @log.error_backtrace
        close
      end

      def on_read_msgpack(data)
        @u.feed_each(data, &@on_message) # デシリアライザに feed しながら、obj 毎に on_message を呼ぶ
      rescue => e
        @log.error "forward error", :error => e, :error_class => e.class
        @log.error_backtrace
        close
      end

      def on_close
        @log.trace { "closed fluent socket object_id=#{self.object_id}" }
      end
    end
```

[in_forward.rb#L112-L147](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/plugin/in_forward.rb#L112-L147)

要素毎に on_message メソッドが呼ばれる。Engine.emit_stream(tag, es) または Engine.emit(tag, time, record) を呼び出す。
結果、tag にマッチする <match **> ディレクティブに指定された output プラグインが呼び出される。

output プラグイン呼び出しの詳細を次の章で詰めていく。

```ruby
def on_message(msg)
  if msg.nil?
    # for future TCP heartbeat_request
    return
  end

  # TODO format error
  tag = msg[0].to_s
  entries = msg[1]

  if entries.class == String
    # PackedForward
    es = MessagePackEventStream.new(entries, @cached_unpacker)
    Engine.emit_stream(tag, es)

  elsif entries.class == Array
    # Forward
    es = MultiEventStream.new
    entries.each {|e|
      record = e[1]
      next if record.nil?
      time = e[0].to_i
      time = (now ||= Engine.now) if time == 0
      es.add(time, record)
    }
    Engine.emit_stream(tag, es)

  else
    # Message
    record = msg[2]
    return if record.nil?
    time = msg[1]
    time = Engine.now if time == 0
    Engine.emit(tag, time, record)
  end
end
```

# Fluentd 本体のプラグイン呼び出し

## Input プラグイン呼び出しの流れ

### [Fluent::Supervisor#start](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/supervisor.rb#L79-L99)

Fluent::Supervisor は daemon 起動を司るクラス。lib/fluent/command/fluentd.rb で `Fluent::Supervisor.new(opts).start` のように呼ばれる

ここでプラグイン呼び出しに関して着目すべきは `#run_configure` と `#run_engine`

```ruby
    def start
      require 'fluent/load'
      @log.init

      dry_run if @dry_run
      start_daemonize if @daemonize
      install_supervisor_signal_handlers
      until @finished
        supervise do
          read_config
          change_privilege
          init_engine
          install_main_process_signal_handlers
          run_configure
          finish_daemonize if @daemonize
          run_engine
          exit 0
        end
        $log.error "fluentd main process died unexpectedly. restarting." unless @finished
      end
    end

    ...

    def run_configure
      Fluent::Engine.parse_config(@config_data, @config_fname, @config_basedir)
    end

    ...

    def run_engine
      Fluent::Engine.run
    end
```

### [Fluent::Engine#configure](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/engine.rb#L83-L124)

1. conf ファイルの source ディレクティブを読み込んで、in_xxx プラグインを new -> configure。
1. 同様に match ディレクティブを読み込んで、out_xxx プラグインを new -> configure。

```ruby
    def parse_config(io, fname, basepath=Dir.pwd)
      conf = if fname =~ /\.rb$/
               Config::DSL::Parser.parse(io, File.join(basepath, fname))
             else
               Config.parse(io, fname, basepath)
             end
      configure(conf)
      conf.check_not_fetched {|key,e|
        $log.warn "parameter '#{key}' in #{e.to_s.strip} is not used."
      }
    end

    def configure(conf)
      # plugins / configuration dumps
      Gem::Specification.find_all.select{|x| x.name =~ /^fluent(d|-(plugin|mixin)-.*)$/}.each do |spec|
        $log.info "gem '#{spec.name}' version '#{spec.version}'"
      end

      unless @suppress_config_dump
        $log.info "using configuration file: #{conf.to_s.rstrip}"
      end

      conf.elements.select {|e|
        e.name == 'source'
      }.each {|e|
        type = e['type']
        unless type
          raise ConfigError, "Missing 'type' parameter on <source> directive"
        end
        $log.info "adding source type=#{type.dump}"

        input = Plugin.new_input(type) ## 1. 各プラグインの #initialize
        input.configure(e) ## 2. 各プラグインの #configure

        @sources << input
      }

      conf.elements.select {|e|
        e.name == 'match'
      }.each {|e|
        type = e['type']
        pattern = e.arg
        unless type
          raise ConfigError, "Missing 'type' parameter on <match #{e.arg}> directive"
        end
        $log.info "adding match", :pattern=>pattern, :type=>type

        output = Plugin.new_output(type) ## 1. 各プラグインの #initialize
        output.configure(e) ## 2. 各プラグインの #configure

        match = Match.new(pattern, output)
        @matches << match
      }
    end
```

### [FluentEnging#run](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/engine.rb#L203-L236)

1. in, out に関わらず、プラグインの start メソッドを呼ぶ。各プラグインはそこで Thread を切るなどする。

    * input プラグインの場合は、起動したスレッドでデータを読む込むコードを書いたりする。
    * output プラグインの場合は、スレッドを起動せず、#emit が呼ばれたらなにか処理する、というコードを書いたりする。

2. Ctrl-C など stop 要請があったら shutdown メソッドを呼んで後処理。スレッドを閉じるなど。

```ruby
    def run
      begin
        start

        if match?($log.tag)
          $log.enable_event
          @log_emit_thread = Thread.new(&method(:log_event_loop))
        end

        unless @engine_stopped
          # for empty loop
          @default_loop = Coolio::Loop.default
          @default_loop.attach Coolio::TimerWatcher.new(1, true)
          #  attach async watch for thread pool
          @default_loop.run
        end

        if @engine_stopped and @default_loop
          @default_loop.stop
          @default_loop = nil
        end

      rescue => e
        $log.error "unexpected error", :error_class=>e.class, :error=>e
        $log.error_backtrace
      ensure
        $log.info "shutting down fluentd"
        shutdown
        if @log_emit_thread
          @log_event_loop_stop = true
          @log_emit_thread.join
        end
      end
    end

    ...

    def start
      @matches.each {|m|
        m.start
        @started << m
      }
      @sources.each {|s|
        s.start
        @started << s
      }
    end

    ...

    def shutdown
      @started.map {|s|
        Thread.new do
          begin
            s.shutdown
          rescue => e
            $log.warn "unexpected error while shutting down", :error_class=>e.class, :error=>e
            $log.warn_backtrace
          end
        end
      }.each {|t|
        t.join
      }
    end

```

## Output プラグイン呼び出しの流れ

Input プラグイン、または Filter プラグイン(Engine#emit を呼ぶ Output プラグインを便宜上 Filter プラグインと呼ぶことにする。たとえば [fluent-plugin-grep](https://github.com/sonots/fluent-plugin-grep) など)
が Engine#emit を呼び出すと、tag がマッチする <match **> 節に指定した Output プラグインの #emit メソッドが呼び出される。

### [Engine#emit_stream](https://github.com/fluent/fluentd/blob/9fea4bd69420daf86411937addc6000dfcc6043b/lib/fluent/engine.rb#L130-L163)

event_stream で match(tag) オブジェクト、つまり output プラグインのインスタンスの emit メソッドを呼び出している。


```ruby
    def emit(tag, time, record)
      unless record.nil?
        emit_stream tag, OneEventStream.new(time, record)
      end
    end

    def emit_array(tag, array)
      emit_stream tag, ArrayEventStream.new(array)
    end

    def emit_stream(tag, es)
      target = @match_cache[tag]
      unless target
        target = match(tag) || NoMatchMatch.new
        # this is not thread-safe but inconsistency doesn't
        # cause serious problems while locking causes.
        if @match_cache_keys.size >= MATCH_CACHE_SIZE
          @match_cache_keys.delete @match_cache_keys.shift
        end
        @match_cache[tag] = target
        @match_cache_keys << tag
      end
      target.emit(tag, es)
    rescue => e
      if @suppress_emit_error_log_interval == 0 || now > @next_emit_error_log_time
        $log.warn "emit transaction failed ", :error_class=>e.class, :error=>e
        $log.warn_backtrace
        # $log.debug "current next_emit_error_log_time: #{Time.at(@next_emit_error_log_time)}"
        @next_emit_error_log_time = Time.now.to_i + @suppress_emit_error_log_interval
        # $log.debug "next emit failure log suppressed"
        # $log.debug "next logged time is #{Time.at(@next_emit_error_log_time)}"
      end
      raise
    end
```

## Input から Output を通した流れまとめ

起動時

1. config に登場する全プラグインをインスタンス化
2. 全プラグイン#configure
3. 全プラグイン#start。ここで通常 Input プラグインはスレッド(Coolio)を切る。

Input

1. Input プラグインのイベントループが入力を受け付ける(on_read)
2. Input プラグインが、[tag, chunk] ペアの obj を取り出せるだけのデータを受け取っていたら、Engine.emit を呼び出す(on_message)
3. Fluentd 本体が tag に match する output プラグインを見つけて output#emit を呼び出す

Output

1. output#emit が呼ばれる
2. HTTP ポストするなど Output プラグインの処理を行う。
3. 別の Output プラグインを Engine#emit から呼び出している場合は、そちらの output プラグインの #emit 処理に入る。

流れの例としては、

* input#on_read => input#on_message

  * => output1#emit => output2#emit => output3#emit => HTTPポスト

のようになるだろう ここで注意すべき点は、on_messaage メソッドは、HTTPポスト出力まで終わって、
始めて終了するため、それらの処理がブロックしている間は、入力受付(on_read呼び出し)ができない、という点である。

そこで、[スレッドを切って処理する BufferedOutput を継承して使おう](http://chopl.in/blog/2014/01/19/user_bufferedoutput_for_blocking_plugin_instead_of_output.html)
のように BufferedOuput プラグインにして #emit 呼び出しでは enqueue するだけにして、
別スレッドでデータ処理や HTTP ポストのようなブロックする処理をさせる方法が取られる。

しかし、今度は捌ききれない量のデータをとりあえず enqueue するようになってしまうため
メモリ上にデータが溜まりまくり、buffer_queue_limit 溢れ、となる問題が発生しうる。
一長一短である。


# まとめおよび補足

in_forward プラグインをサンプルに、Fluentd 本体のプラグイン呼び出しを解説した。

ブロックしている時間を可視化するために、[fluent-plugin-elapsed-time](https://github.com/sonots/fluent-plugin-elapsed-time) というプラグインを作ってある。
上の例で言うと、output1#emit が呼び出されてから、HTTPポストが終わるまでの時間を計測できる。

より厳密に input#on_read, input#on_message でブロックされている時間を可視化するには in_forward.rb を直接いじる必要がある。
その場合は[こちらのパッチ](https://github.com/sonots/fluentd/commit/364d4e0ec320ee4a44880ee6651a983103af6c71) が利用できる。
=> 黒魔術を使って外から hook する方法を思いついたので作って gem 化してみた [fluent-plugin-measure_time](https://github.com/sonots/fluent-plugin-measure_time)
