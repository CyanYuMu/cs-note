# ETCD

1. ## ETCD是什么？有什么用？

GO语言实现的分布式系统重要基础组件。可用来构建高可用的分布式键值对数据库。为用户提供了HTTP api。数据分层再文件目录中。类似于文件系统。高性能。基于Raft共识算法使它很可靠。支持跨平台，拥有强大的社区。etcd 的 Raft 算法，提供了可靠的方式存储分布式集群涉及的数据。etcd 广泛应用在微服务架构和 Kubernates 集群中，不仅可以作为服务注册与发现，还可以作为键值对存储的中间件。从业务系统 Web 到 Kubernetes 集群，都可以很方便地从 etcd 中读取、写入数据。

etcd 经常用于服务注册与发现的场景，此外还有键值对存储、消息发布与订阅、分布式锁等场景。

#### 键值对存储

etcd 是一个**「键值存储」**的组件，存储是 etcd 最基本的功能，其他应用场景都是建立在 etcd 的可靠存储上。etcd 的存储有如下特点：

- 采用键值对数据存储，读写性能一般高于[关系型数据库]；
- etcd 集群分布式存储，多节点集群更加可靠；
- etcd 的存储采用类似文件目录的结构：
- 叶子节点存储数据，其他节点不存储，这些数据相当于文件；
- 非叶节点一定是目录，这些节点不能存储数据。

比如 Kubernetes 将一些元数据存储在 etcd 中，将存储状态数据的的复杂工作交给 etcd，Kubernetes 自身的功能和架构能够更加专注

服务注册与发现

分布式环境中，业务服务多实例部署，这个时候涉及到服务之间调用，就不能简单使用硬编码的方式指定服务实例信息。服务注册与发现就是解决如何找到分布式集群中的某一个服务（进程），并与之建立联系。

服务注册与发现涉及三个主要的角色：服务请求者、服务提供者和[服务注册中心]

![image-20251212214200859](C:\Users\CyanYuMu\AppData\Roaming\Typora\typora-user-images\image-20251212214200859.png)

##### 三大支柱

服务提供者启动的时候，在服务注册中心进行注册自己的服务名、主机地址、端口等信息；服务请求者需要调用对应的服务时，一般通过服务名请求服务注册中心，服务注册中心返回对应的实例地址和端口；服务请求者获取到实例地址、端口之后，绑定对应的服务提供者，实现远程调用。

etcd 基于 Raft 算法，能够有力地保证分布式场景中的一致性。各个服务启动时注册到 etcd 上，同时为这些服务配置键的 TTL 时间，定时保持服务的心跳以达到监控健康状态的效果。通过在 etcd 指定的主题下注册的服务也能在对应的主题下查找到。为了确保连接，我们可以在每个服务机器上都部署一个 Proxy 模式的 etcd，这样就可以确保访问 etcd 集群的服务都能够互相连接。

#### 消息的发布与订阅

在分布式系统中，服务之间还可以通过消息通信，即消息的发布与订阅。通过构建一个消息中间件，服务提供者发布对应主题的消息，而消费者则订阅他们关心的主题，一旦对应的主题有消息发布，即会产生订阅事件，消息中间件就会通知该主题所有的订阅者。

![image-20251212214449768](C:\Users\CyanYuMu\AppData\Roaming\Typora\typora-user-images\image-20251212214449768.png)

如微服务架构中的认证鉴权服务，Auth 服务的实例地址、端口和实例节点的状态存放在 etcd 中，客户端应用订阅对应的主题，而 etcd 设置 key TTL 可以确保存储的服务实例的健康状态。

#### 分布式锁

分布式系统中涉及到多个服务实例，存在跨进程之间资源调用，对于资源的协调分配，单体架构中的锁已经无法满足需要，需要引入分布式锁的概念。分布式锁可以将资源标记存储，这里的存储不是单纯属于某个进程，而是公共存储，诸如Redis、Memcache、关系型数据库、文件等。

etcd 基于 Raft 算法，实现分布式集群的一致性，存储到 etcd 集群中的值必然是全局一致的，因此基于 etcd 很容易实现分布式锁。分布式锁有两种使用方式：保持独占和控制时序。

保持独占，从字面可以知道，所有获取资源的请求，只有一个成功。etcd 通过分布式锁原子操作 CAS 的 API，设置 prevExist 值，从而保证在多个节点同时去创建某个目录时，最后只有一个成功，创建成功的请求获取到锁。

![image-20251212214736137](C:\Users\CyanYuMu\AppData\Roaming\Typora\typora-user-images\image-20251212214736137.png)

控制时序，有点类似于队列缓冲，所有的请求都会被安排分配资源，但是获得锁的顺序也是全局唯一的，执行按照先后的顺序。etcd 提供了一套自动创建有序键的 API，对一个目录的建值操作，这样 etcd 会自动生成一个当前最大的值为键，并存储该值。同时还可以使用 API 按顺序列出当前目录下的所有键值。

#### 涉及到的一些概念

- Raft：分布式一致性算法；
- Node：Raft 状态机实例；
- Member：管理着 Node 的 etcd 实例，为客户端请求提供服务；
- Cluster：etcd 集群，由多个 Member 构成；
- Peer：同一个 etcd 集群中的另一个 Member；
- Client：客户端，向 etcd 发送 HTTP 请求；
- WAL：持久化存储的日志格式，预写式日志；
- Snapshot：etcd 数据快照，防止 WAL 文件过多而设置的快照。

### 可以用docker先拉一个到本地来用。

#### etcd 集群部署

etcd 是分布式环境中重要的中间件，一般在生产环境不会单节点部署 etcd，为了 etcd 的高可用，避免单点故障，etcd 通常都是集群部署。本小节将会介绍如何进行 etcd 集群部署。引导 etcd 集群的启动有以下三种方式：

- 静态指定
- etcd 动态发现
- DNS 发现

静态指定的方式需要事先知道集群中的所有节点。在许多情况下，群集成员的信息是动态生成。这种情况下，可以在动态发现服务的帮助下启动 etcd 群集。

下面我们将会分别介绍这几种方式。

##### 3.3.1 静态方式启动 etcd 集群

用goreman工具。是一个多进程管理工具进行tecd集群的搭建。搭建的集群实例如下：

| HostName | ip        | 客户端交互端口 | peer 通信端口 |
| :------- | :-------- | :------------- | :------------ |
| infra1   | 127.0.0.1 | 2379           | 2380          |
| infra2   | 127.0.0.1 | 22379          | 22380         |
| infra3   | 127.0.0.1 | 32379          | 32380         |

```yaml
etcd: etcd --name infra1 --listen-client-ruls http://127.0.0.1:2379
--advertise-client-urls http://127.0.0.1:22379
--listen-peer-urls http:..127.0.0.1:2380
--initial-advertise-peer-urls http://127.0.0.1:2380
 --initail-cluster-token etcd-cluster-1
 
```

其他两个 etcd 成员的配置类似，不在赘述。配置项说明如下：

- --name：etcd 集群中的节点名；
- --listen-peer-urls：用于节点之间通信的地址，可以监听多个；
- --initial-advertise-peer-urls：与其他节点之间通信的地址；
- --listen-client-urls：监听客户端通信的地址，可以有多个；
- --advertise-client-urls：用于客户端与节点通信的地址；
- --initial-cluster-token：标识不同 etcd 集群的 token；
- --initial-cluster：即指定的 initial-advertise-peer-urls 的所有节点；
- --initial-cluster-state：new，新建集群的标志。

**注意上面的脚本**，*etcd 命令执行时需要根据本地实际的安装地址进行配置。下面我们启动 etcd 集群。*

```shell
goreman -f /opt/procfile start
```

使用如上的命令启动启动 etcd 集群，启动完成之后查看集群内的成员。

```shell
$ etcdctl --endpoints=http://localhost:22379  member list

8211f1d0f64f3269, started, infra1, http://127.0.0.1:12380, http://127.0.0.1:12379, false
91bc3c398fb3c146, started, infra2, http://127.0.0.1:22380, http://127.0.0.1:22379, false
fd422379fda50e48, started, infra3, http://127.0.0.1:32380, http://127.0.0.1:32379, false
```

##### docker 启动集群

etcd 使用 gcr.io/etcd-development/etcd 作为容器的主要加速器， quay.io/coreos/etcd 作为辅助的加速器。可惜这两个加速器我们都没法访问，如果下载不了，可以使用笔者提供的地址：

```sh
docker pull bitnami/etcd:3.4.7
```

然后将拉取的镜像重新 tag：

```shell
docker image tag bitnami/etcd:3.4.7 quay.io/coreos/etcd:3.4.7
```

镜像设置好之后，我们启动 3 个节点的 etcd 集群，脚本命令如下：

```javascript
REGISTRY=quay.io/coreos/etcd

# For each machine
ETCD_VERSION=3.4.5
TOKEN=my-etcd-token
CLUSTER_STATE=new
NAME_1=etcd-node-0
NAME_2=etcd-node-1
NAME_3=etcd-node-2
HOST_1= 192.168.202.128
HOST_2= 192.168.202.129
HOST_3= 192.168.202.130
CLUSTER=${NAME_1}=http://${HOST_1}:2380,${NAME_2}=http://${HOST_2}:2380,${NAME_3}=http://${HOST_3}:2380
DATA_DIR=/var/lib/etcd

# For node 1
THIS_NAME=${NAME_1}
THIS_IP=${HOST_1}
docker run \
  -p 2379:2379 \
  -p 2380:2380 \
  --volume=${DATA_DIR}:/etcd-data \
  --name etcd ${REGISTRY}:${ETCD_VERSION} \
  /usr/local/bin/etcd \
  --data-dir=/etcd-data --name ${THIS_NAME} \
  --initial-advertise-peer-urls http://${THIS_IP}:2380 --listen-peer-urls http://0.0.0.0:2380 \
  --advertise-client-urls http://${THIS_IP}:2379 --listen-client-urls http://0.0.0.0:2379 \
  --initial-cluster ${CLUSTER} \
  --initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}

# For node 2
THIS_NAME=${NAME_2}
THIS_IP=${HOST_2}
docker run \
  -p 2379:2379 \
  -p 2380:2380 \
  --volume=${DATA_DIR}:/etcd-data \
  --name etcd ${REGISTRY}:${ETCD_VERSION} \
  /usr/local/bin/etcd \
  --data-dir=/etcd-data --name ${THIS_NAME} \
  --initial-advertise-peer-urls http://${THIS_IP}:2380 --listen-peer-urls http://0.0.0.0:2380 \
  --advertise-client-urls http://${THIS_IP}:2379 --listen-client-urls http://0.0.0.0:2379 \
  --initial-cluster ${CLUSTER} \
  --initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}

# For node 3
THIS_NAME=${NAME_3}
THIS_IP=${HOST_3}
docker run \
  -p 2379:2379 \
  -p 2380:2380 \
  --volume=${DATA_DIR}:/etcd-data \
  --name etcd ${REGISTRY}:${ETCD_VERSION} \
  /usr/local/bin/etcd \
  --data-dir=/etcd-data --name ${THIS_NAME} \
  --initial-advertise-peer-urls http://${THIS_IP}:2380 --listen-peer-urls http://0.0.0.0:2380 \
  --advertise-client-urls http://${THIS_IP}:2379 --listen-client-urls http://0.0.0.0:2379 \
  --initial-cluster ${CLUSTER} \
  --initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}
```

注意，上面的脚本是部署在三台机器上面，每台机器执行对应的脚本即可。在运行时可以指定 API 版本：

```javascript
docker exec etcd /bin/sh -c "export ETCDCTL_API=3 && /usr/local/bin/etcdctl put foo bar"
```

docker 的安装方式比较简单，根据需要可以定制一些配置。

#### **etcdctl 的实践应用**

etcdctl 是 etcd 的命令行客户端，用户通过 etcdctl 直接跟 etcd 进行交互，可以实现 HTTP API 的功能。在测试阶段，etcdctl 可以方便对服务进行操作或更新数据库内容。一开始可以通过 etdctl 来熟悉 etcd 相关操作。需要注意的是，etcdctl 在两个不同的 etcd 版本下的行为方式也完全不同。

```javascript
export ETCDCTL_API=2
export ETCDCTL_API=3
```

## 常用指令

#### ctcdctl常用：

```javascript
--debug 输出 CURL 命令，显示执行命令的时候发起的请求
--no-sync 发出请求之前不同步集群信息
--output, -o 'simple' 输出内容的格式(simple 为原始信息，json 为进行 json 格式解码，易读性好一些)
--peers, -C 指定集群中的同伴信息，用逗号隔开(默认为: "127.0.0.1:4001")
--cert-file HTTPS 下客户端使用的 SSL 证书文件
--key-file HTTPS 下客户端使用的 SSL 密钥文件
--ca-file 服务端使用 HTTPS 时，使用 CA 文件进行验证
--help, -h 显示帮助命令信息
--version, -v 打印版本信息
```

#### 常用的数据库命

etcd 在键的组织上采用了如同类似文件目录的结构，即层次化的空间结构，我们可以为键指定单独的名字。etcd 数据库提供的操作，则主要围绕对键值和目录的增删改查。

###### **5.2.1 键操作**

set  指定某个键的值。例如：

```javascript
$ etcdctl put /testdir/testkey "Hello world"
$ etcdctl put /testdir/testkey2 "Hello world2"
$ etcdctl put /testdir/testkey3 "Hello world3"
```

成功写入三对键值，/testdir/testkey、/testdir/testkey2 和 /testdir/testkey3。

get  获取指定键的值。例如：

```javascript
$ etcdctl get /testdir/testkey
Hello world
```

get 十六进制读指定的值：

```javascript
$ etcdctl get /testdir/testkey --hex
\x2f\x74\x65\x73\x74\x64\x69\x72\x2f\x74\x65\x73\x74\x6b\x65\x79 #键
\x48\x65\x6c\x6c\x6f\x20\x77\x6f\x72\x6c\x64 #值
```

加上 `--print-value-only` 可以读取对应的值。

get 范围内的值

```javascript
 $ etcdctl get /testdir/testkey /testdir/testkey3

/testdir/testkey
Hello world
/testdir/testkey2
Hello world2
```

可以看到，获取了大于等于 /testdir/testkey，且小于 /testdir/testkey3 的键值对。testkey3 不在范围之内，因为范围是半开区间 [testkey, testkey3), 不包含 testkey3。

获取某个前缀的所有键值对，通过 --prefix 可以指定前缀：

```javascript
$ etcdctl get --prefix /testdir/testkey
/testdir/testkey
Hello world
/testdir/testkey2
Hello world2
/testdir/testkey3
Hello world3
```

这样既可获取所有以 /testdir/testkey 开头的键值对。当前缀获取的结果过多时，还可以通过  --limit=2 限制获取的数量：

```javascript
etcdctl get --prefix --limit=2 /testdir/testkey
```

读取键过往版本的值 应用可能想读取键的被替代的值。例如，应用可能想通过访问键的过往版本来回滚到旧的配置。或者，应用可能想通过多个请求来得到一个覆盖多个键的统一视图，而这些请求可以通过访问键历史记录而来。因为 etcd 集群上键值存储的每个修改都会增加 etcd 集群的全局修订版本，应用可以通过提供旧有的 etcd 修改版本来读取被替代的键。现有如下这些键值对：

```javascript
foo = bar         # revision = 2
foo1 = bar2       # revision = 3
foo = bar_new     # revision = 4
foo1 = bar1_new   # revision = 5
```

以下是访问以前版本 key 的示例：

```javascript
$ etcdctl get --prefix foo # 访问最新版本的 key
foo
bar_new
foo1
bar1_new

$ etcdctl get --prefix --rev=4 foo # 访问第 4 个版本的 key
foo
bar_new
foo1
bar1

$ etcdctl get --prefix --rev=3 foo #  访问第 3 个版本的 key
foo
bar
foo1
bar1

$ etcdctl get --prefix --rev=2 foo #  访问第 3 个版本的 key
foo
bar

$ etcdctl get --prefix --rev=1 foo #  访问第 1 个版本的 key
```

读取大于等于指定键的 byte 值的键 应用可能想读取大于等于指定键 的 byte 值的键。假设 etcd 集群已经有下列键：

```javascript
a = 123
b = 456
z = 789
```

读取大于等于键 b 的 byte 值的键的命令：

```javascript
$ etcdctl get --from-key b
b
456
z
789
```

删除键。客户端应用可以从 etcd 数据库中删除指定的键。假设 etcd 集群已经有下列键：

```javascript
foo = bar
foo1 = bar1
foo3 = bar3
zoo = val
zoo1 = val1
zoo2 = val2
a = 123
b = 456
z = 789
```

删除键 foo 的命令：

```javascript
$ etcdctl del foo
1 # 删除了一个键
```

删除从 foo to foo9 范围的键的命令：

```javascript
$ etcdctl del foo foo9
2 # 删除了两个键
```

删除键 zoo 并返回被删除的键值对的命令：

```javascript
$ etcdctl del --prev-kv zoo
1   # 一个键被删除
zoo # 被删除的键
val # 被删除的键的值
```

删除前缀为 zoo 的键的命令：

```javascript
$ etcdctl del --prefix zoo
2 # 删除了两个键
```

删除大于等于键 b 的 byte 值的键的命令：

```javascript
$ etcdctl del --from-key b
2 # 删除了两个键
```

###### **5.2.2 watch 历史改动**

watch 可以用来监测一个键值的变化，当该键值更新，控制台就会输出最新的值。例如：用户更新 watchkey 键值为 newwatchvalue。

```javascript
$ etcdctl watch  watchkey
# 在另外一个终端: etcdctl put  watchkey newwatchvalue
watchkey
newwatchvalue
```

从 foo to foo9 范围内键的命令：

```javascript
$ etcdctl watch foo foo9
# 在另外一个终端: etcdctl put foo bar
PUT
foo
bar
# 在另外一个终端: etcdctl put foo1 bar1
PUT
foo1
bar1
```

以 16 进制格式在键 foo 上进行观察的命令：

```javascript
$ etcdctl watch foo --hex
# 在另外一个终端: etcdctl put foo bar
PUT
\x66\x6f\x6f          # 键
\x62\x61\x72          # 值
```

观察多个键 foo 和 zoo 的命令：

```javascript
$ etcdctl watch -i
$ watch foo
$ watch zoo
# 在另外一个终端: etcdctl put foo bar
PUT
foo
bar
# 在另外一个终端: etcdctl put zoo val
PUT
zoo
val
```

查看 key 的历史修订版本。客户端应用需要获取某个键的所有修改。那么客户端应用连接到 etcd，watch 对应的 key 即可。如果 Watch 的过程中，etcd 或者客户端应用出错，又恰好发生了改动，这种情况下客户端应用可以在 Watch 时指定历史修订版本。 假设我们完成了下列操作序列：

```javascript
$ etcdctl put foo bar         # revision = 2
OK
$ etcdctl put foo1 bar1       # revision = 3
OK
$ etcdctl put foo bar_new     # revision = 4
OK
$ etcdctl put foo1 bar1_new   # revision = 5
OK
```

观察历史改动：

```javascript
# 从修订版本 2 开始观察键 `foo` 的改动
$ etcdctl watch --rev=2 foo
PUT
foo
bar
PUT
foo
bar_new
```

```javascript
# 在键 `foo` 上观察变更并返回被修改的值和上个修订版本的值
$ etcdctl watch --prev-kv foo
# 在另外一个终端: etcdctl put foo bar_latest
PUT
foo         # 键
bar_new     # 在修改前键 foo 的上一个值
foo         # 键
bar_latest  # 修改后键 foo 的值
```

压缩修订版本。etcd 保存了历史修订版本，客户端应用可以读取键的历史版本。大量的历史版本数据，会占据很多存储，因此需要压缩历史修订版本。经过压缩，etcd 会删除历史修订版本，释放出资源。压缩修订版本之前的版本数据不可访问。压缩修订版本的命令如下所示：

```javascript
$ etcdctl compact 5
compacted revision 5  $ etcdctl get --rev=4 foo
{"level":"warn","ts":"2020-05-04T16:37:38.020+0800","caller":"clientv3/retry_interceptor.go:62","msg":"retrying of unary invoker failed","target":"endpoint://client-c0d35565-0584-4c07-bfeb-034773278656/127.0.0.1:2379","attempt":0,"error":"rpc error: code = OutOfRange desc = etcdserver: mvcc: required revision has been compacted"}
Error: etcdserver: mvcc: required revision has been compacted
```

###### **5.2.3 租约**

授予租约 客户端应用可以为 etcd 数据库存储内的键授予租约。当 etcd 中的键被授予租约时，该键的存活时间与租约的时间绑定，而租约的存活时间相应的被 time-to-live (TTL)管理。在租约授予时每个租约的最小 TTL 值由客户端应用指定。当租约的 TTL 到期，即代表租约就过期，此时该租约绑定的键都将被删除。

```javascript
# 授予租约，TTL 为 100 秒
$ etcdctl lease grant 100
lease 694d71ddacfda227 granted with TTL(10s)

# 附加键 foo 到租约 694d71ddacfda227
$ etcdctl put --lease=694d71ddacfda227 foo10 bar
OK
```

建议时间设置久一点，否则来不及操作会出现如下的错误：

撤销租约 应用通过租约 id 可以撤销租约。撤销租约将删除所有它附带的 key。假设我们完成了下列的操作：

```javascript
$ etcdctl lease revoke 694d71ddacfda227
lease 694d71ddacfda227 revoked

$ etcdctl get foo10
```

刷新租期 应用程序可以通过刷新其 TTL 来保持租约活着，因此不会过期。

```javascript
$ etcdctl lease keep-alive 694d71ddacfda227
lease 694d71ddacfda227 keepalived with TTL(100)
lease 694d71ddacfda227 keepalived with TTL(100)
...
```

查询租期 应用程序可能想要了解租赁信息，以便它们可以续订或检查租赁是否仍然存在或已过期。应用程序也可能想知道特定租约所附的 key。

假设我们完成了以下一系列操作：

```javascript
$ etcdctl lease grant 300
lease 694d71ddacfda22c granted with TTL(300s)

$ etcdctl put --lease=694d71ddacfda22c foo10 bar
OK
```

获取有关租赁信息以及哪些 key 使用了租赁信息：

```javascript
$ etcdctl lease timetolive 694d71ddacfda22c
lease 694d71ddacfda22c granted with TTL(300s), remaining(282s)

$ etcdctl lease timetolive --keys 694d71ddacfda22c
lease 694d71ddacfda22c granted with TTL(300s), remaining(220s), attached keys([foo10])
```