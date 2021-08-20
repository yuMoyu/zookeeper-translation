> 碎碎念：启动成功了一半。可以启动，可以debug，但是有些方法无法访问，而且create在哪里，我还不清楚。那个DataMonitor，不能完全按照官网写，要像我一样改一下，不然会报werror，因为有些过时了

# ZooKeeper的Java实例
# 一个简单的watch客户端
作用：监视ZooKeeper节点的更改，对程序的启动或停止做出响应。
## 要求
+ 1 它需要四个参数：
&nbsp;&nbsp;zk服务器的地址
&nbsp;&nbsp;被监视节点的名字
&nbsp;&nbsp;输出要写入的文件名
&nbsp;&nbsp;带参数的可执行文件
> 是这样理解吗？

+ 2 获取与znode关联的数据并启动可执行文件
+ 3 若被监视的znode发生更改，客户机将重新获取内容并重启启动可执行程序
+ 4 若被监视的znode消失，客户端将杀死可执行程序
## 程序设计
ZooKeeper应用程序分为两部分：维护与服务器连接和监视节点数据。`Executor`类负责维护连接部分，`DataMonitor`负责监视zookeeper树中的数据。Executor包含主线程和执行逻辑。它负责少量的用户交互，以及与可执行程序（根据被监视的znode节点的状态停止或重启）的交互。
## Executor
Executor对象是这个简单示例中的基本容器。它包含了ZooKeeper对象和DataMonitor。
```java
// from the Executor class...

public static void main(String[] args) {
    if (args.length < 4) {
        System.err
                .println("USAGE: Executor hostPort znode filename program [args ...]");
        System.exit(2);
    }
    String hostPort = args[0];
    String znode = args[1];
    String filename = args[2];
    String exec[] = new String[args.length - 3];
    System.arraycopy(args, 3, exec, 0, exec.length);
    try {
    //Executor的任务是根据命令行中输入去启动和停止的可执行程序，以响应zk对象触发的时间。(前面的args）
        new Executor(hostPort, znode, filename, exec).run();
    } catch (Exception e) {
        e.printStackTrace();
    }
}

public Executor(String hostPort, String znode, String filename,
        String exec[]) throws KeeperException, IOException {
    this.filename = filename;
    this.exec = exec;
    
    zk = new ZooKeeper(hostPort, 3000, this);
    dm = new DataMonitor(zk, znode, null, this);
}

public void run() {
    try {
        synchronized (this) {
            while (!dm.dead) {
                wait();
            }
        }
    } catch (InterruptedException e) {
    }
}
```
Executor实现了这些

```java
public class Executor implements Watcher, Runnable, DataMonitor.DataMonitorListener {
...
//DataMonitor.DataMonitorListener这个是啥？
```

```

ZooKeeper的java api定义了watcher 接口，zk用watcher与容器（如Executor）进行通信。watcher仅包含process()方法。zk利用它去传递主线程感兴趣的时间，例如zk连接或会话的状态。本例中的Executor仅将事件下发给DataMonitor ，然后由DataMonitor 决定如何处理。
>  就说Executor接收了，但不想做就交给DataMonitor 处理了？是不是也可以交由其他的呢？

```java
public void process(WatchedEvent event) {
    dm.process(event);
}
```
下面的DataMonitorListener接口是本案例设计的，不属于zkAPI。DataMonitor 对象使用它来与其容器(如Executor)进行通信（DataMonitor.DataMonitorListener）。

```java
public interface DataMonitorListener {

    void exists(byte data[]);

    /**
    *
    * @param rc
    * the ZooKeeper reason code
    */
    void closing(int rc);
```
> DataMonitor.DataMonitorListener这个是啥？ 
> 就是DataMonitor里面定义了DataMonitorListener接口，并由Executor实现了。

```java
package example;

import org.apache.zookeeper.WatchedEvent;

public class DataMonitor {
    DataMonitor dm;
    public void process(WatchedEvent event) {
        dm.process(event);
    }
    public interface DataMonitorListener {
        /**
         * 节点存活与否判断
         */
        void exists(byte data[]);

        /**
         * ZooKeeper会话失效
         * @param src
         * ZooKeeper的原因码(reason code)
         */
        void closing(int src);
    }
}

下面是Executor 对 DataMonitorListener.exists ()和 DataMonitorListener.closing 的实现:

```java
public void exists( byte[] data ) {
    if (data == null) {
        if (child != null) {
            System.out.println("Killing process");
            child.destroy();
            try {
                child.waitFor();
            } catch (InterruptedException e) {
           }
        }
        child = null;
    } else {
        if (child != null) {
            System.out.println("Stopping child");
            child.destroy();
            try {
               child.waitFor();
            } catch (InterruptedException e) {
            e.printStackTrace();
            }
        }
        try {
            FileOutputStream fos = new FileOutputStream(filename);
            fos.write(data);
            fos.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
        try {
            System.out.println("Starting child");
            child = Runtime.getRuntime().exec(exec);
            new StreamWriter(child.getInputStream(), System.out);
            new StreamWriter(child.getErrorStream(), System.err);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

public void closing(int rc) {
    synchronized (this) {
        notifyAll();
    }
}
```
## DataMonitor
DataMonitor是本程序ZooKeeper逻辑的核心，它是异步和事件驱动的。

```java
public DataMonitor(ZooKeeper zk, String znode, Watcher chainedWatcher,
        DataMonitorListener listener) {
    this.zk = zk;
    this.znode = znode;
    this.chainedWatcher = chainedWatcher;
    this.listener = listener;
    //检查znode是否存在，并设置监视。
    //传递自身作为回调对象，watcher触发时就会引起真实的处理流程
    //exists在服务器端完成，但其回调在客户端完成
    zk.exists(znode, true, this, null);
```
Note
+ 1 不要将完成回调和watch的回调搞混。exists()的完成回调——（客户机端）processResult ()是在服务器上的watch（ZooKeeper.exists()的）操作之后执行。
+ 2 Executor注册为了zk对象的一个watcher，所以watch触发时会向Executor对象发送一个事件
+ 3 zk3.0后DataMonitor也可以注册为特定事件的watcher，本例不支持。

```java
public void processResult(int rc, String path, Object ctx, Stat stat) {
        boolean exists;
        //1 检查节点是否存在
        switch (rc) {
            //节点存在
            case Code.OK:
                exists = true;
                break;
            //节点不存在
            case Code.NoNode:
                exists = false;
                break;
            //会话被服务器终止（致命错误）
            case Code.SESSIONEXPIRED:
            //未认证（致命错误）
            case Code.NoAuth:
                dead = true;
                listener.closing(rc);
                return;
            default:
                //可恢复的服务
                zk.exists(znode,true,this,null);
                return;
        }

        byte b[] = null;
        // 2 存在则从znode获取数据
        if (exists) {
            try{
                //这两句不是太理解欸
                //如果节点在调用zookeeper.getData之前被删除，
                //zookeeper.exists()设置的watch将会触发一个回调

                //如果由通信错误，连接的watch事件会在连接恢复时触发
                b = zk.getData(znode, false,null);
            } catch (KeeperException e) {
                e.printStackTrace();
            } catch (InterruptedException e) {
                return;
            }
        }
        //如果状态发生变化，调用Executor 的 exists() 回调函数
        //???这里不是太理解
        if ((b == null && b!= prevData)
              || b != null && !Arrays.equals(prevData, b)) {
            listener.exists(b);
            prevData = b;
        }
    }
```
# 完整代码
## Executor 
```java
package example;

import org.apache.zookeeper.KeeperException;
import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooKeeper;

import java.io.FileOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;

public class Executor implements Watcher, Runnable, DataMonitor.DataMonitorListener {
    String filename;
    String exec[];
    ZooKeeper zk;
    DataMonitor dm;
    Process child;

    public static void main(String[] args) {
        if (args.length < 4) {
            //（标准错误输出流）实时输出错误，显示为红色。out要累计到一定程度才输出
            //https://blog.csdn.net/weixin_42153410/article/details/94618943
            System.err.
                    println("USAGE: Executor hostPort znode filename program [args ...]");
            System.exit(2);
        }
        String hostPort = args[0];
        String znode = args[1];
        String filename = args[2];
        String exec[] = new String[args.length - 3];
        /**
         *  Object src : 原数组
         *    int srcPos : 从元数据的起始位置开始
         * 　　Object dest : 目标数组
         * 　　int destPos : 目标数组的开始起始位置
         * 　　int length  : 要copy的数组的长度
         */
        System.arraycopy(args, 3, exec, 0, exec.length);
        try {
            new Executor(hostPort, znode, filename, exec).run();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    public Executor(String hostPort, String znode, String filename,
                    String exec[]) throws KeeperException, IOException {
        this.filename = filename;
        this.exec = exec;
        //public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher)这个？
        //this是Executor对自身的引用
        zk = new ZooKeeper(hostPort, 3000, this);
        dm = new DataMonitor(zk, znode, null, this);
    }

    public void run() {
        try {
            //同一时间只能有一个线程访问
            synchronized (this) {
                while (!dm.dead) {
                    wait();
                }
            }
        }catch (InterruptedException e) {
        }
    }
    public void exists(byte[] data) {
        //没有传数据
        if (data == null) {
            //如果进程不为空
            if (child != null) {
                System.out.println("Killing process");
                //杀死子进程
                child.destroy();
                try {
                    //让线程等待到终止为止
                    child.waitFor();
                } catch (InterruptedException e) {
                    //线程在等待时中断
                    e.printStackTrace();
                }
            }
            //进程置空
            child = null;
        } else {
            //data有数据（znode存在，或发生变化？）
            if (child != null) {
                //但是进程不为空
                System.out.println("Stopping child");
                //先终止进程
                child.destroy();
                try {
                    child.waitFor();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            try {
                //将znode数据存入文件
                FileOutputStream fos = new FileOutputStream(filename);
                fos.write(data);
                fos.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
            try {
                //启动进程
                System.out.println("Starting child");
                //getRuntime 返回与当前Java应用程序关联的运行时对象(Runtime)
                //exec 在单独的进程中执行指定的字符串命令
                // 即，线程执行用户的命令
                child = Runtime.getRuntime().exec(exec);
                //两个线程输出执行结果及日志
                new StreamWriter(child.getInputStream(), System.out);
                new StreamWriter(child.getErrorStream(), System.err);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
    public void closing(int rc) {
        synchronized (this) {
            //唤醒对象的等待池中的所有线程，进入锁池
            // 和会话失效有啥关系？
            notifyAll();
        }
    }

    static class StreamWriter extends Thread {
        OutputStream os;
        InputStream is;
        StreamWriter(InputStream is,OutputStream os) {
            this.is = is;
            this.os = os;
            start();
        }
        public void run() {
            byte b[] = new byte[80];
            int rc;
            try {
                while ((rc = is.read(b)) > 0) {
                    os.write(b,0,rc);
                }
            } catch (IOException e) {}
        }
    }
    public void process(WatchedEvent event) {
        dm.process(event);
    }
}

```
## DataMonitor 
```java
package example;

import org.apache.zookeeper.*;
import org.apache.zookeeper.data.Stat;
import org.apache.zookeeper.KeeperException.Code;

import java.util.Arrays;

public class DataMonitor implements Watcher, AsyncCallback.StatCallback {

    ZooKeeper zk;
    String znode;
    Watcher chainedWatcher;
    DataMonitorListener listener;
    boolean dead;
    byte prevData[];
    public DataMonitor(ZooKeeper zk, String znode, Watcher chainedWatcher,
                       DataMonitorListener listener) {
        this.zk = zk;
        this.znode = znode;
        this.listener = listener;
        //检查znode是否存在，并设置监视。
        //传递自身作为回调对象，watcher触发时就会引起真实的处理流程
        //exists在服务器端完成，但其回调在客户端完成
        zk.exists(znode,true,this,null);
    }

    public interface DataMonitorListener {
        /**
         * 节点存活与否判断
         */
        void exists(byte data[]);

        /**
         * ZooKeeper会话失效
         * @param src
         * ZooKeeper的原因码(reason code)
         */
        void closing(int src);
    }

    /**
     * 检查节点存在与否。存在且状态变化的调用 Executor 的 exists() 回调函数
     * @param rc
     * @param path
     * @param ctx
     * @param stat
     */
    public void processResult(int rc, String path, Object ctx, Stat stat) {
        boolean exists = true;
        //1 检查节点是否存在
        Code code = Code.get(rc);
        switch (code) {
            //节点存在
            case OK:
                exists = true;
                break;
            //节点不存在
            case NONODE:
                exists = false;
                break;
            //会话被服务器终止（致命错误）
            case SESSIONEXPIRED:
            //未认证（致命错误）
            case NOAUTH:
                dead = true;
                listener.closing(rc);
                return;
            default:
                //可恢复的服务
                zk.exists(znode,true,this,null);
                return;
        }

        byte b[] = null;
        // 2 存在则从znode获取数据
        if (exists) {
            try{
                //这两句不是太理解欸
                //如果节点在调用zookeeper.getData之前被删除，
                //zookeeper.exists()设置的watch将会触发一个回调

                //如果由通信错误，连接的watch事件会在连接恢复时触发
                b = zk.getData(znode, false,null);
            } catch (KeeperException e) {
                e.printStackTrace();
            } catch (InterruptedException e) {
                return;
            }
        }
        //如果状态发生变化，调用Executor 的 exists() 回调函数
        //???这里不是太理解
        if ((b == null && b!= prevData)
                || b != null && !Arrays.equals(prevData, b)) {
            listener.exists(b);
            prevData = b;
        }
    }

    /**
     * 处理watch事件
     */
    public void process(WatchedEvent event) {
        String path = event.getPath();
        if (event.getType() == Watcher.Event.EventType.None) {
            //我们被告知连接状态已经变化
            switch(event.getState()) {
                case SyncConnected:
                    //在这个例子中，不需要做任何事情-watch自动和客户端重连和注册；
                    //客户端断连时watch依次触发
                    break;
                case Expired:
                    dead = true;
                    listener.closing(Code.SESSIONEXPIRED.intValue());
                    break;
            }
        } else {
            if (path != null && path.equals(znode)) {
                zk.exists(znode,true,this,null);
            }
        }
        if (chainedWatcher != null) {
            chainedWatcher.process(event);
        }
    }
}

```
# 启动
ZookeeperServerMain先启动，可[参考](https://editor.csdn.net/md/?articleId=119541703)

```bash
-Dlog4j.configuration=file:conf/log4j.properties
conf/zoo.cfg
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/ecf1245a35ac4ed9b2df95447cefe49a.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)

```bash
-Dlog4j.configuration=file:conf/log4j.properties
//以空格作为分割，第一个参数是地址，第二个是监视node，第三个是输出的文件地址 第四个是命令

127.0.0.1:2181 /zk_test E:/soft/kj/ZooKeeper/zookeeper-3.4.13/1.txt create
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/937ed6b52e60434791acaa984e8925b5.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)

# 参考链接
[Zookeeper 初体验之——JAVA实例](https://www.cnblogs.com/haippy/archive/2012/07/20/2600077.html)
[ZooKeeper官方Java例子解读](https://blog.csdn.net/liyiming2017/article/details/83276706)

> 挣扎了一两周，都没有完全启动成功，暂时放弃，先学学其他的。希望路过的大佬指导一下。

![在这里插入图片描述](https://img-blog.csdnimg.cn/f8247d2ec2954143a6a848b612eb7c9d.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21vcWlhbm1vcWlhbg==,size_16,color_FFFFFF,t_70)


最后求一键三连。
