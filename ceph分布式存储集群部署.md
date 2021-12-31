# **ceph分布式存储集群部署**



## **集群原理图**

​                 ![img](https://docimg5.docs.qq.com/image/tHR3fvWGG8yfPzS0o8HL0g.png?w=1280&h=757.1692307692308)        

## **集群规划**

| **主机名** | **IP地址**   | **角色**                         |
| ---------- | ------------ | -------------------------------- |
| ceph-0     | 100.65.20.11 | mon,mgr,osd，ceph-deploy管理节点 |
| ceph-1     | 100.65.20.12 | mon,mgr,osd                      |
| ceph-2     | 100.65.20.13 | mon,mgr,osd                      |



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

```
cat <<EOF >> /etc/hosts
100.65.20.11  ceph-0
100.65.20.12  ceph-1
100.65.20.13  ceph-2
EOF
```

### 5. **创建部署ceph用户**

后期部署管理集群都用cephadm这个用户

```
useradd cephadm && echo cephadm123 | passwd --stdin cephadm
echo "cephadm  ALL=(ALL)  NOPASSWD: ALL" | sudo tee /etc/sudoers.d/cephadm 
chmod 0440 /etc/sudoers.d/cephadm
```

### 6. **配置用户密码ssh认证**

```
su - cephadm
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
ssh-copy-id -i ~/.ssh/id_rsa.pub cephadm@localhost
for i in ceph-1 ceph-2;do
  scp -rp ~/.ssh/ cephadm@${i}:~
done
```

## 

## **部署RADOS存储集群**

### 1. **在管理节点安装ceph-deploy**

 这里我们用ceph-0主机作为管理节点

```
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
$ ceph-deploy new --cluster-network=172.16.5.0/24 --public-network=100.65.0.0/16 ceph-0
```

#### 2.3. **安装ceph集群：**

```
$ sudo yum -y install ceph ceph-radosgw  python-enum34
```

 \# 建议在每台服务器上手动执行，因如下命令不会自动安装python-enum34包，经测试python-enum34是必要依赖包，不然后期可能会出现Module 'volumes' has failed dependency: No module named enum报错。

```
$ ceph-deploy install --no-adjust-repos ceph-0 ceph-1 ceph-2
```

 注释：--no-adjust-repos 直接使用本地源，不生成官方源

#### 2.4. **初始化mon节点**

```
$ ceph-deploy mon create-initial
$ ceph-deploy --overwrite-conf mon create-initial  # 如有手动修改ceph.conf配置文件，请执行此命令
```

#### 2.5. **把配置文件和admin秘钥拷贝到ceph集群各个节点**

```
$ ceph-deploy admin ceph-0 ceph-1 ceph-2
$ sudo setfacl -m u:cephadm:r /etc/ceph/ceph.client.admin.keyring  # 在3台服务器都要执行
```







1. #### **配置manager节点，启动ceph-mgr进程**



```
$ ceph-deploy mgr create ceph-0
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

$ ceph-deploy disk list ceph-0 ceph-1 ceph-2



1. #### **擦净磁盘（删除分区表）**

命令格式：ceph-deploy disk zap {osd-server-name}:{disk-name}

```
$ ceph-deploy disk zap ceph-0 /dev/sda
$ ceph-deploy disk zap ceph-1 /dev/sda
$ ceph-deploy disk zap cehp-2  /dev/sda
```

注释：因我们的环境系统盘安装在sdy/sdz，所以第一块是从sda开始，可以用系统命令lsblk来查看服务器有多少块硬盘



1. #### **添加OSD**

早期ceph-deploy 命令支持在将添加OSD的过程分为两个步骤：准备OSD和激活OSD，在新版本中，此种操作方式已经被废除，添加OSD的步骤只能由命令："ceph-deploy osd create {node} --data {data-disk}" 一次完成，默认使用的存储引擎bluestore

$ ceph-deploy osd create ceph-0 --data /dev/sda

$ ceph-deploy osd create ceph-1 --data /dev/sdb

$ ceph-deploy osd create ceph-2  --data /dev/sdb



1. #### **列出指定节点上的OSD**

$ ceph-deploy osd list ceph-0 ceph-1 ceph-2



1. #### **ceph命令查看OSD的相关信息**

ceph osd stat

ceph osd ls

ceph osd tree



## **测试上传下载数据对象**

存储数据时，客户端必须首先连接至RADOS集群上某存储磁，而后根据对象名称由相关的CRUSH规则完成数据对象寻址。



## **扩展ceph集群**

1. #### **扩展mon监视器节点**

$ ceph-deploy mon add ceph-1

$ ceph-deploy mon add ceph-2



1. #### **添加mgr节点**

$ ceph-deploy mgr create ceph-1 ceph-2



## **PG规置组计算方法**

​                    (OSDs * 100)

  Total PGs =  ----------------------

​                       pool size
