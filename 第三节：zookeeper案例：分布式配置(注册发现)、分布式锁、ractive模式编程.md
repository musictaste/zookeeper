[TOC]

# zookeeper案例

**zookeeper 是做分布式协调的，接下来通过几个场景演示一下怎么协调？协调什么东西？**


分布式：很多节点、进程在不同的物理位置，它们之间有协作，需要互相传递数据等等

## 场景1：分布式配置、配置中心、注册发现

**独立的位置存放配置文件，可以是DB、redis、ZK**

**ZK存放配置文件最大的优势：不需要每隔一段时间去查询配置，当配置文件修改时，如果watch了，修改配置文件就会回调你，可以第一时间通知到**

配置数据放在ZK中，客户端可以通过get获取配置信息，也可以watch配置信息（当配置信息有修改时，会callback客户端）

笔记：补代码思路


```java
package com.msb.zookeeper.config;

import org.apache.zookeeper.ZooKeeper;
import java.util.concurrent.CountDownLatch;

/**
 * @author: 马士兵教育
 * @create: 2019-09-20 20:08
 */
public class ZKUtils {

    private  static ZooKeeper zk;

    private static String address = "192.168.150.11:2181,192.168.150.12:2181,192.168.150.13:2181,192.168.150.14:2181/testLock";

    private static DefaultWatch watch = new DefaultWatch();

    private static CountDownLatch init  =  new CountDownLatch(1);
    public static ZooKeeper  getZK(){

        try {
            zk = new ZooKeeper(address,1000,watch);
            watch.setCc(init);
            init.await();

        } catch (Exception e) {
            e.printStackTrace();
        }

        return zk;
    }

}
```


```java
package com.msb.zookeeper.config;

import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;

import java.util.concurrent.CountDownLatch;

/**
 * @create: 2019-09-20 20:12
 * DefaultWater  跟session有关系，跟path没有关系
 */
public class DefaultWatch  implements Watcher {

    CountDownLatch cc ;

    public void setCc(CountDownLatch cc) {
        this.cc = cc;
    }

    @Override
    public void process(WatchedEvent event) {
        System.out.println(event.toString());
        switch (event.getState()) {
            case Unknown:
                break;
            case Disconnected:
                break;
            case NoSyncConnected:
                break;
            case SyncConnected:
                cc.countDown();
                break;
            case AuthFailed:
                break;
            case ConnectedReadOnly:
                break;
            case SaslAuthenticated:
                break;
            case Expired:
                break;
        }
    }
}
```

```
package com.msb.zookeeper.config;

/**
 * @author: 李淼
 * @create: 2019-09-20 20:28
 * 这个class才是你未来最关心的地方‘
 * 依据：未来配置文件的格式
 */
public class MyConf {

    private  String conf ;

    public String getConf() {
        return conf;
    }

    public void setConf(String conf) {
        this.conf = conf;
    }
}
```

```java
package com.msb.zookeeper.config;

import org.apache.zookeeper.AsyncCallback;
import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooKeeper;
import org.apache.zookeeper.data.Stat;

import java.util.concurrent.CountDownLatch;

/**
 * @author: 马士兵教育
 * @create: 2019-09-20 20:21
 */
public class WatchCallBack  implements Watcher ,AsyncCallback.StatCallback, AsyncCallback.DataCallback {
    ZooKeeper zk ;
    MyConf conf ;
    CountDownLatch cc = new CountDownLatch(1);

    public MyConf getConf() {
        return conf;
    }

    public void setConf(MyConf conf) {
        this.conf = conf;
    }

    public ZooKeeper getZk() {
        return zk;
    }

    public void setZk(ZooKeeper zk) {
        this.zk = zk;
    }

    //语义：无论节点存在不存在，取节点数据
    public void aWait(){
        //路径是：/testLock/AppConf
        zk.exists("/AppConf",this,this ,"ABC");
        try {
            cc.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    //有数据的callback，getData触发的callback
    @Override
    public void processResult(int rc, String path, Object ctx, byte[] data, Stat stat) {

        if(data != null ){
            String s = new String(data);
            conf.setConf(s);
            cc.countDown();
        }
    }

    //状态的callback,exists触发的callback
    @Override
    public void processResult(int rc, String path, Object ctx, Stat stat) {
        //节点存在的话，get数据
        if(stat != null){
            //getData消耗一个watch以后，再注册了一个watch
            zk.getData("/AppConf",this,this,"sdfs");
        }

    }

    //对节点的watch
    @Override
    public void process(WatchedEvent event) {

        switch (event.getType()) {
            case None:
                break;
            case NodeCreated:
                zk.getData("/AppConf",this,this,"sdfs");

                break;
            case NodeDeleted:
                //容忍性：如果要求数据一致性就取配置，不要求数据一致性则不取配置
                conf.setConf("");
                cc = new CountDownLatch(1); //重新设置为1
                break;
            case NodeDataChanged:
                zk.getData("/AppConf",this,this,"sdfs");

                break;
            case NodeChildrenChanged:
                break;
        }

    }
}


```

```
package com.msb.zookeeper.config;

import org.apache.zookeeper.ZooKeeper;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;

/**
 * @author: 马士兵教育
 * @create: 2019-09-20 20:07
 */
public class TestConfig {
    ZooKeeper zk;

    @Before
    public void conn (){
        zk  = ZKUtils.getZK();
    }

    @After
    public void close (){
        try {
            zk.close();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    @Test
    public void getConf(){
        WatchCallBack watchCallBack = new WatchCallBack();
        watchCallBack.setZk(zk);
        MyConf myConf = new MyConf();
        watchCallBack.setConf(myConf);

        watchCallBack.aWait();
        //1，节点不存在
        //2，节点存在

        //真正的业务代码在这儿：要么打印，要么等着去数据
        while(true){
            if(myConf.getConf().equals("")){
                System.out.println("conf diu le ......");
                watchCallBack.aWait();
            }else{
                System.out.println(myConf.getConf());
            }

            try {
                Thread.sleep(200);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

```



```
代码；ZKUtils
ip:端口+path    kafka也会使用zk，这样别人看不到这个目录 

DefaultWater  跟session有关系，跟path没有关系

zk.exists  有两大类，四种实现形式的API

代码逻辑梳理：节点存在、节点不存在

需要手动在ZK创建节点：create /testLock

手动创建配置：create /testConf/Appconf "olddata"

手动修改配置：set /testConf/Appconf "newData"

delete /testLock/Appconf

```

### 作业：昨天、今天的代码敲10遍

注意：梳理代码思路

### 注册发现：将IP+port 换乘 childern

### 作业：有兴趣把ractive模型改为阻塞模型

### ZK的知识点+Ractive模型 =ZK知识最大化

## 场景2：分布式锁

分布式锁是现在考的比较火的，虽然工作中不会接触分布式锁，但是面试官可以通过这个知识get到你知道很多的知识点

两个应用/程序 访问同一个资源，争抢资源访问权，引入锁的概念

JVM的锁，只能锁住应用本身；两个应用都相当于只锁住自己，解决不了问题

redis、DB、文件都可以做锁

==为什么redis不适合做分布式锁？==**redis的特点是快、但是reids是单点的，存在单点故障问题，需要持久化，持久化就会染指磁盘IO，就会降低性能，速度变慢；另外如果redis挂了，又得重新锁，很麻烦**

做分布式锁，最合适的就是Zookeeper

**以Zookeeper为第，用Ractive编程模型，手写分布式锁**

**1. 争抢锁，只有一个人能获得锁**
**2. 获得锁的人出问题，会造成死锁，解决方案：临时节点（session）。**
**3. 获得锁的人成功了，会释放锁。**
**4. 锁被释放、删除，别人怎么知道？**

4-1：主动轮询，心跳。。。弊端：延迟(时序性不强)，压力（多台并发轮询）

4-2：watch：  回调可以解决延迟问题。。  弊端：压力（通信压力；通知大家 + 大家重新抢锁）

**4-2：sequence+watch：watch谁？watch前一个，最小(近)的获得锁~！一旦，最小的释放了锁，成本：zk只给第二个(紧挨着的)发事件回调！！！！**


思路： 多线程：每个线程：通过zk抢锁-》业务处理-》释放锁

每个线程中的CountDowmLatch、Watch、callback都是独立的

Ractive模型：没有返回类型

抢锁：创建序列+临时节点 -》创建节点的callback   -> 节点创建成功，名称不为空， 并拿到其他的节点【不需要关注锁目录的变化，只需要关注其他节点的callbake】   -》 children2Callback【核心方法】 -》 并发情况下线程不是顺序拿到节点的（抢锁），并且一定可以看到之前创建的节点 -》 取到的子节点，是乱序的，需要排序  -》 是不是第一个；是：countdowm；不是：监控前一个节点的状态，是否释放锁 -》 watch【前面一个节点的删除事件 】 、 回调 （细节）【需要回调，因为前面一个节点有可能失败】 -》 StatCallback  --》 watch前面一个节点的删除事件  -》关心的时候前面一个节点，所以不需要watch(最烧脑)

业务处理后，释放锁  -》 删除节点

运行结果：209.png
![209](E3E3110E61214BE3986264EE3993C334)

注意：如果把业务处理的睡眠去掉，这样执行速度很快，第一个节点抢锁、干活、释放锁；第二个节点监控了第一个节点，但是没有捕捉到删除节点的事件【执行速度太快了】 -》解决：1.可以睡眠；2.setData,重入锁


问题：countdownLatch（10）设置为10可以吗？不可以

因为10个线程访问jvm的内存状态，现在是zk的数据状态，已经移到外部了

在分布式情况下，JVM的锁不能实现，所以不用考虑JVM的实现方式，要依赖外部的实现方式

IDEA小技巧：iter +回车，生成默认的迭代器

### 作业：代码敲10遍，最好能够自己默写出来

### 面试：netty占30/40， redis/zk占30/40   spring+其他剩余部分

### 视频一定要多看，记笔记