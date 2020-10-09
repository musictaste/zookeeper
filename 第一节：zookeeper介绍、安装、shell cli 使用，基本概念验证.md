[TOC]

# redis回顾

**redis是单线程单进程单实例的，底层系统调用时epoll**

**因为redis是内存数据库，特点就是快**

因为redis是单实例的，必然会复制集群；既然是集群必然会有HA高可用，**redis集群高可用的方案就是哨兵sentinel**

客户端访问redis集群时，访问哨兵，不要访问主从中的主机

**主从复制时，也不是绝对的实时同步（全量RDB+增量日志AOF）**，**并且可能连最终一致性都谈不上(因为有一部分数据在缓存中，没有及时写到磁盘中)**

**集群模式是sharding分片的（modula、random、kemata）**

**因为分片，所以数据是分散各个节点上，那么聚合操作、事务就不能很好保证**

**也因为集群，那么完成分布式协调是很难的**，**解决分布式协调需要分布式锁（sennx + key过期时间 + 多线程【守护线程】来延长key的过期时间）**


# zookeeper介绍

zookeeper -> 分布式协调服务(分布式锁)，并且分布式锁只是zk功能的冰山一角

官网：http://zookeeper.apache.org/

下载

## 学习-overview

分布式应用程序的分布式协调服务

原语 simple API  ->  **高级服务，实现分布式同步、配置、分组**

设计目标：分层命名空间，类似标准文件系统

application可能是同一类软件或进程，有相同的东西：锁、或其他资源

覆盖，分层可以独立

重视高性能、高可用、严格有序的访问

**zk的数据保存在内存中，意味着zk可以实现高吞吐量和低延迟，跟redis很像**

## 集群模型

zk会当做集群来使用，

**以集群的模型的角度：主从模型、无主cluster；**

**机器存储数据的角度：分片集群，复制集群**

**ZK是复制集群（写操作【增删改】发生在主机，查询发生主和从机上）,主从集群**

图server-client


**ZK server有一对服务server，供client使用，其中有一个leader ，说明是主从集群**

**看到主从集群，第一反应就是主机的单点故障问题**

为了解决单点故障问题，引入新的技术

![101](81DCBABDC85D4C02BAAF060C3A0AD126)

推导过程：

leader肯定会挂 ->那么服务不可用  -> 就是不可靠的集群 -> 事实上，zk集群及其高可用 -> **如果有一种方式可以快速的恢复出一个leader**

zookeeper有2中运行状态：1，可用状态；2，不可用状态【原因就是leader是否可用】

**那么 不可用状态恢复到可用状态应该越快越好**

现在将文章拉到下方的Performance 评测中定义了5个事件

有两次leader不可用的情况，对应的可靠性的图

**结论：zk选择新领导的时间不到200毫秒，随着leader的恢复，又可以恢复高吞吐量**

**zk很快，ZooKeeper应用程序可在数千台计算机上运行，​​并且在读取比写入更常见的情况下，其性能最佳，比率约为10：1。**

下面的性能图

==当100%是读操作时，有13个节点，吞吐量可以达到14W；3个节点也可以达到8/9万==

## 持久化节点、临时节点、序列节点

![103](4AB1A060C84E436AA654E04138BCAD92)

目录树结构


注意：虽然每个节点都可以存数据，==但是节点里可以存的数据很少，只有1M==

**为什么限制节点存储的数据大小？**==不要把zk当做数据库用==

快速响应，节点传输的体量要小

==spark消费kafka中的数据，消费数据的偏移量offset都会记到zk中，如果未来一个机器重启，可以从zk中拿到偏移量offset，看起来zk可以协调分布式集群中的数据一致性，但是其实错了，这种情况就是zk当数据库用了==，造成了大量的读写，这样就弄反了


---


目录树结构中有节点的概念，官方上线是1M，有持久化节点和临时节点

补一个概念：每个客户端连接到zk的服务，都有一个session会话来代表这个客户端

任何一个客户端连接到tomcat也都有一个session

session会话有创建、销毁的过程，那么就会有临时节点

**有session时，zk做分布式协调时，就可以支持事件通知**

**如果在使用zk的时候，创建了一个session，在会话中创建了一把锁**（redis分布式锁中的锁有过期时间，如果锁过期了，就会造成别的连接抢占这把锁),

在客户端在判断要不要更新过期时间的时候，是周期判定，是有一个时间窗的，

**当zk有session过期时间点的话**，依托session创建了一个临时节点，如果我在，那么session就会在；如果我删除了节点，那么session就会消失；如果我挂了，因为有过期时间，那么session也会消失

这样对比redis分布式锁要设置过期时间，zk的session不需要设置过期时间；并且锁如果没有执行完毕，需要多线程【守护线程】去延迟过期时间

==redis分布式锁中 过期时间+多线程更新过期时间  = zk中的session==

==这就是zk为什么有临时节点==


---

还支持序列节点，**序列节点可以是持久节点，也可以是临时节点，只是带了序号**


## 四个特征/保障

![102](190AF784952540D9999F6032FA7DE8C7)

承诺：由于其目标是构建更复杂的服务（如同步），因此提供一系列的保证。这些是：

- **顺序一致性**-来自客户端的更新将按照发送的顺序应用。类似redis的顺序一致性，**由集群模型的主从模型保证的**

- **原子性**-更新成功或失败。没有部分结果。**是面向集群的，要么全成功，要么全失败，是最终一致性，过半原则**

- **单个系统映像**-无论客户端连接到哪个服务器，客户端都将看到相同的服务视图。**也就是说，即使客户端故障转移到具有相同会话的其他服务器，客户端也永远不会看到系统的较旧视图**。
    
    无论客户端连接哪个服务，其他服务都能看到


- 可靠性-应用更新后，此更新将一直持续到客户端覆盖更新为止。

    **就是持久性，因为是内存性的，所以也会有镜像和日志**

- 及时性-确保系统的客户视图在特定时间范围内是最新的。

    **也是最终一致性 ；发起sync，会找leader去同步，在特定时间范围内提供最新的**




---

**有主leader，有从follow，client可以访问follow，如果client有写操作，那么写操作必然会转到leader上**

==注意：模型的切换是面试常问的==


# 安装

建议从linux转到一个没有任何安装软件的新linux系统中

**zk是基于java语言开发的**
    
    下载zookeeper
    tar xf zookeeper-3.4.6.tar.gz  # 不要用yum安装zk，只对oracle的jdk进行测试，其他不保证
    mkdir /opt/mashibing
    mv zookeeper-3.4.6 /opt/mashibing
    cd  /opt/mashibing
    cd zookeeper-3.4.6
        #目录结构 bin  z
    cd ../conf
        #zoo_sample.cfg 配置模板
    cp zoo_sample.cfg   zoo.cfg
    vi zoo.cfg
        tickTime=2000 #2000毫秒，心跳隔间时间，维护对方是否还活着
        initLimit=10  #  等待2s*10 =20s的初始延迟
        syncLimit=5 # 2S*5=10秒 leader给follow下发数据同步等协作时，超过10S认为follow有问题
        
        dataDir=/var/mashibing/zk       #var存放临时数据,日志、快照、myID，数据文件放在这个目录下
    
        clientPort=2181  #客户端连接zk服务的端口号
        #maxClientCnxns=60  #允许客户端最大连接数
        
        #redis中master、slave中过半的数是手动配置的，只给出了过半的数，没有给出哨兵都有哪些；哨兵通过发布订阅，订阅了master的通道知道还有其他别的哨兵，这是一种形式，通过发现的形式
        
        server.1=node01:2888:3888
        server.2=node02:2888:3888
        server.3=node03:2888:3888
        server.4=node04:2888:3888
        #zk需要人为的配置好都有哪些节点   n/2+1就是过半数，需要人为规划节点数
        
        #端口号3888：leader没有启动或挂掉的情况下，通过这个端口号建立连接，连接建立以后，在这个端口号的socket中进行通信、投票，选出一个leader
        
        #端口号2888：leader选出来以后，会启用这个端口号，其他节点通过这个端口号建立socket连接，后续的创建节点、事务等等行为是在这个端口下进行通信的

        #server.n :能够在200ms中推举出leader，其实不是推举出来的，是谦让出来的；目前配置的情况来，过半原则，要么节点3，要么节点4成为leader，，注意这个误区
        
    pwd   //  #/opt/mashibing/zookeeper-3.4.6/conf
    
    mkdir -p  /var/mashibing/zk  #因为mashibing目录不存在，所以需要加-p
    cd /var/mashibing/zk
    vi myid
        1  #因为当前节点是1，所以写1


    #分发
    scp -r ./mashibing/  node02:`pwd`
    
    #登录节点node02,配置文件都是一样的
    mkdir -p /var/mashibing/zk
    echo 2 > /var/mashibing/zk/myid  # 将2重定向到myid中
    cat /var/mashibing/zk/myid
    
    
    #分发节点03
    scp -r ./mashibing/  node03:`pwd`
    mkdir -p /var/mashibing/zk
    echo 3 > /var/mashibing/zk/myid
    
    
    #节点04
    scp -r ./mashibing/  node04:`pwd`
    mkdir -p /var/mashibing/zk
    echo 4 > /var/mashibing/zk/myid
    
    #完成四个节点的部署
    
    vi /etc/profile  # 增加环境变量
        export ZOOKEEPER_HOME=/opt/mashibing/zookeeper-3.4.6
        export PATH=$PATH:$ZOOKEEPER_HOME/bin
    
    .  /etc/profile  #加载配置文件，到任意目录zk就可以执行命令了  注意：.空格 配置文件
    
    scp /ect/profile node02:/etc
    scp /ect/profile node03:/etc
    scp /ect/profile node04:/etc
    
    source /etc/profile  #在下面写source /etc/profile，选择右边的“发送全部会话”，这个命令就会发送所有会话了
    
    #启动
    zkServer.sh start-foreground #前台启动，前台会阻塞，并打印日志
        #node03启动以后，node01的日志中显示：Getting a diff from the laeder
        #node03  zkServer.sh status可以看到Mode:Leader,node01,node02的mode:follower
        
        #node04启动，Getting a snapshot from leader数据同步，最终一致性
        
    #node03挂掉，node04为新leader
    #现在推举是根据myid来进行，后续如果运行很长时间，推荐时会根据谁的数据更完整；如果有多份完整数据的节点，会再根据myid进行推举
    
**推举：(数据一致)myid值大的为leader**

**推荐：（数据不一致）数据完整的为leader**  

## 验证session、节点特征、节点恢复、节点选举
    
    #连接客户端
    zkCli.sh
    
    help
        #ls  path  因为是文件系统，显示目录结构
        
    create
        # create [-s] [-e] path data acl  必须有data数据，可以为空“”
        #-s  序列节点    -e 临时节点
    create /ooxx ""
    create /ooxx/xxoo ""

    set /ooxx "hello"  #放入的数据最大为1M，也是二进制安全
    get /ooxx
        zk是单机程序，所以维护一个递增的数据很容易
        # cZxid=0x200000002 : 创建的事务id
        c:create 创建
        64位，16进制，每一位代表4个二进制位；4个二进制位=16
        后32位=事务递增序列的值
        前面的32位如果全部为0，可以省略不写
        前32位= leader的纪元，也就是你是第几个纪元
        
        #ctime 创建时间
        
        #mZxid=0x200000004  修改的事务id
        
        #mtime 修改的时间
        
        #pZxid=0x00000003  当前节点ooxx下，最后创建的节点xxoo的id号，是创建的事务id，不是修改的事务id
        
        #ephemeralOwner=0x0 临时归属者 ，0x0代表没有归属者，创建时没有带选项，所以就是持久节点
    
    set /ooxx/xxoo "fafa"
    create /ooxx/oxox "12131"
    get /ooxx #pZxid=0x200000006
    
    
    #ephemeralOwner
    zkCli.sh
        #zk客户端连接以后，在zk server的日志中可以看到有客户端请求连接，并创建了sessionID
        client attempting to establish new session at /127.0.0.1:45419
        Established session 0x45d3a399275002 with negotiated timeout 30000 for client /127.0.0.1:45419
        
    create -e /xoxo "123"  #-e = ephemeral  临时节点
    get /xoxo
        #ephemeralOwner=0x45d3a399275002
    quit #客户端退出以后，xoxo这个节点就没有；没有退出之前，别的客户端是可以看到这个节点的
    
临时节点，伴随session的会话期的

==面试题：客户端连接zk server以后，会分配一个sessionID，并且呢创建了一个临时节点；现在呢连接的这个server挂了，重新连接到另一个zk server,那么问题是：这个session创建的临时节点还存在吗？ 存在==

==原因：集群中的server会统一视图，统一的视图中包含sessionID==

**连接server的时候会分配一个sessionID，会走leader将sessionID写给所有服务的内存；（客户端记录这个sessionID);如果连接的server挂了，在超时时间之内又连接到别的server，那么sessionID是不会消失的,并且创建的临时节点也不会消失**

**中间差了一个事务id（这个事务id就是用于同步sessionID的）**

==连接客户端、断开客户端都会消耗事务id==

    #验证统一视图中的sessionID
    zkCli.sh #连接node04的客户端
    create /xoox "fafa" #cZxid=0x200000010
    
    zkCli.sh #**连接node02的客户端；连接server的时候会分配一个sessionID，会走leader将sessionID写给所有服务的内存；（客户端记录这个sessionID);如果连接的server挂了，在超时时间之内又连接到别的server，那么sessionID是不会消失的,并且创建的临时节点也不会消失
    
    crete /oxxo "edsfaf"   #cZxid=0x200000012,中间差了一个事务id（这个事务id就是用于同步sessionID的）
    
    #node02的客户端quit,消耗了一个事务id0x200000013
    
    create aaa "aaa" # cZxid=cZxid=0x200000014 ,

==结论：临时节点会创建session，会消耗统一视图，会消耗事务ID==

#**多个客户端操作一个节点，可能会发生覆盖**

    create -s /aaa/xxx "fafaf" #会在后面自动拼接一个数值
    create -s /aaa/xxx "fafaf" #另一个节点的客户端也操作这个节点，拼接的数值会递增
    
**分布式情况下，如何规避覆盖的情况：create -s 创建序列节点**
        
客户端想在zk上各自创建属于自己的节点，又担心把别人的节点覆盖掉，可以采用create -s

可以做数据隔离，统一命名

**当前的节点把创建的序列节点删除后，再次新建，序列号是递增的**
    
    rmr /abc/xxx000000001
    create -s /abc/xxx   #结果是xxx000000002

==注意：笔试时，会问cZxid、mZxid、pZxid代表的意思==

    cZid:创建的事务id
    mXid:修改的事务id
    pXid:当前节点下，最后创建的节点的事务id，是创建的事务id，不是修改的事务id
    
**什么是二进制安全？就是客户端给什么的数据就存什么数据，不关心客户端的编解码格式**
    
## 总结

**1.统一配置管理：由节点(path)中的数据(最大为1M)实现的**

**2.分组管理：由path结构实现的**

**3.统一命名：由sequential序列实现**

**4.同步：由临时节点实现**


---


**应用场景1：分布式锁（实现：设置一个临时节点）**

进阶：==锁依托一个父节点，且具备-s ,代表这个父节点下可以有多把锁==  -> 这样就可以实现 ==队列式或事务锁==

注意：这个需要客户端代码自己实现


---

应用场景2：HA,选主

**HDFS就是由zookeeper通过锁的方式来选出主节点，zkfc就是客户端，代码就是在客户端实现的**

**抢上锁就是actived，没有抢上锁就是standby**

## 安装笔记：

    准备  node01~node04
    1,安装jdk，并设置javahome
    
    *node01:
    2，下载zookeeper    zookeeper.apache.org
    3,tar xf zookeeper.*.tar.gz
    4,mkdir /opt/mashibing
    5, mv  zookeeper  /opt/mashibing
    6,vi /etc/profile
           export ZOOKEEPER_HOME=/opt/mashibing/zookeeper-3.4.6
           export PATH=$PATH:$ZOOKEEPER_HOME/bin
    7,cd zookeeper/conf
    8,cp zoo.sample.cfg   zoo.cfg
    9,vi zoo.cfg
         dataDir=
         server.1=node01:2888:3888
    10, mkdir -p /var/mashibing/zk
    11,echo 1 >  /var/mashibing/zk/myid
    12,cd /opt  &&  scp -r ./mashibing/  node02:`pwd`
    13:node02~node04   创建 myid
    14：启动顺序  1，2，3，4
    15：zkServer.sh   start-foreground
    
    
    
    zkCli.sh
       help
       ls /
        create  /ooxx  ""
        create -s /abc/aaa
        create -e /ooxx/xxoo
        create -s -e /ooxx/xoxo
        get /ooxx
    
    
    netstat -natp   |   egrep  '(2888|3888)' 



## 验证2888和3888

    netstat -natp   |   egrep  '(2888|3888)' 

    node01
        监听了自己的3888
        其他三台通过随机端口号，连接了它的3888
        自己连接了04的2888
        
    node02
        监听了自己的3888
        03/04通过随机端口号连接了它的3888
        它通过随机端口号连接了01的3888
        自己连接了04的2888
    node03
        监听了自己的3888
        04通过随机端口号连接了它的3888
        它通过随机端口号访问了01/02的3888
    node04(leader)
        监听了自己的2888/3888
        它通过随机端口号连接了01/02/03的3888
        01/02/03连接了自己的2888
    
==这样就是四个节点，每个节点都通过3888跟其他节点有通信；两两之间有连接==

    
==端口号3888： 选主投票用的==

==端口号2888:  leader接受write请求==


![108](E711DB7A74684FB1AC59BE333E1A9351)

![104](B4DBF44812F54774AEBBF278E9757A83)

![105](F6DD6213779E4DB296CBBD920D84B9D4)

![106](FAFD9F2B71AF49FBB9704335572C7BC2)

![107](0D3A18F2DA064CAD88DE22BDD9F3D9F8)