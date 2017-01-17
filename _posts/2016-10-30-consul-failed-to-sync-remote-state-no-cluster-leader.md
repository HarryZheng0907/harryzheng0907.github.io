---
layout:     post
title:      "Consul: failed to sync remote state: No cluster leader"
date:       2016-10-30 19:52:00
author:     "Harry"
header-img: "img/post-bg-2015.jpg"
tags:
    - Consul
---

### Consul: failed to sync remote state: No cluster leader

今天启动consul时，发现报这个错误，以下是配置文件和启动指令

启动指令

```shell
consul agent --config-file /etc/consul.d/basic_config.json
```

配置文件

```json
{
        "datacenter": "dc-dev",
        "data_dir": "/tmp/consul",
        "log_level": "INFO",
        "node_name": "service",
        "server": true,
        "services": [
                {
                    "name": "Shop",
                    "port": 9091
                },
                {
                    "name": "User",
                    "port": 9092
                }
        ]
}
```

配置文件没有什么特别之处，指令也没有设置其他参数，所以目测应该还是缺少某个关键参数

经过研究Consul的官方文档，看到了两个关键参数

- [`-bootstrap`](https://www.consul.io/docs/agent/options.html#_bootstrap) - This flag is used to control if a server is in "bootstrap" mode. It is important that no more than one server *per* datacenter be running in this mode. Technically, a server in bootstrap mode is allowed to self-elect as the Raft leader. It is important that only a single node is in this mode; otherwise, consistency cannot be guaranteed as multiple nodes are able to self-elect. It is not recommended to use this flag after a cluster has been bootstrapped.
- [`-bootstrap-expect`](https://www.consul.io/docs/agent/options.html#_bootstrap_expect) - This flag provides the number of expected servers in the datacenter. Either this value should not be provided or the value must agree with other servers in the cluster. When provided, Consul waits until the specified number of servers are available and then bootstraps the cluster. This allows an initial leader to be elected automatically. This cannot be used in conjunction with the legacy [`-bootstrap`](https://www.consul.io/docs/agent/options.html#_bootstrap) flag.

由上可见，consul启动之后需要选举出Leader，用于同步数据，而当server的节点只有一个时，需要设置-bootstrap的参数用于完成consul的自选举，不然就会报：failed to sync remote state: No cluster leader

当server的节点有多个时，则需设置-bootstrap-expect，用于设置预期的server数量，当server的数量达到预设值时，才启动选举机制，server的数量达不到预设值则也会报：failed to sync remote state: No cluster leader

通过上面的分析，我们可以对开头的错误给出两种解决方案

```shell
consul agent --config-file /etc/consul.d/basic_config.json -bootstrap
```

```shell
consul agent --config-file /etc/consul.d/basic_config.json -bootstrap-expect 1
```

