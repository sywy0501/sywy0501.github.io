---
layout: post
title: "SpringCloud Eureka承载千万级访问的原理解析"
subtitle: ""
date: 2020-10-15 20:00:00
author: "cs"
header-img: "img/post-bg-2020-10-15.jpg"
tags: 
- Java
- SpringCloud
---

#### 场景 

Eureka作为注册中心，每隔30s Eureka Client组件会发送一个请求给Eureka Server来获取最近的注册表信息，看其他服务的地址有无变化，此外Eureka Client每隔30s会发送一次心跳到Eureka Server通知注册中心服务依旧存活。  

假设有100个服务，每个服务部署到20台机器上，每分钟每个服务会拉取2次注册表发送2次心跳给Eureka Server，每分钟Eureka Server被请求20×100×4=8000次，每天8000×60×24=1152万，也就是说每天至少千万级的访问量。  

 ***如此大的访问量Eureka Server如何抗住？***

#### 注册表的数据结构
首先看一下注册表的数据结构
 ```java
private final ConcurrentHashMap<String,Map<String,Lease<InstanceInfo>>> registry 
 = new ConcurrentHashMap<String,Lease<InstanceInfo>>();
 ```

**registry**就是注册表的核心结构，注册表直接存于**内存**中.维护注册表,拉取注册表，更新心跳时间都发生在内存里。

* ConcurrentHashMap中的key就是服务名称
* value代表了一个服务的多个实例
* Map中的key就是服务实例的id
* value是Lease类，其泛型是InstanceInfo
* **InstanceInfo**代表了**服务实例的具体信息**，例如ip地址，hsotname以及端口号等
* **Lease**里面会维护每个服务**最近一次心跳的发送时间**

#### Eureka Server的多级缓存机制
纯内存操作即使有额外的网络开销处理请求的效率也是很高的，Eureka Server为了避免同时读写内存数据造成并发冲突，还采用了**多级缓存机制**。

##### 拉取注册表

* 首先从ReadOnlyCacheMap里查缓存的注册表。
* 若无，就查询ReadWriteCacheMap里缓存的注册表。
* 如果依然没有，就从内存中获取实际的注册表数据。

##### 注册表变更时

* 更新变更的注册表数据，同时过期掉ReadWriteCacheMap，此过程不会影响使用查询ReadOnlyCacheMap中的注册表。
* 一段时间内(30s)，各服务拉取注册表会直接读ReadOnlyCacheMap。
* 30s后，Eureka Server的后台线程发现ReadWriteCacheMap已经清空，也会清空ReadOnlyCacheMap中的缓存。
* 下次服务拉取注册表会从内存中获取最新的数据，同时填充各个缓存。

##### 优点

- 尽可能保证了内存注册表数据不会出现频繁的读写冲突问题。
- 进一步保证对 Eureka Server 的大量请求都是快速从纯内存走，性能极高。

#### 总结

- Eureka 通过设置适当的请求频率（拉取注册表 30s 间隔，发送心跳 30s 间隔），可以保证一个大规模的系统每秒请求 Eureka Server 的次数在几百次。
- 通过纯内存的注册表，保证了所有的请求都可以在内存处理，确保了极高的性能。
- 多级缓存机制确保了不会针对内存数据结构发生频繁的读写并发冲突操作，进一步提升性能。