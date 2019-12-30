#PostDock-Postgres + Docker

PostgreSQL集群具有适用于任何云和Docker环境（Amazon, Google Cloud, Kubernetes, Docker Compose, Docker Swarm, Apache Mesos）的**高可用**和**自修复**特点

![Formula](https://raw.githubusercontent.com/paunin/PostDock/master/artwork/formula2.png)

[![Build Status](https://travis-ci.org/paunin/PostDock.svg?branch=master)](https://travis-ci.org/paunin/PostDock)

- [信息](#info)
  * [特点](#features)
  * [盒子里有什么](#whats-in-the-box)
  * [Docker镜像tag约定](#docker-images-tags-convention)
    * [存储库](#repositories)
- [使用docker-compose启动集群](#start-cluster-with-docker-compose)
- [在Kubernetes中启动集群](#start-cluster-in-kubernetes)
- [自适应模式](#adaptive-mode)
- [SSH访问](#ssh-access)
- [复制插槽](#replication-slots)
- [配置集群](#configuring-the-cluster)
  * [Postgres](#postgres)
  * [Pgpool](#pgpool)
  * [Barman](#barman)
  * [其他配置](#other-configurations)
- [Postgres的扩展版本](#extended-version-of-postgres)
- [备份和恢复](#backups-and-recovery)
- [健康检查](#health-checks)
- [有用的命令](#useful-commands)
- [出版物](#publications)
- [场景](#scenarios)
- [如何贡献](#how-to-contribute)
- [FAQ](#faq)
- [文档和手册](#documentation-and-manuals)

-------

## 信息
### 特征
* 高可用性
* 自修复和自动重建
* [Split Brain](https://en.wikipedia.org/wiki/Split-brain_(computing)) Tolerance
* 最终/部分严格/严格一致性模式
* 读取负载平衡和连接池
* 增量备份（可选零数据丢失，[RPO = 0](https://en.wikipedia.org/wiki/Recovery_point_objective))
* 半自动时间点恢复程序
* 监视导出器的所有组件(节点，平衡器，备份)

### 盒子里有什么
[此项目](https://github.com/paunin/postgres-docker-cluster)包括:
* 用于`postgresql`集群和备份系统的Dockerfile
    * [postgresql](./src/Postgres-latest.Dockerfile)
    * [pgpool](./src/Pgpool-latest.Dockerfile)
    * [barman](./src/Barman-latest.Dockerfile)
* 用法示例（适用于生产环境，因为架构具有自动故障转移功能的故障保护）
    * 启动此集群的示例[docker-compose](./docker-compose/latest.yml)文件。
    * 目录[k8s](./k8s)包含在Kubernetes构建此集群信息


### Docker镜像tag约定

考虑到PostDock项目本身具有版本控制架构，该存储库生成的所有Docker映像都具有架构 - `postdock/<component>:<postdock_version>-<component><component_version>-<sub_component><sub_component_version>[...]`, 其中:

* `<postdock_version>` - 没有`bug-fix`组件的语义版本(可以是`1.1`, `1.2`, ...)
* `<component>`, `<component_version>` - 依赖组件:
    * `postgres`,`postgres-extended` - 主版本号和子版本号之间没有点号(可以是`95`, `96`, `10`, `11`, ...)
    * `pgpool` - 组件的主版本号和子版本号，中间没有点号(可以是`33`, `36`, `37`, ...)
    * `barman` - 仅主版本号(可以是`23`, `24`, ...)
* `<sub_component>`, `<sub_component_version>` - 依赖组件:
    * for `postgres` - `repmgr`可以是`32`, `40`, ...
    * for `barman` - `postgres`可以是`96`, `10`, `11`, ...
    * for `pgpool` - `postgres`可以是`96`, `10`, `11`, ...

可用别名 **(不建议用于生产环境)**:

* `postdock/<component>:latest-<component><component_version>[-<sub_component><sub_component_version>[...]]` - 指postdock的最新版本，该组件的某些版本，子组件(例如:`postdock/postgres:latest-postgres101-repmgr32`,`postdock/postgres:latest-barman23-postgres101`)
* `postdock/<component>:latest` - 指最新版本的postdock以及所有组件和子组件的最新版本 (例如: `postdock/postgres:latest`)
* `postdock/<component>:edge` - 指的是从master到postdock的构建，其中包含该组件以及所有子组件的最新版本(例如:`postdock/postgres:edge`)

#### 存储库

所有可用的标签（版本及其组合）在相应的docker-hub存储库中列出：

* [Postgres](https://hub.docker.com/r/postdock/postgres)
* [Pgpool](https://hub.docker.com/r/postdock/pgpool)
* [Barman](https://hub.docker.com/r/postdock/barman)

## 使用docker-compose启动集群

要启动集群，请以正常的`docker-compose`应用程序`docker-compose -f ./docker-compose/latest.yml up -d pgmaster pgslave1 pgslave2 pgslave3 pgslave4 pgpool backup`来运行集群。

示例集群的结构:

```
pgmaster (primary node1)  --|
|- pgslave1 (node2)       --|
|  |- pgslave2 (node3)    --|----pgpool (master_slave_mode stream)
|- pgslave3 (node4)       --|
   |- pgslave4 (node5)    --|
```


## 在Kubernetes中启动集群

### 使用 Helm (推荐用于生产)
您可以使用[Helm](https://helm.sh/)包管理器安装PostDock，查看[包中的README.md](./k8s/helm/PostDock/README.md)以获取更多信息。

### 简单 (不建议用于生产)
要更容易使用`k8s`目录下包含的服务组件，按照[例子](./k8s/README.md)的步骤安装`PostgreSQL`集群. 它还包含有关如何检查群集状态的信息

## 配置集群

您可以使用ENV变量`CONFIGS`(格式: `variable1:value1[,variable2:value2[,...]]`, 您可以使用变量`CONFIGS_DELIMITER_SYMBOL`，`CONFIGS_ASSIGNMENT_SYMBOL`重新定义定界符和赋值符号)来配置集群(`postgres.conf`)或 pgpool(`pgpool.conf`)的任何节点。另请参阅存储库根目录中的Dockerfiles和[docker-compose/latest.yml](./docker-compose/latest.yml)文件，以了解所有可用和已使用的配置！


### Postgres

其余的 - 您最好**听从**建议，查看[src/Postgres-latest.Dockerfile](./src/Postgres-latest.Dockerfile)文件，里面有丰富的注释:)

### Pgpool

在Pgpool中配置的最重要的部分(通用配置`CONFIGS`的一部分)是后端和可以访问这些后端的用户。您可以使用ENV变量配置后端。您可以在[docker-compose/latest.yml](./docker-compose/latest.yml)文件中找到设置pgpool的好示例:

```
DB_USERS: monkey_user:monkey_pass # in format user:password[,user:password[...]]
BACKENDS: "0:pgmaster:5432:1:/var/lib/postgresql/data:ALLOW_TO_FAILOVER,1:pgslave1::::,3:pgslave3::::,2:pgslave2::::" #,4:pgslaveDOES_NOT_EXIST::::
            # in format num:host:port:weight:data_directory:flag[,...]
            # defaults:
            #   port: 5432
            #   weight: 1
            #   data_directory: /var/lib/postgresql/data
            #   flag: ALLOW_TO_FAILOVER
REQUIRE_MIN_BACKENDS: 3 # minimal number of backends to start pgpool (some might be unreachable)
```


### Barman

Barman最重要的部分是设置访问变量。可以在[docker-compose/latest.yml](./docker-compose/latest.yml)文件中找到示例:

```
REPLICATION_USER: replication_user # default is replication_user
REPLICATION_PASSWORD: replication_pass # default is replication_pass
REPLICATION_HOST: pgmaster
POSTGRES_PASSWORD: monkey_pass
POSTGRES_USER: monkey_user
POSTGRES_DB: monkey_db
```


### 其他配置

**请参阅存储库根目录中的Dockerfiles和[docker-compose/latest.yml](./docker-compose/latest.yml)文件，以了解所有可用和已使用的配置!**


## 自适应模式

'自适应模式'意味着节点将能够决定是在启动还是切换到备用角色时，而不是充当主节点。
如果您传递`PARTNER_NODES` (同一级别的集群中节点的逗号分隔列表).
因此，每次容器启动时，它都会检查它之前是否是主服务器，以及周围是否没有新的主服务器(来自`PARTNER_NODES`列表),
否则它将以群集中的`upstream = new master`作为新的备用节点启动.

请记住：此功能不适用于级联复制，您不应将`PARTNER_NODES`传递给集群第二级上的节点。
取而代之的只是确保第一级上的所有节点都在运行，因此从第二级中重新启动任何节点后都将能够跟随第一级的初始上游。
这也意味着-从第二级复制可能会连接到根主服务器... 如果您决定采用自适应模式，那没什么大不了的。
但是，尽管如此，您仍可以使用`NODE_PRIORITY`环境变量，并确保第二级复制的入口点永远不会被选作新的根主机。


## SSH访问

如果您需要使用一些棘手的逻辑或较少麻烦的交叉检查来组织集群。您可以在每个节点上启用SSH服务器。只需在所有容器(包括 pgpool 和 barman)中设置ENV变量`SSH_ENABLE=1`(默认情况下禁用)即可。这样，您就可以通过`postgres`用户下的简单命令`gosu postgres ssh {NODE NETWORK NAME}`从任何节点连接到任何节点。

您还必须为所有容器设置相同的ssh密钥。为此，您需要将密钥文件保存在路径`/tmp/.ssh/keys/id_rsa`, `/tmp/.ssh/keys/id_rsa.pub`.


## 复制插槽

如果要禁用Postgres>=9.4的功能 - [复制插槽](https://www.postgresql.org/docs/9.4/static/catalog-pg-replication-slots.html)只需设置ENV变量`USE_REPLICATION_SLOTS=0` (默认启用)。因此，集群将仅依赖于Postgres配置`wal_keep_segments` (默认是`500`). 您还应该记住，配置`max_replication_slots`默认值`5`. 您可以使用ENV变量`CONFIGS`修改(与其他任何配置一样).


## Postgres的扩展版本

如果要使postgresql具有扩展功能和库，则应使用[Docker镜像tag约定](#docker-images-tags-convention)中的组件`postgres-extended`。每个目录[此处](./src/pgsql/extensions/bin/extensions)表示镜像中包含的扩展。

## 备份和恢复

[Barman](http://docs.pgbarman.org/) 用于提供实时备份和时间点恢复(PITR)..
从[Dockerfile](./src/Barman-latest.Dockerfile)中可以看到，该镜像需要连接信息（主机，端口）和两套凭据。

* 复制凭证
* Postgres管理员凭证

Barman充当热备用，并从源以流式传输WAL。另外，它定期使用`pg_basebackup`进行远程物理备份。
这样就可以在指定大小的窗口内，使PITR在合理的时间内运行，因为您只需从最新的基本备份中重放WAL。
Barman根据防护策略自动删除旧的备份和WAL。
备份源是静态的 — pgmaster节点。
如果发生主服务器故障转移，将从备用服务器继续备份。
整个备份过程是远程执行的，但是要恢复则需要SSH访问。

*在生产中使用之前，请阅读以下文档:*
 * http://docs.pgbarman.org/release/2.2/index.html
 * https://www.postgresql.org/docs/current/static/continuous-archiving.html

*有关灾难恢复过程，请参见[RECOVERY.md](./doc/RECOVERY.md)*

Barman在`:8080/metrics`上提供了多个指标，有关更多信息，请参见[Barman docs](./barman/README.md)

## 健康检查

为了确保您的群集正常工作而不会出现'split-brain'或其他问题, 您必须设置运行状况检查并在任何运行状况检查返回非零结果时停止容器. 当您使用具有livenessProbe的Kubernetes时将会很有用(请在[the example](./k8s/example2-single-statefulset/nodes/node.yml)中查看如何使用) 

* Postgres容器:
    * `/usr/local/bin/cluster/healthcheck/is_major_master.sh` - 检测节点是否充当'false'-master并且存在另一个master - 具有更多备用数据库
* Pgpool
    * `/usr/local/bin/pgpool/has_enough_backends.sh [REQUIRED_NUM_OF_BACKENDS, default=$REQUIRE_MIN_BACKENDS]` - 检查`pgpool`后面是否有足够的后端
    * `/usr/local/bin/pgpool/has_write_node.sh` - 检查后端之一是否可以用作具有写访问权的主机



## 有用的命令

* 获取当前集群的地图(在任何`postgres` 节点上): 
    * `gosu postgres repmgr cluster show` - 尝试根据请求连接到所有节点，并忽略状态在`$(get_repmgr_schema).$REPMGR_NODES_TABLE`中的节点
    * `gosu postgres psql $REPLICATION_DB -c "SELECT * FROM $(get_repmgr_schema).$REPMGR_NODES_TABLE"` - 仅从表中选择数据
* 获取`pgpool`状态 (在任何`pgpool`节点上): `PGPASSWORD=$CHECK_PASSWORD psql -U $CHECK_USER -h localhost template1 -c "show pool_nodes"`
* 在`pgpool`容器中检查主节点是否存在: `/usr/local/bin/pgpool/has_write_node.sh` 

任何命令都可以用`docker-compose`或`kubectl`包装 - `docker-compose exec {NODE} bash -c '{COMMAND}'` or `kubectl exec {POD_NAME} -- bash -c '{COMMAND}'`


## 场景

检查 [文档](./doc/FLOWS.md) 以了解故障转移，split-brain抵抗和恢复的不同情况

## 出版物
* [Medium.com上的文章](https://medium.com/@dpaunin/postgresql-cluster-into-kubernetes-cluster-f353cde212de)
* [Статья на habr-e](https://habrahabr.ru/post/301370/)


## 如何贡献

查看[文档](./doc/CONTRIBUTE.md)以了解如何做出贡献

## 常问问题

*实际/实时用法示例: 
    * [Lazada/Alibaba Group](http://lazada.com/)
* 为什么不[sorintlab/stolon](https://github.com/sorintlab/stolon):
    * 带有大量代码的复杂逻辑
    * 生态系统的非标准工具
* [如何在使用docker在Postgresql上进行故障转移后提升主服务器](http://stackoverflow.com/questions/37710868/how-to-promote-master-after-failover-on-postgresql-with-docker)
* 杀死中间的节点(例如`pgslave1`)将导致[整个分支死亡](https://groups.google.com/forum/?hl=fil#!topic/repmgr/lPAYlawhL0o)
   * 作为第二层或更深层复制的中介，应该根本无法连接到根主服务器 (这会给服务器增加额外的负载)或根本无法向上游更改


## 文档和手册

* Postgres中的流式复制: https://wiki.postgresql.org/wiki/Streaming_Replication
* Repmgr: https://github.com/2ndQuadrant/repmgr
* Pgpool2: http://www.pgpool.net/docs/latest/pgpool-en.html
* Barman: http://www.pgbarman.org/
* Kubernetes: http://kubernetes.io/
