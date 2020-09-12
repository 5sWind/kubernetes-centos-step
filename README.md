# LOG

## 关闭防火墙
```bash
systemctl stop firewalld.service
systemctl disable firewalld.service
systemctl stop iptables
```

## 永久关闭SELinux
```bash
setenforce 0
```

## 设置内核参数及加载ipvs模块
```bash
vim /etc/sysctl.conf
>
net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.netfilter.nf_conntrack_max=1048576
```

## 启用ipvsadm
```bash
modprobe br_netfilter
modprobe ip_vs
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
modprobe nf_conntrack_ipv4
```

vim /etc/security/limits.conf
>
```
* soft nofile 65536
* hard nofile 65536
* soft nproc 65536
* hard nproc 65536
* soft memlock unlimited
* hard memlock unlimited
```

## 关闭swap
```bash
sudo swapoff -a
cat /etc/fstab | grep -v '^#' | grep -v 'swap' | sudo tee /etc/fstab
```

## Docker 
```bash
sudo yum install -y yum-utils
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install docker-ce docker-ce-cli containerd.io
sudo systemctl start docker
```

vim /etc/docker/daemon.json
>
```
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": [
    "https://hub-mirror.c.163.com",
    "https://mirror.baidubce.com"
  ]
}
```

```bash
mkdir -p /etc/systemd/system/docker.service.d
systemctl daemon-reload
systemctl restart docker
systemctl enable docker
```

## 加载ipvs
```bash
lsmod | grep ^ip_vs | awk '{print $1}' | xargs -I {} modprobe {}
```

## 安装kubeadm,kubelet,kubectl

vim /etc/yum.repos.d/kubernetes.repo

>
```
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
```

```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
yum -y install kubeadm kubelet kubectl ipvsadm
systemctl enable kubelet
```

## 获取镜像列表
```bash
kubeadm config images list
```

```bash
images=(
    kube-apiserver:v1.19.1
    kube-controller-manager:v1.19.1
    kube-scheduler:v1.19.1
    kube-proxy:v1.19.1
    pause:3.2
    etcd:3.4.13-0
    coredns:1.7.0
)

for imageName in ${images[@]} ; do
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
    docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
done
```

## 搭建集群
```bash
kubeadm init --pod-network-cidr=10.244.0.0/16
export KUBECONFIG=/etc/kubernetes/admin.conf
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Flannel 组网
```bash
sysctl net.bridge.bridge-nf-call-iptables=1
kubectl apply -f https://cdn.jsdelivr.net/gh/coreos/flannel@v0.12.0/Documentation/kube-flannel.yml
```

## Worker JOIN
```bash
kubeadm join 192.168.0.108:6443 --token ngo18n.7vekj9609d8o0x48 \
    --discovery-token-ca-cert-hash sha256:1383767cb71314342c10f6476e7cf35e332afc7a380af53131f6da3f4a0e202b 
```

## 创建ingress反向代理及内部负载均衡器插件
```bash
kubectl apply -f https://cdn.jsdelivr.net/gh/5sWind/kubernetes-centos-step@master/deploy.yaml
```

## 搭建图形控制台
```bash
kubectl apply -f https://cdn.jsdelivr.net/gh/kubernetes/dashboard@v2.0.4/aio/deploy/recommended.yaml
```

### 本地访问
```bash
kubectl proxy --address='192.168.0.108' --port=8001 --accept-hosts='^$' --disable-filter=true
```

http://192.168.0.108:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login

### 为控制台免除权限认证(不推荐)
vim dashboard-admin.yaml

>
```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
```

```bash
kubectl apply -f dashboard-admin.yaml 
```
