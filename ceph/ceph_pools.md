# Ceph Pools 池

## 概览
第一次部署ceph集群的时候如果没有创建一个存储池，ceph会使用默认的存储池来存储数据。

* 复制：可以设置一个对象期望的副本数量。典型配置为size=2，但是也可以更改这个数量。
* 配置组：可以设置一个存储池的配置组数量。典型配置为每个osd上使用大约100个归置组。
* CRUSH规则：当在存储池存储数据的时候，映射到存储池的CRUSH规则集使得CRUSH确定一条规则，用于集群内主对象归置和其副本的复制。存储池可以定制CRUSH规则。
* 快照：创建快照的时候，实际上创建了一小部分存储池的快照。
* 所有者：可以设置一个用户ID为一个存储池的所有者。

## 相关操作
### 查看池
```
# ceph osd lspools
0 rbd,1 bak-t-5e80315aa9724e91adaac915ad201f4e,2 pri-c-8d38cb388fbd45feaa9dcffd3a4b04e8,3 pri-v-r-8d38cb388fbd45feaa9dcffd3a4b04e8,4 pri-v-d-8d38cb388fbd45feaa9dcffd3a4b04e8,

# rados lspools
rbd
bak-t-5e80315aa9724e91adaac915ad201f4e
pri-c-8d38cb388fbd45feaa9dcffd3a4b04e8
pri-v-r-8d38cb388fbd45feaa9dcffd3a4b04e8
pri-v-d-8d38cb388fbd45feaa9dcffd3a4b04e8

# ceph osd dump | grep pool
pool 0 'rbd' replicated size 3 min_size 2 crush_ruleset 0 object_hash rjenkins pg_num 64 pgp_num 64 last_change 1 flags hashpspool stripe_width 0
pool 1 'bak-t-5e80315aa9724e91adaac915ad201f4e' replicated size 3 min_size 2 crush_ruleset 0 object_hash rjenkins pg_num 100 pgp_num 100 last_change 40 flags hashpspool stripe_width 0
pool 2 'pri-c-8d38cb388fbd45feaa9dcffd3a4b04e8' replicated size 3 min_size 2 crush_ruleset 0 object_hash rjenkins pg_num 100 pgp_num 100 last_change 34 flags hashpspool stripe_width 0
pool 3 'pri-v-r-8d38cb388fbd45feaa9dcffd3a4b04e8' replicated size 3 min_size 2 crush_ruleset 0 object_hash rjenkins pg_num 100 pgp_num 100 last_change 39 flags hashpspool stripe_width 0
pool 4 'pri-v-d-8d38cb388fbd45feaa9dcffd3a4b04e8' replicated size 3 min_size 2 crush_ruleset 0 object_hash rjenkins pg_num 100 pgp_num 100 last_change 36 flags hashpspool stripe_width 0

```
查看池的操作有几种，dump出的信息最详尽，包括pool ID、副本数量、CRUSH规则集以及PG等数量信息等。

默认的存储池包括数据、元数据、rbd。

### 创建池
```
# ceph osd pool create {pool-name} {pg-num} [{pgp-num}]
例如
# ceph osd pool create test-pool 128
```
官方推荐pg_num数量：

若少于5个OSD，设置pg_num为128。

5~10个OSD，设置pg_num为512。

10~50个OSD，设置pg_num为4096。

超过50个OSD，可以参考pgcalc计算。
### 删除池
```
# ceph osd pool delete {pool-name} {pool-name} --yes-i-really-really-mean-it
例如
# ceph osd pool delete test-pool test-pool --yes-i-really-really-mean-it
```
删除一个pool会同时清空pool的所有数据，因此非常危险。(和rm -rf /类似）。因此删除pool时ceph要求必须输入两次pool名称，同时加上--yes-i-really-really-mean-it选项。
### 重命名池
```
# ceph osd pool rename {current-pool-name} {new-pool-name}
例如
# ceph osd pool rename test-pool new-test-pool
```
将test-pool重命名为new-test-pool
### 查看池状态
```
# rados df
```
### 配置池
```
设置池的值
# ceph osd pool set {pool-name} {key} {value}
```
可以设置的值如下:size、min_size、crash_replay_interval、pgp_num、crush_ruleset等。

```
# ceph osd pool get {pool-name} {key}
```
### 快照
ceph支持对整个pool创建快照，作用于这个pool的所有对象。

```
创建快照
# ceph osd pool mksnap {pool-name} {snap-name}
或者
# rados mksnap {snap-name} -p {pool-name}

查询快照
# rados lssnap -p {pool-name}

删除快照
# ceph osd pool rmsnap {pool-name} {snap-name}
或者
# rados rmsnap {snap-name} -p {pool-name}
```
删除pool会同步删除所有快照。在删除pool后，需要删除pool的CRUSH规则集，假如你手工创建过它们。
***
### rados指令相关
#### 查看pool
```
# rados lspools
```
#### 查看pool详细容量以及利用情况
```
# rados df
```
#### 创建pool
```
创建名为test的pool
# rados mkpool test
```
#### 查看ceph pool中的object（object是以块形式存储的）
```
查看名为test的pool中的object
# rados ls -p test | more
```
#### 创建/删除对象object
```
# rados create test-object -p test

# rados rm test-object -p test
```
***
### rbd指令相关
#### 查看ceph一个pool里的所有镜像
```
列出名为test的pool里面的所有镜像，其结果为UUID
# rbd ls test
```
#### 查看具体某一个镜像的信息
```
查看名为test的池中的镜像 e231dc91b4ca4744894b3230b4f34a0b 信息
# rbd info -p test --image e231dc91b4ca4744894b3230b4f34a0b
rbd image 'e231dc91b4ca4744894b3230b4f34a0b':	size 20481 MB in 5121 objects	order 22 (4096 kB objects)	block_name_prefix: rbd_data.159503d1b58ba	format: 2	features: layering	flags: 
```
#### 创建镜像
```
在test池中创建一个命名为ceshi的10000M的镜像
# rbd create -p test --size 10000 ceshi
```
#### 删除镜像
```
删除test池中名为ceshi的镜像
# rbd rm  -p test ceshi
```
#### 调整镜像尺寸
```
#  rbd resize -p test --size 20000 ceshi
```
#### 导出镜像
```
将test池中uuid为e231dc91b4ca4744894b3230b4f34a0b的镜像导出
# rbd export -p test --image e231dc91b4ca4744894b3230b4f34a0b /root/aaa.img
```
#### 导入镜像
```
# rbd import /root/aaa.img -p test
```