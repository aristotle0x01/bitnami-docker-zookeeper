## 源码

基于**3.7.0**

### 类的关系

![](https://user-images.githubusercontent.com/2216435/119318586-194ff780-bcac-11eb-9c90-28aef0b1d731.png)



### 执行逻辑

**QuorumPeer**类从逻辑上完备代表了zookeeper集群的一个节点，可以完整实现zab协议，以及对应节点角色的消息处理流程。执行逻辑的设计上充分利用了线程，BlockingQueue

1）不同线程处理zab不同阶段的重要逻辑

2)  通过blockingqueue作为消息总线，解耦并保证消息有序性

![](https://user-images.githubusercontent.com/2216435/119611332-20524380-be2d-11eb-84cc-82c0f243a4c7.png)

## 集群如何启动？

根据zookeeper官网dockerfile分析[zookeeper-docker-3.7.0](https://github.com/31z4/zookeeper-docker/blob/088d6cc2d70c0d6898c4d0afd953d85052dd2aa2/3.7.0/Dockerfile)：

`CMD ["zkServer.sh", "start-foreground"]`

那么开始从代码zookeeper-3.7.0源码查看**zkServer.sh**

`org.apache.zookeeper.server.quorum.QuorumPeerMain`

来看看**QuorumPeerMain** 做了什么？

```
main(String[] args)
	QuorumPeerMain.initializeAndRun(args)
		runFromConfig(config) // cluster mode
			QuorumPeer.start()
				loadDataBase()	// 从磁盘初始化zk内存数据
				startLeaderElection() // 开始选主...
				thread.start()
			QuorumPeer.join()
		ZooKeeperServerMain.main(args) // standalone mode
			// 无选主等过程，直接启动
		
```



## 选主逻辑？

```
// 启动线程，准备好选主端口tcp所需的接收和发送线程
QuorumPeer.start()
	startLeaderElection()
		createElectionAlgorithm
			fle = FastLeaderElection(this, qcm)
			fle.start()
				messenger.start()
					wsThread.start() // 选主消息发送线程
          wrThread.start() // 选主消息接收线程 *recvqueue
	thread.start()
	
// 节点循环，在looking, following, leading状态间转换
// 完成在各状态下的功能
QuorumPeer.run()
	while (running)
		case looking:
			setCurrentVote(makeLEStrategy().lookForLeader()) // 完成快速选主 *recvqueue
				sendNotifications()
		case following:
			follower.followLeader()
		case leading:
			leader.lead()

```

## 选主后如何同步数据？

![](https://user-images.githubusercontent.com/2216435/119791304-49470700-bf07-11eb-8c1a-b1687af1e76b.png)

## 怎么写磁盘？

变更皆由leader发起，由quorum协同完成，属于系统节点间通信

其核心类是**SyncRequestProcessor**，通过**zks.getZKDatabase().commit()**完成内存数据存盘。

对于从节点，SyncRequestProcessor.nextprocessor=SendAckRequestProcessor (FollowerZooKeeperServer配置)，通过flush完成对leader的ack确认。

对于主节点SyncRequestProcessor.nextprocessor=AckRequestProcessor (LeaderZooKeeperServer配置)，通过**leader.processAck**对自身确认。

```
/**
 * SyncRequestProcessor
 * 
 * This RequestProcessor logs requests to disk. It batches the requests to do
 * the io efficiently. The request is not passed to the next RequestProcessor
 * until its log has been synced to disk.
 *
 * SyncRequestProcessor is used in 3 different cases
 * 1. Leader - Sync request to disk and forward it to AckRequestProcessor which
 *             send ack back to itself.
 * 2. Follower - Sync request to disk and forward request to
 *             SendAckRequestProcessor which send the packets to leader.
 *             SendAckRequestProcessor is flushable which allow us to force
 *             push packets to leader.
 * 3. Observer - Sync committed request to disk (received as INFORM packet).
 *             It never send ack back to the leader, so the nextProcessor will
 *             be null. This change the semantic of txnlog on the observer
 *             since it only contains committed txns.
 */
```

[^刷盘]: zookeeper.forceSync默认为"yes"

## 通信是如何管理的？

### 选主通信管理

```
QuorumPeer
	startLeaderElection()
		createElectionAlgorithm()
			QuorumCnxManager qcm = createCnxnManager()

/**
 * QuorumCnxManager
 * 
 * This class implements a connection manager for leader election using TCP. It
 * maintains one connection for every pair of servers. The tricky part is to
 * guarantee that there is exactly one connection for every pair of servers that
 * are operating correctly and that can communicate over the network.
 */
```

### peer-peer通信管理

```
# follower
followLeader()
	leaderServer = findLeader()
	connectToLeader(leaderServer.addr, leaderServer.hostname)
		learner.connectToLeader()
			new LeaderConnector(address, socket, latch)
				sock.connect()
	while(true){
	 ...
	}
```

```
# leader
lead()
	// Start thread that waits for connection requests from
  // new followers.
  cnxAcceptor = new LearnerCnxAcceptor()
  cnxAcceptor.start()
  cnxAcceptor.run()
    // 所有follower
  	serverSockets.forEach(serverSocket ->
                        executor.submit(new LearnerCnxAcceptorHandler(serverSocket, latch)))
    LearnerCnxAcceptorHandler.run()
    	acceptConnections()
    		LearnerHandler.run()
    			...
    			while(true){
    			 ...
    			}
```

### client-server通信管理

会话，sock连接，节点主要类之间的关系：

![](https://user-images.githubusercontent.com/2216435/119586542-81fbb900-bdff-11eb-8c57-545a93fbefc5.jpg)

```
main(String[] args)
	QuorumPeerMain.initializeAndRun(args)
		runFromConfig(config) // cluster mode
		  cnxnFactory = ServerCnxnFactory.createFactory()
		  	NIOServerCnxnFactory or NettyServerCnxnFactory
		  cnxnFactory.startup(zkServer)
		  	NIOServerCnxnFactory.startup()
		  		start()
		  		  # 线程池处理请求
		  			workerPool = new WorkerService
		  			# selector 机制io
		  			selectorThreads.start()
		  				run()
		  					handleIO(key)
		  						workerPool.schedule(workRequest)
		  							(ExecutorService)worker.execute(scheduledWorkRequest)
		  								IOWorkRequest.doWork()
		  									(NIOServerCnxn)cnxn.doIO(key)
		  										readPayload()
		  										  # session管理
		  											readConnectRequest()
		  												zkServer.processConnectRequest(this, incomingBuffer)
		  													createSession(cnxn, passwd, sessionTimeout)
		  														sessionId = sessionTracker.createSession(timeout)
		  														cnxn.setSessionId(sessionId)
		  											# 请求处理
		  											readRequest()
		  												zkServer.processPacket(this, incomingBuffer)
		  													submitRequest(si)
		  														enqueueRequest(si)
		  															RequestThrottler.submitRequest()
		  															RequestThrottler.run()
		  																zks.submitRequestNow(request)
		  																	firstProcessor.processRequest(si)
		  										handleWrite(k)

		  			acceptThread.start()
		  			expirerThread.start()
		  		ZooKeeperServer.startup()
		  			startupWithServerState(State state)
		  				startSessionTracker()
		  				setupRequestProcessors()
		  quorumPeer.setCnxnFactory(cnxnFactory)
		  ...
			QuorumPeer.start()
			...
			QuorumPeer.join()
```

## session是如何管理和自动切换节点的？

## 如何snapshot?

逻辑详见SyncRequestProcessor

关键在于明确snapshot和txn 日志在恢复系统的意义

## 解析txn和snapshot日志

本来想tcpdump后自己解析，但是zookeeper已经提供了相关工具

```
// 具体数据格式定义参考
FileTxnLog
zookeeper.jute
```

transaction解析命令行：

```
./zkTxnLogToolkit.sh --dump /zoo1/log/version-2/log.200000001
```

![](https://user-images.githubusercontent.com/2216435/119638417-353cd000-be49-11eb-8781-9b1bcea41a96.png)

snapshot解析命令行：

```
./zkSnapShotToolkit.sh -d /zoo1/version-2/snapshot.800000008
```

![](https://user-images.githubusercontent.com/2216435/119638154-f7d84280-be48-11eb-9047-73b0fd5404ed.png)

![](https://user-images.githubusercontent.com/2216435/119638236-0b83a900-be49-11eb-97c9-20c4d0b087f3.png)

## 数据变更请求？

## 内部功能实现

watch机制

临时节点，序列节点实现

锁的实现

内存数据结构

## 延伸技术点

- blockingqueue的实现

  

  ```
      /**
       * Removes a node from head of queue.
       */
      private E dequeue() {
          Node<E> h = head;
          Node<E> first = h.next;
          h.next = h; // help GC
          head = first;
          E x = first.item;
          // head永远指向第一个元素，被删除的元素变为head，且其元素为null
          first.item = null;
          return x;
      }
      
      public LinkedBlockingQueue(int capacity) {
          if (capacity <= 0) throw new IllegalArgumentException();
          this.capacity = capacity;
          last = head = new Node<E>(null);
      }
  ```

  关于两个条件变量和while 检查的详细解释

  [Operating Systems: Three Easy Pieces-->Condition Variables](https://pages.cs.wisc.edu/~remzi/OSTEP/threads-cv.pdf) 

  ```
       private final Condition notEmpty = takeLock.newCondition();
       private final Condition notFull = putLock.newCondition();
       
       try {
              /*
               * Note that count is used in wait guard even though it is
               * not protected by lock. This works because count can
               * only decrease at this point (all other puts are shut
               * out by lock), and we (or some other waiting put) are
               * signalled if it ever changes from capacity. Similarly
               * for all other uses of count in other wait guards.
               */
              while (count.get() == capacity) {
                  notFull.await();
              }
              enqueue(node);
              c = count.getAndIncrement();
              if (c + 1 < capacity)
                  notFull.signal();
          } finally {
              putLock.unlock();
          }
  ```

- join / interupt, java 线程与操作系统线程的关系

- java socket / nio / selector / netty

- ```
  this.logStream = null 会发生什么？
  ```

  

