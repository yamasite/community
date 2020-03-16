---
id: 2020-3-12-mishards.md
title: Mishards - Distributed vector search in Milvus
author: Peng Xu
---

# Mishards - Distributed vector search in Milvus

> Author: Peng Xu
>
> Date: 2020-3-12

Milvus aims to achieve efficient similarity search and analytics for massive-scale vectors. A standalone Milvus instance can easily handle vector search among billion-scale vectors. However, for 10 billion, 100 billion or even larger datasets, a Milvus cluster is needed. The cluster can be used as a standalone instance for upper-level applications and can meet the business needs of low latency, high concurrency for massive-scale data. A Milvus cluster can resend requests, separate reading and writing, scale horizontally, and expand dynamically, thus providing a Milvus instance that can expand without limit. Mishards is a distributed solution for Milvus.

This article will briefly introduce components of the Mishards architecture. More detailed information will be introduced in the upcoming articles.

![image_1](../assets/mishards/image_01.png)

## Distributed architecture overview

![image_2](../assets/mishards/image_02.png)

### Service tracing

![image_3](../assets/mishards/image_03.png)

### Primary service components

- Service discovery framework, such as ZooKeeper, etcd, and Consul.
- Load balancer, such as Nginx, HAProxy, Ingress Controller.
- Mishards node: stateless, scalable.
- Write-only Milvus node: single node and not scalable. You need to use high availability solutions for this node to avoid single-point-of-failure.
- Read-only Milvus node: stateful node and scalable.
- Shared storage service: all Milvus nodes use shared storage service to share data, such as NAS or NFS.
- Metadata service: All Milvus nodes use this service to share metadata. Currently, only MySQL is supported. This service requires MySQL high availability solution.

### Scalable components

- Mishards
- Read-only Milvus nodes

### Components introduction

#### Mishards nodes

Mishards is responsible for breaking up upstream requests and routing sub-requests to sub-services. The results are summarized to return to upstream.

![image_4](../assets/mishards/image_04.png)

As is indicated in the chart above, after accepting a TopK search request, Mishards first breaks up the request into sub-requests and send the sub-requests to the downstream service. When all sub-responses are collected, the sub-responses are merged and returned to upstream.

Because Mishards is a stateless service, it does not save data or participate in complex computation. Thus, nodes do not have high configuration requirements and the computing power is mainly used in merging sub-results. So, it is possible to increase the number of Mishards nodes for high concurrency.

#### Milvus nodes

Milvus nodes are responsible for CRUD related core operations, so they have relatively high configuration requirements. Firstly, memory size should be large enough to avoid too many disk IO operations. Secondly, CPU configurations can also affect performance. As the cluster size increases, more Milvus nodes are required to increase the system throughput.

- Read-only nodes and writable nodes

  - Core operations of Milvus are vector insertion and search. Search has extremely high requirements on CPU and GPU configurations, while insertion or other operations have relatively low requirements. Separating the node that runs search from the node that runs other operations leads to more economical deployment.
  - In terms of service quality, when a node is performing search operations, the related hardware is running in full load and cannot ensure the service quality of other operations. Therefore, two node types are used. Search requests are processed by read-only nodes and other requests are processed by writable nodes.

- Only one writable node is allowed

  - Currently, Milvus does not support sharing data for multiple writable instances.
  - During deployment, single-point-of-failure of writable nodes needs to be considered. High availability solutions need to be prepared for writable nodes.

- Read-only node scalability

  When the data size is extremely large, or the latency requirement is extremely high, you can horizontally scale read-only nodes as stateful nodes. Assume there are 4 hosts and each has the following configuration: CPU Cores: 16, GPU: 1, Memory: 64 GB. The following chart shows the cluster when horizontally scaling stateful nodes. Both computing power and memory scales linearly. The data is split into 8 shards with each node processing requests from 2 shards.

    ![image_5](../assets/mishards/image_05.png)

- When the number of requests is large for some shards, stateless read-only nodes can be deployed for these shards to increase throughput. Take the hosts above as an example. when the hosts are combined into a serverless cluster, the computing power increases linearly. Because the data to process does not increase, the processing power for the same data shard also increases linearly.

    ![image_6](../assets/mishards/image_06.png)

#### Metadata service

> Keywords: MySQL

For more information about Milvus metadata, refer to [How to view metadata](https://medium.com/@milvusio/milvus-metadata-management-1-6b9e05c06fb0). In a distributed system, Milvus writable nodes are the only producer of metadata. Mishards nodes, Milvus writable nodes, and Milvus read-only nodes are all consumers of metadata. Currently, Milvus only supports MySQl and SQLite as the storage backend of metadata. In a distributed system, the service can only be deployed as highly-available MySQL.

#### Service discovery

> Keywords: Apache Zookeeper、etcd、Consul、Kubernetes

![image_7](../assets/mishards/image_07.png)

Service discovery provides information about all Milvus nodes. Milvus nodes register their information when going online and logs out when going offline. Milvus nodes can also detect abnormal nodes by periodically checking the health status of services.

Service discovery contains a lot of frameworks, including etcd, Consul, ZooKeeper, etc. Mishards defines the service discovery interfaces and provides possibilities for scaling by plugins. Currently, Mishards provides two kinds of plugins, which correspond to Kubernetes cluster and static configurations. You can customize your own service discovery by following the implementation of these plugins. The interfaces are temporary and needs redesign. More information about writing your own plugin will be elaborated in the upcoming articles.

#### Load balancing and service sharding

> Keywords: NGINX、HAPROXY、Kubernetes

![image_8](../assets/mishards/image_08.png)

服务发现和负载均衡配合使用，负载均衡策略可以配置成轮询、哈希和一致性哈希等。

负载均衡器负责将用户请求转发至 Mishards 节点。

每个 Mishards 节点通过服务发现中心拿到所有下游 Milvus 节点的信息，通过元数据服务知晓整个数据相关元数据。Mishards 实现服务分片就是对于这些素材的一种消费。Mishards 定义了路由策略相关的接口，并通过插件提供扩展。目前 Mishards 默认提供了基于存储最底层 segment 级别的一致性哈希路由策略。如图有 10 个数据段 s1, s2, s3… s10, 现在选择基于数据段的一致性哈希路由策略，Mishards 会将涉及 s1, s4, s6, s9 数据段的请求路由到 Milvus 1 节点, s2, s3, s5 路由到 Milvus 2 节点, s7, s8, s10 路由到 Milvus 3 节点。

用户可以仿照默认的一致性哈希路由插件，根据自己的业务特点，定制个性化路由。

#### Tracing

> Keywords: OpenTracing、YAEGER、ZIPKIN

分布式系统错综复杂，请求往往会分发给内部多个服务调用，为了方便问题的定位，我们需要跟踪内部的服务调用链。随着系统的复杂性越来越高，一个可行的链路追踪系统带来的好处就越显而易见。我们选择了已进入 CNCF 的 OpenTracing 分布式追踪标准，OpenTracing 通过提供平台无关，厂商无关的 API ,方便开发人员能够方便的实现链路跟踪系统。

![image_9](../assets/mishards/image_09.png)

上图是 Mishards 服务中调用搜索时链路跟踪的例子，Search 顺序调用 `get_routing`, `do_search` 和 `do_merge`。而 `do_search`又调用了 `search_127.0.0.1`。

整个链路跟踪记录形成下面一个树：

![image_10](../assets/mishards/image_10.png)

下图是每个节点 request/response info 和 tags 的具体实例：

![image_11](../assets/mishards/image_11.png)

OpenTracing 已经集成到 Milvus 中，我将会在 Milvus 与 OpenTracing 一文中详细讲解相关的概念以及实现细节。

#### Monitoring and alerting

> Keywords: Prometheus、Grafana

![image_12](../assets/mishards/image_12.png)

Milvus 已集成开源 Prometheus 采集指标数据, Grafana 实现指标的监控，Alertmanager 用于报警机制。Mishards 也会将 Prometheus 集成进去。

#### Log analysis

> Keywords: Elastic、Logstash、Kibana

集群服务日志文件分布在不同服务节点上，排查问题需要登录到相关服务器获取。分析排查问题时，往往需要结合多个日志文件协同分析。使用 ELK 日志分析组件是一个不错的选择。

## Summary

Mishards 作为 Milvus 服务中间件，集合了服务发现，请求路由，结果聚合，链路跟踪等功能，同时也提供了基于插件的扩展机制。目前，基于 Mishards 的分布式方案还存在以下几点不足:

- Mishards 采用代理模式作为中间层，有一定的延迟损耗。
- Milvus 写节点是单点服务。
- 依赖高可用 MySQL 服务。
- 对于多分片且单分片多副本的情况，部署比较复杂。
- 缺少缓存层，比如对元数据的访问。

我们会在之后的版本中尽快解决这些已知的问题，让 Mishards 可以更加方便的应用生产环境。
