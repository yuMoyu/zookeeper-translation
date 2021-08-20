> 本来以为学一篇都会很难很难，但是好像也没有那么难。虽然有些名词不太理解，但我决定后续学习中应该会遇到吧？

# 入门：使用ZooKeeper协调分布式应用程序
## 先决条件
> 还没有看

请参阅管理指南中的系统要求。

## 下载
[下载链接](https://zookeeper.apache.org/releases.html)
Apache ZooKeeper 3.7.0 是编译好的版本，带有各种jar包。另一个Release版本还要安装jar，因此我们这里用Apache ZooKeeper 3.7.0。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210427112632225.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
## 独立运行
[参考链接](https://blog.csdn.net/qq_33316784/article/details/88563482)
##### 1 选择一个合适的目录，然后用winzip解压即可
#####  2 创建zoo.cfg
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210427140315371.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
```c
tickTime=2000
dataDir=/var/lib/zookeeper
clientPort=2181
```
（1）tickTime：服务器之间或客户端与服务器之间维持心跳的时间间隔。单位毫秒，最小会话超时是tickTime的两倍
（2）dataDir：保存数据，如果没有指明写数据的日志的存放地址，也会放在这里
（3）dataLogDir：因此设置这个，存放日志
（4）clientPort：客户端连接 Zookeeper 服务器的端口
##### 3 创建空的数据和日志存放地址并修改zoo.cfg
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210427141432689.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)

```c
tickTime=2000
dataDir=D:\kj\ZooKeeper\apache-zookeeper-3.7.0-bin\data
dataLogDir=D:\kj\ZooKeeper\apache-zookeeper-3.7.0-bin\log
clientPort=2181
```

##### 4 尝试启动（服务器）（单机版）
双击zkServer.cmd
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210427141613848.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)

> 如果闪退，可以右键编辑zkServer.cmd，在末尾输入pause，保存推书，再次运行，查看报错原因。[参考链接](https://blog.csdn.net/sky198989/article/details/82778957)
##### 5 看一下是否启动成功
cmd输入`netstat -ano`
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210427143019162.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
## 管理ZooKeeper存储
先空着~
## 连接到ZooKeeper（客户端）（单机版）
双击zkCli.cmd
看起来是失败了，找一下原因
```c
Connecting to localhost:2181
......
2021-04-27 15:44:54,291 [myid:] - INFO  [main:ClientCnxn@1726] - zookeeper.request.timeout value is 0. feature enabled=false
Welcome to ZooKeeper!
2021-04-27 15:44:54,299 [myid:localhost:2181] - INFO  [main-SendThread(localhost:2181):ClientCnxn$SendThread@1171] - Opening socket connection to server localhost/0:0:0:0:0:0:0:1:2181.
2021-04-27 15:44:54,299 [myid:localhost:2181] - INFO  [main-SendThread(localhost:2181):ClientCnxn$SendThread@1173] - SASL config status: Will not attempt to authenticate using SASL (unknown error)
JLine support is enabled
[zk: localhost:2181(CONNECTING) 0] 2021-04-27 15:44:56,323 [myid:localhost:2181] - WARN  [main-SendThread(localhost:2181):ClientCnxn$SendThread@1290] - Session 0x0 for sever localhost/0:0:0:0:0:0:0:1:2181, Closing socket connection. Attempting reconnect except it is a SessionExpiredException.
java.net.ConnectException: Connection refused: no further information
        at sun.nio.ch.SocketChannelImpl.checkConnect(Native Method)
        at sun.nio.ch.SocketChannelImpl.finishConnect(SocketChannelImpl.java:717)
        at org.apache.zookeeper.ClientCnxnSocketNIO.doTransport(ClientCnxnSocketNIO.java:344)
        at org.apache.zookeeper.ClientCnxn$SendThread.run(ClientCnxn.java:1280)

```


经过我种种查找，最后发现是我在点开zkCli.cmd的时候把zkServer.cmd关了。他们一个服务器，一个客户端，服务器关了，客户端自然连接不上。嘤嘤嘤，还是太菜惹的祸。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210427160703546.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
##### 一些指令
+ 1 help
获取客户端命令列表
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210428085341792.png)
+ 2 创建新节点
（1）`create/zk _ test my _ data`：创建新的节点(znode)，zk_test，并将其与字符串"my_data"关联
（2）`ls /`：查看目录
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210428085731732.png)
+ 3 获取节点数据
`get /zk_test`
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210428090554871.png)
这个和官网的图不一样欸，官网可以获取到一长串信息（如下所示），我们来找找原因。

```c
[zkshell: 12] get /zk_test
my_data
cZxid = 5
ctime = Fri Jun 05 13:57:06 PDT 2009
mZxid = 5
mtime = Fri Jun 05 13:57:06 PDT 2009
pZxid = 5
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0
dataLength = 7
numChildren = 0
```
（1）我在想是不是版本问题，于是取下载了一个zookeeper-3.4.13
然后准备创建节点，同一个名字。
（2）首先，发现他们用的同一个zookeeper目录（为什么呢？以后遇到再说）。
（3）然后，我就get
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210428092050726.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
所以还真的是版本问题
（4） 如果想在3.7.0上看版本信息的话，可以用：`stat /zk_test`
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210428092127761.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
+ 4 删除节点
`delete /zk_test`
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210428094210814.png)
更多详见[程序员指南](https://zookeeper.apache.org/doc/r3.5.5/zookeeperProgrammers.html)
## ZooKeeper编程
ZooKeeper有一个Java绑定和c绑定，功能上等价。不同之处在于消息传递循环是如何完成的。详见[程序员指南](https://zookeeper.apache.org/doc/r3.5.5/zookeeperProgrammers.html)。
## 集群模式下运行ZooKeeper
单机的ZooKeeper适用于评估开发和测试。但是生产环境应该在集群模式下运行ZooKeeper。ZooKeeper集群中复制的服务器组称为法定人数 quorum, 法定人数中的所有服务器都有一样的配置文件。
##### Note
+ 1 集群模式，服务器数量至少为3台，且强烈建议使用奇数个服务器。因为如果只有两台服务器，其中一台服务器发生故障时，就剩了一台服务器，就没法形成多数派。（选举用？）
+ 2 在一台机器上设置多个服务器不会产生冗余，所以机器故障后，所有服务器都会宕机。真是情况下需要每个服务器都有完全独立的物理服务器。
### 在一台计算机上搭建伪集群
[参考链接](https://blog.csdn.net/zhangphil/article/details/99990669)
##### 1 复制
将之前的ZooKeeper文件复制3份，我顺便换了各名字
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210428102424660.png)
##### 2 修改zoo.cg
（1）initLimit：初始化连接时，Zookeeper 服务器集群中连接到 Leader 的 Follower 服务器最长可以忍受的心跳时长（`initLimit*tickTime`），如果超出时长还没有连接成功，则说明连接失败。
（2）syncLimit：Leader 与 Follower 之间发送消息，请求和应答时间长度，最长不超过`syncLimit*tickTime`。
（3）server.A=B:C:D A是当前服务的序号，B是该服务所在IP地址，C是节点之间信息交流的端口，D是服务出现异常后，选举leader的端口

zk1
```c
tickTime=2000
dataDir=D:\kj\ZooKeeper\zk1\data
dataLogDir=D:\kj\ZooKeeper\zk2\log
clientPort=2181
initLimit=5
syncLimit=2
server.1=localhost:2888:3888
server.2=localhost:2888:3889
server.3=localhost:2888:3890
```
zk2
```c
tickTime=2000
dataDir=D:\kj\ZooKeeper\zk2\data
dataLogDir=D:\kj\ZooKeeper\zk2\log
clientPort=2182
initLimit=5
syncLimit=2
server.1=localhost:2888:3888
server.2=localhost:2888:3889
server.3=localhost:2888:3890
```
zk3
```c
tickTime=2000
dataDir=D:\kj\ZooKeeper\zk3\data
dataLogDir=D:\kj\ZooKeeper\zk3\log
clientPort=2183
initLimit=5
syncLimit=2
server.1=localhost:2888:3888
server.2=localhost:2888:3889
server.3=localhost:2888:3890
```

##### 3 创建myid文件
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210428103833978.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210428103859606.png)
myid的值是zoo.cfg文件里定义的server.A项A的值。Zookeeper启动时会读取这个文件。因此zk1的myid内容是1，zk2的是2，zk3的是3。
##### 4 启动
分别点击各自的zkServer
+ 1 报错：`Invalid config, exiting abnormally`
+ 2 忘了改myid了，改一下，再试一次。依然报这个错：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210428110356497.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
这里的主要原因是myid文件没有找到，和目录有关系，把目录改成：
`D:\\kj\\ZooKeeper\\zk1\\data`这样，就好了。（我也不知道为什么）
+ 3 三个全打开
不然会报错
```c
//这个应该是选举机制（只启动了集群的部分服务器）？全打开就不报这个了
Canot open channel to 2 at election localhost...
```
+ 4 zk1报错

```c
.1:2890:Learner$LeaderConnector@445] - Unexpected exception, tries=1, remaining init limit=5840, connecting to ..
java.net.ConnectException: Connection refused: connect
```
我一气之下，就把他们端口号全改了
```c
server.1=localhost:1100:1200
server.2=localhost:2100:2200
server.3=localhost:3100:3200
```
然后就好了，虽然我没有找到其他应用占用这几个端口，但改了的确好了。
##### 5 如何判断好没有呢？
点击任意一个zCli.cmd，出现下图。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021042814465154.png)
当然也可以不手动点，输命令：[参考链接](https://www.cnblogs.com/yangzhenlong/p/8270835.html)

## 其他优化
将事务日志和数据快照分离，也就是前面的log目录
> 恭喜你和我又学完了一篇，可真棒呐！
