# Ceph 扩容
当你有一个集群运行起来，你可能添加新的OSD或在运行时从集群中删除的OSD。

当你想扩展集群，你可以在运行时添加的OSD。使用Ceph的OSD一般是一个Ceph的ceph-osd守护一个存储驱动器的主机内。如果主机有多个存储驱动器，你可以映射为每个驱动器一个ceph-osd的守护进程。

一般来说，检查您的群集能力查看是否达到其容量的上端是一个好主意。当您的集群达到其接近满比率，你应该添加一个或多个OSD来扩大集群的容量。

## 增加OSD
一般一个OSD对应一个硬盘，设置一个ceph-osd守护进程，集群把数据发布到OSD。如果一台主机有多个硬盘，可以重复此过程，并且把每个硬盘配置为一个OSD。

要添加OSD，步骤要依次创建数据目录、把硬盘挂载到目录、把OSD加入集群、然后把它加入CRUSH图。往 CRUSH 图里添加 OSD 时建议设置权重，较新的 OSD 主机拥有更大的空间，即它们可以有更大的权重。

### 1.创建OSD。如果未指定UUID，OSD启动时会自动生成一个。
```
# ceph osd create [{uuid} [{id}]
```
如果指定了可选参数 {id} ，那么它将作为 OSD id 。要注意，如果此数字已使用，此命令会出错。

一般来说，我们不建议指定 {id} 。因为 ID 是按照数组分配的，跳过一些依然会浪费内存；尤其是跳过太多、或者集群很大时，会更明显。若未指定 {id} ，将用最小可用数字。
### 2.在新OSD主机上创建默认目录或者挂载目录
```
创建目录
# mkdir /var/lib/ceph/osd/ceph-{osd-number}

挂载目录并写入开机启动
# mkfs.xfs -f -i size=512 -l size=128m,lazy-count=1 /dev/sdd
# mkdir -p /data/disk3/
# mount -t xfs -o noatime,nodiratime,nobarrier /dev/disk/by-uuid/29d11ed5-e35e-4b15-b581-cffc49d31017 /data/disk3/
```
### 3.初始化OSD数据目录
```
# ceph-osd -i {osd-num} --mkfs --mkkey
```
运行 ceph-osd 时目录必须是空的。
### 4.注册 OSD 认证密钥
```
# ceph auth add osd.{osd-num} osd 'allow *' mon 'allow rwx' -i /var/lib/ceph/osd/ceph-{osd-num}/keyring
```
ceph-{osd-num} 路径里的 ceph 值应该是 $cluster-$id ，如果你的集群名字不是 ceph ，那就用改过的名字。
### 5.把 OSD 加入 CRUSH 图
```
# ceph osd crush add {id-or-name} {weight}  [{bucket-type}={bucket-name} ...]
```
把 OSD 加入 CRUSH 分级结构的合适位置。如果你指定了不止一个桶，此命令会把它加入你所指定的桶中最具体的一个，并且把此桶挪到你指定的其它桶之内。重要：如果你只指定了 root 桶，此命令会把 OSD 直接挂到 root 下面，但是 CRUSH 规则期望它位于主机内。

### 案例
快速增加vp-02节点的/date/disk3/目录为osd，执行以下步骤：

```
进入ceph-config目录
# cd ceph-config/

创建osd
# ceph-deploy --overwrite-conf osd create vp-02:/data/disk3/

激活osd
# ceph-deploy --overwrite-conf osd activate vp-02:/data/disk3/

更新配置至所有节点
# ceph-deploy --overwrite-conf config push vp-01 vp-02 vp-03

修改crush规则配置
# ceph osd crush add osd.5 0.4 host=vp-02
```
**把新OSD加入CRUSH图后，Ceph会重新均衡服务器，一些归置组会迁移到新OSD里，你可以用ceph -s命令观察此过程，直到集群状态变为HEALTH_OK即可**

注意：1. 目录必须为纯净目录 2.这个目录一般情况下挂载了一整块硬盘

## 删除OSD
要想缩减集群尺寸或替换硬件，可在运行时删除OSD。在Ceph里，一个OSD通常是一台主机上的一个ceph-osd守护进程、它运行在一个硬盘之上。如果一台主机上有多个数据盘，你得挨个删除其对应ceph-osd。通常，操作前应该检查集群容量，看是否快达到上限了，确保删除OSD后不会使集群达到near full比率。
### 1. 移出OSD集群
```
# ceph osd out {osd-num}
```
删除OSD前，它通常是up且in的，要先把它踢出集群，以使Ceph启动重新均衡、把数据拷贝到其他OSD。
### 2. 观察数据迁移
```
# ceph -s
```
一旦把OSD踢出 (out)集群，Ceph就会开始重新均衡集群、把归置组迁出将删除的OSD。你可以用ceph工具观察此过程。

有时候,（通常是只有几台主机的“小”集群，比如小型测试集群）拿出（out）某个OSD可能会使CRUSH进入临界状态，这时某些PG一直卡在active+remapped状态。如果遇到了这种情况，你应该把此OSD标记为 in，用这个命令：

```
# ceph osd in {osd-num}
```

等回到最初的状态后，把它的权重设置为0，而不是标记为out，用此命令：

```
# ceph osd crush reweight osd.{osd-num} 0
```

执行后，你可以观察数据迁移过程，应该可以正常结束。把某一OSD标记为out和权重改为0的区别在于，前者，包含此OSD的桶、其权重没变；而后一种情况下，桶的权重变了（降低了此OSD的权重）。某些情况下，reweight命令更适合“小”集群。
### 3.停止OSD
```
# /etc/init.d/ceph stop osd.{osd-num}
```
把 OSD 踢出集群后，它可能仍在运行，就是说其状态为 up 且 out 。删除前要先停止 OSD 进程。
### 4.删除OSD
此步骤依次把一个 OSD 移出集群 CRUSH 图、删除认证密钥、删除 OSD 图条目、删除 ceph.conf 条目。如果主机有多个硬盘，每个硬盘对应的 OSD 都得重复此步骤。

```
删除 CRUSH 图的对应 OSD 条目
# ceph osd crush remove {name}

删除 OSD 认证密钥
# ceph auth del osd.{osd-num}

删除 OSD
# ceph osd rm {osd-num}

登录到保存ceph.conf主拷贝的主机，从ceph.conf配置文件里删除对应条目，把更新过的 ceph.conf 拷贝到集群其他主机的 /etc/ceph 目录下
```
### 案例
例如，需要移除故障的数据盘 OSD.5，执行以下步骤:

```
# ceph osd out 5

登陆到osd.5的服务器，关闭osd服务
# service ceph stop osd.5

移出crush图
# ceph osd crush remove osd.5

删除认证密钥
# ceph auth del osd.5

删除osd
# ceph osd rm 5

进入ceph-config目录，更新配置至所有节点
#  ceph-deploy --overwrite-conf config push vp-01 vp-02 vp-03
```