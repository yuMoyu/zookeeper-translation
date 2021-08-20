>在执行官网的java例子的时候，发现根本无法启动，后来发现是我的zookeeper就没编译好？试一下

# idea编译启动zookeeper源码
# 修改配置文件
类似[ZooKeeper官方文档学习笔记02－ZooKeeper入门指南](https://blog.csdn.net/moqianmoqian/article/details/116196513)这个
### 1 zoo.cfg
在zookeeper源码中，寻找`zoo_sample.cfg`。然后复制一份放在文件夹下，重命名为`zoo.cfg`

```bash
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
dataDir=./tmp/data
dataLogDir=./tmp/log
clientPort=2181
```

### 2 log4j.properties
将`conf`下的`log4j.properties`复制到`zookeeper-server/src/main/resources`
>zoo.cfg貌似也要复制，不复制应该也行，这两个文件

![在这里插入图片描述](https://img-blog.csdnimg.cn/dfc8b8008c934b27a28b90e32226feec.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
### 3 将文件夹标记为Resources Root
文件夹：`zookeeper-server/src/main/resources`
（编译的时候按照原文件输出（[其余的标记参考](https://blog.csdn.net/xiaohei_neko/article/details/79353605)）
![在这里插入图片描述](https://img-blog.csdnimg.cn/9d94eabb492d4155992a9f65250e3d08.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
# 增加启动项
### 1 QuorumPeerMain
zookeeper集群的启动入口类，也可以选择`ZooKeeperServerMain`应该，路径都是zookeeper-server/src/main下
![在这里插入图片描述](https://img-blog.csdnimg.cn/a1ec25a72a9b4579a13d6ba26ef2e123.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
> vm options配置了日志位置，如果这个文件没有复制过来，用绝对 路径就可以了，con.cfg应该是同理的。
> 还有右上角的**Allow parallel run**要勾选，这样才能同时启动服务器和客户端

```bash
-Dlog4j.configuration=file:conf/log4j.properties
conf/zoo.cfg
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/34b81ca76ae54e759c6d28f094d8b4ac.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)

# 注释掉pom.xml的scope
目录如下图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/eab05532d1d64d2283dc8a3d3826f360.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)

注释掉所有的provied,像这样：`<!--<scope>provided</scope>-->`。作用是可以避免出现`java.lang.NoClassDefFoundError`这种错误。但是有时候即使你去掉了依然会报错，需要你试试一下方法。
+ 1 build->rebuild project
![在这里插入图片描述](https://img-blog.csdnimg.cn/c692d381510e4e5ea5d0967b8a6fb324.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
+ 2 maven->clean
![在这里插入图片描述](https://img-blog.csdnimg.cn/592ffa6e6e704368a31d4e99265bdff8.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
+ 3 maven->install
+ 4 mvn clean install -DskipTests
>我试了好多次才生效，不知道具体为什么，应该是没找到最正确的地方。
>
> 如果你的启动提示log4j不存在，那就是你的Resources忘了复制log4j文件或者忘记将文件夹标记为Resources Root了。

>还有个很奇怪的是，如果你复制过来，没有配置log4j第一次启动没有问题，第二次会提示没有log4j啥的。

# 启动服务端
首先编译一下，用命令行，右侧的maven可能会报错

```bash
mvn clean install -DskipTests
```
编译完成后，启动main，启动成功如图
![在这里插入图片描述](https://img-blog.csdnimg.cn/cf175df563dd4118a5df9c8386110097.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
# 启动客户端
```bash
org.apache.zookeeper.ZooKeeperMain
```
注释掉ZookeeperMain类对应的pom.xml下的这个scope
![在这里插入图片描述](https://img-blog.csdnimg.cn/cce51bbff19243e8b1a2ed0474b133b1.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)

```bash
-server 127.0.1:2181
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/1cc6f4d0aecf46cb86173136cc61bea5.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/dadb61c2147b4d5b9d3918533e77b2ac.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
