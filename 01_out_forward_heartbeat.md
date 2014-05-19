# Fluentd の out_forward と heartbeat

フォワードを制するものは Fluentd を制す

## out_forward.rb の構造

### サンプル設定

```apache
<match **>
  type forward
  # try_flush_interval 1
  flush_interval 0s # 60s
  buffer_queue_limit 312
  buffer_chunk_limit 10m
  num_threads 2
  retry_wait 1
  retry_limit 17
  max_retry_wait 4 # 131072
  # send_timeout 60s
  heartbeat_type udp
  heartbeat_interval 10s # 1s
  phi_threshold 65535 # 35 # 8
  recover_wait 50s # 10s
  hard_timeout 23s # 60s
  <server>
    host host3
    port 12000
    weight 60
  </server>
  <server>
    host host2
    port 12001
    weight 60
  </server>
  <server>
    host host1
    port 13000
    weight 60
    standby true
  </server>
</match>
```

ついでの in_forward サンプル

```apache
<source>
  type forward
  port 12000
</source>
```
### パラメータたち

まずドキュメントを見よう => [out_forward](http://docs.fluentd.org/ja/articles/out_forward)

[out_forward.rb#L30-L50](https://github.com/fluent/fluentd/blob/443650b6b275e7e9f3a5afca2315019556439c38/lib/fluent/plugin/out_forward.rb#L30-L50)
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

[output.rb#L162](https://github.com/fluent/fluentd/blob/443650b6b275e7e9f3a5afca2315019556439c38/lib/fluent/output.rb#L162) こっちは次回

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

### クラスたち

```ruby
class ForwardOutput < ObjectBufferedOutput
   # heartbeat を @heartbeat_interval ごとに打つ Timer スレッド
   class HeartbeatRequestTimer < Coolio::TimerWatcher
   # heartbeat を受け取る. event driven で io 多重化しているらしいが、Coolio 詳しくない
   class HeartbeatHandler < Coolio::IO
   # <server> ディレクティブで設定したノードを表現するクラス
   class Node
   # heartbeat の結果を元に接続が失敗しているかを判定するクラス。端的には phi の計算
   class FailureDetector
end
```

### メソッドたち

```ruby
class ForwardOutput < ObjectBufferedOutput
  def initialize # new
  def configure(conf) # 設定. start より先に呼ばれる
  def start # engine の run ループ先頭で呼ばれる
  def shutdown # engine の run ループ終了で呼ばれる
  def run # engine の run ループで呼ばれる
  def write_objects # ObjectBufferedOutput のスレッドでデータを送るときに呼ばれる。node を選択してデータを投げる
  private
  def rebuild_weight_array # weight と heartbeat 結果を元に weight_array を作る
  def send_heartbeat_tcp # heartbeat_type :tcp 用 heartbeat
  def send_data # データをなげる
  def connect # 接続

  # heartbeat を @heartbeat_interval ごとにうつ Timer スレッド
  class HeartbeatRequestTimer < Coolio::TimerWatcher
    def initailize
    def on_timer
  end
  def on_timer # 時間がきたら heartbeat を投げる

  # heartbeat を受け取る. event driven で io 多重化しているらしいが、Coolio 詳しくない
  class HeartbeatHandler < Coolio::IO
    def initialize
    def on_readable # upd heartbeat の返答を受け取る
  end
  def on_heartbeat(sockaddr, msg) # heartbeat を元に rebuild_weight_array

  # <server> ディレクティブで設定したノードを表現するクラス
  class Node
    def available?
    def standby?
    def resolved_host
    def resolve_dns!
    def tick # on_timer から呼ばれる。hard_timeout や phi_threshold の判定
    def heartbeat(detect=true) # heartbeat が成功したら呼ばれる
    def to_msgpack(out = '')
  end

  # heartbeat の結果を元に接続が失敗しているかを判定するクラス。端的には phi の計算
  class FailureDetector
    def initialize(heartbeat_interval, hard_timeout, init_last)
    def hard_timeout?(now) # hard_timeout 判定
    def add(now) # 成功時に add される
    def phi(now) # phi を計算する
    def sample_size
    def clear
  end
end
```

## heartbeat の流れ

ToDo: in_forward も絡めたシーケンス図をここに貼る


[out_forward.rb#L108](https://github.com/fluent/fluentd/blob/443650b6b275e7e9f3a5afca2315019556439c38/lib/fluent/plugin/out_forward.rb#L108) at start

heatbeat_interval 1s ごとに起動するコールバックメソッド on_timer を登録.

```ruby
      if @heartbeat_type == :udp
        # assuming all hosts use udp
        @usock = SocketUtil.create_udp_socket(@nodes.first.host)
        @usock.fcntl(Fcntl::F_SETFL, Fcntl::O_NONBLOCK)
        @hb = HeartbeatHandler.new(@usock, method(:on_heartbeat))
        @loop.attach(@hb)
      end

      @timer = HeartbeatRequestTimer.new(@heartbeat_interval, method(:on_timer))
      @loop.attach(@timer)
```

"\0" を全 node になげる. tcp の場合は connect(2) してすぐ close(2) [out_forward.rb#L205-L218](https://github.com/fluent/fluentd/blob/443650b6b275e7e9f3a5afca2315019556439c38/lib/fluent/plugin/out_forward.rb#L205-L218)

```ruby
    def on_timer
      return if @finished
      @nodes.each {|n|
        if n.tick
          rebuild_weight_array
        end
        begin
          #log.trace "sending heartbeat #{n.host}:#{n.port} on #{@heartbeat_type}"
          if @heartbeat_type == :tcp
            send_heartbeat_tcp(n)
          else
            @usock.send "\0", 0, Socket.pack_sockaddr_in(n.port, n.resolved_host)
          end
        rescue Errno::EAGAIN, Errno::EWOULDBLOCK, Errno::EINTR
          # TODO log
          log.debug "failed to send heartbeat packet to #{n.host}:#{n.port}", :error=>$!.to_s
        end
      }
    end
```

Node#tick。true を返す => heartbeat 失敗の意。イミフ

```ruby
    def tick
        now = Time.now.to_f
        if !@available
          if @failure.hard_timeout?(now)
            @failure.clear
          end
          return nil
        end

        if @failure.hard_timeout?(now)
          @log.warn "detached forwarding server '#{@name}'", :host=>@host, :port=>@port, :hard_timeout=>true
          @available = false
          @resolved_host = nil  # expire cached host
          @failure.clear
          return true
        end

        phi = @failure.phi(now)
        #$log.trace "phi '#{@name}'", :host=>@host, :port=>@port, :phi=>phi
        if phi > @phi_threshold
          @log.warn "detached forwarding server '#{@name}'", :host=>@host, :port=>@port, :phi=>phi
          @available = false
          @resolved_host = nil  # expire cached host
          @failure.clear
          return true
        else
          return false
        end
      end
```

rebuld_weight_array

```
<match foobar>
  type forward
  <server>
    host xxxx
    port xxxx
    weight 60
  </server>
  <server>
    host xxxx
    port xxxx
    weight 40
  </server>
</match>
```

weight と node が生きているかどうかの情報を元に weight_array を構築。


```ruby
    def rebuild_weight_array
      standby_nodes, regular_nodes = @nodes.partition {|n|
        n.standby?
      }

      lost_weight = 0
      regular_nodes.each {|n|
        unless n.available?
          lost_weight += n.weight
        end
      }
      log.debug "rebuilding weight array", :lost_weight=>lost_weight

      if lost_weight > 0
        standby_nodes.each {|n|
          if n.available?
            regular_nodes << n
            log.warn "using standby node #{n.host}:#{n.port}", :weight=>n.weight
            lost_weight -= n.weight
            break if lost_weight <= 0
          end
        }
      end

      weight_array = []
      gcd = regular_nodes.map {|n| n.weight }.inject(0) {|r,w| r.gcd(w) }
      regular_nodes.each {|n|
        (n.weight / gcd).times {
          weight_array << n
        }
      }

      # for load balancing during detecting crashed servers
      coe = (regular_nodes.size * 6) / weight_array.size
      weight_array *= coe if coe > 1

      r = Random.new(@rand_seed)
      weight_array.sort_by! { r.rand }

      @weight_array = weight_array
    end
```

udp heartbeat を送ると、in_forward で受け取って、udp で返す

[in_forward.rb#L205-L232](https://github.com/fluent/fluentd/blob/443650b6b275e7e9f3a5afca2315019556439c38/lib/fluent/plugin/in_forward.rb#L205-L232)

```ruby
    class HeartbeatRequestHandler < Coolio::IO
      def initialize(io, callback)
        super(io)
        @io = io
        @callback = callback
      end

      def on_readable
        begin
          msg, addr = @io.recvfrom(1024)
        rescue Errno::EAGAIN, Errno::EWOULDBLOCK, Errno::EINTR
          return
        end
        host = addr[3]
        port = addr[1]
        @callback.call(host, port, msg)
      rescue
        # TODO log?
      end
    end

    def on_heartbeat_request(host, port, msg)
      #log.trace "heartbeat request from #{host}:#{port}"
      begin
        @usock.send "\0", 0, host, port
      rescue Errno::EAGAIN, Errno::EWOULDBLOCK, Errno::EINTR
      end
    end
```

upd heartbeat が帰ってきたら、rebuild_weight_array する

[out_forward.rb#L295-L325](https://github.com/fluent/fluentd/blob/443650b6b275e7e9f3a5afca2315019556439c38/lib/fluent/plugin/out_forward.rb#L295-L325)

```ruby
    class HeartbeatHandler < Coolio::IO
      def initialize(io, callback)
        super(io)
        @io = io
        @callback = callback
      end

      def on_readable
        begin
          msg, addr = @io.recvfrom(1024)
        rescue Errno::EAGAIN, Errno::EWOULDBLOCK, Errno::EINTR
          return
        end
        host = addr[3]
        port = addr[1]
        sockaddr = Socket.pack_sockaddr_in(port, host)
        @callback.call(sockaddr, msg)
      rescue
        # TODO log?
      end
    end

    def on_heartbeat(sockaddr, msg)
      port, host = Socket.unpack_sockaddr_in(sockaddr)
      if node = @nodes.find {|n| n.sockaddr == sockaddr }
        #log.trace "heartbeat from '#{node.name}'", :host=>node.host, :port=>node.port
        if node.heartbeat
          rebuild_weight_array
        end
      end
    end
```

Node#heartbeat.[out_forward.rb#L295-L325](https://github.com/fluent/fluentd/blob/443650b6b275e7e9f3a5afca2315019556439c38/lib/fluent/plugin/out_forward.rb#L295-L325)

引数は見つかった場合 true なのかというと、そういうことはなくて、
heartbeat で反応があった場合 true で、send_data が成功した場合 false。というだけ。
`@available` じゃない、かつ `@failure.sample_size > @recover_sample_size` の場合、recover させる。


```ruby
      def heartbeat(detect=true)
        now = Time.now.to_f
        @failure.add(now)
        #@log.trace "heartbeat from '#{@name}'", :host=>@host, :port=>@port, :available=>@available, :sample_size=>@failure.sample_size
        if detect && !@available && @failure.sample_size > @recover_sample_size
          @available = true
          @log.warn "recovered forwarding server '#{@name}'", :host=>@host, :port=>@port
          return true
        else
          return nil
        end
      end
```

`@failure.sample_size` は heartbeat 失敗した数かと思いきや、そんなことはなくて、`#heartbeat` が呼ばれた回数。どちらかというと heartbeat 成功の数。
deteched forwarding されると 0 にクリアされる。

recover_sample_size は #configure の中で計算されていて、`@recover_wait` はデフォルト 10s で、`@heartbeat_interval` はデフォルト 1s。
`recover_sample_size = @recover_wait / @heartbeat_interval`

つまり、`@failure.sample_size > @recover_sample_size` は detached forwarding されてから、
何回 heartbeat 成功すれば recoverd forwarding 判定するか、を定義している。
あくまで heartbeat 成功であって、recovered forwarding したノードにデータを送っていることを意味してはいない。
`heartbeat_interval` を変えた場合は、`recover_wait` も変更すべきであろう。


Failure Detector. [out_forward.rb#L435-L496](https://github.com/fluent/fluentd/blob/443650b6b275e7e9f3a5afca2315019556439c38/lib/fluent/plugin/out_forward.rb#L435-L496)

```ruby
    class FailureDetector
      PHI_FACTOR = 1.0 / Math.log(10.0)
      SAMPLE_SIZE = 1000

      def initialize(heartbeat_interval, hard_timeout, init_last)
        @heartbeat_interval = heartbeat_interval
        @last = init_last
        @hard_timeout = hard_timeout

        # microsec
        @init_gap = (heartbeat_interval * 1e6).to_i
        @window = [@init_gap]
      end

      def hard_timeout?(now)
        now - @last > @hard_timeout
      end

      def add(now)
        if @window.empty?
          @window << @init_gap
          @last = now
        else
          gap = now - @last
          @window << (gap * 1e6).to_i
          @window.shift if @window.length > SAMPLE_SIZE
          @last = now
        end
      end

      def phi(now)
        size = @window.size
        return 0.0 if size == 0

        # Calculate weighted moving average
        mean_usec = 0
        fact = 0
        @window.each_with_index {|gap,i|
          mean_usec += gap * (1+i)
          fact += (1+i)
        }
        mean_usec = mean_usec / fact

        # Normalize arrive intervals into 1sec
        mean = (mean_usec.to_f / 1e6) - @heartbeat_interval + 1

        # Calculate phi of the phi accrual failure detector
        t = now - @last - @heartbeat_interval + 1
        phi = PHI_FACTOR * t / mean

        return phi
      end

      def sample_size
        @window.size
      end

      def clear
        @window.clear
        @last = 0
      end
    end
```

まとめというか補足

* デフォルトでは毎秒 udp 接続してパケットを投げる
* in_forward は event driven 的に受け取ったらパケットを返して来る
* packet がかえってくるまでの時間で phi がきまる
* hard_timeout の時間１つもかえってこなかったらあきらめる
* もっと早くあきらめるためのアルゴリズムとして phi_threshold がある

* 毎秒パケットなげるのは量が多すぎじゃないのか。10s にしただけでネットワークだけでなく CPU 使用量も激減
* phi の取り扱いが難しいので、hard_timeout だけでコントロールしている
* event driven 的に I/O 多重化されているので accept までは１プロセスで複数やってくれるが、プラグインの処理が blocking 処理なので、accept した時点でブロックする。
* heartbeat が受け取れなくなる
* udp heartbeat なのでそんなに信頼性は高くない. heartbeat がロストする
