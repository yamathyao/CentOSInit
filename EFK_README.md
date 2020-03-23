# EFK

Fluentbit （收集）、Fluentd（过滤）、Elasticsearch（存储）、Kibana（展示）

# 部署elasticsearch集群

使用3个 Elasticsearch Pod 来避免高可用下多节点集群中出现的“脑裂”问题

### 创建一个名为 logging 的 namespace
<pre>
$ kubectl  create  namespace logging
</pre>
### 创建建一个名为 elasticsearch 的无头服务
elasticsearch-svc.yaml
<pre>
kind: Service
apiVersion: v1
metadata:
  name: elasticsearch
  namespace: logging
  labels:
    app: elasticsearch
spec:
  selector:
    app: elasticsearch
  clusterIP: None
  ports:
    - port: 9200
      name: rest
    - port: 9300
      name: inter-node
</pre>
创建服务资源对象
<pre>
$ kubectl create -f elasticsearch-svc.yaml
$ kubectl get svc -n logging  // 指定 logging 中的service
</pre>

### 创建ES持久化存储
创建StorageClass

创建elasticsearch-storageclass.yaml
<pre>
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: es-data-db
  namespace: logging
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
</pre>
创建服务资源对象
<pre>
$ kubectl create -f elasticsearch-storageclass.yaml 
$ kubectl get storageclass
</pre>

### 创建 storage pv, storageclass-pv.yaml
<pre>
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
  namespace: logging
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: es-data-db
  local:
    path: /usr/share/elasticsearch/data/local/vol1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - minikube
</pre>
创建
<pre>
$ kubectl create -f storageclass-pv.yaml
$ kubectl get pv -n logging
</pre>

### 创建 storage pvc, storage-pvc.yaml
<pre>
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim
  namespace: logging
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: es-data-db
</pre>
创建
<pre>
$ kubectl create -f storage-pvc.yaml
$ kubectl get pvc -n logging
</pre>

### 使用 StatefulSet 创建Es Pod
Kubernetes StatefulSet 允许我们为 Pod 分配一个稳定的标识和持久化存储，Elasticsearch 需要稳定的存储来保证 Pod 在重新调度或者重启后的数据依然不变，所以需要使用 StatefulSet 来管理 Pod。

elasticsearch-statefulset.yaml
<pre>
apiVersion: apps/v1
kind: StatefulSet  #定义了名为 es-cluster 的 StatefulSet 对象
metadata:
  name: es-cluster
  namespace: logging
spec:
  serviceName: elasticsearch # 和前面创建的 Service 相关联，这可以确保使用以下 DNS 地址访问 StatefulSet 中的每一个 Pod：es-cluster-[0,1,2].elasticsearch.logging.svc.cluster.local，其中[0,1,2]对应于已分配的 Pod 序号。
  replicas: 3 #3个副本
  selector:
    matchLabels:
      app: elasticsearch #设置匹配标签为app=elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec: #定义Pod模板
      containers:
      - name: elasticsearch
        image: docker.io/elasticsearch:6.5.0
        resources: {}
#            limits:
#              cpu: 1000m
#            requests:
#              cpu: 100m
        ports:
        - containerPort: 9200
          name: rest
          protocol: TCP
        - containerPort: 9300
          name: inter-node
          protocol: TCP
        volumeMounts: #声明数据持久化目录
        - name: mypd
          mountPath: /usr/share/elasticsearch/data
        env: #定义变量
          - name: cluster.name #Elasticsearch 集群的名称
            value: k8s-logs
          - name: node.name 
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: discovery.zen.ping.unicast.hosts #设置在 Elasticsearch 集群中节点相互连接的发现方法。
            value: "es-cluster-0.elasticsearch.logging.svc.cluster.local,es-cluster-1.elasticsearch.logging.svc.cluster.local,es-cluster-2.elasticsearch.logging.svc.cluster.local" # 在一个namespace，EsPodDNS 域简写,规则：{statefulset.name}-{0~N}.{service.name}.svc.cluster.local
          - name: discovery.zen.minimum_master_nodes #我们将其设置为(N/2) + 1，N是我们的群集中符合主节点的节点的数量。我们有3个 Elasticsearch 节点，因此我们将此值设置为2（向下舍入到最接近的整数）
            value: "2"
          - name: node.max_local_storage_nodes
            value: "5"
          - name: ES_JAVA_OPTS #JVM限制
            value: "-Xms512m -Xmx512m"
      volumes:
        - name: mypd
          persistentVolumeClaim:
            claimName: claim
      initContainers: #环境初始化容器
      - name: fix-permissions
        image: busybox
        command: ["sh", "-c", "chown -R 1000:1000 /usr/share/elasticsearch/data"] 
        securityContext:
          privileged: true
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
      - name: increase-vm-max-map
        image: busybox
        command: ["sysctl", "-w", "vm.max_map_count=262144"]
        securityContext:
          privileged: true
      - name: increase-fd-ulimit
        image: busybox
        command: ["sh", "-c", "ulimit -n 65536"]
        securityContext:
          privileged: true
</pre>

### 验证es服务是否正常

将本地端口9200转发到 Elasticsearch 节点（如 es-cluster-0）对应的端口
<pre>
$ kubectl port-forward es-cluster-0 9200:9200 --namespace=logging
Forwarding from 127.0.0.1:9200 -> 9200
Forwarding from [::1]:9200 -> 9200
</pre>
其他一个终端
<pre>
$ curl http://localhost:9200/_cluster/state?pretty
会显示 es-cluster-0, es-cluster-1, es-cluster-2 信息
</pre>

# 部署 kibana
<pre>
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: logging
  labels:
    app: kibana
spec:
  ports:
  - port: 5601
  type: NodePort
  selector:
    app: kibana

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: logging
  labels:
    app: kibana
spec:
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: docker.io/kibana:6.5.0
        resources: {}
          # limits:
          #  cpu: 1000m
          # requests:
          #  cpu: 100m
        env:
          - name: ELASTICSEARCH_URL
            value: http://elasticsearch:9200
        ports:
        - containerPort: 5601
</pre>
创建 kibana
<pre>
$ kubectl create -f kibana.yaml
service/kibana created
deployment.apps/kibana created
$ kubectl get pods --namespace=logging

$ kubectl get svc  --namespace=logging
</pre>

# 部署 fluentd

使用DasemonSet 控制器来部署 Fluentd 应用，确保在集群中的每个节点上始终运行一个 Fluentd 容器

新建 fluentd-configmap.yaml
<pre>
kind: ConfigMap
apiVersion: v1
metadata:
  name: fluentd-config
  namespace: logging
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
data:
  system.conf: |-
    <system>
      root_dir /tmp/fluentd-buffers/
    </system>
  containers.input.conf: |-
    <source>
      @id fluentd-containers.log     # 表示引用该日志源的唯一标识符，可用于进一步过滤和路由结构化日志数据
      @type tail                     # Fluentd 内置的指令，tail表示 Fluentd 从上次读取的位置通过 tail 不断获取数据另外一个是 http 表示通过一个 GET 来收集数据
      path /var/log/containers/*.log # tail 类型下特定参数，告诉 Fluentd 采集 /var/log/containers 目录下的日志，这是 docker 在 Kubernetes 节点上用来存储运行容器 stdout 输出日志数据的目录
      pos_file /var/log/es-containers.log.pos # 检查点，如果 Fluentd 程序重新启动，将使用此文件中的位置来恢复日志数据收集
      time_format %Y-%m-%dT%H:%M:%S.%NZ
      localtime
      tag raw.kubernetes.* # 用来将日志源与目标或者过滤器匹配的自定义字符串，Fluentd 匹配源/目标标签来路由日志数据
      format json
      read_from_head true
    </source>
    <match raw.kubernetes.**>
      @id raw.kubernetes
      @type detect_expections
      remove_tag_prefix raw
      message log
      stream stream
      multiline_flush_interval 5
      max_bytes 500000
      max_lines 1000
    </match>
  system.input.conf: |-
    <source>
      @id journald-docker
      @type systemd
      filters [{ "_SYSTEMD_UNIT": "docker.service" }]
      <storage>
        @type local
        persistent true
      </storage>
      read_from_head true
      tag docker
    </source>
    <source>
      @id journald-kubelet
      @type systemd
      filters [{ "_SYSTEMD_UNIT": "kubelet.service" }]
      <storage>
        @type local
        persistent true
      <storage>
      read_from_head true
      tag kubelet
    </source>
  forward.input.conf: |-
    <source>
      @type forward
    </source>
  output.conf: |-
    <filter kubernetes.**>
      @type kubernetes_metadata
    </filter>
    <match **>  # 标识一个目标标签，后面是匹配日志源的正则表达式，想要捕获所有的日志并发送给 Elasticsearch，所以要配置成**
      @id elasticsearch
      @type elasticsearch  # 输出到 Elasticsearch
      @log_level info  # 捕获日志级别 INFO 级别以上
      include_tag_key true
      host elasticsearch
      port 9200
      logstash_format true  # Elasticsearch 服务对日志数据构建反向索引搜索，将 logstash_format 设置true，Fluentd 将会以 logstash 格式来转发结构化的日志数据
      request_timeout 30s
      <buffer>  # Fluentd 允许在目标不可用时进行缓存，比如网络出现故障或者 Elasticsearch 不可用时。缓冲区配置也有助于降低磁盘 IO
        @type file
        path /var/log/fluentd-buffers/kubernetes.system.buffer
        flush_mode interval
        retry_type exponential_backoff
        flush_thread_count 2
        flush_interval 5s
        retry_forever
        retry_max_interval 30
        chunk_limit_size 2M
        queue_limit_length 8
        overflow_action block
      </buffer>
    </match>
</pre>
