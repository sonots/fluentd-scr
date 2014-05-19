# fluent-agent-lite

Fluentd の agent (ログを読み込んで送信する)としてだけ使える機能を持つ @tagomoris 氏による perl 実装

## ディレクトリ構造

```
$ tree
.
├── Makefile.PL
├── README.md
├── SPECS
│   └── fluent-agent-lite.spec
├── bin
│   ├── cpanm
│   ├── fluent-agent-lite
│   └── install.sh
├── lib
│   └── Fluent
│       └── AgentLite.pm
└── package
    ├── fluent-agent-lite.conf
    └── fluent-agent-lite.init
```

本体は lib/Fluent/AgentLite.pm だが、
オプションを足すような改修をする場合は大体、
bin/fluent-agent-lite、fluent-agent-lite.init もいじることになる

例えば [pull request #22](https://github.com/tagomoris/fluent-agent-lite/pull/22)

## コマンドオプション

bin/fluent-agent-lite コマンドオプション

```
/usr/local/fluent-agent-lite/bin/fluent-agent-lite -h
Usage: fluent-agent-lite [options] TAG TARGET_FILE PRIMARY_SERVER[:PORT] [SECONDARY_SERVER[:PORT]]
       fluent-agent-lite -p SERVER_LIST_FILE [-s SERVER_LIST_FILE] [options] TAG TARGET_FILE

       port default: 24224
Options:
     -f FIELDNAME                fieldname of fluentd log message attribute (DEFAULT: message)
     -p LIST_PATH                primary servers list (server[:port] per line, random selected one server)
     -s LIST_PATH                secondary servers list (server[:port] per line, random selected one server)
     -b BUF_SIZE                 log tailing buffer size (DEFAULT: 1MB)
     -n NICE                     tail process nice (DEFAULT: 0)
     -t TAIL_PATH                tail path (DEFAULT: /usr/bin/tail)
     -i SECONDS                  tail -F sleep interval (GNU tail ONLY, DEFAULT: tail default)
     -l LOG_PATH                 log file path (DEFAULT: /tmp/fluent-agent.log)
     -P TAG:DATA                 send a ping message per minute with specified TAG and DATA (DEFAULT: not to send)
                                   (see also: fluent-plugin-ping-message)
     -S SECONDS                  ping message interval seconds (DEFAULT: 60)
     -d DRAIN_LOG_TAG            emits drain log to fluentd: messages per drain/send (DEFAULT: not to emits)
     -k KEEPALIVE_TIME           connection keepalive time in seconds. 0 means infinity (DEFAULT: 1800, minimum: 120)
     -w RECONNECT_WAIT_MAX       the maximum wait time for TCP socket reconnection in seconds (DEFAULT: 3600, minimum: 0.5)
     -r RECONNECT_WAIT_INCR_RATE the rate to increment the reconnect time (DEFAULT: 1.5, minimum: 1.0)
     -j                          use JSON for message structure in transfering (highly experimental)
     -v                          output logs of level debug and info (DEFAULT: warn/crit only)
     -F                          force start even if input file is not found
     -h                          print this message
```

## 設定ファイル

[fluent-agent-lite.conf](https://github.com/tagomoris/fluent-agent-lite/blob/53223a057bff596bbe2a3393b30ec10ca017a9c9/package/fluent-agent-lite.conf)

```
# fluentd tag prefix of all LOGS
TAG_PREFIX=""

# fluentd message log attribute name (default: message)
# FIELD_NAME="message"

### LOGS: tag /path/to/log/file
#
# LOGS=$(cat <<"EOF"
# apache2     /var/log/apache2/access.log
# # yourservice /var/log/yourservice/access.log
# EOF
# )

### or, read from external file
# LOGS=$(cat /etc/fluent-agent.logs)

# SERVERNAME[:PORTNUM]
# port number is optional (default: 24224)
PRIMARY_SERVER="primary.fluentd.local:24224"

### or, PRIMARY SERVER LIST FILE of servers
# PRIMARY_SERVERS_LIST="/etc/fluent-agent.servers.primary"

# secondary server setting is optional...
# SECONDARY_SERVER="secondary.fluentd.local:24224"

# SECONDARY_SERVERS_LIST is available as like as PRIMARY_SERVERS_LIST

# max bytes to try read as one action from tail (default: 1MB)
# READ_BUFFER_SIZE=1048576

# PROCESS_NICE default: 0
# PROCESS_NICE=-1

# TAIL_INTERVAL=0.1
# TAIL_PATH=/usr/bin/tail

# Tag , data and interval of ping message (not to output ping message when tag not specified)
# fluent-plugin-ping-message を用いた agent の死活監視
# PING_TAG=ping
# PING_DATA=`hostname`
# PING_INTERVAL=60

# Tag name of 'drain_log' (none: not to output drain_log)
# tail からログを読み込んだ(drainした)という記録(ログ)を fluentd に送信する
# DRAIN_LOG_TAG=

# connection keepalive time in seconds. 0 means infinity (DEFAULT: 1800)
# KEEPALIVE_TIME=1800

# The maximum wait time for TCP socket reconnection in seconds (Default: 3600, minimum: 0.5)
# RECONNECT_WAIT_MAX=3600

# The rate to increment the reconnect time (Default: 1.5)
# RECONNECT_WAIT_INCR_RATE=1.5

# LOG_PATH=/tmp/fluent-agent.log
# LOG_VERBOSE=false

# PERL_PATH=/usr/bin/perl
```

↓うちの設定

この conf は実はシェルスクリプトとして実行されるため、
起動時に hostname を取得したり、primary_server_list.conf を生成して指定するなどしている！


```bash
# pre-process
hostname=$(hostname)
mkdir -p /etc/fluent-agent-lite
cat <<"EOF" > /etc/fluent-agent-lite/primary_servers_list.conf
ホスト名:ポート番号
...
...
EOF
# configuration
TAG_PREFIX=raw
LOGS=$(cat <<EOF
用途.ログパス.$hostname ログパス
...
...
EOF
)
PRIMARY_SERVERS_LIST=/etc/fluent-agent-lite/primary_servers_list.conf
LOG_PATH=/var/log/fluent-agent-lite/fluent-agent.log
LOG_VERBOSE=yes
KEEPALIVE_TIME=10800
FORCE_START=1
[ -f /usr/dena/bin/infraperl ] && PERL_PATH=/usr/dena/bin/infraperl
```

## 起動スクリプト


/etc/init.d/fluent-agent-lite start すると、init スクリプト内で fluent-agent-lite.conf をシェルスクリプトして実行(source)し、
結果を元に fluent-agent-lite コマンドをの引数を組みたてて実行する。
ログファイル１つにつき１つ fluent-agent-lite プロセスを立ち上げている。

つまり、起動スクリプトにもロジックがあるということ。つらい。。。


[fluent-agent-lite.init#L58-L65](https://github.com/tagomoris/fluent-agent-lite/blob/53223a057bff596bbe2a3393b30ec10ca017a9c9/package/fluent-agent-lite.init#L58-L65)

```bash
42    lines=$(echo "$LOGS" | grep -v '^#' | grep -v '^$' | wc -l | awk '{print $1;}')
    ...
58    for (( i = 0; i < $lines; i++ )); do
59        COMMANDLINE=$(build_command_line $i)
60        $COMMANDLINE &
61
62        CPID=$!
63        disown $CPID > /dev/null 2>&1
64        echo $CPID >> $PID_FILE
65    done
```

蛇足: この disown で daemon 化している所がよくなくて、
ssh -t 経由で fluent-agent-lite を起動しようとするとログアウト後にすぐ fluent-agent-lite プロセスが HUP されて死ぬという問題がある。
See [Issue #16](https://github.com/tagomoris/fluent-agent-lite/issues/16).

## bin/fluent-agent-lite

[bin/fluent-agent-lite](https://github.com/tagomoris/fluent-agent-lite/blob/53223a057bff596bbe2a3393b30ec10ca017a9c9/bin/fluent-agent-lite)

conf の値を読み込んでいるだけ。と思いきや、もうちょい仕事をしている。
tail コマンドの組み立てや open はこっちでやって AgentLite.pm に渡している

```
sub HELP_MESSAGE # ヘルプ
sub server_entry # host:port を [host, port] に split
sub load_server_list  # server_list ファイルを読み込む
sub build_tail_command # tail コマンドの引数組み立て
sub open_tail # tail コマンドを nonblocking に pipe open
sub open_stdin # STDIN からログを読み込む場合
sub close_fd # tail or STDIN を閉じる
sub main # open_tail して、AgentLite を new して、execute して、おわったら tail を close_fd
```

あと、signal trap もこちらで定義していて、

[fluent-agent-lite#L21-L33](https://github.com/tagomoris/fluent-agent-lite/blob/53223a057bff596bbe2a3393b30ec10ca017a9c9/bin/fluent-agent-lite#L21-L33)

```perl
my $HUPPED = undef;
$SIG{ HUP } = sub { $HUPPED = 1; };      # to reconnect
my $TERMINATED = undef;
$SIG{ INT } = $SIG{ TERM } = sub { $TERMINATED = 1; }; # to terminate

my $checker_terminated = sub { $TERMINATED; };
my $checker_reconnect = sub {
    if (shift) {
        $HUPPED = undef;
    } else {
        $HUPPED or $TERMINATED;
    }
};
```

AgentLite#execute に無名関数を渡していたりもする。ここで tailfd も渡っている。

[fluent-agent-lite#L252-L259](https://github.com/tagomoris/fluent-agent-lite/blob/53223a057bff596bbe2a3393b30ec10ca017a9c9/bin/fluent-agent-lite#L252-L259)

```perl
    $agent->execute( {
        fieldname => $fieldname,
        tailfd => $tailfd,
        checker => {
            term => $checker_terminated,
            reconnect => $checker_reconnect,
        },
    } );
```

## AgentLite.pm

[AgentLite.pm](https://github.com/tagomoris/fluent-agent-lite/blob/53223a057bff596bbe2a3393b30ec10ca017a9c9/lib/Fluent/AgentLite.pm)

本体

### メソッド一覧

```
sub connection_keepalive_time # keepalive time を設定値を元に乱数を使って決定
# class AgentLite
sub new # 初期化. bin/fluent-agent-lite から設定値を受け取る
sub execute # bin/fluent-agent-lite から呼び出される本体
sub drain # tail からログを読み込む
sub pack # ログを msgpack
sub pack_json # ログを json に変換
sub pack_ping_message # ping_message を msgpack
sub pack_drainlog # drainしたというログを msgpack
sub pack_drainlog_json # drainしたというログを json に変換
sub choose # random に送信先 server を選択
sub connect # server に接続
sub send # server にデータ送信
sub close # server への接続を閉じる
```

### [sub new](https://github.com/tagomoris/fluent-agent-lite/blob/53223a057bff596bbe2a3393b30ec10ca017a9c9/lib/Fluent/AgentLite.pm#L41-L67)

bin/fluent-agent-lite から渡された設定を取り込んでいる。

```perl
sub new {
    my $this = shift;
    my ($tag, $primary_servers, $secondary_servers, $configuration) = @_;
    my $self = {
        tag => $tag,
        servers => {
            primary => $primary_servers,
            secondary => $secondary_servers,
        },
        buffer_size => $configuration->{buffer_size},
        ping_message => $configuration->{ping_message},
        drain_log_tag => $configuration->{drain_log_tag},
        keepalive_time => $configuration->{keepalive_time},
        reconnect_wait_incr_rate => $configuration->{reconnect_wait_incr_rate},
        reconnect_wait_max => $configuration->{reconnect_wait_max},
        output_format => $configuration->{output_format},
    };

    # デフォルトは msgpack だが、オプションで json で投げるようにもできる
    if (defined $self->{output_format} and $self->{output_format} eq 'json') {
        *pack = *pack_json;
        *pack_drainlog = *pack_drainlog_json;
    }

    srand (time ^ $PID ^ unpack("%L*", `ps axww | gzip`));

    bless $self, $this;
}
```

### [sub execute](https://github.com/tagomoris/fluent-agent-lite/blob/53223a057bff596bbe2a3393b30ec10ca017a9c9/lib/Fluent/AgentLite.pm#L69-L207)

長いのでちょっと分割して読む

この辺では、受け取った引数から変数を初期化したり、[上にある](https://github.com/tagomoris/fluent-agent-lite/blob/53223a057bff596bbe2a3393b30ec10ca017a9c9/lib/Fluent/AgentLite.pm#L209-L255)定数達から変数を初期化したりしているだけ

```perl
sub execute {
    my $self = shift;
    my $args = shift;

    my $fieldname = $args->{fieldname};
    my $tailfd = $args->{tailfd};

    my $check_terminated = ($args->{checker} || {})->{term} || sub { 0 };
    my $check_reconnect = ($args->{checker} || {})->{reconnect} || sub { 0 };

    my $packer = Data::MessagePack->new();

    my $reconnect_wait = RECONNECT_WAIT_MIN;
    my $reconnect_wait_max = RECONNECT_WAIT_MAX;
    if (defined $self->{reconnect_wait_max}) {
        $reconnect_wait_max = $self->{reconnect_wait_max};
        if ($reconnect_wait_max < RECONNECT_WAIT_MIN) {
            warnf 'Reconnect wait max is too short. Set minimum value %s', RECONNECT_WAIT_MIN;
            $reconnect_wait_max = RECONNECT_WAIT_MIN;
        }
    }
    my $reconnect_wait_incr_rate = RECONNECT_WAIT_INCR_RATE;
    if (defined $self->{reconnect_wait_incr_rate}) {
        $reconnect_wait_incr_rate = $self->{reconnect_wait_incr_rate};
        if ($reconnect_wait_incr_rate < RECONNECT_WAIT_INCR_RATE_MIN) {
            warnf 'Reconnect wait incr rate is too small. Set minimum value %s', RECONNECT_WAIT_INCR_RATE_MIN;
            $reconnect_wait_incr_rate = RECONNECT_WAIT_INCR_RATE_MIN;
        }
    }

    my $last_ping_message = time;
    if ($self->{ping_message}) {
        $last_ping_message = time - $self->{ping_message}->{interval} * 2;
    }
    my $keepalive_time = CONNECTION_KEEPALIVE_TIME;
    if (defined $self->{keepalive_time}) {
        $keepalive_time = $self->{keepalive_time};
        if ($keepalive_time < CONNECTION_KEEPALIVE_MIN and $keepalive_time != CONNECTION_KEEPALIVE_INFINITY) {
            warnf 'Keepalive time setting is too short. Set minimum value %s', CONNECTION_KEEPALIVE_MIN;
            $keepalive_time = CONNECTION_KEEPALIVE_MIN;
        }
    }

    my $pending_packed;
    my $continuous_line;
    my $disconnected_primary = 0;
    my $expiration_enable = $keepalive_time != CONNECTION_KEEPALIVE_INFINITY;
```

メインループ。SIGINT or SIGTERM されるまでループを続ける

```perl
    while(not $check_terminated->()) {
        # at here, connection initialized (after retry wait if required)

        # connect to servers
        my $primary = $self->choose($self->{servers}->{primary});
        my $secondary;

        my $sock = $self->connect($primary) unless $disconnected_primary;
        if (not $sock and $self->{servers}->{secondary}) {
            $secondary = $self->choose($self->{servers}->{secondary});
            $sock = $self->connect($self->choose($self->{servers}->{secondary}));
        }
        $disconnected_primary = 0;
        # retry ループ
        # 実は retry 最中はこのループ内の処理しかされず、tail 子プロセスの監視がされないという問題あったりする
        # See https://github.com/tagomoris/fluent-agent-lite/issues/23
        # nonblocking sleep にして他の処理もやりつつ reconnect_wait を待つようにする必要がある
        unless ($sock) {
            # failed to connect both of primary / secondary
            # Fluentd server に負荷がかかっているなどしてつながらない場合、この warn が出る
            warnf 'failed to connect servers, primary: %s, secondary: %s', $primary, ($secondary || 'none');
            warnf 'waiting %s seconds to reconnect', $reconnect_wait;

            Time::HiRes::sleep($reconnect_wait);
            $reconnect_wait *= $reconnect_wait_incr_rate;
            $reconnect_wait = $reconnect_wait_max if $reconnect_wait > $reconnect_wait_max;
            next;
        }

        # succeed to connect. set keepalive disconnect time
        my $connecting = $secondary || $primary;

        my $expired = time + connection_keepalive_time($keepalive_time) if $expiration_enable;
        $reconnect_wait = RECONNECT_WAIT_MIN;
```

データ読み込み送信ループ。

```perl
        # SIGHUP が送られた場合 reconnect するために一旦ループを抜ける
        while(not $check_reconnect->()) {
            # connection keepalive expired
            if ($expiration_enable and time > $expired) {
                infof "connection keepalive expired.";
                last;
            }

            # ping message (if enabled)
            my $ping_packed = undef;
            if ($self->{ping_message} and time >= $last_ping_message + $self->{ping_message}->{interval}) {
                $ping_packed = $self->pack_ping_message($packer, $self->{ping_message}->{tag}, $self->{ping_message}->{data});
                $last_ping_message = time;
            }

            # drain (sysread)
            my $lines = 0;
            # 送信されていないデータがない場合、tail から読み込みを試みる(drainする)
            if (not $pending_packed) {
                my $buffered_lines;
                ($buffered_lines, $continuous_line, $lines) = $self->drain($tailfd, $continuous_line);

                if ($buffered_lines) {
                    # msgpack にパックする
                    $pending_packed = $self->pack($packer, $fieldname, $buffered_lines);
                    # drain_log オプションが有効の場合、drain したよ記録(ログ)も pack する
                    if ($self->{drain_log_tag}) {
                        $pending_packed .= $self->pack_drainlog($packer, $self->{drain_log_tag}, $lines);
                    }
                }
                # ping オプションが有効の場合、くっつける
                if ($ping_packed) {
                    $pending_packed ||= '';
                    $pending_packed .= $ping_packed;
                }
                unless ($pending_packed) {
                    Time::HiRes::sleep READ_WAIT;
                    next;
                }
            }
            # send
            my $written = $self->send($sock, $pending_packed);
            unless ($written) { # failed to write (socket error).
                $disconnected_primary = 1 unless $secondary;
                last; # 失敗した場合は $pending_packed をリセットせずに last して別のノードに reconnect
            }

            # 成功した場合は $pending_packed をリセット。
            $pending_packed = undef;
        }
```

reconnect 処理

```perl
        if ($check_reconnect->()) {
            infof "SIGHUP (or SIGTERM) received";
            $disconnected_primary = 0;
            $check_reconnect->(1); # clear SIGHUP signal
        }
        infof "disconnecting to current server";
        if ($sock) {
            $sock->close;
            $sock = undef;
        }
        infof "disconnected.";
    }
    if ($check_terminated->()) {
        warnf "SIGTERM received";
    }
    infof "process exit";
}
```

### [sub drain](https://github.com/tagomoris/fluent-agent-lite/blob/53223a057bff596bbe2a3393b30ec10ca017a9c9/lib/Fluent/AgentLite.pm#L209-L255)

tail パイプから読み込む

```perl
sub drain {
    # if broken child process (undefined return value of $fd->sysread())
    #   if content exists, return it.
    #   else die
    my ($self,$fd, $continuous_line) = @_;
    my $readlimit = $self->{buffer_size};
    my $readsize = 0;
    my $readlines = 0;
    my @buffered_lines;

    my $chunk;
    while ($readsize < $readlimit) {
        my $bytes = $fd->sysread($chunk, $readlimit);
        if (defined $bytes and $bytes == 0) { # EOF (child process exit)
            last if $readsize > 0;
            warnf "failed to read from child process, maybe killed.";
            # tail プロセスが死んだ場合は、ここで fluent-agent-lite も死ぬ
            confess "give up to read tailing fd, see logs";
        }
        if (not defined $bytes and $! == EAGAIN) { # I/O Error (no data in fd): "Resource temporarily unavailable"
            last;
        }
        if (not defined $bytes) { # Other I/O error... what?
            warnf "I/O error with tail fd: $!";
            last;
        }

        $readsize += $bytes;
        my $terminated_line = chomp $chunk;
        my @lines = split(m!\n!, $chunk);
        if ($continuous_line) {
            $lines[0] = $continuous_line . $lines[0];
            $continuous_line = undef;
        }
        if (not $terminated_line) {
            $continuous_line = pop @lines;
        }
        if (scalar(@lines) > 0) {
            push @buffered_lines, @lines;
            $readlines += scalar(@lines);
        }
    }
    if ($readlines < 1) {
        return undef, $continuous_line, 0;
    }

    return (\@buffered_lines, $continuous_line, $readlines);
}
```

### [sub send](https://github.com/tagomoris/fluent-agent-lite/blob/53223a057bff596bbe2a3393b30ec10ca017a9c9/lib/Fluent/AgentLite.pm#L318-L344)

```perl
sub send {
    my ($self,$sock,$data) = @_;
    my $length = length($data);
    my $written = 0;
    my $retry = 0;

    local $SIG{"PIPE"} = sub { die $! };

    eval {
        while ($written < $length) {
            my $wbytes = $sock->syswrite($data, $length, $written);
            if ($wbytes) {
                $written += $wbytes;
            }
            else {
                die "failed $retry times to send data: $!" if $retry > SEND_RETRY_MAX;
                $retry += 1;
            }
        }
    };
    if ($@) {
        my $error = $@;
        # 送信が失敗した場合は、このログが出る
        # 2013-11-21T21:11:21 [WARN] (21263) Cannot send data: パイプが切断されました at /usr/local/fluent-agent-lite/bin/../lib/Fluent/AgentLite.pm line 305.\n at /usr/local/fluent-agent-lite/bin/../lib/Fluent/AgentLite.pm line 321
        warnf "Cannot send data: $error";
        return undef;
    }
    $written;
}
```

## まとめというか補足

* init スクリプト内でログの数だけ fluent-agent-lite を起動して disown でデーモン化している
* tail コマンドを open してログを読み込んでいる。結果、Fluentd のようにログ position を覚えたりなどさせるのが難しい。
* Fluentd out_forward のように heartbeat は送っていない。送信に失敗したらランダムで別のノードに reconnect してみるだけ
* Fluentd out_forward と違って keepalive 接続してログを送っている
* Fluentd への接続が不安定になった場合は `Cannot send data` か `failed to connect servers` の warn ログが出る
* 送信が失敗している間は新たに tail から読み込むようなことはない。

   * Fluentd は out_forward が失敗していも、in_tail は別プラグインであるため感知できず、読み込み続けてメモリ溢れしてしまう

* パフォーマンス検証すると、fluent-agent-lite 52万/sec。Fluentd 7万/sec (しかも詰まる)

機能的に足りない点などはあるが、パフォーマンスが素晴らしい
