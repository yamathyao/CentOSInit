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
