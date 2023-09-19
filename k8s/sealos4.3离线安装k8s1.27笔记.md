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

# 在内网机器上创建repo配置文件
rm -rf /etc/yum.repos.d/*
cat > /etc/yum.repos.d/123.repo  << EOF 
[cby]
name=CentOS-$releasever - Media
baseurl=file:///root/centos7/
gpgcheck=0
enabled=1
EOF

# 安装下载好的包
yum clean all
yum makecache
yum install /root/centos7/* --skip-broken -y
```
#### 1.1.2 sealos基础设施准备
``` shell
# 查看sealos可用版本
curl --silent "https://api.github.com/repos/labring/sealos/releases" | jq -r '.[].tag_name'

# 二进制安装4.3版本
$ wget https://github.com/labring/sealos/releases/download/${VERSION}/sealos_${VERSION#v}_linux_amd64.tar.gz \
   && tar zxvf sealos_${VERSION#v}_linux_amd64.tar.gz sealos && chmod +x sealos && mv sealos /usr/bin
``` 
### 1.2 master节点和计算节点
#### 1.2.1 环境准备
``` shell
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
```
#### 1.2.2 基础设施准备
