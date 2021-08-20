>我的每一篇这种正经文章，都是我努力克制玩心的成果，我可太难了，和自己做斗争。

@[TOC](ZooKeeper官方文档学习笔记04－程序员指南03)
# 绑定
zk客户端支持两种语言：Java和C
## Java绑定
相关包：org.apache.zookeeper 和 org.apache.zookeeper.data，data主要用作容器，由生成的类组成（？？）

Zookeeper的java客户端使用的主类是ZooKeeper类（org.apache.zookeeper.ZooKeeper）。他的构造方法通过两方面区分：会话id和会话密码。Java程序可以将会话id和密码保存下来（方便使用？）
创建zk对象会创建两个线程：一个IO线程（IO事件发生地）和一个事件线程（事件回调发生地）。
> 常用的IO事件：重新连接到zk服务器、维护心跳、同步方法的响应
常用的事件回调：对异步方法和监视事件的响应

Note:
+ 1 异步调用和watcher回调都是按顺序进行的，一次执行一个？在此期间不处理其他回调
+ 2 事件回调不会阻塞IO线程或同步调用的处理。
+ 3 同步调用存在不按顺序执行的可能性。例如，假设客户端执行以下处理：在watcher设置为true(监听数据变更)的情况下发出对节点/a的异步读取，然后再读取的完成回调中执行/a的同步读取。
> 这样举例，可能会比较好理解。

```c
$.ajax({
    type:'POST',
    url:"a.html"，
    success:function(msg) {
         $("#form").submit();
     }
 });
 ...
 <ww:form id="form" action="a"/>
```
> 其中watcher和异步方法是事件回调，同步方法是IO事件

注意，如果异步读取和同步读取之间的/a发生了更改，则客户端将在同步读取响应之前收到一个watch事件，表示/a changed。但由于完成回调阻塞了事件队列。因此在处理watch事件之前，同步读取将返回/a的新值。
> 完成回调是指异步读取吗？因为异步读取在watch之前，所以回调堵塞了队列。导致同步先读取到值，而不是先接收到节点a的变化？

与关闭相关的规则：一旦zk对象关闭或收到致命事件（SESSION_expired和AUTH_failed），zk对象就无效了。关闭时，这两个线程关闭了，后续对zookeeper句柄的任何访问都将导致不确定的行为，应该避免。
## 客户端配置参数
+ 1 `zookeeper.sasl.client`：`false`代表禁用SASL身份验证，默认为`true`。
+ 2 `zookeeper.sasl.clientconfig`：指定JAAS登陆文件中的上下文键，默认为`"Client`"。

+ 3 zookeeper.sasl.client.username：一般，一个主体被分为3部分：primary、instance、和realm。典型的 Kerberos V5主体的格式是 primary/instance@REALM。instance派生自服务器IP。`zookeeper.sasl.client.username`指定primary即username，默认是"zookeeper"。所以最终是username/IP@realm
+ 4 zookeeper.server.realm：上文的realm
+ 5 zookeeper.disableAutoWatchReset：是否启用自动watch重置。默认，客户端在会话重连期间自动重置watch，设置为"`true`"可以关闭此行为。
+ 6 zookeeper.client.secure：为`true`可以让客户端连接到服务器安全客户端端口，需要Netty客户端，且需使用带有指定凭据的SSL连接。
+ 7 zookeeper.clientCnxnSocket：指定使用哪个ClientCnxnSocket。可选的有org.apache.zookeeper.ClientCnxnSocketNIO and org.apache.zookeeper.ClientCnxnSocketNetty 。默认的是第一个。如果想要连接到服务器的安全客户端端口，要在客户端上设置成第二种。
+ 8 zookeeper.ssl.keyStore.location and zookeeper.ssl.keyStore.password：指定包含用于SSL连接的本地凭据的JKS的文件路径和解锁文件的密码。
+ 9 zookeeper.ssl.trustStore.location and zookeeper.ssl.trustStore.password：指定包含用于SSL连接的远程凭据的JKS的文件路径和解锁文件的密码。
+ 10 jute.maxbuffer：指定服务器传入数据的最大大小。默认值是4194304字节，或者只有4MB。这至关重要，zookeeper的服务器可存储和发送数千字节的数据。如果长度打了，将引发IOException。
+ 11 zookeeper.kinit：指定kinit二进制文件的路径，默认值“/usr/bin/kinit”。
# C绑定
诶呀，同上文的C，暂不负责（~~这样就可以在今天下午写完程序员指南了，嘿嘿~~~ ）

绑定都会发生错误，Java 客户机绑定通过抛出 KeeperException 来实现，对异常调用 code ()将返回特定的错误代码。C 客户端绑定返回一个错误代码，这个错误代码定义在 enum ZOO _ errors 中，具体参考API。
# 陷阱: 常见问题及故障排除
+ 1 watch只能监听已连接事件：zk客户端与服务器断连时，创建并删除znode，将不会收到watch事件。
+ 2 zk服务器只要大多服务器存活就可以。但你要保证在发生故障后恢复你的状态和未完成的请求。
+ 3 客户端使用的zk服务器连接字符串必须和zk服务器匹配，不匹配无法连接。（必须是现有的zk服务器）
+ 4 事务日志是影响zk性能的**最关键**部分，位置谨慎选择。zk必须在返回响应前将事物同步到媒体。所以要搞个专门的事务日志（之前好像有提到？）
+ 5 正确设置Java最大堆大小，避免内存交换。内存交换到磁盘会降低性能（参考操作系统的缓存、硬盘？）zk请求是有序的，如果一个请求到达磁盘，所有排队的请求也会到达磁盘。所以尝试将heapsize设置为拥有的物理内存量。比如4g的机器上设置3g。确定大小的最佳方法是运行负载测试。

终于写完程序员指南这篇了，可真不容易。来都来了，点个赞再走呗，会回访呐。![在这里插入图片描述](https://img-blog.csdnimg.cn/f1cd19ecb6b74eb59e6bdd974da6f801.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
