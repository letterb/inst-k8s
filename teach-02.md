# 2.部署k8s

## 前提条件：每台机器已经设置好网络。并且内网可`ping`通。此教程不涉及网络设置
## 使用的是：CentOS-7-x86_64-Minimal-2009.iso
## 安装的是版本是k8s-v1.23.9 , docker-ce-v20.10[docker是默认安装，教程中未指定版本

***

## 每台机器都要执行

***

### 以下是分步骤执行。最下面有全部自动脚本

#### 安装以下的软件包，可以解决99%的依赖问题
```shell
yum -y install gcc gcc-c++ make autoconf libtool-ltdl-devel gd-devel freetype-devel libxml2-devel libjpeg-devel libpng-devel openssh-clients openssl-devel curl-devel bison patch libmcrypt-devel libmhash-devel ncurses-devel binutils compat-libstdc++-33 elfutils-libelf elfutils-libelf-devel glibc glibc-common glibc-devel libgcj libtiff pam-devel libicu libicu-devel gettext-devel libaio-devel libaio libgcc libstdc++ libstdc++-devel unixODBC unixODBC-devel numactl-devel glibc-headers sudo bzip2 mlocate flex lrzsz sysstat lsof setuptool system-config-network-tui system-config-firewall-tui ntsysv ntp pv lz4 dos2unix unix2dos rsync dstat iotop innotop mytop telnet iftop expect cmake nc gnuplot screen xorg-x11-utils xorg-x11-xinit rdate bc expat-devel compat-expat1 tcpdump sysstat man nmap curl lrzsz elinks finger bind-utils traceroute mtr ntpdate zip unzip vim wget net-tools
```

#### 关闭firewalld服务
```shell
systemctl stop firewalld && systemctl disable firewalld &&
```

#### 关闭iptables服务
```shell
systemctl stop iptables && systemctl disable iptables &&
```

#### 禁用selinux`永久和临时关闭`
```shell
sed -i 's/enforcing/disabled/' /etc/selinux/config &&
setenforce 0
```

#### 禁用swap分区
##### 临时关闭
```shell
swapoff -a
```
##### 永久关闭`编辑/etc/fstab`，注释掉最后一行
```shell
vim /etc/fstab
```

#### 由于开启内核 ipv4 转发需要加载 br_netfilter 模块，所以加载下该模块
```shell
modprobe br_netfilter &&
modprobe ip_conntrack
``````

#### 将上面的命令设置成开机启动，因为重启后模块失效，下面是开机自动加载模块的方式。首先新建 `/etc/rc.sysinit` 文件，内容如下所示
```sheel
cat >>/etc/rc.sysinit<<EOF
#!/bin/bash
for file in /etc/sysconfig/modules/*.modules ; do
[ -x $file ] && $file
done
EOF
```

#### 然后在`/etc/sysconfig/modules/`目录下新建如下文件
```shell
echo "modprobe br_netfilter" >/etc/sysconfig/modules/br_netfilter.modules &&

echo "modprobe ip_conntrack" >/etc/sysconfig/modules/ip_conntrack.modules &&

chmod 755 /etc/sysconfig/modules/br_netfilter.modules &&

chmod 755 /etc/sysconfig/modules/ip_conntrack.modules
```
##### 然后重启后，模块就可以自动加载了

#### 优化内核参数
```shell
cat > kubernetes.conf <<EOF
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
vm.swappiness=0 # 禁止使用 swap 空间，只有当系统 OOM 时才允许使用它
vm.overcommit_memory=1 # 不检查物理内存是否够用
vm.panic_on_oom=0 # 开启 OOM
fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=1048576
fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1
net.netfilter.nf_conntrack_max=2310720
EOF
```
##### 复制
```shell
cp kubernetes.conf  /etc/sysctl.d/kubernetes.conf
```
##### 执行
```shell
sysctl -p /etc/sysctl.d/kubernetes.conf
```

#### 所有节点安装ipvs
```shell
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack
EOF
```
##### 修改权限并执行和查看
```shell
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack
```

#### 所有节点安装ipset
```shell
yum install ipset -y
```
#### 为了方面ipvs管理，这里安装一下ipvsadm
```shell
yum install ipvsadm -y
```

#### 所有节点设置系统时区同步时间
```
timedatectl set-timezone Asia/Shanghai &&
timedatectl set-local-rtc 0 &&
systemctl restart rsyslog &&
systemctl restart crond
```

#### 安装`docker-ce`默认最新版本为20.10。如需其他版本自行指定版本

##### 获取镜像源
```shell
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
```

##### 安装docker-de
```shell
yum install docker-ce -y
```

##### 设置驱动`Docker在默认情况下使用的Cgroup Driver为cgroupfs，而kubernetes推荐使用systemd来代替cgroupfs`
```shell
mkdir /etc/docker &&
cat <<EOF > /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": [
    "http://hub-mirror.c.163.com",
    "https://registry.docker-cn.com",
    "https://docker.mirrors.ustc.edu.cn"
  ],
  "log-opts": {
    "max-size": "100m"
  }
}
EOF
```
##### 设置开机启动并且重启docker
```shell
systemctl enable docker.service && systemctl start docker
```
##### 重载`daemon.json`
```shell
systemctl daemon-reload
```

#### 安装k8s`v1.23.9版本`

##### 由于 kubernetes 的镜像源在国外，速度比较慢，因此我们需要切换成国内的镜像源
```shell
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

##### 指定版本安装
```shell
yum install kubeadm-1.23.9 kubectl-1.23.9 kubelet-1.23.9 -y
```

##### 编辑`/etc/sysconfig/kubelet`指定kubelet的驱动。对应docker的驱动`systemd`
```shell
vim /etc/sysconfig/kubelet
```
##### 修改成以下配置
```shell
KUBELET_CGROUP_ARGS="--cgroup-driver=systemd"
KUBE_PROXY_MODE="ipvs"
```

##### 设置`kubelet`为开机启动并启动
```shell
systemctl enable kubelet && systemctl start kubelet
```

## 至此。`k8s`安装成功

***

### 以下是自动化脚本

#### 由于`shell`不是很熟，下面的编辑操作手动执行
```shell
# 永久关闭编辑/etc/fstab，注释掉最后一行
vim /etc/fstab
```

#### 接下来是所有的自动化脚本
```shell
# 安装以下的软件包，可以解决99%的依赖问题
yum -y install gcc gcc-c++ make autoconf libtool-ltdl-devel gd-devel freetype-devel libxml2-devel libjpeg-devel libpng-devel openssh-clients openssl-devel curl-devel bison patch libmcrypt-devel libmhash-devel ncurses-devel binutils compat-libstdc++-33 elfutils-libelf elfutils-libelf-devel glibc glibc-common glibc-devel libgcj libtiff pam-devel libicu libicu-devel gettext-devel libaio-devel libaio libgcc libstdc++ libstdc++-devel unixODBC unixODBC-devel numactl-devel glibc-headers sudo bzip2 mlocate flex lrzsz sysstat lsof setuptool system-config-network-tui system-config-firewall-tui ntsysv ntp pv lz4 dos2unix unix2dos rsync dstat iotop innotop mytop telnet iftop expect cmake nc gnuplot screen xorg-x11-utils xorg-x11-xinit rdate bc expat-devel compat-expat1 tcpdump sysstat man nmap curl lrzsz elinks finger bind-utils traceroute mtr ntpdate zip unzip vim wget net-tools &&

# 关闭firewalld服务
systemctl stop firewalld && systemctl disable firewalld &&

# 关闭iptables服务
systemctl stop iptables && systemctl disable iptables

# 禁用selinux.永久和临时关闭
sed -i 's/enforcing/disabled/' /etc/selinux/config &&
setenforce 0 &&

# 禁用swap分区 - 临时关闭
swapoff -a &&

# 由于开启内核 ipv4 转发需要加载 br_netfilter 模块，所以加载下该模块
modprobe br_netfilter &&
modprobe ip_conntrack &&

# 将上面的命令设置成开机启动，因为重启后模块失效，下面是开机自动加载模块的方式。首先新建 /etc/rc.sysinit 文件，内容如下所示

cat >>/etc/rc.sysinit<<EOF
#!/bin/bash
for file in /etc/sysconfig/modules/*.modules ; do
[ -x $file ] && $file
done
EOF

# 然后在/etc/sysconfig/modules/目录下新建如下文件.然后重启后，模块就可以自动加载了
echo "modprobe br_netfilter" >/etc/sysconfig/modules/br_netfilter.modules &&
echo "modprobe ip_conntrack" >/etc/sysconfig/modules/ip_conntrack.modules && 
chmod 755 /etc/sysconfig/modules/br_netfilter.modules &&
chmod 755 /etc/sysconfig/modules/ip_conntrack.modules &&

# 优化内核参数
cat > kubernetes.conf <<EOF
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
vm.swappiness=0 # 禁止使用 swap 空间，只有当系统 OOM 时才允许使用它
vm.overcommit_memory=1 # 不检查物理内存是否够用
vm.panic_on_oom=0 # 开启 OOM
fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=1048576
fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1
net.netfilter.nf_conntrack_max=2310720
EOF
# 复制
cp kubernetes.conf  /etc/sysctl.d/kubernetes.conf &&
# 执行
sysctl -p /etc/sysctl.d/kubernetes.conf &&

# 所有节点安装ipvs
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack
EOF
## 修改权限并执行和查看
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack &&

# 所有节点安装ipset
yum install ipset -y &&

# 为了方面ipvs管理，这里安装一下ipvsadm
yum install ipvsadm -y &&

# 所有节点设置系统时区同步时间
timedatectl set-timezone Asia/Shanghai &&
timedatectl set-local-rtc 0 &&
systemctl restart rsyslog &&
systemctl restart crond &&

# 安装docker-ce
## 获取镜像源
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo &&
## 执行安装
yum install docker-ce -y &&
## 设置驱动Docker在默认情况下使用的Cgroup Driver为cgroupfs，而kubernetes推荐使用systemd来代替cgroupfs
mkdir /etc/docker &&
cat <<EOF > /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": [
    "http://hub-mirror.c.163.com",
    "https://registry.docker-cn.com",
    "https://docker.mirrors.ustc.edu.cn"
  ],
  "log-opts": {
    "max-size": "100m"
  }
}
EOF
## 设置开机启动并且重启docker
systemctl enable docker.service && systemctl start docker &&

# 安装k8sv1.23.9版本
## 由于 kubernetes 的镜像源在国外，速度比较慢，因此我们需要切换成国内的镜像源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
## 指定版本安装
yum install kubeadm-1.23.9 kubectl-1.23.9 kubelet-1.23.9 -y &&

## 开机启动kubelet和现在启动kubelet
systemctl enable kubelet && systemctl start kubelet

```

## 以上就是自动化脚本。全部安装后，手动修改k8s的配置

### 编辑`/etc/sysconfig/kubelet`指定`kubelet`的驱动。对应`docker`的驱动`systemd`
```shell
vim /etc/sysconfig/kubelet
```

### 修改成以下配置
```shell
KUBELET_CGROUP_ARGS="--cgroup-driver=systemd"
KUBE_PROXY_MODE="ipvs"
```

### 修改后保存退出

## 至此。k8s全部部署安装完成

***




