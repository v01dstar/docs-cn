---
title: 只读存储节点最佳实践
summary: 介绍如何通过使用只读存储节点，达到物理隔离部分流量的目的。
---

# 只读存储节点最佳实践

本文将介绍如何配置"只读"存储节点，以及如何将部分流量，如备份，分析，测试等，导向这些节点, 以实现和重要线上流量达到物理隔离的目的。

## 原理

在 TiDB 集群中，每一分片的数据都被复制了3份，分别存储在3个不同的节点上。每个分片的其中一份会被选为 leader, 承载主要的读写请求。 TiDB 同时还支持 follower read，即在显式声明的情况下，允许客户端从 follower 节点读取数据。 但是，因为在集群中，会有很多分片，每个分片的 leader 又会均匀的分布在所有的 TiKV 节点上，所以,
使用 follower read 去读某些分片的话，虽然会使这些分片的 leader 负载降低，但却会因为占用了宿主机的资源，而导致其他 leader 处在这台机器的分片读写性能收到影响。所以，从整体来看，并不会带来资源隔离的效果(当然，如果使用得当， follower read 有其他的好处)。

![without-read-only-nodes](/media/readonly-nodes-figure1.png)

假如，我们有一部分不紧急，优先度较低的任务需要被执行，同时又想最大程度的减少对其他重要线上服务的影响，那么我们可以这么做：

1. 分配几个专用节点，将所有，或部分数据再复制一个副本出来放在这些节点上。
2. 使用 follower read 的功能，将低优先级的任务，导向这些特殊节点。
3. 利用 Raft 协议 Learner 无法成为 leader 的特性，将这些副本设置为 Learner，以防这些节点上的副本偶然成为 leader，导致高低优先级的任务互相影响。

![with-read-only-nodes](/media/readonly-nodes-figure2.png)

## 操作步骤

### 1. 将部分 TiKV 节点指定为只读节点

通过给 TiKV 节点标记特殊 label（使用 `$` 作为 label key 的前缀）的方式，可以把部分节点指定为特殊只读节点。除非通过设置 Placement Rules 的方式显式指定这些节点存储某些数据，否则 PD 不会调度任何数据将其存储在这些节点上。

例如，使用 tiup cluster edit-config 子命令，进行如下配置：

```
tikv_servers:
  - host: ...
    ...
    labels:
      $mode: readonly
```

### 2. 使用 placement rules 将数据以 learner 形式存储在只读节点上

由于当前我们无法通过 placement rules SQL 接口对 meta 数据的分布进行配置，所以我们使用 placement rules pd-ctl 接口来配置 placement rules。

首先，我们可以使用如下命令，将默认 placement rules 导出，作为修改基础：

```
pd-ctl config placement-rules rule-bundle load --out="rules.json"
```

如果之前没有配置过 placement rules，那么会导出如下内容：

```
[
  {
    "group_id": "pd",
    "group_index": 0,
    "group_override": false,
    "rules": [
      {
        "group_id": "pd",
        "id": "default",
        "start_key": "",
        "end_key": "",
        "role": "voter",
        "count": 3
      }
    ]
  }
]
```

我们可以基于默认配置，将所有数据在只读节点上以 learner 方式存储一份。

```
[
  {
    "group_id": "pd",
    "group_index": 0,
    "group_override": false,
    "rules": [
      {
        "group_id": "pd",
        "id": "default",
        "start_key": "",
        "end_key": "",
        "role": "voter",
        "count": 3
      },
      {
        "group_id": "pd",
        "id": "readonly",
        "start_key": "",
        "end_key": "",
        "role": "learner",
        "count": 1,
        "label_constraints": [
          {
            "key": "$mode",
            "op": "in",
            "values": [
              "readonly"
            ]
          }
        ],
        "version": 1
      }
    ]
  }
]
```

接着，使用 placement rules pd-ctl 接口，将配置写入 PD：

```
pd-ctl config placement-rules rule-bundle save --in="rules.json"
```

> **__注意：__** 当对已存在大量数据的集群进行如上操作时，整个集群可能需要一段时间才能将数据完全复制只读节点上，这期间，只读节点可能无法进行服务。

### 3. 使用 follower read 功能

#### 3.1 在 TiDB 中使用 follower read

我们可以通过设置 tidb_replica_read 的值为 learner (在6.6后新加入的 option )，来读取只读节点上的数据。

```
set tidb_replica_read=learner;
```

#### 3.2 在 TiSpark 中使用 follower read

你可以在 Spark 配置文件设置中 `spark.tispark.replica_read = learner` 来读取只读节点上的数据：

```
spark.tispark.replica_read learner 
```

#### 3.3 在 Backup 时中使用 follower read

我们可以在 br 命令行中添加 `--backup-replica-read` 参数，来读取只读节点上的数据。

```
br backup full ... --backup-replica-read learner
```