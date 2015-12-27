#Application启动
略过脚本和SparkSubmit程序,直接看SparkContext类   
###配置: 

`spark.ui.showConsoleProgress`=`true` 可以关闭每次action时终端上显示的stage进度条
`spark.driver.allowMultipleContexts`=`false` 判断是否允许一个JVM启动多个sparkcontext`spark.serializer`=`org.apache.spark.serializer.JavaSerializer`  
`spark.closure.serializer`=`org.apache.spark.serializer.JavaSerializer`  
`spark.shuffle.manager`=`sort`  :

     "hash" -> "org.apache.spark.shuffle.hash.HashShuffleManager",
     "sort" -> "org.apache.spark.shuffle.sort.SortShuffleManager",
     "tungsten-sort" -> "org.apache.spark.shuffle.unsafe.UnsafeShuffleManager")
`spark.shuffle.blockTransferService`=`netty`	:  

	netty=NettyBlockTransferService  
	nio=NioBlockTransferService  : will be remove in spark 1.6.0
`spark.fileserver.port`=`0`  
`spark.unsafe.offHeap`=`false` #true=MemoryAllocator.UNSAFE || false =MemoryAllocator.HEAP  
`spark.ui.enabled`=`true`  
`spark.executor.memory>SPARK_EXECUTOR_MEMORY>SPARK_MEM`=`1024`  
`spark.dynamicAllocation.enabled`=`false` && `spark.executor.instances`=`0` == `0`  #dynamicAllocationEnabled  根据这两个参数来判断是否启动动态分配策略,默认是关闭的  
`spark.cleaner.referenceTracking`=`true`  
`spark.scheduler.mode`=`FIFO`  
`spark.speculation`=`false`
##SparkContext:
####1. 实例化过程:
只记录了自认为的重点,有很多遗漏 
1.无条件创建了`private val creationSite: CallSite = Utils.getCallSite()` # __*没搞懂callsite是干什么用的,待解决!!!*__  
2.通过配置`spark.driver.allowMultipleContexts`=`true`允许在一个jvm启动多个sparkcontext而不抛出异常,仅仅是报出warn日志,但是正常情况下一个jvm里只允许一个sparkconstext存在的,可以看一下注释:  
	
	Called at the beginning of the SparkContext constructor to ensure that no SparkContext is running.  Throws an exception if a running context is detected and logs a warning if another thread is constructing a SparkContext.  This warning is necessary because the current locking scheme prevents us from reliably distinguishing between cases where another context is being constructed and cases where another constructor threw an exception.  
3.参数sparkConfig被深度复制到私有变量的_conf中 
4.无条件创建`listenerBus=new LiveListenerBus`   
5.无条件创建JobProgressListener
	
	jobProgressListener = new JobProgressListener(_conf)
	listenerBus.addListener(jobProgressListener)
6.通过 `SparkEnv.createDriverEnv(conf, isLocal, listenerBus, SparkContext.numDriverCores(master))`设置了sparkEnv,返回了以下构造参数构造的sparkEnv对象: 
	
	1. executorId="driver",
    2. rpcEnv,
    3. serializer,
    4. closureSerializer,
    5. cacheManager = new CacheManager,
    6. mapOutputTracker = new MapOutputTrackerMaster(conf),
    7. shuffleManager : org.apache.spark.shuffle.sort.SortShuffleManager,
    8. broadcastManager = new BroadcastManager,
    9. blockTransferService =  new NettyBlockTransferService(conf, securityManager, numUsableCores),//numUsableCores=0 driver is not used for execution
    10. blockManager = new BlockManager,
    11. securityManager = new SecurityManager(conf),
    12. httpFileServer = new HttpFileServer(conf, securityManager, fileServerPort),
    13. sparkFilesDir = Utils.createTempDir(Utils.getLocalDir(conf), "userFiles").getAbsolutePath,
    14. metricsSystem = MetricsSystem.createMetricsSystem("driver", conf, securityManager),
    15. shuffleMemoryManager = ShuffleMemoryManager.create(conf, numUsableCores),
    16. executorMemoryManager = new ExecutorMemoryManager(allocator), #allocator=MemoryAllocator.UNSAFE/MemoryAllocator.HEAP 见spark.unsafe.offHeap
    17. outputCommitCoordinator=new OutputCommitCoordinator(conf, isDriver),
    18. conf = SparkConf
7.`_metadataCleaner = new MetadataCleaner(MetadataCleanerType.SPARK_CONTEXT, this.cleanup, _conf)`  

8.`_statusTracker = new SparkStatusTracker(this)`  

9.`_hadoopConfiguration = SparkHadoopUtil.get.newConfiguration(_conf)` 
 
10.`_heartbeatReceiver = env.rpcEnv.setupEndpoint(
      HeartbeatReceiver.ENDPOINT_NAME, new HeartbeatReceiver(this))`  
      
11.`val (sched, ts) = SparkContext.createTaskScheduler(this, master)`创建TaskScheduler,创建过程是根据master进行match,standalone匹配的是`"""spark://(.*)""".r`,在这个case中:  

	val scheduler = new TaskSchedulerImpl(sc)  
	val masterUrls = sparkUrl.split(",").map("spark://" + _)  
    val backend = new SparkDeploySchedulerBackend(scheduler, sc, masterUrls)
        scheduler.initialize(backend)

12.创建了`_dagScheduler = new DAGScheduler(this)`,继而启动`_taskScheduler.start()`  
13.根据dynamicAllocationEnabled的boolean值true/false来创建_executorAllocationManager 为`Some(new ExecutorAllocationManager(this, listenerBus, _conf))`/`None`并启动  
根据`spark.cleaner.referenceTracking`booleantrue/false来创建_cleaner为`Some(new ContextCleaner(this))`/`None`并启动  
14.

	setupAndStartListenerBus()  #设置listenerbus的listener,并启动listenerBus
    postEnvironmentUpdate()  #通知各个listener更新环境信息
    postApplicationStart()  #通知各个listener系统启动
    _taskScheduler.postStartHook() #检查scheduleBackEnd是否启动成功,如果没有成功,等待100毫秒或者被通知
以上是sparkcontext启动主函数的概要分析,下面主要分析这中间一些重要的模块
######1. TaskSchedulerImpl 实例化
除了使用yarn-client和yarn-cluster以外,在1.5.0版本中其他模式使用的tasksheduler使用的都是TaskSchedulerImpl. *创建方式见`上11`*  
`taskResultGetter = new TaskResultGetter(sc.env, this)`  
######2. SparkDeploySchedulerBackend 实例化
在spark的各种部署模式中只有standalone模式以及local[N, maxRetries]模式的shedulerbackend采用SparkDeploySchedulerBackend类,该类继承了CoarseGrainedSchedulerBackend以及AppClientListener,这里主要讲CoarseGrainedSchedulerBackend,实例化过程中只是读取一些配置和声明一些变量.
######3.DAGScheduler 实例化  
简单粗暴,直接`new DAGScheduler(this)`,这中间做了以下事情:  
`metricsSource: DAGSchedulerSource = new DAGSchedulerSource(this)`  
`val outputCommitCoordinator = env.outputCommitCoordinator`
`taskScheduler.setDAGScheduler(this)` taskScheduler和dagScheduler进行了关联,dagscheduler的实例化是在taskscheduler后面,下面会讲到

####2. TaskSchedulerImpl,SparkDeploySchedulerBackend,DAGScheduler  
#####1. 实例化顺序:
抽离其他代码,合并后如下:
	
	val scheduler = new TaskSchedulerImpl(sc)
    val masterUrls = sparkUrl.split(",").map("spark://" + _)
    val backend = new SparkDeploySchedulerBackend(scheduler, sc, masterUrls)
    scheduler.initialize(backend)
    _dagScheduler = new DAGScheduler(this)
    _taskScheduler.start()
#####2. 内部代码解读:
1.`scheduler.initialize(backend)`,其中:  
taskscheduler和schedulerbackend建立了关联,根据调度模式初始花了一个`rootPool=Pool("", schedulingMode, 0, 0)`,并根据调度模式建立了schedulableBuilder:
	
	schedulableBuilder = {
      schedulingMode match {
        case SchedulingMode.FIFO =>
          new FIFOSchedulableBuilder(rootPool)
        case SchedulingMode.FAIR =>
          new FairSchedulableBuilder(rootPool, conf)
      }
    }
    schedulableBuilder.buildPools()
2.`_taskScheduler.start()`,其中:  
首先启动了backend:`backend.start()`
又根据参数启动了一个定时线程,推测是否有任务需要调度
	
	if (!isLocal && conf.getBoolean("spark.speculation", false)) {
      logInfo("Starting speculative execution thread")
      speculationScheduler.scheduleAtFixedRate(new Runnable {
        override def run(): Unit = Utils.tryOrStopSparkContext(sc) {
          checkSpeculatableTasks()
        }
      }, SPECULATION_INTERVAL_MS, SPECULATION_INTERVAL_MS, TimeUnit.MILLISECONDS)
    }
    
     // Check for speculatable tasks in all our active jobs.
  	def checkSpeculatableTasks() {
      var shouldRevive = false
      synchronized {
        shouldRevive = rootPool.checkSpeculatableTasks()
      }
      if (shouldRevive) {
        backend.reviveOffers()
      }
    }
3.`backend.start()`
SparkDeploySchedulerBackend.start()首先调用的是CoarseGrainedSchedulerBackend.start():`super.start()`  
1).

####3. 
