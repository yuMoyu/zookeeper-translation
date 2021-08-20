>纠结了很久，我决定用官方文档学习


# 学习文档
+ 1 [中文翻译](https://duoke360.com/course/zookeeper-docs/zookeeperOver)
+ 2 [官方文档](https://zookeeper.apache.org/doc/r3.5.5/zookeeperOver.html)
# 学习计划
打算两天学一篇（周末、节假日不学），正常应该5月31号学完，但考虑到我比较菜，给10天冗余（是冗余吧？）时间，6月14号学完，目录层级按照官方文章来。
左边的翻译工具是彩云小译
zookeeper版本：r3.5.5
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210426160601488.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)

# ZooKeeper：分布式应用程序的分布式协调服务
&nbsp;&nbsp;&nbsp;&nbsp;ZooKeeper定义了一些规则，然后让程序可以基于这些规则去实现用于同步，配置维护以及组和命名的更高级别的服务。它的目的是为了解决协调服务中的问题，如比赛条件（多个线程共享同个资源？）和死锁。
## 设计目标
ZooKeeper 数据保存在内存中，帮助ZooKeeper 可以实现高吞吐量和低延迟。
## 数据模型和分层名称空间
Zookeeper名称空间类似标准文件系统，如图所示。Zookeeper的名称空间中的每个节点都由一个路径标识。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210426155856287.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
## 节点和短命节点
Zookeeper的每个节点（Znode）就行[myBase](http://www.wjjsoft.com/mybase_cn.html)软件一样，可以当作一个文件，同时也可以在这个文件下创造文件（当作一个目录）。
ZooKeeper的短命节点会话结束就会被删除。
## 有条件的更新和监视
客户端可以在znode上设置监视。znode更改时，将触发并删除监视。（不太理解？？）
## 保证
ZooKeeper为了构建复杂的服务而设置的的一些基本特性：
+ 顺序一致性：来自客户端的更新将按照它们发送的顺序应用。
+ 原子性：更新成功或失败。没有部分结果。
+ 单一视图：客户机将看到服务的相同视图，而不管它连接到哪个服务器。
+ 可靠性：一旦更新被应用，它将从那时起一直持续，直到客户端覆盖更新。
+ 及时性：系统的客户端视图保证在一定的时间范围内是最新的。
## 简单的 API
+ create 创建节点
+ delete 删除节点
+ exists节点存在与否判断
+ get 获取节点数据
+ set 为节点赋值
+ get children 获取节点的子节点
+ sync（同步吗？）

> 目前来说，这个功能，我感觉有点像，java的list数组
## 执行
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210427104646809.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
> 这副图没有太看懂

我现在知道的是：
+ Replicated Database 是复制的数据库，它是包含整个数据树的内存数据库。作用是更新记录到磁盘以确保可恢复性
+ Atomic Broadcast 原子消息传递协议
+ Read Request 读取请求每个服务器数据库的本地副本提供服务
## 用途
简单的接口完成复杂的操作
## 性能
高性能（协调服务，多少读取代码写入）
## 可靠性
+ 跟随者失败且迅速恢复，ZooKeeper可以维持高吞吐量
+ 领导者选举算法可以帮助系统快速恢复
> 回顾解决问题：
> 领导者选举算法是什么？
> 领导者恢复了，为了有利于观察者呢？

