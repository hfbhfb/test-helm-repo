


master: 
ssh root@120.46.134.34
node1: 
ssh root@121.36.57.253



kubeadm join 192.168.0.174:6443 --token 0rrosc.oyd0bt4fs81yt7w9 --discovery-token-ca-cert-hash sha256:2f68f8860335e5931c158e881b16b70c0e2290d8cdee1cdb0b95d99712bdc6d6

## 安装docker
``` bash
# 配置yum源库
wget -P /etc/yum.repos.d/ https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# 查看yum源支持的docker版本
yum list docker-ce --showduplicates | sort -r
# docker执行安装命令
yum install docker-ce-20.10.8-3.el7 -y
systemctl start docker
systemctl enable docker

## 配置docker
mkdir -p /etc/docker
cp -n /etc/docker/daemon.json /etc/docker/daemon.json.bak
cat >/etc/docker/daemon.json <<HFBEOF
{
"exec-opts":["native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://o4jtien3.mirror.aliyuncs.com"]
}
HFBEOF
systemctl daemon-reload
systemctl restart docker

```


## 配置内核参数
``` bash
cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables=1
echo "net.bridge.bridge-nf-call-ip6tables=1
echo "net.ipv4.ip_forward=1
sysctl -p /etc/sysctl.conf
EOF
# 运用内核网络参数
sysctl -a
# 检查内核网络参数
sysctl -a|grep ip_forward

```

## 安装kubeadm,kubectl,kubelet
``` bash
#国内镜像
cat > /etc/yum.repos.d/kubernetes.repo << HFBEOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
HFBEOF

# 安装指定版本 kubeadm,kubectl,kubelet

version=1.23.12-0
#sudo yum install -y kubelet.x86_64-$version kubeadm.x86_64-$version kubectl.x86_64-$version --disableexcludes=kubernetes
yum install -y kubelet-$version kubeadm-$version kubectl-$version --disableexcludes=kubernetes

systemctl status kubelet
systemctl enable kubelet


```


## 使用kubeadm 创建 单master集群  (master节点)
``` bash

# 创建(使用国内镜像)
masterip=192.168.0.174
kubeadm init \
--apiserver-advertise-address $masterip \
--image-repository registry.aliyuncs.com/google_containers \
--pod-network-cidr 10.0.0.0/16 \
--service-cidr 10.247.0.0/16 

# 导出工作节点join命令
masterip=192.168.0.174
tokenname=`kubeadm token list |grep 'token'|awk '{print $1}'`
token=`openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex|awk '{print $2}'`
echo $tokenname
echo "kubeadm join $masterip:6443 --token  $tokenname --discovery-token-ca-cert-hash sha256:$token"

# calico网络安装
echo "185.199.109.133 raw.githubusercontent.com" >> /etc/hosts
echo "#199.232.68.133 raw.githubusercontent.com" >> /etc/hosts
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.1/manifests/calico.yaml

```


## 安装metrics-server
``` bash

cat > metrics_components.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    k8s-app: metrics-server
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
    rbac.authorization.k8s.io/aggregate-to-view: "true"
  name: system:aggregated-metrics-reader
rules:
- apiGroups:
  - metrics.k8s.io
  resources:
  - pods
  - nodes
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    k8s-app: metrics-server
  name: system:metrics-server
rules:
- apiGroups:
  - ""
  resources:
  - nodes/metrics
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - pods
  - nodes
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server-auth-reader
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: extension-apiserver-authentication-reader
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server:system:auth-delegator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    k8s-app: metrics-server
  name: system:metrics-server
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:metrics-server
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
spec:
  ports:
  - name: https
    port: 443
    protocol: TCP
    targetPort: https
  selector:
    k8s-app: metrics-server
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
  strategy:
    rollingUpdate:
      maxUnavailable: 0
  template:
    metadata:
      labels:
        k8s-app: metrics-server
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls
        #image: k8s.gcr.io/metrics-server/metrics-server:v0.6.1
        image: bitnami/metrics-server:0.6.1
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /livez
            port: https
            scheme: HTTPS
          periodSeconds: 10
        name: metrics-server
        ports:
        - containerPort: 4443
          name: https
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /readyz
            port: https
            scheme: HTTPS
          initialDelaySeconds: 20
          periodSeconds: 10
        resources:
          requests:
            cpu: 100m
            memory: 200Mi
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
        volumeMounts:
        - mountPath: /tmp
          name: tmp-dir
      nodeSelector:
        kubernetes.io/os: linux
      priorityClassName: system-cluster-critical
      serviceAccountName: metrics-server
      volumes:
      - emptyDir: {}
        name: tmp-dir
---
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  labels:
    k8s-app: metrics-server
  name: v1beta1.metrics.k8s.io
spec:
  group: metrics.k8s.io
  groupPriorityMinimum: 100
  insecureSkipTLSVerify: true
  service:
    name: metrics-server
    namespace: kube-system
  version: v1beta1
  versionPriority: 100
EOF

# 安装
kubectl apply -f metrics_components.yaml

#kubectl delete -f metrics_components.yaml

```



## 节点使用kubeadm join加入集群  (工作节点)
``` bash
# 使用 master节点上看到的join命令
```


