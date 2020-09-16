# CentOSInit

参考：https://blog.csdn.net/babyxue/article/details/80970526

安装完成后的配置：

1、配置IP信息

 命令：
  <pre>
  vi /etc/sysconfig/network-scripts/ifcfg-ens33
  </pre>
  修改：
  <pre>
  TYPE="Ethernet"           # 网络类型为以太网
  BOOTPROTO="static"        # 手动分配ip
  NAME="ens33"              # 网卡设备名，设备名一定要跟文件名一致
  DEVICE="ens33"            # 网卡设备名，设备名一定要跟文件名一致
  ONBOOT="yes"              # 该网卡是否随网络服务启动
  IPADDR="192.168.220.101"  # 该网卡ip地址就是你要配置的固定IP，
                            #  如果你要用xshell等工具连接，220这个网段最好和你自己的电脑网段一致，否则有可能用 xshell连接失败
  GATEWAY="192.168.220.2"   # 网关
  NETMASK="255.255.255.0"   # 子网掩码
  DNS1="8.8.8.8"            # DNS，8.8.8.8为Google提供的免费DNS服务器的IP地址
  </pre>
  
2、配置网络

  命令：
  <pre>
  vi /etc/sysconfig/network
  </pre>
  修改：
  <pre>
  NETWORKING=yes            # 网络是否工作，此处一定不能为no
  </pre>
  
3、配置公共DNS服务（可选）

  命令：
  <pre>
  vi /etc/resolv.conf
  </pre>
  增加：
  <pre>
  nameserver 8.8.8.8
  </pre>
  
4、关闭防火墙

  命令：
  <pre>
  systemctl stop firewalld    # 临时关闭防火墙
  systemctl disable firewalld # 禁止开机启动
  </pre>

5、重启网络

  命令：
  <pre>
  service network restart
  </pre>
  如果遇到 ipv4 forward 不可用
  <pre>
  /usr/lib/sysctl.d/00-system.conf
  添加 net.ipv4.ip_forward=1
  </pre>
  
6、磁盘扩展

查看磁盘空间
<pre>
$ fdisk -l
</pre>
<pre>
$ fdisk /dev/sda
-- 获取帮助
$ m
-- 增加分区
$ n
-- 增加主分区，分区号 default
$ p
-- 起始跟结束扇区 enter 默认
$ 
-- 创建物理卷
$ pvcreate /dev/sda3
-- 如果提示找不到 /dev/sda3 ，输入partprobe
$ partprobe
$ pvcreate /dev/sda3
-- vgscan查询物理卷，查询到本机物理卷 cl，扩展 cl
$ vgextend cl /dev/sda3
-- 扩展 lv，增加 20G
$ lvextend -L +20G
-- df -h 发现没有改变，要对文件系统扩容, 'xfs_growfs 扩展名'，或者 'resize2fs -f 扩展名'
$ xfs_growfs /dev/mapper/cl-root

</pre>
  
# 配置阿里源

<pre>
touch /etc/yum.repos.d/kubernetes.repo
</pre>
<pre>
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
</pre>
  
# 安装kubectl

下载最新kubectl
<pre>
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
</pre>
下载特定版本
<pre>
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.7.0/bin/linux/amd64/kubectl
</pre>
使二进制文件可以执行
<pre>
chmod +x ./kubectl
</pre>
将二进制文件移动到移动到PATH中
<pre>
mv ./kubectl /usr/local/bin/kubectl
</pre>

# 安装minikube

阿里镜像
<pre>
curl -Lo minikube http://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/releases/v1.4.0/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
</pre>
官方
<pre>
curl -Lo minikube https://github.com/kubernetes/minikube/releases/download/v1.6.2/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
</pre>

# 安装docker

<pre>
yum install -y docker
systemctl enable docker
systemctl start docker
</pre>

# 设置docker

可以用 docker info|grep -i driver 查看默认的driver是不是systemd
命令：
<pre>
vi /usr/lib/systemd/system/docker.service
</pre>
找到
<pre>
native.cgroupdriver=systemd
</pre>
修改为
<pre>
--exec-opt native.cgroupdriver=cgroupfs
</pre>
重启docker
如果docker出现"Cannot connect to the Docker daemon at unix:///var/run/docker.sock. ..." 重启docker
<pre>
systemctl daemon-reload
systemctl restart docker
</pre>
遇到error
<pre>
shell-init: error retrieving current directory: getcwd:
</pre>
解决
<pre>
cd /usr/libexec/docker/
ln -s docker-runc-current docker-runc
</pre>
遇到
<pre>
Error loading config file "/var/lib/minikube/kubeconfig": open /var/lib/minikube/kubeconfig: permission denied
</pre>
解决
<pre>
vi /etc/selinux/config
将 SELINUX=enforcing 改为 SELINUX=disabled
</pre>

配置docker
<pre>
$ vim /etc/sysconfig/docker
</pre>
<pre>
OPTIONS='--selinux-enabled --log-driver=json-file --log-opt max-size=100m --log-opt max-file=5 --signature-verification=false'
</pre>


# 启动minikube
<pre>
minikube start --vm-driver=none --registry-mirror=https://registry.docker-cn.com
</pre>
--vm-driver 本机运行 国内镜像下载docker

# 启动要素

<pre>
systemctl enable kubelet.service
systemctl start docker
minikube start
启动代理，访问dashboard
kubectl proxy --address='0.0.0.0' --accept-hosts='^*$' &
</pre>

# docker image

准备 jar包, Dockerfile, app.yaml
<pre>
docker build -t &lt;docker image&gt;:&lt;版本&gt; . 
# .代表当前目录的Dockerfile
</pre>
创建deployment, service
<pre>
kubectl create -f app.yaml
kubectl create -f ./[dir] 
</pre>
Node port 30000-32767

查看service, pods
<pre>
kubectl get pods
kubectl get service
</pre>
