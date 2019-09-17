---
layout: post
title: Hbase队列
date: 2019-06-15
tags: Hbase
---

HRegionServer

```
public HRegionServer(Configuration conf, CoordinatedStateManager csm)
      throws IOException, InterruptedException {
      ...
      //初始化RpcService
      rpcServices = createRpcServices();
      ...
      rpcServices.start();
      ...
}
```

RSRpcServices

```
RSRpcServices(HRegionServer rs, LogDelegate ld) throws IOException {
    ...
    RpcSchedulerFactory rpcSchedulerFactory;
    try {
      Class<?> rpcSchedulerFactoryClass = rs.conf.getClass(
          REGION_SERVER_RPC_SCHEDULER_FACTORY_CLASS,
          SimpleRpcSchedulerFactory.class);
      rpcSchedulerFactory = (RpcSchedulerFactory)
          rpcSchedulerFactoryClass.getDeclaredConstructor().newInstance();
    } catch (Exception e) {
      throw new IllegalArgumentException(e);
    }
    ...
    //通过注解方式加载RegionServer的
    //openRegion,closeRegion,compactRegion,flushRegion,getOnlineRegion,getRegionInfo,
    //getServerInfo,getStoreFile,mergeRegions,replay,replicateWALEntry,splitRegion,stopServer
    //方法
    priority = createPriority();
    ...
    try {
      rpcServer = new RpcServer(rs, name, getServices(),
          bindAddress, // use final bindAddress for this server.
          rs.conf,
          //创建regionServer的调度器
          rpcSchedulerFactory.create(rs.conf, this, rs));
      rpcServer.setRsRpcServices(this);
    } catch (BindException be) {
      String configName = (this instanceof MasterRpcServices) ? HConstants.MASTER_PORT :
          HConstants.REGIONSERVER_PORT;
      throw new IOException(be.getMessage() + ". To switch ports use the '" + configName +
          "' configuration property.", be.getCause() != null ? be.getCause() : be);
    }
}


  protected PriorityFunction createPriority() {
    return new AnnotationReadingPriorityFunction(this);
  }

```

AnnotationReadingPriorityFunction
```
  public AnnotationReadingPriorityFunction(final RSRpcServices rpcServices,
      Class<? extends RSRpcServices> clz) {
    Map<String,Integer> qosMap = new HashMap<String,Integer>();
    //遍历并加载所有带 QosPriority 注解的方法
    for (Method m : clz.getMethods()) {
      QosPriority p = m.getAnnotation(QosPriority.class);
      if (p != null) {
        // Since we protobuf'd, and then subsequently, when we went with pb style, method names
        // are capitalized.  This meant that this brittle compare of method names gotten by
        // reflection no longer matched the method names coming in over pb.  TODO: Get rid of this
        // check.  For now, workaround is to capitalize the names we got from reflection so they
        // have chance of matching the pb ones.
        String capitalizedMethodName = capitalize(m.getName());
        qosMap.put(capitalizedMethodName, p.priority());
      }
    }
    this.rpcServices = rpcServices;
    this.annotatedQos = qosMap;
    if (methodMap.get("getRegion") == null) {
      methodMap.put("hasRegion", new HashMap<Class<? extends Message>, Method>());
      methodMap.put("getRegion", new HashMap<Class<? extends Message>, Method>());
    }
    for (Class<? extends Message> cls : knownArgumentClasses) {
      argumentToClassMap.put(cls.getName(), cls);
      try {
        methodMap.get("hasRegion").put(cls, cls.getDeclaredMethod("hasRegion"));
        methodMap.get("getRegion").put(cls, cls.getDeclaredMethod("getRegion"));
      } catch (Exception e) {
        throw new RuntimeException(e);
      }
    }

    Configuration conf = rpcServices.getConfiguration();
    scanVirtualTimeWeight = conf.getFloat(SCAN_VTIME_WEIGHT_CONF_KEY, 1.0f);
  }
```

SimpleRpcSchedulerFactory

```
  public RpcScheduler create(Configuration conf, PriorityFunction priority, Abortable server) {
    //hbase.regionserver.handler.count
    //等待响应用户表级请求的线程数
    //默认值 30
    int handlerCount = conf.getInt(HConstants.REGION_SERVER_HANDLER_COUNT,
        HConstants.DEFAULT_REGION_SERVER_HANDLER_COUNT);
    return new SimpleRpcScheduler(
      conf,
      handlerCount,
      //hbase.regionserver.metahandler.count
      //处理meta表请求的线程数
      //默认值 20
      conf.getInt(HConstants.REGION_SERVER_HIGH_PRIORITY_HANDLER_COUNT,
        HConstants.DEFAULT_REGION_SERVER_HIGH_PRIORITY_HANDLER_COUNT),
      //hbase.regionserver.replication.handler.count
      //拷贝队列的大小
      //默认值 3
      conf.getInt(HConstants.REGION_SERVER_REPLICATION_HANDLER_COUNT,
          HConstants.DEFAULT_REGION_SERVER_REPLICATION_HANDLER_COUNT),
      priority,
      server,
      HConstants.QOS_THRESHOLD);
  }
```

RpcServer
```
public RpcServer(final Server server, final String name,
      final List<BlockingServiceAndInterface> services,
      final InetSocketAddress bindAddress, Configuration conf,
      RpcScheduler scheduler){
      this.scheduler = scheduler;      
}
 protected void processRequest(ByteBuffer buf) throws IOException, InterruptedException {
    ...
    if (!scheduler.dispatch(new CallRunner(RpcServer.this, call))) {
            callQueueSize.add(-1 * call.getSize());
    
            ByteArrayOutputStream responseBuffer = new ByteArrayOutputStream();
            metrics.exception(CALL_QUEUE_TOO_BIG_EXCEPTION);
            setupResponse(responseBuffer, call, CALL_QUEUE_TOO_BIG_EXCEPTION,
                "Call queue is full on " + server.getServerName() +
                    ", too many items queued ?");
            responder.doRespond(call);
          }
 }
```

SimpleRpcScheduler
```
 public SimpleRpcScheduler(
      Configuration conf,
      int handlerCount,
      int priorityHandlerCount,
      int replicationHandlerCount,
      PriorityFunction priority,
      Abortable server,
      int highPriorityLevel){
      ...
      //默认FIFO队列
          String callQueueType = conf.get(RpcExecutor.CALL_QUEUE_TYPE_CONF_KEY,
            RpcExecutor.CALL_QUEUE_TYPE_CONF_DEFAULT);
          float callqReadShare = conf.getFloat(RWQueueRpcExecutor.CALL_QUEUE_READ_SHARE_CONF_KEY, 0);
      
          if (callqReadShare > 0) {
            // at least 1 read handler and 1 write handler
            callExecutor = new RWQueueRpcExecutor("default.RWQ", Math.max(2, handlerCount),
              maxQueueLength, priority, conf, server);
          } else {
          //callQeueType有fifo,deadline,codel三种
          //fifo和codel使用 FastPathBalancedQueueRpcExecutor
          //deadline使用 BalancedQueueRpcExecutor
            if (RpcExecutor.isFifoQueueType(callQueueType) || RpcExecutor.isCodelQueueType(callQueueType)) {
              callExecutor = new FastPathBalancedQueueRpcExecutor("default.FPBQ", handlerCount,
                  maxQueueLength, priority, conf, server);
            } else {
              callExecutor = new BalancedQueueRpcExecutor("default.BQ", handlerCount, maxQueueLength,
                  priority, conf, server);
            }
          }
      
          // Create 2 queues to help priorityExecutor be more scalable.
          this.priorityExecutor = priorityHandlerCount > 0 ? new FastPathBalancedQueueRpcExecutor(
              "priority.FPBQ", priorityHandlerCount, RpcExecutor.CALL_QUEUE_TYPE_FIFO_CONF_VALUE,
              maxPriorityQueueLength, priority, conf, abortable) : null;
          this.replicationExecutor = replicationHandlerCount > 0 ? new FastPathBalancedQueueRpcExecutor(
              "replication.FPBQ", replicationHandlerCount, RpcExecutor.CALL_QUEUE_TYPE_FIFO_CONF_VALUE,
              maxQueueLength, priority, conf, abortable) : null;
      
}
 public boolean dispatch(CallRunner callTask) throws InterruptedException {
    RpcServer.Call call = callTask.getCall();
    int level = priority.getPriority(call.getHeader(), call.param, call.getRequestUser());
    if (priorityExecutor != null && level > highPriorityLevel) {
      return priorityExecutor.dispatch(callTask);
    } else if (replicationExecutor != null && level == HConstants.REPLICATION_QOS) {
      return replicationExecutor.dispatch(callTask);
    } else {
      return callExecutor.dispatch(callTask);
    }
  }
```

RpcExecutor
```
public RpcExecutor(final String name, final int handlerCount, final String callQueueType,
      final int maxQueueLength, final PriorityFunction priority, final Configuration conf,
      final Abortable abortable) {
    if (isDeadlineQueueType(callQueueType)) {
      this.name += ".Deadline";
      this.queueInitArgs = new Object[] { maxQueueLength,
        new CallPriorityComparator(conf, this.priority) };
      this.queueClass = BoundedPriorityBlockingQueue.class;
    } else if (isCodelQueueType(callQueueType)) {
      this.name += ".Codel";
      int codelTargetDelay = conf.getInt(CALL_QUEUE_CODEL_TARGET_DELAY,
        CALL_QUEUE_CODEL_DEFAULT_TARGET_DELAY);
      int codelInterval = conf.getInt(CALL_QUEUE_CODEL_INTERVAL, CALL_QUEUE_CODEL_DEFAULT_INTERVAL);
      double codelLifoThreshold = conf.getDouble(CALL_QUEUE_CODEL_LIFO_THRESHOLD,
        CALL_QUEUE_CODEL_DEFAULT_LIFO_THRESHOLD);
      queueInitArgs = new Object[] { maxQueueLength, codelTargetDelay, codelInterval,
          codelLifoThreshold, numGeneralCallsDropped, numLifoModeSwitches };
      queueClass = AdaptiveLifoCoDelCallQueue.class;
    } else {
      this.name += ".Fifo";
      queueInitArgs = new Object[] { maxQueueLength };
      queueClass = LinkedBlockingQueue.class;
    }

    LOG.info("RpcExecutor " + " name " + " using " + callQueueType
        + " as call queue; numCallQueues=" + numCallQueues + "; maxQueueLength=" + maxQueueLength
        + "; handlerCount=" + handlerCount);      
}
```

