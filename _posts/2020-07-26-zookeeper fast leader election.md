---
layout: post
title:  "zookeeper֮����ѡ���㷨"
categories: zookeeper
tags:  ����ѡ���㷨
author: zhangxiankui
---

* content
{:toc}

## һ��Zookeeper֧�ֵ�ѡ�ٻ���
### 1.ͨ������electionAlgȥָ��ѡ�ٵĲ���
> 1. valueֵ1��Ӧ���ǻ���UDP���޼�Ȩ�汾�Ŀ���ѡ��
> 2. valueֵ2��Ӧ�ĵ��ǻ���UDP���м�Ȩ�汾�Ŀ���ѡ��
> 3. valueֵ3��Ӧ�Ļ���TCP�Ŀ���ѡ�٣�Ĭ�ϵ�ѡ�ٷ�ʽ
#### 1.1 ע���
> 1. ��ʽ1�ͷ�ʽ2��3.4.0�汾���Ѿ���deprecated�ˣ���3.6.0�汾��ֻ��3�ǿ��õģ����������ʱ����Ҫע��������ã�Ҫô����3Ҫôɾ���������ʹ��Ĭ��ֵ

## ����Դ�����
### 1. ѡ�ٷ�������
#### 1.1 Zookeeper������ -> QuorumPeerMain
�����߼���
> ����ֻ�Ǽ򵥵�ѡ�ٷ����������߼������������߼��������ٷ���

```
sequenceDiagram
QuorumPeerMain->>QuorumPeerMain: main
QuorumPeerMain->>QuorumPeerMain: initializeAndRun
QuorumPeerMain->>ZooKeeperServerMain: �������������standaloneEnabled=true��
QuorumPeerMain->>QuorumPeerMain: runFromConfig
QuorumPeerMain->>QuorumPeer: start
QuorumPeer->>QuorumPeer: startLeaderElection
QuorumPeer->>QuorumPeer: createElectionAlgorithm
QuorumPeer->>FastLeaderElection: ����ѡ�ٷ���
```

##### 1.1.1 main����

```
/**
 * To start the replicated server specify the configuration file name on
 * the command line.
 * @param args path to the configfile
 */
public static void main(String[] args) {
    QuorumPeerMain main = new QuorumPeerMain();
    try {
        // ��ʼ��������
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

    // �������zk���ݿ���պ�������־�Ķ�ʱ����
    DatadirCleanupManager purgeMgr = new DatadirCleanupManager(
        config.getDataDir(),
        config.getDataLogDir(),
        config.getSnapRetainCount(),
        config.getPurgeInterval());
    purgeMgr.start();

    if (args.length == 1 && config.isDistributed()) {
        // ��Ⱥzk�Ŀ���
        runFromConfig(config);
    } else {
        LOG.warn("Either no config or no quorum defined in config, running in standalone mode");
        // there is only server in the quorum -- run as standalone
        ZooKeeperServerMain.main(args);
    }
}
```

##### 1.1.3 runFromConfig
QuorumPeer��ʵ��һ���̣߳�Ҳ������һ��zk������������Ϣ

```
public void runFromConfig(QuorumPeerConfig config) throws IOException, AdminServerException {
        ...
        quorumPeer.initialize();

        if (config.jvmPauseMonitorToRun) {
            quorumPeer.setJvmPauseMonitor(new JvmPauseMonitor(config));
        }

        // ��������
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
    // ���ر����ļ����ָ�����
    loadDataBase();
    // �������񣬽�������
    startServerCnxnFactory();
    try {
        // �������������
        adminServer.start();
    } catch (AdminServerException e) {
        LOG.warn("Problem starting AdminServer", e);
        System.out.println(e);
    }
    // ����ѡ�ٷ���
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

    // ����ѡ���㷨
    this.electionAlg = createElectionAlgorithm(electionType);
}
```

##### 1.2.3 createElectionAlgorithm

```
protected Election createElectionAlgorithm(int electionAlgorithm) {
    Election le = null;

    // 3.6.0�汾��ֻ��3����Ч�ģ��������׳��쳣
    switch (electionAlgorithm) {
    case 1:
        throw new UnsupportedOperationException("Election Algorithm 1 is not supported.");
    case 2:
        throw new UnsupportedOperationException("Election Algorithm 2 is not supported.");
    case 3:
        // ����Connection�Ĵ�����
        QuorumCnxManager qcm = createCnxnManager();
        QuorumCnxManager oldQcm = qcmRef.getAndSet(qcm);
        if (oldQcm != null) {
            LOG.warn("Clobbering already-set QuorumCnxManager (restarting leader election?)");
            oldQcm.halt();
        }
        QuorumCnxManager.Listener listener = qcm.listener;
        if (listener != null) {
            // ����ѡ�ټ���������������socket
            listener.start();
            // ��װ����leaderѡ���㷨
            FastLeaderElection fle = new FastLeaderElection(this, qcm);
            // ����leaderѡ���̣߳���ϸ�������FastLeaderElection����
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
> 1. ��QuorumCnxManager����Connection���Ӻ�Socket����
> 2. FastLeaderElection����ѡ�ٵĺ����߼�����

```
sequenceDiagram
QuorumCnxManager->>QuorumCnxManager: Listener������Socketί�и�RecvWorker����
QuorumCnxManager->>QuorumCnxManager: RecvWorker��Message��װ��recvQueue��
QuorumCnxManager->>FastLeaderElection: WorkerReceiver����recvQueue�е�Message���������Լ���recvqueue��
FastLeaderElection->>FastLeaderElection:lookForLeader����recvqueue��Ϣ����������Ϣ����sendqueue��
FastLeaderElection->>FastLeaderElection:WorkerSender����sendqueue��Ϣ
FastLeaderElection->>QuorumCnxManager:WorkerSender����Ϣ����queueSendMap��
QuorumCnxManager->>QuorumCnxManager:SendWorker����sendQueue,������Ϣ���͸���Ӧ�Ľڵ�
```

#### 2.1 ѡƱ�������� -> start
##### 2.1.1 FastLeaderElection��ʼ��

```
public FastLeaderElection(QuorumPeer self, QuorumCnxManager manager) {
    this.stop = false;
    this.manager = manager;
    starter(self, manager);
}
// ������Ϣ���������У��޽����
LinkedBlockingQueue<ToSend> sendqueue;
// ������Ϣ���������У��޽����
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
##### 2.1.2 start����

```
/**
 * This method starts the sender and receiver threads.
 */
public void start() {
    this.messenger.start();
}
```

#### 2.2 Messenger
Messager��������װ���ͺͽ���ѡƱ�̵߳Ķ���
##### 2.2.1 Messager��ʼ��

```
Messenger(QuorumCnxManager manager) {
    // ��������������ѡƱ�߳�
    this.ws = new WorkerSender(manager);
    this.wsThread = new Thread(this.ws, "WorkerSender[myid=" + self.getId() + "]");
    this.wsThread.setDaemon(true);

    // ��������������ѡƱ�߳�
    this.wr = new WorkerReceiver(manager);
    this.wrThread = new Thread(this.wr, "WorkerReceiver[myid=" + self.getId() + "]");
    this.wrThread.setDaemon(true);
}
```

#### 2.3 WorkSender
WorkSender�Ƿ���ѡƱ��Ϣ�Ķ���
##### 2.3.1 ZooKeeperThread
> 1. ZooKeeperThread��zk�Զ�����̣߳���������һЩ�ķ���
> 2. ZooKeeperThread�бȽ���Ҫ�ĵ�����ָ�����쳣����ʱHandler��������ĳЩ�̷߳�������ʱ�쳣��ʱ����ӡ�����Ե���ʾ��Ϣ
> 3. ����QuorumPeer��WorkSender��WorkerReceiver���Ǽ̳�ZooKeeperThread�ģ�ͨ�����������߳�

```
public class ZooKeeperThread extends Thread {

    private static final Logger LOG = LoggerFactory.getLogger(ZooKeeperThread.class);

    // �쳣�����Handler
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
�����߼���
> 1. �ӷ��Ͷ���sendqueue���õ���Ϣ�����ҷŵ�manager�ķ��Ͷ����У���managerȥ������Ϣ
> 2. sendqueue����Ϣ����WorkerReceiver�߳��Լ�lookForLeader��ѡ��leader�ģ�����������

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

                // ������Ϣ
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
        // ��װ���͵�ѡƱ��Ϣ
        ByteBuffer requestBuffer = buildMsg(m.state.ordinal(), m.leader, m.zxid, m.electionEpoch, m.peerEpoch, m.configData);

        manager.toSend(m.sid, requestBuffer);

    }

}
```

##### 2.3.3 QuorumCnxManager.toSend
> 1. ����Ƿ����Լ�����ôֱ�Ӽ��뵽QuorumCnxManager�Ľ��ն���recvQueue��
> 2. ������ֲ����Լ�����ô�����sendQueue�У�QuorumCnxManager��Ϊÿ��sid����һ��sendQueue�����Ҹ�==����������ֻ��һ��1����������==����������Ѿɵ�֪ͨ��Ϣ���͸�����������
> 3. ������sid��Ӧ�����������ӣ����һ�������
> 4. ͨ��QuorumCnxManager��SendWorker�̣߳��̳���ZooKeeperThread����������Ϣ


```
public void toSend(Long sid, ByteBuffer b) {
    /*
     * If sending message to myself, then simply enqueue it (loopback).
     */
    if (this.mySid == sid) {
        b.position(0);
        // ���뵽��ǰ�ڵ�Ľ��ն���
        addToRecvQueue(new Message(b.duplicate(), sid));
        /*
         * Otherwise send to the corresponding thread to send.
         */
    } else {
        /*
         * Start a new connection if doesn't have one already.
         */
        // Ϊÿ��sid��Ӧ�Ľڵ㴴��һ����СΪ1����������
        BlockingQueue<ByteBuffer> bq = queueSendMap.computeIfAbsent(sid, serverId -> new CircularBlockingQueue<>(SEND_CAPACITY));
        addToSendQueue(bq, b); // ����send����
        connectOne(sid); // ��������
    }
}
```


#### 2.4 WorkerReceiver
WorkerReceiver����ѡƱ��Ϣ�Ķ���
> 1. ��ȡQuorumCnxManager.recvQueue�����е���Ϣ
> 2. �ֲ�ͬ�汾��ȡͶƱ��Ϣ��δ���͵�ǰ����������Ϊ28�ֽںͷ���Ϊ40�ֽڣ�
> 3. version�����ã�restarting leader election��
> 4. �ж�ѡƱ�Ƿ���Ч��observer��non-voter��ѡƱ��Ч������Ч��֪ͨ���ͽڵ�ѡ���Լ���Ϊleader
> 5. �����Ч����ǰ�ڵ㴦��Looking״̬����ô������recvqueue�м���һ�����ݣ��ڵ������Ҫͨ������ѡƱ������leader����ͬʱ���������Ϣ�Ľڵ�Ҳ����Looking״̬�ҳ������Լ��ϣ���ô�ѵ�ǰ�ڵ��ѡƱ��Ϣ���ͻظýڵ㣨��������Ϊ��leader��Ϣ��
> 6. �����ǰ�ڵ㲻����Looking״̬�����Ƿ��ͽڵ㴦��Looking״̬�����͵�ǰ��ѡƱ��Ϣ�����ͽڵ�

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
                // ��ȡQuorumCnxManager.recvQueue���е���Ϣ������ʱʱ��ĵȴ�
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
#### 2.5 ѡ��leader

##### 2.5.1 ���ڵ㴦��Looking״̬ʱ�������ڵ�ѡ��
> 1. ��leader����looking״̬ʱ�ᴥ��ѡ��ͳ��

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
                        // ����ѡ��
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
�����߼���
> 1. �Ƚ�ѡ���Լ���Ϊleader��ͬʱ�㲥���п���ͶƱ�Ľڵ�
- ѡƱ�а���leader���Լ���id��leader��zxid����ǰѡ�ٵ�ʱ�����ڡ���ǰ������״̬��leader�ĳ���
- ����ѡ�ٵ�ʱ�����������ж�ѡƱ�Ƿ�����ͬһ�֣�leader��id��zxid������������leaderѡ�ٵ��ж�
> 2. �������յ���ѡƱʱ�����ж�ѡƱ�ĳ����ĵ�ǰ�������ĳ����Ĵ�С
> 3. ���ѡƱ�ĳ���С�ڵ�ǰ�������ĳ������߼�ʱ�ӣ�����ôֱ�Ӻ������ѡƱ
> 4. ��ѡƱ�ĳ����͵�ǰ�������ĳ������ʱ������totalOrderPredicate�����ж�˭��������Ϊleader�������updateProposal�����Լ���ѡƱ��Ϣ��������sendNotificatons֪ͨ�����ڵ㵱ǰ�Լ���ͶƱ��Ϣ
> 5. ��ѡƱ�ĳ������ڵ�ǰ�������ĳ���ʱ�������Լ��߼�ʱ�ӣ�����updateProposal����ѡƱ��Ϣ��������sendNotificatons֪ͨ�����ڵ㵱ǰ�Լ���ͶƱ��Ϣ
> 6. ����hasAllQuorums�����жϵ�ǰ�Ƿ���ĳ���ڵ�����һ�����ϵ�ѡƱ��������µ�ǰ��leader�ڵ���Ϣ

```
public Vote lookForLeader() throws InterruptedException {
    // ��ʼѡ�ٵ�ʱ��
    self.start_fle = Time.currentElapsedTime();
    try {
        /*
         * The votes from the current leader election are stored in recvset. In other words, a vote v is in recvset
         * if v.electionEpoch == logicalclock. The current participant uses recvset to deduce on whether a majority
         * of participants has voted for it.
         */
        // �����洢�ڵ��ͶƱ��Ϣ��keyΪ�ڵ��id��valueΪͶƱ��Ϣ�������ж��Ƿ��д����ͶƱͶ����ĳ���ڵ�
        Map<Long, Vote> recvset = new HashMap<Long, Vote>();

        /*
         * The votes from previous leader elections, as well as the votes from the current leader election are
         * stored in outofelection. Note that notifications in a LOOKING state are not stored in outofelection.
         * Only FOLLOWING or LEADING notifications are stored in outofelection. The current participant could use
         * outofelection to learn which participant is the leader if it arrives late (i.e., higher logicalclock than
         * the electionEpoch of the received notifications) in a leader election.
         */
        // �����洢���ڷ�looking״̬�ڵ��ͶƱ��Ϣ����ǰ�ڵ�����������ж��ĸ���leader�ڵ�
        Map<Long, Vote> outofelection = new HashMap<Long, Vote>();

        int notTimeout = minNotificationInterval;

        /*
         * ֪ͨ�����ڵ㣬ѡ���Լ���Ϊleader
         */
        synchronized (this) {
            // ���·���һ��ͶƱʱ���߼�ʱ�Ӽ�һ����ʾ��ǰ��ѡ�ٳ���
            logicalclock.incrementAndGet();
            updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
        }
        LOG.info(
            "New election. My id = {}, proposed zxid=0x{}",
            self.getId(),
            Long.toHexString(proposedZxid));
        // �������нڵ�֪ͨ
        sendNotifications();

        SyncedLearnerTracker voteSet;

        /*
         * Loop in which we exchange notifications until we find a leader
         */
        while ((self.getPeerState() == ServerState.LOOKING) && (!stop)) {
            // ��recvqueue�����л�ȡ����Quorum�ڵ㷢������ͶƱ��Ϣ
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
                    // ���յ���ѡ�ٳ������ڵ�ǰ��ʱ��ʱ�����õ�ǰʱ��Ϊѡ�ٳ���
                    if (n.electionEpoch > logicalclock.get()) {
                        logicalclock.set(n.electionEpoch);
                        recvset.clear(); // ��ѡƱ��Ϣ���
                        // totalOrderPredicate�����ж�ѡƱ�ڵ��Ƿ�ȵ�ǰ�ڵ��������Ϊleader���������totalOrderPredicate����
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
                    
                    // ����������������ص�ǰ���µ�ͶƱ��Ϊleader
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
> 1. �ȱȽ�ѡ�ٳ���������Խ��ѡ�����ȼ�Խ��
> 2. ������ͬ������idԽ�����ȼ�Խ��
> 3. ����id��ͬ���ڵ�idԽ�����ȼ�Խ��

```
protected boolean totalOrderPredicate(long newId, long newZxid, long newEpoch, long curId, long curZxid, long curEpoch) {
    LOG.debug(
        "id: {}, proposed id: {}, zxid: 0x{}, proposed zxid: 0x{}",
        newId,
        curId,
        Long.toHexString(newZxid),
        Long.toHexString(curZxid));

    // Ĭ��weightΪ1
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
> ��������ͶƱ�Ľڵ㷢��ѡƱ��Ϣ

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
> ����ѡƱ��leader��Ϣ

```
synchronized void updateProposal(long leader, long zxid, long epoch) {
    proposedLeader = leader;
    proposedZxid = zxid;
    proposedEpoch = epoch;
}
```
###### 2.5.2.4 SyncedLearnerTracker
�����������ͳ��ѡƱ��Ϣ

```
public class SyncedLearnerTracker {

    protected ArrayList<QuorumVerifierAcksetPair> qvAcksetPairs = new ArrayList<QuorumVerifierAcksetPair>();

    // ����QuorumVerifier��һ��ֻ����QuorumMaj�����
    public void addQuorumVerifier(QuorumVerifier qv) {
        qvAcksetPairs.add(new QuorumVerifierAcksetPair(qv, new HashSet<Long>(qv.getVotingMembers().size())));
    }

    // ����ѡƱ��Ϣ��qvAckset�У����������ж��Ƿ��г���һ��Ľڵ�ͶƱ��sid
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

    // �����ж�sid�Ƿ��г���һ���ѡƱ�����忴�����containsQuorum����
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

    // ���нڵ�
    private Map<Long, QuorumServer> allMembers = new HashMap<Long, QuorumServer>();
    // ����ͶƱ�Ľڵ�
    private Map<Long, QuorumServer> votingMembers = new HashMap<Long, QuorumServer>();
    // observer�Ľڵ�
    private Map<Long, QuorumServer> observingMembers = new HashMap<Long, QuorumServer>();
    private long version = 0;
    private int half; // ����votingMembers.size()/2

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
        // �ж��յ�ѡƱ��size�Ƿ��half��
        return (ackSet.size() > half);
    }
}
```

