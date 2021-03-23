[TOC]

# 一、kubernetes 用户认证

[访问API](https://kubernetes.io/zh/docs/reference/access-authn-authz/)

> API请求过程：用户认证 -> 授权 ->准入控制

[REFENERCE](https://kubernetes.io/zh/docs/reference/access-authn-authz/authentication/#users-in-kubernetes)

## kubernetes 用户
- Kubernetes中有两种用户(User)
服务账号(ServiceAccount)和普通意义上的用户(User)
ServiceAccount是由K8S管理的，而User通常是在外部管理，K8S不存储用户列表——也就是说，添加/编辑/删除用户都是在外部进行，无需与K8S API交互，虽然K8S并不管理用户，但是在K8S接收API请求时，是可以认知到发出请求的用户的，实际上，所有对K8S的API请求都需要绑定身份信息(User或者ServiceAccount)，这意味着，可以为User配置K8S集群中的请求权限

- 两种用户有什么区别？
最主要的区别上面已经说过了，即ServiceAccount是K8S内部资源，而User是独立于K8S之外的。
User通常是人来使用，而ServiceAccount是某个服务/资源/程序使用的
User独立在K8S之外，也就是说User是可以作用于全局的，在任何命名空间都可被认知，并且需要在全局唯一，而ServiceAccount作为K8S内部的某种资源，是存在于某个命名空间之中的，在不同命名空间中的同名ServiceAccount被认为是不同的资源
K8S不会管理User，所以User的创建/编辑/注销等，需要依赖外部的管理机制，K8S所能认知的只有一个用户名 ServiceAccount是由K8S管理的，创建等操作，都通过K8S完成

这里说的添加用户指的是普通意义上的用户，即存在于集群外的用户，为k8s的使用者。
实际上叫做添加用户也不准确，用户早已存在，这里所做的只是使K8S能够认知此用户，并且控制此用户在集群内的权限。

## API 认证的几种方式
Kubernetes 用户使用 client certificates, bearer tokens, an authenticating proxy, or HTTP basic auth 来认证 API 请求的身份。HTTP 请求发给 API 服务器时， 插件会将以下属性关联到请求本身：
- Username: 用户名比如 kube-admin or jane@example.com.
- UID: 用户UID比用户名有更好的一致性和唯一性.
- Groups：用户组比如 system:masters or devops-team.
- Extra fields: 一组额外的键值对，用来保存一些额外的信息
- 
可以同时启用多种身份认证方法，至少使用两种方法：
- 针对 Serviceaccount 使用Serviceaccount Token
- 至少一种方法对User的身份进行认证

### X509 Client Certs 客户端证书
客户端证书验证通过为API Server指定 --client-ca-file=SOMEFILE 选项启用，API Server通过此ca文件来验证API请求携带的客户端证书的有效性，一旦验证成功，API Server就会将客户端证书Subject里的CN属性作为此次请求的用户名，客户端证书还可以通过证书的 organization 字段标明用户的组成员信息。

### Static Token File 静态token文件
API Server 通过指定 --token-auth-file=SOMEFILE 选项来启用bearer token验证方式，token文件是一个 CSV 文件，包含了 token,用户名,用户ID 的csv文件，比如token,user,uid,"group1,group2,group3"，例如：如果持有者 token为 31ada4fd-adec-460c-809a-9e56ceb75269，则其 出现在 HTTP 头部时如下所示：Authorization: Bearer 31ada4fd-adec-460c-809a-9e56ceb75269头信息即可通过bearer token验证.

### Bootstrap Tokens 启动引导 token

[官方文档](https://kubernetes.io/zh/docs/reference/access-authn-authz/bootstrap-tokens/)

API Server 通过制定 --enable-bootstrap-token-auth 选项启用用于 API 服务器请求的身份认证，一种简单的持有者令牌（Bearer Token），这种令牌是在新建集群 或者在现有集群中添加新节点时使用的。 它被设计成能够支持 kubeadm， 但是也可以被用在其他的案例中以便用户在不使用 kubeadm 的情况下启动集群。 它也被设计成可以通过 RBAC 策略，结合 Kubelet TLS 启动引导 系统进行工作。启动引导令牌被定义成一个特定类型的 Secret （bootstrap.kubernetes.io/token）， 并存在于 kube-system 名字空间中。 这些 Secret 会被 API 服务器上的启动引导认证组件（Bootstrap Authenticator）读取。 控制器管理器中的控制器 TokenCleaner 能够删除过期的令牌。 这些令牌也被用来在节点发现的过程中会使用的一个特殊的 ConfigMap 对象。 BootstrapSigner 控制器也会使用这一 ConfigMap。

令牌格式使用 abcdef.0123456789abcdef 的形式。 更加规范地说，它们必须符合正则表达式 [a-z0-9]{6}\.[a-z0-9]{16}。令牌的第一部分是 "Token ID"，它是一种公开信息，用于引用令牌并确保不会 泄露认证所使用的秘密信息。 第二部分是"令牌秘密（Token Secret）"，它应该被共享给受信的第三方。

令牌认证为用户名 system:bootstrap:\<token id\> 并且是组 system:bootstrappers 的成员。额外的组信息可以通过令牌的 Secret 来设置。

过期的令牌可以通过启用控制器管理器中的 tokencleaner 控制器来删除。

### Service Account Tokens
服务账号（Service Account）是一种自动被启用的用户认证机制，使用经过签名的 持有者令牌来验证请求。该插件可接受两个可选参数：

- --service-account-key-file 一个包含用来为持有者令牌签名的 PEM 编码密钥。 若未指定，则使用 API 服务器的 TLS 私钥。
- --service-account-lookup 如果启用，则从 API 删除的令牌会被回收。

服务账号通常由 API 服务器自动创建并通过 ServiceAccount 准入控制器 关联到集群中运行的 Pod 上。 持有者令牌会挂载到 Pod 中可预知的位置，允许集群内进程与 API 服务器通信。 服务账号也可以使用 Pod 规约的 serviceAccountName 字段显式地关联到 Pod 上。serviceAccountName 通常会被忽略，因为关联关系是自动建立的。

在集群外部使用服务账号持有者令牌也是完全合法的，且可用来为长时间运行的、需要与 Kubernetes API 服务器通信的任务创建标识。要手动创建服务账号，可以使用 kubectl create serviceaccount <名称> 命令。此命令会在当前的名字空间中生成一个 服务账号和一个与之关联的 Secret。

### Kube-apiserver 启动配置参数

--client-ca-file=/etc/kubernetes/ssl/ca.pem  # 开启 x509证书认证
--token-auth-file=/etc/kubernetes/ssl/token.csv  # 开启 bearer token 认证
--enable-bootstrap-token-auth  # 开启 bootstrap token 认证
--tls-cert-file=/etc/kubernetes/ssl/kube-apiserver.pem  # https自签证书
--tls-private-key-file=/etc/kubernetes/ssl/kube-apiserver-key.pem # 私钥
--authorization-mode=RBAC,Node  # 开启 RBAC，Node授权
--kubelet-client-certificate=/etc/kubernetes/ssl/kube-apiserver.pem
--kubelet-client-key=/etc/kubernetes/ssl/kube-apiserver-key.pem
--service-account-key-file=/etc/kubernetes/ssl/sa.pub  # 开启 service account认证
--service-account-signing-key-file=/etc/kubernetes/ssl/sa.key  # 如果此参数未指定，则使用--tls-private-key-file。 --service-account-signing-key-file 提供密钥


# 二、kubernetes RBAC授权
[REFERENCE](https://kubernetes.io/zh/docs/reference/access-authn-authz/rbac/)

基于角色（Role）的访问控制（RBAC）是一种基于组织中用户的角色来调节控制对 计算机或网络资源的访问的方法。RBAC 鉴权机制使用 rbac.authorization.k8s.io API 组 来驱动鉴权决定，允许你通过 Kubernetes API 动态配置策略。kube-apiserver 配置文件中定义 --authorization-mode=Example,RBAC 开启

## API 对象
RBAC API 声明了四种 Kubernetes 对象：Role、ClusterRole、RoleBinding 和 ClusterRoleBinding。

### Role 和 ClusterRole
RBAC 的 Role 或 ClusterRole 中包含一组代表相关权限的规则。 这些权限是纯粹累加的（不存在拒绝某操作的规则）。Role 总是用来在某个名字空间 内设置访问权限；在你创建 Role 时，你必须指定该 Role 所属的名字空间。
与之相对，ClusterRole 则是一个集群作用域的资源。这两种资源的名字不同（Role 和 ClusterRole）是因为 Kubernetes 对象要么是名字空间作用域的，要么是集群作用域的， 不可两者兼具。ClusterRole 有若干用法。你可以用它来：

- 定义对某名字空间域对象的访问权限，并将在各个名字空间内完成授权；
- 为名字空间作用域的对象设置访问权限，并跨所有名字空间执行授权；
- 为集群作用域的资源定义访问权限。
- 
如果你希望在名字空间内定义角色，应该使用 Role； 如果你希望定义集群范围的角色，应该使用 ClusterRole

### RoleBinding 和 ClusterRoleBinding
角色绑定（Role Binding）是将角色中定义的权限赋予一个或者一组用户。 它包含若干 主体（用户、组或服务账户）的列表和对这些主体所获得的角色的引用。 RoleBinding 在指定的名字空间中执行授权，而 ClusterRoleBinding 在集群范围执行授权。

一个 RoleBinding 可以引用同一的名字空间中的任何 Role。 或者，一个 RoleBinding 可以引用某 ClusterRole 并将该 ClusterRole 绑定到 RoleBinding 所在的名字空间。 如果你希望将某 ClusterRole 绑定到集群中所有名字空间，你要使用 ClusterRoleBinding。

# 三、kubernetes 准入控制
[REFERENCE](https://kubernetes.io/zh/docs/reference/access-authn-authz/admission-controllers/)

> 准入控制器是一段代码，它会在请求通过认证和授权之后、对象被持久化之前拦截到达 API 服务器的请求。控制器由下面的[列表](https://kubernetes.io/zh/docs/reference/access-authn-authz/admission-controllers/#what-does-each-admission-controller-do)组成， 并编译进 `kube-apiserver` 二进制文件，并且只能由集群管理员配置。 在该列表中，有两个特殊的控制器：MutatingAdmissionWebhook 和 ValidatingAdmissionWebhook。 它们根据 API 中的配置，分别执行变更和验证 [准入控制 webhook](https://kubernetes.io/zh/docs/reference/access-authn-authz/extensible-admission-controllers/#admission-webhooks)。准入控制器可以执行 “验证（Validating）” 和/或 “变更（Mutating）” 操作。 变更（mutating）控制器可以修改被其接受的对象；验证（validating）控制器则不行。准入控制过程分为两个阶段。第一阶段，运行变更准入控制器。第二阶段，运行验证准入控制器。 再次提醒，某些控制器既是变更准入控制器又是验证准入控制器。

kube-apiserver 配置文件中定义：

--enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,ResourceQuota

这里没有用到 MutatingAdmissionWebhook 和 ValidatingAdmissionWebhook


# 四、例子：X509认证

> x509 证书认证，用户创建 -> RBAC授权
证书认证，表现在 kubeconfig 配置文件

### 1. 创建用户证书签名请求
> 两种方法：1.cfssl生成  2.openssl生成，看ca用哪种方法生成的,这里CA是用cfssl生成的，所以这里也用cfssl
#### (1).cfssl生成
```shell
cd /etc/kubernetes/ssl/
cat > xiaozhiqi-csr.json  <<EOF
{
  "CN": "xiaozhiqi",
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
EOF

cfssl gencert --ca ca.pem --ca-key ca-key.pem --config ca-config.json --profile kubernetes xiaozhiqi-csr.json | cfssljson --bare xiaozhiqi

openssl x509 -in xiaozhiqi.pem -text -noout

ls xiaozhiqi*
xiaozhiqi.csr  xiaozhiqi-csr.json  xiaozhiqi-key.pem  xiaozhiqi.pem
```
#### (2).openssl生成
```shell
# 创建一个叫xiaozhiqi的私钥
(umask 077; openssl genrsa -out xiaozhiqi.key 2048)

# 基于私钥生成证书签名请求
openssl req -new -key xiaozhiqi.key -out xiaozhiqi.csr -subj "/CN=xiaozhiqi"

# 使用ca进行签名
openssl x509 -req -in xiaozhiqi.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out xiaozhiqi.crt -days 3650000
```

### 2.生成用户kubeconfig
> 这个 xiaozhiqi 怎么理解呢？
> 在搭建K8s集群的时候kube-apiserver创建了kubernetes集群，kubectl 使用 xiaozhiqi的配置文件，操作kubernetes集群内的 namespace 内的资源

```shell
# 设置xiaozhiqi为集群用户
kubectl config set-credentials xiaozhiqi \
--client-certificate=xiaozhiqi.pem \
--client-key=xiaozhiqi-key.pem \
--embed-certs=true \
--kubeconfig=xiaozhiqi.kubeconfig

# 设置上下文将xiaozhiqi用户和kubertest这个集群绑定在一起
kubectl config set-context xiaozhiqi \
--cluster=kubernetes \
--user=xiaozhiqi \
--kubeconfig=xiaozhiqi.kubeconfig

# 切换到 xiaozhiqi 这个上下文
kubectl config use-context xiaozhiqi

# 查看现在的配置，当前的上下文是 xiaozhiqi
kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: REDACTED
    server: https://192.168.200.88:8443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: admin
  name: kubernetes
- context:
    cluster: kubernetes
    user: xiaozhiqi
  name: xiaozhiqi
current-context: xiaozhiqi
kind: Config
preferences: {}
users:
- name: admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
- name: xiaozhiqi
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED


# 由于xiaozhiqi这个没有绑定role所以使用 kubectl 都没权限
kubectl get pods
No resources found.
Error from server (Forbidden): pods is forbidden: User "xiaozhiqi" cannot list pods in the namespace "default"

# 先切回 admin 这个上下文中
kubectl config use-context admin
```

### 3. Role、Rolebinding授权
xiaozhiqi 这个用户没有绑定Role所以没权限对Namespace进行操作，现在创建Role和Rolebinding，xiaozhiqi这个k8s用户就能对当前 Namespace 有get,list,watch的权限，其他 namespace 是没有权限的。Rolebinding 没有指定 namespace 那么就是默认 default 这个名称空间，如果有指定的话，那么 xiaozhiqi 用户只能在指定的名称空间，进行 get ,list , watch 操作

#### (1) 创建 Role
```shell
kubectl create role pods-reader --verb=get,list,watch --resource=pods -o yaml --dry-run > xiaozhiqi-role.yaml

cat > role-demo.yaml<<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pods-reader
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
  - list
  - watch
EOF

kubectl apply -f xiaozhiqi-role.yaml
kubectl get role
kubectl describe role pods-reader
```

#### (2) 创建 Rolebinding
```shell
kubectl create rolebinding xiaozhiqi-read-pods --role=pods-reader --user=xiaozhiqi -o yaml --dry-run > manifests/rolebinding-demo.yaml

cat > xiaozhiqi-rolebinding.yaml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: xiaozhiqi-read-pods
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pods-reader
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: xiaozhiqi
EOF

kubectl apply -f manifests/rolebinding-demo.yaml 
kubectl get rolebinding
kubectl describe rolebinding xiaozhiqi-read-pods
```
### 4. Role、Rolebinding验证
```shell
# 切回 xiaozhiqi 这个上下文
kubectl config use-context xiaozhiqi

kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
nginx-sa1                    1/1       Running   0          13h
nginxtest-6c544f5669-n77hg   1/1       Running   0          14h

# 创建Role的时候没有指定Namespace，所以只有default名称空间的权限
kubectl get pods -n kube-system
No resources found.
Error from server (Forbidden): pods is forbidden: User "xiaozhiqi" cannot list pods in the namespace "kube-system"

# 切回 admin 上下文
kubectl config use-context kubernetes

# 删掉 rolebinding, 后续将 clustrole 用 clusterrolebinding 
kubectl delete rolebinding xiaozhiqi-read-pods
```

### 5. ClusterRole、ClusterRolebinding授权
这里 k8s 用户 xiaozhiqi 使用了 ClusterRolebinding 绑定 ClusterRole，xiaozhiqi这个k8s用户就能对所有 Namespace 都有get,list,watch的权限,ClusterRolebinding是针对集群的所以不需要指定名称空间 namespace

#### (1)创建ClusterRole
```shell
kubectl create clusterrole cluster-reader --verb=get,list,watch --resource=pods -o yaml --dry-run > clusterrole-demo.yaml

cat > xiaozhiqi-clusterrole.yaml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-reader
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
  - list
  - watch
EOF

kubectl apply -f xiaozhiqi-clusterrole.yaml
kubectl get clusterrole
kubectl describe clusterrole cluster-reader
```
#### (2)创建ClusterRolebinding
```shell
kubectl create clusterrolebinding xiaozhiqi-read-all-pods --clusterrole=cluster-reader --user=xiaozhiqi --dry-run -o yaml > xiaozhiqi-clusterrolebinding.yaml

cat > xiaozhiqi-clusterrolebinding.yaml <<EOF
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: xiaozhiqi-read-all-pods
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-reader
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: xiaozhiqi
EOF

kubectl apply -f xiaozhiqi-clusterrolebinding.yaml
kubectl get clusterrolebinding
kubectl describe clusterrolebinding xiaozhiqi-read-all-pods 
```

### 4. ClusterRole、ClusterRolebinding验证
```shell
# 切换到 xiaozhiqi@kubernetes 这个上下文
kubectl config use-context xiaozhiqi

# 可以列出当前 namespace 的 Pods
kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
nginx-sa1                    1/1       Running   0          14h
nginxtest-6c544f5669-n77hg   1/1       Running   0          14h

# 也可以列出当前其他 namespace kube-system 的 Pods
kubectl get pods -n kube-system
NAME                                   READY     STATUS    RESTARTS   AGE
heapster-5f78b96d49-q5qhj              1/1       Running   1          4d
kube-dns-595cc5f6c9-qcm9x              3/3       Running   3          4d
kubernetes-dashboard-df9db564b-bfgbj   1/1       Running   1          4d
monitoring-grafana-775964cccd-76vpl    1/1       Running   1          4d
monitoring-influxdb-57bdd7c6cc-rpfq7   1/1       Running   1          4d
```

# 五、例子：Token认证

> token 认证，serviceaccount 创建 -> RBAC授权
token 认证，表现在 pod 客户端 serviceaccount，每个Pod都需要和 API 交互，只不过权限有大有小
每个 POD 运行都会有默认的 serviceaccount 和 默认的 token，这个token是隐藏在serviceaccount中的，token是自动生成的也可以自己通过secret来生成，来和 API 交互
这个token 是不带权限的，只用于认证，所以有了认证不一定能操作 集群的，还需要授权权限的

## 查看默认Pod使用token
```shell
kubectl run nginxtest --image=nginx --port=80

kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
nginxtest-6c544f5669-n77hg   1/1       Running   0          35s

# 这个默认的 token 是通过 secret 来创建的，这个 token 就是和 API 交互的
kubectl describe pods nginxtest-6c544f5669-n77hg | grep 'token'
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-xqkqs (ro)
  default-token-xqkqs:
    SecretName:  default-token-xqkqs

# 查看 secret
kubectl get secret
NAME                  TYPE                                  DATA      AGE
default-token-xqkqs   kubernetes.io/service-account-token   3         3d
```

## 1.自定义 serviceaccount 并自动创建 token
>自定义 serviceaccount 并自动创建 token 并挂载到pod的容器 /var/run/secrets/kubernetes.io/serviceaccount/

```shell
# 创建 mysa serviceaccount
kubectl create serviceaccount mysa

# 查看 mysa serviceaccount,k8s自动为其分配token
kubectl describe serviceaccount mysa
Name:                mysa
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   mysa-token-dd885
Tokens:              mysa-token-dd885
Events:              <none>

# 创建nginx yaml使用自定serviceaccount mysa
cat > nginx.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx-sa
  namespace: default
spec:
  containers:
    - name: nginx-sa
      image: rnginx
  serviceAccountName: mysa
EOF

kubectl apply -f nginx.yaml

kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
nginx-sa                     1/1       Running   0          13s
nginxtest-6c544f5669-n77hg   1/1       Running   0          24m

kubectl describe pods nginx-sa | grep 'token'
      /var/run/secrets/kubernetes.io/serviceaccount from mysa-token-dd885 (ro)
  mysa-token-dd885:
    SecretName:  mysa-token-dd885

# 进入容器查看
kubectl exec -it nginx-sa /bin/bash      
root@nginx-sa:/# ls /var/run/secrets/kubernetes.io/serviceaccount/
ca.crt  namespace  token
```

## 2.自定义 serviceaccount 并手动创建 token
>自定义 serviceaccount 和手动创建 token 并挂载到pod的容器 /var/run/secrets/kubernetes.io/serviceaccount/

```shell
# 创建一个 mysa1 的serviceAccount
cat > mysa1.yaml << EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mysa1
EOF
 
kubectl apply -f mysa1.yaml 
 
# 查看 serviceaccount 对象的完整输出信息
kubectl get serviceaccounts/mysa1 -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"ServiceAccount","metadata":{"annotations":{},"name":"mysa1","namespace":"default"}}
  creationTimestamp: 2018-09-12T23:19:30Z
  name: mysa1
  namespace: default
  resourceVersion: "31599"
  selfLink: /api/v1/namespaces/default/serviceaccounts/mysa1
  uid: 4d09f9a7-b6e2-11e8-bdc8-000c296e7b8a
secrets:
- name: mysa1-token-fw29r


#手动创建 API token mysa1-secret 绑定到 mysa1 serviceAccount 上
cat > mysa1-secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: mysa1-secret
  annotations:
    kubernetes.io/service-account.name: mysa1
type: kubernetes.io/service-account-token
EOF

kubectl apply -f mysa1-secret.yaml

# 查看 secrets
kubectl describe secrets/mysa1-secret


# 然后在将 serviceaccount 应用到 Pod 的上即可
cat > nginx1.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx-sa1
  namespace: default
spec:
  containers:
    - name: nginx-sa1
      image: registry.cn-hangzhou.aliyuncs.com/xzqk8s/nginx:1.7.9
  serviceAccountName: mysa1
EOF

kubectl apply -f nginx1.yaml

kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
nginx-sa1                    1/1       Running   0          14s
nginxtest-6c544f5669-n77hg   1/1       Running   0          43m

# 查看token是否是我们自己所定义的
kubectl describe pods nginx-sa1 | grep 'token'
      /var/run/secrets/kubernetes.io/serviceaccount from mysa1-token-fw29r (ro)
  mysa1-token-fw29r:
    SecretName:  mysa1-token-fw29r
```

