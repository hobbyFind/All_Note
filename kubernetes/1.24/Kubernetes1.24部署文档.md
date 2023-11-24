# Kubernetes1.24部署

```
    1.24+版本的runtime支持docker，但是移除了docker-shim,即kubenetes连接docker容器运行时的接口。所以1.24+版本部署，若用到docker则还需要安装ari-doockerd用来取代docker-shim。
```

### 主机规划

| 角色   | 主机名   | IP地址        |
| ------ | -------- | ------------- |
| Master | master01 | 192.168.77.10 |
| Node   | node01   | 192.168.77.11 |
| Node   | node02   | 192.168.77.12 |

### 一、环境初始化（每个节点都需要操作）

##### 1.关闭防火墙和Selinux

```powershell
$ systemctl stop firewalld && systemctl disable firewalld
$ setenforce 0
$ sed  -i -r 's/SELINUX=[ep].*/SELINUX=disabled/g' /etc/selinux/config
```

##### 2.添加域名解析

```powershell
$ cat  >> /etc/hosts << EOF
192.168.77.10 master01
192.168.77.11 node01
192.168.77.12 node02
EOF
```

##### 3.时间同步

```powershell
$ yum -y install ntpdate
$ ntpdate ntp1.aliyun.com
```

##### 4.关闭swap分区

```powershell
$ swapoff --all
$ sed -i -r '/swap/ s/^/#/' /etc/fstab
```

##### 5.修改Linux内核参数

```powershell
$ cat >> /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
$ sysctl -p /etc/sysctl.d/kubernetes.conf
```

##### 6.加载网桥过滤器模块

```powershell
$ cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
$ modprobe br_netfilter
$ lsmod | grep br_netfilter 
```

```powershell
# 查看是否添加成功
$ lsmod | grep br_netfilter 
br_netfilter           22256  0
bridge                151336  1 br_netfilter
```

##### 7.配置IPVS功能及相关模块

```powershell
$ yum -y install ipset ipvsadm
$ cat > /etc/sysconfig/modules/ipvs.modules <<EOF
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4  
EOF
$ chmod +x /etc/sysconfig/modules/ipvs.modules
$ /etc/sysconfig/modules/ipvs.modules
```

```powershell
# 查看相关模块是否配置成功
$ lsmod | grep -e ip_vs -e nf_conntrack_ipv4
nf_conntrack_ipv4      15053  3
nf_defrag_ipv4         12729  1 nf_conntrack_ipv4
ip_vs_sh               12688  0
ip_vs_wrr              12697  0
ip_vs_rr               12600  0
ip_vs                 145458  6 ip_vs_rr,ip_vs_sh,ip_vs_wrr
nf_conntrack          139264  10 ip_vs,nf_nat,nf_nat_ipv4,nf_nat_ipv6,xt_conntrack,nf_nat_masquerade_ipv4,nf_nat_masquerade_ipv6,nf_conntrack_netlink,nf_conntrack_ipv4,nf_conntrack_ipv6
libcrc32c              12644  4 xfs,ip_vs,nf_nat,nf_conntrack
```

##### 8.Docker安装

```powershell
# step 1: 安装必要的一些系统工具
$ yum install -y yum-utils device-mapper-persistent-data lvm2
# Step 2: 添加软件源信息
$ yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# Step 3： 修改yum源地址为aliyun
$ sed -i 's+download.docker.com+mirrors.aliyun.com/docker-ce+' /etc/yum.repos.d/docker-ce.repo
# Step 4: 更新并安装Docker-CE
$ yum makecache fast
# kubernetes官方文档中提出1.24版本依赖的docker版本由20.10.7->20.10.12 所以要安装20.10.12以上的版本
# github.com/docker/docker: v20.10.7+incompatible → v20.10.12+incompatible
# 以下命令查看可用的docker版本
$ yum list docker-ce.x86_64 --showduplicates | sort -r
   Loading mirror speeds from cached hostfile
   Loaded plugins: branch, fastestmirror, langpacks
   docker-ce.x86_64            17.03.1.ce-1.el7.centos            docker-ce-stable
   docker-ce.x86_64            17.03.1.ce-1.el7.centos            @docker-ce-stable
   docker-ce.x86_64            17.03.0.ce-1.el7.centos            docker-ce-stable
   Available Packages
# 安装指定版本的Docker-CE: (VERSION例如上面的17.03.0.ce.1-1.el7.centos)
# yum -y install docker-ce-[VERSION]
# 本次安装docker版本为20.10.5
$ yum -y install docker-ce-20.10.5-3.el7
```

```powershell
# 修改docker配置文件，将镜像仓库改为国内
$ vim /etc/docker/daemon.json
{
        "registry-mirrors": ["https://registry.docker-cn.com","http://192.168.77.20","https://kr5tbqen.mirror.aliyuncs.com","https://registry.docker-cn.com","https://docker.mirrors.ustc.edu.cn","https://dockerhub.azk8s.cn","http://hub-mirror.c.163.com","http://qtid6917.mirror.aliyuncs.com", "https://rncxm540.mtc.edu.cn","https://dockerhub.azk8s.cn","http://hub-mirror.c.163.com","http://qtid6917.mirroirror.aliyuncs.com"],
        "insecure-registries": ["192.168.77.20"],
        "exec-opts": ["native.cgroupdriver=systemd"]
}

# 加载配置文件
$ systemctl daemon-reload
# 启动docker并设置为开机自启动
$ systemctl start docker && systemctl enable docker
```

##### 9.cri-dockerd安装

```powershell
# 拉取官方代码到本地
$ git clone https://github.com/Mirantis/cri-dockerd.git
# 进入代码目录
$ cd cri-dockerd
# 安装二进制命令行文件
$ wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.7/cri-dockerd-0.3.7.amd64.tgz
$ tar -zxf cri-dockerd-0.3.7.amd64.tgz
$ cd cri-dockerd
$ mkdir -p /usr/local/bin
# 改属组，属主，权限，并添加到/usr/local/bin下
$ install -o root -g root -m 0755 cri-dockerd /usr/local/bin/cri-dockerd
# 将启动文件部署到启动目录下，以便systemctl可以控制启动
$ install ../packaging/systemd/* /etc/systemd/system
```

```powershell
# 修改启动参数，可以使用cri-dockerd -h  查看启动参数
$ sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
$ cat /etc/systemd/system/cri-docker.service
···
[Service]
Type=notify
ExecStart=/usr/local/bin/cri-dockerd --container-runtime-endpoint unix:///var/run/cri-dockerd.sock --docker-endpoint unix:///var/run/docker.sock --network-plugin cni --pod-infra-container-image registry.aliyuncs.com/google_containers/pause:3.6 --cni-bin-dir /opt/cni/bin                                           #注意：这是一行，默认只添加--pod-infra-container-image为国内源就可以，其他可以不加
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always
···
```

```powershell
# 启动cri-docker
$ systemctl daemon-reload
$ systemctl start cri-docker
$ systemctl enable cri-docker
```

##### 10.kubernetes部署

```powershell
# 添加Yum源
$ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
# 可以使用下面命令查看可装的版本
# yum list kubeadm --showduplicates | sort -r
$ yum -y install kubelet-1.24.5 kubeadm-1.24.5 kubectl-1.24.5
$ systemctl enable kubelet && systemctl start kubelet
```

### 二、Kubernetes初始化（Master节点操作）

```powershell
方法一（简便不推荐）：
$ kubeadm init \
  --apiserver-advertise-address=192.168.77.10 \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version v1.24.5 \
  --service-cidr=10.96.0.0/16 \
  --pod-network-cidr=172.16.10.0/24 \
  --cri-socket=unix:///var/run/cri-dockerd.sock
```

```powershell
方法二：
$ mkdir kube-init && cd kube-init
$ kubeadm config print init-defaults > /kube-init/kubeadm.yaml
$ vim kubeadm.yaml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.77.10                  # Master节点IP
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/cri-dockerd.sock           # 配置为cri-docker的socket文件
  imagePullPolicy: IfNotPresent
  name: master01                                          # 修改为当前主机名
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers              # 改为aliyun镜像仓库或者其他本地仓库
kind: ClusterConfiguration
kubernetesVersion: 1.24.5                                             # 改为目前要安装的版本号
networking:
  dnsDomain: cluster.local
  podSubnet: 172.16.0.0/16                                            # pod网段
  serviceSubnet: 10.96.0.0/16                                         # service网段
scheduler: {}

ps: 
podSubnet、serviceSubnet、advertiseAddress三个网段所包含的IP地址一定不可以重复了
advertiseAddress：更改为master的IP地址
criSocket：指定容器运行时
imageRepository：配置国内加速源地址
podSubnet：pod网段地址
serviceSubnet：services网段地址
末尾添加了指定使用ipvs，开启systemd
nodeRegistration.name：改为当前主机名称

$ kubeadm init --config kubeadm.yaml
```

```powershell
# 初始化成功界面
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

kubeadm join 192.168.77.10:6443 --token abcdef.0123456789abcdef \
	--discovery-token-ca-cert-hash sha256:afef55c724c1713edb7926d98f8c4063fbae928fc4eb11282589d6485029b9a6 
```

```powershell
# 添加集群文件
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

### 三、Node节点加入集群

```powershell
$ kubeadm join 192.168.77.10:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:a94515862c918c0a24ef888cf7ac4f53ff85f0fd38cb02a887f66c638a8d6f8d --cri-socket unix:///var/run/cri-dockerd.sock                                # --cri-socket不可以省略不然会报错
```

### 四、集群完善

```powershell
# 首先到master节点查看node节点是否成功加入   NotReady是由于网络组件没有安装，正常
$ kubectl get node
NAME     STATUS     ROLES           AGE    VERSION
master01 Ready      control-plane   7d2h   v1.24.5
node01   NotReady   <none>          7d2h   v1.24.5
node02   NotReady   <none>          7d2h   v1.24.5

# 安装calico网络组件(注意找到对应版本的calico组件yaml，不然会安装失败)
$ wget https://github.com/projectcalico/calico/blob/master/manifests/calico.yaml
$ kubectl apply -f calico.yaml
# 查看calico组件安装完成
$ kubectl get pod -n kube-system
NAME                                       READY   STATUS        RESTARTS        AGE
calico-kube-controllers-5464b57b48-l9bnw   1/1     Running       0               115m
calico-kube-controllers-5464b57b48-wddps   1/1     Running       10 (4d1h ago)   7d2h
calico-node-2bk95                          1/1     Running       4 (135m ago)    7d2h
calico-node-797h4                          1/1     Running       3 (4d1h ago)    7d2h
calico-node-crmzn                          1/1     Running       4 (122m ago)    7d2h
coredns-74586cf9b6-6bhcq                   1/1     Running       5 (122m ago)    7d2h
coredns-74586cf9b6-6gjbl                   1/1     Running       5 (122m ago)    7d2h
etcd-node                                  1/1     Running       5 (122m ago)    7d2h
kube-apiserver-node                        1/1     Running       6 (122m ago)    7d2h
kube-controller-manager-node               1/1     Running       18 (122m ago)   7d2h
kube-proxy-mj86p                           1/1     Running       4 (135m ago)    7d2h
kube-proxy-nrsxc                           1/1     Running       5 (122m ago)    7d2h
kube-proxy-pgfhv                           1/1     Running       3 (4d1h ago)    7d2h
kube-scheduler-node                        1/1     Running       16 (122m ago)   7d2h
metrics-server-567996d574-tg8mv            1/1     Running       1 (3d19h ago)   4d1h
metrics-server-567996d574-v8vsl            1/1     Running       0               115m
snapshot-controller-0                      1/1     Running       1 (135m ago)    4d1h
# 查看节点状态为Ready
$ kubectl get node
NAME     STATUS     ROLES           AGE    VERSION
master01 Ready      control-plane   7d2h   v1.24.5
node01   Ready      <none>          7d2h   v1.24.5
node02   Ready      <none>          7d2h   v1.24.5
```

### 补充：

```powershell
# kubectl命令补全
$ yum -y install bash-completion
$ echo "source /usr/share/bash-completion/bash_completion" >>/etc/profile
$ echo "source <(kubectl completion bash)" >>/etc/profile
$ source /etc/profile
```

```powershell
# kubernetes环境重置
# Master节点不需要加--cri-socket unix:///var/run/cri-dockerd.sock 也可以
$ kubeadm reset --cri-socket unix:///var/run/cri-dockerd.sock
$ iptables -F
```

```powershell
# 集群token过期处理
$ kubeadm token create --print-join-command
```

