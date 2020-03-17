### 安装 elasticsearch

### 安装 kibana

### 安装 filebeat
配置 filebeat.yml，以下手动配置 其他默认
<pre>
filebeat.inputs:
- type:
  enabled: true
  paths:
    - /var/log/tomcat/***
    
  multiline.pattern: '^[0-9]{4}-[0-9]{1,2}-[0-9]{1,2}'
  multiline.negate: true   # 未匹配条件的是同一行
  multiline.match: after   # 跟在上一条之后
  
output.logstash:
  hosts: ['localhost:5044']
</pre>

### 安装 logstash
logstash配置 通过pipelines.yml读取conf.d文件夹下所有 '.conf' 结尾的文件。

创建 .conf 文件
<pre>
input {
  beats {
    port => 5044
  }
}

output {
  elasticsearch {
    hosts => "127.0.0.1:9200"
    manage_template => false
    index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"  # elasticsearch中每天的index
  }
}
</pre>
