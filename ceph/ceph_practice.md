# Ceph 存储集群构建实践
本文档简述了一个三节点副本的ceph存储集群的搭建过程。

其中三个物理主机上分别使用一块SATA硬盘作为osd。

网络使用管理网段172.16.xx.xx。

## 基础环境准备
### 主机名设置
```
# 分别设置主机名称
[root@vp-01 ~]# hostnamectl set-hostname xxx
```
### 配置网络
```
[root@vp-01 ~]# zs-network-setting -b 172.20.12.154 255.255.0.0 172.20.0.1 eth0
```
一般Ceph的网络不和管理网段和业务网段相同，会单独划分出一个存储网络
### 设置主机解析
```
[root@vp-01 ~]# vim /etc/hosts
···
172.16.17.21	vp-01
172.16.17.22	vp-02
172.16.17.23	vp-03
···
```
### 密钥配对，免密访问
```
# 在管理节点上生成密钥
[root@vp-01 ~]# ssh-keygen -t rsa

# 在Ceph集群的每个节点上都执行拷贝动作
[root@vp-01 ~]# ssh-copy-id vp-01
···
```
### 传输/etc/hosts文件
```
# 将hosts文件拷贝到集群的每个节点上
[root@vp-01 ~]# scp /etc/hosts vp-01:/etc/
···
```
### 防火墙配置
```
# 在Ceph集群的每台节点上都执行
[root@vp-01 ~]# iptables -I INPUT -s 172.16.0.0/16 -j ACCEPT && service iptables save

```
### 配置时间同步
```
# 使用zstack-local repo
[root@vp-01 ~]# yum --disablerepo=* --enablerepo=zstack-local,ceph-hammer install ntp
[root@vp-01 ~]# systemctl restart ntpd
[root@vp-01 ~]# systemctl enable ntpd.service
```
## 磁盘初始化
通常情况下，使用一整个磁盘作为ceph的一个osd
### 机械硬盘格式化操作
```
[root@vp-01 ~]# mkfs.xfs -f -i size=512 -l size=128m,lazy-count=1 /dev/sdb
[root@vp-01 ~]# mkdir -p /data/disk1/
```
查看机械硬盘UUID

```
[root@vp-01 ~]# ll /dev/disk/by-uuid/
total 0
lrwxrwxrwx 1 root root 10 Nov 24 17:22 26eac1c5-2d61-4ddb-b59d-7e4347c22975 -> ../../sda1
lrwxrwxrwx 1 root root  9 Jan 18 10:11 c26175f8-8b15-4710-8481-6a386fbc9e33 -> ../../sdb
lrwxrwxrwx 1 root root 10 Nov 24 12:37 de8cab44-3ce0-4263-b8ae-584340255e0a -> ../../sda2
lrwxrwxrwx 1 root root 10 Nov 24 12:37 ec38e293-3ac5-4544-bc71-6408cb2f8c40 -> ../../sda3
```
通过UUID挂载数据目录

```
[root@vp-01 ~]# mount -t xfs -o noatime,nodiratime,nobarrier /dev/disk/by-uuid/c26175f8-8b15-4710-8481-6a386fbc9e33 /data/disk1
```
写入系统启动

```
[root@vp-01 ~]# chmod +x /etc/rc.local
[root@vp-01 ~]# vim /etc/rc.local
...
```
## 安装Ceph
### 安装Ceph Hammer版本
管理节点安装ceph以及ceph-deploy，其他节点仅安装ceph即可

```
[root@vp-01 ~]# yum --disablerepo=* --enablerepo=zstack-local,ceph-hammer install ceph ceph-deploy -y

[root@vp-02 ~]# yum --disablerepo=* --enablerepo=zstack-local,ceph-hammer install ceph -y
```
### Ceph创建配置集群
```
# 创建config目录
[root@vp-01 ~]# mkdir ceph-config
[root@vp-01 ~]# cd ceph-config

# 新建ceph集群
[root@vp-01 ~]# ceph-deploy new vp-01 vp-02 vp-03

#修改ceph.conf文件
[root@vp-01 ~]# vim ceph.conf
...
[global]
fsid = cd828b80-7b9a-4ee0-9ed4-35f9aa634039
mon_initial_members = vp-01, vp-02, vp-03
mon_host = 172.16.17.21,172.16.17.22,172.16.17.23
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx

public_network = 10.60.250.0/24
cluster_network = 10.60.250.0/24
filestore_xattr_use_omap = true

osd_pool_default_size = 2
osd_pool_default_min_size = 1
osd_pool_default_pg_num = 4096
osd_pool_default_pgp_num = 4096

osd_max_backfills = 1
osd_recovery_max_active = 1
osd_crush_update_on_start = 0
debug_ms = 0
debug_osd = 0

osd_recovery_max_single_start = 1
filestore_max_sync_interval = 15
filestore_min_sync_interval = 10
filestore_queue_max_ops = 65536
filestore_queue_max_bytes = 536870912
filestore_queue_committing_max_bytes = 536870912
filestore_queue_committing_max_ops = 65536

filestore_wbthrottle_xfs_bytes_start_flusher = 419430400
filestore_wbthrottle_xfs_bytes_hard_limit = 4194304000
filestore_wbthrottle_xfs_ios_start_flusher = 5000
filestore_wbthrottle_xfs_ios_hard_limit = 50000
filestore_wbthrottle_xfs_inodes_start_flusher = 5000
filestore_wbthrottle_xfs_inodes_hard_limit = 50000

journal_max_write_bytes = 1073714824
journal_max_write_entries = 5000
journal_queue_max_ops = 65536
journal_queue_max_bytes = 536870912
osd_client_message_cap = 65536
osd_client_message_size_cap = 524288000
ms_dispatch_throttle_bytes = 536870912
filestore_fd_cache_size = 4096
osd_op_threads = 10
osd_disk_threads = 2
filestore_op_threads = 6
osd_client_op_priority = 100
osd_recovery_op_priority = 5

# 配置文件下发至集群每个节点并收集keys
[root@vp-01 ~]# ceph-deploy --overwrite-conf mon create vp-01 vp-02 vp-03
[root@vp-01 ~]# ceph-deploy --overwrite-conf config push vp-01 vp-02 vp-03
[root@vp-01 ~]# ceph-deploy gatherkeys vp-01 vp-02 vp-03

#创建osd
#数据和日志在一个盘上
[root@vp-01 ~]# ceph-deploy --overwrite-conf osd create vp-01:/data/disk1 vp-02:/data/disk1 vp-03:/data/disk1
#数据和日志在两个盘上
[root@vp-01 ~]# ceph-deploy --overwrite-conf osd create vp-01:/data/disk1:/dev/disk/by-partlabel/journal-1

# 激活osd，更新配置
[root@vp-01 ~]# ceph-deploy --overwrite-conf osd activate vp-01:/data/disk1 vp-02:/data/disk1 vp-03:/data/disk1
[root@vp-01 ~]# ceph-deploy --overwrite-conf config push vp-01 vp-02 vp-03

#查看集群初始化后的状态
[root@vp-01 ~]# ceph -s
[root@vp-01 ~]# ceph osd tree

#建立CRUSH结构树
[root@vp-01 ~]# ceph osd crush add-bucket vp-01 host
[root@vp-01 ~]# ceph osd crush add-bucket vp-02 host
[root@vp-01 ~]# ceph osd crush add-bucket vp-03 host

[root@vp-01 ~]# ceph osd crush move vp-01 root=default
[root@vp-01 ~]# ceph osd crush move vp-02 root=default
[root@vp-01 ~]# ceph osd crush move vp-03 root=default

[root@vp-01 ~]# ceph osd crush create-or-move osd.0 0.4 root=default host=vp-01
[root@vp-01 ~]# ceph osd crush create-or-move osd.1 0.4 root=default host=vp-02
[root@vp-01 ~]# ceph osd crush create-or-move osd.2 0.4 root=default host=vp-03

# 再次查看ceph集群以及osd的状态
[root@vp-01 ~]# ceph -s
    cluster cd828b80-7b9a-4ee0-9ed4-35f9aa634039
     health HEALTH_WARN
            too many PGs per OSD (464 > max 300)
     monmap e1: 3 mons at {vp-01=172.16.17.21:6789/0,vp-02=172.16.17.22:6789/0,vp-03=172.16.17.23:6789/0}
            election epoch 4, quorum 0,1,2 vp-01,vp-02,vp-03
     osdmap e28: 3 osds: 3 up, 3 in
      pgmap v18702: 464 pgs, 5 pools, 6314 MB data, 1599 objects
            34432 MB used, 1641 GB / 1674 GB avail
                 464 active+clean
  client io 2183 B/s rd, 873 B/s wr, 5 op/s
  
[root@vp-01 ~]# ceph osd tree
ID WEIGHT  TYPE NAME      UP/DOWN REWEIGHT PRIMARY-AFFINITY
-1 1.19998 root default
-2 0.39999     host vp-01
 0 0.39999         osd.0       up  1.00000          1.00000
-3 0.39999     host vp-02
 1 0.39999         osd.1       up  1.00000          1.00000
-4 0.39999     host vp-03
 2 0.39999         osd.2       up  1.00000          1.00000
```



