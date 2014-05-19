# fluend-scr

1. [Fluentd の out_forward と heartbeat](./01_out_forward_heartbeat.md)
2. [Fluentd の out_forward と BufferedOutput](./02_out_forward_buffered.md)
3. [Fluentd の in_forward とプラグイン呼び出し](./03_in_forward_plugin.md)
4. [fluent-agent-lite](./04_fluent_agent_lite.md)

# 前準備

https://github.com/fluent/fluentd を clone して動かせるようにしておこう ^^

ruby は入っているものとして、

```
$ git clone git@github.com:fluent/fluentd.git
$ cd fluentd
$ bundle
$ bundle exec fluentd -c fluent.conf # 動いてるっぽければとりあえずOK
```

