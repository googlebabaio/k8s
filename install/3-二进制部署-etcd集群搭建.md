
<!-- toc --> 
# 搭建三节点的etcd
搭建步骤：
1. 准备安装包
2. 创建etcd签名请求：
3. 生成etcd证书和私钥
4. 分发etcd的证书
5. 设置ETCD配置文件
6. 配置完后启动etcd
7. 查看状态集群的状态是否正常


## 1.准备安装包
```
[root@master ssl]# cd /usr/local/src/
[root@master src]# ll
total 575416
-rw-r--r-- 1 root root  17108856 Sep 18 16:59 cni-plugins-amd64-v0.7.1.tgz
-rw-r--r-- 1 root root  11254519 Sep 18 16:57 etcd-v3.3.9-linux-amd64.tar.gz
-rw-r--r-- 1 root root   9706487 Sep 18 16:58 flannel-v0.10.0-linux-amd64.tar.gz
drwxr-xr-x 9 root root      4096 Sep 10 02:44 kubernetes
-rw-r--r-- 1 root root  13908969 Sep 18 16:59 kubernetes-client-linux-amd64.tar.gz
-rw-r--r-- 1 root root  99541161 Sep 18 16:59 kubernetes-node-linux-amd64.tar.gz
-rw-r--r-- 1 root root 435868550 Sep 18 16:59 kubernetes-server-linux-amd64.tar.gz
-rw-r--r-- 1 root root   1818410 Sep 18 16:59 kubernetes.tar.gz
[root@master src]# tar zxf etcd-v3.3.9-linux-amd64.tar.gz 
[root@master src]# cd etcd-v3.3.9-linux-amd64
[root@master etcd-v3.3.9-linux-amd64]# ls
Documentation  etcd  etcdctl  README-etcdctl.md  README.md  READMEv2-etcdctl.md
[root@master etcd-v3.3.9-linux-amd64]# cp etcd etcdctl /etc/kubernetes/bin
[root@master etcd-v3.3.9-linux-amd64]# scp etcd etcdctl node1:/etc/kubernetes/bin
[root@master etcd-v3.3.9-linux-amd64]# scp etcd etcdctl node2:/etc/kubernetes/bin
[root@master etcd-v3.3.9-linux-amd64]# 
```

## 2.创建etcd签名请求：
```
[root@master temp]# cat etcd-csr.json 
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
"172.18.53.221",
"172.18.53.223",
"172.18.53.224"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

## 3.生成etcd证书和私钥
```
[root@master temp]# cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
   -ca-key=/etc/kubernetes/ssl/ca-key.pem \
   -config=/etc/kubernetes/ssl/ca-config.json \
   -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
2018/09/20 09:21:30 [INFO] generate received request
2018/09/20 09:21:30 [INFO] received CSR
2018/09/20 09:21:30 [INFO] generating key: rsa-2048
2018/09/20 09:21:30 [INFO] encoded CSR
2018/09/20 09:21:30 [INFO] signed certificate with serial number 59419064252234230113380579545600310151937183620
2018/09/20 09:21:30 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
[root@master temp]# 
[root@master temp]# ls -lrt etcd*
-rw-r--r-- 1 root root  287 Sep 20 09:20 etcd-csr.json
-rw-r--r-- 1 root root 1436 Sep 20 09:21 etcd.pem
-rw------- 1 root root 1675 Sep 20 09:21 etcd-key.pem
-rw-r--r-- 1 root root 1062 Sep 20 09:21 etcd.csr
```

## 4.分发etcd的证书
```
[root@master temp]# cp etcd*.pem /etc/kubernetes/ssl
[root@master temp]# scp etcd*.pem node1:/etc/kubernetes/ssl
[root@master temp]# scp etcd*.pem node2:/etc/kubernetes/ssl
[root@master temp]# 
```

## 5.设置ETCD配置文件
设置etcd的配置有两种方式：
* 所有的配置都写在service文件中
* 配置文件写在`/ect/kubernetes/cfg`中，然后在service文件中加上配置的地址

下面我把两种方式的实现都写出来
### 5.1 配置方法1
先配置etcd.conf
注意：除了`ETCD_INITIAL_CLUSTER`不改变外，剩下的其他地方涉及到了ip地址的，都需要替换成相应node的ip地址。
另外，要注意`ETCD_NAME`要和下面的`ETCD_INITIAL_CLUSTER`中描述的名字对应上。
```
[root@master ssl]# cat /etc/kubernetes/cfg/etcd.conf
#[member]
ETCD_NAME="etcd-node1"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
#ETCD_SNAPSHOT_COUNTER="10000"
#ETCD_HEARTBEAT_INTERVAL="100"
#ETCD_ELECTION_TIMEOUT="1000"
ETCD_LISTEN_PEER_URLS="https://172.18.53.221:2380"
ETCD_LISTEN_CLIENT_URLS="https://172.18.53.221:2379,https://127.0.0.1:2379"
#ETCD_MAX_SNAPSHOTS="5"
#ETCD_MAX_WALS="5"
#ETCD_CORS=""
#[cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://172.18.53.221:2380"
# if you use different ETCD_NAME (e.g. test),
# set ETCD_INITIAL_CLUSTER value for this name, i.e. "test=http://..."
ETCD_INITIAL_CLUSTER="etcd-node1=https://172.18.53.221:2380,etcd-node2=https://172.18.53.223:2380,etcd-node3=https://172.18.53.224:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="k8s-etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="https://172.18.53.221:2379"
#[security]
CLIENT_CERT_AUTH="true"
ETCD_CA_FILE="/etc/kubernetes/ssl/ca.pem"
ETCD_CERT_FILE="/etc/kubernetes/ssl/etcd.pem"
ETCD_KEY_FILE="/etc/kubernetes/ssl/etcd-key.pem"
PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_CA_FILE="/etc/kubernetes/ssl/ca.pem"
ETCD_PEER_CERT_FILE="/etc/kubernetes/ssl/etcd.pem"
ETCD_PEER_KEY_FILE="/etc/kubernetes/ssl/etcd-key.pem"
```

再在服务文件中将配置加上去
```
[root@master ssl]# cat /etc/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target

[Service]
Type=simple
WorkingDirectory=/var/lib/etcd
EnvironmentFile=-/etc/kubernetes/cfg/etcd.conf
# set GOMAXPROCS to number of processors
ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /etc/kubernetes/bin/etcd"
Type=notify

[Install]
WantedBy=multi-user.target
[root@master ssl]# 
```

### 5.2 配置方法2
注意：除了`initial-cluster`不改变外，剩下的其他地方涉及到了ip地址的，都需要替换成相应node的ip地址
```
[root@master ssl]# cat /usr/lib/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos
[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
ExecStart=/etc/kubernetes/bin/etcd \
--name=etcd-node1 \
--cert-file=/etc/kubernetes/ssl/etcd.pem \
--key-file=/etc/kubernetes/ssl/etcd-key.pem \
--peer-cert-file=/etc/kubernetes/ssl/etcd.pem \
--peer-key-file=/etc/kubernetes/ssl/etcd-key.pem \
--trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
--peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
--initial-advertise-peer-urls=https://172.18.53.221:2380 \
--listen-peer-urls=https://172.18.53.221:2380 \
--listen-client-urls=https://172.18.53.221:2379,https://127.0.0.1:2379,http://127.0.0.1:2379 \
--advertise-client-urls=https://172.18.53.221:2379 \
--initial-cluster-token=etcd-cluster-0 \
--initial-cluster="etcd-node1=https://172.18.53.221:2380,etcd-node2=https://172.18.53.223:2380,etcd-node3=https://172.18.53.224:2380" \
--initial-cluster-state=new \
--data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
```

## 6.配置完后启动etcd
注意，配置完第一个先不着急启动，等其他两个节点的配置完成后，三个节点同时启动etcd，否则第一个节点先启动时会一直夯在那儿。
```
[root@master ssl]# systemctl daemon-reload
[[root@master ssl]# systemctl enable etcd

复制相应的配置信息到其他节点，记住一定要修改后才能用！
[[root@master ssl]# scp /etc/kubernetes/cfg/etcd.conf node1:/etc/kubernetes/cfg/
[[root@master ssl]# scp /etc/systemd/system/etcd.service node1:/etc/systemd/system/
[[root@master ssl]# scp /etc/kubernetes/cfg/etcd.conf node2:/etc/kubernetes/cfg/
[[root@master ssl]# scp /etc/systemd/system/etcd.service node2s:/etc/systemd/system/

在所有节点上创建etcd存储目录并启动etcd
[root@master ssl]# mkdir /var/lib/etcd
[root@master ssl]# systemctl start etcd
[root@master ssl]# systemctl status etcd
```

## 7.查看状态
### master(etcd-node1)的状态：
```
[root@master etcd]# systemctl status etcd
● etcd.service - Etcd Server
   Loaded: loaded (/etc/systemd/system/etcd.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2018-09-20 11:16:26 CST; 9s ago
 Main PID: 1311 (etcd)
   CGroup: /system.slice/etcd.service
           └─1311 /etc/kubernetes/bin/etcd

Sep 20 11:16:26 master etcd[1311]: set the initial cluster version to 3.0
Sep 20 11:16:26 master etcd[1311]: enabled capabilities for version 3.0
Sep 20 11:16:26 master etcd[1311]: updated the cluster version from 3.0 to 3.3
Sep 20 11:16:26 master etcd[1311]: enabled capabilities for version 3.3
Sep 20 11:16:26 master etcd[1311]: published {Name:etcd-node1 ClientURLs:[https://172.18.53.221:2379]} to cluster b4fed7ac46f8b765
Sep 20 11:16:26 master etcd[1311]: ready to serve client requests
Sep 20 11:16:26 master systemd[1]: Started Etcd Server.
Sep 20 11:16:26 master etcd[1311]: serving client requests on 127.0.0.1:2379
Sep 20 11:16:26 master etcd[1311]: ready to serve client requests
Sep 20 11:16:26 master etcd[1311]: serving client requests on 172.18.53.221:2379
```

### node1(etcd-node2)的状态：
```
[root@node1 ~]# systemctl status etcd
● etcd.service - Etcd Server
   Loaded: loaded (/etc/systemd/system/etcd.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2018-09-20 11:16:22 CST; 28s ago
 Main PID: 24275 (etcd)
   CGroup: /system.slice/etcd.service
           └─24275 /etc/kubernetes/bin/etcd

Sep 20 11:16:27 node1 etcd[24275]: 68bdac33fe7fb4f1 received MsgVoteResp from 68bdac33fe7fb4f1 at term 188
Sep 20 11:16:27 node1 etcd[24275]: 68bdac33fe7fb4f1 [logterm: 2, index: 7] sent MsgVote request to 329981bcd604a91f at term 188
Sep 20 11:16:27 node1 etcd[24275]: 68bdac33fe7fb4f1 [logterm: 2, index: 7] sent MsgVote request to e6faa5d2f5954b4e at term 188
Sep 20 11:16:27 node1 etcd[24275]: 68bdac33fe7fb4f1 received MsgVoteResp from 329981bcd604a91f at term 188
Sep 20 11:16:27 node1 etcd[24275]: 68bdac33fe7fb4f1 [quorum:2] has received 2 MsgVoteResp votes and 0 vote rejections
Sep 20 11:16:27 node1 etcd[24275]: 68bdac33fe7fb4f1 became leader at term 188
Sep 20 11:16:27 node1 etcd[24275]: raft.node: 68bdac33fe7fb4f1 elected leader 68bdac33fe7fb4f1 at term 188
Sep 20 11:16:27 node1 etcd[24275]: updating the cluster version from 3.0 to 3.3
Sep 20 11:16:27 node1 etcd[24275]: updated the cluster version from 3.0 to 3.3
Sep 20 11:16:27 node1 etcd[24275]: enabled capabilities for version 3.3
```

### node2(etcd-node3)的状态：
```
[root@node2 ~]# systemctl status etcd
● etcd.service - Etcd Server
   Loaded: loaded (/etc/systemd/system/etcd.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2018-09-20 11:16:22 CST; 1min 0s ago
 Main PID: 24324 (etcd)
   CGroup: /system.slice/etcd.service
           └─24324 /etc/kubernetes/bin/etcd

Sep 20 11:16:25 node2 etcd[24324]: health check for peer e6faa5d2f5954b4e could not connect: dial tcp 172.18.53.221:2380: connect: connection refused
Sep 20 11:16:26 node2 etcd[24324]: 329981bcd604a91f [term: 186] received a MsgVote message with higher term from e6faa5d2f5954b4e [term: 187]
Sep 20 11:16:26 node2 etcd[24324]: 329981bcd604a91f became follower at term 187
Sep 20 11:16:26 node2 etcd[24324]: 329981bcd604a91f [logterm: 2, index: 7, vote: 0] rejected MsgVote from e6faa5d2f5954b4e [logterm: 1, index: 3] at term 187
Sep 20 11:16:27 node2 etcd[24324]: 329981bcd604a91f [term: 187] received a MsgVote message with higher term from 68bdac33fe7fb4f1 [term: 188]
Sep 20 11:16:27 node2 etcd[24324]: 329981bcd604a91f became follower at term 188
Sep 20 11:16:27 node2 etcd[24324]: 329981bcd604a91f [logterm: 2, index: 7, vote: 0] cast MsgVote for 68bdac33fe7fb4f1 [logterm: 2, index: 7] at term 188
Sep 20 11:16:27 node2 etcd[24324]: raft.node: 329981bcd604a91f elected leader 68bdac33fe7fb4f1 at term 188
Sep 20 11:16:27 node2 etcd[24324]: updated the cluster version from 3.0 to 3.3
Sep 20 11:16:27 node2 etcd[24324]: enabled capabilities for version 3.3
```

## 8.用命令查看集群的状态是否正常
```
[root@master etcd]# etcdctl --endpoints=https://172.18.53.221:2379,https://172.18.53.223:2379,https://172.18.53.224:2379 \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/kubernetes/ssl/etcd.pem \
  --key-file=/etc/kubernetes/ssl/etcd-key.pem cluster-health
member 329981bcd604a91f is healthy: got healthy result from https://172.18.53.224:2379
member 68bdac33fe7fb4f1 is healthy: got healthy result from https://172.18.53.223:2379
member e6faa5d2f5954b4e is healthy: got healthy result from https://172.18.53.221:2379
cluster is healthy
``````