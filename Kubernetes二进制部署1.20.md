# Kubernetes二进制部署

[TOC]

## 环境

| 主机名      | IP          | 系统              | 用途                                        |
| ----------- | ----------- | ----------------- | ------------------------------------------- |
| k8s-master1 | 192.168.8.1 | CentOS 7.6 x86_64 | k8s主节点、etcd节点、kube-apiserver高可用LB |
| k8s-master2 | 192.168.8.2 | CentOS 7.6 x86_64 | k8s主节点、etcd节点、kube-apiserver高可用LB |
| k8s-master3 | 192.168.8.3 | CentOS 7.6 x86_64 | k8s主节点、etcd节点、kube-apiserver高可用LB |
| k8s-node1   | 192.168.8.4 | CentOS 7.6 x86_64 | k8s Worker节点                              |
| k8s-node2   | 192.168.8.5 | CentOS 7.6 x86_64 | k8s Worker节点                              |

 kube-apiserver的VIP地址使用`192.168.8.100`



## Selinux关闭

```shell
sed -i '/SELINUX=enforcing/s#.*#SELINUX=disabled/g' /etc/selinux/config
```



## Firewalld防火墙关闭

```shell
systemctl disable --now
```



## 禁用swap交换

```shell
swapoff -a
sysctl -w vm.swappiness=0
echo "vm.swappiness" >> /etc/sysctl.conf
```



## 配置limit

```shell
#文件描述符优化
cp -p /etc/security/limits.conf /etc/security/limits.conf.bak
sed -i  '/# End of file/i*	soft	nofile	65536' /etc/security/limits.conf
sed -i  '/# End of file/i*	hard    nofile	65536' /etc/security/limits.conf
sed -i  '/# End of file/i*	soft    nproc	65535' /etc/security/limits.conf
sed -i  '/# End of file/i*	hard    nproc65535\n' /etc/security/limits.conf
```



## 时间同步

```shell
# 安装ntpdate
yum -y install ntpdate

# 同步时间
ntpdate ntp.aliyun.com
echo "*/10 * * * * /usr/sbin/ntpdate -s ntp.aliyun.com" >> /var/spool/cron/root
```





## 配置master节点间无密码登录

```shell
# 在k8s-master1上执行
ssh-keygen -t rsa 2048
for i in $(seq 2 5);do ssh-copy-id 192.168.8.$i ;done
```



## 内核升级

```shell
# 导入Elrepo公钥信息
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org

# 安装Elrepo源
yum -y install https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm

# 查看可用的内核包
yum --disablerepo="*" --enablerepo="elrepo-kernel" list available

# 查看当前内核
uname -sr
Linux 3.10.0-1160.15.2.el7.x86_64

# 升级内核
yum --enablerepo=elrepo-kernel install kernel-lt
# 内核建议升级到4.19，elrepo可能找不到该内核，可以从http://193.49.22.109/elrepo/找

# 打印当前系统上所有可用内核
awk -F\' '$1=="menuentry " {print $2}' /etc/grub2.cfg
CentOS Linux (4.19.12-1.el7.elrepo.x86_64) 7 (Core)
CentOS Linux (3.10.0-1160.15.2.el7.x86_64) 7 (Core)
CentOS Linux (0-rescue-309db06d532b4c64ba14a884ebb40dc1) 7 (Core)

# 设置默认启动内核
grub2-set-default "CentOS Linux (4.19.12-1.el7.elrepo.x86_64) 7 (Core)"
```



## 加载ipvs模块

```shell
# 让系统开机自动加载模块
cat > /etc/modules-load.d/k8s.conf <<EOF
ip_vs
ip_vs_sh
ip_vs_rr
ip_vs_wrr
nf_conntrack

# 上述步骤需要重启才能生成，可使用modprobe手动加载不需要重启
modprobe ip_vs
modprobe ip_vs_sh
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe nf_conntrack
```



## 内核参数调整

```shell
cat >> /etc/sysctl.d/k8s-cri.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```





## 证书生成

```shell
# 下载cfssl证书生成工具
curl -L https://github.com/cloudflare/cfssl/releases/download/v1.5.0/cfssl_1.5.0_linux_amd64 -o /usr/local/bin/cfssl
curl -L https://github.com/cloudflare/cfssl/releases/download/v1.5.0/cfssljson_1.5.0_linux_amd64 -o /usr/local/bin/cfssljson

chmod +x /usr/local/bin/cfssl*

# 创建证书配置存在目录
mkdir -pv /usr/local/src/kubernetes/ssl/{k8s,etcd}

# 创建etcd-ca、kubernetes-ca和kubernetes-front-proxy-ca的ca证书csr签名请求文件，用于生成自签名CA
# 可使用cfssl print-defaults csr打印默认csr配置文件
cat > /usr/local/src/kubernetes/ssl/etcd/etcd-ca-csr.json <<EOF
{
    "CN": "etcd-ca",
    "key": {
        "algo": "rsa",
        "size": 4096
    },
    "names": [
        {
            "C": "CN",
            "ST": "GuangDong",
            "L": "ShenZhen"
        }
    ],
    # 添加该段，设置ca证书有效期，也可以不设置，默认ca证书只有5年有效期
    "ca": {
        "expiry": "876000h"
    }
}
EOF

cat > /usr/local/src/kubernetes/ssl/k8s/kubernetes-ca-csr.json <<EOF
{
    "CN": "kubernetes-ca",
    "key": {
        "algo": "rsa",
        "size": 4096
    },
    "names": [
        {
            "C": "CN",
            "ST": "GuangDong",
            "L": "ShenZhen"
        }
    ],
    # 添加该段，设置ca证书有效期，也可以不设置，默认ca证书只有5年有效期
    "ca": {
        "expiry": "876000h"
    }
}
EOF

cat > /usr/local/src/kubernetes/ssl/k8s/kubernetes-front-proxy-ca-csr.json <<EOF
{
    "CN": "kubernetes-front-proxy-ca",
    "key": {
        "algo": "rsa",
        "size": 4096
    },
    "names": [
        {
            "C": "CN",
            "ST": "GuangDong",
            "L": "ShenZhen"
        }
    ],
    # 添加该段，设置ca证书有效期，也可以不设置，默认ca证书只有5年有效期
    "ca": {
        "expiry": "876000h"
    }
}
EOF

# 创建证书配置文件，配置通用，可在配置文件中定义多个profiles，在创建证书时使用指定的profiles，server auth为服务端证书，client auth表示客户端证书，即有server auth又有client auth表示双向认证证书。可使用cfssl print-defaults config生成默认配置
cat > /usr/local/src/kubernetes/ssl/config.json <<EOF
{
    "signing": {
        "default": {
            "expiry": "876000h"
        },
        "profiles": {
            "server": {
                "expiry": "876000h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth"
                ]
            },
            "client": {
                "expiry": "876000h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "client auth"
                ]
            },
            "peer": {
                "expiry": "876000h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
EOF

# 创建kubue-etcd证书csr请求文件
cat > /usr/local/src/kubernetes/ssl/kubue-etcd-csr.json <<EOF
{
    "CN": "kube-etcd",
    # 一般只需要localhost和127.0.0.1就可以了，这样kube-apiserver只连接本机etcd，如果需要连接其它连接，则把etcd节点的IP地址加上就可以了
    "hosts": [
        "192.168.8.1",		# 为etcd节点IP
        "192.168.8.2",		# 为etcd节点IP
        "192.168.8.3",		# 为etcd节点IP
        "localhost",
        "127.0.0.1"
    ],
    "key": {
        "algo": "rsa",
        "size": 4096
    },
    "names": [
        {
            "C": "CN",
            "ST": "GuangDong",
            "L": "ShenZhen"
        }
    ]
}
EOF

# 创建kube-etcd-peer证书csr请求文件
cat > /usr/local/src/kubernetes/ssl/etcd/kube-etcd-peer-csr.json <<EOF
{
    "CN": "kube-etcd-peer",
    "hosts": [
        "192.168.8.1",	# 为etcd节点IP
        "192.168.8.2",
        "192.168.8.3",
        "localhost",
        "127.0.0.1"
    ],
    "key": {
        "algo": "rsa",
        "size": 4096
    },
    "names": [
        {
            "C": "CN",
            "ST": "GuangDong",
            "L": "ShenZhen"
        }
    ]
}
EOF

# 创建kube-apiserver-etcd-client证书csr请求文件
cat > /usr/local/src/kubernetes/ssl/etcd/kube-apiserver-etcd-client-csr.json <<EOF
{
    "CN": "kube-apiserver-etcd-client",
    "key": {
        "algo": "rsa",
        "size": 4096
    },
    "names": [
        {
            "C": "CN",
            "ST": "GuangDong",
            "L": "ShenZhen",
            "O": "system:masters"
        }
    ]
}
EOF

# kube-apiserver证书csr请求文件
cat > /usr/local/src/kubernetes/ssl/k8s/kube-apiserver-csr.json <<EOF
{
    "CN": "kube-apiserver",
    "hosts": [
        "kubernetes",
        "kubernetes.default",
        "kubernetes.default.svc",
        "kubernetes.default.svc.cluster",
        "kubernetes.default.svc.cluster.local",
        "127.0.0.1",
        "192.168.8.100",    # kube-apiserver高可用负载均衡IP
        "10.32.0.1"      # kubernetes集群service第一个IP地址，即kubernetes集群地址
    ],
    "key": {
        "algo": "rsa",
        "size": 4096
    },
    "names": [
        {
            "C": "CN",
            "ST": "GuangDong",
            "L": "ShenZhen"
        }
    ]
}
EOF

# kube-apiserver-kubelet-client证书csr请求文件
cat > /usr/local/src/kubernetes/ssl/k8s/kube-apiserver-csr.json <<EOF
{
    "CN": "kube-apiserver-kubelet-client",
    "key": {
        "algo": "rsa",
        "size": 4096
    },
    "names": [
        {
            "C": "CN",
            "ST": "GuangDong",
            "L": "ShenZhen",
            "O": "system:masters"
        }
    ]
}
EOF

# kube-controller-manager证书csr请求文件
cat > /usr/local/src/kubernetes/ssl/k8s/kube-controller-manager-csr.json <<EOF
{
    "CN": "system:kube-controller-manager",		# 对应clusterrolebinding的system:kube-controller-manager中subject部分kind为user的用户名
    "key": {
        "algo": "rsa",
        "size": 4096
    },
    "names": [
        {
            "C": "CN",
            "ST": "GuangDong",
            "L": "ShenZhen"
        }
    ]
}
EOF
# kube-scheduler证书csr请求文件
cat > /usr/local/src/kubernetes/ssl/k8s/kube-scheduler-csr.json <<EOF
{
    "CN": "system:kube-scheduler",	# 对应clusterrolebinding的system:kube-scheduler中subject部分kind为user的用户名
    "key": {
        "algo": "rsa",
        "size": 4096
    },
    "names": [
        {
            "C": "CN",
            "ST": "GuangDong",
            "L": "ShenZhen"
        }
    ]
}
EOF
# kube-proxy证书csr请求文件
cat > /usr/local/src/kubernetes/ssl/k8s/kube-proxy-csr.json <<EOF
{
    "CN": "system:kube-proxy",	# 对应clusterrolebinding的system:node-proxier中subject部分kind为user的用户名
    "key": {
        "algo": "rsa",
        "size": 4096
    },
    "names": [
        {
            "C": "CN",
            "ST": "GuangDong",
            "L": "ShenZhen"
        }
    ]
}
EOF

# kubernetes-admin证书csr请求文件，该证书用于与kube-apiserver做权限验证管理k8s集群使用，CN部分为用户名，可以随意，"names"节点下的"O"部分必须为"system:masters","O"对应clusterrolebinding的cluster-admin中subject部分kind为Group的组名，表示将CN对应的用户名加入到该组，从而具有管理员权限
cat > /usr/local/src/kubernetes/ssl/k8s/kubernetes-admin-csr.json <<EOF
{
    "CN": "kubernetes-admin",
    "key": {
        "algo": "rsa",
        "size": 4096
    },
    "names": [
        {
            "C": "CN",
            "ST": "GuangDong",
            "L": "ShenZhen",
            "O": "system:masters"
        }
    ]
}
EOF

# front-proxy-client证书csr请求文件，CN默认应该与kube-apiserver的--requestheader-allowed-names值相同
cat > /usr/local/src/kubernetes/ssl/k8s/kubernetes-front-proxy-ca-csr.json <<EOF
{
    "CN": "front-proxy-client",
    "key": {
        "algo": "rsa",
        "size": 4096
    },
    "names": [
        {
            "C": "CN",
            "ST": "GuangDong",
            "L": "ShenZhen"
        }
    ]
}
EOF



# 创建证书存放目录
mkdir -pv /etc/{etcd,kubernetes}/pki


# 生成etcd认证相关证书
cd /usr/local/src/kubernetes/ssl/etcd
# 生成etcd-ca CA证书
cfssl gencert \
-initca etcd-ca-csr.json \
| cfssljson -bare /etc/etcd/pki/etcd-ca

# 生成kubue-etcd双向认证证书，用于etcd集群和etcdctl客户端使用，profile指定为peer
cfssl gencert \
-ca /etc/etcd/pki/etcd-ca.pem \
-ca-key /etc/etcd/pki/etcd-ca-key.pem \
-config config.json \
-profile peer \
kube-etcd-csr.json \
| cfssljson -bare /etc/etcd/pki/kube-etcd

# 生成kube-etcd-peer双向认证证书，用于etcd集群内部通讯，profile指定为peer
cfssl gencert \
-ca /etc/etcd/pki/etcd-ca.pem \
-ca-key /etc/etcd/pki/etcd-ca-key.pem \
-config config.json \
-profile peer \
kube-etcd-peer-csr.json \
| cfssljson -bare /etc/etcd/pki/kube-etcd-peer

# 生成kube-apiserver-etcd-client客户端证书，用于kube-apiserver访问etcd集群用，profile指定为client
cfssl gencert \
-ca /etc/etcd/pki/etcd-ca.pem \
-ca-key /etc/etcd/pki/etcd-ca-key.pem \
-config config.json \
-profile client \
kube-apiserver-etcd-client-csr.json \
| cfssljson -bare /etc/kubernetes/pki/kube-apiserver-etcd-client


# 生成kubernetes认证相
cd /usr/local/src/kubernetes/ssl/k8s
# 生成kubernetes-ca CA证书
cfssl gencert -initca kubernetes-ca-csr.json | cfssljson -bare /etc/kubernetes/pki/kubernetes-ca

# 生成kubernetes-front-proxy-ca CA证书
cfssl gencert -initca kubernetes-front-proxy-ca-csr.json | cfssljson -bare /etc/kubernetes/pki/kubernetes-front-proxy-ca

# 生成kube-apiserver服务端证书,profile指定为server,用于kube-apiserver服务端认证
cfssl gencert \
-ca /etc/kubernetes/pki/kubernetes-ca.pem \
-ca-key /etc/kubernetes/pki/kubernetes-ca-key.pem 、
-config config.json \
-profile server kube-apiserver-csr.json \
| cfssljson -bare /etc/kubernetes/pki/kube-apiserver

# 生成kube-apiserver-kubelet-client客户端证书，profile指定为client
cfssl gencert \
-ca /etc/kubernetes/pki/kubernetes-ca.pem \
-ca-key /etc/kubernetes/pki/kubernetes-ca-key.pem \
-config config.json \
-profile client \
kube-apiserver-kubelet-client-csr.json \
| cfssljson -bare /etc/kubernetes/pki/kube-apiserver-kubelet-client

# 生成kube-controller-manager客户端证书，profile指定为client
cfssl gencert \
-ca /etc/kubernetes/pki/kubernetes-ca.pem \
-ca-key /etc/kubernetes/pki/kubernetes-ca-key.pem \
-config config.json \
-profile client \
kube-controller-manager-csr.json \
| cfssljson -bare /etc/kubernetes/pki/kube-controller-manager

# 生成kube-scheduler客户端证书，profile指定为client
cfssl gencert \
-ca /etc/kubernetes/pki/kubernetes-ca.pem \
-ca-key /etc/kubernetes/pki/kubernetes-ca-key.pem \
-config config.json \
-profile client \
kube-scheduler-csr.json \
| cfssljson -bare /etc/kubernetes/pki/kube-scheduler

# 生成kube-proxy客户端证书，profile指定为client
cfssl gencert \
-ca /etc/kubernetes/pki/kubernetes-ca.pem \
-ca-key /etc/kubernetes/pki/kubernetes-ca-key.pem \
-config config.json \
-profile client kube-proxy-csr.json \
| cfssljson -bare /etc/kubernetes/pki/kube-proxy

# 生成kubernetes-admin客户端证书，用于管理kubernetes集群，证书O为system:masters，表示证该用户加入到system:masters组
cfssl gencert \
-ca /etc/kubernetes/pki/kubernetes-ca.pem \
-ca-key /etc/kubernetes/pki/kubernetes-ca-key.pem \
-config config.json \
-profile client \
kubernetes-admin-csr.json \
| cfssljson -bare /etc/kubernetes/pki/kubernetes-admin


# 生成front-proxy-client证书
cfssl gencert \
-ca /etc/kubernetes/pki/kubernetes-front-proxy-ca.pem \
-ca-key /etc/kubernetes/pki/kubernetes-front-proxy-ca-key.pem \
-config config.json \
-profile client \
front-proxy-client-csr.json \
| cfssljson -bare /etc/kubernetes/pki/front-proxy-client


# 生成serviceaccout使用的私钥和公钥
openssl genrsa -out /etc/kubernetes/pki/sa.key 2048
openssl rsa -in /etc/kubernetes/pki/sa.key -pubout -out /etc/kubernetes/pki/sa.pub


# 复制证书到相关节点
for i in 2 3; do scp /etc/etcd/pki/* 192.168.8.$i:/etc/etcd/pki ; done
for i in 2 3; do scp /etc/kubernetes/pki/* 192.168.8.$i:/etc/kubernetes/pki/ ; done
for i in 4 5; do scp /etc/kubernetes/pki/kube-proxy 192.168.8.$i:/etc/kubernetes/pki/ ; done
```



## 创建基于TLS安全认证的etcd集群

```shell
# 在k8s-master1下载etcd
wget https://github.com/etcd-io/etcd/releases/download/v3.4.13/etcd-v3.4.13-linux-amd64.tar.gz

# 解压etcd、etcdctl到/usrl/local/bin目录
tar --strip-components=1 \
--no-same-owner \
-C /usr/local/bin \
-xf etcd-v3.4.13-linux-amd64.tar.gz etcd-v3.4.13-linux-amd64/{etcd,etcdctl}

# 创建配置文件
# k8s-master1节点etcd配置文件
cat > /etc/etcd/etcd.conf <<EOF
#[Member]
ETCD_NAME="etcd-node1"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.8.1:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.8.1:2379,http://127.0.0.1:2379"

#[Clustering]
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.8.1:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.8.1:2380"
ETCD_INITIAL_CLUSTER="etcd-node1=https://192.168.8.1:2380,etcd-node2=https://192.168.8.2:2380,etcd-node3=https://192.168.8.3:2380"
ETCD_INITIAL_CLUSTER_STATE="new"		# 创建一个新集群必须为new
ETCD_INITIAL_CLUSTER_TOKEN="yZAyShN8Z+Harw=="	# 集群通信token，三个节点设置一样

#[Security]
ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/etcd/pki/etcd-ca.pem"
ETCD_CERT_FILE="/etc/etcd/pki/kube-etcd.pem"
ETCD_KEY_FILE="/etc/etcd/pki/kube-etcd-key.pem"
ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/pki/etcd-ca.pem"
ETCD_PEER_CERT_FILE="/etc/etcd/pki/kube-etcd-peer.pem"
ETCD_PEER_KEY_FILE="/etc/etcd/pki/kube-etcd-peer-key.pem"
EOF

# k8s-master2节点etcd配置文件
cat > /etc/etcd/etcd.conf <<EOF
#[Member]
ETCD_NAME="etcd-node2"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.8.2:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.8.2:2379,http://127.0.0.1:2379"

#[Clustering]
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.8.2:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.8.2:2380"
ETCD_INITIAL_CLUSTER="etcd-node1=https://192.168.8.1:2380,etcd-node2=https://192.168.8.2:2380,etcd-node3=https://192.168.8.3:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="yZAyShN8Z+Harw=="

#[Security]
ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/etcd/pki/etcd-ca.pem"
ETCD_CERT_FILE="/etc/etcd/pki/kube-etcd.pem"
ETCD_KEY_FILE="/etc/etcd/pki/kube-etcd-key.pem"
ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/pki/etcd-ca.pem"
ETCD_PEER_CERT_FILE="/etc/etcd/pki/kube-etcd-peer.pem"
ETCD_PEER_KEY_FILE="/etc/etcd/pki/kube-etcd-peer-key.pem"
EOF


# k8s-master3节点etcd配置文件
cat > /etc/etcd/etcd.conf <<EOF
#[Member]
ETCD_NAME="etcd-node3"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.8.3:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.8.3:2379,http://127.0.0.1:2379"

#[Clustering]
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.8.3:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.8.3:2380"
ETCD_INITIAL_CLUSTER="etcd-node1=https://192.168.8.1:2380,etcd-node2=https://192.168.8.2:2380,etcd-node3=https://192.168.8.3:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="yZAyShN8Z+Harw=="

#[Security]
ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/etcd/pki/etcd-ca.pem"
ETCD_CERT_FILE="/etc/etcd/pki/kube-etcd.pem"
ETCD_KEY_FILE="/etc/etcd/pki/kube-etcd-key.pem"
ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/pki/etcd-ca.pem"
ETCD_PEER_CERT_FILE="/etc/etcd/pki/kube-etcd-peer.pem"
ETCD_PEER_KEY_FILE="/etc/etcd/pki/kube-etcd-peer-key.pem"
EOF

# 创建etcd的systemd配置文件
cat > /usr/lib/systemd/system/etcd.service <<EOF
[Unit]
Description=Etcd Server
Documentation=https://etcd.io/docs/
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=/etc/etcd/etcd.conf
WorkingDirectory=/var/lib/etcd
ExecStartPre=/usr/sbin/useradd -r -s /sbin/nologin etcd
ExecStartPre=/usr/bin/chown -R etcd.etcd /var/lib/etcd /etc/etcd/pki
ExecStart=/usr/local/bin/etcd
LimitNOFILE=65536
User=etcd
Group=etcd
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# 创建相关用户及目录(在k8s-master1、k8s-master2、k8s-master3上都执行)
useradd -s -r /sbin/nologin etcd
mkdir -pv /var/lib/etcd
chown -R etcd.etcd /var/lib/etcd /etc/etcd/pki

# 复制etcd、etcdctl、etcd.service到k8s-master2和k8s-master3
for i in 2 3; do scp /usr/local/bin/etcd* 192.168.8.$i:/usr/local/bin/ ; done
for i in 2 3; do scp /usr/lib/systemd/system/etcd.service 192.168.8.$i:/usr/lib/systemd/system/ ; done

# 启动etcd（3个master节点上执行）
systemctl daemon-reload
systemctl enable etcd --now

# 集群验证
cd /etc/etcd/pki
etcdctl \
--cacert /etcd/etcd/pki/etcd-ca.pem \
--cert /etcd/etcd/pki/kube-etcd.pem \
--key /etcd/etcd/pki/kube-etcd-key.pem \
--endpoints="https://192.168.8.1:2379,https://192.168.8.2:2379,https://192.168.8.3:2379" endpoint status -w table

etcdctl \
--cacert /etcd/etcd/pki/etcd-ca.pem \
--cert /etcd/etcd/pki/kube-etcd.pem \
--key /etcd/etcd/pki/kube-etcd-key.pem \
--endpoints="https://192.168.8.1:2379,https://192.168.8.2:2379,https://192.168.8.3:2379" member list
```



## kubernetes基本组件安装

```shell
# 下载kubernetes二进制包
wget https://dl.k8s.io/v1.20.3/kubernetes-server-linux-amd64.tar.gz
tar \
--strip-components=3 \
--no-same-owner \
-C /usr/local/bin \
-xf kubernetes-server-linux-amd64.tar.gz kubernetes/server/bin/kube{let,ctl,-apiserver,-proxy,-controller-manager,-scheduler}

# 复制相关组件到相关节点
for i in $(seq 2 5); do scp /usr/local/bin/kube{let,-proxy} 192.168.8.$i:/usr/local/bin ; done # 如果master节点不运行pod，可以不需要kubelet和kube-proxy
for i in 2 3;do scp/usr/local/bin/kube{ctl,-apiserver,-controller-manager,-scheduler} 192.168.8.$i:/usr/local/bin ; done
```



## kubernetes高可用组件部署

``` shell
# 安装nginx
wget http://nginx.org/download/nginx-1.18.0.tar.gz
tar -xf nginx-1.18.0.tar.gz
cd nginx-1.18.0
useradd -r -s /sbin/nologin nginx
./configure --prefix=/usr/local/nginx \
--conf-path=/etc/nginx/nginx.conf \
--error-log-path=/var/log/nginx/error.log \
--pid-path=/var/run/nginx.pid \
--lock-path=/var/lock/nginx.lock \
--user=nginx \
--group=nginx \
--without-http \
--without-http-cache \
--with-threads \
--with-stream \
--with-stream_realip_module
make -j $(nproc) && make install
echo 'export PATH=$PATH:/usr/local/nginx/sbin' > /etc/profile.d/nginx.sh
source /etc/profile.d/nginx.sh

# nginx配置
cat > /etc/nginx/nginx.conf <<EOF
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  2048;
    multi_accept on;
    use epoll;
}

stream {
    upstream kube-apiserver {
        server 192.168.8.1:6443 weight=1 max_fails=2 fail_timeout=10;
        server 192.168.8.2:6443 weight=1 max_fails=2 fail_timeout=10;
        server 192.168.8.3:6443 weight=1 max_fails=2 fail_timeout=10;
    }

    server {
        listen 8443;
        proxy_protocol_timeout 10s;
        proxy_pass kube-apiserver;
    }
}
EOF

# 创建nginx的systemd配置
cat > /usr/lib/systemd/system/nginx.service <<EOF
[Unit]
Description=nginx - high performance web server
Documentation=http://nginx.org/en/docs/
After=network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/var/run/nginx.pid
ExecStart=/usr/local/nginx/sbin/nginx -c /etc/nginx/nginx.conf
ExecReload=/bin/sh -c "/bin/kill -s HUP $(/bin/cat /var/run/nginx.pid)"
ExecStop=/bin/sh -c "/bin/kill -s TERM $(/bin/cat /var/run/nginx.pid)"

[Install]
WantedBy=multi-user.target
EOF

# 启动nginx
nginx -t
systemctl enable nginx --now

# 安装keepalived
yum -y install keepalived

# 配置keepalived
cat > /etc/keepalived/keepalived.conf <<EOF
! Configuration File for keepalived

global_defs {
    script_user root
    enable_script_security
}

vrrp_script chk_apiserver {
    script "/etc/keepalived/chk_apiserver.sh"
    interval 5
    weight -10
    rise 1
    fall 2
}

vrrp_instance VI_1 {
    state MASTER			# 其它节点配置为BACKUP
    interface ens33			# 根据实际情况配置，可能需要修改网卡名称
    virtual_router_id 88	# 虚拟路由IP不要跟局域网其它的冲突了
    priority 100			# 其它节点配置比它小一点，例如95
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8S_HA
    }
    virtual_ipaddress {
        192.168.8.100/24 dev ens33 label ens33:0	# 根据实际情况配置，可能需要修改网卡名称
    }
    
    track_script {
        chk_apiserver
    }
}
EOF

# 创建健康状态检测脚本
cat > /etc/keepalived/chk_apiserver.sh <<EOF
#!/bin/bash
# Author: 沉醉寒风
# Email: 957414423@qq.com
ERROR=0

# 检测Nginx进程是否存在
for i in $(seq 1 3)
do
    if [[ $(pgrep nginx) == "" ]];then
        ERROR=$(($ERROR+1))
        sleep 1
        continue
    else
        ERROR=0
        break
    fi
done

# ERROR变量不为零，表示检测不到Nginx进程，需要停止keepalived，让VIP飘移到其它正常节点
if [[ $ERROR != "0" ]];then
    /usr/bin/systemctl stop keepalived
    exit 1
else
    exit 0
fi
EOF
chmod +x /etc/keepalived/chk_apiserver.sh

# 启动keepalived
systemctl enable keepalived --now
```



## kubernetes部署

### 部署kube-apiserver服务

```shell
# 创建kube-apiserver配置文件
cat > /etc/kubernetes/kube-apiserver.conf <<EOF
KUBE_API_ARGS="--bind-address=0.0.0.0 \
--etcd-cafile=/etc/etcd/pki/etcd-ca.pem \
--etcd-certfile=/etc/kubernetes/pki/kube-apiserver-etcd-client.pem \
--etcd-keyfile=/etc/kubernetes/pki/kube-apiserver-etcd-client-key.pem \
--etcd-servers=https://192.168.8.1:2379,https://192.168.8.2:2379,https://192.168.8.3:2379 \
--secure-port=6443 \
--client-ca-file=/etc/kubernetes/pki/kubernetes-ca.pem \
--tls-cert-file=/etc/kubernetes/pki/kube-apiserver.pem \
--tls-private-key-file=/etc/kubernetes/pki/kube-apiserver-key.pem \
--enable-bootstrap-token-auth=true \
--kubelet-client-certificate=/etc/kubernetes/pki/kube-apiserver-kubelet-client.pem \
--kubelet-client-key=/etc/kubernetes/pki/kube-apiserver-kubelet-client-key.pem \
--kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname \
--proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.pem \
--proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client-key.pem \
--requestheader-allowed-names=front-proxy-client \
--requestheader-client-ca-file=/etc/kubernetes/pki/kubernetes-front-proxy-ca.pem \
--requestheader-extra-headers-prefix=X-Remote-Extra- \
--requestheader-group-headers=X-Remote-Group \
--requestheader-username-headers=X-Remote-User \
--service-account-issuer=https://kubernetes.default.svc.cluster.local \
--service-account-key-file=/etc/kubernetes/pki/sa.pub \
--service-account-signing-key-file=/etc/kubernetes/pki/sa.key \
--authorization-mode=Node,RBAC \
--allow-privileged \
--service-cluster-ip-range=10.32.0.0/12 \
--service-node-port-range=30000-32767 \
--enable-swagger-ui=true \
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,PersistentVolumeLabel,DefaultStorageClass,ResourceQuota,DefaultTolerationSeconds"
EOF

# 创建kube-apiserver的systemd服务脚本
cat > /usr/lib/systemd/system/kube-apiserver.service <<EOF
[Unit]
Description=Kube API Server
Documentation=https://kubernetes.io/docs/
After=network-online.target
Wants=network-online.target

[Service]
EnvironmentFile=/etc/kubernetes/kube-apiserver.conf
ExecStart=/usr/local/bin/kube-apiserver \$KUBE_API_ARGS
LimitNOFILE=65536
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# 启动kube-apiserver
systemctl enable kube-apiserver --now
```



### 部署kube-controller-manager服务

```shell
# 创建kube-controller-manager配置文件
cat > /etc/kubernetes/kube-controller-manager.conf <<EOF
KUBE_CONTROLLER_MANAGER_ARGS="--bind-address=127.0.0.1 \
--allocate-node-cidrs=true \
--authentication-kubeconfig=/etc/kubernetes/controller-manager-kubeconfig.conf \
--authorization-kubeconfig=/etc/kubernetes/controller-manager-kubeconfig.conf \
--client-ca-file=/etc/kubernetes/pki/kubernetes-ca.pem \
--cluster-cidr=10.16.0.0/12 \
--cluster-name=kubernetes \
--cluster-signing-cert-file=/etc/kubernetes/pki/kubernetes-ca.pem \
--cluster-signing-key-file=/etc/kubernetes/pki/kubernetes-ca-key.pem \
--cluster-signing-duration=43800h \
--controllers=*,bootstrapsigner,tokencleaner \
--kubeconfig=/etc/kubernetes/controller-manager-kubeconfig.conf \
--leader-elect=true \
--requestheader-client-ca-file=/etc/kubernetes/pki/kubernetes-front-proxy-ca.pem \
--root-ca-file=/etc/kubernetes/pki/kubernetes-ca.pem \
--service-account-private-key-file=/etc/kubernetes/pki/sa.key \
--service-cluster-ip-range=10.32.0.0/12 \
--use-service-account-credentials=true"
EOF

# 创建kube-controller-manager的systemd配置文件
cat > /usr/lib/systemd/system/kube-controller-manager.service <<EOF
[Unit]
Description=kube-controller-manager
Documentation=https://kubernetes.io/docs/
After=network-online.target
Wants=network-online.target

[Service]
EnvironmentFile=/etc/kubernetes/kube-controller-manager.conf
ExecStart=/usr/local/bin/kube-controller-manager \$KUBE_CONTROLLER_MANAGER_ARGS
LimitNOFILE=65536
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# 创建kube-controller-manager的kubeconfig文件
kubectl config set-cluster kubernetes --server=https://192.168.8.100:8443 --certificate-authority=/etc/kubernetes/pki/kubernetes-ca.pem --embed-certs --kubeconfig=/etc/kubernetes/controller-manager-kubeconfig.conf

kubectl config set-credentials system:kube-controller-manager --embed-certs --client-certificate=/etc/kubernetes/pki/kube-controller-manager.pem --client-key=/etc/kubernetes/pki/kube-controller-manager-key.pem --kubeconfig=/etc/kubernetes/controller-manager-kubeconfig.conf

kubectl config set-context system:kube-controller-manager@kubernetes --cluster=kubernetes --user=system:kube-controller-manager --kubeconfig=/etc/kubernetes/controller-manager-kubeconfig.conf

kubectl config use-context system:kube-controller-manager@kubernetes --kubeconfig=/etc/kubernetes/controller-manager-kubeconfig.conf

# 启动kube-controller-manager服务
systemctl enable kube-controller-manager --now
```



### 部署kube-scheduler服务

```shell
# 创建kube-scheduler配置文件
cat > /etc/kubernetes/kube-scheduler.conf <<EOF
KUBE_SCHEDULER_ARGS="--bind-address=127.0.0.1 \
--authentication-kubeconfig=/etc/kubernetes/scheduler-kubeconfig.conf \
--authorization-kubeconfig=/etc/kubernetes/scheduler-kubeconfig.conf \
--kubeconfig=/etc/kubernetes/scheduler-kubeconfig.conf \
--leader-elect=true"
EOF

# 创建kube-scheduler的systemd配置文件
cat > /usr/lib/systemd/system/kube-scheduler.service <<EOF
[Unit]
Description=kube-scheduler
Documentation=https://kubernetes.io/docs/
After=network-online.target
Wants=network-online.target

[Service]
EnvironmentFile=/etc/kubernetes/kube-scheduler.conf
ExecStart=/usr/local/bin/kube-scheduler \$KUBE_SCHEDULER_ARGS
LimitNOFILE=65536
RestartSec=10
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# 创建kube-scheduler的kubeconfig文件
kubectl config set-cluster kubernetes \
--server=https://192.168.8.100:8443 \
--certificate-authority=/etc/kubernetes/pki/kubernetes-ca.pem \
--kubeconfig=/etc/kubernetes/scheduler-kubeconfig.conf \
--embed-certs

kubectl config set-credentials system:kube-scheduler \
--client-certificate=/etc/kubernetes/pki/kube-scheduler.pem \
--client-key=/etc/kubernetes/pki/kube-scheduler-key.pem \
--kubeconfig=/etc/kubernetes/scheduler-kubeconfig.conf  \
--embed-certs

kubectl config set-context system:kube-scheduler@kubernetes \
--cluster=kubernetes \
--user=system:kube-scheduler \
--kubeconfig=/etc/kubernetes/scheduler-kubeconfig.conf

kubectl config use-context system:kube-scheduler@kubernetes \
--kubeconfig=/etc/kubernetes/scheduler-kubeconfig.conf

# 创建kube-scheduler的systemd配置文件
cat > /usr/lib/systemd/system/kube-scheduler.service <<EOF
[Unit]
Description=kube-scheduler
Documentation=https://kubernetes.io/docs/
After=network-online.target
Wants=network-online.target

[Service]
EnvironmentFile=/etc/kubernetes/kube-scheduler.conf
ExecStart=/usr/local/bin/kube-scheduler \$KUBE_SCHEDULER_ARGS
LimitNOFILE=65536
RestartSec=10
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# 启动kube-scheduler服务
systemctl daemon-reload
systemctl enable kube-scheduler --now
```



### 配置管理用户config

```shell
# 使用前面生成的kubeernetes-admin证书设置kubernetes-admin的kubeconfig，用于管理kubernetes集群
kubectl config set-cluster kubernetes \
--server=https://192.168.8.100:8443 \
--certificate-authority=/etc/kubernetes/pki/kubernetes-ca.pem \
--embed-certs

kubectl config set-credentials kubernetes-admin \
--client-certificate=/etc/kubernetes/pki/kubernetes-admin.pem \
--client-key=/etc/kubernetes/pki/kubernetes-admin-key.pem
--embed-certs

kubectl config set-context kubernetes-admin@kubernetes \
--cluster=kubernetes \
--user=kubernetes-admin

kubectl config use-context kubernetes-admin@kubernetes

# 验证，能正常连接集群表示成功
kubectl cluster-info
```



### TLS启动引导配置

官方文档参考：

https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/

https://kubernetes.io/zh/docs/reference/access-authn-authz/bootstrap-tokens/

https://kubernetes.io/zh/docs/reference/access-authn-authz/authentication/

https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/

https://github.com/kubernetes/community/blob/master/contributors/design-proposals/cluster-lifecycle/bootstrap-discovery.md

```shell
# 允许kublet TLS bootstrap创建CSR请求
kubectl create clusterrolebinding create-csrs-for-bootstrapping \
--clusterrole=system:node-bootstrapper \
--group=system:bootstrappers

# 允许kube-controller-manager自动为system:bootstrappers组的用户颁发证书
kubectl create clusterrolebinding auto-approve-csrs-for-group \
--clusterrole=system:certificates.k8s.io:certificatesigningrequests:nodeclient \
--group=system:bootstrappers

# 允许system:nodes组用户自动续期证书
kubectl create clusterrolebinding auto-approve-renewals-for-nodes \
--clusterrole=system:certificates.k8s.io:certificatesigningrequests:selfnodeclient \
--group=system:nodes

# 创建bootstrap-token，在master节点上执行
cat > create-bootstrap-token.sh <<EOF
#!/bin/bash
# Author: 沉醉寒风
# Email: 957414423@qq.com
#

# 指定 apiserver 地址
KUBE_APISERVER="https://192.168.8.100:8443"

# 生成 Bootstrap Token
TOKEN_ID=$(cat /dev/urandom | tr -dc 0-9a-z | head -c 6)
TOKEN_SECRET=$(cat /dev/urandom | tr -dc 0-9a-z | head -c 16)
BOOTSTRAP_TOKEN="\${TOKEN_ID}.\${TOKEN_SECRET}"
echo "Bootstrap Token: \${BOOTSTRAP_TOKEN}"

# 使用secret存储token 
cat > bootstrap-secret.yaml <<AAA
apiVersion: v1
kind: Secret
metadata:
  # 名字必须该格式
  name: bootstrap-token-\${TOKEN_ID}
  namespace: kube-system
# 必须该类型
type: bootstrap.kubernetes.io/token
stringData:
  description: "TLS引导令牌"
  token-id: \${TOKEN_ID}
  token-secret: \${TOKEN_SECRET}
  # expiration可选参数，设置token过期时间,时间格式为RFC33399格式2021-03-11T22:35:29+08:00
  expiration: $(date --rfc-3339=seconds -d "6 hours" | sed 's/ /T/')
  usage-bootstrap-authentication: "true"
  usage-bootstrap-signing: "true"
  #auth-extra-groups: system:bootstrappers:default-node-token
AAA
kubectl apply -f bootstrap-secret.yaml
EOF

bash create-bootstrap-token.sh
```



### kubelet服务部署

```shell
# 生成 kubelet bootstrap-kubeconfig引导文件，注意：这里的${TOKEN_ID}和${BOOTSTRAP_TOKEN}改成上面TLS引导配置部分创建的值
TOKEN_ID=$(cat bootstrap-secret.yaml | sed -n '/token-id/s#.*: ##p')
BOOTSTRAP_TOKEN=$(cat bootstrap-secret.yaml | sed -n '/token-secret/s#.*: ##p')

kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/pki/kubernetes-ca.pem \
--server=https://192.168.8.100:8443 \
--kubeconfig=/etc/kubernetes/bootstrap-kubeconfig \
--embed-certs=true

kubectl config set-credentials "system:bootstrap:${TOKEN_ID}" \
--token=${BOOTSTRAP_TOKEN} \
--kubeconfig=/etc/kubernetes/bootstrap-kubeconfig

kubectl config set-context default \
--cluster=kubernetes \
--user="system:bootstrap:${TOKEN_ID}" \
--kubeconfig=/etc/kubernetes/bootstrap-kubeconfig

kubectl config use-context default --kubeconfig=/etc/kubernetes/bootstrap-kubeconfig


# 创建kubelet配置文件
cat > /etc/kubernetes/kubelet.conf <<EOF
KUBELET_ARGS="--config=/etc/kubernetes/kubelet.yaml \
--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubeconfig \
--kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
--network-plugin=cni \
--cni-conf-dir=/etc/cni/net.d \
--pod-infra-container-image=liaoronghui/pause:3.2"
EOF

# 创建kubelet配置文件，即--config指定的配置，kubelet有些配置会逐渐的弃用命令行，改为这种方式加载
cat > /etc/kubernetes/kubelet.yaml <<EOF
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 0s
    cacheUnauthorizedTTL: 0s
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
    cacheTTL: 0s
  x509:
    clientCAFile: /etc/kubernetes/pki/kubernetes-ca.pem
cgroupDriver: systemd
rotateCertificates: true
#serverTLSBootstrap: true
staticPodPath: /etc/kubernetes/manifests
clusterDomain: cluster.local
clusterDNS:
- 10.32.0.10
maxPods: 110
EOF


# 创建kubelet的systemd配置
cat > /usr/lib/systemd/system/kubelet.service <<EOF
[Unit]
Description=Kubernetes kubelet
Documentation=https://kubernetes.io/docs/
After=network-online.target
Wants=network-online.target

[Service]
EnvironmentFile=/etc/kubernetes/kubelet.conf
ExecStartPre=/bin/mkdir /etc/kubernetes/manifests
ExecStart=/usr/local/bin/kubelet \$KUBELET_ARGS
LimitNOFILE=65536
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# 启动kubelet
systemctl daemon-reload
systemctl enable kubelet --now
```



### kube-proxy服务部署

```shell
# 创建kube-proxy配置文件
cat > /etc/kubernetes/kube-proxy.yaml <<EOF
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
clientConnection:
  kubeconfig: /etc/kubernetes/kube-proxy.kubeconfig
bindAddress: 0.0.0.0
clusterCIDR: "10.16.0.0/12"		# pod的网段
mode: "ipvs"
ipvs:
  excludeCIDRs: null
  minSyncPeriod: 0s
  scheduler: "rr"
  strictARP: false
  syncPeriod: 30s
  tcpFinTimeout: 0s
  tcpTimeout: 0s
  udpTimeout: 0s
EOF

# 创建kube-proxy的kubeconfig文件
kubectl config set-cluster kubernetes \
--server=https://192.168.8.100:8443 \
--certificate-authority=/etc/kubernetes/pki/kubernetes-ca.pem \
--kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \
--embed-certs

kubectl config set-credentials system:kube-proxy \
--client-certificate=/etc/kubernetes/pki/kube-proxy.pem \
--client-key=/etc/kubernetes/pki/kube-proxy-key.pem \
--kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \
--embed-certs

kubectl config set-context system:kube-proxy@kubernetes \
--cluster=kubernetes \
--user=system:kube-proxy \
--kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig

kubectl config use-context system:kube-proxy@kubernetes \
--kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig


# 创建kube-proxy的systemd配置文件
cat > /usr/lib/systemd/system/kube-proxy.service <<EOF
[Unit]
Description=Kubernetes Proxy
Documentation=https://kubernetes.io/docs/
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/kube-proxy --config=/etc/kubernetes/kube-proxy.yaml
LimitNOFILE=65536
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# 启动kube-proxy服务
systemctl daemon-reload
systemctl enable kube-proxy --now
```



### 网络插件安装

```shell
# 以下安装根据实际情况选择一种即可
# 使用kubernetes Api数据存储库（不超过50个节点）安装calico
curl https://docs.projectcalico.org/manifests/calico.yaml -O

# 如果pid网络不是使用192.168.0.0/16则取消CALICO_IPV4POOL_CIDR变量的注释，并把值设置为pod的网段
sed -i '/CALICO_IPV4POOL_CIDR/,+1c\            - name: CALICO_IPV4POOL_CIDR\n              value: "10.16.0.0/12"' calico.yaml

# 部署calico
kubectl apply -f calico.yaml



# 使用kubernetes Api数据存储库（超过50个节点）安装calico
curl https://docs.projectcalico.org/manifests/calico-typha.yaml -o calico.yaml

# 如果pid网络不是使用192.168.0.0/16则取消CALICO_IPV4POOL_CIDR变量的注释，并把值设置为pod的网段
sed -i '/CALICO_IPV4POOL_CIDR/,+1c\            - name: CALICO_IPV4POOL_CIDR\n              value: "10.16.0.0/12"' calico.yaml

# 部署calico
kubectl apply -f calico.yaml


# 使用etcd存储安装calico
curl https://docs.projectcalico.org/manifests/calico-etcd.yaml -o calico.yaml

# 如果pid网络不是使用192.168.0.0/16则取消CALICO_IPV4POOL_CIDR变量的注释，并把值设置为pod的网段
sed -i '/CALICO_IPV4POOL_CIDR/,+1c\            - name: CALICO_IPV4POOL_CIDR\n              value: "10.16.0.0/12"' calico.yaml

# 修改etcd TLS认证配置
sed -i "/# etcd-key/c\  etcd-key: $(cat /etc/kubernetes/pki/kube-apiserver-etcd-client-key.pem | base64 -w 0)/" calico-etcd.yaml 
sed -i "/# etcd-cert/c\  etcd-cert: $(cat /etc/kubernetes/pki/kube-apiserver-etcd-client.pem | base64 -w 0)/" calico-etcd.yaml 
sed -i "/# etcd-ca/c\  etcd-key: $(cat /etc/etcd/pki/etcd-ca.pem | base64 -w 0)/" calico-etcd.yaml

sed -i \
'/  etcd_endpoints:/c\  etcd_endpoints: "https://192.168.8.1:2379,https://192.168.8.2:2379,https://192.168.8.3:2379"' \
calico-etcd.yaml
sed -i '/etcd_ca:/c\  etcd_ca: "/calico-secrets/etcd-ca"' calico-etcd.yaml
sed -i '/etcd_cert:/c\  etcd_cert: "/calico-secrets/etcd-cert"' calico-etcd.yaml
sed -i '/etcd_key:/c\  etcd_key: "/calico-secrets/etcd-key"' calico-etcd.yaml


# 部署calico
kubectl apply -f calico.yaml
```



### CoreDNS插件安装

```shell
# 下载CoreDNS部署的源代码
yum -y install git
git clone https://github.com/coredns/deployment.git

# 安装CoreDNS
cd deployment/kubernetes
./deploy.sh -s -i 10.32.0.10 | kubectl apply -f -		# 10.32.0.10为DNS地址

# DNS解析测试，注意：如果使用busybox镜像测试需要1.28及以下版本，1.28以上版本nslookup解析有问题
kubectl run test --image=busybox:1.28 --rm -it  /bin/sh
If you don't see a command prompt, try pressing enter.
/ # nslookup kubernetes.default
Server:    10.32.0.10
Address 1: 10.32.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes.default
Address 1: 10.32.0.1 kubernetes.default.svc.cluster.local
```



### metrics-server插件安装

```shell
# 安装metrics-server
curl -L https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.4.2/components.yaml -o metrics-server.yaml
# 跳过kubelet服务端TLS认证，在kubelet没有启用服务端验证的情况会不修改metrics-server会启动不起来，可以不修改的情况下查看日志可以发现
sed -i '/secure-port/a\        - --kubelet-insecure-tls' metrics-server.yaml
kubectl apply -f metrics-server.yaml

# 查看运行状态
kubectl get pods -l k8s-app=metrics-server -n kube-system

# 验证是否正常收集集群度量值
kubectl top nodes
kubectl top pods -A
```



### Kubernetes Dashboad安装

```shell
# 安装Dashboad
curl https://raw.githubusercontent.com/kubernetes/dashboard/v2.2.0/aio/deploy/recommended.yaml -o kubernetes-dashboad.yaml
kubectl apply -f kubernetes-dashboad.yaml

# Dashboad登录认证
# 创建ServiceAccout dashboard-admin授予ServicAccout dashboard-admin管理员权限，注意：生产环境谨慎使用管理员权限，建议授权特定某个名称空间权限
cat > dashboard-admin.yaml << EOF 
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: kubernetes-dashboard
  name: dashboard-admin
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dashboard-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - name: dashboard-admin
    kind: ServiceAccount
    namespace: kubernetes-dashboard
EOF

kubectl apply -f dashboard-admin.yaml

# Token方式，在Dashboad UI将选择Token登录方式，然后将打印出的token在Dashboad UI中填入登录
SECRET=$(kubectl get sa -n kubernetes-dashboard dashboard-admin -o jsonpath={.secrets[0].name})
kubectl get secret ${SECRET} -n kubernetes-dashboard -o jsonpath={.data.token} | base64 -d	# 将打印出的token在Dashboad UI使用登录


# kubeconfig方式，在Dashboad UI将选择Kubeconfig登录方式，然后选择下面生成的kubeconfig
SECRET=$(kubectl get sa -n kubernetes-dashboard dashboard-admin -o jsonpath={.secrets[0].name})
TOKEN=$(kubectl get secret ${SECRET} -n kubernetes-dashboard -o jsonpath={.data.token} | base64 -d)

kubectl config set-cluster kubernetes \
--server=https://192.168.8.100:8443 \
--certificate-authority=/etc/kubernetes/pki/kubernetes-ca.pem \
--kubeconfig=/etc/kubernetes/dashboard-admin.kubeconfig --embed-certs

kubectl config set-credentials dashboard-admin --token=${TOKEN} --kubeconfig=/etc/kubernetes/dashboard-admin.kubeconfig

kubectl config set-context dashboard-admin \
--cluster=kubernetes \
--user=dashboard-admin \
--kubeconfig=/etc/kubernetes/dashboard-admin.kubeconfig

kubectl config use-context dashboard-admin --kubeconfig=/etc/kubernetes/dashboard-admin.kubeconfig

```

