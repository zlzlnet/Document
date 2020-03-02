# Elasticsearch

## 简介
Elasticsearch 是一个分布式、RESTful 风格的搜索和数据分析引擎，能够解决不断涌现出的各种用例。作为 Elastic Stack 的核心，它集中存储您的数据，帮助您发现意料之中以及意料之外的情况。

官网：
English：https://www.elastic.co/products/elasticsearch
中文：https://www.elastic.co/cn/products/elasticsearch


## 搭建
支持多种安装部署方式，此处介绍使用docker部署elasticsearch

#### 1.下载镜像
```
docker pull docker.elastic.co/elasticsearch/elasticsearch:6.5.4
```
参考地址：
https://hub.docker.com/_/elasticsearch
https://www.docker.elastic.co/

#### 2.启动容器
##### 开发模式
```
docker run -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:6.5.4
```

##### 生产模式
*重要：* 修改系统`vm.max_map_count`参数
在linux系统`/etc/sysctl.conf`文件中添加`vm.max_map_count=262144`
并执行命令
```
sysctl -w vm.max_map_count=262144
```

**使用docker-compose命令启动elasticsearch集群，2节点。**
（docker-compose安装请自行谷歌）
```
docker-compose up -d
```

其中
`docker-compose.yml`文件参考如下：
```
version: '2.2'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:6.5.4
    container_name: elasticsearch
    environment:
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata1:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
    networks:
      - esnet
  elasticsearch2:
    image: docker.elastic.co/elasticsearch/elasticsearch:6.5.4
    container_name: elasticsearch2
    environment:
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - "discovery.zen.ping.unicast.hosts=elasticsearch"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata2:/usr/share/elasticsearch/data
    networks:
      - esnet

volumes:
  esdata1:
    driver: local
  esdata2:
    driver: local

networks:
  esnet:
```

删除elasticsearch集群以及数据目录：
```
docker-compose down -v
```

**使用docker命令，启动elasticsearch单节点**
```
docker run -p 9200:9200 -p 9300:9300 -e cluster.name=elasticsearch -e xpack.security.enabled=false --name=elasticsearch --restart=always -d wutang/elasticsearch-shanghai-zone
```

#### 3.查看结果
执行以下命令查看elasticsearch集群状态
```
[root@localhost ~]# curl http://127.0.0.1:9200/_cat/health
1546413475 07:17:55 docker-cluster green 2 2 0 0 0 0 0 0 - 100.0%
```

浏览器访问 主机ip+端口，例如 http://172.16.18.169:9200/ 可查看状态
```
{
  "name" : "c6XgdE_",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "7AiVdiT4RjilrOV1GkBOMw",
  "version" : {
    "number" : "6.5.4",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "d2ef93d",
    "build_date" : "2018-12-17T21:17:40.758843Z",
    "build_snapshot" : false,
    "lucene_version" : "7.5.0",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

其他启动参数修改可参考：
https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html


