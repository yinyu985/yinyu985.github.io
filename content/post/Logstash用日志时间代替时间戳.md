---
title: Logstash 用日志时间代替时间戳
slug: logstash-replaces-timestamp-with-log-time
tags: [ELK, Linux]
date: 2022-12-29T21:08:18+08:00
---

最近工作中遇到的一个问题，网络组把网络设备的“陈年”老日志传到 ELK，这样的问题就是日志的时间是过去的，但是 Logstash 在生成时间戳然后输入到 ES 时，默认的是当前的时间戳。于是需求就是将日志中的时间代替它生成的时间戳，开搞！<!--more-->

```lua
input {
    file {
        path => ["/var/log/messages"]
        start_position => "beginning"
        sincedb_path => "/dev/null"
    }
}
filter {
    grok {
        patterns_dir => "/data/logstash/config/conf.d/patterns"
        match => ["message", "%{LOGTIME:logtime}"]
    }
    date {
        match => ["logtime", "yyyy-MM-dd HH:mm:ss"]
        timezone => "Asia/Shanghai"
    }
    mutate {
        remove_field => ["logtime"]
    }
}
output {
    elasticsearch {
        hosts => ["http://10.23.188.108:9200"]
        index => "test_%{+YYYY-MM-dd}"
        user => elastic
        password => "*************"
    }
}
```

以上，就是全部实现的配置了。看起来很简单，我被折腾好久。

原本的日志长成这样，需要提取出 `2022-12-01 07:59:59`：

```
[log_type:other_log] [record_time:2022-12-01 07:59:59] [user:shusenwang] [group:/xiaohongshu.sh/xiaohongshu-sh/社区部/社区技术部/算法组/基础模型组/] [host_ip:10.23.145.24] [dst_ip:17.248.165.75] [serv:网络存储] [app:iCloud_Drive[浏览]] [site:未定义位置] [tm_type:/移动终端/Android系统移动终端] [net_action:记录] [url:-] [DNS:p112-contacts.icloud.com] [filename:-] [filetype:-] 
```

为了匹配到这个时间，我写了一个正则到 `/data/logstash/config/conf.d/patterns`：

```
LOGTIME \d{4}-\d{2}-\d{2}\s\d{2}:\d{2}:\d{2}
```

整体思路就是，用这个 `LOGTIME`，在 `message` 这个串儿中匹配，匹配出来的东西，就叫做 `logtime`，然后 `date` 插件在 `logtime` 这个小串中匹配一个符合 `yyyy-MM-dd HH:mm:ss` 的小小串，就这个 `yyyy-MM-dd HH:mm:ss`。一开始我并不知道写啥，查阅文章，还写了 ISO8601 的标准时间戳格式，后来看到一篇文章说 `date` 插件默认会把匹配到的时间转换成时间戳 `@timestamp`，后来我一想，既然它默认转成时间戳，那还需要我告诉它时间戳长成啥样吗？那肯定是告诉它你要把什么样的时间格式转成时间戳啦。不过我传入 `date` 的串就是 `logtime` 啊，我感觉上面的 `grok` 都多余了。直接用 `date` 插件就可以了，反正你喜欢找是吧，自己在 `message` 里面找吧！！！

```lua
input {
    file {
        path => ["/var/log/messages"]
        start_position => "beginning"
        sincedb_path => "/dev/null"
    }
}
filter {
    date {
        match => ["message", "yyyy-MM-dd HH:mm:ss"]
        timezone => "Asia/Shanghai"
    }
    mutate {
        remove_field => ["logtime"]
    }
}
output {
    elasticsearch {
        hosts => ["http://10.23.188.108:9200"]
        index => "test_%{+YYYY-MM-dd}"
        user => elastic
        password => "*************"
    }
}
```

我也是写这个记录的时候才想到的这个问题，没有实际实验过，以上内容自测哦。

附加，因为把时间戳替换成了过去的时间，所以在 Kibana 查询最近的记录时会发现，咦，我日志呢？不用慌，它们回到了过去。

## 参考文章

[logstash最佳实践--时间处理](https://doc.yonyoucloud.com/doc/logstash-best-practice-cn/filter/date.html)

[logstash采集日志的时间格式探微](https://wiki.eryajf.net/pages/3396.html#_1-date-filter-%E6%8F%92%E4%BB%B6)