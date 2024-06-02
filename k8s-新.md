# k8s 部署文档新版



# 环境准备

### 网络配置

==注意：ip 配置需要和 vm中 net 网络的配置保持同步==

vim /etc/sysconfig/network-scripts/ifcfg-ens33

```sh
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="static"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="ens33"
UUID="12e875a7-5564-4b3d-9ab6-0d2a4c3a33e3"
DEVICE="ens33"
ONBOOT="yes"
IPV6_PRIVACY="no"
IPADDR="192.168.5.40" # 根据 vm 配置来
GATEWAY="192.168.5.2"
NETMASK="255.255.255.0"
DNS1="192.168.5.2"
```

### 节点环境

master 192.168.5.41

node1 192.168.5.42

node2 192.168.5.43

node3 192.168.5.44

### cnetos7 配置阿里云镜像源

```sh
//下载阿里yum源
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo     
//扩展包
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo  

yum makecache        //更新yum仓库
yum clean all       //清理缓存

yum repolist  // 查看仓库列表
```

### 更新工具，下载相关工具

```sh
yum update -y
yum install -y yum-utils
```

### 节点基本设置,所有节点上

```sh
# 设置主机名
hostnamectl set-hostname master

# ip 
192.168.5.41 master 
192.168.5.42 node1 
192.168.5.43 node2
192.168.5.44 node3 

# 时间同步
systemctl start chronyd
systemctl enable chronyd
date
# 禁用SELinux和Firewalld服务：
systemctl stop firewalld
systemctl disable firewalld

sed -i 's/enforcing/disabled/' /etc/selinux/config # 重启后生效

# 临时禁用swap分区
swapoff -a

# 永久禁用swap分区
sed -ri 's/.*swap.*/#&/' /etc/fstab
$ swapoff -a # 临时关闭swap分区
$ vi /etc/fstab # 永久关闭swap分区，注释掉fstab中包含swap的这一行即可
# /dev/mapper/centos-swap swap                    swap    defaults        0 0
$ reboot #重启使其生效
# 需要重启服务器生效
reboot

# 配置网桥过滤和地址转发
cat > /etc/sysctl.d/kubernetes.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# 然后执行,生效
sysctl --system
```



# 安装 Docker ，每个节点上

```sh
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

yum makecache fast(timer)

# 查看可用版本
yum list docker-ce --showduplicates | sort -r

# 安装 
yum -y install docker-ce-20.10.10-3.el8 
yum -y install docker-ce # 安装最新本

systemctl start docker 

# 配置
# 4、添加一个配置文件
#Docker 在默认情况下使用Vgroup Driver为cgroupfs，而Kubernetes推荐使用systemd来替代cgroupfs
[root@master ~]# mkdir /etc/docker
[root@master ~]# vim /etc/docker/daemon.json
{
	"exec-opts": ["native.cgroupdriver=systemd"],
	"registry-mirrors": ["https://kn0t2bca.mirror.aliyuncs.com"]
}

systemctl restart  docker 
systemctl enable  docker # 开机自动启动
```



# k8s 安装

### 设置国内镜像源

```sh
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

### cri-docker

```sh
# 下载地址
https://github.com/Mirantis/cri-dockerd/releases

# 解压并复制过去
tar -zxvf ./cri-dockerd-0.3.14.amd64.tgz
cp -r ./cri-dockerd /usr/bin/
chmod +x /usr/bin/cri-dockerd/cri-dockerd 

# 配置启动
cat >  /etc/systemd/system/cri-dockerd.service << EOF
[Unit]
Description=CRI Interface for Docker Application Container Engine
Documentation=https://docs.mirantis.com
After=network-online.target firewalld.service docker.service
Wants=network-online.target
 
[Service]
Type=notify
ExecStart=/usr/local/bin/cri-dockerd --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.9 --network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin --container-runtime-endpoint=unix:///var/run/cri-dockerd.sock --cri-dockerd-root-directory=/var/lib/dockershim --docker-endpoint=unix:///var/run/docker.sock --cri-dockerd-root-directory=/var/lib/docker
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always
StartLimitBurst=3
StartLimitInterval=60s
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
Delegate=yes
KillMode=process
[Install]
WantedBy=multi-user.target
EOF

# 配置
cat > /etc/systemd/system/cri-dockerd.socket <<EOF
[Unit]
Description=CRI Docker Socket for the API
PartOf=cri-docker.service
[Socket]
ListenStream=/var/run/cri-dockerd.sock
SocketMode=0660
SocketUser=root
SocketGroup=docker
[Install]
WantedBy=sockets.target
EOF


# 
systemctl daemon-reload # 重新加载服务
systemctl start cri-dockerd.service # 启动服务
systemctl enable cri-dockerd.service # 开机自启
```

### 安装组件

```sh
# 安装组件并记录好版本
yum install -y kubelet kubeadm kubectl

chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -enf_conntrack_ipv4

```

### 初始化集群

```sh

 kubeadm init \
--apiserver-advertise-address=192.168.5.50 \
--image-repository registry.aliyuncs.com/google_containers \
--kubernetes-version v1.28.2 \
--service-cidr=10.9.0.0/16 \
--pod-network-cidr=10.8.0.0/16 \
--cri-socket unix:///var/run/cri-dockerd.sock

  
 # 如果初始化失败
 kubeadm reset 
 rm -r $HOME/.kube/config
 
 
kubeadm join 192.168.5.50:6443 --token s7gyv0.952oopqzi48s2t4m \
        --discovery-token-ca-cert-hash sha256:7b590b05e95ae5bbebed6f3f96880533b9149b57c4288fb9e283aff3f2ef6d2a \
        --cri-socket unix:///var/run/cri-dockerd.sock
```

### 部署网络插件 fannel

```sh
vi calio.yaml

- name: CALICO_IPV4POOL_CIDR
   value: "10.8.0.0/16" # 对应初始化命令的 ip
```



### 卸载 k8s 

https://www.jb51.net/server/313713crh.htm



# 使用



















