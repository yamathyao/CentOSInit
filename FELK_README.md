### 安装 elasticsearch

### 安装 kibana

### 安装 kafka

### 安装 filebeat
配置 filebeat.yml，以下手动配置 其他默认
<pre>
filebeat.inputs:
- type:
  enabled: true
  paths:
    - /var/log/tomcat/***
    
  field: # 自定义 field， kibana index pattern 需要 refresh
    app_name: ''    
    ip_addr: ''
  fields_under_root: true # 自定义 fields 放root下，false 放在fields下
    
  multiline.pattern: '^[0-9]{4}-[0-9]{1,2}-[0-9]{1,2}'
  multiline.negate: true   # 未匹配条件的是同一行
  multiline.match: after   # 跟在上一条之后
  
# 直接传到logstash
output.logstash: 
  hosts: ['localhost:5044']
# 传 kafka
output.kafka:
  enabled: true
  hosts: ["127.0.0.1:9092"]
  max_retries: 5
  timeout: 300
  topic: "filebeat"
</pre>

### 安装 logstash
logstash配置 通过pipelines.yml读取conf.d文件夹下所有 '.conf' 结尾的文件。

创建 .conf 文件
<pre>
# 直接从 filebeat 取
input {
  beats {
    port => 5044
  }
}
# 从 kafka 取
input {
  kafka {
    bootstrap_servers => "127.0.0.1:9092"
    topics => ["filebeat"]
    group_id => "logstash_group"
    codec => "json"
    consumer_threads => 1
    decorate_events => true
  }
}

output {
  elasticsearch {
    hosts => "127.0.0.1:9200"
    manage_template => false
    index => "filebeat-%{+YYYY.MM.dd}"  # elasticsearch中每天的index, kibana index pattern为filebeat*
  }
}
</pre>
