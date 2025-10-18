---
title: 在Ubuntu上安装k8s集群
categories: 服务器
date: 2022-02-10 22:38:00
---

## 虚拟机准备

使用multipass（[multipass使用文档](/2022/02/10/server/multipass/)）准备三台Ubuntu虚拟机：

先生成一个ssh key,执行命令`

```
ssh-keygen -t rsa -C "test@163.com"
```

准备一个配置文件`init.yaml`，追加内容

```
#cloud-config
ssh_authorized_keys:
  - ssh-rsa AAAAB3.......hh32R test@163.com(生成的公钥)
runcmd:
  - sudo apt-get install python -y
```


创建三个实例：

```
multipass launch --name ubuntu1 --cpus 2 --mem 2G --disk 5G --cloud-init init.config 18.04
multipass launch --name ubuntu2 --cpus 2 --mem 2G --disk 5G --cloud-init init.config 18.04
multipass launch --name ubuntu3 --cpus 2 --mem 2G --disk 5G --cloud-init init.config 18.04
```

其中`--name`为要创建的实例的名字，`18.04`为镜像，第一次执行时会自动从官方下载此镜像。

待命令执行完成后，可使用以下命令查看：

```
PS C:\Users\xxxxx> multipass list
Name                    State             IPv4             Image
ubuntu1                 Running           172.19.9.102     Ubuntu 18.04 LTS
ubuntu2                 Running           172.19.8.72      Ubuntu 18.04 LTS
ubuntu3                 Running           172.19.2.58      Ubuntu 18.04 LTS
```

如果是使用Hyper虚拟化安装的multipass，可以在hyper管理器中查看实例状态

![](https://static.jiangliuhong.top/images/2022/1/1643270468080.png)


## 安装docker

下载源

```
curl -fsSL https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
```

配置源

```
sudo add-apt-repository "deb https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
```

安装docker

```
sudo apt update && sudo apt install -y docker-ce
```

使用` docker --version`命令检查docker是否安装成功。

配置docker镜像加速器(使用DaoCloud提供的，参考地址：[https://www.daocloud.io/mirror#accelerator-doc](https://www.daocloud.io/mirror#accelerator-doc))以及配置cgroup驱动：

```
cat <<EOF | sudo tee  /etc/docker/daemon.json
{"registry-mirrors": ["http://f1361db2.m.daocloud.io"],"exec-opts": ["native.cgroupdriver=systemd"]}
EOF
```

配置完成后重启docker:

```
systemctl restart docker
```

## 安装kubeadm

> 参考地址：[https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

iptable配置：根据官方的描述，需要确保`br_netfilter`模块被加载。这一操作可以通过运行`lsmod | grep br_netfilter`来完成。若要显式加载该模块，可执行`sudo modprobe br_netfilter`。为了让你的Linux节点上的`iptables`能够正确地查看桥接流量，你需要确保在你的 sysctl 配置中将`net.bridge.bridge-nf-call-iptables`设置为1。

```

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sudo sysctl --system
```

安装依赖的包；
```
sudo apt-get install -y apt-transport-https ca-certificates curl
```

下载谷歌公开的签名密钥,并添加到kubernetes apt仓库(这里使用的是清华的镜像源)：

```
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://static.jiangliuhong.top/tools/apt-key.gpg

echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://mirrors.tuna.tsinghua.edu.cn/kubernetes/apt kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

安装kubeadm、kubelet 和 kubectl：


```
sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl && sudo apt-mark hold kubelet kubeadm kubectl
```

同时也可指定要安装的版本，不指定，默认安装最新版(此处我使用的版本是v1.23.3)

```
apt-get install -y kubelet=1.18.0-00 kubeadm=1.18.0-00 kubectl=1.18.0-00
```

由于kubeadm需要root用户，所以给实例设置root用户密码，并切换到root用户：

```
sudo passwd root
su root
```

由于国内无法访问`k8s.cgr.io`，所以手动下载镜像，先使用`kubeadm config images list`命令查看需要下载的镜像版本，对应着修改为阿里云的镜像地址：

```
docker pull registry.aliyuncs.com/google_containers/kube-apiserver:v1.23.3 && \
docker pull registry.aliyuncs.com/google_containers/kube-controller-manager:v1.23.3 && \
docker pull registry.aliyuncs.com/google_containers/kube-scheduler:v1.23.3 && \
docker pull registry.aliyuncs.com/google_containers/kube-proxy:v1.23.3 && \
docker pull registry.aliyuncs.com/google_containers/pause:3.6 && \
docker pull registry.aliyuncs.com/google_containers/etcd:3.5.1-0 && \
docker pull registry.aliyuncs.com/google_containers/coredns:v1.8.6
```

对镜像进行重命令：

```
docker tag registry.aliyuncs.com/google_containers/kube-apiserver:v1.23.3 k8s.gcr.io/kube-apiserver:v1.23.3 && \
docker tag registry.aliyuncs.com/google_containers/kube-controller-manager:v1.23.3 k8s.gcr.io/kube-controller-manager:v1.23.3 && \
docker tag registry.aliyuncs.com/google_containers/kube-scheduler:v1.23.3 k8s.gcr.io/kube-scheduler:v1.23.3 && \
docker tag registry.aliyuncs.com/google_containers/kube-proxy:v1.23.3 k8s.gcr.io/kube-proxy:v1.23.3 && \
docker tag registry.aliyuncs.com/google_containers/pause:3.6 k8s.gcr.io/pause:3.6 && \
docker tag registry.aliyuncs.com/google_containers/etcd:3.5.1-0 k8s.gcr.io/etcd:3.5.1-0 && \
docker tag registry.aliyuncs.com/google_containers/coredns:v1.8.6 k8s.gcr.io/coredns/coredns:v1.8.6
```

为节省空间，删除阿里云的镜像:

```
docker rmi registry.aliyuncs.com/google_containers/kube-apiserver:v1.23.3 && \
docker rmi registry.aliyuncs.com/google_containers/kube-controller-manager:v1.23.3 && \
docker rmi registry.aliyuncs.com/google_containers/kube-scheduler:v1.23.3 && \
docker rmi registry.aliyuncs.com/google_containers/kube-proxy:v1.23.3 && \
docker rmi registry.aliyuncs.com/google_containers/pause:3.6 && \
docker rmi registry.aliyuncs.com/google_containers/etcd:3.5.1-0 && \
docker rmi registry.aliyuncs.com/google_containers/coredns:v1.8.6
```

## 初始化master节点

```
kubeadm init --service-cidr=10.96.0.0/12 --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=Swap
```

当看到以下日志时，说明master节点初始化成功：

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.23.216.152:6443 --token rjyzv5.7g1i46k0gqo4dj4b \
        --discovery-token-ca-cert-hash sha256:14feb2234fee5a396d82e6f32266b4f0b5a41c547ba83fbff8368a40ae547025
```

配置kubectl：

```
mkdir -p $HOME/.kube && \
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config && \
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

执行` kubectl get nodes`命令得到如下结果，代表配置成功：
```
NAME      STATUS     ROLES                  AGE    VERSION
ubuntu1   NotReady   control-plane,master   7m5s   v1.23.3
```


使用flannel配置集群网络:

```
wget https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml -O kube-flannel.yml && 
kubectl apply -f kube-flannel.yml
```

看见如下信息，则代表配置成功：

```
podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds created
```

## 初始化工作节点

在工作节点执行下面的命令（该命令来自初始化master节点后的日志）：

```
kubeadm join 172.23.216.152:6443 --token rjyzv5.7g1i46k0gqo4dj4b \
        --discovery-token-ca-cert-hash sha256:14feb2234fee5a396d82e6f32266b4f0b5a41c547ba83fbff8368a40ae547025
```

执行完成后，查询节点信息：

```
# kubectl get nodes
kNAME      STATUS   ROLES                  AGE   VERSION
ubuntu1   Ready    control-plane,master   15h   v1.23.3
ubuntu2   Ready    <none>                 15h   v1.23.3
ubuntu3   Ready    <none>                 15h   v1.23.3
```

如果发现两个节点的ROLES为none，可以执行下面的命令修改一下：

```
 kubectl label node ubuntu2 node-role.kubernetes.io/worker=worker
 kubectl label node ubuntu3 node-role.kubernetes.io/worker=worker
```

## 测试集群可用性

> 安装一个nginx以测试集群可用性

准备nginx-deployment文件，`ngxin-deployment.yml`：

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

准备一个ngxin-service文件`ngxin-svc.yml`:

```yml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  type: NodePort
  ports:
  - port: 80
    nodePort: 32633
```

分别执行两个部署文件：

```
kubectl apply -f nginx-deploy.yml
kubectl apply -f nginx-svc.yml
```

这里部署的svc的端口是32633，部署完成后，在宿主机访问端口，能访问成功，则代表部署完成

![](https://static.jiangliuhong.top/images/2022/2/1644286977700.png)

## 部署dashboard

执行命令,这里我根据官方提供的yml，进行了修改，主要对dashboard的service增加了nodePort配置，端口为32634，官方地址为：[https://raw.githubusercontent.com/kubernetes/dashboard/v2.2.0/aio/deploy/recommended.yaml](https://raw.githubusercontent.com/kubernetes/dashboard/v2.2.0/aio/deploy/recommended.yaml))：

```
kubectl apply -f kubectl apply -f https://static.jiangliuhong.top/tools/k8s-dashboard.yml
```

 创建一个用户：
 
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

获取token:

```
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

执行完成后，访问`https://172.23.216.152:32634/`，使用上面得到的token进入dashboard即可

![](https://static.jiangliuhong.top/images/2022/2/1644289063490.png)

通过dashboard可轻松查看到前面通过命令创建的nginx应用

![](https://static.jiangliuhong.top/images/2022/2/1644290606327.png)