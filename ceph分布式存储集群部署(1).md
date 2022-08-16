# 

# **ceph分布式存储集群部署**



## **集群原理图**

​                 ![img](https://docimg5.docs.qq.com/image/tHR3fvWGG8yfPzS0o8HL0g.png?w=1280&h=757.1692307692308)        

## **集群规划**

| **主机名** | **IP地址**   | **角色**                         |
| ---------- | ------------ | -------------------------------- |
| node1      | 100.65.20.11 | mon,mgr,osd，ceph-deploy管理节点 |
| node2      | 100.65.20.12 | mon,mgr,osd                      |
| node3      | 100.65.20.13 | mon,mgr,osd                      |



## **系统优化**

前提条件：

​		离线安装，需提前配置好局域网yum源： centos os源，epel源，ceph源

### 1. **禁用selinux**

### 2. **关闭防火墙**

### 3. **配置NTP时间同步**

```
yum install chrony -y
sed -i '/iburst/d' /etc/chrony.conf
echo "server 100.65.34.36  iburst"  >>  /etc/chrony.conf
systemctl enable chronyd --now
```

### 4. **添加本地解析**

```shell
cat <<EOF >> /etc/hosts
100.65.20.11  node1
100.65.20.12  node2
100.65.20.13  node3
EOF
```

### 5. **创建部署ceph用户**

后期部署管理集群都用cephadm这个用户

```shell
useradd cephadm && echo cephadm123 | passwd --stdin cephadm
echo "cephadm  ALL=(ALL)  NOPASSWD: ALL" | sudo tee /etc/sudoers.d/cephadm 
chmod 0440 /etc/sudoers.d/cephadm
```

### 6. **配置用户密码ssh认证**

```shell
su - cephadm
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
ssh-copy-id -i ~/.ssh/id_rsa.pub cephadm@localhost
for i in node2 node3;do
  scp -rp ~/.ssh/ cephadm@${i}:~
done
```

## 

## **部署RADOS存储集群**

### 1. **在管理节点安装ceph-deploy**

 这里我们用node1主机作为管理节点

```shell
# su - cephadm
$ sudo yum -y install ceph-deploy python-setuptools python2-subprocess32
```

### 2. **初始化RADOS集群**

#### 2.1. **在管理节点以cephadm用户创建集群相关的配置文件目录**

```
# su - cephadm
$ mkdir ceph-cluster && cd ceph-cluster
```

#### 2.2. **初始化第一个mon节点，准备创建集群**

```
$ ceph-deploy new --cluster-network=172.16.5.0/24 --public-network=100.65.0.0/16 node1
```

#### 2.3. **安装ceph集群：**

```
$ sudo yum -y install ceph ceph-radosgw  python-enum34
```

 \# 建议在每台服务器上手动执行，因如下命令不会自动安装python-enum34包，经测试python-enum34是必要依赖包，不然后期可能会出现Module 'volumes' has failed dependency: No module named enum报错。

```
$ ceph-deploy install --no-adjust-repos node1 node2 node3
```

 注释：--no-adjust-repos 直接使用本地源，不生成官方源

#### 2.4. **初始化mon节点**

```
$ ceph-deploy mon create-initial
$ ceph-deploy --overwrite-conf mon create-initial  # 如有手动修改ceph.conf配置文件，请执行此命令
```

#### 2.5. **把配置文件和admin秘钥拷贝到ceph集群各个节点**

```
$ ceph-deploy admin node1 node2 node3
$ sudo setfacl -m u:cephadm:r /etc/ceph/ceph.client.admin.keyring  # 在3台服务器都要执行
```







1. #### **配置manager节点，启动ceph-mgr进程**



```
$ ceph-deploy mgr create node1
```







1. #### **查看集群状态**



```
$ ceph -s
$ ceph health
HEALTH_WARN OSD count 0 < osd_pool_default_size 3
```





​     如没有其他报错，表示集群安装完成。



## **向RADOS集群添加OSD**

1. #### **列举磁盘**

命令格式： ceph-deploy disk list {node-name [node-name]...}

$ ceph-deploy disk list node1 node2 node3



1. #### **擦净磁盘（删除分区表）**

命令格式：ceph-deploy disk zap {osd-server-name}:{disk-name}

```
$ ceph-deploy disk zap node1 /dev/sda
$ ceph-deploy disk zap node2 /dev/sda
$ ceph-deploy disk zap cehp-2  /dev/sda
```

注释：因我们的环境系统盘安装在sdy/sdz，所以第一块是从sda开始，可以用系统命令lsblk来查看服务器有多少块硬盘



1. #### **添加OSD**

早期ceph-deploy 命令支持在将添加OSD的过程分为两个步骤：准备OSD和激活OSD，在新版本中，此种操作方式已经被废除，添加OSD的步骤只能由命令："ceph-deploy osd create {node} --data {data-disk}" 一次完成，默认使用的存储引擎bluestore

$ ceph-deploy osd create node1 --data /dev/sda

$ ceph-deploy osd create node2 --data /dev/sda

$ ceph-deploy osd create node3  --data /dev/sda



1. #### **列出指定节点上的OSD**

$ ceph-deploy osd list node1 node2 node3



1. #### **ceph命令查看OSD的相关信息**

ceph osd stat

ceph osd ls

ceph osd tree



## **测试上传下载数据对象**

存储数据时，客户端必须首先连接至RADOS集群上某存储磁，而后根据对象名称由相关的CRUSH规则完成数据对象寻址。



## **扩展ceph集群**

1. #### **扩展mon监视器节点**

$ ceph-deploy mon add node2

$ ceph-deploy mon add node3



1. #### **添加mgr节点**

$ ceph-deploy mgr create node2 node3



## **PG规置组计算方法**

​                    (OSDs * 100)

  Total PGs =  ----------------------

​                       pool size

```
存储原理
要实现数据存取需要创建一个pool,创建pool要先分配PG。pool里要分配PG，PG里可以存放多个对象， 对象就是由客户端写入的数据分离的单位。
```

![å¨è¿éæå¥å¾çæè¿°](D:\k8s\watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBATXNzR3Vv,size_20,color_FFFFFF,t_70,g_se,x_16)

# 创建pool

```shell

[root@node1 ~]# ceph osd pool create test_pool 16				#创建一个test_pool的池，并分配16个pg	
[root@node1 ~]# ceph osd pool get test_pool pg_num				#查看指定池的pg数量
pg_num: 16
[root@node1 ~]# ceph osd pool set test_pool pg_num 18			#调整pg的数量
set pool 1 pg_num to 18
[root@node1 ~]# ceph osd pool get test_pool pg_num				#再次查看指定池的pg数量
pg_num: 18
[root@node1 ~]# ceph -s

```

```shell
删除pool
#删除pool之前需在ceph.conf配置文件添加mon_allow_pool_delete=true参数，并重启ceph-mon.target服务
[root@node1 ~]# cd /etc/ceph
[root@node1 ceph]# vim ceph.conf								#打开配置文件添加下面这行参数
mon_allow_pool_delete=true
[root@node1 ceph]# ceph-deploy --overwrite-conf admin node1 node2 node3		#同步文件
[root@node1 ceph]# systemctl restart ceph-mon.target			#重启ceph-mon.target服务
[root@node1 ~]# ceph osd pool delete test_pool test_pool --yes-i-really-really-mean-it
[root@node1 ceph]# ceph -s


```

# 创建Ceph文件存储

```shell

要运行Ceph文件系统, 你必须先创建至少带一个mds的Ceph存储集群(Ceph块设备和Ceph对象存储不使用MDS)。
Ceph MDS： Ceph文件存储类型存放与管理元数据metadata的服务。

#第1步、 在node1节点创建mds服务(最少创建一个mds,也可以做多个mds实现HA)
[root@node1 ceph]# cd /etc/ceph
[root@node1 ceph]# ceph-deploy  mds create node1 node2 node3		#创建3个mds
[root@node1 ceph]# ceph mds stat									#查看mds状态
cephfs-1/1/1 up  {0=node3=up:active}, 2 up:standby
[root@node1 ceph]#
#第2步、 一个Ceph文件系统需要至少两个RADOS存储池，一个用于存放数据，一个用于存放元数据，下面我们就来创建这两个池
[root@node1 ceph]# ceph osd pool  create ceph_data 16				#创建ceph_data池,用于存数据
[root@node1 ceph]# ceph osd pool  create ceph_metadata 8			#创建ceph_metadata池,用于存元数据
#第3步、创建ceph文件系统，并确认客户端访问的节点
[root@node1 ceph]# ceph fs new cephfs ceph_metadata ceph_data    	#cephfs就是ceph文件系统名，即客户端挂载点，ceph_metadata是上一步创建的元数据池，ceph_data是上一步创建的数据此，这两个池的顺序不能乱
new fs with metadata pool 3 and data pool 4
[root@node1 ceph]# ceph fs ls										#查看我们创建的ceph文件系统
name: cephfs, metadata pool: ceph_metadata, data pools: [ceph_data ]
[root@node1 ceph]#

```



## 客户端挂载Ceph文件存储

上一步我们已经创建好了一个叫cephfs的文件存储系统，下面我们就来进行客户端挂载；
一个客户端想要挂载远程上的ceph文件存储系统，它需要连接谁呢？答案是连接monitor监控，所以客户端只需要连接mon监控的节点和端口，而mon监控端口为6789；
在ceph的配置文件ceph.conf中，有auth_client_required = cephx的几行，这几行表示cepg启用了cephx认证, 所以客户端的挂载必须要验证秘钥，而ceph在创建集群的时候就生成一个客户端秘钥；

```
[root@node1 ~]#cd /etc/ceph
[root@node1 ceph]# cat ceph.client.admin.keyring					#在node1上查看客户端的秘钥文件内容，这个文件叫ceph.client.admin.keyring，其中admin是用户名
[client.admin]														#admin是用户名
        key = AQAGn9JhQ3AUOxAAHz+xz6M8B86uoj+gV/ElIw==				#这个就是客户端的秘钥
        caps mds = "allow *"
        caps mgr = "allow *"
        caps mon = "allow *"
        caps osd = "allow *"
[root@node1 ceph]#
[root@client ~]# mkdir /etc/ceph									#在cline客户端创建一个/etc/ceph目录
[root@client ceph]# echo 'AQAGn9JhQ3AUOxAAHz+xz6M8B86uoj+gV/ElIw==' >>admin.key		#新建一个秘钥文件，并把从node1上看到的客户端秘钥复制到这个文件里来
[root@client ceph]# cat admin.key 
AQAGn9JhQ3AUOxAAHz+xz6M8B86uoj+gV/ElIw==
[root@client ceph]#  mkdir /cephfs_data								#先创建一个目录作为挂载点
[root@client ceph]# mount.ceph  node1:6789:/ /cephfs_data/ -o name=admin,secretfile=/etc/ceph/admin.key			
#解读：node1:6789:/ /cephfs_data/，其中6789是mon的端口，客户端挂载连接找mon的，因为我们node1上创建了3个mon,所以这里写node2,node3都可以，斜杠表示找根，找node1的根就是找我们在node1上创建的cephfs文件系统，/cephfs_data/表示挂载到本地的ceph_data目录,-o表示指定参数选项，name=admin,secretfile=/etc/ceph/admin.key 表示使用用户名为admin,秘钥使用/etc/ceph/admin.key秘钥文件进行连接
[root@client ceph]# df -h											#查看是否挂载了
df: 鈥data/gv8鈥 Transport endpoint is not connected
Filesystem               Size  Used Avail Use% Mounted on
devtmpfs                 475M     0  475M   0% /dev
tmpfs                    487M     0  487M   0% /dev/shm
tmpfs                    487M   14M  473M   3% /run
tmpfs                    487M     0  487M   0% /sys/fs/cgroup
/dev/mapper/centos-root  6.2G  1.6G  4.7G  26% /
/dev/sda1               1014M  138M  877M  14% /boot
tmpfs                     98M     0   98M   0% /run/user/0
192.168.118.128:6789:/   532M     0  532M   0% /cephfs_data			#已成功挂载

#可以使用多个客户端, 同时挂载此文件存储,可实现同读同写

```

## 删除Ceph文件存储

```shell
#第1步: 在客户端上删除数据,并umount所有挂载
[root@client /]# cd  /cephfs_data/ 
[root@client cephfs_data]# rm -rf *								#删除全部文件
[root@client /]# umount  /cephfs_data/							#在客户端先卸载
[root@client /]# df -h											#查看一下，发现已经卸载了
Filesystem               Size  Used Avail Use% Mounted on
devtmpfs                 475M     0  475M   0% /dev
tmpfs                    487M     0  487M   0% /dev/shm
tmpfs                    487M   14M  473M   3% /run
tmpfs                    487M     0  487M   0% /sys/fs/cgroup
/dev/mapper/centos-root  6.2G  1.6G  4.7G  26% /
/dev/sda1               1014M  138M  877M  14% /boot
tmpfs                     98M     0   98M   0% /run/user/0
[root@client /]#

#第2步: 停掉所有节点的mds(只有停掉mds才能删除文件存储)，因为我们刚开始的创建了3个mds,所以需要都停掉
[root@node1 ~]# systemctl stop ceph-mds.target					#停掉node1上的mds服务
[root@node2 ~]# systemctl stop ceph-mds.target					#停掉node2上的mds服务
[root@node3 ~]# systemctl stop ceph-mds.target					#停掉node3上的mds服务

#第3步: 回到集群任意一个节点上(node1,node2,node3其中之一)删除Ceph文件存储
[root@node1 ceph]# ceph fs rm cephfs --yes-i-really-mean-it		#删除文件存储，文件存储名就是cephfs 
[root@node1 ceph]#  ceph fs ls									#查看文件存储，已经没有了
No filesystems enabled
[root@node1 ceph]#

#第4步: 删除文件存储的那两个池
[root@node1 ceph]# ceph osd pool ls								#查看池
ceph_metadata
ceph_data
[root@node1 ceph]# ceph osd pool delete ceph_metadata ceph_metadata  --yes-i-really-really-mean-it	#删除ceph_metadata池
pool 'ceph_metadata' removed
[root@node1 ceph]# 
[root@node1 ceph]# ceph osd pool delete ceph_data ceph_data --yes-i-really-really-mean-it			#删除ceph_data池
pool 'ceph_data' removed
[root@node2 ceph]# ceph osd pool ls								#查看池已经被删除了

#第4步: 如果你下次还要创建ceph文件存储，可以把mds服务启动
[root@node1 ~]# systemctl start ceph-mds.target
[root@node2 ~]# systemctl start ceph-mds.target
[root@node3 ~]# systemctl start ceph-mds.target

```

# 创建Ceph块存储

注意：块存储要在客户端上面做的，这一点与gluster不同。

```shell
#第1步: 在node1上同步配置文件到client客户端，因为要在客户端上做块储存
[root@node1 ~]# cd /etc/ceph									#切换目录
[root@node1 ceph]# ceph-deploy admin client						#将node1上的配置文件同步给client客户端

#第2步: 客户端建立存储池,并初始化
[root@client ~]# cd /etc/ceph									#切换目录
[root@client ceph]# ceph osd pool create rbd_pool 64			#创建pool,pg为64个
pool 'rbd_pool' created
[root@client ceph]# rbd pool init rbd_pool						#使用rdb命令初始化存储池

#第3步: 创建一个image(我这里image名为volume1,大小为1024M)
[root@client ceph]# rbd create volume1 --pool rbd_pool --size 1024		#创建一个1G的image，名为volume1 
[root@client ceph]# rbd ls rbd_pool								#查看image
volume1
[root@client ceph]# rbd info volume1 -p rbd_pool				#查看刚才创建image 
rbd image 'volume1':
        size 1 GiB in 256 objects
        order 22 (4 MiB objects)
        id: 86256b8b4567
        block_name_prefix: rbd_data.86256b8b4567
        format: 2
        features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
        op_features: 
        flags: 
        create_timestamp: Wed Jan  5 21:19:19 2022
[root@client ceph]# 
#第4步: 将创建的image 映射成块设备
[root@client ceph]# rbd map rbd_pool/volume1					#将创建的volume1映射成块设备
rbd: sysfs write failed											#因为rbd镜像的一些特性，OS kernel并不支持，所以映射报错
RBD image feature set mismatch. You can disable features unsupported by the kernel with "rbd feature disable rbd_pool/volume1 object-map fast-diff deep-flatten".
In some cases useful info is found in syslog - try "dmesg | tail".
rbd: map failed: (6) No such device or address
[root@client ceph]# rbd feature disable rbd_pool/volume1 object-map fast-diff deep-flatten		#根据上面的报错提示执行这一条命令
[root@client ceph]# rbd map rbd_pool/volume1					#继续将创建的volume1映射成块设备，映射成功
/dev/rbd0														#这个就是映射成功的块设备，可以自由分区格式化了

#第5步: 查看映射
[root@client ceph]# rbd showmapped								#显示已经映射的块设备
id pool     image   snap device    
0  rbd_pool volume1 -    /dev/rbd0 
[root@client ceph]# rbd unmap /dev/rbd0							#使用这条命令可以取消映射

#第6步: 分区，格式化,挂载
[root@client ceph]# fdisk /dev/rbd0								#分区
[root@client ceph]# lsblk										#查看磁盘设备
NAME            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda               8:0    0    8G  0 disk 
sda1            8:1    0    1G  0 part /boot
sda2            8:2    0    7G  0 part 
  centos-root 253:0    0  6.2G  0 lvm  /
  centos-swap 253:1    0  820M  0 lvm  [SWAP]
sr0              11:0    1  4.4G  0 rom  
rbd0            252:0    0    1G  0 disk 
rbd0p1        252:1    0 1020M  0 part 							#已经分好区了
[root@client ceph]# mkfs.xfs /dev/rbd0p1 						#格式化分区，创建文件系统
Discarding blocks...Done.
meta-data=/dev/rbd0p1            isize=512    agcount=8, agsize=32768 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=261120, imaxpct=25
         =                       sunit=1024   swidth=1024 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=624, version=2
         =                       sectsz=512   sunit=8 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[root@client ceph]# 
[root@client ceph]# mount /dev/rbd0p1 /mnt/						#挂载
[root@client ceph]# df -Th										#查看已挂载设备
df: 鈥data/gv8鈥 Transport endpoint is not connected
Filesystem              Type      Size  Used Avail Use% Mounted on
devtmpfs                devtmpfs  475M     0  475M   0% /dev
tmpfs                   tmpfs     487M     0  487M   0% /dev/shm
tmpfs                   tmpfs     487M   14M  473M   3% /run
tmpfs                   tmpfs     487M     0  487M   0% /sys/fs/cgroup
/dev/mapper/centos-root xfs       6.2G  1.6G  4.7G  26% /
/dev/sda1               xfs      1014M  138M  877M  14% /boot
tmpfs                   tmpfs      98M     0   98M   0% /run/user/0
/dev/rbd0p1             xfs      1018M   33M  986M   4% /mnt	#已成功挂载
[root@client ceph]# cat /etc/passwd >>/mnt/file.txt				#读写文件测试，正常

#注意：块存储是不能实现同读同写的,请不要两个客户端同时挂载进行读写

```

## 块存储在线扩容

经测试，分区后`/dev/rbd0p1`不能在线扩容，直接使用`/dev/rbd0`才可以。为了更好的演示扩容，这里把上一步的分区删掉。

```shell
#先把上一步创建的分区删除，直接使用整块磁盘不分区
[root@client ~]# cd /etc/ceph
[root@client ceph]# umount /mnt										#先卸载
[root@client ceph]# fdisk /dev/rbd0									#输入d删除分区
[root@client ceph]# mkfs.xfs /dev/rbd0 -f							#将整个设备格式化，不分区了
[root@client ceph]# mount /dev/rbd0 /mnt							#重新挂载
[root@client ceph]# df -h |tail -1									#查看是否挂载成功，已经挂载成功了
/dev/rbd0               1014M   33M  982M   4% /mnt

#开始在线扩容
[root@client ceph]# rbd resize --size 1500 rbd_pool/volume1			#扩容成1500M，原来是1024M
Resizing image: 100% complete...done.
[root@client ceph]#  rbd info rbd_pool/volume1  |grep size			#现在变成1.5G了
        size 1.5 GiB in 375 objects
[root@client ceph]# xfs_growfs -d /mnt/								#扩展文件系统，虽然上一步扩容了磁盘，但还需要使用xfs_growfs命令来扩容xfs文件系统
[root@client ceph]# df -h |tail -1									#查看是否扩容成功，已经显示为1.5G了
/dev/rbd0                1.5G   33M  1.5G   3% /mnt
[root@client ceph]#

```

## 块存储离线缩容

缩容必须要数据备份，卸载，缩容后重新格式化再挂载，所以这里只做了解，因为企业线上环境一般不会做缩容。

```shell
[root@client ceph]# rbd resize --size 500 rbd_pool/volume1 --allow-shrink	#容量缩小为500M
Resizing image: 100% complete...done.
[root@client ceph]# rbd info rbd_pool/volume1  |grep size					#查看容量，已经是500M了
        size 500 MiB in 125 objects
[root@client ceph]# umount /mnt/											#客户端卸载
[root@client ceph]# mkfs.xfs -f /dev/rbd0									#重新格式化设备
Discarding blocks...Done.
meta-data=/dev/rbd0              isize=512    agcount=8, agsize=16384 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=128000, imaxpct=25
         =                       sunit=1024   swidth=1024 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=624, version=2
         =                       sectsz=512   sunit=8 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[root@client ceph]# mount /dev/rbd0 /mnt/								#重新挂载
[root@client ceph]# df -h |tail -1										#查看已挂载设备，已经是500M了，缩容成功
/dev/rbd0                498M   26M  473M   6% /mnt
[root@client ceph]# 

```

```shell
删除块存储
当一个块存储不需要的时候，我们可以删除块存储。

[root@client ceph]# umount /mnt/										#客户端先卸载
[root@client ceph]# rbd unmap /dev/rbd0									#取消映射
[root@client ceph]# ceph osd pool delete rbd_pool rbd_pool --yes-i-really-really-mean-it		#删除pool
pool 'rbd_pool' removed

```

# 对象存储

```shell
#第1步: 在node1上创建rgw对象网关

[root@node1 ~]# cd /etc/ceph/													#必须先切换到ceph的配置文件目录
[root@node1 ceph]# ceph-deploy rgw create node1									#创建一个对象存储，关键字就是rgw
[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/bin/ceph-deploy rgw create node1
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  rgw                           : [('node1', 'rgw.node1')]
[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
[ceph_deploy.cli][INFO  ]  subcommand                    : create
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7f7b32ef6f38>
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  func                          : <function rgw at 0x7f7b3353f500>
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.rgw][DEBUG ] Deploying rgw, cluster ceph hosts node1:rgw.node1
[node1][DEBUG ] connected to host: node1 
[node1][DEBUG ] detect platform information from remote host
[node1][DEBUG ] detect machine type
[ceph_deploy.rgw][INFO  ] Distro info: CentOS Linux 7.9.2009 Core
[ceph_deploy.rgw][DEBUG ] remote host will use systemd
[ceph_deploy.rgw][DEBUG ] deploying rgw bootstrap to node1
[node1][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[node1][WARNIN] rgw keyring does not exist yet, creating one
[node1][DEBUG ] create a keyring file
[node1][DEBUG ] create path recursively if it doesn't exist
[node1][INFO  ] Running command: ceph --cluster ceph --name client.bootstrap-rgw --keyring /var/lib/ceph/bootstrap-rgw/ceph.keyring auth get-or-create client.rgw.node1 osd allow rwx mon allow rw -o /var/lib/ceph/radosgw/ceph-rgw.node1/keyring
[node1][INFO  ] Running command: systemctl enable ceph-radosgw@rgw.node1
[node1][WARNIN] Created symlink from /etc/systemd/system/ceph-radosgw.target.wants/ceph-radosgw@rgw.node1.service to /usr/lib/systemd/system/ceph-radosgw@.service.
[node1][INFO  ] Running command: systemctl start ceph-radosgw@rgw.node1
[node1][INFO  ] Running command: systemctl enable ceph.target
[ceph_deploy.rgw][INFO  ] The Ceph Object Gateway (RGW) is now running on host node1 and default port 7480		#7480就是对象存储的对外端口
[root@node1 ceph]# lsof -i:7480						#7480就是对象存储的对外端口					
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
radosgw 94111 ceph   40u  IPv4 720930      0t0  TCP *:7480 (LISTEN)
[root@node1 ceph]# 

#第2步: 在客户端测试连接对象网关。连接对象存储需要用户账号秘钥连接，所以需要再客户端创建用户秘钥

#创建账号秘钥，如下所示（radosgw-admin命令其实是yum -y  install ceph-common安装的）
[root@client ceph]# radosgw-admin user create --uid="iflytek" --display-name="iflytek"		
{
    "user_id": "iflytek",						
    "display_name": "iflytek",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "auid": 0,
    "subusers": [],
    "keys": [
        {
            "user": "iflytek",																#用户名
            "access_key": "KPCUG4915AJIEWIYRPP5",											#访问秘钥	
            "secret_key": "bobYij5DMcMM7qnXyQrr8jtcuoPjhFH2nsB7Psim"						#安全密码
        }
    ],
    "swift_keys": [],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": []
}

#第3步：使用S3连接ceph对象网关
#AmazonS3是一种面向Internet的对象存储服务.我们这里可以使用s3工具连接ceph的对象存储进行操作。
[root@client ceph]# yum install s3cmd -y							#安装s3模块，用于连接对象存储

#安装好s3包后就会有s3cmd命令用于连接对象存储，为了方便，我们可以把一些连接参数写成一个.s3cfg文件，s3cmd命令默认会去读这个文件
[root@client ~]# vim /root/.s3cfg									#创建文件，并写入一下内容
[default]
access_key = KPCUG4915AJIEWIYRPP5									#这个访问秘钥就是我们创建iflytek用户时的访问秘钥					
secret_key = bobYij5DMcMM7qnXyQrr8jtcuoPjhFH2nsB7Psim				#这个安全秘钥就是我们创建iflytek用户时的安全秘钥	
host_base = 192.168.118.128:7480									#对象存储的IP和端口
host_bucket = 192.168.118.128:7480/%(bucket)						#桶，对象存储的IP和端口
cloudfront_host = 192.168.118.128:7480
use_https = False

#第4步：s3cmd命令测试连接对象存储
[root@client ceph]# s3cmd mb s3://iflytek							#创建一个名为iflytek的桶，桶的概念可理解为目录
Bucket s3://iflytek/ created
[root@client ceph]# s3cmd ls										#查看桶
2022-01-05 16:19  s3://iflytek
[root@client ~]# s3cmd put /var/log/yum.log  s3://iflytek			#put文件到桶，即上传文件
upload: /var/log/yum.log -> s3://iflytek/yum.log  [1 of 1]
 5014 of 5014   100% in    1s     3.72 KB/s  done
[root@client ~]# s3cmd get s3://iflytek/yum.log						#get文件到本地，即下载文件到本地
download: s3://iflytek/yum.log -> ./yum.log  [1 of 1]
 5014 of 5014   100% in    0s  1106.05 KB/s  done
[root@client ~]# ll													#查看文件
total 12
-rw-------. 1 root root 1203 Jan  1 15:45 anaconda-ks.cfg
-rw-r--r--. 1 root root 5014 Jan  5 16:22 yum.log					#文件已经下载到本地了

```

