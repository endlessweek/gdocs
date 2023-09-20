官方文档：https://sealos.io/zh-Hans/docs/lifecycle-management/quick-start/installation
# 单master，node安装
## 1.  环境及基础设施准备
| 主机名 | ip地址 | 用途 | 其他说明 |
| ----- | ----- | ----- | -----|
| deploy |192.168.192.4|拉取并推送镜像包|内网双网卡|
|master1|192.168.192.5||centos7.8|
|node1|192.168.192.6||centos7.8|
### 1.1 部署节点
#### 1.1.1 环境准备
``` shell
# 准备一台可以连接外网的节点用来拉取镜像包，这台机器需要两块网卡，联通内网集群和外网环境
# deploy主机关闭firewalld，selinux，更新yum源，下载必要的包等
# 配置yum源，用清华大学yum源
sudo sed -e 's|^mirrorlist=|#mirrorlist=|g' \
         -e 's|^#baseurl=http://mirror.centos.org/centos|baseurl=https://mirrors.tuna.tsinghua.edu.cn/centos|g' \
         -i.bak \
         /etc/yum.repos.d/CentOS-*.repo
# 安装必要工具
yum update -y && yum -y install  wget psmisc vim net-tools nfs-utils telnet yum-utils device-mapper-persistent-data lvm2 git tar curl jq

# 下载必要工具
yum -y install createrepo yum-utils wget epel*

# 下载全量依赖包
repotrack createrepo wget psmisc vim net-tools nfs-utils telnet yum-utils device-mapper-persistent-data lvm2 git tar curl gcc keepalived haproxy bash-completion chrony sshpass ipvsadm ipset sysstat conntrack libseccomp

# 删除libseccomp
rm -rf libseccomp-*.rpm

# 下载libseccomp
wget http://rpmfind.net/linux/centos/8-stream/BaseOS/x86_64/os/Packages/libseccomp-2.5.1-1.el8.x86_64.rpm

# 创建yum源信息
createrepo -u -d /data/centos7/

# 拷贝包到内网机器上
scp -r /data/centos7/ root@192.168.192.5:
scp -r /data/centos7/ root@192.168.192.6:

```
#### 1.1.4 下载操作系统内核升级包
``` shell
# 稳定版kernel-ml   如需更新长期维护版本kernel-lt  
直接去以下网址下载内核
http://mirrors.coreix.net/elrepo-archive-archive/kernel/el7/x86_64/RPMS/
#我这里下载的是 kernel-lt-5.4.256-1.el7.elrepo.x86_64.rpm
scp kernel-lt-5.4.256-1.el7.elrepo.x86_64.rpm master1:/data node1:/data
``` 
#### 1.1.3 sealos基础设施准备
``` shell
# 查看sealos可用版本
curl --silent "https://api.github.com/repos/labring/sealos/releases" | jq -r '.[].tag_name'

# 下载二进制4.3.0版本sealos
$ wget https://ghproxy.com/https://github.com/labring/sealos/releases/download/v4.3.0/sealos_4.3.0_linux_amd64.tar.gz \
   && ssh master1 'mkdir /data' \
   && scp sealos_4.3.0_linux_amd64.tar.gz master1:/data/ 
``` 
#### 1.1.4 k8s镜像拉取及打包
##### 支持 containerd 的 Kubernetes（k8s 版本 >=1.18.0）
| k8s 版本 | sealos 版本 | cri 版本 | 镜像版本 |
| --- | --- | --- | --- |
|<1.25|	>=v4.0.0|v1alpha2|labring/kubernetes:v1.24.0|
|>=1.25|>=v4.1.0|v1alpha2|labring/kubernetes:v1.25.0|
|>=1.26|>=v4.1.4-rc3 |v1|labring/kubernetes:v1.26.0|
|>=1.27|>=v4.2.0-alpha3	|v1|labring/kubernetes:v1.27.0|
##### 支持 docker 的 Kubernetes（k8s 版本 >=1.18.0）
|k8s 版本|sealos 版本|cri 版本|镜像版本|
|---|---|---|---|
|<1.25|>=v4.0.0|v1alpha2|labring/kubernetes-docker:v1.24.0|
|>=1.25|>=v4.1.0|v1alpha2|labring/kubernetes-docker:v1.25.0|
|>=1.26|>=v4.1.4-rc3|v1|labring/kubernetes-docker:v1.26.0|
|>=1.27|>=v4.2.0-alpha3|v1|labring/kubernetes-docker:v1.27.0|
``` shell
# 拉取所需镜像
sealos pull labring/kubernetes-docker:v1.27-latest
sealos pull labring/calico:3.25.1
sealos pull labring/helm:v3.12.0
# 将下载镜像打包
sealos save -o kubernetes.tar labring/kubernetes-docker:v1.27-latest
sealos save -o calico.tar labring/calico:3.25.1
sealos save -o helm.tar labring/helm:v3.12.0
# 将镜像包传到内网master节点
scp calico.tar helm.tar kubernetes.tar master1:/data/

``` 
### 1.2 master节点和node节点
#### 1.2.1 环境准备
``` shell
# 关闭swap
sed -ri 's/.*swap.*/#&/' /etc/fstab
swapoff -a && sysctl -w vm.swappiness=0
# 关闭firewall
systemctl disable --now firewalld
# 关闭selinux
setenforce 0
sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/selinux/config
# 修改主机名和ip地址，虚拟机克隆的主机注意修改网卡UUID
hostnamectl --static set-hostname master1 && bash
# 查看当前的网卡列表和 UUID：
nmcli con show
# 删除要更改 UUID 的网络连接：
nmcli con delete uuid <原 UUID> 
或 nmcli con delete <name>
# 重新生成 UUID：
nmcli con add type ethernet ifname <接口名称ens33> con-name <新名称ens33>
# 配置ip
nmcli con mod ens33 ipv4.addr 192.168.192.6/24 ipv4.method mannual connection.autoconnect yes
# 重新启用网络连接：
nmcli con up <新名称ens33>
# 关闭NetworkManger
systemctl disable NetworkManager --now

# 在内网机器上创建repo配置文件
rm -rf /etc/yum.repos.d/*
cat > /etc/yum.repos.d/local.repo  << EOF 
[local]
name=CentOS-$releasever - Media
baseurl=file:///data/centos7/
gpgcheck=0
enabled=1
EOF

# 安装全量依赖包
yum clean all
yum makecache
yum install /root/centos7/* --skip-broken -y

```
#### 1.2.2 升级操作系统内核
``` shell
yum install kernel-lt-5.4.256-1.el7.elrepo.x86_64.rpm -y
grubby --set-default $(ls /boot/vmlinuz-* | grep elrepo) 
grubby --default-kernel 
reboot
``` 
#### 1.2.3 sealos基础设施准备
``` shell
# 安装二进制版本sealos
cd /data/ && tar zxvf sealos_4.3.0_linux_amd64.tar.gz sealos && chmod +x sealos && mv sealos /usr/bin && sealos version 
# 导入镜像
sealos load -i kubernetes.tar
sealos load -i calico.tar
sealos load -i helm.tar
```
## 2. 在master节点安装部署k8s
### 2.1 运行部署离线镜像
``` shell
# sealos version must >= v4.1.0
# 不能在集群外部机器执行，需要在master上执行，helm镜像必须写在calico之前
sealos run labring/kubernetes-docker:v1.27-latest labring/helm:v3.12.0 labring/calico:3.25.1 \
     --masters 192.168.192.5 \
     --nodes 192.168.192.6
``` 
### 2.2 常用sealos指令
``` shell
# 安装其他应用都按照之前的操作即可
sealos run 镜像名
# 添加/删除node节点
sealos add/delete --nodes 192.168.64.21,192.168.64.19
# 添加/删除master节点
sealos add/delete --masters 192.168.64.21,192.168.64.19
# 清理集群
sealos reset

``` 