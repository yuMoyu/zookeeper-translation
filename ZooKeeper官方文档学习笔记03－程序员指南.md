> 害，终究是我高估了自己，这个也太多了。不过里面有些思想，让我感觉似曾相识，比如他的session就有点像HTTPS，然后session的管理有点像人的管理

# 使用 ZooKeeper 开发分布式应用程序
# 引言
本指南前四部分是概念讨论（一致性保证及以前），后四部分是实用的编程信息。
# 数据模型
前文提到的分层名称空间，这里多写了一些命名规则。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210429102554778.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
##  Znodes
Znode包含一些基本信息可以帮他验证缓存并协调更新。Znode的数据发生变化时，它也要进行更新。
#### 名词解释
+ 1 Znode：数据节点
+ 2 Server：ZooKeeper服务的机器
+ 3 quorum peers：组成整体的服务器，像之前的zServer.cmd后启动的
+ 4 client：ZooKeeper服务的任何主机或进程，像之前的zCli.cmd后启动的
### Watches
>这个应该翻译成啥呢？监控？监视

客户端可以在znode上设置Watch。对该znode的更改会触发Watch，然后清除Watch。当Watch触发后，ZooKeeper会向客户端发送一个通知。
### 数据读写
Znode中数据的的读写是原子性的。读的时候会读取与一个znode关联的所有数字字节，写替换所有数据。每一个节点都有访问控制列表（ACL），用于限制谁做什么。
znode的数据少于1M，因此如果数据量大可能会造成某些操作的延迟。所以，如果需要大数据存储，则处理此类数据的通常方式是将其存储在大容量存储系统上。
### 临时节点
仅存在于会话期，不允许有子节点
### 序列节点--唯一命名
给Znode搞一个唯一标识类似ID。方法：创建Znode时，请求ZooKeeper在路径的末尾附加一个递增的计数器。该计数器对于父znode唯一，个hi是为%010d，即“000000000001”，计数器超过2147483647时溢出。
### 容器节点
特殊的Znode，容器节点是为leader、lock等操作存在的，当容器的子容器均被删除时，该容器将成为服务器将来要删除的候选容器。在容器znode内创建子znode时，请始终检查KeeperException.NoNodeException并在发生时重新创建容器znode。
### TTL节点
>3.5.3中添加

在TTL时长内，节点没有被修改也没有子节点，那么它将成为服务器未来某个时候被删除的候选节点。
+ 1 节点： PERSISTENT 或 PERSISTENT _ sequential znodes 
+ 2 TTL设置：在创建节点时设置，单位毫秒
+ 3 默认禁用，需要通过“系统”属性启动
+ 4 如果设置不正确就创建TTL节点，服务器会抛出KeeperException异常
### ZooKeeper的时间
Zookeeper跟踪记录时间的方式：
+ 1 zxid：ZooKeeper状态的每一次改变, 都对应着一个递增的Transaction id, 该id称为zxid
+ 2 Version numbers：对节点的每次更改都会导致该节点版本号增加1，包括version、cversion、aversion
+ 3 Ticks：当使用多节点的ZooKeeper时，ticks表示事件的次数，事件包括：状态上载、session超时、节点间的连接超时等。
+ 4 Real time：在创建znode和修改znode时将时间戳记入stat结构中
### ZooKeeper统计结构
ZooKeeper的znode的Stat结构由以下字段组成：
+ 1 czxid：创建节点时的zxid
+ 2 mzxid：修改节点时的zxid
+ 3 pzxid：最后一次修改此znode子级的更改的zxid。
+ 4 ctime：创建此znode时从纪元开始的时间（以毫秒为单位）。
+ 5 mtime：自上次修改此znode以来的时间（以纪元为单位）。
+ 5 version：版本此znode的数据更改数。
+ 6 cversion：此znode的子级更改的数量。
+ 7 aversion：此znode的ACL的更改数。
+ 8 ephemeralOwner：如果znode是一个临时节点，则此znode的所有者的会话ID。如果它不是临时节点，则它将为零。
+ 9 dataLength：此znode的数据字段的长度。
+ 10 numChildren：此znode的子级数。
> 今天就先到这里了！
# ZooKeeper会话(session)
+ 1 Start：客户端尝试创建一个和zk服务端连接的句柄——session，放入requests queued（排队？后半句，我猜的）
+ 2 CONECTING：session创建成功后（CONNECTED event），处于CONNECTING状态，客户端尝试连接服务器
+ 3 CONNECTED：客户端连接上服务器的时候处于CONNECTED状态
+ 4 DISCONNECTED event pending requests return CONNECTION_LOSS：断开连接，挂起请求，返回CONNECTION_LOSS，并变成CONNECTING状态
+ 5 无法恢复的故障会导致变成CLOSE状态
（1）AUTH_FAILED event （认证失败）
（2）SESSION_EXPIRED（会话过期）
（3）close called（客户端关闭连接）
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210511154338891.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
## 会话创建
+ 1 代码中必须提供连接字符串——`host:port`（如：127.0.0.1:4545），客户端随机选择服务器连接，断开时再选择其他连接。
+ 2 （3.2增加的）`host:port`后面加了个后缀（如：127.0.0.1:4545/app/a），此时访问的/foo/bar节点，其实是zk服务端的/app/a/foo/bar节点，多应用共享zookeeper服务器时比较方便。
+ 3 sessionID：zk服务端生成的64位的数字（就像数据表的唯一标识），交由zk客户端保存，为了更加安全，同时传给客户端一个密码。客户端重连服务器时需要传递这两样东西。
>（有点像之前学的加密技术，对称加密什么的）
>有点像[HTTPS](https://blog.csdn.net/moqianmoqian/article/details/104880066),下面的超时时间选择也有点像

+ 4 超时时间（毫秒）：客户端发给服务端一个超时时间，服务端会返回一个他可以接受的值给客户端。标准其实就是tickTime*2 <= session timeout <= tickTime*20。
> + 1 超时时间内，会自动重连，连接上session状态也会变为CONNECTED。超出此时间，session会变成expired（zk依然会自动重连）。只有当收到会话过期通知时才需要手动创建新的会话
> + 2 session过期与否，交由zk服务器集群判断。若zk服务器集群在超时时间内未收到zk客户端的心跳（你可以把它想想成一个人的死亡与复活，zk服务器集群就是那个管理部门），就会把这个客户端标记未过期，然后删除session创建的所有临时节点，通知监听了这些节点的其他session。如果断开的session再次连接成功，会收到被标记为过期的提醒。

+ 5 默认观察者：当客户机中发生任何状态更改时，观察员都会得到通知。
> 当空闲一段时间会话超时时(qq离线？)，客户端将发送PING请求以使会话保持活动状态（让服务器知道客户端活跃，让客户端知道和服务器之间的连接还在。）

## 更新服务器列表
客户端可以使用一个新的连接字符串来更新服务列表（增加或删除服务器）。此机制会调用负载均衡算法，平衡客户端与服务器的连接情况（一些客户端断连，然后换其他的服务器重连，最后结果是每个服务器连接的客户端数都几乎一样）
# ZooKeeper 监视(Watches)
Watch是一次性（类似一次性筷子，只能用一次）触发器，数据发生变化时向设置了watch的客户端发送消息（ZooKeeper提供算法保证不同客户机看到的顺序一致）。
> 顺序问题可以参照一下 下面的zookeeper对watches的担保？

getData（）、getChildren（）、exists（）可以设置此触发器，其中getData和exists是监视节点的，getChildren是监视子节点列表的。
这些watch都是在服务器上维护的，所以当客户端重新连接的时候，watch会重新注册并可以触发。
## 创建Watches
exists, getData, 以及getChildren。各创建方法可以触发的watch:
exists：Created event、Deleted event、Changed event
getData：Deleted event、Changed event
getChildren：Deleted event、Child event
> 不是太理解，也不知道我的这个理解对不对，我感觉这个得结合代码理解一下？

## 移除Watches
可以通过removeWatches来移除注册在znode的watch。也可以通过zk客户端将local标志设为true来移除本地watch。（即使没有服务器连接）
>watch不是在服务器吗，为啥又移除本地，有点疑惑

1）子节点移除会触发：getChildren
2）数据删除会触发：exists或者getData
## ZooKeepr对watches的保证
+ 1 确保watch分发有序
+ 2 客户端先看到watch事件，再看到znode新数据。
+ 3 watch 事件的顺和zk服务器收到的更新顺序一致
## watches注意点
+ 1 watch是一次性的，用一次设置一次
+ 2 watch存在延迟，会造成空窗期？
> 你因为一个节点触发了watch，它还在发消息的路上。但此时你又想更改这个节点的数据，又设置了watch。但是可能会数据更改了，新watch还没有设置好。

+ 3 watch每次每个对象只能设置一次，设置多了也只会触发一个
+ 4 与服务器断开连接时，会话事件发送给未完成的watch（*这就是客户端和服务器重新连接可以收到与服务器断联期间消息的原因吗？*），并进入安全模式。重新连接会收到watch。 
