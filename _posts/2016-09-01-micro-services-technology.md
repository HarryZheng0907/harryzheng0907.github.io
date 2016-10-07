---
layout:     post
title:      "服务化改造（二） - 技术概览"
date:       2016-09-09 13:20:00
author:     "Harry"
header-img: "img/post-bg-2015.jpg"
tags:
    - 微服务
---

在上一篇文章中，我总结了服务化第一阶段需要完成的功能，如下图所示

![服务化第一阶段](http://oeii54s39.bkt.clouddn.com/%E6%9C%8D%E5%8A%A1%E5%8C%96%E7%AC%AC%E4%B8%80%E9%98%B6%E6%AE%B5.png)

现在需要根据需要实现的功能寻找相应的技术，由于公司的项目是以PHP为主，所以需要寻找以PHP为主的技术栈，由于现在的服务化架构都是以Java、C/C++为主，所以要寻找成熟的基于PHP的服务化架构方案非常困难，可以说几乎没有，我google了半天，没看到有谁用PHP去做服务化架构的主语言的，大概是由于PHP在大部分程序员看来，都是偏向写前端业务逻辑，不太适合写服务。

PHP能不能写服务这个暂不评论，但是我们公司现在都是PHPer，这时候要转成Java或者其他语言都不太现实，所以只能继续研究如何用PHP来实现服务化，所幸现在的服务化都提倡支持多语言，所以很多框架都会支持PHP，只是支持程度有所差别，还需要自己筛选，下面这张图是我找的PHP能用的服务化相关技术

![服务化第一阶段-技术概览](http://oeii54s39.bkt.clouddn.com/%E6%9C%8D%E5%8A%A1%E5%8C%96%E7%AC%AC%E4%B8%80%E9%98%B6%E6%AE%B5-%E6%8A%80%E6%9C%AF%E6%A6%82%E8%A7%88.png)

#### 注册中心

##### ZooKeeper

ZooKeeper是一个分布式的，开放源码的分布式应用程序协调服务。 它是一个为分布式应用提供一致性服务的软件

##### etcd+Registrator+Confd

etcd是一个采用HTTP协议的健/值对存储系统，它是一个分布式和功能层次配置系统，可用于构建服务发现系统。其很容易部署、安装和使用，提供了可靠的数据持久化特性。它是安全的并且文档也十分齐全。Registrator通过检查容器在线或者停止运行状态自动注册和去注册服务，Confd是一个轻量级的配置管理工具，常见的用法是通过使用存储在etcd、consul和其他一些数据登记处的数据保持配置文件的最新状态

##### Consul

Consul是强一致性的数据存储，它提供分级键/值存储方式，不仅可以存储数据还提供健康检查、Web界面和数据中心等功能，是专门为了服务发现实现的一站式解决方案

注册中心ZooKeeper是最老牌的，但是注册中心只是其中一项用途，如果ZooKeeper还需要了解它其他的用法，会引入额外的复杂性，而且ZooKeeper并不支持PHP，需要通过C扩展来间接支持，但是这样的话，php对ZooKeeper的支持就受限于所用的C扩展，所以暂不考虑，etcd+Registrator+Confd是通过检查容器来实现自动注册跟去注册的，所以不适用于非容器的环境，而Consul是专为注册中心设计，而且提供Restful API，对任何语言都有非常好的支持，所以最后决定选择Consul

#### 通信框架

##### Restful

基于HTTP的通讯方式

##### Thrift

由Facebook开发的一款RPC通讯框架

##### gRPC+Protocol Buffers

由Google开发的基于HTTP/2的RPC通讯框架

通信框架中，Restful是基于HTTP协议，是短连接且数据没有压缩，还有不小的头部，所以基本不考虑，gPRC对PHP的支持较弱，只支持客户端，还不支持pb3，所以也不考虑，所以唯一的选择就是Thrift

#### 消息队列

##### RabbitMQ

使用Erlang编写的队列，支持多协议，全功能，高并发，缺点是比较重

##### ActiveMQ

功能与RabitMQ类型，相对较轻量，但性能不如RabitMQ

##### ZeroMQ

高TPS，但仅提供非持久性队列

##### Kafka

为日志而生，超高TPS跟吞吐量，但不能保证数据不丢失

消息队列主要用于日志还有事件，由于Kafka是转为日志设计的，所以日志基本上确定是使用Kafka，而用于事件的话，要求队列能够保障事件必须送达，从而保证一致性，所以需要队列必须高可用还有稳定，RabbitMQ跟ActiveMQ都可选择，鉴于RabbitMQ性能较好且用户量大，所以选择RabbitMQ

#### 日志

##### ElasticSearch

ElasticSearch是个开源分布式搜索引擎，它的特点有：分布式，零配置，自动发现，索引自动分片，索引副本机制，restful风格接口，多数据源，自动搜索负载等。

##### Logstash

Logstash是一款轻量级的日志搜集处理框架，可以方便的把分散的、多样化的日志搜集起来，并进行自定义的处理，然后传输到指定的位置，比如某个服务器或者文件。

##### Kibana

kibana 也是一个开源和免费的工具，他可以帮助您汇总、分析和搜索重要数据日志并提供友好的web界面。他可以为 Logstash 和 ElasticSearch 提供的日志分析的 Web 界面

##### Zipkin

Zipkin 是一款开源的分布式实时数据追踪系统（Distributed Tracking System），基于 Google Dapper 的论文设计而来，由 Twitter 公司开发贡献。其主要功能是聚集来自各个异构系统的实时监控数据，用来追踪微服务架构下的系统延时问题

日志上ElasticSearch+Logstash+Kibana是经典组合，加上Zipkin用于调用链，四个缺一不可

综上所述，最后选择的技术如图所示

![](http://oeii54s39.bkt.clouddn.com/%E6%9C%8D%E5%8A%A1%E5%8C%96%E7%AC%AC%E4%B8%80%E9%98%B6%E6%AE%B5%EF%BC%88%E5%B7%B2%E7%A1%AE%E5%AE%9A%EF%BC%89.png)




