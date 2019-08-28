# kubernetes 1.15.3源码部署

## kubernets核心组件

- Etcd: 集群存储, 保存整个集群的状态信息

- kube-apiserver: 提供了资源操作的唯一入口, 并提供认证、授权访问控制、API 注册和发现等机制

- kube-scheduler: 负责资源的调度, 按照预定的调度策略将 Pod 调度到相应的节点上

- kube-controller-manager: 负责维护集群的状态, 比如故障检测、自动扩展、滚动更新等

- kubelet: 负责维持容器的生命周期, 同时也负责 Volume (CVI) 和网络 (CNI) 的管理

- Container runtime: 负责镜像管理以及 Pod 和容器的真正运行 (CRI), 默认的容器运行时为 Docker

- kube-proxy: 负责为 Service 提供集群内部的服务发现和负载均衡

## 推荐Add-ons

- CoreDNS: 负责为整个集群提供 DNS 服务

- Ingress Controller: 为服务提供外网入口

- metrics-server: 提供资源监控

- Dashboard: 提供 GUI

- EFK: 集群日志采集、存储与查询

## 前期准备

| 软件包名称                                | 说明          | 下载路径                                                                                           | 版本     |
|:------------------------------------:|:-----------:|:----------------------------------------------------------------------------------------------:| ------ |
| kubernetes-server-linux-amd64.tar.gz | master节点安装包 | https://dl.k8s.io/v1.15.3/kubernetes-server-linux-amd64.tar.gz                                 | 1.15.3 |
| kubernetes-node-linux-amd64.tar.gz   | node节点安装包   | https://dl.k8s.io/v1.15.3/kubernetes-node-linux-amd64.tar.gz                                   | 1.15.3 |
| etcd-v3.3.15-linux-amd64.tar.gz      | etcd集群安装包   | https://github.com/etcd-io/etcd/releases/download/v3.3.15/etcd-v3.3.15-linux-amd64.tar.gz      | 3.3.15 |
| flannel-v0.11.0-linux-amd64.tar.gz   | CNI网络插件     | https://github.com/coreos/flannel/releases/download/v0.11.0/flannel-v0.11.0-linux-amd64.tar.gz | 0.11.0 |

#### 安装规划

- pod分配 IP 段: 10.244.0.0/16

- ClusterIP 地址: 10.99.0.0/16

- CoreDns 地址: 10.99.110.110

- 统一安装路径: /data/apps/

| *hostname* | IP           | role   | 组件                                                                                                 | VIP       |
|:----------:|:------------:|:------:|:--------------------------------------------------------------------------------------------------:|:---------:|
| k8s1       | 10.0.100.178 | Master | kube-apiserver, kube-controller-manager, kube-scheduler, etcd, kube-proxy, docker, flanel, kubelet | 10.0.0.17 |
| k8s2       | 10.0.100.147 | Master | kube-apiserver, kube-controller-manager, kube-scheduler, etcd, kube-proxy, docker, flanel, kubelet | 10.0.0.17 |
| k8s3       | 10.0.101.78  | Master | kube-apiserver, kube-controller-manager, kube-scheduler, etcd, kube-proxy, docker, flanel, kubelet | 10.0.0.17 |
| k8s4       | 10.0.101.54  | Node   | kubelet,kube-proxy, docker,flannel                                                                 | None      |
| k8s5       | 10.0.100.164 | Node   | kubelet,kube-proxy, docker,flannel                                                                 | None      |
| k8s6       | 10.0.101.215 | Node   | kubelet,kube-proxy, docker,flannel                                                                 | None      |
| LB1        | 10.0.0.18    | LB     | haproxy,keepalived                                                                                 | 10.0.0.17 |
| LB2        | 10.0.0.19    | LB     | haproxy,keepalived                                                                                 | 10.0.0.17 |

## Docker安装

*在各节点安装docker*

```bash
$ sudo apt-get install apt-transport-https ca-certificates curl software-properties-common
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
$ sudo add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) \
  stable"
$ sudo apt-get update && sudo apt-get install docker-ce
$ systemctl enable docker
$ systemctl start docker
```

## cfssl安装

*在 k8s1节点操作*

```bash
$ wget -O /bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
$ wget -O /bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 
$ wget -O /bin/cfssl-certinfo https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 $ for cfssl in `ls /bin/cfssl*`;do chmod +x $cfssl;done
```

## Etcd集群搭建

etcd 是基于 Raft 的分布式 key-value 存储系统, 由 CoreOS 开发, 常用于服务发现、共享配置以及并发控制(如 leader 选举、分布式锁等). kubernetes 使用 etcd 存储所有运行数据. 

#### 1. 制作etcd证书

*在k8s1节点操作*

```bash
$ mkdir -pv /root/software/etcd-ssl && cd /root/software/etcd-ssl
$ pwd
/root/software/etcd-ssl
```

**制作CA证书, expiry为证书过期时间(10年)**

```bash
$ cat > ca-config.json << EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
          "signing",
          "key encipherment",
          "server auth",
          "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
EOF
```

**生成 etcd CA 证书请求文件， ST/L/字段请自行修改**

```bash
$ cat > etcd-ca-csr.json << EOF
{
  "CN": "etcd",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShangHai",
      "L": "ShangHai",
      "O": "etcd",
      "OU": "Etcd Security"
    }
  ]
}
EOF
```

**生成etcd CA 证书**

```bash
$ cfssl gencert -initca etcd-ca-csr.json | cfssljson -bare etcd-ca
$ ls etcd-ca*.pem
 etcd-ca-key.pem etcd-ca.pem
```

**生成 etcd 证书请求文件，ST/L/字段可自行修改, hosts字段ip为安装etcd节点ip及vip**

```bash
$ cat > etcd-csr.json << EOF
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "10.0.100.178",
    "10.0.100.147",
    "10.0.101.78",
    "10.0.0.17"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShangHai",
      "L": "ShangHai",
      "O": "etcd",
      "OU": "Etcd Security"
    }
  ]
}
EOF
```

**生成etcd 证书**

```bash
$ cfssl gencert -ca=etcd-ca.pem -ca-key=etcd-ca-key.pem -config=ca-config.json -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
$ ls etcd*.pem 
etcd-key.pem  etcd.pem
```

*分别在各 etcd节点创建以下目录*

```bash
$ mkdir -pv /data/apps/etcd/{ssl,bin,etc,data}
$ ls /data/apps/etcd/
bin  data  etc  ssl
```

**分发证书文件**

```bash
# to k8s1
$ cp /root/software/etcd-ssl/etcd*.pem /data/apps/etcd/ssl
# to k8s2
$ scp /root/software/etcd-ssl/etcd*.pem root@k8s2:/data/apps/etcd/ssl/
# to k8s3
$ scp /root/software/etcd-ssl/etcd*.pem root@k8s3:/data/apps/etcd/ssl/
```

#### 2. 各etcd节点配置

```bash
$ cd /root/software/
$ wget https://github.com/etcd-io/etcd/releases/download/v3.3.15/etcd-v3.3.15-linux-amd64.tar.gz
$ ls etcd-v3.3.15-linux-amd64.tar.gz
etcd-v3.3.15-linux-amd64.tar.gz
$ tar -xzvf etcd-v3.3.15-linux-amd64.tar.gz
$ mv etcd-v3.3.15-linux-amd64/etcd* /data/apps/etcd/bin/
$ echo "PATH=$PATH:/data/apps/etcd/bin/" >> /etc/profile.d/etcd.sh
$ chmod +x /etc/profile.d/etcd.sh
$ source /etc/profile.d/etcd.sh
```

**各etcd节点创建系统服务文件**

```bash
$ cat > /lib/systemd/system/etcd.service << EOF
[Unit]
Description=Etcd Server 
After=network.target 
After=network-online.target 
Wants=network-online.target 

[Service] 
Type=notify 
WorkingDirectory=/data/apps/etcd/ 
EnvironmentFile=-/data/apps/etcd/etc/etcd.conf 
User=etcd 
#set GOMAXPROCS to number of processors 
ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /data/apps/etcd/bin/etcd --name=\"${ETCD_NAME}\" --data-dir=\"${ETCD_DATA_DIR}\" --listen-client-urls=\"${ETCD_LISTEN_CLIENT_URLS}\"" 
Restart=on-failure 
LimitNOFILE=65536 

[Install] 
WantedBy=multi-user.target
EOF
```

**各etcd 节点创建配置文件, 根据实际情况修改**

```bash
$ cd /data/apps/etcd/etc/
$ cat > etcd.conf << EOF
[Member] 
#ETCD_CORS="" 
ETCD_NAME="k8s1" 

ETCD_DATA_DIR="/data/apps/etcd/data/default.etcd" 
#ETCD_WAL_DIR="" 
ETCD_LISTEN_PEER_URLS="https://10.0.100.178:2380" 
ETCD_LISTEN_CLIENT_URLS="https://127.0.0.1:2379,https://10.0.100.178:2379" 
#ETCD_MAX_SNAPSHOTS="5" 
#ETCD_MAX_WALS="5" 
#ETCD_SNAPSHOT_COUNT="100000" 
#ETCD_HEARTBEAT_INTERVAL="100" 
#ETCD_ELECTION_TIMEOUT="1000" 
#ETCD_QUOTA_BACKEND_BYTES="0" 
#ETCD_MAX_REQUEST_BYTES="1572864" 
#ETCD_GRPC_KEEPALIVE_MIN_TIME="5s" 
#ETCD_GRPC_KEEPALIVE_INTERVAL="2h0m0s" 
#ETCD_GRPC_KEEPALIVE_TIMEOUT="20s" 
#
#[Clustering] 
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.0.100.178:2380" 
ETCD_ADVERTISE_CLIENT_URLS="https://127.0.0.1:2379,https://10.0.100.178:2379" 
#ETCD_DISCOVERY="" 
#ETCD_DISCOVERY_FALLBACK="proxy" 
#ETCD_DISCOVERY_PROXY="" 
#ETCD_DISCOVERY_SRV="" 
ETCD_INITIAL_CLUSTER="k8s1=https://10.0.100.178:2380,k8s2=https://10.0.100.147:2380,k8s3=https://10.0.101.78:2380" 
ETCD_INITIAL_CLUSTER_TOKEN="BigBoss" 
#ETCD_INITIAL_CLUSTER_STATE="new" 
#ETCD_STRICT_RECONFIG_CHECK="true" 
#ETCD_ENABLE_V2="true" 
# 
#[Proxy] 
#ETCD_PROXY="off" 
#ETCD_PROXY_FAILURE_WAIT="5000" 
#ETCD_PROXY_REFRESH_INTERVAL="30000" 
#ETCD_PROXY_DIAL_TIMEOUT="1000" 
#ETCD_PROXY_WRITE_TIMEOUT="5000" 
#ETCD_PROXY_READ_TIMEOUT="0" 
# 
#[Security] 
ETCD_CERT_FILE="/data/apps/etcd/ssl/etcd.pem" 
ETCD_KEY_FILE="/data/apps/etcd/ssl/etcd-key.pem" 
#ETCD_CLIENT_CERT_AUTH="false" 
ETCD_TRUSTED_CA_FILE="/data/apps/etcd/ssl/etcd-ca.pem" 
#ETCD_AUTO_TLS="false" 
ETCD_PEER_CERT_FILE="/data/apps/etcd/ssl/etcd.pem" 
ETCD_PEER_KEY_FILE="/data/apps/etcd/ssl/etcd-key.pem" 
#ETCD_PEER_CLIENT_CERT_AUTH="false" 
ETCD_PEER_TRUSTED_CA_FILE="/data/apps/etcd/ssl/etcd-ca.pem" 
#ETCD_PEER_AUTO_TLS="false" 
#
#[Logging] 
ETCD_DEBUG="false" 
#ETCD_LOG_PACKAGE_LEVELS="" 
ETCD_LOG_OUTPUT="default" 
# 
#[Unsafe] 
#ETCD_FORCE_NEW_CLUSTER="false" 
# 
#[Version]
#ETCD_VERSION="false" 
#ETCD_AUTO_COMPACTION_RETENTION="0" 
# 
#[Profiling] 
#ETCD_ENABLE_PPROF="false" 
#ETCD_METRICS="basic" 
# 
#[Auth] 
#ETCD_AUTH_TOKEN="simple"
EOF
```

**各etcd节点启动服务**

```bash
$ useradd -r etcd && chown etcd.etcd -R /data/apps/etcd/
$ systemctl daemon-reload
$ systemctl enable etcd
$ systemctl start etcd
$ systemctl status etcd
● etcd.service - Etcd Server
   Loaded: loaded (/lib/systemd/system/etcd.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2019-08-21 07:47:15 UTC; 5 days ago
 Main PID: 25294 (etcd)
    Tasks: 13 (limit: 4704)
   CGroup: /system.slice/etcd.service
           └─25294 /data/apps/etcd/bin/etcd --name=k8s1 --data-dir=/data/apps/etcd/data/default.etcd --listen-client-urls=https://127.0.0.1:2379,https://10.0.100.178:2379
```

**验证etcd集群状态**

```bash
$ etcdctl --endpoints "https://10.0.100.178:2379,https://10.0.100.147:2379,https://10.0.101.78:2379" --ca-file=/data/apps/etcd/ssl/etcd-ca.pem --cert-file=/data/apps/etcd/ssl/etcd.pem --key-file=/data/apps/etcd/ssl/etcd-key.pem member list
48b276f4ebfb8fd: name=k8s3 peerURLs=https://10.0.101.78:2380 clientURLs=https://10.0.101.78:2379,https://127.0.0.1:2379 isLeader=false
22ef0c58bbf6b461: name=k8s1 peerURLs=https://10.0.100.178:2380 clientURLs=https://10.0.100.178:2379,https://127.0.0.1:2379 isLeader=false
6883a17e87ece290: name=k8s2 peerURLs=https://10.0.100.147:2380 clientURLs=https://10.0.100.147:2379,https://127.0.0.1:2379 isLeader=true
```

## 安装 Kubernetes 组件

*在k8s1节点操作*

```bash
$ mkdir $HOME/software/k8s-ssl && cd $HOMe/software/k8s-ssl
$ pwd
/root/software/k8s-ssl
```

### 制作证书

#### 1. CA证书

**配置ca证书请求文件, ST/L/expiry 字段可自行修改**

```bash
$ cat > ca-csr.json << EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShangHai",
      "L": "ShangHai",
      "O": "k8s",
      "OU": "System"
    }
  ],
  "ca": {
    "expiry": "87600h"
  }
}
EOF
```

**生成CA证书**

```bash
$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
$ ls ca*.pem
ca-key.pem  ca.pem
```

#### 2. kube-apiserver证书

 **配置kube-apiserver证书请求文件** 

*如需自定义集群域名, 请 修改 cluster | cluster.local 字段为自定义域名. hosts字段为master节点ip及vip*

```bash
$ cat > kube-apiserver-csr.json << EOF
{
  "CN": "kube-apiserver",
  "hosts": [
    "127.0.0.1",
    "10.0.100.178",
    "10.0.100.147",
    "10.0.101.78",
    "10.0.0.17",
    "10.99.0.1",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShangHai",
      "L": "ShangHai",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```

**生成kube-apiserver证书**

```bash
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=/root/software/etcd-ssl/ca-config.json -profile=kubernetes kube-apiserver-csr.json | cfssljson -bare kube-apiserver
$ ls kube-apiserver*.pem 
kube-apiserver-key.pem  kube-apiserver.pem
```

#### 3. kube-controller-manager证书

**配置kube-controller-manager证书请求文件**

```bash
$ cat > kube-controller-manager-csr.json << EOF
{
  "CN": "system:kube-controller-manager",
  "hosts": [
    "127.0.0.1",
    "10.0.100.178",
    "10.0.100.147",
    "10.0.101.78",
    "10.0.0.17"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShangHai",
      "L": "ShangHai",
      "O": "system:kube-controller-manager",
      "OU": "System"
    }
  ]
}
EOF
```

**生成kube-controller-manager证书**

```bash
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=/root/software/etcd-ssl/ca-config.json -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
$ ls kube-controller-manager*.pem 
kube-controller-manager-key.pem  kube-controller-manager.pem
```

#### 4. kube-scheduler证书

**配置kube-scheduler证书请求文件**

```bash
$ cat > kube-scheduler-csr.json << EOF
{
  "CN": "system:kube-scheduler",
  "hosts": [
    "127.0.0.1",
    "10.0.100.178",
    "10.0.100.147",
    "10.0.101.78",
    "10.0.0.17"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShangHai",
      "L": "ShangHai",
      "O": "system:kube-scheduler",
      "OU": "System"
    }
  ]
}
EOF
```

**生成kube-scheduler证书**

```bash
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=/root/software/etcd-ssl/ca-config.json -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler
$ ls kube-scheduler*.pem 
kube-scheduler-key.pem  kube-scheduler.pem
```

#### 5. kube-proxy 证书

**配置kube-proxy 证书请求文件**

```bash
$ cat > kube-proxy-csr.json  << EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShangHai",
      "L": "ShangHai",
      "O": "system:kube-proxy",
      "OU": "System"
    }
  ]
}
EOF
```

**生成kube-proxy 证书**

```bash
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=/root/software/etcd-ssl/ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
$ ls kube-proxy*.pem 
kube-proxy-key.pem  kube-proxy.pem
```

#### 6. admin 证书

*为集群组件kubelet 、 kubectl 配置admin TLS 认证证书, 具有访问 kubernetes 所有 api 的权限*

**配置admin 证书请求文件**

```bash
$ cat > admin-csr.json << EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShangHai",
      "L": "ShangHai",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF
```

**生成admin证书**

```bash
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=/root/software/etcd-ssl/ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
$ ls admin*.pem 
admin-key.pem  admin.pem
```

*各节点创建证书存放目录*

```bash
$ mkdir -pv /data/apps/kubernetes/{pki,log,etc,certs}
$ ls /data/apps/kubernetes/
certs  etc  log  pki
```

 **分发证书文件**

*master节点:*

```bash
# to k8s1
$ cp ca*.pem admin*.pem kube-proxy*.pem kube-scheduler*.pem kube-controller-manager*.pem kube-apiserver*.pem /data/apps/kubernetes/pki/
# to k8s2
$ scp ca*.pem admin*.pem kube-proxy*.pem kube-scheduler*.pem kube-controller-manager*.pem kube-apiserver*.pem root@k8s2:/data/apps/kubernetes/pki/
# to k8s3
$ scp ca*.pem admin*.pem kube-proxy*.pem kube-scheduler*.pem kube-controller-manager*.pem kube-apiserver*.pem root@k8s3:/data/apps/kubernetes/pki/
```

*node节点:*

```bash
# to k8s4
$ scp ca*.pem admin*.pem kube-proxy*.pem root@k8s4:/data/apps/kubernetes/pki/
# to k8s5
$ scp ca*.pem admin*.pem kube-proxy*.pem root@k8s5:/data/apps/kubernetes/pki/
# to k8s6
$ scp ca*.pem admin*.pem kube-proxy*.pem root@k8s6:/data/apps/kubernetes/pki/
```

### 配置各master节点

*解压server包并将各组件执行文件移到/usr/bin/下*

```bash
$ cd /root/software
$ wget https://dl.k8s.io/v1.15.3/kubernetes-server-linux-amd64.tar.gz
$ ls kubernetes-server-linux-amd64.tar.gz 
kubernetes-server-linux-amd64.tar.gz
$ tar -xzvf kubernetes-server-linux-amd64.tar.gz
$ for i in `ls kubernetes/server/bin/kube* |grep -v tag |grep -v tar `;do mv $i /usr/bin/;done
$ ls /usr/bin/kube* 
/usr/bin/kube-apiserver  /usr/bin/kube-controller-manager  /usr/bin/kube-proxy  /usr/bin/kube-scheduler  /usr/bin/kubeadm  /usr/bin/kubectl  /usr/bin/kubelet
```

*配置kubectl命令行自动补全*

```bash
$ apt install bash-completion
$ cat > /etc/profile.d/kubernetes.sh << EOF 
source <(kubectl completion bash) 
EOF
$ source /etc/profile.d/kubernetes.sh
```

*验证kubectl版本*

```bash
$ kubectl version 
Client Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.3", GitCommit:"2d3c76f9091b6bec110a5e63777c332469e0cba2", GitTreeState:"clean", BuildDate:"2019-08-19T11:13:54Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/amd64"}
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

### 配置TLS Bootstrapping

*在k8s1上操作*

```bash
$ mkdir /root/software/kubeconfig && cd /root/software/kubeconfig
```

#### 1. 生成token

```bash
$ export BOOTSTRAP_TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')
$ echo ${BOOTSTRAP_TOKEN}
295b197b8da138a1415329665bfd1cf2
$ cat > /data/apps/kubernetes/token.csv << EOF 
${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
```

**Note:**  *将${BOOTSTRAP_TOKEN}替换为实际值,防止启动kubelet出错*

```bash
$ cat token.csv 
295b197b8da138a1415329665bfd1cf2,kubelet-bootstrap,10001,"system:kubelet-bootstrap"
```

#### 2. 创建 kubelet bootstrapping kubeconfig

*设置kube-apiserver访问地址, 后面需要对kube-apiserver配置高可用集群, 这里设置kube-apiserver的vip. KUBE_APISERVER=vip*

```bash
$ export KUBE_APISERVER="https://10.0.0.17:8443"
# 设置集群参数
$ kubectl config set-cluster kubernetes --certificate-authority=/data/apps/kubernetes/pki/ca.pem --embed-certs=true --server=${KUBE_APISERVER} --kubeconfig=kubelet-bootstrap.kubeconfig
# 设置客户端认证参数
$ kubectl config set-credentials kubelet-bootstrap --token=${BOOTSTRAP_TOKEN} --kubeconfig=kubelet-bootstrap.kubeconfig
# 设置上下文参数
$ kubectl config set-context default --cluster=kubernetes --user=kubelet-bootstrap --kubeconfig=kubelet-bootstrap.kubeconfig
# 设置当前使用的上下文
$ kubectl config use-context default --kubeconfig=kubelet-bootstrap.kueconfig

$ cat kubelet-bootstrap.kubeconfig  
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUR3akNDQXFxZ0F3SUJBZ0lVQnpjeEhPNE90dzFVVWtPaitya2VlRlhZMFNNd0RRWUpLb1pJaHZjTkFRRUwKQlFBd1p6RUxNQWtHQTFVRUJoTUNRMDR4RVRBUEJnTlZCQWdUQ0ZOb1lXNW5TR0ZwTVJFd0R3WURWUVFIRXdoVAphR0Z1WjBoaGFURU1NQW9HQTFVRUNoTURhemh6TVE4d0RRWURWUVFMRXdaVGVYTjBaVzB4RXpBUkJnTlZCQU1UCkNtdDFZbVZ5Ym1WMFpYTXdIaGNOTVRrd09ESXhNRGMxTWpBd1doY05Namt3T0RFNE1EYzFNakF3V2pCbk1Rc3cKQ1FZRFZRUUdFd0pEVGpFUk1BOEdBMVVFQ0JNSVUyaGhibWRJWVdreEVUQVBCZ05WQkFjVENGTm9ZVzVuU0dGcApNUXd3Q2dZRFZRUUtFd05yT0hNeER6QU5CZ05WQkFzVEJsTjVjM1JsYlRFVE1CRUdBMVVFQXhNS2EzVmlaWEp1ClpYUmxjekNDQVNJd0RRWUpLb1pJaHZjTkFRRUJCUUFEZ2dFUEFEQ0NBUW9DZ2dFQkFLTWdIRE5JeHhmQnJPRzEKYkRYUWY4TEN0MTNIQUN3N2t4TkMwRExZazFoZUN0SWdWU0ppbVphTU13eldqYWxWT1ZlODNSOFdKTGNBUWpXZwpmMkR3Vy9iZXpjSTYzSThWQWtoT052VUxsTzRaaGJuSE0veWg2cmdqZHNxTUxiV29FR2MxTzNOWm02T2hMU1ZaCk5tN1lCejZiV09sdmFoeFR4QUpsdlJHTFJyMXZhU2cyc3hmOU52eWlLRGpORE5NWDUxUS93aFRoK3Nmc28xblQKMG5DSzRRc2RrSzNhR2s4RGRRY3piYWpuTmdiQURzdHRKcENFZk4yL2JWK3JoNVZRcWFzUTlabVFYQXF3TmN1SAphNU8yL1hSRFdHV2tnZlZPRnJBT3NybDRwOGFIaE9CaXV1Z2F6NFhMOTFHNkd2RUhJRE1taUhZMXpNdXVaOFp4Ck5GUEQ3NkVDQXdFQUFhTm1NR1F3RGdZRFZSMFBBUUgvQkFRREFnRUdNQklHQTFVZEV3RUIvd1FJTUFZQkFmOEMKQVFJd0hRWURWUjBPQkJZRUZLc3N5WHZvNnVNaTJTbERpTVVOcTgwL05rMFdNQjhHQTFVZEl3UVlNQmFBRktzcwp5WHZvNnVNaTJTbERpTVVOcTgwL05rMFdNQTBHQ1NxR1NJYjNEUUVCQ3dVQUE0SUJBUUJlZ0ZsN3FjN3RPYzRwCnhuWVJKdmQ5QlpFWDZhQlF3WjJuYkR4M1RIa1ZLeG4wZmo3LzYzK0JoUnpjcXBuN3N4MDFEWXg5YzB1MDUzR1oKN2ZMbGZEZVkvdzcwa1FyNkNrc3p6QkRtY1Z3cys3Nnd3engxYUFlWmRlWktDUXNWUTdvaWZ5MGlxNDNOZ1dYbwpSc2c5VlE1L0FwYzltaGhJcThhRU9lL2EzaXowYjhRSmdSOE1yMkRldWcxVGZSS2x0YytqMFdFUm9TeUEweUN4CnN0dkkwaG1pSDdmQkhWc24rSThUYk9HKzRpWGFSMS9qejkyVTB1Qlc4NGR2ak53WTd2ellNazRYMmpNN0pLd24KczgycExEZ3BGRGduZTJ5WDIrZGRKQ1dyaGQ3VzFRTDlHR3NiZk1kY0dUdHV4MzVSQVEyRytZb2ZzNldKWVg4agoyaXdMeU94OAotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    server: https://10.0.0.17:8443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubelet-bootstrap
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: kubelet-bootstrap
  user:
    token: 295b197b8da138a1415329665bfd1cf2
```

#### 3. 创建 kube-controller-manager kubeconfig

```bash
# 设置集群参数
$ kubectl config set-cluster kubernetes --certificate-authority=/data/apps/kubernetes/pki/ca.pem  --embed-certs=true --server=${KUBE_APISERVER} --kubeconfig=kube-controller-manager.kubeconfig
# 设置客户端认证参数
$ kubectl config set-credentials kube-controller-manager --client-certificate=/data/apps/kubernetes/pki/kube-controller-manager.pem --client-key=/data/apps/kubernetes/pki/kube-controller-manager-key.pem --embed-certs=true --kubeconfig=kube-controller-manager.kubeconfig
# 设置上下文参数
$ kubectl config set-context default --cluster=kubernetes --user=kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig
# 设置当前使用的上下文
$ kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig

$ cat kube-controller-manager.kubeconfig 
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUR3akNDQXFxZ0F3SUJBZ0lVQnpjeEhPNE90dzFVVWtPaitya2VlRlhZMFNNd0RRWUpLb1pJaHZjTkFRRUwKQlFBd1p6RUxNQWtHQTFVRUJoTUNRMDR4RVRBUEJnTlZCQWdUQ0ZOb1lXNW5TR0ZwTVJFd0R3WURWUVFIRXdoVAphR0Z1WjBoaGFURU1NQW9HQTFVRUNoTURhemh6TVE4d0RRWURWUVFMRXdaVGVYTjBaVzB4RXpBUkJnTlZCQU1UCkNtdDFZbVZ5Ym1WMFpYTXdIaGNOTVRrd09ESXhNRGMxTWpBd1doY05Namt3T0RFNE1EYzFNakF3V2pCbk1Rc3cKQ1FZRFZRUUdFd0pEVGpFUk1BOEdBMVVFQ0JNSVUyaGhibWRJWVdreEVUQVBCZ05WQkFjVENGTm9ZVzVuU0dGcApNUXd3Q2dZRFZRUUtFd05yT0hNeER6QU5CZ05WQkFzVEJsTjVjM1JsYlRFVE1CRUdBMVVFQXhNS2EzVmlaWEp1ClpYUmxjekNDQVNJd0RRWUpLb1pJaHZjTkFRRUJCUUFEZ2dFUEFEQ0NBUW9DZ2dFQkFLTWdIRE5JeHhmQnJPRzEKYkRYUWY4TEN0MTNIQUN3N2t4TkMwRExZazFoZUN0SWdWU0ppbVphTU13eldqYWxWT1ZlODNSOFdKTGNBUWpXZwpmMkR3Vy9iZXpjSTYzSThWQWtoT052VUxsTzRaaGJuSE0veWg2cmdqZHNxTUxiV29FR2MxTzNOWm02T2hMU1ZaCk5tN1lCejZiV09sdmFoeFR4QUpsdlJHTFJyMXZhU2cyc3hmOU52eWlLRGpORE5NWDUxUS93aFRoK3Nmc28xblQKMG5DSzRRc2RrSzNhR2s4RGRRY3piYWpuTmdiQURzdHRKcENFZk4yL2JWK3JoNVZRcWFzUTlabVFYQXF3TmN1SAphNU8yL1hSRFdHV2tnZlZPRnJBT3NybDRwOGFIaE9CaXV1Z2F6NFhMOTFHNkd2RUhJRE1taUhZMXpNdXVaOFp4Ck5GUEQ3NkVDQXdFQUFhTm1NR1F3RGdZRFZSMFBBUUgvQkFRREFnRUdNQklHQTFVZEV3RUIvd1FJTUFZQkFmOEMKQVFJd0hRWURWUjBPQkJZRUZLc3N5WHZvNnVNaTJTbERpTVVOcTgwL05rMFdNQjhHQTFVZEl3UVlNQmFBRktzcwp5WHZvNnVNaTJTbERpTVVOcTgwL05rMFdNQTBHQ1NxR1NJYjNEUUVCQ3dVQUE0SUJBUUJlZ0ZsN3FjN3RPYzRwCnhuWVJKdmQ5QlpFWDZhQlF3WjJuYkR4M1RIa1ZLeG4wZmo3LzYzK0JoUnpjcXBuN3N4MDFEWXg5YzB1MDUzR1oKN2ZMbGZEZVkvdzcwa1FyNkNrc3p6QkRtY1Z3cys3Nnd3engxYUFlWmRlWktDUXNWUTdvaWZ5MGlxNDNOZ1dYbwpSc2c5VlE1L0FwYzltaGhJcThhRU9lL2EzaXowYjhRSmdSOE1yMkRldWcxVGZSS2x0YytqMFdFUm9TeUEweUN4CnN0dkkwaG1pSDdmQkhWc24rSThUYk9HKzRpWGFSMS9qejkyVTB1Qlc4NGR2ak53WTd2ellNazRYMmpNN0pLd24KczgycExEZ3BGRGduZTJ5WDIrZGRKQ1dyaGQ3VzFRTDlHR3NiZk1kY0dUdHV4MzVSQVEyRytZb2ZzNldKWVg4agoyaXdMeU94OAotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    server: https://10.0.0.17:8443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kube-controller-manager
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: kube-controller-manager
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUVOakNDQXg2Z0F3SUJBZ0lVRldSQm1zT0lYMy82SDNYcGc4TVNWRmpnYkFrd0RRWUpLb1pJaHZjTkFRRUwKQlFBd1p6RUxNQWtHQTFVRUJoTUNRMDR4RVRBUEJnTlZCQWdUQ0ZOb1lXNW5TR0ZwTVJFd0R3WURWUVFIRXdoVAphR0Z1WjBoaGFURU1NQW9HQTFVRUNoTURhemh6TVE4d0RRWURWUVFMRXdaVGVYTjBaVzB4RXpBUkJnTlZCQU1UCkNtdDFZbVZ5Ym1WMFpYTXdIaGNOTVRrd09ESXhNRGd3TWpBd1doY05Namt3T0RFNE1EZ3dNakF3V2pDQmxqRUwKTUFrR0ExVUVCaE1DUTA0eEVUQVBCZ05WQkFnVENGTm9ZVzVuU0dGcE1SRXdEd1lEVlFRSEV3aFRhR0Z1WjBoaAphVEVuTUNVR0ExVUVDaE1lYzNsemRHVnRPbXQxWW1VdFkyOXVkSEp2Ykd4bGNpMXRZVzVoWjJWeU1ROHdEUVlEClZRUUxFd1pUZVhOMFpXMHhKekFsQmdOVkJBTVRIbk41YzNSbGJUcHJkV0psTFdOdmJuUnliMnhzWlhJdGJXRnUKWVdkbGNqQ0NBU0l3RFFZSktvWklodmNOQVFFQkJRQURnZ0VQQURDQ0FRb0NnZ0VCQUxLRUlSQktpU2NKWVJZNApHZ09ZUXFyb0xOOW5VYklwaFRUbDFjZ0RuWlhoQzlYQzZNdWwveGpkTFJoaVZ1bXgvSGhFU3ZHZFo1dnhHMTNrCkdVKzBoSnBUY1BjbUZENFBGMjBFTnlrdEQ0cndXSWpWVHd1dERvZkNZdFY5M0x4dTBldGFZb0dMbUcwOVlkd1QKNmhISElOUUluUnJCdWh5K1NxSXE2cUMvaTcyUDd4aHg2NlhjZzB5dWtIZTRFZ3hrRFRldGplc0xKYkw5L2pkYgpuL1lzRDNwRjNWRGpicldhOTdTUi9xZnM0UU9GcTZHSWR2Yit3MjcvZHZDbjdQZ2VzalVjTkMwaEcvUkc5eXNCCjhvMXNNQ05BOGNBc1lPV3RQNlN3OWUzM2c2OGhnZzNFdkhiNGVBOXJwUFVJNGNPWWVRREY3SjA5YTRVa0h0YjgKKzdNTmcrMENBd0VBQWFPQnFUQ0JwakFPQmdOVkhROEJBZjhFQkFNQ0JhQXdIUVlEVlIwbEJCWXdGQVlJS3dZQgpCUVVIQXdFR0NDc0dBUVVGQndNQ01Bd0dBMVVkRXdFQi93UUNNQUF3SFFZRFZSME9CQllFRktaS3dFRDdyR0lwCkNJV2NwVWxOTWJpOVBBcGFNQjhHQTFVZEl3UVlNQmFBRktzc3lYdm82dU1pMlNsRGlNVU5xODAvTmswV01DY0cKQTFVZEVRUWdNQjZIQkg4QUFBR0hCQW9BWkxLSEJBb0FaSk9IQkFvQVpVNkhCQW9BQUJFd0RRWUpLb1pJaHZjTgpBUUVMQlFBRGdnRUJBSlFjOHErMG5KcXFXNEE2bnpjd2JEOXdPSE5qWDBCV2s4RS9WNXZGUnZ0bnBHa2RJSzgzCnFjQ3pkemdoU0QrMWs4di93VEE4dVZJQjVDUUNxQm9LcGhJamdLendEU0JORnFPNlJKUjBvMGp2UCsvcUNJaEIKOEZ5eTUrMnVZODhFdDBzams2UWdVRWh2dFMrY3Q2cG1QaW51S3l6YnNrOGFlZ1dzM0hyOHVicGNsMVlMV2pjQQpDZ1Z4MkZwZ21oYVI4TXJXWHkxbEFWMGhhVmJQVUR3b09vSkl0Z3l3ZEw2NTE1M0JKMU5yb0dZUDZDV2xYRjNECmliZGRXdmxUQ083YndCVXkxSXlCU3BBMjEyVEN4U2pHOWdLcFlIN3dKY0lWaTExK1hGM0hjeDZhbHlSTkFNOW8KTHNQSjlMWVQ3RGxpb1d5WEF3NlJ0WEtZTytPTGt0eE9aMVU9Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFb3dJQkFBS0NBUUVBc29RaEVFcUpKd2xoRmpnYUE1aENxdWdzMzJkUnNpbUZOT1hWeUFPZGxlRUwxY0xvCnk2WC9HTjB0R0dKVzZiSDhlRVJLOFoxbm0vRWJYZVFaVDdTRW1sTnc5eVlVUGc4WGJRUTNLUzBQaXZCWWlOVlAKQzYwT2g4SmkxWDNjdkc3UjYxcGlnWXVZYlQxaDNCUHFFY2NnMUFpZEdzRzZITDVLb2lycW9MK0x2WS92R0hIcgpwZHlEVEs2UWQ3Z1NER1FOTjYyTjZ3c2xzdjMrTjF1Zjlpd1Bla1hkVU9OdXRacjN0SkgrcCt6aEE0V3JvWWgyCjl2N0RidjkyOEtmcytCNnlOUncwTFNFYjlFYjNLd0h5ald3d0kwRHh3Q3hnNWEwL3BMRDE3ZmVEcnlHQ0RjUzgKZHZoNEQydWs5UWpodzVoNUFNWHNuVDFyaFNRZTF2ejdzdzJEN1FJREFRQUJBb0lCQUgrMURvSTlFRWtnNkplZwpvdHVYZlhvT2hxdDdtbkkrU2RGQjZ1SWYxQWg0NnFLTndVU1BDQ09kZHJsUEFLWkdjanNIZ0NYQldYR3gxc1lnCmZBc05OUi9DT2JwVlAzMzJCZWd6YjlMQkxiRlRwOEtiOXVSL2RUbWgwbHF3bzgwWjZvcllLa2hLdVV6TThNa2sKWmZzNTNUNVN1ekY5RGN1cVJuSWxDWnpkNnZZOFltNU9ISDd0dHRoalY1NlB6Rm1yUVJ1Zm5JTjhGNkxlZHN6aApYenRqZXEwODNSQVF5RG1sQ0pzY0xQUkthMkJsMmp0VXpaRC8zWHhNY3RodG8rYXlhbW1aRnJIQmZ5Z29EOTJoCmYxWjVxdFRzVFJBWS9rdmlsK1NpR3Y4WURTVjlVSktyRXU1dzIrY0FKRmN5ZDhZaGpmRkppRGdOV0pKUGoyVXAKSmhYZXJ3RUNnWUVBd056aURLc3RHbmllUUhia0hQMzVzWWFWVS9ZaDluM2NKNDFiZ252WSs0a09oZVJVT0tJbQpLM05udTFBTmtOcWR6OG5wa1g0R05rT3dML0pOVWc2ajdpaHU5clRENW1HMkR2M1ZSTXd4VGVKNTlRNmMwT0lRCklMQ1JaZ2NsUUJUeWdwTS9tWE5jSk9tcWUxSTVyQ2FjMk5jbDI0Q3FOdnkxYXZvSGd3NEl3UkVDZ1lFQTdQVG4KSjU0QVRLUk9tbmlib29oUUlqWkhLWWwrS3VrWGVnb1VKSkp4dGJHZUJxYUVCdWhnN3BhSnI4Z3UwUktaWkJwSQovWUtCOWxydlJMSUowQjErRkdHVGNiTGFoMXZLOEZCZ0FBZlFxY2VRVHRCcFBYaStNSW81L2U1R1BUenp3T2RnCjAvd09CRmhzL1pCcytUMjBZU0twS3BkeDJiS1FNcFRidVlTbFZSMENnWUJxdFl3cElFa0xYWE9LRFg0M2dGcTQKVzlPaHFneXVtb0xHSzVOWFJma1BhNHpxamlQL1ZkQXl1RjdMcUFacGdGeFN6TS83M1RQSXNIajZmbUZEcHJBVApKTElJdEltem5acWkvdFVTaEx3KzhMRXo0c3JuVkQxQ0tRKzUyUGhHVlpDOHFJWkcvQ29lamw3eWJ0TlVLZVVjCm9TWGtKbk9IaXhsQndHZUpucWsvVVFLQmdRRE9MRjBBZEpLd0hQcWpuek5UM1NWVVQwUGwyVk1sQlFFL1Y0dWwKTXFLcncrckt3Skg1N0xHQ2h3c3dIbzdWclVnMytFTHdDV0VKT0tBZGRvZmhRL2dTeGIvajJ3b1hZb0FXVHVqbwp2ZVFLQmJFRFVvVnZUaUsxMjErUUdZV1YvUFhlTDdScFhsUFg1aFNYSDlZaG0xWGFlcTBVZVFjL3N3V1NiVUV0CmowUEg1UUtCZ0JoSzBicDVTam85QUFMNnB3N3hmYXZKcXVncmJMSmRxNzl4OUVObHNrM0Q0cGVWbXJENGZtdTcKRE5mZlpCV0dxTDdqNDJ3QUZhQ2FydUgrc2FEcWlmOWQrMkVieE5YR3UrcTcyNW9RK1Job1puZHp3d2sycFRFSwpTdTV1QkhyNms4a25URTVBWDREQ0d0Z3JaQ0FueVFNSG5EN0RQNExmaTFOMUpHcXFXd01sCi0tLS0tRU5EIFJTQSBQUklWQVRFIEtFWS0tLS0tCg==
```

#### 4. 创建 kube-scheduler kubeconfig

```bash
# 设置集群参数
$ kubectl config set-cluster kubernetes --certificate-authority=/data/apps/kubernetes/pki/ca.pem --embed-certs=true --server=${KUBE_APISERVER} --kubeconfig=kube-scheduler.kubeconfig
# 设置客户端认证参数
$ kubectl config set-credentials kube-scheduler --client-certificate=/data/apps/kubernetes/pki/kube-scheduler.pem --client-key=/data/apps/kubernetes/pki/kube-scheduler-key.pem --embed-certs=true --kubeconfig=kube-scheduler.kubeconfig
# 设置上下文参数
$ kubectl config set-context default --cluster=kubernetes --user=kube-scheduler --kubeconfig=kube-scheduler.kubeconfig
# 设置当前使用的上下文
$ kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig

$ cat kube-scheduler.kubeconfig 
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUR3akNDQXFxZ0F3SUJBZ0lVQnpjeEhPNE90dzFVVWtPaitya2VlRlhZMFNNd0RRWUpLb1pJaHZjTkFRRUwKQlFBd1p6RUxNQWtHQTFVRUJoTUNRMDR4RVRBUEJnTlZCQWdUQ0ZOb1lXNW5TR0ZwTVJFd0R3WURWUVFIRXdoVAphR0Z1WjBoaGFURU1NQW9HQTFVRUNoTURhemh6TVE4d0RRWURWUVFMRXdaVGVYTjBaVzB4RXpBUkJnTlZCQU1UCkNtdDFZbVZ5Ym1WMFpYTXdIaGNOTVRrd09ESXhNRGMxTWpBd1doY05Namt3T0RFNE1EYzFNakF3V2pCbk1Rc3cKQ1FZRFZRUUdFd0pEVGpFUk1BOEdBMVVFQ0JNSVUyaGhibWRJWVdreEVUQVBCZ05WQkFjVENGTm9ZVzVuU0dGcApNUXd3Q2dZRFZRUUtFd05yT0hNeER6QU5CZ05WQkFzVEJsTjVjM1JsYlRFVE1CRUdBMVVFQXhNS2EzVmlaWEp1ClpYUmxjekNDQVNJd0RRWUpLb1pJaHZjTkFRRUJCUUFEZ2dFUEFEQ0NBUW9DZ2dFQkFLTWdIRE5JeHhmQnJPRzEKYkRYUWY4TEN0MTNIQUN3N2t4TkMwRExZazFoZUN0SWdWU0ppbVphTU13eldqYWxWT1ZlODNSOFdKTGNBUWpXZwpmMkR3Vy9iZXpjSTYzSThWQWtoT052VUxsTzRaaGJuSE0veWg2cmdqZHNxTUxiV29FR2MxTzNOWm02T2hMU1ZaCk5tN1lCejZiV09sdmFoeFR4QUpsdlJHTFJyMXZhU2cyc3hmOU52eWlLRGpORE5NWDUxUS93aFRoK3Nmc28xblQKMG5DSzRRc2RrSzNhR2s4RGRRY3piYWpuTmdiQURzdHRKcENFZk4yL2JWK3JoNVZRcWFzUTlabVFYQXF3TmN1SAphNU8yL1hSRFdHV2tnZlZPRnJBT3NybDRwOGFIaE9CaXV1Z2F6NFhMOTFHNkd2RUhJRE1taUhZMXpNdXVaOFp4Ck5GUEQ3NkVDQXdFQUFhTm1NR1F3RGdZRFZSMFBBUUgvQkFRREFnRUdNQklHQTFVZEV3RUIvd1FJTUFZQkFmOEMKQVFJd0hRWURWUjBPQkJZRUZLc3N5WHZvNnVNaTJTbERpTVVOcTgwL05rMFdNQjhHQTFVZEl3UVlNQmFBRktzcwp5WHZvNnVNaTJTbERpTVVOcTgwL05rMFdNQTBHQ1NxR1NJYjNEUUVCQ3dVQUE0SUJBUUJlZ0ZsN3FjN3RPYzRwCnhuWVJKdmQ5QlpFWDZhQlF3WjJuYkR4M1RIa1ZLeG4wZmo3LzYzK0JoUnpjcXBuN3N4MDFEWXg5YzB1MDUzR1oKN2ZMbGZEZVkvdzcwa1FyNkNrc3p6QkRtY1Z3cys3Nnd3engxYUFlWmRlWktDUXNWUTdvaWZ5MGlxNDNOZ1dYbwpSc2c5VlE1L0FwYzltaGhJcThhRU9lL2EzaXowYjhRSmdSOE1yMkRldWcxVGZSS2x0YytqMFdFUm9TeUEweUN4CnN0dkkwaG1pSDdmQkhWc24rSThUYk9HKzRpWGFSMS9qejkyVTB1Qlc4NGR2ak53WTd2ellNazRYMmpNN0pLd24KczgycExEZ3BGRGduZTJ5WDIrZGRKQ1dyaGQ3VzFRTDlHR3NiZk1kY0dUdHV4MzVSQVEyRytZb2ZzNldKWVg4agoyaXdMeU94OAotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    server: https://10.0.0.17:8443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kube-scheduler
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: kube-scheduler
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUVKRENDQXd5Z0F3SUJBZ0lVR3ZPUGhDOEVJWjgwWGZuUDVUem52cEsvbHVvd0RRWUpLb1pJaHZjTkFRRUwKQlFBd1p6RUxNQWtHQTFVRUJoTUNRMDR4RVRBUEJnTlZCQWdUQ0ZOb1lXNW5TR0ZwTVJFd0R3WURWUVFIRXdoVAphR0Z1WjBoaGFURU1NQW9HQTFVRUNoTURhemh6TVE4d0RRWURWUVFMRXdaVGVYTjBaVzB4RXpBUkJnTlZCQU1UCkNtdDFZbVZ5Ym1WMFpYTXdIaGNOTVRrd09ESXhNRGd3TkRBd1doY05Namt3T0RFNE1EZ3dOREF3V2pDQmhERUwKTUFrR0ExVUVCaE1DUTA0eEVUQVBCZ05WQkFnVENGTm9ZVzVuU0dGcE1SRXdEd1lEVlFRSEV3aFRhR0Z1WjBoaAphVEVlTUJ3R0ExVUVDaE1WYzNsemRHVnRPbXQxWW1VdGMyTm9aV1IxYkdWeU1ROHdEUVlEVlFRTEV3WlRlWE4wClpXMHhIakFjQmdOVkJBTVRGWE41YzNSbGJUcHJkV0psTFhOamFHVmtkV3hsY2pDQ0FTSXdEUVlKS29aSWh2Y04KQVFFQkJRQURnZ0VQQURDQ0FRb0NnZ0VCQUt1dkg4ejlSbjliQjRPZ2ZuVHA0NHFNb3BaQTF5VXhpZGwyUDBmawpsbU5HdkU0c0hjWXJTSkhZU3pFdGJYRFM3enVsM2NxNzE0ODZOVmYrZlBGSVhINWI5RmdIS0FtY3VSbStndkU1CjU2dHpERmZVK1VxNUNtblRicmplQlhMeE9MMjZ5SG5tZEtrODJMbWgxSGZ6dHZZQWJxaUJleUY5T0k5Tk5NVEwKeFlXR2tTT2tMRGNmemJsM1plNEU3TnZRYXE0bEhVL2p6S3dpNDdVK1pLUGpiMW5VVFFveWxSaVlTUlFmMmExNQp3SmlrRkQ0VmlwU1FDb1hJcmJTeGRleUZVR3dlN0tSNVJKUUQveDIyWnU0SjR6aTRLWkNtN240ejJtUTdCK2FHClVjSnVlQWRXakFZRWZVdk9NVW02NXZhM29rcEZ1WnUvYTNMcmxCZmIvTzBnQkQ4Q0F3RUFBYU9CcVRDQnBqQU8KQmdOVkhROEJBZjhFQkFNQ0JhQXdIUVlEVlIwbEJCWXdGQVlJS3dZQkJRVUhBd0VHQ0NzR0FRVUZCd01DTUF3RwpBMVVkRXdFQi93UUNNQUF3SFFZRFZSME9CQllFRkd4bStpRU1kalNwdktoRy9JZU14V3QvdjRKUk1COEdBMVVkCkl3UVlNQmFBRktzc3lYdm82dU1pMlNsRGlNVU5xODAvTmswV01DY0dBMVVkRVFRZ01CNkhCSDhBQUFHSEJBb0EKWkxLSEJBb0FaSk9IQkFvQVpVNkhCQW9BQUJFd0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFJUkxMblJYN09qUAoxRXNNeElsWFgyb3hIZmVWcjhWZFNWbGMveCt6U1Zaby9HM3dnVkEwRVRTejVZb0x0Zm51YTZhMGRtZzlrak9UClByU1JIZjFaNklzUFZTTmQvV0NGYWNOS2RRTUVxdU5kMHVPRGFCcmNlUFNmaEVCN1k2MDlESSsySDhwcnhSd1YKNjRFK0V1UVdNeWdiYjZncElEZmtFbW9IbTIxbDN4VStKMmVIamg0ZlRWTnlGZ3czc09MQWF1UnZ3MUVSVGJIdApOb2pza21HUzBaMTZ5Q0pNajRwOUZPOTdmM1lIbFl6d2lmZGY4N2MrRXRJcm1FY3FLUEY0Nkl0eE8yemF2YndSCjhtVEZKNGtSNlFnYUZjM21zYUo0TkpzbWM0d3JTTU5OWlpGTEJrL0RFQVM0aWNJaWlJekdZRGZMbnJWNlVwSEUKUHBnZjVpVFJMSkE9Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFb2dJQkFBS0NBUUVBcTY4ZnpQMUdmMXNIZzZCK2RPbmppb3lpbGtEWEpUR0oyWFkvUitTV1kwYThUaXdkCnhpdElrZGhMTVMxdGNOTHZPNlhkeXJ2WGp6bzFWLzU4OFVoY2ZsdjBXQWNvQ1p5NUdiNkM4VG5ucTNNTVY5VDUKU3JrS2FkTnV1TjRGY3ZFNHZickllZVowcVR6WXVhSFVkL08yOWdCdXFJRjdJWDA0ajAwMHhNdkZoWWFSSTZRcwpOeC9OdVhkbDdnVHMyOUJxcmlVZFQrUE1yQ0xqdFQ1a28rTnZXZFJOQ2pLVkdKaEpGQi9aclhuQW1LUVVQaFdLCmxKQUtoY2l0dExGMTdJVlFiQjdzcEhsRWxBUC9IYlptN2duak9MZ3BrS2J1ZmpQYVpEc0g1b1pSd201NEIxYU0KQmdSOVM4NHhTYnJtOXJlaVNrVzVtNzlyY3V1VUY5djg3U0FFUHdJREFRQUJBb0lCQUgzTVcxdmo5Z1VabVY3cwplZHlIQ05DYmpnTFV6aENWeFBGUUFMeFlGWTMyNWNITjk1OGVWaFZ2ekdEamJYNnZRTmFQQ2Y0a042WGVPL29YCklrdlYvdGdqM3QybG1NTzZUN002Y2szNVpQU3UzMHQ0WlpaSUVnWkxBNlY0SWJ3QVh0Zy9CZWkwWWFVa1RaVnYKcS9TYzR1Sk1uTWpoMzJ4QmlmRU8zR3lhOTBlSGlIRWV2eVg4cVpxcTNvVFJManlRVFdLYmxlK3BMNXpmcTd6TAo3Mkxkbm9DWk9EbEhTZ2x2ZkU2OHIzN0pJL01abXZIV1BBdWNETlRacThQb3hkRGxhSWQ0b2s4cXVEcm41REU5CmhmSUVCL1pQYXdoNGY2aSs4bTZqZXU2VVppTVBDQkdlN29sNlZWMDJPYnNULzh0eVpxMTRIOEwzaFNRK21wMUYKWXltZW55RUNnWUVBeTNDNTZJMUxpN2pVU2swb25jck00cXltSVB4VmJlcS80M3FtT29aVzBQZzlheUtpTEhkawpVUmtGZEI2VDZxSTRiVUVCUWZLdEcxaG5iOW16VzJ5U3E0QkQyYWYzTm9VRXNiZVVmeC8zNUgvUEpxVW5WblFkCjZXYWtpVVpVb05ncmpQVjg1T1c5aUNUSk5WZ3I0ZXBqZ3VScE9ad2RTVFB6bW9XcmRzdHZWRzhDZ1lFQTJBb1UKejBIUHlDNlpOOG5UV2p4OE9GY0FCOEUyMUF3N2dzc1RzWWphQmRsMVcxSWVEcHRIQ0tEZTRWTldJbjQvQXFCSwp6Zk5zY2ZKL0MwRjlKMFR4YitrVk1FYWtDYlhpTklqU29leGZNQ0l2VGQwY3oweWJxRzFkQ2hXYzY1dGlEUFNXCm5Wb29FM052ckgwZGREN0E1VTZCM0hBSk5xTm45QXgrVFFxNFZURUNnWUJrNjd2ZDZGSUVzeURrNXhmeUJ3dlMKbXZFaXhlcWZSMmYvc2ZWS2JTQWVORGRMc1hlZjlXNVhhTUV5MUlSdVRpRU4yY1NFOFp6OFJzT3hVZDdPeUxLTgp6MmhaVGlDdDlCamJESVhtOW5YajdaOVd2WEVoU3lNWGlPcXdpcW9xekhIMlVFV3Z5MlJWYUdKRVMwUWhvMFBRClIvMEhMakc5QWIrajlSR1ZNZUE5a3dLQmdIV25UOXZyZ0dnSmtLSEVSVmtRTmFwTkh4UWFFbXo2MkhJTGZJY2oKKzNCU0ZFcU9keFlIVkhFTGd6WDlONXlEV25kb3FqUnRERE1tR0RBZUV6V09vMW9KK3VNV3BZRXdUNmZDbDh0ZApPaDJ4a0VkOFVwTkdxa0xZaEdIWWtXUHlkRHlQKzNKb1Jna0p4ZGlQTHJvKzdyZ3l3Q0EzMTV5czh4RUN1TW5tCk82c1JBb0dBY0dwTGhVbWNDYkE0Z3lmN3ViOEdEN3NmVDlWTmZENTBGWVZEQTllRGk5MVpweHM2LzNLaS96QnkKTkRCbXhWY0JxbFZPOVY4UEpkUERjdVN0MzdKaWxYT2h3N2lOcGwwbnJRdlAwU2w4OEFzYnBlSk0rME9SMHMxbApNcDNjTkZkTEFBd3pDL3RZUGFyWjJEMjFMSmJzMFloMWFISEljVEpnNmN2RDl5TVhub289Ci0tLS0tRU5EIFJTQSBQUklWQVRFIEtFWS0tLS0tCg==
```

#### 5. 创建 kube-proxy kubeconfig

```bash
# 设置集群参数
$ kubectl config set-cluster kubernetes --certificate-authority=/data/apps/kubernetes/pki/ca.pem --embed-certs=true --server=${KUBE_APISERVER} --kubeconfig=kube-proxy.kubeconfig
# 设置客户端认证参数
$ kubectl config set-credentials kube-proxy --client-certificate=/data/apps/kubernetes/pki/kube-proxy.pem --client-key=/data/apps/kubernetes/pki/kube-proxy-key.pem  --embed-certs=true --kubeconfig=kube-proxy.kubeconfig
# 设置上下文参数
$ kubectl config set-context default --cluster=kubernetes --user=kube-proxy  --kubeconfig=kube-proxy.kubeconfig
# 设置当前使用的上下文
$ kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig

$ cat kube-proxy.kubeconfig 
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUR3akNDQXFxZ0F3SUJBZ0lVQnpjeEhPNE90dzFVVWtPaitya2VlRlhZMFNNd0RRWUpLb1pJaHZjTkFRRUwKQlFBd1p6RUxNQWtHQTFVRUJoTUNRMDR4RVRBUEJnTlZCQWdUQ0ZOb1lXNW5TR0ZwTVJFd0R3WURWUVFIRXdoVAphR0Z1WjBoaGFURU1NQW9HQTFVRUNoTURhemh6TVE4d0RRWURWUVFMRXdaVGVYTjBaVzB4RXpBUkJnTlZCQU1UCkNtdDFZbVZ5Ym1WMFpYTXdIaGNOTVRrd09ESXhNRGMxTWpBd1doY05Namt3T0RFNE1EYzFNakF3V2pCbk1Rc3cKQ1FZRFZRUUdFd0pEVGpFUk1BOEdBMVVFQ0JNSVUyaGhibWRJWVdreEVUQVBCZ05WQkFjVENGTm9ZVzVuU0dGcApNUXd3Q2dZRFZRUUtFd05yT0hNeER6QU5CZ05WQkFzVEJsTjVjM1JsYlRFVE1CRUdBMVVFQXhNS2EzVmlaWEp1ClpYUmxjekNDQVNJd0RRWUpLb1pJaHZjTkFRRUJCUUFEZ2dFUEFEQ0NBUW9DZ2dFQkFLTWdIRE5JeHhmQnJPRzEKYkRYUWY4TEN0MTNIQUN3N2t4TkMwRExZazFoZUN0SWdWU0ppbVphTU13eldqYWxWT1ZlODNSOFdKTGNBUWpXZwpmMkR3Vy9iZXpjSTYzSThWQWtoT052VUxsTzRaaGJuSE0veWg2cmdqZHNxTUxiV29FR2MxTzNOWm02T2hMU1ZaCk5tN1lCejZiV09sdmFoeFR4QUpsdlJHTFJyMXZhU2cyc3hmOU52eWlLRGpORE5NWDUxUS93aFRoK3Nmc28xblQKMG5DSzRRc2RrSzNhR2s4RGRRY3piYWpuTmdiQURzdHRKcENFZk4yL2JWK3JoNVZRcWFzUTlabVFYQXF3TmN1SAphNU8yL1hSRFdHV2tnZlZPRnJBT3NybDRwOGFIaE9CaXV1Z2F6NFhMOTFHNkd2RUhJRE1taUhZMXpNdXVaOFp4Ck5GUEQ3NkVDQXdFQUFhTm1NR1F3RGdZRFZSMFBBUUgvQkFRREFnRUdNQklHQTFVZEV3RUIvd1FJTUFZQkFmOEMKQVFJd0hRWURWUjBPQkJZRUZLc3N5WHZvNnVNaTJTbERpTVVOcTgwL05rMFdNQjhHQTFVZEl3UVlNQmFBRktzcwp5WHZvNnVNaTJTbERpTVVOcTgwL05rMFdNQTBHQ1NxR1NJYjNEUUVCQ3dVQUE0SUJBUUJlZ0ZsN3FjN3RPYzRwCnhuWVJKdmQ5QlpFWDZhQlF3WjJuYkR4M1RIa1ZLeG4wZmo3LzYzK0JoUnpjcXBuN3N4MDFEWXg5YzB1MDUzR1oKN2ZMbGZEZVkvdzcwa1FyNkNrc3p6QkRtY1Z3cys3Nnd3engxYUFlWmRlWktDUXNWUTdvaWZ5MGlxNDNOZ1dYbwpSc2c5VlE1L0FwYzltaGhJcThhRU9lL2EzaXowYjhRSmdSOE1yMkRldWcxVGZSS2x0YytqMFdFUm9TeUEweUN4CnN0dkkwaG1pSDdmQkhWc24rSThUYk9HKzRpWGFSMS9qejkyVTB1Qlc4NGR2ak53WTd2ellNazRYMmpNN0pLd24KczgycExEZ3BGRGduZTJ5WDIrZGRKQ1dyaGQ3VzFRTDlHR3NiZk1kY0dUdHV4MzVSQVEyRytZb2ZzNldKWVg4agoyaXdMeU94OAotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    server: https://10.0.0.17:8443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kube-proxy
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: kube-proxy
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUQ4RENDQXRpZ0F3SUJBZ0lVV0NqSlN3NFk0QnZoTWVuRFFETWZRaURYQURzd0RRWUpLb1pJaHZjTkFRRUwKQlFBd1p6RUxNQWtHQTFVRUJoTUNRMDR4RVRBUEJnTlZCQWdUQ0ZOb1lXNW5TR0ZwTVJFd0R3WURWUVFIRXdoVAphR0Z1WjBoaGFURU1NQW9HQTFVRUNoTURhemh6TVE4d0RRWURWUVFMRXdaVGVYTjBaVzB4RXpBUkJnTlZCQU1UCkNtdDFZbVZ5Ym1WMFpYTXdIaGNOTVRrd09ESXhNRGd3TmpBd1doY05Namt3T0RFNE1EZ3dOakF3V2pCOE1Rc3cKQ1FZRFZRUUdFd0pEVGpFUk1BOEdBMVVFQ0JNSVUyaGhibWRJWVdreEVUQVBCZ05WQkFjVENGTm9ZVzVuU0dGcApNUm93R0FZRFZRUUtFeEZ6ZVhOMFpXMDZhM1ZpWlMxd2NtOTRlVEVQTUEwR0ExVUVDeE1HVTNsemRHVnRNUm93CkdBWURWUVFERXhGemVYTjBaVzA2YTNWaVpTMXdjbTk0ZVRDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVAKQURDQ0FRb0NnZ0VCQUxFUXlTbzNmVUlvc1JtV1B6Y2daR3lXdzNuc0JtM2NqdTJzN0gwRVBvajROZ2lPdlZhUwo3VW9IdTZlTnB0S1pTa1ZkbTJEUWpoSkpRL2ppV05EN29Ic20vWFUzWUg5Ym5PVEZWVndCbFJYL05VWHVYdWQ3CjhKRDFueEhRQmUvNElDWjNiY3BJcUFjMVhUMGRVOUYwZXpyeEFHY05KdFRxSDIwL2VaYnFOUUNFYXFhNnZVT0kKMGd5OExxVndORnpKUGh2UGZmN2ZlZ3RKdDJ3QW5YOTBSZ3hrem5yOHVpV1puTTNSd3BDRjZXWUp3bDJTWTlwaAo1cTUwSVMxN3RodzhxSFExSDVORXROTGhMWTU1bll5TEZhMUhETFc1UTNLRks0VTc4YmJoOXVhVnRHRlMxaUFtCmQ1NERPZStTdjN3STF1eGNyQk9UM0tZUURhdGt2ZU8xanpNQ0F3RUFBYU4vTUgwd0RnWURWUjBQQVFIL0JBUUQKQWdXZ01CMEdBMVVkSlFRV01CUUdDQ3NHQVFVRkJ3TUJCZ2dyQmdFRkJRY0RBakFNQmdOVkhSTUJBZjhFQWpBQQpNQjBHQTFVZERnUVdCQlIrUW5hZEgwNjlwcnlCTVBFQWl1NEQwT1UxZlRBZkJnTlZIU01FR0RBV2dCU3JMTWw3CjZPcmpJdGtwUTRqRkRhdk5QelpORmpBTkJna3Foa2lHOXcwQkFRc0ZBQU9DQVFFQVhOWG9EMFo1aGlyWitXWGgKMTQrcTE5WEZna01CdnJnTTJKVzQzMForbWlUbzdXMG0rUVJUWE40NmxMRjBIRUhlU3Yxc1lSNlpjVlRueTVQVQpMcmRPY0pBZldyTzA1RHJSWVNUREJrM3lzRnNaZGM0dWFzWUFBOUUrL1N0YndiVGVnNlFySHNjK3VJZ3V2N2lFCnVPSUh3bVFyQTQyUnoxY2JUazBhSWRzQmtLbGpSemVqYjBMRzhrVGovSUNIR291NW5VbWFxY3dUWWQ1SDA2a2cKMEJieWVJc05PTTBWcG9CTnptSjhFaW9SVTJyUWJYL09XQ2dVdVZjNFRTM25PN3hNMzArME5tQTN1WU1zeUM4cgpRMmhmKy9XdlJ5dEtqa0hOeVZ6U2ZkcVlCQkQzVUpFbkhIdVBRR2s5OU1wdmpJdndLM3ZQcFFLSmw0dE95YmpNCkV3QkFidz09Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcEFJQkFBS0NBUUVBc1JESktqZDlRaWl4R1pZL055QmtiSmJEZWV3R2JkeU83YXpzZlFRK2lQZzJDSTY5ClZwTHRTZ2U3cDQybTBwbEtSVjJiWU5DT0VrbEQrT0pZMFB1Z2V5YjlkVGRnZjF1YzVNVlZYQUdWRmY4MVJlNWUKNTN2d2tQV2ZFZEFGNy9nZ0puZHR5a2lvQnpWZFBSMVQwWFI3T3ZFQVp3MG0xT29mYlQ5NWx1bzFBSVJxcHJxOQpRNGpTREx3dXBYQTBYTWsrRzg5OS90OTZDMG0zYkFDZGYzUkdER1RPZXZ5NkpabWN6ZEhDa0lYcFpnbkNYWkpqCjJtSG1yblFoTFh1MkhEeW9kRFVmazBTMDB1RXRqbm1kaklzVnJVY010YmxEY29VcmhUdnh0dUgyNXBXMFlWTFcKSUNaM25nTTU3NUsvZkFqVzdGeXNFNVBjcGhBTnEyUzk0N1dQTXdJREFRQUJBb0lCQURBRFdEa2RZTmJPeCs4agpRYk1HRXBVcmNJZ2dDMEpCRzNTeGZsTU1FcFQ3a1ZOU3VWNi9hcDYzYUJnd0hmdGZXN2RoZ1orSURlNUJkYkFJCldJTWFxRktjcVAvZTYwaTlvOWFZOStPQi9sWS9wTWQ0c3IxY2EwZ3pnbFhITGNUN2FHUmw0QnlKQlI4blJrZ3IKS3E1U1FwUWlBN1R0NlFpMUQ1NkZKc2hZYTlUZW41WGJ2bW5IN1lvTGxCV3l6SDNvY2pyM0ZzU3g1ZDVvdUYvVgpLZTl1b0d2VVE0UjdKR1ZOeG5KUkxZZmEvVmVSWWdmdWNwQ2hmdHdzaU9uTXd4Z1NXczFWWXV2T0VqSjJ5QVpKCjAvdnRaOTB6ZlVzclNXRm5sUkI0V2JvcHc0QWFQQmI1QVQvRmNkbWRyZ3g3VjRCNkduYlBJSHNTTjcxUWtFa0EKOVlPR21Pa0NnWUVBNE5zWVdmaWpHczNrVGxaUEpBbG9udmhWN2U0ZW1ibXBaaUlDNHZNRE1KdUhpZnd5WjZCdQpPM3k5VTkzcTFaWE9GQmkrWlJKSm9adjNKOE9TSzdCZWtWcGdLdW5IL210bHFPckp3S1R0bzJMdDJhcVlxdTFnClJwa0ZYMnQ0elJBRHRrMG5yWXVNY3dPeXJRZHZVSW5MUGlZaFJNNDkwZ2djT3FZdW5LSjZBUWNDZ1lFQXlaY20KZnlqZDNEWGFRRkZWdWo0aFZoaklDSWI3N2o2YVBkVnFCS1JSWk03K2pieUpnNGtnNVlPOEpMSTVScElCWDRKdQorZzQwbVMwZVJYU2JGd1YzZ3JNMFZLK3lMNGhGQ3o1SjhLd3Z6eXNWVzFjdUVrZWVhWVpHSUdXNCtZWXBvdWRICm15M3NkMUFYQlF2WUVHZlduTjNrU2dvTUpON2I5YzBycWYvSmNYVUNnWUVBaVhjb29naVJucGQxRmpkSjF0d3gKcTg1aXFqMURVL1Bmam1NSXBMcXdub3pYQmhLNnRnT3NvSTJZS2Flb0k3K2I1MGxoVE9VclFyUFpHK1JDZnBjcQptVzVKRUxNdjQyakJFODNHWGhIMmZrYkM1cW1YQUJoekhYWDdoT1J0UytDWWhHRVMrdFF2bnprSmlTTGNlTDVsCkZLKzI4eHVyUzdaTm04VnhCYTJITFEwQ2dZRUF1cHhFRTdSRjVFS3B2WjVOS0hHNU5GVU9YdTV0cWphalc1Z0MKWXplazdSZThob0pBSGRaRDhKS0pDTU0reC9nQ2MySnZ6dVIxaGxKQTBuVEYySUxFQmVaVURBejBlcEcvc0UvQgo3SnZJU2hPTTJwZ1NXdk9YVGdIeFNxNC9sQ1RBeUQ4bWh4ejA1K0hvM1ZBQWUvZFRzTFNyVG1xTW9WajM3MHMxCkgxSmNMTDBDZ1lBdUdCdlBrK0FMbnpQRGpUd0J2Nnl6VlB2Y2J2d0gxRStrZUU5RC9IaHluOXBLeHQ1bklPWmIKYzZ2ZDVSQXlLOC9NWlRXRzVhQ3RUT0RvZjRhd0taRjI4dTBFU3dORXFwbWpvN0kwMWhtR05sMmlTNjBJNUxVcQo2Y0lnSXpQTzRKZU9mRlR4V2F1akZoSUllUDVjb3JHclQvWlN0dDAwN1VGQVlnRG04UHdoK1E9PQotLS0tLUVORCBSU0EgUFJJVkFURSBLRVktLS0tLQo=
```

#### 6. 创建 admin kubeconfig

```bash
# 设置集群参数
$ kubectl config set-cluster kubernetes --certificate-authority=/data/apps/kubernetes/pki/ca.pem --embed-certs=true --server=${KUBE_APISERVER} --kubeconfig=admin.conf
# 设置客户端认证参数
$ kubectl config set-credentials admin --client-certificate=/data/apps/kubernetes/pki/admin.pem --client-key=/data/apps/kubernetes/pki/admin-key.pem --embed-certs=true --kubeconfig=admin.conf
# 设置上下文参数
$ kubectl config set-context default --cluster=kubernetes --user=admin --kubeconfig=admin.conf
# 设置当前使用的上下文
$ kubectl config use-context default --kubeconfig=admin.conf

$ cat admin.conf  
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUR3akNDQXFxZ0F3SUJBZ0lVQnpjeEhPNE90dzFVVWtPaitya2VlRlhZMFNNd0RRWUpLb1pJaHZjTkFRRUwKQlFBd1p6RUxNQWtHQTFVRUJoTUNRMDR4RVRBUEJnTlZCQWdUQ0ZOb1lXNW5TR0ZwTVJFd0R3WURWUVFIRXdoVAphR0Z1WjBoaGFURU1NQW9HQTFVRUNoTURhemh6TVE4d0RRWURWUVFMRXdaVGVYTjBaVzB4RXpBUkJnTlZCQU1UCkNtdDFZbVZ5Ym1WMFpYTXdIaGNOTVRrd09ESXhNRGMxTWpBd1doY05Namt3T0RFNE1EYzFNakF3V2pCbk1Rc3cKQ1FZRFZRUUdFd0pEVGpFUk1BOEdBMVVFQ0JNSVUyaGhibWRJWVdreEVUQVBCZ05WQkFjVENGTm9ZVzVuU0dGcApNUXd3Q2dZRFZRUUtFd05yT0hNeER6QU5CZ05WQkFzVEJsTjVjM1JsYlRFVE1CRUdBMVVFQXhNS2EzVmlaWEp1ClpYUmxjekNDQVNJd0RRWUpLb1pJaHZjTkFRRUJCUUFEZ2dFUEFEQ0NBUW9DZ2dFQkFLTWdIRE5JeHhmQnJPRzEKYkRYUWY4TEN0MTNIQUN3N2t4TkMwRExZazFoZUN0SWdWU0ppbVphTU13eldqYWxWT1ZlODNSOFdKTGNBUWpXZwpmMkR3Vy9iZXpjSTYzSThWQWtoT052VUxsTzRaaGJuSE0veWg2cmdqZHNxTUxiV29FR2MxTzNOWm02T2hMU1ZaCk5tN1lCejZiV09sdmFoeFR4QUpsdlJHTFJyMXZhU2cyc3hmOU52eWlLRGpORE5NWDUxUS93aFRoK3Nmc28xblQKMG5DSzRRc2RrSzNhR2s4RGRRY3piYWpuTmdiQURzdHRKcENFZk4yL2JWK3JoNVZRcWFzUTlabVFYQXF3TmN1SAphNU8yL1hSRFdHV2tnZlZPRnJBT3NybDRwOGFIaE9CaXV1Z2F6NFhMOTFHNkd2RUhJRE1taUhZMXpNdXVaOFp4Ck5GUEQ3NkVDQXdFQUFhTm1NR1F3RGdZRFZSMFBBUUgvQkFRREFnRUdNQklHQTFVZEV3RUIvd1FJTUFZQkFmOEMKQVFJd0hRWURWUjBPQkJZRUZLc3N5WHZvNnVNaTJTbERpTVVOcTgwL05rMFdNQjhHQTFVZEl3UVlNQmFBRktzcwp5WHZvNnVNaTJTbERpTVVOcTgwL05rMFdNQTBHQ1NxR1NJYjNEUUVCQ3dVQUE0SUJBUUJlZ0ZsN3FjN3RPYzRwCnhuWVJKdmQ5QlpFWDZhQlF3WjJuYkR4M1RIa1ZLeG4wZmo3LzYzK0JoUnpjcXBuN3N4MDFEWXg5YzB1MDUzR1oKN2ZMbGZEZVkvdzcwa1FyNkNrc3p6QkRtY1Z3cys3Nnd3engxYUFlWmRlWktDUXNWUTdvaWZ5MGlxNDNOZ1dYbwpSc2c5VlE1L0FwYzltaGhJcThhRU9lL2EzaXowYjhRSmdSOE1yMkRldWcxVGZSS2x0YytqMFdFUm9TeUEweUN4CnN0dkkwaG1pSDdmQkhWc24rSThUYk9HKzRpWGFSMS9qejkyVTB1Qlc4NGR2ak53WTd2ellNazRYMmpNN0pLd24KczgycExEZ3BGRGduZTJ5WDIrZGRKQ1dyaGQ3VzFRTDlHR3NiZk1kY0dUdHV4MzVSQVEyRytZb2ZzNldKWVg4agoyaXdMeU94OAotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    server: https://10.0.0.17:8443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: admin
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: admin
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUQ0VENDQXNtZ0F3SUJBZ0lVVmtIUDR0aWxsWERyRlFDdzl3NjYvZURheCtJd0RRWUpLb1pJaHZjTkFRRUwKQlFBd1p6RUxNQWtHQTFVRUJoTUNRMDR4RVRBUEJnTlZCQWdUQ0ZOb1lXNW5TR0ZwTVJFd0R3WURWUVFIRXdoVAphR0Z1WjBoaGFURU1NQW9HQTFVRUNoTURhemh6TVE4d0RRWURWUVFMRXdaVGVYTjBaVzB4RXpBUkJnTlZCQU1UCkNtdDFZbVZ5Ym1WMFpYTXdIaGNOTVRrd09ESXhNRGd3T1RBd1doY05Namt3T0RFNE1EZ3dPVEF3V2pCdE1Rc3cKQ1FZRFZRUUdFd0pEVGpFUk1BOEdBMVVFQ0JNSVUyaGhibWRJWVdreEVUQVBCZ05WQkFjVENGTm9ZVzVuU0dGcApNUmN3RlFZRFZRUUtFdzV6ZVhOMFpXMDZiV0Z6ZEdWeWN6RVBNQTBHQTFVRUN4TUdVM2x6ZEdWdE1RNHdEQVlEClZRUURFd1ZoWkcxcGJqQ0NBU0l3RFFZSktvWklodmNOQVFFQkJRQURnZ0VQQURDQ0FRb0NnZ0VCQU1Jbk1HamMKd0VRRmtzYzVQSE1GTk5WMzdMYTBMZWhtdmk1ZittaXBJY2NSVmNhdHRHclB4bzU5dHhMdUZJNWF4K0FUbFJGYwpGTFRWbDhEZi8yN0x0WDBWN0NHRFhiOVlHR0xvRVhzTjMzdnJnUndGWXJsdC83NFR5bDdTdFFLNjBwV0pXc1RnCkdOVmIvRFF0QTRGMXRjd1U2Z09PWWwwOGdIdDF4S3dwbXMyaW12cGY2NnU0aUh6Q2VlOUhCRW9LUkRtdWpyUkEKekltOUdVMjd2VktIdjRWNThuV3Z2K3pyVnJrYnROTXJrNENvWlc5a3dmZzg5d2VjaCt5ZlNVR0ZGZWs1YU5tMgpSZkRtU1NjdzcxWElXY0d2WXFaMisxQzZ5RG1rRFZPRHNRdTIzUVpNcFlqYklXMWt5eTFJb1VIR0JTNjE0YW1kCjFYbS9CN01WbGg1d0hOTUNBd0VBQWFOL01IMHdEZ1lEVlIwUEFRSC9CQVFEQWdXZ01CMEdBMVVkSlFRV01CUUcKQ0NzR0FRVUZCd01CQmdnckJnRUZCUWNEQWpBTUJnTlZIUk1CQWY4RUFqQUFNQjBHQTFVZERnUVdCQlRFajVlWgpqZ2RvQklaZk5ubC9SOHdja3V0cVVUQWZCZ05WSFNNRUdEQVdnQlNyTE1sNzZPcmpJdGtwUTRqRkRhdk5QelpOCkZqQU5CZ2txaGtpRzl3MEJBUXNGQUFPQ0FRRUFsam0wQS9wN2d2aUMrUCtVUFJsdUs1VEpVQTIzUFdaMDRTTEUKckdlMGNHMEhMMVZxT08wNTNKQncrbXdNYmQwOURpZDh0UThmalZiMTM3K1ZLSmVpWFdJRFNPUXhJanNHZDArTApOQUFKK3FMelk3TzFUKzFhYWZLL2JmaGZ5NlpRNGVUTlg3bm5GdENaeDhVS083NDU3YU0xeEwzWFdNMzNGNnpRCjFaRERCVmhBMEFBVGFGNU1RN2VVbHV0VFVQWFpsVlJtUUkxRnMvTm54UVlQd2ZjUDd6ZnF2NlBZN05aVkZGcnkKQXc3NjJmM09JYlc3UUZIdFp6M3NKQmNJb1RJa2Q2anRnM0xEa2x0MU82S3NpK09uVnRiSE54UXZqRTd5bkFydQphZXVIR2xmZkI2WTlDVjVnMFRNY3pWSUlETFhOOEdpZ2orUmloRUtRYmNRa1BtZXdNUT09Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcEFJQkFBS0NBUUVBd2ljd2FOekFSQVdTeHprOGN3VTAxWGZzdHJRdDZHYStMbC82YUtraHh4RlZ4cTIwCmFzL0dqbjIzRXU0VWpsckg0Qk9WRVZ3VXROV1h3Ti8vYnN1MWZSWHNJWU5kdjFnWVl1Z1JldzNmZSt1QkhBVmkKdVczL3ZoUEtYdEsxQXJyU2xZbGF4T0FZMVZ2OE5DMERnWFcxekJUcUE0NWlYVHlBZTNYRXJDbWF6YUthK2wvcgpxN2lJZk1KNTcwY0VTZ3BFT2E2T3RFRE1pYjBaVGJ1OVVvZS9oWG55ZGErLzdPdFd1UnUwMHl1VGdLaGxiMlRCCitEejNCNXlIN0o5SlFZVVY2VGxvMmJaRjhPWkpKekR2VmNoWndhOWlwbmI3VUxySU9hUU5VNE94QzdiZEJreWwKaU5zaGJXVExMVWloUWNZRkxyWGhxWjNWZWI4SHN4V1dIbkFjMHdJREFRQUJBb0lCQVFDOEloOWRyWE05TnExWgpFVlJMSEdOcTZ1OWN4MUdvM2s0eFA5MjFKeGJOQURZKzlEbGNPd1ByTlZSK0ttZU8zZGJLZ2c4enFDZUVaMmpLCmhBUFBSK1FRVm5yZXFwM2YrU3lBUXVJVmZJYnZYSEJhUjdtM2R5aVc5alJtR0FWQXBPbkQ3em9laGd4cVN0MGoKYmU3MHRxdzRHcGY4WkM5YXEzTFFyM2lwWHhOYmFDSUJFMk80NG5wQXNaTzQzRGp0S2c2cTczdXIvRk5xQmpnNQpFMHQwaSsweG54RjdTM2l2U2ZiM3hCTjZlZmtmQXdtYnBJamtHWVZqZjIrcCtsMjlsTTJMWmJaV0svdEtxOXFLCjhneGxjQkQwQnRURldWUlVGS201Nk5PMVk3bFNhVGh4bGxraVNVZnBsVGZlVXlrR2pIcnhxUlorbitWcUF5bWIKTEFvY3lNOFJBb0dCQU5UdW9yN1o4aXovdHpWdldaRGhuWXh1NFdlOXJkQkNaUnd2TitjdXlNRGY4cmp3V2xZMQpYczV4eTdyTFhIcEYrZlFyQzBMcko3SlJmaEljczR2NUZuRS9nWExWb1c4TEFycjdKT0hOVWdiTEFlb1VSYmlSCnpRS3Z0c1dDeEwvb3lpaFErV2VGQ1FmYlZMRnFPRmtxV3RIdnpacEJaWG1uOW85NUt0c0d5aXNwQW9HQkFPbHMKTWdjVkhGL0YzU2xMcWQxc0hQOStxdjFTd0VnK3Jac2Z4RCtnenMxUW1qSHJyR0cyUjlIaWkzdmtoOEtwNjlDTQptWTdCQjNuTVJGazkxc0VUbE0xRnNpcE5BUStFYmpjRzM0b0p2NUNpM3NzVGs2U1h6RlZ6RllCS2xpUy9QMTJrCnlEUTRlM3NzaTVIZFhlN0xRbnF3a3dqMkZuMFJxYkJvVFJRT29vT2JBb0dBV0NMNjNGS3NTbklDYkt6TmZ3blUKUThlMXAxSTgrdUl3cGV6cGo5aXVvaDlRZ2JxRE9nSFhYMDU5RExHV2NzbzZQeFgrRUZIejJYeWYyWEZsNUQ5VApTY2NHbHZqVVhIbExSUWdsYVEycXNVTWdaTHJGYlROMGozTWFEVUVtbldVSElJNzczUnlVODFxWEFPUzl0REt5CjZ3aitxcVg5RWRFelhvbkI4bTBxQzVrQ2dZRUEzcFN5VzdpUXR1NjVSckNFeU1SWUhuV04zVU8wWU8rTG9mazMKcktqTnFsQnF5S0YvWGlsdjhMN0MzUi85S08zWkZLT05wZWVCRm01bTJtWXlTeWc5NDBQTGNiUytCeXJ6NGZybQoyLzBSczN6clQrQmFFRUJEczFPck5BdHJncHp2Y244My9UdkMyNkNOY2trUlVpeDJOd0g3SXpkdUdGTG9hWFA3CjA5MWtzSE1DZ1lBU25kRFdsZVdVTjZ3eGNwWDNTVGpkSnhVYUhSSzFKakZLclU1Zmc0NkRVam5KTkwvRG94N0IKQzZzLzBKMU05d3ZGSHl0OC9rQXozVm9pNGRDWmF1WWF0OHJhdzhuSitYYmFWNmZrMTJ4S2NkZ3lMUGw5S1N2QQpkQjJYakFzNTJ5MTRCenJVN2xkVlV0ZSs2UXpvcjNScXJkMFpEYTRUUmZVYVdkc0FCcWpYWlE9PQotLS0tLUVORCBSU0EgUFJJVkFURSBLRVktLS0tLQo=
```

#### 7. 分发kubeconfig文件

*node节点:*  *只需要kubelet/kube-proxy*

```bash
# to k8s4
$ scp /root/software/kubeconfig/kubelet-bootstrap.kubeconfig /root/software/kubeconfig/kube-proxy.kubeconfig root@k8s4:/data/apps/kubernetes/etc/
# to k8s5
$ scp /root/software/kubeconfig/kubelet-bootstrap.kubeconfig /root/software/kubeconfig/kube-proxy.kubeconfig root@k8s5:/data/apps/kubernetes/etc/
# to k8s6
$ scp /root/software/kubeconfig/kubelet-bootstrap.kubeconfig /root/software/kubeconfig/kube-proxy.kubeconfig root@k8s6:/data/apps/kubernetes/etc/
```

*master节点:*

```bash
$ cd /root/software/kubeconfig/
# to k8s1 
$ cp admin.conf kube-controller-manager.kubeconfig  kube-proxy.kubeconfig  kube-scheduler.kubeconfig kubelet-bootstrap.kubeconfig token.csv /data/apps/kubernetes/etc/
# to k8s2
$ scp admin.conf kube-controller-manager.kubeconfig kube-proxy.kubeconfig kube-scheduler.kubeconfig kubelet-bootstrap.kubeconfig token.csv root@k8s2:/data/apps/kubernetes/etc/
# to k8s3
$ scp admin.conf kube-controller-manager.kubeconfig kube-proxy.kubeconfig kube-scheduler.kubeconfig kubelet-bootstrap.kubeconfig token.csv root@k8s3:/data/apps/kubernetes/etc/
```

### 配置kube-apiserver组件

*kube-apiserver是Kubernetes最重要的核心组件之一,主要提供以下的功能:*

* *提供集群管理的 REST API 接口,包括认证授权、数据校验以及集群状态变更等*

* *提供其他模块之间的数据交互和通信的枢纽(其他模块通过 API Server 查询或修改数据,只有 API Server才直接操作etcd)*

#### 1. 配置sa证书

```bash
$ cd /data/apps/kubernetes/pki/
$ openssl genrsa -out /data/apps/kubernetes/pki/sa.key 2048
$ openssl rsa -in /data/apps/kubernetes/pki/sa.key -pubout -out /data/apps/kubernetes/pki/sa.pub
$ ls sa.*
sa.key  sa.pub
```

#### 2. 分发sa证书到其他kube-paiserver节点

```bash
# to k8s2
$ scp /data/apps/kubernetes/pki/sa.* root@k8s2:/data/apps/kubernetes/pki/
# to k8s3
$ scp /data/apps/kubernetes/pki/sa.* root@k8s3:/data/apps/kubernetes/pki/
```

#### 3. 各kube-paiserver节点创建系统服务文件

```bash
$ cat > /lib/systemd/system/kube-apiserver.service  << EOF 
[Unit] 
Description=Kubernetes API Service 
Documentation=https://github.com/kubernetes/kubernetes 
After=network.target 

[Service] 
EnvironmentFile=-/data/apps/kubernetes/etc/kube-apiserver.conf 
ExecStart=/usr/bin/kube-apiserver $KUBE_LOGTOSTDERR $KUBE_LOG_LEVEL $KUBE_ETCD_ARGS $KUBE_API_ADDRESS $KUBE_SERVICE_ADDRESSES $KUBE_ADMISSION_CONTROL $KUBE_APISERVER_ARGS 
Restart=on-failure 
Type=notify 
LimitNOFILE=65536 

[Install] 
WantedBy=multi-user.target
EOF
```

#### 4. 各kube-paiserver节点创建配置文件

*KUBE_API_ADDRESS配置为各节点ip*

```bash
$ cat > /data/apps/kubernetes/etc/kube-apiserver.conf << EOF
KUBE_API_ADDRESS="--advertise-address=10.0.100.178" 

KUBE_ETCD_ARGS="--etcd-servers=https://10.0.100.178:2379,https://10.0.100.147:2379,https://10.0.101.78:2379 --etcd-cafile=/data/apps/etcd/ssl/etcd-ca.pem --etcd-certfile=/data/apps/etcd/ssl/etcd.pem --etcd-keyfile=/data/apps/etcd/ssl/etcd-key.pem" 

KUBE_LOGTOSTDERR="--logtostderr=false" 
KUBE_LOG_LEVEL=" --log-dir=/data/apps/kubernetes/log/ --v=2 --audit-log-maxage=7 --audit-log-maxbackup=10 --audit-log-maxsize=100 --audit-log-path=/data/apps/kubernetes/log/kubernetes.audit --event-ttl=12h" 

KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.99.0.0/16" 
KUBE_ADMISSION_CONTROL="--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota" 
KUBE_APISERVER_ARGS="--storage-backend=etcd3 --apiserver-count=3 --endpoint-reconciler-type=lease --runtime-config=api/all,settings.k8s.io/v1alpha1=true,admissionregistration.k8s.io/v1beta1,rbac.authorization.k8s.io/v1beta1 --enable-admission-plugins=NodeRestriction --allow-privileged=true --authorization-mode=Node,RBAC --enable-bootstrap-token-auth=true --token-auth-file=/data/apps/kubernetes/etc/token.csv --service-node-port-range=30000-40000 --tls-cert-file=/data/apps/kubernetes/pki/kube-apiserver.pem --tls-private-key-file=/data/apps/kubernetes/pki/kube-apiserver-key.pem --client-ca-file=/data/apps/kubernetes/pki/ca.pem --service-account-key-file=/data/apps/kubernetes/pki/sa.pub --secure-port=6443 --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname --anonymous-auth=false --kubelet-client-certificate=/data/apps/kubernetes/pki/admin.pem --kubelet-client-key=/data/apps/kubernetes/pki/admin-key.pem"
EOF
```

#### 5. 启动各节点kube-apiserver服务

```bash
$ systemctl daemon-reload
$ systemctl enable kube-apiserver
$ systemctl start kube-apiserver
$ systemctl status kube-apiserver
● kube-apiserver.service - Kubernetes API Service
   Loaded: loaded (/lib/systemd/system/kube-apiserver.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2019-08-23 09:38:23 UTC; 4 days ago
     Docs: https://github.com/kubernetes/kubernetes
 Main PID: 4270 (kube-apiserver)
    Tasks: 13 (limit: 4704)
   CGroup: /system.slice/kube-apiserver.service
           └─4270 /usr/bin/kube-apiserver --logtostderr=false --log-dir=/data/apps/kubernetes/log/ --v=2 --audit-log-maxage=7 --audit-log-maxbackup=10 --audit-log-maxsize=100 --audit-log-path=/data/apps/kuberne..
....
```

### kube-apiserver配置高可用

**Note:**  *在配置HA k8s集群前需要为kube-apiserver创建LoadBalance, 此处选取haproxy + keepalived 搭建, 云环境可以直接使用云服务商的SLB 服务*

#### 1. LB节点安装haproxy , keepalived

```bash
$ sudo apt-get update
$ sudo apt-get haproxy keepalived
```

#### 2. 修改haproxy 配置文件/etc/haproxy/haproxy.cfg

```bash
$ cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
$ > /etc/haproxy/haproxy.cfg
$ cat >  /etc/haproxy/haproxy.cfg << EOF
global
  log 127.0.0.1 local0 err
  maxconn 50000
  daemon
  nbproc 1
  pidfile haproxy.pid

defaults
  mode http
  log 127.0.0.1 local0 err
  maxconn 50000
  retries 3
  timeout connect 5s
  timeout client 30s
  timeout server 30s
  timeout check 2s

listen admin_stats
  mode http
  bind 0.0.0.0:1080
  log 127.0.0.1 local0 err
  stats refresh 30s
  stats uri     /haproxy-status
  stats realm   Haproxy\ Statistics
  stats auth    admin:admin
  stats hide-version
  stats admin if TRUE

frontend k8s-https
  bind 0.0.0.0:8443
  mode tcp
  #maxconn 50000
  default_backend k8s-https

backend k8s-https
  mode tcp
  balance roundrobin
  server k8s1 10.0.100.178:6443 weight 1 maxconn 1000 check inter 2000 rise 2 fall 3
  server k8s2 10.0.100.147:6443 weight 1 maxconn 1000 check inter 2000 rise 2 fall 3
  server k8s3 10.0.101.78:6443 weight 1 maxconn 1000 check inter 2000 rise 2 fall 3

EOF
```

#### 3.启动haproxy

```bash
$ systemctl enable haproxy
$ systemctl start haproxy
```

#### 4. 配置 keepalived

```bash
$ cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak
$ > /etc/keepalived/keepalived.conf
$ cat >  /etc/keepalived/keepalived.conf << EOF
! Configuration File for keepalived

global_defs {
   router_id k8s1   #use hostname 
}

vrrp_script chk_haproxy {                            
  script "/etc/keepalived/check_haproxy.sh"
  interval 5
  timeout 2
  fall 3
}

vrrp_instance VI_1 {
    state MASTER
    interface eno1   #interface
    virtual_router_id 88
    priority 210
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 111111
    }
    virtual_ipaddress {
        10.0.0.17   #vip
    }

    track_script {
        chk_haproxy
    }

    notify_master "/etc/keepalived/haproxy_master.sh"

}
EOF

$ cat > /etc/keepalived/haproxy_check.sh << EOF
#!/bin/bash
LOGFILE="/var/log/keepalived-haproxy-state.log"
date >> $LOGFILE
if [ `ps -C haproxy --no-header|wc -l` -eq 0 ];then
    echo "fail: check_haproxy status" >> $LOGFILE
    systemctl stop keepalived
    exit 1
else
    echo "success: check_haproxy status" >> $LOGFILE
    exit 0
fi
EOF

$ cat >  /etc/keepalived/haproxy_master.sh << EOF 
#!/bin/bash
LOGFILE="/var/log/keepalived-haproxy-state.log"
echo "Being Master ..." >> $LOGFILE
EOF

$ chmod a+x /etc/keepalived/haproxy_check.sh /etc/keepalived/haproxy_master.sh
```

**Note:**  *其他LB节点的/etc/keepalived/keepalived.conf文件中state设置为BACKUP，priority值优先级依次降低*

#### 5. 启动 keepalived

```bash
$ systemctl enable keepalived
$ systemctl start keepalived
```

### 配置kube-controller-manager组件

*kube-controller-manager负责维护集群的状态, 比如故障检测、自动扩展、滚动更新等. 在启动时设置leader-elect=true 后, controller manager会使用多节点选主的方式选择主节点. 只有主节点才会调用 StartControllers() 启动所有控制器,而其他从节点则仅执行选主算法.*

#### 1. 各kube-controller-manager节点创建系统服务文件

```bash
$ cat > /lib/systemd/system/kube-controller-manager.service << EOF
[Unit] 
Description=Kubernetes Controller Manager 
Documentation=https://github.com/kubernetes/kubernetes 

[Service] 
EnvironmentFile=-/data/apps/kubernetes/etc/kube-controller-manager.conf 
ExecStart=/usr/bin/kube-controller-manager $KUBE_LOGTOSTDERR $KUBE_LOG_LEVEL $KUBECONFIG $KUBE_CONTROLLER_MANAGER_ARGS 
Restart=always 
RestartSec=10s 
Restart=on-failure 

LimitNOFILE=65536 

[Install] 
WantedBy=multi-user.target
EOF
```

#### 2. 各kube-controller-manager节点创建配置文件

```bash
$ cat > /data/apps/kubernetes/etc/kube-controller-manager.conf << EOF
KUBE_LOGTOSTDERR="--logtostderr=false" 
KUBE_LOG_LEVEL="--v=2 --log-dir=/data/apps/kubernetes/log/"

KUBECONFIG="--kubeconfig=/data/apps/kubernetes/etc/kube-controller-manager.kubeconfig" 
KUBE_CONTROLLER_MANAGER_ARGS="--bind-address=127.0.0.1 --cluster-cidr=10.99.0.0/16 --cluster-name=kubernetes --cluster-signing-cert-file=/data/apps/kubernetes/pki/ca.pem --cluster-signing-key-file=/data/apps/kubernetes/pki/ca-key.pem --service-account-private-key-file=/data/apps/kubernetes/pki/sa.key --root-ca-file=/data/apps/kubernetes/pki/ca.pem --leader-elect=true --use-service-account-credentials=true --node-monitor-grace-period=10s --pod-eviction-timeout=10s --allocate-node-cidrs=true --controllers=*,bootstrapsigner,tokencleaner --horizontal-pod-autoscaler-use-rest-clients=true --experimental-cluster-signing-duration=87600h0m0s --feature-gates=RotateKubeletServerCertificate=true"
EOF
```

#### 3. 启动各kube-controller-manager节点服务

```bash
$ systemctl daemon-reload
$ systemctl enable kube-controller-manager
$ systemctl start kube-controller-manager
$ systemctl status kube-controller-manager
● kube-controller-manager.service - Kubernetes Controller Manager
   Loaded: loaded (/lib/systemd/system/kube-controller-manager.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2019-08-23 08:19:10 UTC; 4 days ago
     Docs: https://github.com/kubernetes/kubernetes
 Main PID: 30650 (kube-controller)
    Tasks: 10 (limit: 4704)
   CGroup: /system.slice/kube-controller-manager.service
           └─30650 /usr/bin/kube-controller-manager --logtostderr=false --v=2 --log-dir=/data/apps/kubernetes/log/ --kubeconfig=/data/apps/kubernetes/etc/kube-controller-manager.kubeconfig --bind-address=127.0.0.1
....
```

### 配置kube-scheduler组件

*kube-scheduler负责分配调度 Pod到集群内的节点上, 它监听kube-apiserver, 查询还未分配 Node的Pod, 然后根据调度策略为这些 Pod分配节点. 按照预定的调度策略将Pod调度到相应的Node上(更新 Pod的NodeName字段).*

#### 1. 各kube-scheduler节点创建系统服务文件

```bash
$ cat > /lib/systemd/system/kube-scheduler.service  << EOF
[Unit] 
Description=Kubernetes Scheduler Plugin 
Documentation=https://github.com/kubernetes/kubernetes

[Service] 
EnvironmentFile=-/data/apps/kubernetes/etc/kube-scheduler.conf 
ExecStart=/usr/bin/kube-scheduler $KUBE_LOGTOSTDERR $KUBE_LOG_LEVEL $KUBECONFIG $KUBE_SCHEDULER_ARGS 
Restart=on-failure 
LimitNOFILE=65536 

[Install] 
WantedBy=multi-user.target
EOF
```

#### 2. 各kube-scheduler节点创建配置文件

```bash
$ cat > /data/apps/kubernetes/etc/kube-scheduler.conf << EOF
KUBE_LOGTOSTDERR="--logtostderr=false" 
KUBE_LOG_LEVEL="--v=2 --log-dir=/data/apps/kubernetes/log/" 
KUBECONFIG="--kubeconfig=/data/apps/kubernetes/etc/kube-scheduler.kubeconfig" 
KUBE_SCHEDULER_ARGS="--leader-elect=true --address=127.0.0.1"
EOF
```

#### 3. 启动各kube-scheduler节点服务

```bash
$ systemctl daemon-reload
$ systemctl enable kube-scheduler
$ systemctl start kube-scheduler
$ systemctl status kube-scheduler
● kube-scheduler.service - Kubernetes Scheduler Plugin
   Loaded: loaded (/lib/systemd/system/kube-scheduler.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2019-08-23 08:30:21 UTC; 4 days ago
     Docs: https://github.com/kubernetes/kubernetes
 Main PID: 4135 (kube-scheduler)
    Tasks: 11 (limit: 4704)
   CGroup: /system.slice/kube-scheduler.service
           └─4135 /usr/bin/kube-scheduler --logtostderr=false --v=2 --log-dir=/data/apps/kubernetes/log/ --kubeconfig=/data/apps/kubernetes/etc/kube-scheduler.kubeconfig --leader-elect=true --address=127.0.0.1
....
```

### 配置kubectl

```bash
# master节点配置
$ rm -rf $HOME/.kube 
$ mkdir -p $HOME/.kube
$ cp /data/apps/kubernetes/etc/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
# 查看各组件状态
$ kubectl get componentstatuses
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok                  
scheduler            Healthy   ok                  
etcd-1               Healthy   {"health":"true"}   
etcd-0               Healthy   {"health":"true"}   
etcd-2               Healthy   {"health":"true"}
```

### 配置各Node节点

*解压node包并将执行文件移到/usr/bin/下*

```bahs
$ cd /root/software
$ wget https://dl.k8s.io/v1.15.3/kubernetes-node-linux-amd64.tar.gz
$ ls kubernetes-node-linux-amd64.tar.gz 
kubernetes-node-linux-amd64.tar.gz
$ tar -xzvf kubernetes-node-linux-amd64.tar.gz
$ mv kubernetes/node/bin/kube* /usr/bin/
$ ls /usr/bin/kube*
/usr/bin/kube-proxy  /usr/bin/kubeadm  /usr/bin/kubectl  /usr/bin/kubelet
```

### 配置kubelet组件

*kubelet负责维持容器的生命周期, 同时也负责Volume(CVI)和网络(CNI)的管理. 每个节点上都运行一个kubelet服务进程, 默认监听10250端口, 接收并执行master发来的指令, 管理Pod及Pod 中容器. 每个kubelet进程会在API Server上注册节点自身信息, 定期向master节点汇报节点的资源使用情况, 并通过 cAdvisor/metric-server监控节点和容器的资源.*

**集群中配置kubelet使用bootstrap**

```bash
# k8s1节点操作
$ kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
```

**Note:**  *所有节点都需要安装kubelet组件*

#### 1. 各kubelet节点创建系统服务文件

```bash
$ cat > /lib/systemd/system/kubelet.service << EOF
[Unit] 
Description=Kubernetes Kubelet Server 
Documentation=https://github.com/kubernetes/kubernetes 
After=docker.service 
Requires=docker.service 

[Service]
EnvironmentFile=-/data/apps/kubernetes/etc/kubelet.conf 
ExecStart=/usr/bin/kubelet $KUBE_LOGTOSTDERR $KUBE_LOG_LEVEL $KUBELET_CONFIG $KUBELET_HOSTNAME $KUBELET_POD_INFRA_CONTAINER $KUBELET_ARGS 
Restart=on-failure 

[Install] 
WantedBy=multi-user.target
EOF
```

#### 2. 各kubelet节点创建配置文件

```bash
# --hostname-override为各节点ip
$ cat > /data/apps/kubernetes/etc/kubelet.conf << EOF
KUBE_LOGTOSTDERR="--logtostderr=false" 
KUBE_LOG_LEVEL="--v=2 --log-dir=/data/apps/kubernetes/log/" 
KUBELET_HOSTNAME="--hostname-override=10.0.101.78" 
KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=k8s.gcr.io/pause:3.1" 

KUBELET_CONFIG="--config=/data/apps/kubernetes/etc/kubelet-config.yml" 
KUBELET_ARGS="--bootstrap-kubeconfig=/data/apps/kubernetes/etc/kubelet-bootstrap.kubeconfig --kubeconfig=/data/apps/kubernetes/etc/kubelet.kubeconfig --cert-dir=/data/apps/kubernetes/pki"
EOF

# address为各节点ip
$ cat > /data/apps/kubernetes/etc/kubelet-config.yml  << EOF
kind: KubeletConfiguration 
apiVersion: kubelet.config.k8s.io/v1beta1 
address: 10.0.101.78 
port: 10250 
cgroupDriver: cgroupfs 
clusterDNS: 
  - 10.99.110.110 
clusterDomain: cluster.local. 
hairpinMode: promiscuous-bridge 
maxPods: 200 
failSwapOn: false
imageGCHighThresholdPercent: 90 
imageGCLowThresholdPercent: 80 
imageMinimumGCAge: 5m0s 
serializeImagePulls: false 
authentication: 
  x509: 
    clientCAFile: /data/apps/kubernetes/pki/ca.pem 
  anonymous: 
    enbaled: false 
  webhook: 
    enbaled: false
EOF
```

#### 3. 启动各kubelet节点服务

```bash
$ systemctl daemon-reload 
$ systemctl enable kubelet 
$ systemctl restart kubelet 
$ systemctl status kubelet
● kubelet.service - Kubernetes Kubelet Server
   Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2019-08-25 07:10:21 UTC; 2 days ago
     Docs: https://github.com/kubernetes/kubernetes
 Main PID: 1528 (kubelet)
    Tasks: 15 (limit: 4704)
   CGroup: /system.slice/kubelet.service
           └─1528 /usr/bin/kubelet --logtostderr=false --v=2 --log-dir=/data/apps/kubernetes/log/ --config=/data/apps/kubernetes/etc/kubelet-config.yml --hostname-override=10.0.101.78 --pod-infra-container-imag..
....
```

### 配置kube-proxy组件

*kube-proxy负责为 Service提供 cluster内部的服务发现和负载均衡. 每个节点上都运行一个 kube-proxy服务, 它监听 API server中 service和 endpoint 的变化情况,并通过 ipvs/iptables等来为服务配置负载均衡(仅支持 TCP和 UDP)*

**Note:** *使用 ipvs模式时, 需要预先在每个节点上加载内核模块 nf_conntrack_ipv4, ip_vs, ip_vs_rr, ip_vs_wrr, ip_vs_sh等*

```bash
# 所有节点执行
$ apt-get install ipset ipvsadm conntrack
$ lsmod |grep ip_vs
ip_vs_sh               16384  0
ip_vs_wrr              16384  0
ip_vs_rr               16384  27
ip_vs                 147456  33 ip_vs_rr,ip_vs_sh,ip_vs_wrr
nf_conntrack          131072  12 xt_conntrack,nf_nat_masquerade_ipv4,nf_conntrack_ipv6,nf_conntrack_ipv4,nf_nat,nf_nat_ipv6,ipt_MASQUERADE,nf_nat_ipv4,xt_nat,openvswitch,nf_conntrack_netlink,ip_vs
libcrc32c              16384  6 nf_conntrack,nf_nat,openvswitch,xfs,raid456,ip_vs
```

**Note:** *所有节点均需要安装kube-proxy组件*

#### 1. 各kube-proxy节点创建系统服务文件

```bash
$ cat > /lib/systemd/system/kube-proxy.service << EOF 
[Unit] 
Description=Kubernetes Kube-Proxy Server 
Documentation=https://github.com/kubernetes/kubernetes 
After=network.target 

[Service] 
EnvironmentFile=-/data/apps/kubernetes/etc/kube-proxy.conf 
ExecStart=/usr/bin/kube-proxy $KUBE_LOGTOSTDERR $KUBE_LOG_LEVEL $KUBECONFIG $KUBE_PROXY_ARGS
Restart=on-failure 
LimitNOFILE=65536 
KillMode=process 

[Install] 
WantedBy=multi-user.target
EOF
```

#### 2. 各kube-proxy节点创建配置文件

```bash
# proxy-mode使用ipvs模式
$ cat > /data/apps/kubernetes/etc/kube-proxy.conf << EOF
KUBE_LOGTOSTDERR="--logtostderr=false" 
KUBE_LOG_LEVEL="--v=2 --log-dir=/data/apps/kubernetes/log/" 
KUBECONFIG="--kubeconfig=/data/apps/kubernetes/etc/kube-proxy.kubeconfig" 
KUBE_PROXY_ARGS="--proxy-mode=ipvs --masquerade-all=true --cluster-cidr=10.99.0.0/16"
EOF
```

#### 3. 启动各kube-proxy节点服务

```bash
$ systemctl daemon-reload
$ systemctl enable kube-proxy
$ systemctl start kube-proxy
$ systemctl status kube-proxy
● kube-proxy.service - Kubernetes Kube-Proxy Server
   Loaded: loaded (/lib/systemd/system/kube-proxy.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2019-08-25 02:48:06 UTC; 3 days ago
     Docs: https://github.com/kubernetes/kubernetes
 Main PID: 467 (kube-proxy)
    Tasks: 8 (limit: 4704)
   CGroup: /system.slice/kube-proxy.service
           └─467 /usr/bin/kube-proxy --logtostderr=false --v=2 --log-dir=/data/apps/kubernetes/log/ --kubeconfig=/data/apps/kubernetes/etc/kube-proxy.kubeconfig --proxy-mode=ipvs --masquerade-all=true --cluster..
....
```

### 通过证书验证添加各节点

```bash
# 在k8s1节点操作
# 查看CSR请求
$ kubectl get csr
NAME AGE REQUESTOR CONDITION 
node-csr-NT3oojJY6_VRvkkNKQMEBZyK6BZoOnEbKCqcEvjnqco 16h kubelet-bootstrap Pending node-csr-O8Zzno_W7X7QLt7EqdaVvvXfg0RS3AaSbdoOzUO821M 16h kubelet-bootstrap Pending node-csr-gXAsSMLxfcmdwaOeu8swemiqFQOcrZaPw85yfcguNlA 16h kubelet-bootstrap Pending node-csr-sjskQXhS3v8ZS7Wi0rjSdGocdTsMw_zHWL8quCATR5c 16h kubelet-bootstrap Pending

# 通过CSR请求
$ kubectl get csr | awk '/node/{print $1}' | xargs kubectl certificate approve
# 单独执行approve命令
$ kubectl certificate approve node-csr-O8Zzno_W7X7QLt7EqdaVvvXfg0RS3AaSbdoOzUO821M

$ kubectl get csr
node-csr-NT3oojJY6_VRvkkNKQMEBZyK6BZoOnEbKCqcEvjnqco 16h kubelet-bootstrap Approved,Issued
node-csr-O8Zzno_W7X7QLt7EqdaVvvXfg0RS3AaSbdoOzUO821M 16h kubelet-bootstrap Approved,Issued 
node-csr-gXAsSMLxfcmdwaOeu8swemiqFQOcrZaPw85yfcguNlA 16h kubelet-bootstrap Approved,Issued 
node-csr-sjskQXhS3v8ZS7Wi0rjSdGocdTsMw_zHWL8quCATR5c 16h kubelet-bootstrap Approved,Issued

# 查看节点, 由于没有配置网络插件, 状态均为NotReady
$ kubectl get node 
NAME           STATUS      ROLES    AGE     VERSION
10.0.100.147   NotReady    <none>   4d15h   v1.15.3
10.0.100.164   NotReady    <none>   4d15h   v1.15.3
10.0.100.178   NotReady    <none>   4d15h   v1.15.3
10.0.101.215   NotReady    <none>   4d15h   v1.15.3
10.0.101.54    NotReady    <none>   4d15h   v1.15.3
10.0.101.78    NotReady    <none>   4d15h   v1.15.3

# 在节点查看生成的证书文件
$ ls -l /data/apps/kubernetes/pki/kubelet* 
-rw------- 1 root root 1277 Aug 23 12:04 /data/apps/kubernetes/pki/kubelet-client-2019-08-23-12-04-45.pem
lrwxrwxrwx 1 root root   64 Aug 23 12:04 /data/apps/kubernetes/pki/kubelet-client-current.pem -> /data/apps/kubernetes/pki/kubelet-client-2019-08-23-12-04-45.pem
-rw-r--r-- 1 root root 2181 Aug 23 09:51 /data/apps/kubernetes/pki/kubelet.crt
-rw------- 1 root root 1675 Aug 23 09:51 /data/apps/kubernetes/pki/kubelet.key
```

### 配置flannel网络组件

**Note:**  *所有节点均需要安装flannel组件*

```bash
$ cd /root/software/
$ wget https://github.com/coreos/flannel/releases/download/v0.11.0/flannel-v0.11.0-linux-amd64.tar.gz
$ ls flannel-v0.11.0-linux-amd64.tar.gz
flannel-v0.11.0-linux-amd64.tar.gz
$ tar -xzvf flannel-v0.11.0-linux-amd64.tar.gz
$ mv flanneld mk-docker-opts.sh /usr/bin/
$ ls /usr/bin/flanneld /usr/bin/mk-docker-opts.sh 
/usr/bin/flanneld  /usr/bin/mk-docker-opts.sh
```

#### 1. etcd集群创建网络段

```bash
# k8s1节点操作
$ etcdctl --ca-file=/data/apps/etcd/ssl/etcd-ca.pem --cert-file=/data/apps/etcd/ssl/etcd.pem --key-file=/data/apps/etcd/ssl/etcd-key.pem --endpoints="https://10.0.100.178:2379,https://10.0.100.147:2379,https://10.0.101.78:2379" set /coreos.com/network/config '{ "Network": "10.99.0.0/16", "Backend": {"Type": "vxlan"}}'

$ etcdctl --ca-file=/data/apps/etcd/ssl/etcd-ca.pem --cert-file=/data/apps/etcd/ssl/etcd.pem --key-file=/data/apps/etcd/ssl/etcd-key.pem --endpoints="https://10.0.100.178:2379,https://10.0.100.147:2379,https://10.0.101.78:2379" get /coreos.com/network/config | jq 
{
  "Network": "10.99.0.0/16",
  "Backend": {
    "Type": "vxlan"
  }
}
```

#### 2. 分发etcd证书到node节点

*node节点创建etcd证书存放路径, 并分发etcd证书到Node节点*

```bash
# node节点操作
$ mkdir -p /data/apps/etcd/ssl/
$ ls /data/apps/etcd/
ssl

# k8s1节点操作
# to k8s4
$ scp /data/apps/etcd/ssl/etcd*.pem root@k8s4:/data/apps/etcd/ssl/
# to k8s5
$ scp /data/apps/etcd/ssl/etcd*.pem root@k8s5:/data/apps/etcd/ssl/
# to k8s6
$ scp /data/apps/etcd/ssl/etcd*.pem root@k8s6:/data/apps/etcd/ssl/
```

#### 3. 各节点创建flannel系统服务文件

```bash
$ cat > /lib/systemd/system/flanneld.service  << EOF
[Unit] 
Description=Flanneld overlay address etcd agent 
After=network-online.target network.target 
Before=docker.service

[Service] 
Type=notify 
EnvironmentFile=/data/apps/kubernetes/etc/flanneld.conf 
ExecStart=/usr/bin/flanneld --ip-masq $FLANNEL_OPTIONS 
ExecStartPost=/usr/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/subnet.env 
Restart=on-failure 

[Install] 
WantedBy=multi-user.target
EOF
```

#### 4. 各节点创建flannel配置文件

```bash
$ cat > /data/apps/kubernetes/etc/flanneld.conf << EOF
FLANNEL_OPTIONS="--etcd-endpoints=https://10.0.100.178:2379,https://10.0.100.147:2379,https://10.0.101.78:2379 -etcd-cafile=/data/apps/etcd/ssl/etcd-ca.pem -etcd-certfile=/data/apps/etcd/ssl/etcd.pem -etcd-keyfile=/data/apps/etcd/ssl/etcd-key.pem"
EOF
```

#### 5. 修改docker系统服务文件

```bash
# 在[Service]配置段添加如下两行
EnvironmentFile=/run/flannel/subnet.env 
ExecStart=/usr/bin/dockerd -H fd:// $DOCKER_NETWORK_OPTIONS $DOCKER_DNS_OPTIONS

$ cat /lib/systemd/system/docker.service
....
[Service]
....
EnvironmentFile=/run/flannel/subnet.env 
ExecStart=/usr/bin/dockerd -H fd:// $DOCKER_NETWORK_OPTIONS $DOCKER_DNS_OPTIONS
#ExecStart=/usr/bin/dockerd -H fd://
....

# 添加docker服务启动文件，注入dns参数
$ mkdir /lib/systemd/system/docker.service.d/

# dns参数根据实际dns修改
$ cat > /lib/systemd/system/docker.service.d/docker-dns.conf << EOF
[Service] 
Environment="DOCKER_DNS_OPTIONS=--dns 10.99.110.110  --dns-search default.svc.cluster.local --dns-search svc.cluster.local --dns-opt ndots:2 --dns-opt timeout:2 --dns-opt attempts:2"
EOF
```

#### 6. 启动各flanne节点服务

```bash
$ systemctl daemon-reload
$ systemctl enable flanneld
$ systemctl start flanneld
$ systemctl restart docker
$ systemctl status flanneld
● flanneld.service - Flanneld overlay address etcd agent
   Loaded: loaded (/lib/systemd/system/flanneld.service; disabled; vendor preset: enabled)
   Active: active (running) since Sun 2019-08-25 05:56:57 UTC; 3 days ago
 Main PID: 958 (flanneld)
    Tasks: 12 (limit: 4704)
   CGroup: /system.slice/flanneld.service
           └─958 /usr/bin/flanneld --ip-masq --etcd-endpoints=https://10.0.100.178:2379,https://10.0.100.147:2379,https://10.0.101.78:2379 -etcd-cafile=/data/apps/etcd/ssl/etcd-ca.pem -etcd-certfile=/data/apps/..
....
```

### 配置CoreDNS组件

```bash
$ cat > coredns.yml << EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:coredns
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa  {

          pods insecure
          upstream
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        #forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/name: "CoreDNS"
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      k8s-app: kube-dns
  template:
    metadata:
      labels:
        k8s-app: kube-dns
    spec:
      priorityClassName: system-cluster-critical
      serviceAccountName: coredns
      tolerations:
        - key: "CriticalAddonsOnly"
          operator: "Exists"
      nodeSelector:
        beta.kubernetes.io/os: linux
      containers:
      - name: coredns
        image: coredns/coredns:1.5.0
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            memory: 170Mi
          requests:
            cpu: 100m
            memory: 70Mi
        args: [ "-conf", "/etc/coredns/Corefile" ]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
          readOnly: true
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - all
          readOnlyRootFilesystem: true
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
            - key: Corefile
              path: Corefile
---
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  annotations:
    prometheus.io/port: "9153"
    prometheus.io/scrape: "true"
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.99.110.110
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
  - name: metrics
    port: 9153
    protocol: TCP
EOF

$ kubectl apply -f coredns.yml
serviceaccount/coredns created 
clusterrole.rbac.authorization.k8s.io/system:coredns created clusterrolebinding.rbac.authorization.k8s.io/system:coredns created configmap/coredns created 
deployment.apps/coredns created 
service/coredns created

#查看coredns是否运行正常
$ kubectl get svc,pod -n kube-system  
NAME                           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
service/kube-dns               ClusterIP   10.99.110.110   <none>        53/UDP,53/TCP,9153/TCP   3d

NAME                                        READY   STATUS    RESTARTS   AGE
pod/coredns-66b65fdb59-2495v                1/1     Running   0          3d
pod/coredns-66b65fdb59-mjn6v                1/1     Running   0          3d

# 创建一个pod用来测试dns
$ kubectl run curl --image=radial/busyboxplus:curl -i --tty
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
If you don't see a command prompt, try pressing enter.
[ root@curl-6bf6db5c4f-stfvb:/ ]$ nslookup kubernetes
Server:    10.99.110.110
Address 1: 10.99.110.110 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.99.0.1 kubernetes.default.svc.cluster.local
```

### 设置集群角色label

```bash
# 任意master节点执行
$ kubectl label nodes 10.0.100.178 node-role.kubernetes.io/master=k8s1
$ kubectl label nodes 10.0.100.147  node-role.kubernetes.io/master=k8s2
$ kubectl label nodes 10.0.101.78  node-role.kubernetes.io/master=k8s3
$ kubectl label nodes 10.0.101.54 node-role.kubernetes.io/node=k8s4
$ kubectl label nodes 10.0.100.164 node-role.kubernetes.io/node=k8s5
$ kubectl label nodes 10.0.101.215 node-role.kubernetes.io/node=k8s6
```

```bash
# 配置master节点不接受负载
$ kubectl taint node 10.0.100.178 node-role.kubernetes.io/master=k8s1:NoSchedule 
$ kubectl taint node 10.0.100.147 node-role.kubernetes.io/master=k8s2:NoSchedule 
$ kubectl taint node 10.0.101.78 node-role.kubernetes.io/master=k8s3:NoSchedule
```

### 查看集群状态

```bash
$ kubectl get node,cs
NAME                STATUS   ROLES    AGE     VERSION
node/10.0.100.147   Ready    master   4d19h   v1.15.3
node/10.0.100.164   Ready    node     4d19h   v1.15.3
node/10.0.100.178   Ready    master   4d19h   v1.15.3
node/10.0.101.215   Ready    node     4d19h   v1.15.3
node/10.0.101.54    Ready    node     4d19h   v1.15.3
node/10.0.101.78    Ready    master   4d19h   v1.15.3

NAME                                 STATUS    MESSAGE             ERROR
componentstatus/scheduler            Healthy   ok                  
componentstatus/controller-manager   Healthy   ok                  
componentstatus/etcd-0               Healthy   {"health":"true"}   
componentstatus/etcd-1               Healthy   {"health":"true"}   
componentstatus/etcd-2               Healthy   {"health":"true"}
```
