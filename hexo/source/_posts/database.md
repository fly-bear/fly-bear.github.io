---
title: 日志系统搭建
date: 2019-09-08 21:51:34
tags:
  - ELK
  - Filebeat
  - Fluentd
---
# 日志系统搭建

### 产生日志

搭建 **logback + slf4j** 框架，定义日志文件路径和级别，在项目中输出日志

### 日志管理

##### **收集日志**

* 每台服务器启动 filebeat 服务，读取日志文件，正则匹配，统一发送到 logstash
* logstash 服务器收集所有日志保存到统一路径下管理
* fluentd 读取日志文件，解析成 Elasticsearch 所用格式，调用 ES 接口导入

##### **日志显示**

部署 Kibana 连接 ES，建立日常所用索引

### filebeat.yaml 配置

```
filebeat.inputs:
- type: log
  paths:
    - /home/ubuntu/logs/app.log
  fields:
    type: app
    business: my-project
  fields_under_root: true
  multiline.pattern: '^\[[0-9]{4}-[0-9]{2}-[0-9]{2}'
  multiline.negate: true
  multiline.match: after
  multiline.max_lines: 10000
  multiline.timeout: 20

- type: log
  paths:
    - /home/ubuntu/logs/access/access.log
  fields:
      type: access
      business: my-project
  fields_under_root: true
  
#---------------------- Logstash output --------------------------
output.logstash:
  hosts: ["logstash-prod.flybear.com:5044"]
```

#### 多行合并
当 log 输出多行时，kibana 上显示为多条，需要进行合并。日志以日期格式 yyyy-MM-dd 开头，配置 filebeat.yml 添加
 ```
 multiline.pattern: '^\[[0-9]{4}-[0-9]{2}-[0-9]{2}'
 multiline.negate: true
 multiline.match: after
 multiline.max_lines: 10000
 multiline.timeout: 20
```
**pattern**：正则表达式

**negate**：true 或 false；默认是false，匹配pattern的行合并到上一行；true，不匹配pattern的行合并到上一行

**match**：after 或 before，合并到上一行的末尾或开头

### logstash.conf 配置
```
 input {
  beats {
    port => 5044
  }
}

filter {
   ruby {
       code => "
            event.timestamp.time.localtime
            tstamp = event.get('@timestamp').to_i
            event.set('date_str', Time.at(tstamp).strftime('%Y-%m-%d'))
        "
   }
}

output {
  file {
    path => "/var/log/remote/logstash/%{business}-%{type}-%{[agent][hostname]}.%{date_str}.log"
    codec => line { format => "%{message}" }
  }
}
```

###  fluentd.conf 配置

#### 配置日志源
```
<source>
  @type tail
  @id mongodb-prod1
  path /logs/logstash/my-project-app-*.log
  pos_file /tmp/my-project.pos
  refresh_interval 30
  tag mongodb-prod1
  path_key log_path
  <parse>
    @type multiline
    format_firstline /^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}/
    format1 /^(?<mongotime>[^ ]*)[ ]*(?<level>[^ ]*)[ ]*(?<command>[^ ]*)[ ]*\[(?<content>[^\]]*)\][ ]*(?<msg>[^$]*)/
    time_format %Y-%m-%d-%H:%M:%S
  </parse>
</source>
```
可以为不同的日志文件配置各自的解析格式

#### 配置输出

```
<match my-project my-project2>
@type elasticsearch
@id elasticsearch_output
  host 172.31.26.211
  port 9200
  logstash_prefix ${tag}
  logstash_format true
  slow_flush_log_threshold 40.0
  <buffer>
    @type file
    path /fluentd/log/elasticsearch.log
    flush_mode interval
    retry_type exponential_backoff
    flush_thread_count 32
    flush_interval 2s
    retry_forever false
    retry_max_interval 60
    chunk_limit_size 5M
    queue_limit_length 2000
    overflow_action throw_exception
  </buffer>
</match>
```
指定所需的所有源，配置输出到 es 的 ip, port，指定缓存路径，分批导入 ES。

