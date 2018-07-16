# Redis

Redis 是一个开源（BSD许可）的内存数据结构存储系统，它可以用作数据库、缓存和消息中间件。

- 官网：http://redis.io/
- 中文网：http://redis.cn/

当前最新稳定版本是 Redis 4.0.9。

## 安装

单机版安装非常简单。

**1. 创建管理用户：**

```
# 创建用户
[root@centos ~]# useradd redis

# 设置密码
[root@centos ~]# passwd redis
```

**2. 登录到管理用户：**

```
[root@centos ~]# su - redis
[redis@centos ~]$ pwd
/home/redis
```

**3. 下载安装包：**

[redis-4.0.9.tar.gz](http://redis.cn/download.html)

**4. 解压并安装：**

```
[redis@centos ~]$ tar -zxvf redis-4.0.9.tar.gz
[redis@centos ~]$ cd redis-4.0.9/

# 非root用户不可执行 `make install` 命令
[redis@centos redis-4.0.9]$ make
```

**5. 启动服务：**

```
[redis@centos ~]$ ./redis-4.0.9/src/redis-server ./redis-4.0.9/redis.conf &
```

**6. 命令行工具：**

```
[redis@centos ~]$ ./redis-4.0.9/src/redis-cli -p 6379
```

Redis 单机版安装完成！

## 配置

一般情况上，建议使用Redis集群部署方案。这里介绍 Redis 配置文件的主要参数：

- 配置参考：https://www.cnblogs.com/pqchao/p/6558688.html
- 持久化配置：https://www.cnblogs.com/xiaoxi/p/7065328.html

```
# 允许访问的IP地址
bind 127.0.0.1

# 服务端口号
port 6379

# 后台运行
daemonize no

# PID文件路径
pidfile /var/run/redis_6379.pid

# DB文件名
dbfilename dump.rdb

# 工作目录
dir ./

# 授权密码（建议开启）
requirepass foobared
```

## 集群搭建

http://www.cnblogs.com/hjwublog/category/848303.html

集群规划（3机6节点，也就是三主三从模式）。

需要注意的是，集群中的每个节点都会涉及到两个端口，一个用于客户端操作（6379,6380），一个用于集群各个节点间通信（+10000: 16379,16380）。

> 此次安装将10.7.112.125作为集群控制端，也就是用于创建集群的节点，需要[安装ruby环境](./RUBY.md)。

| ip | node |
|:---|:---|
| 10.7.112.125 | 6379, 6380 |
| 10.7.112.126 | 6379, 6380 |
| 10.7.112.126 | 6379, 6380 |

**1. 创建redis管理用户：**

```
# 创建用户
[root@centos ~]# useradd redis

# 设置密码
[root@centos ~]# passwd redis
```

**2. 登录到管理用户：**

```
[root@centos ~]# su - redis
[redis@centos ~]$ pwd
/home/redis
```

**3. 解压安装：**

```
[redis@centos ~]$ tar -zxvf redis-4.0.9.tar.gz
[redis@centos ~]$ cd redis-4.0.9/
[redis@centos redis-4.0.9]$ make
```

**4. 创建集群配置文件：**

```
[redis@centos ~]$ mkdir -p cluster/6379 cluster/6380
[redis@centos ~]$ cp /home/redis/redis-4.0.9/redis.conf /home/redis/cluster/6379
[redis@centos ~]$ cp /home/redis/redis-4.0.9/redis.conf /home/redis/cluster/6380
```

**5. 编辑配置文件：**

```
## 6379
[redis@centos ~]$ vi /home/redis/cluster/6379/redis.conf
# bind 127.0.0.1
protected-mode no
port 6379
daemonize yes
pidfile redis_6379.pid
dbfilename dump.rdb
dir /home/redis/cluster/6379/
cluster-enabled yes
cluster-config-file nodes-6379.conf
cluster-node-timeout 15000

## 6380
[redis@centos ~]$ vi /home/redis/cluster/6380/redis.conf
# bind 127.0.0.1
protected-mode no
port 6380
daemonize yes
pidfile redis_6380.pid
dbfilename dump.rdb
dir /home/redis/cluster/6380/
cluster-enabled yes
cluster-config-file nodes-6380.conf
cluster-node-timeout 15000
```

**6. 复制到另外两台机器：**

```
[redis@centos ~]$ scp -r /home/redis/cluster/ redis@10.7.112.126:/home/redis
[redis@centos ~]$ scp -r /home/redis/cluster/ redis@10.7.112.127:/home/redis
```

**7. 启动所有服务：**

```
[redis@centos ~]$ cd /home/redis/
[redis@centos ~]$ ./redis-4.0.9/src/redis-server ./cluster/6379/redis.conf
[redis@centos ~]$ ./redis-4.0.9/src/redis-server ./cluster/6380/redis.conf
```

**8. 创建集群（注意：要关闭防火墙）：**

```
[redis@centos ~]$ ./redis-4.0.9/src/redis-trib.rb create --replicas 1 10.7.112.125:6379 10.7.112.125:6380 10.7.112.126:6379 10.7.112.126:6380 10.7.112.127:6379 10.7.112.127:6380
```

**9. 查看集群状态（可连接任意节点）：**

```
[redis@centos ~]$ ./redis-4.0.9/src/redis-cli -c -h 10.7.112.125
10.7.112.125:6379> CLUSTER NODES
c598542698a736c5f7cc938530d8fa2e84a46f6e 10.7.112.126:6379@16379 master - 0 1524851892502 3 connected 5461-10922
db1770011399266b25b5ce616776ce08dbc7879b 10.7.112.125:6380@16380 slave 75c7dcbb357811a7de146bd83b82239dee0afcfe 0 1524851893303 5 connected
81bfbd435e7dd95d7d827b9f4526b87b564e0ce1 10.7.112.126:6380@16380 slave 01d97c4d1a39662266cf842090719ca37ebfc34b 0 1524851893000 4 connected
01d97c4d1a39662266cf842090719ca37ebfc34b 10.7.112.125:6379@16379 myself,master - 0 1524851889000 1 connected 0-5460
4255521aaf1e3e019f328b81ed8e948d43e9831e 10.7.112.127:6380@16380 slave c598542698a736c5f7cc938530d8fa2e84a46f6e 0 1524851893805 6 connected
75c7dcbb357811a7de146bd83b82239dee0afcfe 10.7.112.127:6379@16379 master - 0 1524851892502 5 connected 10923-16383
```

根据状态可知，集群中服务之间的关系：

| master | slave |
|:---|:---|
| 10.7.112.125:6379 | 10.7.112.126:6380 |
| 10.7.112.126:6379 | 10.7.112.127:6380 |
| 10.7.112.127:6379 | 10.7.112.125:6380 |

**总结**：Redis 集群实际上是通过hash slots 的方式实现负载均衡（总共16384个slot），采用了非一致hash，使得集群节点可动态增加或减少，通过key计算slot，然后将值存储到对应的slot关联的机器上。

## 集群注意事项

Redis 集群相对单机在功能上存在一些限制，在使用时做好规避。限制如下：

1. key批量操作支持有限。如mset、mget，目前只支持具有相同slot值的
key执行批量操作。对于映射为不同slot值的key由于执行mget、mget等操作可
能存在于多个节点上因此不被支持。
2. key事务操作支持有限。同理只支持多key在同一节点上的事务操
作，当多个key分布在不同的节点上时无法使用事务功能。
3. key作为数据分区的最小粒度，因此不能将一个大的键值对象如
hash、list等映射到不同的节点。
4. 不支持多数据库空间。单机下的Redis可以支持16个数据库，集群模
式下只能使用一个数据库空间，即db0。
5. 复制结构只支持一层，从节点只能复制主节点，不支持嵌套树状复
制结构。

## redis-trib.rb

https://blog.csdn.net/huwei2003/article/details/50973967

redis-trib.rb 是 redis 官方推出的管理 redis 集群的工具，将集群命令封装成简单、便捷、实用的操作工具。该脚本是基于 ruby 编写的，因此，需要在集群的某一台机器上安装 ruby 环境。

```
[redis@centos ~]$ ruby ./redis-4.0.9/src/redis-trib.rb help
Usage: redis-trib <command> <options> <arguments ...>

  create          host1:port1 ... hostN:portN
                  --replicas <arg>
  check           host:port
  info            host:port
  fix             host:port
                  --timeout <arg>
  reshard         host:port
                  --from <arg>
                  --to <arg>
                  --slots <arg>
                  --yes
                  --timeout <arg>
                  --pipeline <arg>
  rebalance       host:port
                  --weight <arg>
                  --auto-weights
                  --use-empty-masters
                  --timeout <arg>
                  --simulate
                  --pipeline <arg>
                  --threshold <arg>
  add-node        new_host:new_port existing_host:existing_port
                  --slave
                  --master-id <arg>
  del-node        host:port node_id
  set-timeout     host:port milliseconds
  call            host:port command arg arg .. arg
  import          host:port
                  --from <arg>
                  --copy
                  --replace
  help            (show this help)
```

具体功能简述：

- create：创建集群
- check：检查集群
- info：查看集群信息
- fix：修复集群
- reshard：在线迁移slot
- rebalance：平衡集群节点slot数量
- add-node：将新节点加入集群
- del-node：从集群中删除节点
- set-timeout：设置集群节点间心跳连接的超时时间
- call：在集群全部节点上执行命令
- import：将外部redis数据导入集群

### create

创建集群。

- `--replicas`：表示需要有几个slave。

```
# 创建集群（无slave）
[redis@centos ~]$ ruby ./redis-4.0.9/src/redis-trib.rb create 10.7.112.125:6379 10.7.112.126:6379 10.7.112.127:6379

# 创建集群（有一个slave）
[redis@centos ~]$ ruby ./redis-4.0.9/src/redis-trib.rb create --replicas 1 10.7.112.125:6379 10.7.112.125:6380 10.7.112.126:6379 10.7.112.126:6380 10.7.112.127:6379 10.7.112.127:6380 
```

### check

检查集群状态。

```
[redis@centos ~]$ ruby ./redis-4.0.9/src/redis-trib.rb check 10.7.112.125:6379
```

### info

查看集群信息。

```
[redis@centos ~]$ ruby ./redis-4.0.9/src/redis-trib.rb info 10.7.112.125:6379
```

### fix

修复集群。

```
[redis@centos ~]$ ruby ./redis-4.0.9/src/redis-trib.rb fix 10.7.112.125:6379
```

### reshard

把集群的一些slot从集群原来slot负责节点迁移到新的节点，利用reshard可以完成集群的在线横向扩容和缩容。

- `host:port`：用来从一个节点获取整个集群信息，相当于获取集群信息的入口。
- `--from <arg>`：需要从哪些源节点上迁移slot，可从多个源节点完成迁移，用逗号隔开，传递的是节点的node id，还可以直接传递--from all，这样源节点就是集群的所有节点，不传递该参数的话，则会在迁移过程中提示用户输入。
- `--to <arg>`：slot需要迁移的目的节点的node id，目的节点只能填写一个，不传递该参数的话，则会在迁移过程中提示用户输入。
- `--slots <arg>`：需要迁移的slot数量，不传递该参数的话，则会在迁移过程中提示用户输入。
- `--yes`：设置该参数，可以在打印执行reshard计划的时候，提示用户输入yes确认后再执行reshard。
- `--timeout <arg>`：设置migrate命令的超时时间。
- `--pipeline <arg>`：定义cluster getkeysinslot命令一次取出的key数量，不传的话使用默认值为10。

```
[redis@centos ~]$ ruby ./redis-4.0.9/src/redis-trib.rb reshard --from all --to 494e7eed4052bc0ade9621c5657b70bf83e50444 --slots 1000 10.7.112.125:6379
```

### rebalance

根据用户传入的参数平衡集群节点的slot数量。

- `host:port`：用来从一个节点获取整个集群信息，相当于获取集群信息的入口。
- `--weight <arg>`：节点的权重，格式为`node_id=weight`，如果需要为多个节点分配权重的话，需要添加多个--weight <arg>参数，即--weight b31e3a2e=5 --weight 60b8e3a1=5，node_id可为节点名称的前缀，只要保证前缀位数能唯一区分该节点即可。没有传递–weight的节点的权重默认为1。
- `--threshold <arg>`：只有节点需要迁移的slot阈值超过threshold，才会执行rebalance操作。
- `--use-empty-masters`：rebalance是否考虑没有节点的master，默认没有分配slot节点的master是不参与rebalance的，设置--use-empty-masters可以让没有分配slot的节点参与rebalance。
- `--timeout <arg>`：设置migrate命令的超时时间。
- `--simulate`：设置该参数，可以模拟rebalance操作，提示用户会迁移哪些slots，而不会真正执行迁移操作。
- `--pipeline <arg>`：与reshar的pipeline参数一样，定义cluster getkeysinslot命令一次取出的key数量，不传的话使用默认值为10。

```
[redis@centos ~]$ ruby ./redis-4.0.9/src/redis-trib.rb rebalance --threshold 1 --weight 083b39d746ece38440f7e950b1a0e775a62a5995=5 --weight 65838f51f279dbb8dd421f0ed5d9f6724a53c5fd=5 --use-empty-masters --simulate 10.7.112.125:6379
```

### add-node

将新节点加入集群，节点可以为master，也可以为某个master节点的slave。

- `--slave`：设置该参数，则新节点以slave的角色加入集群。
- `--master-id`：这个参数需要设置了--slave才能生效，--master-id用来指定新节点的master节点。如果不设置该参数，则会随机为节点选择master节点。

```
# 1. 添加新的master节点（添加之后需要执行reshard分配slots，才能正常提供服务。）
[redis@centos ~]$ ruby ./redis-4.0.9/src/redis-trib.rb add-node 10.7.112.124:6379 10.7.112.125:6379

# 2. 添加新的slave节点
[redis@xbird-dev ~]$ ruby ./redis-4.0.9/src/redis-trib.rb add-node --slave --master-id 494e7eed4052bc0ade9621c5657b70bf83e50444 10.7.112.124:6380 10.7.112.125:6379
```

### del-node

把某个节点从集群中删除（del-node只能删除没有分配slot的节点，如果要删除某个节点，可以先使用reshard将slots迁移到其他节点）。

```
[redis@centos ~]$ ruby ./redis-4.0.9/src/redis-trib.rb del-node 10.7.112.124:6380 0b426834eb1d8af3cf7f1b8fc401ae2d85eee90e
```

### set-timeout

设置集群节点间心跳连接的超时时间，单位是毫秒，不得小于100毫秒，因为100毫秒对于心跳时间来说太短了。该命令修改是节点配置参数cluster-node-timeout，默认是15000毫秒。

```
[redis@centos ~]$ ruby ./redis-4.0.9/src/redis-trib.rb set-timeout 10.7.112.125:6379 30000
```

### call

在集群的全部节点执行相同的命令。需要通过集群的一个节点地址，连上整个集群，然后在集群的每个节点执行该命令。

```
[redis@centos ~]$ ruby ./redis-4.0.9/src/redis-trib.rb call 10.7.112.125:6379 get key
```

### import

把外部的redis节点数据导入集群。import命令更适合离线的把外部redis数据导入，在线导入的话最好使用更专业的导入工具，以slave的方式连接redis节点去同步节点数据应该是更好的方式。

```
# 将 10.7.112.120:6379 上的数据导入到 10.7.112.125:6379 节点所在的集群
[redis@centos ~]$ ruby ./redis-4.0.9/src/redis-trib.rb import --from 10.7.112.120:6379 10.7.112.125:6379
```

## 开发规范

**1. 冷热数据分离，不要将所有数据全部都放到Redis中**

虽然Redis支持持久化，但是Redis的数据存储全部都是在内存中的，成本昂贵。建议根据业务只将高频热数据存储到Redis中【QPS大于5000】，对于低频冷数据可以使用 MySQL/ElasticSearch/MongoDB 等基于磁盘的存储方式，不仅节省内存成本，而且数据量小在操作时速度更快、效率更高！

**2. 不同的业务数据要分开存储**

不要将不相关的业务数据都放到一个Redis实例中，建议新业务申请新的单独实例。因为Redis为单线程处理，独立存储会减少不同业务相互操作的影响，提高请求响应速度；同时也避免单个实例内存数据量膨胀过大，在出现异常情况时可以更快恢复服务！

**3. 存储的Key一定要设置超时时间**

如果应用将Redis定位为缓存Cache使用，对于存放的Key一定要设置超时时间！因为若不设置，这些Key会一直占用内存不释放，造成极大的浪费，而且随着时间的推移会导致内存占用越来越大，直到达到服务器内存上限！另外Key的超时长短要根据业务综合评估，而不是越长越好！

**4. 对于必须要存储的大文本数据一定要压缩后存储**

对于大文本【超过500字节】写入到Redis时，一定要压缩后存储！大文本数据存入Redis，除了带来极大的内存占用外，在访问量高时，很容易就会将网卡流量占满，进而造成整个服务器上的所有服务不可用，并引发雪崩效应，造成各个系统瘫痪！

**5. 线上Redis禁止使用Keys正则匹配操作**

Redis是单线程处理，在线上KEY数量较多时，操作效率极低【时间复杂度为O(N)】，该命令一旦执行会严重阻塞线上其它命令的正常请求，而且在高QPS情况下会直接造成Redis服务崩溃！如果有类似需求，请使用scan命令代替！

**6. 可靠的消息队列服务**

Redis List经常被用于消息队列服务。假设消费者程序在从队列中取出消息后立刻崩溃，但由于该消息已经被取出且没有被正常处理，那么可以认为该消息已经丢失，由此可能会导致业务数据丢失，或业务状态不一致等现象发生。为了避免这种情况，Redis提供了RPOPLPUSH命令，消费者程序会原子性的从主消息队列中取出消息并将其插入到备份队列中，直到消费者程序完成正常的处理逻辑后再将该消息从备份队列中删除。同时还可以提供一个守护进程，当发现备份队列中的消息过期时，可以重新将其再放回到主消息队列中，以便其它的消费者程序继续处理。

**7. 谨慎全量操作Hash、Set等集合结构**

在使用HASH结构存储对象属性时，开始只有有限的十几个field，往往使用HGETALL获取所有成员，效率也很高，但是随着业务发展，会将field扩张到上百个甚至几百个，此时还使用HGETALL会出现效率急剧下降、网卡频繁打满等问题【时间复杂度O(N)】。建议根据业务拆分为多个Hash结构；或者，如果大部分都是获取所有属性的操作，可以将所有属性序列化为一个STRING类型存储！同样在使用SMEMBERS操作SET结构类型时也是相同的情况！

**8. 根据业务场景合理使用不同的数据结构类型**

目前Redis支持的数据库结构类型较多：字符串（String），哈希（Hash），列表（List），集合（Set），有序集合（Sorted Set）, Bitmap, HyperLogLog和地理空间索引（geospatial）等。需要根据业务场景选择合适的类型，常见的如：

- String可以用作普通的K-V、计数类；
- Hash可以用作对象如商品、经纪人等，包含较多属性的信息；
- List可以用作消息队列、粉丝/关注列表等；
- Set可以用于推荐；
- Sorted Set可以用于排行榜。

**9. 命名规范**

Redis 的 key 命名尽量简单明确，容易阅读理解，如：`系统名_业务名_标识` => `QR_PREPAY_[ID]`

**10. 线上禁止使用monitor命令**

禁止生产环境使用monitor命令，monitor命令在高并发条件下，会存在内存暴增和影响Redis性能的隐患。

**11. 禁止大string**

核心集群禁用1mb的string大key(虽然redis支持512MB大小的string)，如果1mb的key每秒重复写入10次，就会导致写入网络IO达10MB。

**12. 容量控制**

单实例的内存大小不建议过大，建议在10~20GB以内。Redis实例包含的键个数建议控制在1kw内，单实例键个数过大，可能导致过期键的回收不及时。

## 资源

- [唯品会Redis cluster大规模生产实践](https://www.cnblogs.com/thrillerz/p/5604752.html)
- [阿里云redis开发规范](https://yq.aliyun.com/articles/531067)
- [Redis 使用规范](https://blog.csdn.net/zdx_csdn/article/details/70807663)

## 问题列表

- [Redis上踩过的一些坑-美团](https://blog.csdn.net/chenleixing/article/details/50530419)