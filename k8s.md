[TOC]

# K8S 集群部署

[REFERENCE](https://kubernetes.io/docs/home/)

## 环境介绍

节点部署
| 主机名 | IP | 配置 | Role | 备注 |
| ---- | ------ | ---- | ---- | ------- |
| docker-k8s01 | 192.168.200.8 | 2C4G | Master,Etcd | kube-apiserver、kube-controller-manager、kube-scheduler |
| docker-k8s02 | 192.168.200.9 | 2C4G | Master,Etcd | kube-apiserver、kube-controller-manager、kube-scheduler |
| docker-k8s03 | 192.168.200.10 | 2C4G | Master,Etcd | kubelet、kube-proxy |
| docker-k8s04 | 192.168.200.11 | 2C4G | Node,Etcd | kubelet、kube-proxy |
| docker-k8s05 | 192.168.200.12 | 2C4G | Node,Etcd | kubelet、kube-proxy |
| VIP | 192.168.200.88 |   |   | kube-apiserver VIP |

网络部署
|网络 | IP段 |
| ---- | ------ |
| 物理网络 | 192.168.200.0/24 |
| POD网络 | 10.244.0.0/16 |
| SERVICE网络 | 172.21.0.0/16 |


依赖组件
|  组件名称  |  组件版本  |
| ------------- | ----------- |
| Etcd | 3.4.12 |
| Docker | 19.03.15 |
| Go | |
| CNI | 0.8.7 |
| CSI | |
| Dashboard |  |
| Heapster | |
| Cluster | |
| kube-dns | |
| Influxdb | |
| Grafana | |
| Kibana | |
| cAdvisor | |
| Fluentd | |
| Elasticsearch | |
| go-oidc | |
| calico | |
| crictl | |
| CoreDNS | 1.8.3 |
| event-exporter | |
| metrics-server | |
| ingress-gce | |
| ingress-nginx | |
| ip-masq-agent | |
| hcsshim | |

## 环境初始化（所有节点)
```
# 密钥互通node1分发
ssh-keygen -P "" -f /root/.ssh/id_rsa
cat /root/.ssh/id_rsa.pub > /root/.ssh/authorized_keys
for i in {2..5};do scp -r .ssh node$i:~/ ;done

# 配置Yum源
rm -f /etc/yum.repos.d/* && wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo && wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo && yum clean all && yum makecache fast

# 主机名解析
cat /etc/hosts
192.168.200.8 docker-k8s01
192.168.200.9 docker-k8s02
192.168.200.10 docker-k8s03
192.168.200.11 docker-k8s04
192.168.200.12 docker-k8s05

# 关闭防火墙
systemctl disable firewalld;systemctl stop firewalld

# 禁用SELinux，允许容器访问宿主机的文件系统
setenforce 0
sed -i 's@SELINUX=.*@SELINUX=disabled@' /etc/selinux/config

# 同步时间
yum -y install chrony
systemctl start chronyd && systemctl enable chronyd
ntpdate ntp1.aliyun.com

# 关闭SWAP
swapoff -a && sed -ri 's/.*swap.*/#&/' /etc/fstab
echo "vm.swappiness = 0">> /etc/sysctl.conf  && sysctl -p
```

## 升级内核（所有节点)
```
# 升级内核,安装长期支持版本
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh https://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
yum --enablerepo=elrepo-kernel install kernel-lt -y
grub2-set-default 0

# 开启 User namespaces
grubby --args="user_namespace.enable=1" --update-kernel=/boot/vmlinuz-4.4.154-1.el7.elrepo.x86_64
grep user_namespace /boot/grub2/grub.cfg

# 重启
reboot

# 查看内核版本
uname -a
Linux docker-k8s01 4.4.118-1.el7.elrepo.x86_64 #1 SMP Sun Feb 25 19:10:14 EST 2018 x86_64 x86_64 x86_64 GNU/Linux
```

## 加载模块
```

# 加载 br_netfilter iptables 模块
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

# 将 net.bridge.bridge-nf-call-iptables设为1
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system

# 加载ipvs模块
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
lsmod | grep ip_vs && lsmod | grep nf_conntrack_ipv4

yum install -y ipvsadm
```



## 安装 Docker

> 在 master 、worker节点上安装

### 1. 配置docker
[Docker官网安装文档](https://docs.docker.com/engine/install/)
[Docker镜像国内](https://kubernetes.feisky.xyz/appendix/mirrors)
```
yum install -y yum-utils device-mapper-persistent-data lvm2

wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo

yum -y install docker-ce-19.03.15-3.el7 docker-ce-cli-19.03.15-3.el7 containerd.io-1.4.3-3.1.el7 

# 配置docker目录
mkdir /data/docker -p && mkdir /etc/docker -p && mkdir -p /etc/systemd/system/docker.service.d

# 配置国内镜像加速
$ cat > /etc/docker/daemon.json << EOF
{
  "graph": "/data/docker",
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
  "registry-mirrors": [
      "https://registry.docker-cn.com",
      "https://1nj0zren.mirror.aliyuncs.com",
      "https://kfwkfulq.mirror.aliyuncs.com",
      "https://2lqq34jg.mirror.aliyuncs.com",
      "https://pee6w651.mirror.aliyuncs.com",
      "http://hub-mirror.c.163.com",
      "https://docker.mirrors.ustc.edu.cn",
      "http://f1361db2.m.daocloud.io",
      "https://registry.docker-cn.com"
  ],
  "insecure-registries": ["192.168.200.7"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "live-restore": true,
  "log-driver": "json-file",
  "log-level": "warn",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF

# 启动dcoker
$ systemctl start docker && systemctl enable docker && systemctl stop docker
```



## 安装haproxy,keepalived(master所有节点)

### 1. haproxy

```
# 安装 haproxy keepalived 和依赖
yum install haproxy keepalived libnl3-devel ipset-devel -y

# 配置 haproxy，注意修改成自己的IP
vim /etc/haproxy/haproxy.cfg

global
        log /dev/log    local0
        log /dev/log    local1 notice
        chroot /var/lib/haproxy
        stats socket /var/run/admin.sock mode 660 level admin
        stats timeout 30s
        user haproxy
        group haproxy
        daemon
        nbproc 1

defaults
        log     global
        timeout connect 5000
        timeout client  50000
        timeout server  50000

listen kube-master
        bind 0.0.0.0:8443
        mode tcp
        option tcplog
        balance source
        server s1 192.168.200.8:6443  check inter 10000 fall 2 rise 2 weight 1
        server s2 192.168.200.9:6443  check inter 10000 fall 2 rise 2 weight 1
        server s3 192.168.200.10:6443  check inter 10000 fall 2 rise 2 weight 1

# 分发其他节点
for node in node2 node3;do scp /etc/haproxy/haproxy.cfg $node:/etc/haproxy/;done

# 启动 haproxy
for haproxy in node1 node2 node3;do ssh ${haproxy} "systemctl start haproxy && systemctl enable haproxy";done

# 查看下服务情况
systemctl status haproxy
```

### 2. keepalived
```
# 配置 keepalived，注意修改成自己的IP和网卡名称
# master1 配置
vim /etc/keepalived/keepalived.conf

global_defs {
    script_user root 
    enable_script_security
    router_id lb-master
}

vrrp_script check-haproxy {
    script "/bin/bash -c 'if [[ $(netstat -nlp | grep 8443) ]]; then exit 0; else exit 1; fi'"
    interval 2
    weight -50
}

vrrp_instance VI-kube-master {
    state MASTER
    priority 120
    dont_track_primary
    interface ens33
    virtual_router_id 120
    advert_int 3
    track_script {
        check-haproxy
    }
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.200.88
    }
}

# notify "/container/service/keepalived/assets/notify.sh"


# 分发 keepalived
for keepalived in node2 node3;do scp /etc/keepalived/keepalived.conf ${keepalived}:/etc/keepalived/;done

# master2 修改
sed -i -e 's@state MASTER@state BACKUP@' -e 's@priority 120@priority 90@' /etc/keepalived/keepalived.conf

# master3 修改
sed -i -e 's@state MASTER@state BACKUP@' -e 's@priority 120@priority 100@' /etc/keepalived/keepalived.conf


# 启动 keepalived
for keepalived in node1 node2 node3;do ssh ${keepalived} "systemctl start keepalived && systemctl enable keepalived";done


# 查看下服务情况
systemctl status keepalived
```


## 生成CA证书（Master01）

### 安装 CFSSL
```
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl*
mv cfssl_linux-amd64 /usr/local/bin/cfssl
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo
```

### 准备CA证书

#### 1.准备相关目录
```
### 创建ssl文件夹
mkdir /root/k8s/ssl/ -p

for node in node1 node2 node3 node4 node5;do ssh ${node} "mkdir -p /etc/kubernetes/ssl/ && mkdir -p /var/log/kubernetes";done
```

#### 2. ca-config.json CA配置文件
```
cat > /root/k8s/ssl/ca-config.json  <<'HERE'
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
HERE
```

#### 3. ca-csr.json CA证书签名请求
```
cat > /root/k8s/ssl/ca-csr.json  <<'HERE'
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 4096
  },
  "names": [
    {
      "C": "CN",
      "ST": "Shenzhen",
      "L": "Shenzhen",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
HERE
```
#### 4. 生成CA证书
```
cd /root/k8s/ssl/
cfssl gencert --initca=true ca-csr.json | cfssljson --bare ca

for node in node1 node2 node3;do scp -r /root/k8s/ssl/ca*.pem $node:/etc/kubernetes/ssl/;done
```


## 安装配置ETCD集群（node1）

#### 1. 准备相关目录
```
mkdir -p /root/k8s/etcd/{bin/systemd}
```

#### 2. etcd 证书签名请求
```
cat > /root/k8s/ssl/etcd-csr.json  <<'HERE'
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "192.168.200.8",
    "192.168.200.9",
    "192.168.200.10"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
    "names": [{
      "C": "CN",
      "ST": "Shenzhen",
      "L": "Shenzhen",
      "O": "k8s",
      "OU": "System"
    }]
}
HERE
```

#### 3. 生成etcd证书
```
cd /root/k8s/ssl/
cfssl gencert --ca ca.pem --ca-key ca-key.pem --config ca-config.json --profile kubernetes etcd-csr.json | cfssljson --bare etcd

for node in node1 node2 node3;do scp -r /root/k8s/ssl/etcd*.pem $node:/etc/kubernetes/ssl/;done
```

#### 4.下载etcd并解压
```
cd
wget https://github.com/etcd-io/etcd/releases/download/v3.4.13/etcd-v3.4.13-linux-amd64.tar.gz
tar xf etcd-v3.4.13-linux-amd64.tar.gz
cp -f /root/etcd-v3.4.13-linux-amd64/etcd* /root/k8s/etcd/bin/
```

#### 5.创建etcd启动文件
```
cat > /root/k8s/etcd/systemd/etcd.service <<'HERE'
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos
[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
ExecStart=/usr/local/bin/etcd \
  --name=etcd01 \
  --cert-file=/etc/kubernetes/ssl/etcd.pem \
  --key-file=/etc/kubernetes/ssl/etcd-key.pem \
  --peer-cert-file=/etc/kubernetes/ssl/etcd.pem \
  --peer-key-file=/etc/kubernetes/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --initial-advertise-peer-urls=https://192.168.200.8:2380 \
  --listen-peer-urls=https://192.168.200.8:2380 \
  --listen-client-urls=https://192.168.200.8:2379,http://127.0.0.1:2379 \
  --advertise-client-urls=https://192.168.200.8:2379 \
  --initial-cluster-token=etcd-cluster \
  --initial-cluster=etcd01=https://192.168.200.8:2380,etcd02=https://192.168.200.9:2380,etcd03=https://192.168.200.10:2380 \
  --initial-cluster-state=new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
HERE
```

#### 6. 下发etcd二进制文件
```
# 分发etcd命令
for node in node1 node2 node3;do rsync -avP /root/k8s/etcd/bin/ ${node}:/usr/local/bin/;done

# 下发etcd所需的service文件
for node in node1 node2 node3;do rsync -avP /root/k8s/etcd/systemd/ ${node}:/lib/systemd/system/;done

# 创建etcd所需的数据目录
for node in node1 node2 node3;do ssh ${node} "mkdir /etc/etcd;mkdir /var/lib/etcd";done
```

#### 7.修改service相应节点IP(node2,node3节点执行)
```
# node2 操作
sed -i -e 's@192.168.200.8@192.168.200.9@g' -e 's@etcd01@etcd02@' -e 's@--initial-cluster=etcd02=https://192.168.200.9:2380@--initial-cluster=etcd01=https://192.168.200.8:2380@g' /lib/systemd/system/etcd.service

# node3 操作
sed -i -e 's@192.168.200.8@192.168.200.10@g' -e 's@etcd01@etcd03@' -e 's@--initial-cluster=etcd03=https://192.168.200.10:2380@--initial-cluster=etcd01=https://192.168.200.8:2380@g' /lib/systemd/system/etcd.service
```

#### 8.启动etcd集群,由于集群需要保证quorum机制，因此至少有两个节点启动服务，集群初始化启动成功
```
for node in node1 node2 node3;do ssh ${node} "systemctl daemon-reload && systemctl start etcd && systemctl enable etcd";done
```

#### 9.检查集群健康
```
# node1,node2,node3,随便1个节点执行
ETCDCTL_API=3 etcdctl --write-out="table"\
  --cacert=/etc/kubernetes/ssl/ca.pem\
  --cert=/etc/kubernetes/ssl/etcd.pem \
  --key=/etc/kubernetes/ssl/etcd-key.pem \
  --endpoints=https://192.168.200.8:2379,https://192.168.200.9:2379,https://192.168.200.10:2379 endpoint health
  
+-----------------------------+--------+-------------+-------+
|          ENDPOINT           | HEALTH |    TOOK     | ERROR |
+-----------------------------+--------+-------------+-------+
|  https://192.168.200.8:2379 |   true | 12.972928ms |       |
| https://192.168.200.10:2379 |   true | 14.814913ms |       |
|  https://192.168.200.9:2379 |   true | 18.428591ms |       |
+-----------------------------+--------+-------------+-------+
```


## kubernetes 组件部署

### 1. 下载kubernetes二进制并下发
```
# 下载k8s 1.20.1二进制包
wget https://dl.k8s.io/v1.20.1/kubernetes-server-linux-amd64.tar.gz

# 目录准备
mkdir -p /root/k8s/{master,node}/{bin,systemd}

# 解压
tar xf kubernetes-server-linux-amd64.tar.gz && tar xf kubernetes/kubernetes-src.tar.gz -C kubernetes/

# 根据master，node进行归类
cp -f /root/kubernetes/server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl} /root/k8s/master/bin/

cp -f /root/kubernetes/server/bin/{kubelet,kube-proxy} /root/k8s/node/bin/

# 下发master二进制文件
for node in node1 node2 node3;do rsync -avP /root/k8s/master/bin/ ${node}:/usr/local/bin/;done

# 下发node二进制文件
for node in node1 node2 node3 node4 node5;do rsync -avP /root/k8s/node/bin/ ${node}:/usr/local/bin/;done
```

### 2. 部署kube-apiserver 
>注意：此处需要将dns首ip、etcd、master、node、service、VIP的ip都填上

#### 1. 创建csr请求文件
```
cat > /root/k8s/ssl/kube-apiserver-csr.json  <<'HERE'
{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "192.168.200.7",
    "192.168.200.8",
    "192.168.200.9",
    "192.168.200.10",
    "192.168.200.11",
    "192.168.200.12",
    "192.168.200.88",
    "172.21.0.1",
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
      "ST": "Shenzhen",
      "L": "Shenzhen",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
HERE
```
#### 2. 生成证书
```
cd /root/k8s/ssl/ && cfssl gencert --ca ca.pem --ca-key ca-key.pem --config ca-config.json --profile kubernetes kube-apiserver-csr.json | cfssljson --bare kube-apiserver

for node in node1 node2 node3;do scp -r /root/k8s/ssl/kube-apiserver*.pem $node:/etc/kubernetes/ssl/;done
```

#### 3. 生成token文件
```
cd /root/k8s/ssl

cat > token.csv <<EOF
$(head -c 16 /dev/urandom | od -An -t x | tr -d ' '),kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF

for node in node1 node2 node3;do scp -r /root/k8s/ssl/token.csv $node:/etc/kubernetes/ssl/;done
```

#### 4. kube-apiserver 启动文件
```
cat > /root/k8s/master/systemd/kube-apiserver.service  <<'HERE'
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target
[Service]
ExecStart=/usr/local/bin/kube-apiserver \
--enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
--bind-address=192.168.200.8 \
--advertise-address=192.168.200.8 \
--insecure-bind-address=127.0.0.1 \
--anonymous-auth=false \
--authorization-mode=RBAC,Node \
--kubelet-https=true \
--enable-bootstrap-token-auth \
--token-auth-file=/etc/kubernetes/ssl/token.csv \
--service-cluster-ip-range=172.21.0.0/16 \
--service-node-port-range=30000-50000 \
--tls-cert-file=/etc/kubernetes/ssl/kube-apiserver.pem \
--tls-private-key-file=/etc/kubernetes/ssl/kube-apiserver-key.pem \
--client-ca-file=/etc/kubernetes/ssl/ca.pem \
--kubelet-client-certificate=/etc/kubernetes/ssl/kube-apiserver.pem \
--kubelet-client-key=/etc/kubernetes/ssl/kube-apiserver-key.pem \
--service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \
--service-account-signing-key-file=/etc/kubernetes/ssl/ca-key.pem  \
--service-account-issuer=https://kubernetes.default.svc.cluster.local \
--etcd-cafile=/etc/kubernetes/ssl/ca.pem \
--etcd-certfile=/etc/kubernetes/ssl/etcd.pem \
--etcd-keyfile=/etc/kubernetes/ssl/etcd-key.pem \
--etcd-servers=https://192.168.200.8:2379,https://192.168.200.9:2379,https://192.168.200.10:2379 \
--enable-swagger-ui=true \
--allow-privileged=true \
--audit-log-maxage=30 \
--audit-log-maxbackup=3 \
--audit-log-maxsize=100 \
--audit-log-path=/var/lib/audit.log \
--event-ttl=1h \
--v=4
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
HERE
```

#### 5. 分发 kube-apiserver 启动文件
```
for node in node1 node2 node3;do scp -r /root/k8s/master/systemd/kube-apiserver.service $node:/lib/systemd/system/;done

# node2修改IP
sed -i -e 's@192.168.200.8@192.168.200.9@g' -e 's@192.168.200.9:2379@192.168.200.8:2379@' /lib/systemd/system/kube-apiserver.service

# node3修改IP
sed -i -e 's@192.168.200.8@192.168.200.10@g' -e 's@192.168.200.10:2379@192.168.200.8:2379' /lib/systemd/system/kube-apiserver.service
```

#### 6. 启动 kube-apiserver
```
for node in node1 node2 node3;do 
ssh $node "systemctl daemon-reload && systemctl start kube-apiserver && systemctl enable kube-apiserver && systemctl status kube-apiserver";done

# 有返回则正常
curl --insecure https://192.168.200.8:6443/
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
}

```

### 3. 部署 kubectl

#### 1. kubectl 创建csr请求文件
```
cat > /root/k8s/ssl/admin-csr.json  <<'HERE'
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Shenzhen",
      "L": "Shenzhen",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
HERE
```

#### 2. 生成 kubectl 证书
```
cd /root/k8s/ssl/ && cfssl gencert --ca ca.pem --ca-key ca-key.pem --config ca-config.json --profile kubernetes admin-csr.json | cfssljson --bare admin

for node in node1 node2 node3;do scp -r /root/k8s/ssl/admin*.pem $node:/etc/kubernetes/ssl/;done
```

#### 3. 生成集群管理员admin kubeconfig配置文件供kubectl调用
>kubeconfig 为 kubectl 的配置文件，包含访问 apiserver 的所有信息，如 apiserver 地址、CA 证书和自身使用的证书
```
# 设置一个名字叫 kubernetes 的集群
kubectl config set-cluster kubernetes \
  --certificate-authority=ca.pem\
  --embed-certs=true \
  --server=https://192.168.200.88:8443 \
  --kubeconfig=kube.kubeconfig
 
# 设置一个用户名叫 admin 的用户
kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --client-key=admin-key.pem \
  --embed-certs=true \
  --kubeconfig=kube.kubeconfig
 
# 设置 admin 用户绑定到 kubernetes 集群
kubectl config set-context admin \
  --cluster=kubernetes \
  --user=admin \
  --kubeconfig=kube.kubeconfig
 
# 设置默认admin上下文
kubectl config use-context admin \
    --kubeconfig=kube.kubeconfig
```

#### 4.创建master /root/.kube 目录,复制超级admin授权config
```
# 创建目录
for node in node1 node2 node3;do ssh $node "mkdir -p /root/.kube";done

# 分发复制
for node in node1 node2 node3;do scp /root/k8s/ssl/kube.kubeconfig  $node:/root/.kube/config;done
```

#### 5.授权kubernetes证书访问kubelet api权限
```
kubectl create clusterrolebinding kube-apiserver:kubelet-apis --clusterrole=system:kubelet-api-admin --user kubernetes
```

#### 6. 查看集群组件状态
>上面步骤完成后，kubectl就可以与kube-apiserver通信了
```
kubectl cluster-info
# 此时kube-controller-manager kube-scheduler没部署的状态是Unheathly的
kubectl get componentstatuses                                        
kubectl get all --all-namespaces
```

#### 7. kubectl 命令不全
```
yum install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
kubectl completion bash > ~/.kube/completion.bash.inc
source '/root/.kube/completion.bash.inc'
source $HOME/.bash_profile
```

### 4. 部署 kube-controller-manager
#### 1.创建csr请求文件
```
cat > /root/k8s/ssl/kube-controller-manager-csr.json  <<'HERE'
{
  "CN": "system:kube-controller-manager",
  "hosts": [
    "127.0.0.1",
    "192.168.200.8",
    "192.168.200.9",
    "192.168.200.10"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Shenzhen",
      "L": "Shenzhen",
      "O": "system:kube-controller-manager",
      "OU": "System"
    }
  ]
}
HERE
```
#### 2. 生成 kube-controller-manager 证书
```
cd /root/k8s/ssl/ && cfssl gencert --ca ca.pem --ca-key ca-key.pem --config ca-config.json --profile kubernetes kube-controller-manager-csr.json | cfssljson --bare kube-controller-manager

# 分发到Master
for node in node1 node2 node3;do scp -r /root/k8s/ssl/kube-controller-manager*.pem $node:/etc/kubernetes/ssl/;done
```
#### 3. 创建kube-controller-manager的kubeconfig
```
# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=ca.pem\
  --embed-certs=true \
  --server=https://192.168.200.88:8443 \
  --kubeconfig=kube-controller-manager.kubeconfig

# 设置客户端认证参数
kubectl config set-credentials system:kube-controller-manager \
  --client-certificate=kube-controller-manager.pem \
  --client-key=kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-controller-manager.kubeconfig
  
# 设置上下文参数
kubectl config set-context system:kube-controller-manager \
  --cluster=kubernetes \
  --user=system:kube-controller-manager \
  --kubeconfig=kube-controller-manager.kubeconfig

# 设置默认上下文
kubectl config use-context system:kube-controller-manager \
  --kubeconfig=kube-controller-manager.kubeconfig
```

#### 4. 复制 kube-controller-manager.kubeconfig 到 /etc/kubernetes/
```
# 分发复制
for node in node1 node2 node3;do scp /root/k8s/ssl/kube-controller-manager.kubeconfig  $node:/etc/kubernetes/;done
```

#### 5. 启动配置文件
```
cat > /root/k8s/master/systemd/kube-controller-manager.service <<'HERE'
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
[Service]
ExecStart=/usr/local/bin/kube-controller-manager \
  --port=0 \
  --bind-address=127.0.0.1 \
  --kubeconfig=/etc/kubernetes/kube-controller-manager.kubeconfig \
  --allocate-node-cidrs=true \
  --service-cluster-ip-range=172.21.0.0/16 \
  --cluster-cidr=10.244.0.0/16 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \
  --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --root-ca-file=/etc/kubernetes/ssl/ca.pem \
  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --leader-elect=true \
  --feature-gates=RotateKubeletServerCertificate=true \
  --controllers=*,bootstrapsigner,tokencleaner \
  --horizontal-pod-autoscaler-use-rest-clients=true \
  --horizontal-pod-autoscaler-sync-period=10s \
  --tls-cert-file=/etc/kubernetes/ssl/kube-controller-manager.pem \
  --tls-private-key-file=/etc/kubernetes/ssl/kube-controller-manager-key.pem \
  --use-service-account-credentials=true \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/log/kubernetes \
--v=2
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
HERE
```

#### 6. 分发 kube-controller-manager 启动文件
```
# 分发启动文件到Master
for node in node1 node2 node3;do scp -r /root/k8s/master/systemd/kube-controller-manager.service $node:/lib/systemd/system/;done

# 启动 kube-controller-manager
for node in node1 node2 node3;do ssh $node "systemctl daemon-reload && systemctl start kube-controller-manager && systemctl enable kube-controller-manager && systemctl status kube-controller-manager";done
```

### 5. 部署 kube-scheduler
#### 1.创建csr请求文件
```
cat > /root/k8s/ssl/kube-scheduler-csr.json  <<'HERE'
{
  "CN": "system:kube-scheduler",
  "hosts": [
    "127.0.0.1",
    "192.168.200.8",
    "192.168.200.9",
    "192.168.200.10"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Shenzhen",
      "L": "Shenzhen",
      "O": "system:kube-scheduler",
      "OU": "System"
    }
  ]
}
HERE
```

#### 2. 生成 kube-scheduler 证书
```
# 生成证书
cd /root/k8s/ssl/ && cfssl gencert --ca ca.pem --ca-key ca-key.pem --config ca-config.json --profile kubernetes kube-scheduler-csr.json | cfssljson --bare kube-scheduler

# 分发到 Master
for node in node1 node2 node3;do scp -r /root/k8s/ssl/kube-scheduler*.pem $node:/etc/kubernetes/ssl/;done
```
#### 3. 创建 kube-scheduler 的kubeconfig
```
# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=ca.pem\
  --embed-certs=true \
  --server=https://192.168.200.88:8443 \
  --kubeconfig=kube-scheduler.kubeconfig

# 设置客户端认证参数
kubectl config set-credentials system:kube-scheduler \
  --client-certificate=kube-scheduler.pem \
  --client-key=kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-scheduler.kubeconfig
  
# 设置上下文参数
kubectl config set-context system:kube-scheduler \
  --cluster=kubernetes \
  --user=system:kube-scheduler \
  --kubeconfig=kube-scheduler.kubeconfig

# 设置默认上下文
kubectl config use-context system:kube-scheduler \
  --kubeconfig=kube-scheduler.kubeconfig
```

#### 4. 复制 kube-scheduler.kubeconfig 到 /etc/kubernetes
```
# 分发复制
for node in node1 node2 node3;do scp /root/k8s/ssl/kube-scheduler.kubeconfig $node:/etc/kubernetes/;done
```

#### 5. 启动配置文件
```
cat > /root/k8s/master/systemd/kube-scheduler.service <<'HERE'
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
[Service]
ExecStart=/usr/local/bin/kube-scheduler \
  --bind-address=127.0.0.1 \
  --kubeconfig=/etc/kubernetes/kube-scheduler.kubeconfig \
  --leader-elect=true \
  --log-dir=/var/log/kubernetes \
  --v=2
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
HERE
```

#### 6. 分发  kube-scheduler 启动文件
```
# 分发启动文件到Master
for node in node1 node2 node3;do scp -r /root/k8s/master/systemd/kube-scheduler.service $node:/lib/systemd/system/;done

# 启动 kube-controller-manager
for node in node1 node2 node3;do ssh $node "systemctl daemon-reload && systemctl start kube-scheduler && systemctl enable kube-scheduler && systemctl status kube-scheduler";done
```

### 6. 部署 kubelet

[REFERENCE](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/)

#### 1. 创建 kubelet-bootstrap.kubeconfig
```
BOOTSTRAP_TOKEN=$(awk -F "," '{print $1}' /etc/kubernetes/ssl/token.csv)

# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://192.168.200.88:8443 \
  --kubeconfig=kubelet-bootstrap.kubeconfig

# 设置客户端认证参数
kubectl config set-credentials kubelet-bootstrap \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=kubelet-bootstrap.kubeconfig

# 设置上下文参数
kubectl config set-context default --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=kubelet-bootstrap.kubeconfig

# 设置默认上下文
kubectl config use-context default \
  --kubeconfig=kubelet-bootstrap.kubeconfig

# 创建角色绑定
kubectl create clusterrolebinding kubelet-bootstrap \
  --clusterrole=system:node-bootstrapper \
  --user=kubelet-bootstrap
```

#### 2. 创建配置文件
>"cgroupDriver": "systemd",                     # 如果docker的驱动为 cgroupfs，处修改为 cgroupfs。此处设置很重要，否则后面node节点无法加入到集群 https://blog.csdn.net/Andriy_dangli/article/details/85062983
```
cat > /root/k8s/node/systemd/kubelet.json <<'HERE'
{
  "kind": "KubeletConfiguration",
  "apiVersion": "kubelet.config.k8s.io/v1beta1",
  "authentication": {
    "x509": {
      "clientCAFile": "/etc/kubernetes/ssl/ca.pem"
    },
    "webhook": {
      "enabled": true,
      "cacheTTL": "2m0s"
    },
    "anonymous": {
      "enabled": false
    }
  },
  "authorization": {
    "mode": "Webhook",
    "webhook": {
      "cacheAuthorizedTTL": "5m0s",
      "cacheUnauthorizedTTL": "30s"
    }
  },
  "address": "192.168.200.8",
  "port": 10250,
  "readOnlyPort": 10255,
  "cgroupDriver": "systemd",
  "hairpinMode": "promiscuous-bridge",
  "serializeImagePulls": false,
  "featureGates": {
    "RotateKubeletClientCertificate": true,
    "RotateKubeletServerCertificate": true
  },
  "clusterDomain": "cluster.local.",
  "clusterDNS": ["172.21.0.2"]
}
HERE
```

#### 3. 创建启动文件
```
cat > /root/k8s/node/systemd/kubelet.service <<'HERE'
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/usr/local/bin/kubelet \
  --bootstrap-kubeconfig=/etc/kubernetes/kubelet-bootstrap.kubeconfig \
  --cert-dir=/etc/kubernetes/ssl \
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
  --config=/etc/kubernetes/kubelet.json \
  --network-plugin=cni \
  --cni-conf-dir=/etc/cni/net.d \
  --pod-infra-container-image=k8s.gcr.io/pause:3.2 \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/log/kubernetes \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
HERE
```

#### 4. 分发配置文件和启动文件
```
# kubelet-bootstrap.kubeconfig
for node in node1 node2 node3 node4 node5;do scp -r /root/k8s/ssl/kubelet-bootstrap.kubeconfig $node:/etc/kubernetes/;done

# kubelet.json
for node in node1 node2 node3 node4 node5;do scp -r /root/k8s/node/systemd/kubelet.json $node:/etc/kubernetes/;done

# kube-scheduler.service
for node in node1 node2 node3 node4 node5;do scp -r /root/k8s/node/systemd/kubelet.service $node:/lib/systemd/system/;done

# 其他节点修改/etc/kubernetes/kubelet.json 的IP
sed -i 's@192.168.200.8@192.168.200.9@' /etc/kubernetes/kubelet.json
sed -i 's@192.168.200.8@192.168.200.10@' /etc/kubernetes/kubelet.json
sed -i 's@192.168.200.8@192.168.200.11@' /etc/kubernetes/kubelet.json
sed -i 's@192.168.200.8@192.168.200.12@' /etc/kubernetes/kubelet.json
```

#### 5. 启动 kubelet
```
# 创建 kubelet 数据目录
for node in node1 node2 node3 node4 node5;do ssh $node "mkdir -p /var/lib/kubelet";done

# 启动
for node in node1 node2 node3 node4 node5;do ssh $node "systemctl daemon-reload && systemctl start kubelet && systemctl enable kubelet && systemctl status kubelet";done
```

#### 6. workder加入集群
>kubelet服务启动成功后，接着到master上Approve一下bootstrap请求。执行如下命令可以看到worker节点发送CSR 请求
```
kubectl get csr

NAME                                                   AGE     SIGNERNAME                                    REQUESTOR           CONDITION
node-csr-9XZBBrGZ3ZcZjC9lqIdRl2Fn0gvhfKuAMeezRgXql8A   5m36s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
node-csr-AXEZ65YFmt_vKVq8puhpZJgTMOi12Uzq0WUz7oTQ6ZE   5m36s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
node-csr-J8uS6hmZYETrDwHeRHeBNtauyCOQx82WuxUbXkp7Sh0   52s     kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
node-csr-XaPhQT-q14lOzgsRTm1Z6MWiKXRhnL-ByadXlithFs0   57s     kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
node-csr-sK7uG0Jw7XRR4If2o3bdTswEofy9f0tAANSIxmQ2zLM   5m35s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending

# 通过证书签名请求
for i in $(kubectl get csr | awk 'NR!=1{print $0}' | awk '{print $1}');do kubectl certificate approve $i;done


# 查看节点，此时的状态是NotReady
kubectl get nodes -o wide
NAME           STATUS     ROLES    AGE     VERSION
docker-k8s01   NotReady   <none>   2m31s   v1.20.1
docker-k8s02   NotReady   <none>   2m31s   v1.20.1
docker-k8s03   NotReady   <none>   2m30s   v1.20.1
docker-k8s04   NotReady   <none>   2m30s   v1.20.1
docker-k8s05   NotReady   <none>   2m31s   v1.20.1

```

### 7.  部署 kube-proxy

[REFERENCE](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/)

#### 1.创建csr请求文件
```
cat >/root/k8s/ssl/kube-proxy-csr.json  <<'HERE'
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Shenzhen",
      "L": "Shenzhen",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
HERE
```
#### 2. 生成 kube-proxy 证书
```
# 生成证书
cd /root/k8s/ssl/ && cfssl gencert --ca ca.pem --ca-key ca-key.pem --config ca-config.json --profile kubernetes kube-proxy-csr.json | cfssljson --bare kube-proxy

# 分发到 Master
for node in node1 node2 node3 node4 node5;do scp -r /root/k8s/ssl/kube-proxy*.pem $node:/etc/kubernetes/ssl/;done
```
#### 3. 创建 kube-proxy 的kubeconfig
```
# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://192.168.200.88:8443 \
  --kubeconfig=kube-proxy.kubeconfig

# 设置客户端认证参数
kubectl config set-credentials kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

# 设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

# 设置默认上下文
kubectl config use-context default \
  --kubeconfig=kube-proxy.kubeconfig
```

#### 4. kube-proxy配置文件
>clusterCIDR: 172.21.0.0/16                           # 此处网段必须与service网段保持一致，否则部署
```
cat > /root/k8s/node/systemd/kube-proxy.yaml <<'HERE'
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 192.168.200.8
clientConnection:
  kubeconfig: /etc/kubernetes/kube-proxy.kubeconfig
clusterCIDR: 172.21.0.0/16
healthzBindAddress: 192.168.200.8:10256
kind: KubeProxyConfiguration
metricsBindAddress: 192.168.200.8:10249
mode: "ipvs"
HERE
```

#### 5. kube-proxy 启动文件
```
cat > /root/k8s/node/systemd/kube-proxy.service <<'HERE'
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/usr/local/bin/kube-proxy \
  --config=/etc/kubernetes/kube-proxy.yaml \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/log/kubernetes \
  --v=2
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
HERE
```

#### 6. 分发配置文件和启动文件
```
# kubelet-proxy.kubeconfig
for node in node1 node2 node3 node4 node5;do scp -r /root/k8s/ssl/kube-proxy.kubeconfig $node:/etc/kubernetes/;done

# kube-proxy.yaml
for node in node1 node2 node3 node4 node5;do scp -r  /root/k8s/node/systemd/kube-proxy.yaml $node:/etc/kubernetes/kube-proxy.yaml;done

# kube-proxy.service
for node in node1 node2 node3 node4 node5;do scp -r /root/k8s/node/systemd/kube-proxy.service $node:/lib/systemd/system/;done

# 其他节点修改/etc/kuberneteskube-proxy.yaml 的IP
sed -i 's@192.168.200.8@192.168.200.9@g' /etc/kubernetes/kube-proxy.yaml
sed -i 's@192.168.200.8@192.168.200.10@g' /etc/kubernetes/kube-proxy.yaml
sed -i 's@192.168.200.8@192.168.200.11@g' /etc/kubernetes/kube-proxy.yaml
sed -i 's@192.168.200.8@192.168.200.12@g' /etc/kubernetes/kube-proxy.yaml
```

#### 7. 启动 kube-proxy
```
# 创建工作目录
for node in node1 node2 node3 node4 node5;do ssh $node "mkdir -p /var/lib/kube-proxy";done

# 启动
for node in node1 node2 node3 node4 node5;do ssh $node "systemctl daemon-reload && systemctl start kube-proxy && systemctl enable kube-proxy && systemctl status kube-proxy";done
```

### 部署 Flannel
>此时的状态节点都是NoReady，是因为网络组件没有配置
```
# 创建网络需要的目录
mkdir /opt/cni/bin/ && mkdir /etc/cni/net.d
cd /opt/cni/bin
wget https://github.com/containernetworking/plugins/releases/download/v0.8.7/cni-plugins-linux-amd64-v0.8.7.tgz
tar -zxvf cni-plugins-linux-amd64-v0.8.7.tgz -C /opt/cni/bin/

wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

vim kube-flannel.yml  # 和kube-proxy里的网段一致
"Network": "172.21.0.0/16",

kubectl apply -f kube-flannel.yml

kubectl get pods -A

# 此时再来看各节点的状态
kubectl get nodes
NAME           STATUS   ROLES    AGE   VERSION
docker-k8s01   Ready    <none>   67m   v1.20.1
docker-k8s02   Ready    <none>   67m   v1.20.1
docker-k8s03   Ready    <none>   67m   v1.20.1

```

### 部署 Coredns
>kubernetes cluster.local in-addr.arpa ip6.arpa
>SUBDOMAIN去掉
>forward . /etc/resolv.conf
>clusterIP为：172.21.0.2（kubelet配置文件中的clusterDNS）
```
wget https://raw.githubusercontent.com/coredns/deployment/master/kubernetes/coredns.yaml.sed

mv coredns.yaml.sed coredns.yaml
vim coredns.yaml

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
        health {
          lameduck 5s
        }
        ready
        kubernetes cluster.local  in-addr.arpa ip6.arpa {
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf {
          max_concurrent 1000
        }
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
  # replicas: not specified here:
  # 1. Default is 1.
  # 2. Will be tuned in real time if DNS horizontal auto-scaling is turned on.
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
        kubernetes.io/os: linux
      affinity:
         podAntiAffinity:
           preferredDuringSchedulingIgnoredDuringExecution:
           - weight: 100
             podAffinityTerm:
               labelSelector:
                 matchExpressions:
                   - key: k8s-app
                     operator: In
                     values: ["kube-dns"]
               topologyKey: kubernetes.io/hostname
      containers:
      - name: coredns
        image: coredns/coredns:1.8.0
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
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
        readinessProbe:
          httpGet:
            path: /ready
            port: 8181
            scheme: HTTP
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
  clusterIP: 172.21.0.2
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



kubectl apply -f coredns.yaml
```

https://www.cnblogs.com/technology178/p/14295776.html
https://www.cnblogs.com/dinghc/p/13031436.html












