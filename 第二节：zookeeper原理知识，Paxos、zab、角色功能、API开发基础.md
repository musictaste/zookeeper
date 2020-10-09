[TOC]

# zookeeper分布式协调

**zookeeper分布式协调，具备的特性：扩展，可靠性，时序性、快速！！！！**

## 扩展性：

zookeeper的官网 -> learn about  -> admin &Ops -> Observers Guide

1.从框架、架构来看，**现在所有的架构都是分布式的，关注角色；**

**角色有：leader、 follower、observer**

2.从读写分离来看，leader可以读写；follower只能读；另外注意：==只有follower才能选举==

**如果leader挂了，那么follower进行选举，而observer进行等待**

**当选举出leader后，observer追随leader，同步数据，接受用户的查询，用户对observer进行写操作时，将写操作转给leader**

observer不参与投票选举；**而投票选举的速度取决于follower的数量**

**Observer放大查询能力**

如果公司有30台机器，其中9台为follower，剩余的21台都是observer；

**另外，一般留个位数的奇数(3.5,7.9)台机器作为follower，用于选举**

**21台observer就是为了放大查询能力;另外zk官网也更倾向于让其作为一个查询集群，而非写操作**

那么observer怎么用起来？配置文件：

    zoo.cfg
    server.1=node01:2888:3888
    server.2=node02:2888:3888
    server.3=node03:2888:3888
    server.4=node04:2888:3888:observer
    
所以你公司机器特别多，只需要留个位数的机器不带observer，剩余的机器都是observer


==面试的时候，如果提到ZK了，这些点争取多说几分钟，这样面试官就听起来很饱满==

## 可靠性

如果想把zookeeper记明白，学会，给别人说明白，只需要记一句话：==攘其外必先安其内==

**zookeeper的可靠性 来源于 快速恢复leader**

**攘其外：对外提供服务；安其内：选主的过程**

说到可靠性，就会涉及到数据可用性、可靠性、一致性

攘其外，对外提供服务时的一致性是什么样的呢？虽然是最终一致性，如果5台机器，产生脑裂，三台选举出，其他两台需要进行数据同步；那么在数据同步的过程中，节点是否对外提供服务呢？

==也就是在达到最终一致性的过程汇总，节点是否对外提供服务呢？==

==小技巧：搜索文章，如果印象中某个网站：搜索内容可以为： paxos site:douban.com==


### **Zookeeper全解析——Paxos作为灵魂**

(https://www.douban.com/note/208430424/)

先说Paxos，它是==一个基于消息传递的一致性算法==，Leslie Lamport在1990年提出，近几年被广泛应用于分布式计算中，Google的Chubby，Apache的Zookeeper都是基于它的理论来实现的，**Paxos还被认为是到目前为止唯一的分布式一致性算法**，其它的算法都是Paxos的改进或简化。有个问题要提一下，**Paxos有一个前提：没有拜占庭将军问题**。**就是说Paxos只有在一个可信的计算环境中才能成立，这个环境是不会被入侵所破坏的**。

==没有拜占庭问题：就是没有黑客、局域网通信没有问题==

Paxos描述了这样一个场景，有一个叫做Paxos的小岛(Island)【zk集群】上面住了一批居民，岛上面所有的事情由一些特殊的人决定，他们叫做议员(Senator)【follower】。**议员的总数(Senator Count)是确定的**，不能更改。岛上每次环境事务的变更都需要通过一个提议(Proposal)，每个提议都有一个编号(PID)，**这个编号是一直增长的，不能倒退【Zxid】**。每个提议都需要超过半数((Senator Count)/2 +1)【过半】的议员同意才能生效。**每个议员只会同意大于当前编号的提议**，包括已生效的和未生效的。如果议员收到小于等于当前编号的提议，他会拒绝，并告知对方：你的提议已经有人提过了。这里的当前编号是每个议员在自己记事本上面记录的编号，他不断更新这个编号。整个议会不能保证所有议员记事本上的编号总是相同的。**现在议会有一个目标：保证所有的议员对于提议都能达成一致的看法。**

第一阶段：好，现在议会开始运作，所有议员一开始记事本上面记录的编号都是0。有一个议员发了一个提议：将电费设定为1元/度。他首先看了一下记事本，嗯，当前提议编号是0，那么我的这个提议的编号就是1，于是他给所有议员发消息：1号提议，设定电费1元/度。其他议员收到消息以后查了一下记事本，哦，当前提议编号是0，这个提议可接受，于是他记录下这个提议并回复：我接受你的1号提议，同时他在记事本上记录：当前提议编号为1。**【第一阶段：将操作记录到日志里，保证了数据操作可靠性的支撑】**

第二阶段：发起提议的议员收到了超过半数的回复，立即给所有人发通知：1号提议生效！收到的议员会修改他的记事本，将1好提议由记录改成正式的法令，当有人问他电费为多少时，他会查看法令并告诉对方：1元/度。**【第二阶段：】**

==【过半通过 + 二阶段提交 =分布式情况下消息的传递】==

现在看冲突的解决：假设总共有三个议员S1-S3，S1和S2同时发起了一个提议:1号提议，设定电费。S1想设为1元/度, S2想设为2元/度。结果S3先收到了S1的提议，于是他做了和前面同样的操作。紧接着他又收到了S2的提议，结果他一查记事本，咦，这个提议的编号小于等于我的当前编号1，于是他拒绝了这个提议：对不起，这个提议先前提过了。于是S2的提议被拒绝，S1正式发布了提议:1号提议生效。S2向S1或者S3打听并更新了1号法令的内容，然后他可以选择继续发起2号提议。**【冲撞解决】**

小岛(Island)——ZK Server Cluster

议员(Senator)——ZK Server

提议(Proposal)——ZNode Change(Create/Delete/SetData…)

提议编号(PID)——Zxid(ZooKeeper Transaction Id)

正式法令——所有ZNode及其数据

貌似关键的概念都能一一对应上，但是等一下，**Paxos岛上的议员应该是人人平等的吧，而ZK Server好像有一个Leader的概念**。没错，其实Leader的概念也应该属于Paxos范畴的。如果议员人人平等，在某种情况下会由于提议的冲突而产生一个“活锁”（**所谓活锁我的理解是大家都没有死，都在动，但是一直解决不了冲突问题**）。Paxos的作者Lamport在他的文章”The Part-Time Parliament“中阐述了这个问题并给出了解决方案——在所有议员中设立一个总统，**只有总统有权发出提议，如果议员有自己的提议，必须发给总统并由总统来提出**。好，我们又多了一个角色：总统。

总统——ZK Server Leader

又一个问题产生了，**总统怎么选出来的？**oh, my god! It’s a long story. 在淘宝核心系统团队的Blog上面有一篇文章是介绍如何选出总统的，有兴趣的可以去看看：http://rdc.taobao.com/blog/cs/?p=162

**【最重要的内容：总统如何选举出来的？】**

现在我们假设总统已经选好了，下面看看ZK Server是怎么实施的。

情况一：屁民甲(Client)到某个议员(ZK Server)那里询问(Get)某条法令的情况(ZNode的数据)，议员毫不犹豫的拿出他的记事本(local storage)，查阅法令并告诉他结果，同时声明：我的数据不一定是最新的。你想要最新的数据？没问题，等着，等我找总统Sync一下再告诉你。**【记事本上记录的肯定是过半通过的，只不过可能数据不同步】**

情况二：屁民乙(Client)到某个议员(ZK Server)那里要求政府归还欠他的一万元钱，议员让他在办公室等着，自己将问题反映给了总统，总统询问所有议员的意见，多数议员表示欠屁民的钱一定要还，于是总统发表声明，从国库中拿出一万元还债，国库总资产由100万变成99万。屁民乙拿到钱回去了(Client函数返回)。**【修改数据，写操作，follower将写操作交给leader，leader也需要经过两阶段提交，过半通过，才会将结果返回到客户端】**

情况三：总统突然挂了，议员接二连三的发现联系不上总统，于是各自发表声明，推选新的总统，**总统大选期间政府停业**，拒绝屁民的请求。

呵呵，到此为止吧，当然还有很多其他的情况，但这些情况总是能在Paxos的算法中找到原型并加以解决。这也正是我们认为Paxos是Zookeeper的灵魂的原因。当然ZK Server还有很多属于自己特性的东西：Session, Watcher，Version等等等等，需要我们花更多的时间去研究和学习。


#### 问题1：leader挂了影响读操作吗？-> 影响

**只要是zk server中任何一个节点，只要没有跟leader建立连接(与端口2888建立连接)，这个时候这个节点不是可用状态，会把自己的服务停掉**（进程在，但是对外不提供任何读写操作的）

过半通过，最终一致性，**什么时候才是最终？必须有leader**

**如果leader挂了，那么连最终一致性都保障不了**

### ZAB协议

![109](EC5E7A10F01F4B6196F78E44D6B67240)

==ZAB协议更容易实现在数据不一致的情况下的数据的同步==

**ZAB:zookeeper的原子广播协议**

**ZAB作用在可用状态（有leader）**

**ZAB协议是Paxos协议的精简版**


    1.客户端向follower发起请求：create ooxx
    2.follower把请求转发给leader
    3.leader创建了事务ID  Zxid=8
    
    ZAB是原子广播
    原子：要么成功，要么失败，没有中间状态
    广播：分布式多节点，不代表全部都能听到
    队列：FIFO、顺序性
    
    4.第一阶段：给两个follower发起写日志的操作，zk的数据状态在内存，用磁盘来保存日志，leader为每个follower维护了一个发送队列【将日志写到磁盘】
    
    follower返回消息给leader
    
    如果只有一台follower返回ok，一台follower因为网络延迟导致没有返回ok，leader自己也给出ok，已经过半
    
    第二阶段：leader利用发送队列，将数据写回到两个follower中【将数据写到内存】
    
    follower写成功以后 给reader返回ok
    
    **虽然一台follower没有返回ok，但是如果消息过半确认，那么数据也会写到这台follower中，这就保证了数据的最终一致性**
    
    **两阶段提交就相当于zk利用队列走了一个原子广播协议，广播不代表所有follower都能收到，但是一定过半**
    
    5.follower给client返回ok
    

---

    如果client请求另外一台follower,foller可以通过sync到leader去同步数据
    
==虽然一台follower没有返回ok，但是如果消息过半确认，那么数据也会写到这台follower中，这就保证了**数据的最终一致性**==  

**==两阶段提交==就相当于zk利用==队列==走了一个==原子广播协议==，广播不代表所有follower都能收到，==但是一定过半==**
    
==原子：要么成功，要么失败，没有中间状态(队列+2PC[两阶段提交])==
 
 ==广播：分布式多节点，不代表全部都能听到(过半)==
 
==ZAB是paxos的简化版，主从模型，主是单点，单点处理顺序性的事情，维护队列，维护事务的两阶段提交、维护过半的事==

# 答疑

## 问题：zk挂掉以后，还会对外提供服务吗？不会

不会；

zk leader挂掉以后，会选举新的leader，然后跟follower/observer建立连接，同步数据，然后再对外提供服务

一个follower挂掉以后重新启动，因为已经有主从，通过端口3888知道是leader，然后再通过2888端口进行通信，同步数据；同步数据之后，再对外提供服务

## 问题：如果一个client有写请求，leader还没有做第二阶段提交（还没有给follower提交写操作），leader挂了，这时候会发生什么？

zk服务集群不再对外提供服务，并且刚才事务回滚

## zk的源码可以不用看

## 如果客户端没有调用sync,会不会取到没有同步的数据？是的，会

# 重启集群或leader挂了重启

![201](0BB648CB5ED6443993DF4F72FD32D569)

leader启动涉及到两个场景：

场景1：第一次启动集群【这时候没有数据，相对简单，之前已经讲过】

场景2：重启集群，或leader挂了以后再重启


---

接下来需要明确一些知识点：

节点自己会有myid、还有事务id:Zxid

**新的leader选举的条件是什么呢？**

**1.Zxid：经验最丰富的的，数据最全的**

**2.myid：年龄大的**

**注意一个前提：过半通过的数据才是真数，你见到的可用的Zxid都是过半通过的**

场景1：第一次启动的过程

    node01 :myid=1  zxid=0
    node02 :myid=2  zxid=0
    node03 :myid=3  zxid=0
    node04 :myid=4  zxid=0
    
**因为事务id都为0，比较myid；在启动过程中，4台机器过半就能选举出leader，也就是说启动3台机器就能推举出leader，不需要4台全部启动**
    
场景2：重启集群

    node01 :myid=1  zxid=8
    node02 :myid=2  zxid=8
    node03 :myid=3  zxid=7  
    node04 :myid=4  zxid=8  曾经的leader
    
    投票过程：通过端口3888
    
    如果node03最早发现leader挂了，那么它发起投票，通过socket连接将自己的数据【myid=3,zxid=7】发送给node02/01；它给自己也发送了投票信息【node03+1】；
    
    node02/01接受到的投票数据发现，zxid比自己zxid小，把数据丢弃；并且将自己的推举数据【myid=1,zxid=8】【myid=2,zxid=8】发给node03
    
    注意：收到投票信息的节点，如果不同意投票，也会发起投票；并且也会给自己投一票，也会广播给node01
    
    这时候node03中的投票信息有【node03+1】【node01+1】【node02+1】
    node01中的投票信息有【node01+1】【node02+1】
    node02中的投票信息有【node02+1】
    
    node03/01认为node02的投票数据比自己的大，表示同意，就把node02的投票信息又发送给node02
    
    这时候node03中的投票信息有【node01+1】【node02+1】
    node01中的投票信息有【node02+1】
    node02中的投票信息有【node01+3】
    
    node01/node03不仅把投票信息返回给node02，也会发给跟它建立连接的其他节点；也就是node01给node03发送【node02+1】；node03给node01发送【node02+1】
    
    这时候node03中的投票信息有【node02+3】
    node01中的投票信息有【node02+3】
    
    最后的结果一定是三个节点中都是【node02+3】
    
**选举的过程是很快的，也就是为什么可以在200ms之内可以选举出新的leader**

**zk的选举的过程也是简单的**

## ZK选举过程

1. ==3888造成两两通信！==
2. ==只要任何人投票，都会触发那个准leader发起自己的投票==
3. ==推选制：先比较zxid，如果zxid相同，再比较myid==


# Watch 观察 +Callback

![202](00294A227C5440DCA1D863ADAE750662)

ZK有统一视图（同步数据）、目录树结构

如果节点中有条目/ooxx，客户端A在ooxx下 有自己的目录a;客户端B需要用到a的服务

ZK是做服务协调的，不是提供服务的，那么如果客户端A是Enode（ephemeral临时节点），那么这个客户端连接是有session的

**如果没有ZK，那么需要客户端B向客户端A发起心跳验证，心跳验证是有时间窗口的**

**如果有ZK，伴随目录树结构是有事件的，具体的事件有：creat/delete/change/children（子目录），这样通过事件，ZK就可以主动通知到客户端B**

客户端B访问目录/ooxx/a, 第一步是 get /ooxx/a;第二步是：watch /ooxx/a

通过watch就可以实时得到目录的变化，如果有事件，ZK 回调callcack客户端B

## 所以：心跳验证和watch的区别是什么呢？

**心跳验证 是 客户端自己实现；watch基于zk**

**还有就是方向性、时效性**
        
    心跳验证的方向性是客户端B到客户端A
    watch的方向性是客户端A->ZK->客户端B
        
    心跳验证的时效性：差，有时间窗口
    watch的时效性：强，根据事件主动提醒
    
==注意：客户端要先Watch，这样事件发生时，ZK才能callback==  


## 闲聊Netty的重要性

**Netty在面试中含金量才是最高**的

**一台服务器使不使用Netty、reactor模型是使一台服务器性能发挥50%，还是80%**

**luttce（redis API）底层是netty；dubbo底层也是netty，spark也可以切换成netty底层**

## API


```
package com.msb.zookeeper;

import org.apache.zookeeper.*;
import org.apache.zookeeper.data.Stat;

import java.util.concurrent.CountDownLatch;

/**
 * Hello world!
 *
 */
public class App 
{
    public static void main( String[] args ) throws Exception {
        System.out.println( "Hello World!" );

        //zk是有session概念的，没有连接池的概念
        //watch:观察，回调
        //watch的注册值发生在 读类型调用，get，exites。。。
        //第一类：new zk 时候，传入的watch，这个watch是session级别的，跟path 、node没有关系。
        final CountDownLatch cd = new CountDownLatch(1);
        //虽然配置了多个zk节点，但是客户端连接时是随机连接
        final ZooKeeper zk = new ZooKeeper("192.168.150.11:2181,192.168.150.12:2181,192.168.150.13:2181,192.168.150.14:2181",
                3000, new Watcher() {
            //Watch 的回调方法(callback)！
            @Override
            public void process(WatchedEvent event) {
                Event.KeeperState state = event.getState();
                Event.EventType type = event.getType();
                String path = event.getPath();
                System.out.println("new zk watch: "+ event.toString());

                switch (state) {
                    case Unknown:
                        break;
                    case Disconnected:
                        break;
                    case NoSyncConnected:
                        break;
                    case SyncConnected:
                        System.out.println("connected");
                        cd.countDown();
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

                switch (type) {
                    case None:
                        break;
                    case NodeCreated:
                        break;
                    case NodeDeleted:
                        break;
                    case NodeDataChanged:
                        break;
                    case NodeChildrenChanged:
                        break;
                }


            }
        });

        cd.await();
        ZooKeeper.States state = zk.getState();
        switch (state) {
            case CONNECTING:
                System.out.println("ing......");
                break;
            case ASSOCIATING:
                break;
            case CONNECTED:
                System.out.println("ed........");
                break;
            case CONNECTEDREADONLY:
                break;
            case CLOSED:
                break;
            case AUTH_FAILED:
                break;
            case NOT_CONNECTED:
                break;
        }

        String pathName = zk.create("/ooxx", "olddata".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL);
        final Stat  stat=new Stat();
        byte[] node = zk.getData("/ooxx", new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                System.out.println("getData watch: "+event.toString());
                try {
//                    zk.getData("/ooxx",true,stat);  //true: default Watch  被重新注册,  是new zk的那个watch
                    zk.getData("/ooxx",this  ,stat);
                } catch (KeeperException e) {
                    e.printStackTrace();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, stat);

        System.out.println(new String(node));

        //触发回调
        Stat stat1 = zk.setData("/ooxx", "newdata".getBytes(), 0);
        //还会触发吗？watct是一次性的
        Stat stat2 = zk.setData("/ooxx", "newdata01".getBytes(), stat1.getVersion());

        //异步回到
        System.out.println("-------async start----------");
        zk.getData("/ooxx", false, new AsyncCallback.DataCallback() {
            @Override
            public void processResult(int rc, String path, Object ctx, byte[] data, Stat stat) {
                System.out.println("-------async call back----------");
                System.out.println(ctx.toString());
                System.out.println(new String(data));

            }

        },"abc");
        System.out.println("-------async over----------");



        Thread.sleep(2222222);
    }
}

```


mvn依赖，找服务器使用的zk的版本号，要对应起来

版本3.4.6

**IDEA小技巧：event.getState().var直接生成默认变量**

**IDAE小技巧：switch(status) alt+enter可以自动补全case 可能存在的情况**

因为ZK是先异步建立连接，为了解决异步同步数据的问题，引入CountDowmLatch

只有集群真正连接以后，callback回调，返回SyncConnected事件时，才解除阻塞

session设置了3秒，3秒以后，目录就没有了

create 有两类API，一个是同步阻塞，一个是异步非阻塞(Reactor模型)

    data[]：二进制安全，所以入参是字节数组
    CreateMode：节点类型有四种：永久、临时，永久序列、临时序列
    StringCallback：异步回调方法
    List<ACL>：???
    
    返回pathname: 在序列节点时，path不固定，所以需要返回
    

getData 分为两大类【同步阻塞、异步非阻塞】，四种方式

    path:取数据的路径
    
    boolean watch: false[只取数据，不观察未来是否有事件]，true[watch回调以后，再写一个watch重新监听、因为watch是一次性的]
    
    stat：原数据，也就是Zxid等信息
    
    Watcher：针对这个path，再来一个不同的watch，因为watch是一次性的
    
    DataCallback:异步回调方法
    
**setData -》返回stat:返回原数据**
    
    
运行结果：203.png   204.png    205.png

![203](F0E161F3671846DABFAA6674691C6295)
![204](4F263FB9D48441BDBDD03AE8A4238A60)
![205](2077C8891D9A4BA7BA9670E125B6E3FD)

**因为watch是一次性的，所以当事件发生以后，watch就会消失，如果想要继续watch观察，在getData的watch中再次设置watch【zk.getData("ooxxx",true,stat)】**



---


日志

![206](D1C94F71134A4CAD930C255C8A53EC7D)

连接13节点以后，把13节点的zk服务挂掉以后，客户端重新连接14节点

    1.new zk时的watch是session级别的，发生在session上的事件会被监听到
    2.如果连接的zk server挂了，会切换到别的zk server
    3.切换zk server以后，sessionID不会变

![207](4AB0296E02F944D69406EB91604537C2)

---

异步回调

    DataCallback：
        rc：回调状态码
        path：路径
        ctx：上下文
        data：节点数据
        stat：节点原数据
        
运行结果：208.png
![208](426414457E4B42718340A768DC8F52CC)