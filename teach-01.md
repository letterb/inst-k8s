# 1.安装k8s前准备

## 前提条件：每台机器已经设置好网络。并且内网可`ping`通。此教程不涉及网络设置
## 使用的是：CentOS-7-x86_64-Minimal-2009.iso
## 安装的是版本是k8s-v1.23.9 , docker-ce-v20.10[docker是默认安装，教程中未指定版本]

***

## 更新yum源

### 自动化脚本`update_yum_repo.sh`

#### 创建脚本文件，并修改为可执行文件
```shell
# 创建脚本文件
touch update_yum_repo.sh
# 修改为可执行文件
chmod 777 update_yum_repo.sh
```

#### 复制以下脚本到`update_yum_repo.sh`
```shell
#安装wget vim
yum install wget vim -y &&

#进入/etc/yum.repos.d/目录下，新建一个repo_bak目录，用于保存系统中原来的repo文件
cd /etc/yum.repos.d/ &&
mkdir repo_bak &&
mv *.repo repo_bak/ &&

#到网易和阿里开源镜像站点下载系统对应版本的repo文件
wget http://mirrors.aliyun.com/repo/Centos-7.repo &&
wget http://mirrors.163.com/.help/CentOS7-Base-163.repo &&

#清除系统yum缓存并生成新的yum缓存
yum clean all &&
yum makecache	&&

#安装epel源
yum list | grep epel-release &&
yum install -y epel-release &&

#使用阿里开源镜像提供的epel,再次清除系统yum缓存，并重新生成新的yum缓存
wget -O /etc/yum.repos.d/epel-7.repo http://mirrors.aliyun.com/repo/epel-7.repo &&
yum clean all &&
yum makecache &&

#查看系统可用的yum源和所有的yum源
yum repolist enabled &&
yum repolist all
```

#### 复制后，执行脚本
```shell
./update_yum_repo.sh
```
****

## 升级内核到5.4lt版本.lt表示长期稳定支持版本

### 安装内核自动化脚本`update_kernel.sh`

#### 创建脚本文件，并且修改为可执行
```shell
# 创建脚本文件
touch update_kernel.sh
# 修改为可执行
chmod 777 update_kernel.sh
```

#### 复制以下脚本到`update_kernel.sh`
```shell
#更新yum源仓库
yum -y update &&

#载入ELRepo仓库的公共密钥
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org &&

#安装ELRepo仓库的yum源
yum install -y https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm &&

# 或升级
rpm -Uvh https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm &&

# 安装长期维护版本kernel-lt  如需更新最新稳定版选择kernel-ml
yum  --enablerepo=elrepo-kernel  install  -y  kernel-lt &&

#安装辅助工具（非必须，有些系统自带该工具）：grub2-pc
yum install -y grub2-pc
```

#### 查看可用内核版本及启动顺序命令
```shell
awk -F\' '$1=="menuentry " {print i++ " : " $2}' /boot/grub2/grub.cfg
```

#### 按照顺序设置内核启动顺序
```shell
#设置内核默认启动顺序
grub2-set-default 0
```

#### 编辑`/etc/default/grub`文件,设置`GRUB_DEFAULT=0`。这里按照上面显示的顺序来修改
```shell
vim /etc/default/grub

#设置   GRUB_DEFAULT=0   # 只需要修改这里即可

GRUB_TIMEOUT=1
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved  #---将这里的saved修改成0----
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=auto spectre_v2=retpoline rhgb quiet net.ifnames=0 console=tty0 console=ttyS0,115200n8 noibrs"
GRUB_DISABLE_RECOVERY="true"
```

#### 生成grub 配置文件
```shell
# 运行grub2-mkconfig命令来重新创建内核配置
grub2-mkconfig -o /boot/grub2/grub.cfg
```

#### 重启
```shell
reboot
```

#### 查看系统中已安装的内核
```shell
rpm -qa | grep kernel
```

### 升级内核工具包
```shell
# 删除旧版本工具包--可选
yum remove kernel-3.10.0-1160.el7.x86_64 -y &&
yum remove kernel-tools-libs-3.10.0-1160.71.1.el7.x86_64 -y &&
yum remove kernel-3.10.0-1160.71.1.el7.x86_64 -y &&
yum remove kernel-tools-3.10.0-1160.71.1.el7.x86_64 -y &&

# 安装新版本工具包
yum --disablerepo=\* --enablerepo=elrepo-kernel install -y kernel-lt-tools.x86_64 &&

# 查看已安装内核
rpm -qa | grep kernel
```

## 至此。安装k8s前的准备都已经完成

***
