> 开篇碎碎念：不要试图用断点，或者你断点位置要放好，不然你就会收获许多连接异常。这绝对是我目前翻译过的最流畅的。咳，不是官网流畅，是我笔记流畅，也许是我成长了。（*屏障就是人齐开饭，都吃完散场。然后队列是操作系统的生产者-消费者模型*）
![在这里插入图片描述](https://img-blog.csdnimg.cn/8fdd526a2f78496bb7161441963e362d.jpg)

@[TOC](ZooKeeper的屏障（Barrier）和队列（Queue）教程)
# 引言
这个教程展示屏障和生产者-消费者队列的实现。下文主要涉及Barrier和Queue类。你需要启动至少一个zookeeper服务器。
这两个类都继承了`SyncPrimitive`。
> 注释前的序号是连起来的，若前文没有，可以看后面
## SyncPrimitive
```java
package barrierexample;

import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooKeeper;
import java.io.IOException;

public class SyncPrimitive implements Watcher {
    static ZooKeeper zk = null;
    static Integer mutex;
    String root;
    
    SyncPrimitive(String address) {
    //3 zookeeper实例不存在，则父类自己创造
        if (zk == null) {
            try {
                System.out.println("Starting ZK:");
                zk = new ZooKeeper(address,3000,this);
                mutex = new Integer(-1);
                System.out.println("Finished starting ZK: "+zk);
            } catch (IOException e) {
                System.out.println(e.toString());
                zk = null;
            }
        }
    }
    @Override
    synchronized public void process(WatchedEvent event) {
        synchronized (mutex) {
            mutex.notify();
        }
    }
}

```

我们在第一次实例化barrier对象或queue对象时创建了zookeeper对象。并声明一个静态变量作为该对象的引用。Barrier 和 Queue 的后续实例检查 ZooKeeper 对象是否存在。或者，我们可以让应用程序创建一个 ZooKeeper 对象，并将其传递给 Barrier 和 Queue 的构造函数。
我们使用process()方法去处理watch触发的通知。watch作为内部结构，可以让zookeeper通知客户端节点的变更。若一个客户端在等待其他客户端离开barrier，那么他可以设置一个watch监视节点修改。
# 屏障（Barrier）
Barrier是一个原语，他的作用是同步 计算的开始和结束。
> 有多个进程调用了，等他们都投进去的时候，开始。等他们都结束的时候结束，宏观上看是同步计算的？

实现方法是使用一个屏障节点作为单个流程节点的父节点“/b1”，每个进程"p"创建节点"/b1/p"。只要有足够的进程创建了相应的节点，就可以开始**计算**了。
> 想象一个二叉树，b1是父节点，p是子节点

## 构造函数
```java
public class SyncPrimitive implements Watcher {
    ......
    /**
     * Barrier
     */
    static public class Barrier extends SyncPrimitive {
        int size;
        String name;

        /**
         * 实例化Barrier对象的进程的参数如下：
         * @param address zookeeper服务器地址（如"zoo1.foo.com:2181"）
         * @param root zookeeper上屏障节点的路径（如"/b1")
         * @param size 一组进程的大小
         */
        Barrier(String address, String root, int size) {
            //1 Barrier的构造函数将zookeeper服务器的地址传递给父类的构造函数
            super(address);
            this.root = root;
            this.size = size;
            //1.5 如果zk对象存在
            if (zk != null) {
                try {
                    //2 验证根节点是否存在
                    //(此root非zookeeper的root哦）
                    Stat s = zk.exists(root, false);
                    //2.5 不存在则，Barrier的构造函数在zookeeper上创建一个Barrier节点，作为上述进程节点的父节点，被称作root
                    if (s == null) {
                        zk.create(root,new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE,
                                CreateMode.PERSISTENT);
                    }
                } catch (KeeperException e) {
                    System.out.println("Keeper exception when instantiating queue: "
                    + e.toString());
                } catch (InterruptedException e) {
                    System.out.println("Interrupted exception");
                }
            }
            //获取本地主机地址的完全限定域名
            try {
                name = new String(InetAddress.getLocalHost().getCanonicalHostName().toString());
            } catch (UnknownHostException e) {
                System.out.println(e.toString());
            }
        }
        ......
    }
}
```
进入屏障的方法是`enter()`
## enter()
```java
public class SyncPrimitive implements Watcher {
    ......
    static public class Barrier extends SyncPrimitive {
        .....
        /**
         * 进入屏障
         * @return
         * @throws KeeperException
         * @throws InterruptedException
         */
        boolean enter() throws KeeperException,InterruptedException {
            //4 进程用root下面创建的节点代表自己，用主机名来作为节点名。
            //这里如过写临时顺序节点（EPHEMERAL_SEQUENTIAL）的话，创建节点后就无法根据名字（名字会变为一串0的）删除它，所以，我们把这里设为临时节点即可
            zk.create(root + "/"+name,new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE,
                    CreateMode.EPHEMERAL);
            while (true) {
                //5 它会一直等待直到足够多的进程进入屏障
                synchronized (mutex) {
                    //6进程通过getChidren()检查root子节点的数量是否满足要求，
                    //getChildren两个参数，第一个表示要读取的节点，第二个表示是否设置watch（根节点发生变化时通知），true为设置
                    List<String> list = zk.getChildren(root,true);
                    if (list.size() < size) {
                        mutex.wait();
                    } else {
                        //7 满足要求就结束等待
                        return true;
                    }
                }
            }
        }
    }
}
```
## leave()
计算完成后，进程调用`leave()`离开屏障。

```java
public class SyncPrimitive implements Watcher {
    ......
    static public class Barrier extends SyncPrimitive {
    .....
        /**
         * 在所有子节点被删除后来开屏障
         * @return
         * @throws KeeperException
         * @throws InterruptedException
         */
        boolean leave() throws KeeperException,InterruptedException {
            zk.delete(root + "/"+name,0);
            while (true) {
                synchronized (mutex) {
                    //8 删除对应子节点并设置watch
                    List<String> list = zk.getChildren(root,true);
                    if (list.size() > 0) {
                        mutex.wait();
                    } else {
                        return true;
                    }
                }
            }
        }
    }
}
```
# 生产者-消费者队列(Queue)
> 这是我操作系统中的那个生产者-消费者吗？往下看

![请添加图片描述](https://img-blog.csdnimg.cn/b4be75b1e645449f9d3f53c30564795d.png)
生产者-消费者队列是一种分布式的数据结构。一组进程用它来生产和消费项目。
生产者进程创建新元素并将他们添加到队列，消费者进程从队列中移除元素并使用他们。在这个实现中，元素是相当于原子的存在。队列由根节点来代表。生产者进程是创建根节点的子节点
>呀，和前面的enter()方法有点类似呦
## 构造函数
下面是`Queue`的构造函数
```java
public class SyncPrimitive implements Watcher {
  ......
    /**
     * 生产者-消费者队列
     */
    static public class Queue extends SyncPrimitive {
        Queue(String address,String name) {
            //q1 调用父类构造函数，如果zookeeper对象不存在就自己创建一个
            super(address);
            this.root = name;
            //验证根节点是否存在，不存在自己创建
            if (zk != null) {
                try {
                    Stat s = zk.exists(root,false);
                    if (s == null) {
                        //几个参数：节点路径、节点数据、acl权限、节点类型
                        // OPEN_ACL_UNSAFE：完全开放、PERSISTENT：持久化节点
                        zk.create(root,new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE,
                                CreateMode.PERSISTENT);
                    }
                } catch (KeeperException e) {
                    System.out
                            .println("Keeper exception when instantiating queue: "
                                    + e.toString());
                } catch (InterruptedException e) {
                    System.out.println("Interrupted exception");
                }
            }
        }
    }
}
```
## produce()
生产者进程调用`produce()`向队列中添加元素。

```java
public class SyncPrimitive implements Watcher {
    .....
        /**
         * 向队列添加元素
         * @param i
         * @return
         * @throws KeeperException
         * @throws InterruptedException
         */
        boolean produce(int i) throws KeeperException,InterruptedException {
            //分配堆字节缓冲区，容量为4
            ByteBuffer b = ByteBuffer.allocate(4);
            byte[] value;
            b.putInt(i);
            value = b.array();
            //PERSISTENT_SEQUENTIAL：持久化有序节点，可以让队列按照先进先出（操作系统+1,就是先排队的先吃饭啦）的方法使用元素
            zk.create(root + "/element",value, ZooDefs.Ids.OPEN_ACL_UNSAFE,
                    CreateMode.PERSISTENT_SEQUENTIAL);
            return  true;
        }
    }
}
```
## consume()
消费者使用元素的方法：消费者进程`consume()`获取根节点的子节点，然后依据value去获取排列在最前面的元素。如果同时有两个进程去要获取这个元素，就两个都无法获取，并移除节点
> 死锁啦？

```java
public class SyncPrimitive implements Watcher {
   ......
    /**
     * 生产者-消费者队列
     */
    static public class Queue extends SyncPrimitive {
    ......
        /**
         * 移除队列的第一个元素
         * @return
         * @throws KeeperException
         * @throws InterruptedException
         */
        int consume() throws KeeperException,InterruptedException {
            int retValue = -1;
            Stat stat = null;
            while(true) {
                synchronized (mutex) {
                    List<String> list = zk.getChildren(root,true);
                    //如果list为空，也就是没有子节点，消费者无法消费，所以进程陷入等待
                    if (list.size() == 0) {
                        System.out.println("Going to wait");
                        mutex.wait();
                    } else {
                        //移除前缀。前缀7位：element 节点长啥样，为啥后面是int？
                        Integer min = new Integer(list.get(0).substring(7));
                        String minNode = list.get(0);
                        //寻找最小值(因为children的顺序可能和队列不一致)
                        for (String s: list) {
                            //移除前缀。
                            Integer tempValue = new Integer(s.substring(7));
                            //System.out.println("Temporary value: " + tempValue);
                            if (tempValue < min) {
                                min = tempValue;
                                minNode = s;
                            }
                        }
                        System.out.println("Temporary value: " + root + "/" +minNode);
                        //获取节点值
                        byte[] b = zk.getData(root + "/"+minNode,
                                false,stat);
                        //移除节点
                        zk.delete(root +"/"+minNode,0);
                        //缓冲区的数据会存放在byte数组中
                        ByteBuffer buffer = ByteBuffer.wrap(b);
                        retValue = buffer.getInt();
                        //返回值
                        return retValue;
                    }
                }
            }
        }
    }
}

```
# 测试
## 代码
```java
public class SyncPrimitive implements Watcher {
    .....
    //args接收参数
    public static void main(String args[]) {
        //如果是qTest那么代表是队列的测试
        if (args[0].equals("qTest")) {
            queueTest(args);
        } else {
        //屏障测试
            barrierTest(args);
        }
    }
    public static void queueTest(String args[]) {
        //在地址为args[1]的zookeeper服务器上，建立/app1节点
        Queue q = new Queue(args[1],"/app1");
        System.out.println("Input:"+args[1]);
        //创建元素的数量
        int i;
        Integer max = new Integer(args[2]);
        //创建元素
        if (args[3].equals("p")) {
            System.out.println("Producer");
            for (i = 0; i < max; i++) {
                try {
                    q.produce(10 + i);
                } catch (KeeperException e) {
                } catch (InterruptedException e) {
                }
            }
        } else {
            //消费元素
            System.out.println("Consumer");
            for (i = 0; i < max; i++) {
                try {
                    int r = q.consume();
                    System.out.println("Item:"+r);
                } catch (KeeperException e) {
                    //剩余元素的数量
                    i--;
                } catch (InterruptedException e) {
                }
            }
        }
    }
    public static void barrierTest(String args[]) {
        //创建一个屏障，可以容纳两个参与者
        Barrier b = new Barrier(args[1],"/b1",new Integer(args[2]));

        try{
            boolean flag = b.enter();
            System.out.println("Entered barrier: "+args[2]);
            if (!flag) {
                System.out.println("Error when entering the barrier");
            }
        } catch (KeeperException e) {
            System.out.println("barrierTest enter KeeperException");
        } catch (InterruptedException e) {
            System.out.println("barrierTest enter InterruptedException");
        }
        Random rand = new Random();
        //int r = rand.nextInt(100);
        int r = 200;
        for (int i = 0; i < r; i++) {
            try {
                Thread.sleep(100);
            }catch (InterruptedException e) {
                System.out.println("barrierTest sleep InterruptedException");
            }
        }
        try {
            b.leave();
        } catch (KeeperException e) {
            System.out.println("barrierTest leave KeeperException"+e.toString());
        } catch (InterruptedException e) {
            System.out.println("barrierTest leave InterruptedException");
        }
        System.out.println("Left barrier");
    }
}
```
## 调用方法
+ 1 启动一个zookeeper的服务器
（1）如果你是直接在记事本里写的代码
可以点击zookeeper的zkServer.cmd
（2）如果你和我一样用的idea
可以这样（详细配置[参考](https://blog.csdn.net/moqianmoqian/article/details/119541703)）：
![在这里插入图片描述](https://img-blog.csdnimg.cn/76ea536af4ca47ceaf58bc17964e2cbf.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
然后启动`ZooKeeperServerMain`
+ 2 编译`SyncPrimitive`
（1） 如果你是直接在记事本里写的代码
然后`javac`的话，他可能会报一些换行符什么的问题，或者无法找到主类（也许和最上方有package有关），自己修正一下啦~，因为idea没有呢
（2）如果你是用的idea
先编译一下SyncPrimitive
![在这里插入图片描述](https://img-blog.csdnimg.cn/9a54822296074ee187128e0e0f8bb6c9.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
确保这个目录下有（代表编译成功）
![在这里插入图片描述](https://img-blog.csdnimg.cn/d63ec875ec3b4919a819a4b71d0e9c6e.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
+ 3 传参
（1）第一种 我没有尝试，你自己试试啦~
（2）idea
```bash
qTest 127.0.1:2181 100 p
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/92fec4014a4f4819895de96a22eec322.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
## 调试
### 队列调试
掌握了调用方法，启动了zookeeper服务器就正式开始调试喽
![在这里插入图片描述](https://img-blog.csdnimg.cn/6a245804a0264a72b1dad38d3d87d71e.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
+ 1 先生产100个元素
```bash
qTest 127.0.1:2181 100 p
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/47cc95d8a8c64abb8d5125a86d0a4e20.png)
生产成功与否，可以用zkCli.cmd看一下
![在这里插入图片描述](https://img-blog.csdnimg.cn/72045c5249df4a79baa9f8587588b190.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
+ 2 再消费100个元素

```bash
qTest localhost 100 c
```
> 127.0.1:2181和localhost都可以呀

![在这里插入图片描述](https://img-blog.csdnimg.cn/270b2ab80c53459cb2ec51dacf596f5b.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/3864812a17c147e1966af9e1fd059a2e.png)
好的，成功，真棒呐
![在这里插入图片描述](https://img-blog.csdnimg.cn/9ad3a06e4756402e89ae5bea69455bd1.jpg)
下一个~
### 屏障调试
设置一个障碍与2个参与者
```bash
bTest localhost 2
```
启动之后~~~~
(1) 这里是idea的console
![在这里插入图片描述](https://img-blog.csdnimg.cn/93cd50aafa8b48fcbd4d73c9efb31de5.png)
（2）这个是zServer.cmd的
因为他需要两个参与者，然后我们程序只创建了一个参与者，b.enter()生效，但是这样他只建立了一个参与者，这个时候，我们的程序就陷入了无限的等待中
![在这里插入图片描述](https://img-blog.csdnimg.cn/cca35d12c96b42e98e61e84d9383d32e.png)
（3）创建节点

```bash
create -e /b1/a 1
```
在右图创建节点后，作图触发watch，输出第一个Process:NodeChildrenChanged
然后输出Entered..
等进入后，调用sleep，睡眠之后。删除一个节点，也就是第二个Process。但是删除节点后，因为还有个节点在里面，所以会陷入等待
![请添加图片描述](https://img-blog.csdnimg.cn/b4695c1e0a244ccb8dcc7e66d005da44.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)

（4）然后 我们用
```bash
rmr /b1/a
```
删除另一个节点，这是触发下图的第二个Process
然后离开屏障
![请添加图片描述](https://img-blog.csdnimg.cn/eb0a8505e83c4ab2af4475a87804db45.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)
到此为止，恭喜你完成啦~
![在这里插入图片描述](https://img-blog.csdnimg.cn/3e190c932a174132b2fd822228ca1a9b.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)

# 完整源码

```java
package barrierexample;

import org.apache.zookeeper.*;
import org.apache.zookeeper.data.Stat;

import java.io.IOException;
import java.net.InetAddress;
import java.net.UnknownHostException;
import java.nio.ByteBuffer;
import java.security.KeyPair;
import java.util.List;
import java.util.Random;

public class SyncPrimitive implements Watcher {
    static ZooKeeper zk = null;
    static Integer mutex;
    String root;

    SyncPrimitive(String address) {
        //3 zookeeper实例不存在，则父类自己创造
        if (zk == null) {
            try {
                System.out.println("Starting ZK:");
                zk = new ZooKeeper(address,3000,this);
                mutex = new Integer(-1);
                System.out.println("Finished starting ZK: "+zk);
            } catch (IOException e) {
                System.out.println(e.toString());
                zk = null;
            }
        }
    }
    @Override
    synchronized public void process(WatchedEvent event) {
        synchronized (mutex) {
            System.out.println("Process: " + event.getType());
            mutex.notify();
        }
    }

    /**
     * Barrier
     */
    static public class Barrier extends SyncPrimitive {
        int size;
        String name;

        /**
         * 实例化Barrier对象的进程的参数如下：
         * @param address zookeeper服务器地址（如"zoo1.foo.com:2181"）
         * @param root zookeeper上屏障节点的路径（如"/b1")
         * @param size 一组进程的大小
         */
        Barrier(String address, String root, int size) {
            //1 Barrier的构造函数将zookeeper服务器的地址传递给父类的构造函数
            super(address);
            this.root = root;
            this.size = size;
            //1.5 如果zk对象存在
            if (zk != null) {
                try {
                    //2 验证根节点是否存在
                    //(此root非zookeeper的root哦）
                    Stat s = zk.exists(root, false);
                    //2.5 不存在则，Barrier的构造函数在zookeeper上创建一个Barrier节点，作为上述进程节点的父节点，被称作root
                    if (s == null) {
                        zk.create(root,new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE,
                                CreateMode.PERSISTENT);
                    }
                } catch (KeeperException e) {
                    System.out.println("Keeper exception when instantiating queue: "
                    + e.toString());
                } catch (InterruptedException e) {
                    System.out.println("Interrupted exception");
                }
            }
            //获取本地主机地址的完全限定域名
            try {
                name = new String(InetAddress.getLocalHost().getCanonicalHostName().toString());
            } catch (UnknownHostException e) {
                System.out.println(e.toString());
            }
        }

        /**
         * 进入屏障
         * @return
         * @throws KeeperException
         * @throws InterruptedException
         */
        boolean enter() throws KeeperException,InterruptedException {
            //4 进程用root下面创建的节点代表自己，用主机名来作为节点名。
            //这里如过写临时顺序节点（EPHEMERAL_SEQUENTIAL）的话，创建节点后就无法根据名字（名字会变为一串0的）删除它，所以，我们把这里设为临时节点即可
            zk.create(root + "/"+name,new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE,
                    CreateMode.EPHEMERAL);
            while (true) {
                //5 它会一直等待直到足够多的进程进入屏障
                synchronized (mutex) {
                    //6进程通过getChidren()检查root子节点的数量是否满足要求，
                    //getChildren两个参数，第一个表示要读取的节点，第二个表示是否设置watch（根节点发生变化时通知），true为设置
                    List<String> list = zk.getChildren(root,true);
                    if (list.size() < size) {
                        mutex.wait();
                    } else {
                        //7 满足要求就结束等待
                        return true;
                    }
                }
            }
        }

        /**
         * 在所有子节点被删除后来开屏障
         * @return
         * @throws KeeperException
         * @throws InterruptedException
         */
        boolean leave() throws KeeperException,InterruptedException {
            zk.delete(root + "/"+name,0);
            while (true) {
                synchronized (mutex) {
                    //8 删除对应子节点并设置watch
                    List<String> list = zk.getChildren(root,true);
                    if (list.size() > 0) {
                        mutex.wait();
                    } else {
                        return true;
                    }
                }
            }
        }
    }

    /**
     * 生产者-消费者队列
     */
    static public class Queue extends SyncPrimitive {

        Queue(String address,String name) {
            //q1 调用父类构造函数，如果zookeeper对象不存在就自己创建一个
            super(address);
            this.root = name;
            //验证根节点是否存在，不存在自己创建
            if (zk != null) {
                try {
                    Stat s = zk.exists(root,false);
                    if (s == null) {
                        //几个参数：节点路径、节点数据、acl权限、节点类型
                        // OPEN_ACL_UNSAFE：完全开放、PERSISTENT：持久化节点
                        zk.create(root,new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE,
                                CreateMode.PERSISTENT);
                    }
                } catch (KeeperException e) {
                    System.out
                            .println("Keeper exception when instantiating queue: "
                                    + e.toString());
                } catch (InterruptedException e) {
                    System.out.println("Interrupted exception");
                }
            }
        }

        /**
         * 向队列添加元素
         * @param i
         * @return
         * @throws KeeperException
         * @throws InterruptedException
         */
        boolean produce(int i) throws KeeperException,InterruptedException {
            //分配堆字节缓冲区，容量为4
            ByteBuffer b = ByteBuffer.allocate(4);
            byte[] value;
            b.putInt(i);
            value = b.array();
            //PERSISTENT_SEQUENTIAL：持久化有序节点，可以让队列按照先进先出（操作系统+1）的方法使用元素
            zk.create(root + "/element",value, ZooDefs.Ids.OPEN_ACL_UNSAFE,
                    CreateMode.PERSISTENT_SEQUENTIAL);
            return  true;
        }

        /**
         * 移除队列的第一个元素
         * @return
         * @throws KeeperException
         * @throws InterruptedException
         */
        int consume() throws KeeperException,InterruptedException {
            int retValue = -1;
            Stat stat = null;
            while(true) {
                synchronized (mutex) {
                    List<String> list = zk.getChildren(root,true);
                    //如果list为空，也就是没有子节点，消费者无法消费，所以进程陷入等待
                    if (list.size() == 0) {
                        System.out.println("Going to wait");
                        mutex.wait();
                    } else {
                        //移除前缀。前缀7位：element 节点长啥样，为啥后面是int？
                        Integer min = new Integer(list.get(0).substring(7));
                        String minNode = list.get(0);
                        //寻找最小值(因为children的顺序可能和队列不一致)
                        for (String s: list) {
                            //移除前缀。
                            Integer tempValue = new Integer(s.substring(7));
                            //System.out.println("Temporary value: " + tempValue);
                            if (tempValue < min) {
                                min = tempValue;
                                minNode = s;
                            }
                        }
                        System.out.println("Temporary value: " + root + "/" +minNode);
                        //获取节点值
                        byte[] b = zk.getData(root + "/"+minNode,
                                false,stat);
                        //移除节点
                        zk.delete(root +"/"+minNode,0);
                        //缓冲区的数据会存放在byte数组中
                        ByteBuffer buffer = ByteBuffer.wrap(b);
                        retValue = buffer.getInt();
                        //返回值
                        return retValue;
                    }
                }
            }
        }
    }
    //args接收参数
    public static void main(String args[]) {
        //如果是qTest那么代表是队列的测试
        if (args[0].equals("qTest")) {
            queueTest(args);
        } else {
            //屏障测试
            barrierTest(args);
        }
    }
    public static void queueTest(String args[]) {
        //在地址为args[1]的zookeeper服务器上，建立/app1节点
        Queue q = new Queue(args[1],"/app1");
        System.out.println("Input:"+args[1]);
        //创建元素的数量
        int i;
        Integer max = new Integer(args[2]);
        //创建元素
        if (args[3].equals("p")) {
            System.out.println("Producer");
            for (i = 0; i < max; i++) {
                try {
                    q.produce(10 + i);
                } catch (KeeperException e) {
                } catch (InterruptedException e) {
                }
            }
        } else {
            //消费元素
            System.out.println("Consumer");
            for (i = 0; i < max; i++) {
                try {
                    int r = q.consume();
                    System.out.println("Item:"+r);
                } catch (KeeperException e) {
                    //剩余元素的数量
                    i--;
                } catch (InterruptedException e) {
                }
            }
        }
    }
    public static void barrierTest(String args[]) {
        //创建一个屏障，可以容纳两个参与者
        Barrier b = new Barrier(args[1],"/b1",new Integer(args[2]));

        try{
            boolean flag = b.enter();
            System.out.println("Entered barrier: "+args[2]);
            if (!flag) {
                System.out.println("Error when entering the barrier");
            }
        } catch (KeeperException e) {
            System.out.println("barrierTest enter KeeperException");
        } catch (InterruptedException e) {
            System.out.println("barrierTest enter InterruptedException");
        }
        Random rand = new Random();
        //int r = rand.nextInt(100);
        int r = 200;
        for (int i = 0; i < r; i++) {
            try {
                Thread.sleep(100);
            }catch (InterruptedException e) {
                System.out.println("barrierTest sleep InterruptedException");
            }
        }
        try {
            b.leave();
        } catch (KeeperException e) {
            System.out.println("barrierTest leave KeeperException"+e.toString());
        } catch (InterruptedException e) {
            System.out.println("barrierTest leave InterruptedException");
        }
        System.out.println("Left barrier");
    }
}

```
都看到这里了，点个赞再走呗
![在这里插入图片描述](https://img-blog.csdnimg.cn/f2e4f5cc35d548988da55917beab1e06.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)


