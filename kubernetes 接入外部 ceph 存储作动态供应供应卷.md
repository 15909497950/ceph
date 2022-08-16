# kubernetes 接入外部 ceph 存储作动态供应卷

通过 ceph [-csi](https://github.com/ceph/ceph-csi/)将 Ceph Block Device images 与 Kubernetes v1.13 及更高版本一起使用 ，该映像动态提供 RBD 映像以支持 Kubernetes [卷，](https://kubernetes.io/docs/concepts/storage/volumes/)并将这些 RBD 映像映射为工作节点上的块设备（可选择挂载包含在映像中的文件系统）运行 引用 RBD 支持的卷的[pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/)。Ceph 将块设备映像作为跨集群的对象进行条带化，这意味着大型 Ceph 块设备映像比独立服务器具有更好的性能！

![img](https://docs.ceph.com/en/latest/_images/ditaa-b2ae4b6fe24a5fb214de31111e1dd566eb60e83e.png)

`ceph-csi`默认情况下使用 RBD 内核模块，该模块可能不支持所有 Ceph [CRUSH 可调参数](https://docs.ceph.com/en/latest/rados/operations/crush-map/#tunables)或[RBD 映像功能](https://docs.ceph.com/en/latest/rbd/rbd-config-ref/#image-features)。

测试环境：k8s v1.20 和 ceph nautilus  14.2.22

```
[root@k8s-0 ~]# kubectl get node
NAME              STATUS   ROLES                  AGE     VERSION
k8s-0.novalocal   Ready    control-plane,master   5d23h   v1.20.7
k8s-1.novalocal   Ready    control-plane,master   5d23h   v1.20.7
k8s-2.novalocal   Ready    control-plane,master   5d23h   v1.20.7
k8s-3.novalocal   Ready    <none>                 5d23h   v1.20.7
k8s-4.novalocal   Ready    <none>                 5d23h   v1.20.7
k8s-5.novalocal   Ready    <none>                 5d23h   v1.20.7
k8s-6.novalocal   Ready    <none>                 5d23h   v1.20.7
k8s-7.novalocal   Ready    <none>                 5d23h   v1.20.7
```



```
[root@ceph-0 ~]# ceph version
ceph version 14.2.22 (ca74598065096e6fcbd8433c8779a2be0c889351) nautilus (stable)
[root@ceph-0 ~]# ceph -s
  cluster:
    id:     de8760e9-91e3-40b2-88f4-ec4767cb3adf
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum ceph-0,ceph-1,ceph-2 (age 5d)
    mgr: ceph-0(active, since 2w)
    osd: 9 osds: 9 up (since 5d), 9 in (since 5d)
 
  data:
    pools:   1 pools, 128 pgs
    objects: 12 objects, 66 B
    usage:   9.1 GiB used, 741 GiB / 750 GiB avail
    pgs:     128 active+clean
```

下载CEPH-CSI包：

https://github.com/ceph/ceph-csi

https://github.com/ceph/ceph-csi/releases/tag/v3.4.0

```
wget -O ceph-csi-3.4.0.tar.gz https://codeload.github.com/ceph/ceph-csi/tar.gz/refs/tags/v3.4.0

# tar -xvf ceph-csi-3.4.0.tar.gz 
# cd ceph-csi-3.4.0/deploy/rbd/kubernetes/
# ls
csi-config-map.yaml  csi-nodeplugin-psp.yaml   csi-provisioner-psp.yaml   csi-rbdplugin-provisioner.yaml
csidriver.yaml       csi-nodeplugin-rbac.yaml  csi-provisioner-rbac.yaml  csi-rbdplugin.yaml
```

## 创建池

```
ceph osd pool create kubernetes 128 128
rbd pool init kubernetes               # 初始化池
ceph osd pool application enable kubernetes rbd
```

## 配置CEPH-CSI

```
$ ceph auth get-or-create client.kubernetes mon 'profile rbd' osd 'profile rbd pool=kubernetes' mgr 'profile rbd pool=kubernetes'
[client.kubernetes]
    key = AQC+1mthdO4aFBAA7xeTW3mUbuM2jTXMKkXnIA==
    
$ ceph auth get client.kubernetes
[client.kubernetes]
	key = AQC+1mthdO4aFBAA7xeTW3mUbuM2jTXMKkXnIA==
	caps mgr = "profile rbd pool=kubernetes"
	caps mon = "profile rbd"
	caps osd = "profile rbd pool=kubernetes"
exported keyring for client.kubernetes
```

### 生成CEPH-CSI CONFIGMAP

```
$ ceph mon dump
epoch 3
fsid de8760e9-91e3-40b2-88f4-ec4767cb3adf
last_changed 2021-10-17 12:47:35.979700
created 2021-10-08 20:49:45.684629
min_mon_release 14 (nautilus)
0: [v2:172.16.2.96:3300/0,v1:172.16.2.96:6789/0] mon.ceph-0
1: [v2:172.16.2.97:3300/0,v1:172.16.2.97:6789/0] mon.ceph-1
2: [v2:172.16.2.98:3300/0,v1:172.16.2.98:6789/0] mon.ceph-2
dumped monmap epoch 3
```

`ceph-csi`目前仅支持[旧版 V1 协议](https://docs.ceph.com/en/latest/rados/configuration/msgr2/#address-formats)。

生成类似于以下示例的csi-config-map.yaml文件，将fsid替换为“clusterID”，将监视器地址替换为“监视器”：

```
$ cat <<EOF > csi-config-map.yaml 
---
apiVersion: v1
kind: ConfigMap
data:
  config.json: |-
    [
      {
        "clusterID": "de8760e9-91e3-40b2-88f4-ec4767cb3adf",
        "monitors": [
          "172.16.2.96:6789",
          "172.16.2.97:6789",
          "172.16.2.98:6789"
        ]
      }
    ]

metadata:
  name: ceph-csi-config
  
$ kubectl apply -f csi-config-map.yaml
```

ceph-csi 的最新版本还需要一个额外的ConfigMap对象来定义密钥管理服务 (KMS) 提供程序的详细信息。如果未设置 KMS，请将空配置放入csi-kms-config-map.yaml文件或参考[https://github.com/ceph/ceph-csi/tree/master/examples/kms 中的](https://github.com/ceph/ceph-csi/tree/master/examples/kms)示例：

```
$ cat <<EOF > csi-kms-config-map.yaml
---
apiVersion: v1
kind: ConfigMap
data:
  config.json: |-
    {}
metadata:
  name: ceph-csi-encryption-kms-config
EOF

$ kubectl apply -f csi-kms-config-map.yaml
```

ceph-csi 的最新版本还需要另一个ConfigMap对象来定义 Ceph 配置以添加到 CSI 容器内的 ceph.conf 文件：

```
$ cat <<EOF > ceph-config-map.yaml
---
apiVersion: v1
kind: ConfigMap
data:
  ceph.conf: |
    [global]
    auth_cluster_required = cephx
    auth_service_required = cephx
    auth_client_required = cephx
  # keyring is a required key and its value should be empty
  keyring: |
metadata:
  name: ceph-config
EOF

$ kubectl apply -f ceph-config-map.yaml
```

### 生成CEPH -CSI CEPHX SECRET

ceph -csi需要 cephx 凭据才能与 Ceph 集群通信。使用新创建的 Kubernetes 用户 ID 和 cephx 密钥生成一个类似于以下示例的csi-rbd-secret.yaml文件：

```
$ cat <<EOF > csi-rbd-secret.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: csi-rbd-secret
  namespace: default
stringData:
  userID: kubernetes
  userKey: AQC+1mthdO4aFBAA7xeTW3mUbuM2jTXMKkXnIA==
EOF

$ kubectl apply -f csi-rbd-secret.yaml
```

### 配置CEPH-CSI插件

创建所需的ServiceAccount和 RBAC ClusterRole / ClusterRoleBinding Kubernetes 对象。

```
$ kubectl apply -f csi-rbdplugin-provisioner.yaml
$ kubectl apply -f csi-rbdplugin.yaml
```

## 使用 CEPH 块设备

### 创建STORAGECLASS

Kubernetes StorageClass定义了一类存储。 可以创建多个StorageClass对象以映射到不同的服务质量级别（即 NVMe 与基于 HDD 的池）和功能。

例如，要创建一个映射到 上面创建的kubernetes池的ceph -csi StorageClass，在确保“clusterID”属性与您的 Ceph 集群的fsid匹配后，可以使用以下 YAML 文件：

```
$ cat <<EOF > csi-rbd-sc.yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: csi-rbd-sc
   annotations:
     storageclass.beta.kubernetes.io/is-default-class: "true"
provisioner: rbd.csi.ceph.com
parameters:
   clusterID: de8760e9-91e3-40b2-88f4-ec4767cb3adf
   pool: kubernetes
   imageFeatures: layering
   csi.storage.k8s.io/provisioner-secret-name: csi-rbd-secret
   csi.storage.k8s.io/provisioner-secret-namespace: default
   csi.storage.k8s.io/controller-expand-secret-name: csi-rbd-secret
   csi.storage.k8s.io/controller-expand-secret-namespace: default
   csi.storage.k8s.io/node-stage-secret-name: csi-rbd-secret
   csi.storage.k8s.io/node-stage-secret-namespace: default
reclaimPolicy: Delete
allowVolumeExpansion: true
mountOptions:
   - discard
EOF

$ kubectl apply -f csi-rbd-sc.yaml
```

### 创建一个PERSISTENTVOLUMECLAIM

甲PersistentVolumeClaim是由用户为抽象存储资源的请求。然后PersistentVolumeClaim将与Pod资源相关联以提供PersistentVolume，该PersistentVolume将由 Ceph 块映像支持。可以包含一个可选的volumeMode以在挂载的文件系统（默认）或基于原始块设备的卷之间进行选择。

使用ceph -csi，为volumeMode指定Filesystem可以支持 ReadWriteOnce和ReadOnlyMany accessMode声明，并且 为volumeMode指定Block可以支持ReadWriteOnce、ReadWriteMany和 ReadOnlyMany访问模式声明。

例如，要使用上面创建的基于ceph-csi的StorageClass创建基于块的PersistentVolumeClaim，可以使用以下 YAML 从csi-rbd-sc StorageClass请求原始块存储：

```
$ cat <<EOF > raw-block-pvc.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: raw-block-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  resources:
    requests:
      storage: 1Gi
  storageClassName: csi-rbd-sc
EOF
$ kubectl apply -f raw-block-pvc.yaml
```

要使用上面创建的基于ceph-csi的StorageClass创建基于文件系统的PersistentVolumeClaim， 可以使用以下 YAML 从csi-rbd-sc StorageClass请求挂载的文件系统（由 RBD 映像支持）：

```
$ cat <<EOF > pvc.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rbd-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 1Gi
  storageClassName: csi-rbd-sc
EOF
$ kubectl apply -f pvc.yaml
```

以下演示和示例将上述PersistentVolumeClaim绑定 到Pod资源作为挂载的文件系统：

```
$ cat <<EOF > pod.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: csi-rbd-demo-pod
spec:
  containers:
    - name: web-server
      #image: nginx
      image: 100.65.34.29:8082/nginx:1.19
      volumeMounts:
        - name: mypvc
          mountPath: /var/lib/www/html
  volumes:
    - name: mypvc
      persistentVolumeClaim:
        claimName: rbd-pvc
        readOnly: false
EOF
$ kubectl apply -f pod.yaml
```



参考：https://docs.ceph.com/en/latest/rbd/rbd-kubernetes/

