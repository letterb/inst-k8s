# 4.添加`Worker`节点

## 前提条件：每台机器已经设置好网络。并且内网可`ping`通。此教程不涉及网络设置
## 使用的是：CentOS-7-x86_64-Minimal-2009.iso
## 安装的是版本是k8s-v1.23.9 , docker-ce-v20.10[docker是默认安装，教程中未指定版本]

***

### 添加`Worker`节点比较简单，直接在集群部署好之后的`join`代码复制过来执行即可，命令如下
```shell
# join 后面的IP地址为教程第三步中，成功运行master集群后显示的信息
kubeadm join 192.168.216.10:6443 --token 3ytn64.ydofua5scugonhi5 \
    --discovery-token-ca-cert-hash sha256:c5069a3fe75d89a5c09d7f7db3484dc664b3b8d8e64fce62bc83af0f4ddcb79a
```

### 添加之后到`master`节点查看`node`状态。一开始会显示`NotReady`状态，需要等个`30`秒左右
```shell
# 查看命令如下
kubectl get nodes

# 显示以下信息表示Worker节点运行成功
[root@k8s-master01 john]# kubectl get nodes
NAME           STATUS     ROLES                  AGE   VERSION
k8s-master01   Ready      control-plane,master   15m   v1.23.9
k8s-node01     Ready   <none>                 32s   v1.23.9
```

### 所有正常工作的`Node`以及`Pod`显示如下信息
```shell
# 查看命令
kubectl get pod --all-namespaces -o wide

# 显示如下信息，所有Status为Ready表示k8s集群正常运行
[root@k8s-master01 john]# kubectl get pod --all-namespaces -o wide
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE    IP                NODE           NOMINATED NODE   READINESS GATES
kube-system   calico-kube-controllers-5c64b68895-87w7d   1/1     Running   0          14m    10.244.85.193     k8s-node01     <none>           <none>
kube-system   calico-node-fkph9                          1/1     Running   0          14m    192.168.216.10    k8s-master01   <none>           <none>
kube-system   calico-node-hwcqf                          1/1     Running   0          3m3s   192.168.216.101   k8s-node01     <none>           <none>
kube-system   coredns-6d8c4cb4d-9t6qz                    1/1     Running   0          17m    10.244.32.128     k8s-master01   <none>           <none>
kube-system   coredns-6d8c4cb4d-g7lvw                    1/1     Running   0          17m    10.244.32.130     k8s-master01   <none>           <none>
kube-system   etcd-k8s-master01                          1/1     Running   0          17m    192.168.216.10    k8s-master01   <none>           <none>
kube-system   kube-apiserver-k8s-master01                1/1     Running   0          17m    192.168.216.10    k8s-master01   <none>           <none>
kube-system   kube-controller-manager-k8s-master01       1/1     Running   0          17m    192.168.216.10    k8s-master01   <none>           <none>
kube-system   kube-proxy-5hprx                           1/1     Running   0          17m    192.168.216.10    k8s-master01   <none>           <none>
kube-system   kube-proxy-dd8gd                           1/1     Running   0          3m3s   192.168.216.101   k8s-node01     <none>           <none>
kube-system   kube-scheduler-k8s-master01                1/1     Running   0          17m    192.168.216.10    k8s-master01   <none>           <none>
```
