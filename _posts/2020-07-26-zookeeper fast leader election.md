---
layout: post
title:  "zookeeper之快速选举算法"
categories: zookeeper
tags:  快速选举算法
author: zhangxiankui
---

* content
{:toc}

## 一、Zookeeper支持的选举机制
### 1.通过参数electionAlg去指定选举的策略
> 1. value值1对应的是基于UDP的无鉴权版本的快速选举
> 2. value值2对应的的是基于UDP的有鉴权版本的快速选举
> 3. value值3对应的基于TCP的快速选举，默认的选举方式
#### 1.1 注意点
> 1. 方式1和方式2在3.4.0版本中已经被deprecated了，在3.6.0版本中只有3是可用的；因此升级的时候需要注意这个配置，要么配置3要么删除这个配置使用默认值

## 二、源码解析
### 1. 选举服务启动
#### 1.1 Zookeeper启动类 -> QuorumPeerMain
核心逻辑：
> 下面只是简单的选举服务启动的逻辑，具体启动逻辑，后面再分析

```
sequenceDiagram
QuorumPeerMain->>QuorumPeerMain: main
QuorumPeerMain->>QuorumPeerMain: initializeAndRun
QuorumPeerMain->>ZooKeeperServerMain: 如果单机发布（standaloneEnabled=true）
QuorumPeerMain->>QuorumPeerMain: runFromConfig
QuorumPeerMain->>QuorumPeer: start
QuorumPeer->>QuorumPeer: startLeaderElection
QuorumPeer->>QuorumPeer: createElectionAlgorithm
QuorumPeer->>FastLeaderElection: 启动选举服务
```

##### 1.1.1 main方法

```
/**
 * To start the replicated server specify the configuration file name on
 * the command line.
 * @param args path to the configfile
 */
public static void main(String[] args) {
    QuorumPeerMain main = new QuorumPeerMain();
    try {
        // 初始化和运行
        main.initializeAndRun(args);
    } catch (Exception e) {
        LOG.error("Unexpected exception, exiting abnormally", e);
        ZKAuditProvider.addServerStartFailureAuditLog();
        ServiceUtils.requestSystemExit(ExitCode.UNEXPECTED_ERROR.getValue());
    }
    LOG.info("Exiting normally");
    ServiceUtils.requestSystemExit(ExitCode.EXECUTION_FINISHED.getValue());
}
```

##### 1.1.2 initializeAndRun

```
protected void initializeAndRun(String[] args) throws ConfigException, IOException, AdminServerException {
    QuorumPeerConfig config = new QuorumPeerConfig();
    if (args.length == 1) {
        config.parse(args[0]);
    }

    // 开启清除zk数据库快照和事务日志的定时任务
    DatadirCleanupManager purgeMgr = new DatadirCleanupManager(
        config.getDataDir(),
        config.getDataLogDir(),
        config.getSnapRetainCount(),
        config.getPurgeInterval());
    purgeMgr.start();

    if (args.length == 1 && config.isDistributed()) {
        // 集群zk的开启
        runFromConfig(config);
    } else {
        LOG.warn("Either no config or no quorum defined in config, running in standalone mode");
        // there is only server in the quorum -- run as standalone
        ZooKeeperServerMain.main(args);
    }
}
```

##### 1.1.3 runFromConfig
QuorumPeer其实是一个线程，也就是起一个zk服务来接受消息

```
public void runFromConfig(QuorumPeerConfig config) throws IOException, AdminServerException {
        ...
        quorumPeer.initialize();

        if (config.jvmPauseMonitorToRun) {
            quorumPeer.setJvmPauseMonitor(new JvmPauseMonitor(config));
        }

        // 开启服务
        quorumPeer.start();
        ZKAuditProvider.addZKStartStopAuditLog();
        quorumPeer.join();
    } catch (InterruptedException e) {
        // warn, but generally this is ok
        LOG.warn("Quorum Peer interrupted", e);
    } finally {
        if (metricsProvider != null) {
            try {
                metricsProvider.stop();
            } catch (Throwable error) {
                LOG.warn("Error while stopping metrics", error);
            }
        }
    }
}
```

#### 1.2 QuorumPeer
##### 1.2.1 start
```
public synchronized void start() {
    if (!getView().containsKey(myid)) {
        throw new RuntimeException("My id " + myid + " not in the peer list");
    }
    // 加载本地文件，恢复数据
    loadDataBase();
    // 开启服务，接收请求
    startServerCnxnFactory();
    try {
        // 命令服务器启动
        adminServer.start();
    } catch (AdminServerException e) {
        LOG.warn("Problem starting AdminServer", e);
        System.out.println(e);
    }
    // 开启选举服务
    startLeaderElection();
    startJvmPauseMonitor();
    super.start();
}
```

##### 1.2.2 startLeaderElection

```
public synchronized void startLeaderElection() {
    try {
        if (getPeerState() == ServerState.LOOKING) {
            currentVote = new Vote(myid, getLastLoggedZxid(), getCurrentEpoch());
        }
    } catch (IOException e) {
        RuntimeException re = new RuntimeException(e.getMessage());
        re.setStackTrace(e.getStackTrace());
        throw re;
    }

    // 生成选举算法
    this.electionAlg = createElectionAlgorithm(electionType);
}
```

##### 1.2.3 createElectionAlgorithm

```
protected Election createElectionAlgorithm(int electionAlgorithm) {
    Election le = null;

    // 3.6.0版本中只有3是有效的，其他会抛出异常
    switch (electionAlgorithm) {
    case 1:
        throw new UnsupportedOperationException("Election Algorithm 1 is not supported.");
    case 2:
        throw new UnsupportedOperationException("Election Algorithm 2 is not supported.");
    case 3:
        // 生成Connection的处理器
        QuorumCnxManager qcm = createCnxnManager();
        QuorumCnxManager oldQcm = qcmRef.getAndSet(qcm);
        if (oldQcm != null) {
            LOG.warn("Clobbering already-set QuorumCnxManager (restarting leader election?)");
            oldQcm.halt();
        }
        QuorumCnxManager.Listener listener = qcm.listener;
        if (listener != null) {
            // 开启选举监听器，用来接收socket
            listener.start();
            // 封装快速leader选举算法
            FastLeaderElection fle = new FastLeaderElection(this, qcm);
            // 开启leader选举线程，详细见下面的FastLeaderElection解析
            fle.start();
            le = fle;
        } else {
            LOG.error("Null listener when initializing cnx manager");
        }
        break;
    default:
        assert false;
    }
    return le;
}
```

### 2. FastLeaderElection
> 1. 类QuorumCnxManager负责Connection连接和Socket交互
> 2. FastLeaderElection负责选举的核心逻辑处理

```
sequenceDiagram
QuorumCnxManager->>QuorumCnxManager: Listener将监听Socket委托给RecvWorker处理
QuorumCnxManager->>QuorumCnxManager: RecvWorker将Message封装到recvQueue中
QuorumCnxManager->>FastLeaderElection: WorkerReceiver消费recvQueue中的Message，并放入自己的recvqueue中
FastLeaderElection->>FastLeaderElection:lookForLeader消费recvqueue消息并将发送消息放入sendqueue中
FastLeaderElection->>FastLeaderElection:WorkerSender消费sendqueue消息
FastLeaderElection->>QuorumCnxManager:WorkerSender将消息放入queueSendMap中
QuorumCnxManager->>QuorumCnxManager:SendWorker消费sendQueue,并将消息发送给对应的节点
```

#### 2.1 选票服务启动 -> start
##### 2.1.1 FastLeaderElection初始化

```
public FastLeaderElection(QuorumPeer self, QuorumCnxManager manager) {
    this.stop = false;
    this.manager = manager;
    starter(self, manager);
}
// 发送消息的阻塞队列，无界队列
LinkedBlockingQueue<ToSend> sendqueue;
// 接收消息的阻塞队列，无界队列
LinkedBlockingQueue<Notification> recvqueue;
private void starter(QuorumPeer self, QuorumCnxManager manager) {
    this.self = self;
    proposedLeader = -1;
    proposedZxid = -1;

    sendqueue = new LinkedBlockingQueue<ToSend>();
    recvqueue = new LinkedBlockingQueue<Notification>();
    this.messenger = new Messenger(manager);
}
```
##### 2.1.2 start启动

```
/**
 * This method starts the sender and receiver threads.
 */
public void start() {
    this.messenger.start();
}
```

#### 2.2 Messenger
Messager是用来封装发送和接收选票线程的对象
##### 2.2.1 Messager初始化

```
Messenger(QuorumCnxManager manager) {
    // 创建并开启发送选票线程
    this.ws = new WorkerSender(manager);
    this.wsThread = new Thread(this.ws, "WorkerSender[myid=" + self.getId() + "]");
    this.wsThread.setDaemon(true);

    // 创建并开启接收选票线程
    this.wr = new WorkerReceiver(manager);
    this.wrThread = new Thread(this.wr, "WorkerReceiver[myid=" + self.getId() + "]");
    this.wrThread.setDaemon(true);
}
```

#### 2.3 WorkSender
WorkSender是发送选票信息的对象
##### 2.3.1 ZooKeeperThread
> 1. ZooKeeperThread是zk自定义的线程，用来开启一些的服务
> 2. ZooKeeperThread中比较重要的点是它指定了异常发生时Handler，这样在某些线程发生运行时异常的时候会打印出明显的提示信息
> 3. 其中QuorumPeer、WorkSender、WorkerReceiver都是继承ZooKeeperThread的，通过它来创建线程

```
public class ZooKeeperThread extends Thread {

    private static final Logger LOG = LoggerFactory.getLogger(ZooKeeperThread.class);

    // 异常处理的Handler
    private UncaughtExceptionHandler uncaughtExceptionalHandler = new UncaughtExceptionHandler() {

        @Override
        public void uncaughtException(Thread t, Throwable e) {
            handleException(t.getName(), e);
        }
    };

    public ZooKeeperThread(String threadName) {
        super(threadName);
        setUncaughtExceptionHandler(uncaughtExceptionalHandler);
    }

    /**
     * This will be used by the uncaught exception handler and just log exception and threadname
     */
    protected void handleException(String thName, Throwable e) {
        LOG.warn("Exception occurred from thread {}", thName, e);
    }

}
```
##### 2.3.2 WorkerSender
核心逻辑：
> 1. 从发送队列sendqueue中拿到信息，并且放到manager的发送队列中，让manager去发送消息
> 2. sendqueue的信息是由WorkerReceiver线程以及lookForLeader（选择leader的）方法产生的

```
/**
 * This worker simply dequeues a message to send and
 * and queues it on the manager's queues
 */
class WorkerSender extends ZooKeeperThread {

    volatile boolean stop;
    QuorumCnxManager manager;

    WorkerSender(QuorumCnxManager manager) {
        super("WorkerSender");
        this.stop = false;
        this.manager = manager;
    }

    public void run() {
        while (!stop) {
            try {
                ToSend m = sendqueue.poll(3000, TimeUnit.MILLISECONDS);
                if (m == null) {
                    continue;
                }

                // 发送信息
                process(m);
            } catch (InterruptedException e) {
                break;
            }
        }
        LOG.info("WorkerSender is down");
    }

    /**
     * Called by run() once there is a new message to send.
     *
     * @param m     message to send
     */
    void process(ToSend m) {
        // 封装发送的选票信息
        ByteBuffer requestBuffer = buildMsg(m.state.ordinal(), m.leader, m.zxid, m.electionEpoch, m.peerEpoch, m.configData);

        manager.toSend(m.sid, requestBuffer);

    }

}
```

##### 2.3.3 QuorumCnxManager.toSend
> 1. 如果是发给自己，那么直接加入到QuorumCnxManager的接收队列recvQueue中
> 2. 如果发现不是自己，那么会加入sendQueue中，QuorumCnxManager会为每个sid创建一个sendQueue，并且该==队列是容量只有一个1的阻塞队列==，这样避免把旧的通知信息发送给其他服务器
> 3. 建立和sid对应服务器的连接，并且缓存起来
> 4. 通过QuorumCnxManager的SendWorker线程（继承自ZooKeeperThread）来发送消息


```
public void toSend(Long sid, ByteBuffer b) {
    /*
     * If sending message to myself, then simply enqueue it (loopback).
     */
    if (this.mySid == sid) {
        b.position(0);
        // 加入到当前节点的接收队列
        addToRecvQueue(new Message(b.duplicate(), sid));
        /*
         * Otherwise send to the corresponding thread to send.
         */
    } else {
        /*
         * Start a new connection if doesn't have one already.
         */
        // 为每个sid对应的节点创建一个大小为1的阻塞队列
        BlockingQueue<ByteBuffer> bq = queueSendMap.computeIfAbsent(sid, serverId -> new CircularBlockingQueue<>(SEND_CAPACITY));
        addToSendQueue(bq, b); // 加入send队列
        connectOne(sid); // 创建连接
    }
}
```


#### 2.4 WorkerReceiver
WorkerReceiver接收选票信息的对象
> 1. 获取QuorumCnxManager.recvQueue队列中的消息
> 2. 分不同版本读取投票消息（未发送当前服务器朝代为28字节和发送为40字节）
> 3. version的作用？restarting leader election？
> 4. 判断选票是否有效（observer和non-voter的选票无效），无效就通知发送节点选择自己作为leader
> 5. 如果有效，当前节点处于Looking状态，那么往队列recvqueue中加入一条数据（节点后面需要通过接收选票来计算leader），同时如果发送信息的节点也处于Looking状态且朝代比自己老，那么把当前节点的选票信息发送回该节点（发送我认为的leader信息）
> 6. 如果当前节点不处于Looking状态，但是发送节点处于Looking状态，发送当前的选票信息给发送节点

```
/**
 * Receives messages from instance of QuorumCnxManager on
 * method run(), and processes such messages.
 */
class WorkerReceiver extends ZooKeeperThread {

    volatile boolean stop;
    QuorumCnxManager manager;

    WorkerReceiver(QuorumCnxManager manager) {
        super("WorkerReceiver");
        this.stop = false;
        this.manager = manager;
    }

    public void run() {
        Message response;
        while (!stop) {
            // Sleeps on receive
            try {
                // 获取QuorumCnxManager.recvQueue队列的消息，带超时时间的等待
                response = manager.pollRecvQueue(3000, TimeUnit.MILLISECONDS);
                if (response == null) {
                    continue;
                }

                final int capacity = response.buffer.capacity();

                // The current protocol and two previous generations all send at least 28 bytes
                if (capacity < 28) {
                    LOG.error("Got a short response from server {}: {}", response.sid, capacity);
                    continue;
                }

                // this is the backwardCompatibility mode in place before ZK-107
                // It is for a version of the protocol in which we didn't send peer epoch
                // With peer epoch and version the message became 40 bytes
                boolean backCompatibility28 = (capacity == 28);

                // this is the backwardCompatibility mode for no version information
                boolean backCompatibility40 = (capacity == 40);

                response.buffer.clear();

                // Instantiate Notification and set its attributes
                Notification n = new Notification();

                int rstate = response.buffer.getInt();
                long rleader = response.buffer.getLong();
                long rzxid = response.buffer.getLong();
                long relectionEpoch = response.buffer.getLong();
                long rpeerepoch;

                int version = 0x0;
                QuorumVerifier rqv = null;

                try {
                    if (!backCompatibility28) {
                        rpeerepoch = response.buffer.getLong();
                        if (!backCompatibility40) {
                            /*
                             * Version added in 3.4.6
                             */

                            version = response.buffer.getInt();
                        } else {
                            LOG.info("Backward compatibility mode (36 bits), server id: {}", response.sid);
                        }
                    } else {
                        LOG.info("Backward compatibility mode (28 bits), server id: {}", response.sid);
                        rpeerepoch = ZxidUtils.getEpochFromZxid(rzxid);
                    }

                    // check if we have a version that includes config. If so extract config info from message.
                    if (version > 0x1) {
                        int configLength = response.buffer.getInt();

                        // we want to avoid errors caused by the allocation of a byte array with negative length
                        // (causing NegativeArraySizeException) or huge length (causing e.g. OutOfMemoryError)
                        if (configLength < 0 || configLength > capacity) {
                            throw new IOException(String.format("Invalid configLength in notification message! sid=%d, capacity=%d, version=%d, configLength=%d",
                                                                response.sid, capacity, version, configLength));
                        }

                        byte[] b = new byte[configLength];
                        response.buffer.get(b);

                        synchronized (self) {
                            try {
                                rqv = self.configFromString(new String(b));
                                QuorumVerifier curQV = self.getQuorumVerifier();
                                if (rqv.getVersion() > curQV.getVersion()) {
                                    LOG.info("{} Received version: {} my version: {}",
                                             self.getId(),
                                             Long.toHexString(rqv.getVersion()),
                                             Long.toHexString(self.getQuorumVerifier().getVersion()));
                                    if (self.getPeerState() == ServerState.LOOKING) {
                                        LOG.debug("Invoking processReconfig(), state: {}", self.getServerState());
                                        self.processReconfig(rqv, null, null, false);
                                        if (!rqv.equals(curQV)) {
                                            LOG.info("restarting leader election");
                                            self.shuttingDownLE = true;
                                            self.getElectionAlg().shutdown();

                                            break;
                                        }
                                    } else {
                                        LOG.debug("Skip processReconfig(), state: {}", self.getServerState());
                                    }
                                }
                            } catch (IOException | ConfigException e) {
                                LOG.error("Something went wrong while processing config received from {}", response.sid);
                            }
                        }
                    } else {
                        LOG.info("Backward compatibility mode (before reconfig), server id: {}", response.sid);
                    }
                } catch (BufferUnderflowException | IOException e) {
                    LOG.warn("Skipping the processing of a partial / malformed response message sent by sid={} (message length: {})",
                             response.sid, capacity, e);
                    continue;
                }
                /*
                 * If it is from a non-voting server (such as an observer or
                 * a non-voting follower), respond right away.
                 */
                if (!validVoter(response.sid)) {
                    Vote current = self.getCurrentVote();
                    QuorumVerifier qv = self.getQuorumVerifier();
                    ToSend notmsg = new ToSend(
                        ToSend.mType.notification,
                        current.getId(),
                        current.getZxid(),
                        logicalclock.get(),
                        self.getPeerState(),
                        response.sid,
                        current.getPeerEpoch(),
                        qv.toString().getBytes());

                    sendqueue.offer(notmsg);
                } else {
                    // Receive new message
                    LOG.debug("Receive new notification message. My id = {}", self.getId());

                    // State of peer that sent this message
                    QuorumPeer.ServerState ackstate = QuorumPeer.ServerState.LOOKING;
                    switch (rstate) {
                    case 0:
                        ackstate = QuorumPeer.ServerState.LOOKING;
                        break;
                    case 1:
                        ackstate = QuorumPeer.ServerState.FOLLOWING;
                        break;
                    case 2:
                        ackstate = QuorumPeer.ServerState.LEADING;
                        break;
                    case 3:
                        ackstate = QuorumPeer.ServerState.OBSERVING;
                        break;
                    default:
                        continue;
                    }

                    n.leader = rleader;
                    n.zxid = rzxid;
                    n.electionEpoch = relectionEpoch;
                    n.state = ackstate;
                    n.sid = response.sid;
                    n.peerEpoch = rpeerepoch;
                    n.version = version;
                    n.qv = rqv;
                    /*
                     * Print notification info
                     */
                    LOG.info(
                        "Notification: my state:{}; n.sid:{}, n.state:{}, n.leader:{}, n.round:0x{}, "
                            + "n.peerEpoch:0x{}, n.zxid:0x{}, message format version:0x{}, n.config version:0x{}",
                        self.getPeerState(),
                        n.sid,
                        n.state,
                        n.leader,
                        Long.toHexString(n.electionEpoch),
                        Long.toHexString(n.peerEpoch),
                        Long.toHexString(n.zxid),
                        Long.toHexString(n.version),
                        (n.qv != null ? (Long.toHexString(n.qv.getVersion())) : "0"));

                    /*
                     * If this server is looking, then send proposed leader
                     */

                    if (self.getPeerState() == QuorumPeer.ServerState.LOOKING) {
                        recvqueue.offer(n);

                        /*
                         * Send a notification back if the peer that sent this
                         * message is also looking and its logical clock is
                         * lagging behind.
                         */
                        if ((ackstate == QuorumPeer.ServerState.LOOKING)
                            && (n.electionEpoch < logicalclock.get())) {
                            Vote v = getVote();
                            QuorumVerifier qv = self.getQuorumVerifier();
                            ToSend notmsg = new ToSend(
                                ToSend.mType.notification,
                                v.getId(),
                                v.getZxid(),
                                logicalclock.get(),
                                self.getPeerState(),
                                response.sid,
                                v.getPeerEpoch(),
                                qv.toString().getBytes());
                            sendqueue.offer(notmsg);
                        }
                    } else {
                        /*
                         * If this server is not looking, but the one that sent the ack
                         * is looking, then send back what it believes to be the leader.
                         */
                        Vote current = self.getCurrentVote();
                        if (ackstate == QuorumPeer.ServerState.LOOKING) {
                            if (self.leader != null) {
                                if (leadingVoteSet != null) {
                                    self.leader.setLeadingVoteSet(leadingVoteSet);
                                    leadingVoteSet = null;
                                }
                                self.leader.reportLookingSid(response.sid);
                            }


                            LOG.debug(
                                "Sending new notification. My id ={} recipient={} zxid=0x{} leader={} config version = {}",
                                self.getId(),
                                response.sid,
                                Long.toHexString(current.getZxid()),
                                current.getId(),
                                Long.toHexString(self.getQuorumVerifier().getVersion()));

                            QuorumVerifier qv = self.getQuorumVerifier();
                            ToSend notmsg = new ToSend(
                                ToSend.mType.notification,
                                current.getId(),
                                current.getZxid(),
                                current.getElectionEpoch(),
                                self.getPeerState(),
                                response.sid,
                                current.getPeerEpoch(),
                                qv.toString().getBytes());
                            sendqueue.offer(notmsg);
                        }
                    }
                }
            } catch (InterruptedException e) {
                LOG.warn("Interrupted Exception while waiting for new message", e);
            }
        }
        LOG.info("WorkerReceiver is down");
    }

}
```
#### 2.5 选举leader

##### 2.5.1 当节点处于Looking状态时，触发节点选举
> 1. 当leader处于looking状态时会触发选举统计

```
// QuorumPeer
public void run() {
    ...
    try {
        while (running) {
            switch (getPeerState()) {
            case LOOKING:
                LOG.info("LOOKING");
                ServerMetrics.getMetrics().LOOKING_COUNT.add(1);

                if (Boolean.getBoolean("readonlymode.enabled")) {
                    ...
                    try {
                        roZkMgr.start();
                        reconfigFlagClear();
                        if (shuttingDownLE) {
                            shuttingDownLE = false;
                            startLeaderElection();
                        }
                        // 触发选举
                        setCurrentVote(makeLEStrategy().lookForLeader());
                    }
                    ...
                }
                ...
        }
    } finally {
        ...
    }
}
```

##### 2.5.2 lookForLeader
核心逻辑：
> 1. 先将选举自己作为leader，同时广播所有可以投票的节点
- 选票中包含leader和自己的id、leader的zxid、当前选举的时钟周期、当前服务器状态、leader的朝代
- 其中选举的时钟周期用来判断选票是否属于同一轮，leader的id、zxid、朝代用来做leader选举的判断
> 2. 当发现收到的选票时，会判断选票的朝代的当前服务器的朝代的大小
> 3. 如果选票的朝代小于当前服务器的朝代（逻辑时钟），那么直接忽略这个选票
> 4. 当选票的朝代和当前服务器的朝代相等时，调用totalOrderPredicate方法判断谁更合适作为leader，会调用updateProposal更新自己的选票信息，并调用sendNotificatons通知其他节点当前自己的投票信息
> 5. 当选票的朝代大于当前服务器的朝代时，更新自己逻辑时钟，调用updateProposal更新选票信息，并调用sendNotificatons通知其他节点当前自己的投票信息
> 6. 调用hasAllQuorums方法判断当前是否有某个节点获得了一半以上的选票，有则更新当前的leader节点信息

```
public Vote lookForLeader() throws InterruptedException {
    // 开始选举的时间
    self.start_fle = Time.currentElapsedTime();
    try {
        /*
         * The votes from the current leader election are stored in recvset. In other words, a vote v is in recvset
         * if v.electionEpoch == logicalclock. The current participant uses recvset to deduce on whether a majority
         * of participants has voted for it.
         */
        // 用来存储节点的投票信息，key为节点的id，value为投票信息；用来判断是否有大多数投票投给了某个节点
        Map<Long, Vote> recvset = new HashMap<Long, Vote>();

        /*
         * The votes from previous leader elections, as well as the votes from the current leader election are
         * stored in outofelection. Note that notifications in a LOOKING state are not stored in outofelection.
         * Only FOLLOWING or LEADING notifications are stored in outofelection. The current participant could use
         * outofelection to learn which participant is the leader if it arrives late (i.e., higher logicalclock than
         * the electionEpoch of the received notifications) in a leader election.
         */
        // 用来存储处于非looking状态节点的投票信息；当前节点可以用它来判断哪个是leader节点
        Map<Long, Vote> outofelection = new HashMap<Long, Vote>();

        int notTimeout = minNotificationInterval;

        /*
         * 通知其他节点，选择自己作为leader
         */
        synchronized (this) {
            // 重新发起一轮投票时，逻辑时钟加一，表示当前的选举朝代
            logicalclock.incrementAndGet();
            updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
        }
        LOG.info(
            "New election. My id = {}, proposed zxid=0x{}",
            self.getId(),
            Long.toHexString(proposedZxid));
        // 发送所有节点通知
        sendNotifications();

        SyncedLearnerTracker voteSet;

        /*
         * Loop in which we exchange notifications until we find a leader
         */
        while ((self.getPeerState() == ServerState.LOOKING) && (!stop)) {
            // 从recvqueue队列中获取其他Quorum节点发过来的投票信息
            Notification n = recvqueue.poll(notTimeout, TimeUnit.MILLISECONDS);

            /*
             * Sends more notifications if haven't received enough.
             * Otherwise processes new notification.
             */
            if (n == null) {
                if (manager.haveDelivered()) {
                    sendNotifications();
                } else {
                    manager.connectAll();
                }
            } else if (validVoter(n.sid) && validVoter(n.leader)) {
                /*
                 * Only proceed if the vote comes from a replica in the current or next
                 * voting view for a replica in the current or next voting view.
                 */
                switch (n.state) {
                case LOOKING:
                    if (getInitLastLoggedZxid() == -1) {
                        LOG.debug("Ignoring notification as our zxid is -1");
                        break;
                    }
                    if (n.zxid == -1) {
                        LOG.debug("Ignoring notification from member with -1 zxid {}", n.sid);
                        break;
                    }
                    // If notification > current, replace and send messages out
                    // 当收到的选举朝代大于当前的时钟时，设置当前时钟为选举朝代
                    if (n.electionEpoch > logicalclock.get()) {
                        logicalclock.set(n.electionEpoch);
                        recvset.clear(); // 将选票信息清空
                        // totalOrderPredicate用来判断选票节点是否比当前节点更合适作为leader，具体详见totalOrderPredicate方法
                        if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) {
                            updateProposal(n.leader, n.zxid, n.peerEpoch);
                        } else {
                            updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
                        }
                        sendNotifications();
                    } else if (n.electionEpoch < logicalclock.get()) {
                            LOG.debug(
                                "Notification election epoch is smaller than logicalclock. n.electionEpoch = 0x{}, logicalclock=0x{}",
                                Long.toHexString(n.electionEpoch),
                                Long.toHexString(logicalclock.get()));
                        
                        break;
                    } else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid, proposedEpoch)) {
                        updateProposal(n.leader, n.zxid, n.peerEpoch);
                        sendNotifications();
                    }

                    LOG.debug(
                        "Adding vote: from={}, proposed leader={}, proposed zxid=0x{}, proposed election epoch=0x{}",
                        n.sid,
                        n.leader,
                        Long.toHexString(n.zxid),
                        Long.toHexString(n.electionEpoch));

                    // don't care about the version if it's in LOOKING state
                    recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));

                    voteSet = getVoteTracker(recvset, new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch));
                    
                    // 如果超过半数，返回当前最新的投票作为leader
                    if (voteSet.hasAllQuorums()) {

                        // Verify if there is any change in the proposed leader
                        while ((n = recvqueue.poll(finalizeWait, TimeUnit.MILLISECONDS)) != null) {
                            if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid, proposedEpoch)) {
                                recvqueue.put(n);
                                break;
                            }
                        }

                        /*
                         * This predicate is true once we don't read any new
                         * relevant message from the reception queue
                         */
                        if (n == null) {
                            setPeerState(proposedLeader, voteSet);
                            Vote endVote = new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch);
                            leaveInstance(endVote);
                            return endVote;
                        }
                    }
                    break;
                case OBSERVING:
                    LOG.debug("Notification from observer: {}", n.sid);
                    break;
                case FOLLOWING:
                case LEADING:
                    /*
                     * Consider all notifications from the same epoch
                     * together.
                     */
                    if (n.electionEpoch == logicalclock.get()) {
                        recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));
                        voteSet = getVoteTracker(recvset, new Vote(n.version, n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));
                        if (voteSet.hasAllQuorums() && checkLeader(recvset, n.leader, n.electionEpoch)) {
                            setPeerState(n.leader, voteSet);
                            Vote endVote = new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch);
                            leaveInstance(endVote);
                            return endVote;
                        }
                    }

                    /*
                     * Before joining an established ensemble, verify that
                     * a majority are following the same leader.
                     *
                     * Note that the outofelection map also stores votes from the current leader election.
                     * See ZOOKEEPER-1732 for more information.
                     */
                    outofelection.put(n.sid, new Vote(n.version, n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));
                    voteSet = getVoteTracker(outofelection, new Vote(n.version, n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));

                    if (voteSet.hasAllQuorums() && checkLeader(outofelection, n.leader, n.electionEpoch)) {
                        synchronized (this) {
                            logicalclock.set(n.electionEpoch);
                            setPeerState(n.leader, voteSet);
                        }
                        Vote endVote = new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch);
                        leaveInstance(endVote);
                        return endVote;
                    }
                    break;
                default:
                    LOG.warn("Notification state unrecoginized: {} (n.state), {}(n.sid)", n.state, n.sid);
                    break;
                }
            } else {
                if (!validVoter(n.leader)) {
                    LOG.warn("Ignoring notification for non-cluster member sid {} from sid {}", n.leader, n.sid);
                }
                if (!validVoter(n.sid)) {
                    LOG.warn("Ignoring notification for sid {} from non-quorum member sid {}", n.leader, n.sid);
                }
            }
        }
        return null;
    } finally {
        try {
            if (self.jmxLeaderElectionBean != null) {
                MBeanRegistry.getInstance().unregister(self.jmxLeaderElectionBean);
            }
        } catch (Exception e) {
            LOG.warn("Failed to unregister with JMX", e);
        }
        self.jmxLeaderElectionBean = null;
        LOG.debug("Number of connection processing threads: {}", manager.getConnectionThreadCount());
    }
}
```
###### 2.5.2.1 totalOrderPredicate
> 1. 先比较选举朝代，朝代越大，选举优先级越高
> 2. 朝代相同，事务id越大优先级越高
> 3. 事务id相同，节点id越大优先级越高

```
protected boolean totalOrderPredicate(long newId, long newZxid, long newEpoch, long curId, long curZxid, long curEpoch) {
    LOG.debug(
        "id: {}, proposed id: {}, zxid: 0x{}, proposed zxid: 0x{}",
        newId,
        curId,
        Long.toHexString(newZxid),
        Long.toHexString(curZxid));

    // 默认weight为1
    if (self.getQuorumVerifier().getWeight(newId) == 0) {
        return false;
    }
    /*
     * We return true if one of the following three cases hold:
     * 1- New epoch is higher
     * 2- New epoch is the same as current epoch, but new zxid is higher
     * 3- New epoch is the same as current epoch, new zxid is the same
     *  as current zxid, but server id is higher.
     */
    return ((newEpoch > curEpoch)
            || ((newEpoch == curEpoch)
                && ((newZxid > curZxid)
                    || ((newZxid == curZxid)
                        && (newId > curId)))));
}
```
###### 2.5.2.2 sendNotifications
> 向所有能投票的节点发送选票消息

```
private void sendNotifications() {
    for (long sid : self.getCurrentAndNextConfigVoters()) {
        QuorumVerifier qv = self.getQuorumVerifier();
        ToSend notmsg = new ToSend(
            ToSend.mType.notification,
            proposedLeader,
            proposedZxid,
            logicalclock.get(),
            QuorumPeer.ServerState.LOOKING,
            sid,
            proposedEpoch,
            qv.toString().getBytes());

        LOG.debug(
            "Sending Notification: {} (n.leader), 0x{} (n.zxid), 0x{} (n.round), {} (recipient),"
                + " {} (myid), 0x{} (n.peerEpoch) ",
            proposedLeader,
            Long.toHexString(proposedZxid),
            Long.toHexString(logicalclock.get()),
            sid,
            self.getId(),
            Long.toHexString(proposedEpoch));

        sendqueue.offer(notmsg);
    }
}
```
###### 2.5.2.3 updateProposal
> 更新选票的leader信息

```
synchronized void updateProposal(long leader, long zxid, long epoch) {
    proposedLeader = leader;
    proposedZxid = zxid;
    proposedEpoch = epoch;
}
```
###### 2.5.2.4 SyncedLearnerTracker
这个类是用来统计选票信息

```
public class SyncedLearnerTracker {

    protected ArrayList<QuorumVerifierAcksetPair> qvAcksetPairs = new ArrayList<QuorumVerifierAcksetPair>();

    // 加入QuorumVerifier，一般只会有QuorumMaj这个类
    public void addQuorumVerifier(QuorumVerifier qv) {
        qvAcksetPairs.add(new QuorumVerifierAcksetPair(qv, new HashSet<Long>(qv.getVotingMembers().size())));
    }

    // 加入选票信息到qvAckset中，后面用来判断是否有超过一半的节点投票给sid
    public boolean addAck(Long sid) {
        boolean change = false;
        for (QuorumVerifierAcksetPair qvAckset : qvAcksetPairs) {
            if (qvAckset.getQuorumVerifier().getVotingMembers().containsKey(sid)) {
                qvAckset.getAckset().add(sid);
                change = true;
            }
        }
        return change;
    }

    public boolean hasSid(long sid) {
        for (QuorumVerifierAcksetPair qvAckset : qvAcksetPairs) {
            if (!qvAckset.getQuorumVerifier().getVotingMembers().containsKey(sid)) {
                return false;
            }
        }
        return true;
    }

    // 用来判断sid是否有超过一半的选票，具体看下面的containsQuorum方法
    public boolean hasAllQuorums() {
        for (QuorumVerifierAcksetPair qvAckset : qvAcksetPairs) {
            if (!qvAckset.getQuorumVerifier().containsQuorum(qvAckset.getAckset())) {
                return false;
            }
        }
        return true;
    }

    public String ackSetsToString() {
        StringBuilder sb = new StringBuilder();

        for (QuorumVerifierAcksetPair qvAckset : qvAcksetPairs) {
            sb.append(qvAckset.getAckset().toString()).append(",");
        }

        return sb.substring(0, sb.length() - 1);
    }

    public static class QuorumVerifierAcksetPair {

        private final QuorumVerifier qv;
        private final HashSet<Long> ackset;

        public QuorumVerifierAcksetPair(QuorumVerifier qv, HashSet<Long> ackset) {
            this.qv = qv;
            this.ackset = ackset;
        }

        public QuorumVerifier getQuorumVerifier() {
            return this.qv;
        }

        public HashSet<Long> getAckset() {
            return this.ackset;
        }

    }

}
```
2.5.2.5 QuorumMaj.containsQuorum

```
public class QuorumMaj implements QuorumVerifier {

    // 所有节点
    private Map<Long, QuorumServer> allMembers = new HashMap<Long, QuorumServer>();
    // 可以投票的节点
    private Map<Long, QuorumServer> votingMembers = new HashMap<Long, QuorumServer>();
    // observer的节点
    private Map<Long, QuorumServer> observingMembers = new HashMap<Long, QuorumServer>();
    private long version = 0;
    private int half; // 等于votingMembers.size()/2

    public int hashCode() {
        assert false : "hashCode not designed";
        return 42; // any arbitrary constant will do
    }

    public boolean equals(Object o) {
        if (!(o instanceof QuorumMaj)) {
            return false;
        }
        QuorumMaj qm = (QuorumMaj) o;
        if (qm.getVersion() == version) {
            return true;
        }
        if (allMembers.size() != qm.getAllMembers().size()) {
            return false;
        }
        for (QuorumServer qs : allMembers.values()) {
            QuorumServer qso = qm.getAllMembers().get(qs.id);
            if (qso == null || !qs.equals(qso)) {
                return false;
            }
        }
        return true;
    }

    /**
     * Defines a majority to avoid computing it every time.
     *
     */
    public QuorumMaj(Map<Long, QuorumServer> allMembers) {
        this.allMembers = allMembers;
        for (QuorumServer qs : allMembers.values()) {
            if (qs.type == LearnerType.PARTICIPANT) {
                votingMembers.put(Long.valueOf(qs.id), qs);
            } else {
                observingMembers.put(Long.valueOf(qs.id), qs);
            }
        }
        half = votingMembers.size() / 2;
    }

    public QuorumMaj(Properties props) throws ConfigException {
        for (Entry<Object, Object> entry : props.entrySet()) {
            String key = entry.getKey().toString();
            String value = entry.getValue().toString();

            if (key.startsWith("server.")) {
                int dot = key.indexOf('.');
                long sid = Long.parseLong(key.substring(dot + 1));
                QuorumServer qs = new QuorumServer(sid, value);
                allMembers.put(Long.valueOf(sid), qs);
                if (qs.type == LearnerType.PARTICIPANT) {
                    votingMembers.put(Long.valueOf(sid), qs);
                } else {
                    observingMembers.put(Long.valueOf(sid), qs);
                }
            } else if (key.equals("version")) {
                version = Long.parseLong(value, 16);
            }
        }
        half = votingMembers.size() / 2;
    }

    /**
     * Returns weight of 1 by default.
     *
     * @param id
     */
    public long getWeight(long id) {
        return 1;
    }

    /**
     * Verifies if a set is a majority. Assumes that ackSet contains acks only
     * from votingMembers
     */
    public boolean containsQuorum(Set<Long> ackSet) {
        // 判断收到选票的size是否比half大
        return (ackSet.size() > half);
    }
}
```

