# 3.创建k8s集群 2022-08-06

## 前提条件：每台机器已经设置好网络。并且内网可`ping`通。此教程不涉及网络设置
## 使用的是：CentOS-7-x86_64-Minimal-2009.iso
## 安装的是版本是k8s-v1.23.9 , docker-ce-v20.10[docker是默认安装，教程中未指定版本]

***

## `在master服务器执行`

### 以下是`kubeadm`的初始化自动化脚本`kubeadm_init.sh`
#### 新建`kubeadm_init.sh`，然后修改为可执行
#### 注意：下面的`--apiserver-advertise-address=192.168.216.10`参数的IP地址改成自己对应的地址
```shell
# 新建文件
touch kubeadm_init.sh
# 修改为可执行脚本
chmod 777 kubeadm_init.sh
```
#### 复制以下脚本到`kubeadm_init.sh`
```shell
# 先下载所需镜像。此处如果不先下载镜像，下面的参数可以可以下载，不过下载过程看不到
docker pull registry.aliyuncs.com/google_containers/coredns:v1.8.6 &&
docker pull registry.aliyuncs.com/google_containers/etcd:3.5.1-0 &&
docker pull registry.aliyuncs.com/google_containers/kube-apiserver:v1.23.9 &&
docker pull registry.aliyuncs.com/google_containers/kube-proxy:v1.23.9 &&
docker pull registry.aliyuncs.com/google_containers/kube-scheduler:v1.23.9 &&
docker pull registry.aliyuncs.com/google_containers/pause:3.6 &&
docker pull registry.aliyuncs.com/google_containers/kube-controller-manager:v1.23.9

# 开始执行初始化
# 由于默认拉取镜像地址 k8s.gcr.io 国内无法访问，这里指定阿里云镜像仓库地址
# 注意：以下的--apiserver-advertise-address参数修改为你自己本地的ip地址
# --kubernetes-version参数需要修改为kubeadm需要的版本。本次安装的是v.123.9
# --image-repository参数指定k8s的源为阿里云
kubeadm init \
--apiserver-advertise-address=192.168.216.10 \
--image-repository registry.aliyuncs.com/google_containers \
--kubernetes-version=v1.23.9 \
--pod-network-cidr=10.244.0.0/16 \
--service-cidr=10.96.0.0/12

```

#### 复制后退出然后执行脚本。此处要下载镜像并运行，时间有点长，等待一下
```shell
# 执行初始化脚本
./kubeadm_init.sh
```

***

#### 如果中途下载失败，并且有执行了初始化。运行以下命令，用于重置`kubeadm`
```shell
# 重置kubeadm初始化
kubeadm reset -f
```
#### 重置后再次执行初始化
```shell
# 执行初始化脚本
./kubeadm_init.sh
```

***

#### 初始化后显示以下信息表示集群创建成功
```shell
----
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

kubeadm join 192.168.216.11:6443 --token 84lnde.auurvv324oghnfat \
	--discovery-token-ca-cert-hash sha256:ed487d1c133d55deda215d212d363c90718eaf6a517bbb691659ca004c04988b
----
```

#### 成功之后按照提示执行上面的命令
```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### 并且记下加入节点加入集群的`token`
```shell
# 命令以及token
kubeadm join 192.168.216.11:6443 --token 84lnde.auurvv324oghnfat \
	--discovery-token-ca-cert-hash sha256:ed487d1c133d55deda215d212d363c90718eaf6a517bbb691659ca004c04988b
```

#### 如果没记下或者没有显示`token`，用以下命令生成
```shell
kubeadm token create --print-join-command
```

***

### 部署calico网络
#### 下载配置文件并修改

```shell
cd /home

wget https://docs.projectcalico.org/manifests/calico.yaml --no-check-certificate

# 在4434行后面添加以下代码
# value 的值跟--pod-network-cidr=10.244.0.0/16对应
- name: CALICO_IPV4POOL_CIDR
  value: "10.244.0.0/16"
```
#### 新建`apply_calico.sh`，然后修改为可执行
```shell
# 新建文件
touch apply_calico.sh
# 修改为可执行脚本
chmod 777 apply_calico.sh
```
#### 复制以下脚本到`apply_calico.sh`
```shell
# 运行calico.yaml
kubectl apply -f calico.yaml &&

# 去掉主节点上面不可调度pod污点。下面的k8s-master01替换成自己的hostname
kubectl describe node k8s-master01 &&
kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule &&

# 查看kube-system命名空间下的所有pod
kubectl get pod -n kube-system &&

# 查看所有节点情况
kubectl get nodes
```
#### 复制后退出然后执行脚本。此处要下载镜像并运行，时间有点长，等待一下
```shell
# 执行初始化脚本
./apply_calico.sh
```

#### 执行以上的脚本后，显示以下信息即表示`master`节点成功运行。后期其他节点就可以加进来了
#### 如果没显示为`Ready`状态先等一会，让`k8s`把所有服务全部启动来。大概等个`30`秒就可以
```shell
# 显示为Ready状态即表示已经启动且正常运行
[root@k8s-master01 home]# kubectl get nodes
NAME           STATUS   ROLES                  AGE    VERSION
k8s-master01   Ready    control-plane,master   5m6s   v1.23.9
```
## 至此。`k8s`集群安装成功并且已经启动。可以加入节点。`集群高可用`再另外说明

***

## 以下是`k8sv1.24`版本的变化

#### 由于后期版本的`k8s`不再支持`docker`作为运行时容器，而是用`containerd`，所以需要用到`ctr`容器命令。并且启动`k8s集群`用到的镜像后期会放到`ctr`，又因为`k8s.gcr.io`国内访问不了，所以要自行下载镜像并且`retag`到启动`k8s`时所需要的位置。也就是放在`ctr`下命名空间为`k8s.io`，默认是没有这个命名空间，下面一步步创建命名空间并且`pull`国内镜像并且`retag`到`k8s.io`命名空间下
#### 以下是自动化脚本
```shell
ctr ns create k8s.io &&

ctr -n k8s.io i pull registry.aliyuncs.com/google_containers/coredns:v1.8.6 &&
ctr -n k8s.io i pull registry.aliyuncs.com/google_containers/etcd:3.5.1-0 &&
ctr -n k8s.io i pull registry.aliyuncs.com/google_containers/pause:3.6 &&
ctr -n k8s.io i pull registry.aliyuncs.com/google_containers/kube-proxy:v1.23.9 &&
ctr -n k8s.io i pull registry.aliyuncs.com/google_containers/kube-scheduler:v1.23.9 &&
ctr -n k8s.io i pull registry.aliyuncs.com/google_containers/kube-apiserver:v1.23.9 &&
ctr -n k8s.io i pull registry.aliyuncs.com/google_containers/kube-controller-manager:v1.23.9 &&

ctr -n k8s.io i tag registry.aliyuncs.com/google_containers/coredns:v1.8.6 k8s.gcr.io/coredns/coredns:v1.8.6 &&
ctr -n k8s.io i tag registry.aliyuncs.com/google_containers/etcd:3.5.1-0 k8s.gcr.io/etcd:3.5.1-0 &&
ctr -n k8s.io i tag registry.aliyuncs.com/google_containers/kube-apiserver:v1.23.9 k8s.gcr.io/kube-apiserver:v1.23.9 &&
ctr -n k8s.io i tag registry.aliyuncs.com/google_containers/kube-proxy:v1.23.9 k8s.gcr.io/kube-proxy:v1.23.9 &&
ctr -n k8s.io i tag registry.aliyuncs.com/google_containers/kube-scheduler:v1.23.9 k8s.gcr.io/kube-scheduler:v1.23.9 &&
ctr -n k8s.io i tag registry.aliyuncs.com/google_containers/pause:3.6 k8s.gcr.io/pause:3.6  &&
ctr -n k8s.io i tag registry.aliyuncs.com/google_containers/kube-controller-manager:v1.23.9 k8s.gcr.io/kube-controller-manager:v1.23.9 &&

ctr -n k8s.io i rm registry.aliyuncs.com/google_containers/coredns:v1.8.6 &&
ctr -n k8s.io i rm registry.aliyuncs.com/google_containers/etcd:3.5.1-0 &&
ctr -n k8s.io i rm registry.aliyuncs.com/google_containers/kube-apiserver:v1.23.9 &&
ctr -n k8s.io i rm registry.aliyuncs.com/google_containers/kube-proxy:v1.23.9 &&
ctr -n k8s.io i rm registry.aliyuncs.com/google_containers/kube-scheduler:v1.23.9 &&
ctr -n k8s.io i rm registry.aliyuncs.com/google_containers/pause:3.6 &&
ctr -n k8s.io i rm registry.aliyuncs.com/google_containers/kube-controller-manager:v1.23.9
```
