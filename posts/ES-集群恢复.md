+++
title = "ES 集群恢复"
date = "2023-12-01"
tags = ["ES"]

+++



强制关机导致 ES 集群丢失 `.security-7` 索引，无法进行安全验证，尝试恢复的过程中，导致 ES 集群彻底不可用。




## 配置新集群

策略：复制 ES 根目录下的 data 目录到新集群，让新集群自动恢复索引。

 

node-01 配置文件 <u>elasticsearch.yml</u> :

```yml
cluster.name: es-cluster
#
# ------------------------------------ Node ------------------------------------

node.name: node-01
# node.roles: ["master", "data"]
#
# Add custom attributes to the node:
#
#node.attr.rack: r1
#
# ----------------------------------- Paths ------------------------------------
#
# Path to directory where to store the data (separate multiple locations by comma):
#
#path.data: /path/to/data
#
# Path to log files:
#
#path.logs: /path/to/logs
#
# ----------------------------------- Memory -----------------------------------
#
# Lock the memory on startup:
#
bootstrap.memory_lock: true
#
# Make sure that the heap size is set to about half the memory available
# on the system and that the owner of the process is allowed to use this
# limit.
#
# Elasticsearch performs poorly when the system is swapping the memory.
#
# ---------------------------------- Network -----------------------------------
#
# By default Elasticsearch is only accessible on localhost. Set a different
# address here to expose this node on the network:
#
# network.host: localhost
#
# By default Elasticsearch listens for HTTP traffic on the first free port it
# finds starting at 9200. Set a specific HTTP port here:
#
http.host: 0.0.0.0
http.port: 9201
http.cors.enabled: true
http.cors.allow-origin: "*"
http.cors.allow-headers: Authorization,X-Requested-With,Content-Type,Content-Length
#
# Allow other nodes to join the cluster from anywhere
# Connections are encrypted and mutually authenticated
transport.host: 0.0.0.0
transport.port: 9301
#
# --------------------------------- Discovery ----------------------------------
#
# Pass an initial list of hosts to perform discovery when this node is started:
# The default list of hosts is ["127.0.0.1", "[::1]"]
#
discovery.seed_hosts: ["127.0.0.1:9302", "127.0.0.1:9303"]
#
# Bootstrap the cluster using an initial set of master-eligible nodes:
#
# cluster.initial_master_nodes: ["node-01", "node-02"]
cluster.routing.allocation.enable: all
#
# For more information, consult the discovery and cluster formation module documentation.
#
# --------------------------------- Readiness ----------------------------------
#
# Enable an unauthenticated TCP readiness endpoint on localhost
#
#readiness.port: 9399
#
# ---------------------------------- Various -----------------------------------
#
# Allow wildcard deletion of indices:
#
#action.destructive_requires_name: false

#----------------------- BEGIN SECURITY AUTO CONFIGURATION -----------------------
#
# The following settings, TLS certificates, and keys have been automatically      
# generated to configure Elasticsearch security features on 06-11-2023 09:44:41
#
# --------------------------------------------------------------------------------

# Enable security features
xpack.security.enabled: false
xpack.security.enrollment.enabled: false

# Enable encryption for HTTP API client connections, such as Kibana, Logstash, and Agents
xpack.security.http.ssl:
  enabled: false
  keystore.path: certs/http.p12

# Enable encryption and mutual authentication between cluster nodes
xpack.security.transport.ssl:
  enabled: false
  verification_mode: certificate
  keystore.path: certs/transport.p12
  truststore.path: certs/transport.p12

#----------------------- END SECURITY AUTO CONFIGURATION -------------------------

```

node-02 配置文件 <u>elasticsearch.yml</u> :

```yml
...
node.name: node-02

http.port: 9202
transport.port: 9302

discovery.seed_hosts: ["127.0.0.1:9301", "127.0.0.1:9303"]
...
```

node-03 配置文件 <u>elasticsearch.yml</u> :

```yml
...
node.name: node-03

http.port: 9203
transport.port: 9303

discovery.seed_hosts: ["127.0.0.1:9301", "127.0.0.1:9302"]
...
```

由于前一个集群的 `.security-7` 索引丢失，怕新集群安全验证出现意外，关闭所有安全验证。



## 开始恢复

复制旧集群的 data 目录到新集群后，配置文件写好，就可以启动集群。

集群正常启动后，可在 kibana 的 console 里面查看恢复进度。

```
# 集群健康状态，不出意外，是 red
GET /_cluster/health

# 查看集群分片状态
GET /_cat/shards?v

# 允许分片自动分配
PUT _cluster/settings
{
  "persistent": {  
    "cluster.routing.allocation.enable": "all"
  }
}

# 查看正在恢复的分片
GET /_cat/recovery

# 查看特定索引恢复状态
GET /your_index/_recovery?human

# 增加恢复线程，persistent 是永久修改配置，transient 是临时修改配置
PUT /_cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.node_concurrent_recoveries": 50
  },
  "transient": {
    "cluster.routing.allocation.node_initial_primaries_recoveries": 50
  }
}

# 调用分片分配解释 API 找到无法分配副本的具体原因
GET /_cluster/allocation/explain

# 如果分配副本的原因是 max_retry, 可手动强制集群重试分配
POST /_cluster/reroute?retry_failed=true

```



等待索引状态变绿，开启集群的安全验证。

