# 使用kubeadm 搭建k8s 高可用集群

## Requirement

* 操作系统 :  ubuntu 16.04

* Control Plane 节点数量最少为3台

* Workers 节点数量最少为3台

* Kubernetes要求集群中所有机器具有不同的Mac地址、产品uuid、Hostname

| ServerName | IP           | Node          |
|:----------:|:------------:|:-------------:|
| k8s1       | 10.0.100.178 | Control Plane |
| k8s2       | 10.0.100.147 | Control Plane |
| k8s3       | 10.0.101.78  | Control Plane |
| k8s4       | 10.0.101.54  | Workers       |
| k8s5       | 10.0.100.164 | Workers       |
| k8s6       | 10.0.101.215 | Workers       |
| -          | 10.0.0.17    | VIP           |

---

## 前期准备

* 配置免秘钥登陆
* 配置hosts 文件
* 关闭 swap

---

## 配置LoadBalance

> **Note:**  ***在配置HA k8s 集群前需要为kube-apiserver 创建Load Balance，此处选取haproxy + keepalived 搭建，云环境可以直接使用云服务商的SLB 服务***

#### 1. 安装haproxy , keepalived

```bash
shell> sudo apt-get update
shell> sudo apt-get haproxy keepalived
```

#### 2. 修改haproxy 配置文件/etc/haproxy/haproxy.cfg

```bash
shell> vim /etc/haproxy/haproxy.cfg
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
  bind 0.0.0.0:6443
  mode tcp
  #maxconn 50000
  default_backend k8s-https

backend k8s-https
  mode tcp
  balance roundrobin
  server k8s1 10.0.100.178:6443 weight 1 maxconn 1000 check inter 2000 rise 2 fall 3
  server k8s2 10.0.100.147:6443 weight 1 maxconn 1000 check inter 2000 rise 2 fall 3
  server k8s3 10.0.101.78:6443 weight 1 maxconn 1000 check inter 2000 rise 2 fall 3
```

#### 3.启动haproxy

```bash
shell> service haproxy start
```

#### 4. 配置 keepalived

```bash
shell> vim /etc/keepalived/keepalived.conf
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

shell> vim /etc/keepalived/haproxy_check.sh
#!/bin/bash
LOGFILE="/var/log/keepalived-haproxy-state.log"
date >> $LOGFILE
if [ `ps -C haproxy --no-header|wc -l` -eq 0 ];then
    echo "fail: check_haproxy status" >> $LOGFILE
    exit 1
else
    echo "success: check_haproxy status" >> $LOGFILE
    exit 0
fi

shell> vim /etc/keepalived/haproxy_master.sh
#!/bin/bash
LOGFILE="/var/log/keepalived-haproxy-state.log"
echo "Being Master ..." >> $LOGFILE

shell> chmod a+x /etc/keepalived/haproxy_check.sh /etc/keepalived/haproxy_master.sh
```

> **Note:**  ***剩余Control Plane节点的/etc/keepalived/keepalived.conf文件中state设置为BACKUP，priority值优先级依次降低***

#### 5. 启动 keepalived

```bash
shell> service keepalived start
```

---

## Docker,k8s组件安装

#### 1. docker-ce安装

```bash
shell> apt-get update && apt-get install apt-transport-https ca-certificates curl software-properties-common
shell> curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
shell> add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
shell> apt-get update && apt-get install docker-ce
```

#### 2. 启动docker

```bash
shell> service docker start
```

#### 3. 安装k8s组件kubelet kubeadm kubectl

> **Note:**  ***Workers节点可以选择安装kubectl***

```bash
shell> apt-get update && apt-get install -y apt-transport-https curl
shell> curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
shell> cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

shell> apt-get update && apt-get install -y kubelet kubeadm kubectl
shell> apt-mark hold kubelet kubeadm kubectl
```

### 4. 查看是否安装成功

```bash
shell> kubeadm version 
kubeadm version: &version.Info{Major:"1", Minor:"14", GitVersion:"v1.14.1", GitCommit:"b7394102d6ef778017f2ca4046abbaa23b88c290", GitTreeState:"clean", BuildDate:"2019-04-08T17:08:49Z", GoVersion:"go1.12.1", Compiler:"gc", Platform:"linux/amd64"}

shell> kubectl version 
Client Version: version.Info{Major:"1", Minor:"14", GitVersion:"v1.14.1", GitCommit:"b7394102d6ef778017f2ca4046abbaa23b88c290", GitTreeState:"clean", BuildDate:"2019-04-08T17:11:31Z", GoVersion:"go1.12.1", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"14", GitVersion:"v1.14.1", GitCommit:"b7394102d6ef778017f2ca4046abbaa23b88c290", GitTreeState:"clean", BuildDate:"2019-04-08T17:02:58Z", GoVersion:"go1.12.1", Compiler:"gc", Platform:"linux/amd64"}

shell> kubelet --version 
Kubernetes v1.14.1
```

---

## 配置k8s集群

### 1. 在k8s1机器上创建集群配置文件kubeadm-config.yaml

> **Note:**  ***以下文件需要用到域名，请为VIP配置域名解析***

```bash
shell> vim kubeadm-config.yaml  
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
kubernetesVersion: stable
apiServer:
  certSANs:
  - "www.clusterlb.net"
controlPlaneEndpoint: "www.clusterlb.net:6443"
```

> **Note:**  ***www.clusterlb.net为VIP绑定的域名***

### 2. 执行配置命令kubeadm init

```bash
shell> sudo kubeadm init --config=kubeadm-config.yaml --experimental-upload-certs
```

> **Note:**  ***--experimental-upload-certs用于将应该在所有Control Plane节点之间共享的证书上载到集群。如果您希望手动或使用自动化工具跨Control Plane节点复制证书，请去除这个参数选项，并参阅下面的手动证书分发部分。--experimental-upload-certs参数在v1.14.0版本以后才出现，v1.14.0之前的版本需要手动分发证书***

命令完成后，您应该看到如下内容：

```textile
You can now join any number of control-plane node by running the following command on each as a root:
  kubeadm join www.clusterlb.net:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866 --experimental-control-plane --certificate-key f8902e114ef118304e561c3ecd4d0b543adc226b7a07f675f56564185ffe0c07

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use kubeadm init phase upload-certs to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:
  kubeadm join www.clusterlb.net:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866
```

* Copy this output to a text file. You will need it later to join control plane and worker nodes to the cluster.

* When`--experimental-upload-certs`is used with`kubeadm init`, the certificates of the primary control plane are encrypted and uploaded in the`kubeadm-certs`Secret.

* To re-upload the certificates and generate a new decryption key, use the following command on a control plane node that is already joined to the cluster:

  ```bash
  shell> sudo kubeadm init phase upload-certs --experimental-upload-certs
  ```

### 3. 安装Weave CNI网络插件

```bash
shell> kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

执行以下命令并查看control plane组件的pod启动:

```bash
shell> kubectl get pod -n kube-system -w
```

### 4. 其他control plane节点加入

> **Caution:** ***只有在第一个control plane节点完成初始化之后，才能按顺序加入新的control plane节点。***

对于其他的control plane节点，应该:

```bash
shell> kubeadm join www.clusterlb.net:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866 --experimental-control-plane --certificate-key f8902e114ef118304e561c3ecd4d0b543adc226b7a07f675f56564185ffe0c07
```

- The`--experimental-control-plane`flag tells`kubeadm join`to create a new control plane.
- The`--certificate-key ...`will cause the control plane certificates to be downloaded from the`kubeadm-certs`Secret in the cluster and be decrypted using the given key.

### 5. Workers节点加入

```bash
shell> kubeadm join www.clusterlb.net:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866
```

### 6. 搭建完成后确定集群状态

```bash
shell>  kubectl get nodes
NAME   STATUS   ROLES    AGE   VERSION
k8s1   Ready    master   87m   v1.14.1
k8s2   Ready    master   93m   v1.14.1
k8s3   Ready    master   81m   v1.14.1
k8s4   Ready    <none>   69m   v1.14.1
k8s5   Ready    <none>   71m   v1.14.1
k8s6   Ready    <none>   70m   v1.14.1    


shell> kubectl get pods -n kube-system 
NAME                           READY   STATUS    RESTARTS   AGE
coredns-86c58d9df4-2t6jg       1/1     Running   0          94m
coredns-86c58d9df4-x9ssx       1/1     Running   0          94m
etcd-k8s1                      1/1     Running   1          88m
etcd-k8s2                      1/1     Running   0          93m
etcd-k8s3                      1/1     Running   0          82m
kube-apiserver-k8s1            1/1     Running   2          88m
kube-apiserver-k8s2            1/1     Running   0          93m
kube-apiserver-k8s3            1/1     Running   0          82m
kube-controller-manager-k8s1   1/1     Running   2          88m
kube-controller-manager-k8s2   1/1     Running   2          93m
kube-controller-manager-k8s3   1/1     Running   0          82m
kube-proxy-6fx2x               1/1     Running   0          71m
kube-proxy-fckhp               1/1     Running   0          94m
kube-proxy-fgj52               1/1     Running   0          72m
kube-proxy-h5dsc               1/1     Running   0          82m
kube-proxy-r7trm               1/1     Running   1          88m
kube-proxy-rn24g               1/1     Running   0          70m
kube-scheduler-k8s1            1/1     Running   1          88m
kube-scheduler-k8s2            1/1     Running   1          93m
kube-scheduler-k8s3            1/1     Running   1          82m
weave-net-7bjm9                2/2     Running   0          71m
weave-net-9lb2c                2/2     Running   0          70m
weave-net-b9phx                2/2     Running   0          72m
weave-net-lwtp2                2/2     Running   3          88m
weave-net-pxrm5                2/2     Running   0          92m
weave-net-xgj9m                2/2     Running   0          82m
```

> **Note:**  ***Node的status状态均为Ready，pod的status状态均为Running***

---

### 手动证书分发

> **Note:**  ***在v1.14.0版本之前，都需要手动分发证书，或者在v1.14.0版本之后，在kubeadm init 时没有添加--experimental-upload-certs参数***

分发证书文件到其他control plane节点，脚本如下 :

```bash
shell> vim scp.sh
#!/bin/bash
USER=ubuntu # customizable
CONTROL_PLANE_IPS="k8s2 k8s3"  #Other control plane's hostname or ip

for host in ${CONTROL_PLANE_IPS}; do
    scp /etc/kubernetes/pki/ca.crt "${USER}"@$host:
    scp /etc/kubernetes/pki/ca.key "${USER}"@$host:
    scp /etc/kubernetes/pki/sa.key "${USER}"@$host:
    scp /etc/kubernetes/pki/sa.pub "${USER}"@$host:
    scp /etc/kubernetes/pki/front-proxy-ca.crt "${USER}"@$host:
    scp /etc/kubernetes/pki/front-proxy-ca.key "${USER}"@$host:
    scp /etc/kubernetes/pki/etcd/ca.crt "${USER}"@$host:etcd-ca.crt
    scp /etc/kubernetes/pki/etcd/ca.key "${USER}"@$host:etcd-ca.key
    scp /etc/kubernetes/admin.conf "${USER}"@$host:
done

shell> bash scp.sh
```

在其余control plane节点上将复制过来的证书文件移动到相应的目录，脚本如下 :

```bash
shell> vim move_key.sh
#!/bin/bash
USER=ubuntu  #customizable

mkdir -p /etc/kubernetes/pki/etcd
mv /home/${USER}/ca.crt /etc/kubernetes/pki/
mv /home/${USER}/ca.key /etc/kubernetes/pki/
mv /home/${USER}/sa.pub /etc/kubernetes/pki/
mv /home/${USER}/sa.key /etc/kubernetes/pki/
mv /home/${USER}/front-proxy-ca.crt /etc/kubernetes/pki/
mv /home/${USER}/front-proxy-ca.key /etc/kubernetes/pki/
mv /home/${USER}/etcd-ca.crt /etc/kubernetes/pki/etcd/ca.crt
mv /home/${USER}/etcd-ca.key /etc/kubernetes/pki/etcd/ca.key
mv /home/${USER}/admin.conf /etc/kubernetes/admin.conf

shell> bash move_key.sh
```
