# Ceph CRUSH
## 简介
CRUSH全称Controlled Replication Under Scalable Hashing，是一种数据分发算法，类似于哈希和一致性哈希。哈希的问题在于数据增长时不能动态加Bucket，一致性哈希的问题在于加Bucket时数据迁移量比较大，其他数据分发算法依赖中心的Metadata服务器来存储元数据效率较低，CRUSH则是通过计算、接受多维参数的来解决动态数据分发的场景。

CRUSH有两个关键优点：

* 任何组件都可以独立计算出每个object所在的位置(去中心化)。
* 只需要很少的元数据(cluster map)，只要当删除添加设备时，这些元数据才需要改变。

## 算法基础
CRUSH算法通过每个设备的权重来计算数据对象的分布。对象分布是由cluster map和data distribution policy决定的。cluster map描述了可用存储资源和层级结构(比如有多少个机架，每个机架上有多少个服务器，每个服务器上有多少个磁盘)。data distribution policy由placement rules组成。rule决定了每个数据对象有多少个副本，这些副本存储的限制条件(比如3个副本放在不同的机架中)。

### 1.层级的Cluster Map
Cluster Map由device和bucket组成，它们都有权重和id。Bucket可以包含任意数量item。item可以都是的devices或者都是buckets。管理员控制存储设备的权重。权重和存储设备的容量有关。Bucket的权重被定义为它所包含所有item的权重之和。
### 2.副本分布
副本在存储设备上的分布影响数据的安全。cluster map反应了存储系统的物理结构。CRUSH placement policies决定把对象副本分布在不同的区域(某个区域发生故障时并不会影响其他区域)。每个rule包含一系列操作(用在层级结构上)。

* tack(a) ：选择一个item，一般是bucket，并返回bucket所包含的所有item。这些item是后续操作的参数，这些item组成向量i。
* select(n, t)：迭代操作每个item(向量i中的item)，对于每个item(向量i中的item)向下遍历(遍历这个item所包含的item)，都返回n个不同的item(type为t的item)，并把这些item都放到向量i中。select函数会调用c(r, x)函数，这个函数会在每个bucket中伪随机选择一个item。
* emit：把向量i放到result中。

存储设备有一个确定的类型。每个bucket都有type属性值，用于区分不同的bucket类型(比如”row”、”rack”、”host”等，type可以自定义)。rules可以包含多个take和emit语句块，这样就允许从不同的存储池中选择副本的storage target。
### 3.冲突、故障、超载
select(n, t)操作会循环选择第 r=1,…,n 个副本，r作为选择参数。在这个过程中，假如选择到的item遇到三种情况(冲突，故障，超载)时，CRUSH会拒绝选择这个item，并使用r'(r’和r、出错次数、firstn参数有关)作为选择参数重新选择item。

* 冲突：这个item已经在向量i中，已被选择。
* 故障：设备发生故障，不能被选择。
* 超载：设备使用容量超过警戒线，没有剩余空间保存数据对象。

故障设备和超载设备会在cluster map上标记(还留在系统中)，这样避免了不必要的数据迁移。
### 4.改变以及数据迁移
当添加移除存储设备，或有存储设备发生故障时(cluster map发生改变时)，存储系统中的数据会发生迁移。好的数据分布算法可以最小化数据迁移大小。

### 5.Bucket分类
CRUSH定义了四种具有不同算法的的buckets。每种bucket基于不同的数据结构，并有不同的c(r,x)伪随机选择函数。

* Uniform Buckets：适用于具有相同权重的item，而且bucket很少添加删除item。它的查找速度是最快的。
* List Buckets：它的结构是链表结构，所包含的item可以具有任意的权重。CRUSH从表头开始查找副本的位置，它先得到表头item的权重Wh、剩余链表中所有item的权重之和Ws，然后根据hash(x, r, item)得到一个[0~1]的值v，假如这个值v在[0~Wh/Ws)之中，则副本在表头item中，并返回表头item的id。否者继续遍历剩余的链表。
* Tree Buckets：链表的查找复杂度是O(n)，决策树的查找复杂度是O(log n)。item是决策树的叶子节点，决策树中的其他节点知道它左右子树的权重，节点的权重等于左右子树的权重之和。CRUSH从root节点开始查找副本的位置，它先得到节点的左子树的权重Wl，得到节点的权重Wn，然后根据hash(x, r, node_id)得到一个[0~1]的值v，假如这个值v在[0~Wl/Wn)中，则副本在左子树中，否者在右子树中。继续遍历节点，直到到达叶子节点。Tree Bucket的关键是当添加删除叶子节点时，决策树中的其他节点的node_id不变。决策树中节点的node_id的标识是根据对二叉树的中序遍历来决定的(node_id不等于item的id，也不等于节点的权重)。
* Straw Buckets：这种类型让bucket所包含的所有item公平的竞争(不像list和tree一样需要遍历)。这种算法就像抽签一样，所有的item都有机会被抽中(只有最长的签才能被抽中)。每个签的长度是由length = f(Wi)*hash(x, r, i) 决定的，f(Wi)和item的权重有关，i是item的id号。c(r, x) = MAXi(f(Wi) * hash(x, r, i))。

## CRUSH映射
### 标准操作
#### 1.获取CRUSH映射
```
# ceph osd getcrushmap -o {compiled-crushmap-filename}
```
#### 2.反编译CRUSH映射
```
# crushtool -d {compiled-crushmap-filename} -o {decompiled-crushmap-filename}
```
#### 3.编译CRUSH映射
```
# crushtool -c {decompiled-crushmap-filename} -o {compiled-crushmap-filename}
```
#### 4.设置CRUSH映射
```
# ceph osd setcrushmap -i {compiled-crushmap-filename}
```
### 示例
```
获取crushmap
# ceph osd getcrushmap -o crushmap.original
# crushtool -d crushmap.original -o crushmap

查看crushmap
# cat crushmap
# begin crush map
tunable choose_local_tries 0
tunable choose_local_fallback_tries 0
tunable choose_total_tries 50
tunable chooseleaf_descend_once 1
tunable straw_calc_version 1

# devices
device 0 osd.0
device 1 osd.1
device 2 osd.2
device 3 osd.3
device 4 osd.4

# types
type 0 osd
type 1 host
type 2 chassis
type 3 rack
type 4 row
type 5 pdu
type 6 pod
type 7 room
type 8 datacenter
type 9 region
type 10 root

# buckets
host vp-01 {
	id -2		# do not change unnecessarily
	# weight 0.400
	alg straw
	hash 0	# rjenkins1
	item osd.0 weight 0.400
}
host vp-02 {
	id -3		# do not change unnecessarily
	# weight 0.800
	alg straw
	hash 0	# rjenkins1
	item osd.1 weight 0.400
	item osd.2 weight 0.400
}
host vp-03 {
	id -4		# do not change unnecessarily
	# weight 0.800
	alg straw
	hash 0	# rjenkins1
	item osd.3 weight 0.400
	item osd.4 weight 0.400
}
root default {
	id -1		# do not change unnecessarily
	# weight 2.000
	alg straw
	hash 0	# rjenkins1
	item vp-01 weight 0.400
	item vp-02 weight 0.800
	item vp-03 weight 0.800
}

# rules
rule replicated_ruleset {
	ruleset 0
	type replicated
	min_size 1
	max_size 10
	step take default
	step chooseleaf firstn 0 type host
	step emit
}

# end crush map
```
### 参数详情
#### 1.devices
devices由任何对象存储设备组成–比如, 即存储驱动器对应到一个ceph-osd守护进程。不管新增还是删除OSD，这个列表会自动更新。通常你无需更改此处，ceph会自动维护。

```
#设备
device {num} {osd.name}

device 0 osd.0 
device 1 osd.1
...
```
#### 2.types
types定义了在你的crush层次中使用的bucket的种类。包括root、datacenter、room、row、rack、host、osd等。默认的bucket类型对大部分ceph集群来说够用了，不过你也可以增加自己的类型。

```
#类型 type {num} {bucket-name}

type 0 osd 
type 1 host 
type 2 rack
...
```
#### 3.buckets
定义bucket的层次性架构，也可以定义bucket所使用的算法类型。crush算法分配存储设备之间的数据对象，根据每个设备的weight值，近似均匀的概率分布。CRUSH根据你定义分层集群映射分配对象及其复制品。crush映射代表可用的存储设备和包含它们逻辑元素。

声明bucket实例时，您必须指定其类型，给它一个唯一的名字（字符串），给它分配一个唯一的ID表示为一个负整数（可选），指定weight相对的能力，其项目总容量，指定bucket算法（通常是 straw),以及哈希值（通常为0,反映哈希算法 rjenkins1). 一个bucket可具有一个或多个项目.项目可能包括节点的分支或分支片。项目可能有一个weight用来反映该项目的相对weight。

```
# [bucket-type] [bucket-name] {
id [a unique negative numeric ID]
weight [the relative capacity/capability of the item(s)]
alg [the bucket type: uniform | list | tree | straw ]
hash [the hash type: 0 by default]
item [item-name] weight [weight]
}

host vp-01 {
	id -2	# do not change unnecessarily
	# weight 0.400
	alg straw
	hash 0	# rjenkins1
	item osd.0 weight 0.400
	}
	
host vp-02 {
	id -3	# do not change unnecessarily
	# weight 0.800
	alg straw
	hash 0	# rjenkins1
	item osd.1 weight 0.400
	item osd.2 weight 0.400
	}
...
```
#### 4.rules
定义pool里存储的数据应该选择哪个相应的bucket。对较大的集群来说，有多个pool，每个pool有它自己的选择规则。默认有一个规则为每个池分配CRUSH映射，一个规则集分配给每个在默认池。

**注意:在大多数情况下，你将不再需要修改默认的规则。当你创建一个新的池时，它的默认规则集是0**

```
rule <rulename> {
ruleset <ruleset>  #表示分类属于一组规则的规则
type [ replicated | raid4 ] #描述存储驱动器或RAID（副本）的规则
min_size <min-size>
max_size <max-size>
step take <bucket-type>
step [choose|chooseleaf] [firstn|indep] <N> <bucket-type>
step emit
}

rule replicated_ruleset {
	ruleset 0
	type replicated
	min_size 1
	max_size 10
	step take default
	step chooseleaf firstn 0 type host
	step emit
	}
...
```

## 典型案例
Ceph先进的架构，加上SSD固态盘，特别是高速PCIe SSD带来的高性能，无疑将成为Ceph部署的典型场景。

### 1.SSD作为OSD的日志盘
使用快速的SSD作为OSD的日志盘来提高集群性能是ceph最常用的使用场景。
在选择SSD盘时，除了关注IOPS性能外，要重点注意以下3个方面：

1）写密集场景
OSD日志是大量小数据块、随机IO写操作。选购SSD作为日志盘，需要重点关注随机、小块数据、写操作的吞吐量及IOPS；当一块SSD作为多个OSD的日志盘时，因为涉及到多个OSD同时往SSD盘写数据，要重点关注顺序写的带宽。

2）分区对齐
在对SSD分区时，使用Parted进行分区并保证4KB分区对齐，避免分区不当带来的性能下降。

3）O_DIRECT和O_DSYNC写模式及写缓存
由Ceph源码可知(ceph/src/os/FileJournal.cc)，OSD在写日志文件时，使用的flags是：
flags |= O_DIRECT | O_DSYNC
O_DIRECT表示不使用Linux内核Page Cache; O_DSYNC表示数据在写入到磁盘后才返回。
由于磁盘控制器也同样存在缓存，而Linux操作系统不负责管理设备缓存，O_DSYNC在到达磁盘控制器缓存之后会立即返回给调用者，并无法保证数据真正写入到磁盘中，Ceph致力于数据的安全性，对用来作为日志盘的设备，应禁用其写缓存。(# hdparm -W 0 /dev/hda 0)
使用工具测试SSD性能时，应添加对应的flag：dd … oflag=direct,dsync; fio … —direct=1, —sync=1…
### 2.与SAS硬盘混用，组建独立Pool
此种场景是现在主推的一种混用硬盘的方案之一。

基本思路是编辑CRUSH MAP，先标示出散落在各存储服务器的SSD OSD以及硬盘OSD(host元素)，再把这两种类型的OSD聚合起来形成两种不同的数据根(root元素)，然后针对两种不同的数据源分别编写数据存取规则(rule元素)，最后，创建SSD Pool，并指定其数据存取操作都在SSD OSD上进行。

在该场景下，同一个Ceph集群里存在传统机械盘组成的存储池，以及SSD组成的快速存储池，可把对读写性能要求高的数据存放在SSD池，而把备份数据存放在普通存储池。
 
CRUSH规则简单示例如下：

```
# 1. 标示服务器上的SSD与SAS硬盘OSD
# SAS OSD
host ceph-server1-sas {
  id -2
  alg straw
  hash 0
  item osd.0 weight 1.000
  item osd.1 weight 1.000
  …
}

# SSD OSD
host ceph-server1-ssd {
  id -2
  alg straw
  hash 0
  item osd.2 weight 1.000
  …
}

# 2. 聚合OSD，创建数据根
root sas {
  id -1
  alg straw
  hash 0
  item ceph-server1-sas
  …
  item ceph-servern-sas
}
root ssd {
  id -1
  alg straw
  hash 0
  item ceph-server1-ssd
  …
  item ceph-servern-ssd
}

# 3. 创建存取规则
rule sas {
  ruleset 0
  type replicated
  …
  step take sas
  step chooseleaf firstn 0 type host
  step emit
}
rule ssd {
  ruleset 1
  type replicated
  …
  step take ssd
  step chooseleaf firstn 0 type host
  step emit
}

# 4. 编译及使用新的CRUSH MAP
# crushtool -c ssd_sas_map.txt -o ssd_sas_map
# ceph osd setcrushmap -i ssd_sas_map

# 5. 创建Pool并指定存取规则
# ceph osd pool create ssd 4096 4096
# ceph osd pool set ssd crush_ruleset 1
```
### 3.配置CRUSH数据读写规则，使主备数据中的主数据落在SSD的OSD上
独立组建SSD Pool，Pool里的主备数据都是在SSD里，但正常情况下，Ceph客户端直接读写的只有主数据，这对相对昂贵的SSD来说存在一定程度上的浪费。这就引出了此使用场景—配置CRUSH数据读写规则，使主备数据中的主数据落在SSD的OSD上。

SATA/SAS机械盘和SSD混用，但SSD的OSD节点不用来组成独立的存储池，而是配置CURSH读取规则，让所有数据的主备份落在SSD OSD上。Ceph集群内部的数据备份从SSD的主OSD往非SSD的副OSD写数据。
这样，所有的Ceph客户端直接读写的都是SSD OSD 节点，既提高了性能又节约了对OSD容量的要求。

```
# 配置CRUSH读写规则
rule ssd-primary {
  ruleset 1
  …
  step take ssd
  step chooseleaf firstn 1 type host #从SSD根节点下取1个OSD存主数据
  step emit
  step take sas
  step chooseleaf firstn -1 type host #从SAS根节点下取其它OSD节点存副本数据
  step emit
}
```