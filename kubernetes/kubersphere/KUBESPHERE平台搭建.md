# KUBESPHERE平台搭建 

```
KUBESPHERE官网:  https://kubesphere.io/zh/docs/v3.4/quick-start/minimal-kubesphere-on-k8s/        
nfs-subdir-external-provisioner地址:  https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner

ps: 本次安装KUBESPHERE是基于NFS作为动态存储，由于kubernetes内部并没有集成NFS的连接器，需要依赖外部NFS连接器，即：nfs-subdir-external-provisioner，所以本次安装顺序为：NFS服务器搭建 ——> kubernetes的nfs-subdir-external-provisioner安装 ——> StorageClass安装 ——> KUBESPHERE安装
```

#### 通用先决条件如下（https://kubesphere.io/zh/docs/v3.4/installing-on-kubernetes/introduction/prerequisites/）

```
1.如需在 Kubernetes 上安装 KubeSphere 3.4，您的 Kubernetes 版本必须为：v1.20.x、v1.21.x、v1.22.x、v1.23.x、* v1.24.x、* v1.25.x 和 * v1.26.x。带星号的版本可能出现边缘节点部分功能不可用的情况。因此，如需使用边缘节点，推荐安装 v1.23.x。
2.可用 CPU > 1 核；内存 > 2 G。CPU 必须为 x86_64，暂时不支持 Arm 架构的 CPU。
3.Kubernetes 集群已配置默认 StorageClass（请使用 kubectl get sc 进行确认）。
4.若安装devops需要集群配置内存>8.6G，cpu:4
ps: 如需安装更多插件使用，需要查看官网（github搜索kubesphere进入）-启用可插拔组件-概述。建议先把有关镜像拉取到本地，防止安装失败。
```

```powershell
ps: 如需安装更多插件使用，需要查看官网（github搜索kubesphere进入）-启用可插拔组件-概述。建议先把有关镜像拉取到本地，防止安装失败。
k8s的每个节点注意安装nfs-utils,防止挂载失败。
$ yum -y install nfs-utils
```

#### 一、利用nfs创建StoragesClass

###### 1.搭建nfs服务器

```powershell
#安装NFS相关软件
$ yum install rpcbind nfs-utils -y
#创建共享目录，并给予权限
$ mkdir /nfs && chmod -R 777 /nfs
#编辑配置文件，共享出去要共享的目录
$ vim /etc/exports
/nfs 192.168.77.0/24(rw,sync)
#启动nfs服务，并设置为开机自启动
$ systemctl start nfs
$ systemctl start rpcbind
$ systemctl enable nfs
$ systemctl enable rpcbind
```

###### 2.安装外部存储nfs

```powershell
#拉取nfs外部连接器代码
$ git clone https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner.git
$ cd nfs-subdir-external-provisioner/deploy
# 创建一个要部署外部存储NFS的命名空间，默认为default
$ kubectl create namespace nfs-external
# 修改rbac.yaml deployment.yaml文件中的命名空间
$ sed -i 's/namespace: default/namespace: nfs-external/g' rbac.yaml deployment.yaml
# 创建RBAC权限控制
$ kubectl apply -f rbac.yaml

ps:  注意查看是否在该命名空间下是不是创建成功
```

###### 3.修改deployment.yaml文件中的参数

```powershell
$ vim deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: nfs-external                            #命名空间检查
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: registry.k8s.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: k8s-sigs.io/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: 192.168.77.20                #nfs服务器地址
            - name: NFS_PATH
              value: /nfs                        #nfs共享文件路径
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.77.20                  #nfs服务器地址
            path: /nfs                             #nfs路径，必须挂载出去，且和环境变量一致。否则利用StorageClass创建PV将不会自动创建相关目录
            
# 创建deployment
$ kubectl apply -f deployment.yaml
```

#### 二、.创建StorageClass用于动态申请pv

Parameters:

| Name            | Description                                                  | Default                                                      |
| --------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| onDelete        | If it exists and has a delete value, delete the directory, if it exists and has a retain value, save the directory. | will be archived with name on the share: `archived-<volume.Name>` |
| archiveOnDelete | If it exists and has a false value, delete the directory. if `onDelete` exists, `archiveOnDelete` will be ignored. | will be archived with name on the share: `archived-<volume.Name>` |
| pathPattern     | Specifies a template for creating a directory path via PVC metadata's such as labels, annotations, name or namespace. To specify metadata use `${.PVC.<metadata>}`. Example: If folder should be named like `<pvc-namespace>-<pvc-name>`, use `${.PVC.namespace}-${.PVC.name}` as pathPattern. | n/a                                                          |

```powershell
#直接使用本路径下的class直接修改，或自己写
$ vim class.yaml

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-pvc
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner # 改名字必须匹配刚创建deployment中的'PROVISIONER_NAME'环境变量
parameters:
  onDelete: delete                    #当删除pv时，自动创建的nfs文件夹也会删除。若想保留，可以用 retain
  archiveOnDelete: "false"            #当onDelete字段存在时，该字段不生效，可以不配置或注释掉。

$ kubectl apply -f class.yaml
```

```powershell
#测试pv是否可以自动申请，且pvc可以正常挂载
$ vim test-claim.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
spec:
  storageClassName: nfs-pvc             #修改为自己的StorageClass名字
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi

$ kubectl apply -f test-claim.yaml
#1.查看是否自动创建pv,且pvc绑定成功。
$ kubectl get pv,pvc
#2.到nfs共享目录下，查看是否自动创建与pv名字相同的目录（NFS服务器上面操作）
$ ls /nfs

$ kubectl apply -f test-pod.yaml
#等容器启动完成，状态为templated，则挂载pvc没问题
$ kubectl get pod

#删除测试的资源
$ kubectl delete -f test-pod.yaml -f test-claim.yaml
#查看pv是否正常删除
$ kubectl get pvc,pv
#NFS服务器上查看自动创建的pv目录是否自动删除
$ ls /nfs

ps: 到此，利用nfs创建的StorageClass配置完成。
```

#### 三、部署KUBESPHERE

###### 1.拉取KUBESPHERE资源

```powershell
#创建一个目录用来放置kubesphere的安装文件
$ mkdir /root/kubesphere
$ cd /root/kubesphere
#拉取KUBESPHERE的安装文件
$ wget https://github.com/kubesphere/ks-installer/releases/download/v3.4.0/kubesphere-installer.yaml
$ wget https://github.com/kubesphere/ks-installer/releases/download/v3.4.0/cluster-configuration.yaml
```

###### 2.修改配置参数

```powershell
#修改默认的动态存储类为自己创建的nfs-pvc
$ vim cluster-configuration.yaml

apiVersion: installer.kubesphere.io/v1alpha1
kind: ClusterConfiguration
metadata:
  name: ks-installer
  namespace: kubesphere-system
  labels:
    version: v3.4.0
spec:
  persistence:
    storageClass: "nfs-pvc"         # 改为已有的storageClass
......

ps: 默认文件是最小化的安装，许多插件默认不会安装，若要一键全部安装，请将所有enabled: false 改为 enabled: true，也可以选择性的修改。
```

###### 3.安装KUBESPHERE

```powershell
kubectl apply -f kubesphere-installer.yaml -f cluster-configuration.yaml
# 查看pod启动是否成功，
kubectl get pod -n kubesphere-system
# 若成功则用以下命令查看安装过程
kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l 'app in (ks-install, ks-installer)' -o jsonpath='{.items[0].metadata.name}') -f
```

###### 4.安装完成显示如下：

```powershell
#####################################################
###              Welcome to KubeSphere!           ###
#####################################################

Console: http://192.168.0.2:30880
Account: admin
Password: P@88w0rd

NOTES：
  1. After logging into the console, please check the
     monitoring status of service components in
     the "Cluster Management". If any service is not
     ready, please wait patiently until all components
     are ready.
  2. Please modify the default password after login.

#####################################################
https://kubesphere.io             20xx-xx-xx xx:xx:xx
#####################################################
```

