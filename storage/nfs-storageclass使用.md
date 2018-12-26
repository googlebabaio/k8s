<!-- toc -->

* * * * *

# nfs-storageclass的配置过程

## 配置nfs
### 查看hosts配置
```
[root@k8s-master ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
192.168.3.6 k8s-master
192.168.3.25 k8s-node1
192.168.3.26 k8s-node2
192.168.3.22 k8s-node3
```
从上面可以看到我的master节点的名字叫做 `k8s-master` ，在这儿我也准备将nfs的server部署到这个节点上。

### 配置一块盘用作nfs
我用的是lvm进行管理的，在这个地方挂载了一个目录为`/nfsdisk`,大小为50G的目录作为nfs磁盘
```
[root@k8s-master ~]# df -h
Filesystem                                    Size  Used Avail Use% Mounted on
/dev/vda1                                      34G   15G   20G  43% /
devtmpfs                                       16G     0   16G   0% /dev
tmpfs                                          16G   84K   16G   1% /dev/shm
tmpfs                                          16G  137M   16G   1% /run
tmpfs                                          16G     0   16G   0% /sys/fs/cgroup
/dev/mapper/dockerlocal_vg-dockerlocal_lv_01   99G  3.2G   91G   4% /dockerlocal
tmpfs                                         1.6G   16K  1.6G   1% /run/user/42
/dev/mapper/dockerlocal_vg-nfsdisk             50G   53M   47G   1% /nfsdisk
```

### 服务端配置


#### 进行nfs的相关配置
```
# vim /etc/exports
# cat /etc/exports
/nfsdisk 192.168.3.0/24(rw,no_root_squash,no_all_squash,sync)
```

#### 安装相关包
```
yum install -y nfs-utils
```

#### 开始相关服务自启动
```
systemctl enable rpcbind.service
systemctl enable nfs-server.service
systemctl start rpcbind.service
systemctl start nfs-server.service
```

#### 检查
```
[root@k8s-master ~]# rpcinfo -p
   program vers proto   port  service
    100000    4   tcp    111  portmapper
    100000    3   tcp    111  portmapper
    100000    2   tcp    111  portmapper
    100000    4   udp    111  portmapper
    100000    3   udp    111  portmapper
    100000    2   udp    111  portmapper
    100005    1   udp  20048  mountd
    100005    1   tcp  20048  mountd
    100005    2   udp  20048  mountd
    100005    2   tcp  20048  mountd
    100005    3   udp  20048  mountd
    100005    3   tcp  20048  mountd
    100024    1   udp  50252  status
    100024    1   tcp  54712  status
    100003    3   tcp   2049  nfs
    100003    4   tcp   2049  nfs
    100227    3   tcp   2049  nfs_acl
    100003    3   udp   2049  nfs
    100003    4   udp   2049  nfs
    100227    3   udp   2049  nfs_acl
    100021    1   udp  51608  nlockmgr
    100021    3   udp  51608  nlockmgr
    100021    4   udp  51608  nlockmgr
    100021    1   tcp  42493  nlockmgr
    100021    3   tcp  42493  nlockmgr
    100021    4   tcp  42493  nlockmgr
[root@k8s-master ~]# exportfs
/nfsdisk      	192.168.3.0/24
```



### 客户端配置(每个节点都要配置)
在相应的3个节点进行配置

#### 安装相关包并开启服务
```
yum install -y nfs-utils
systemctl enable rpcbind.service
systemctl start rpcbind.service
```

#### 查看挂载的情况
```
[root@k8s-node2 ]# showmount -e k8s-master
Export list for k8s-master:
/nfsdisk 192.168.3.0/24
```

## 配置StorageClass
