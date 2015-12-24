#Application启动
略过脚本和SparkSubmit程序,直接看SparkContext类   
__配置:__  
`spark.ui.showConsoleProgress` 可以关闭每次action时终端上显示的stage进度条.默认是`true`
`spark.driver.allowMultipleContexts` 判断是否允许一个JVM启动多个sparkcontext,默认是`false`  
`spark.serializer`=`org.apache.spark.serializer.JavaSerializer`  
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
##SparkContext:
####1. 初始化过程:
1.通过 `SparkEnv.createDriverEnv(conf, isLocal, listenerBus, SparkContext.numDriverCores(master))`设置了sparkEnv,返回了以下构造参数构造的sparkEnv对象 
	
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
	
2.`val (sched, ts) = SparkContext.createTaskScheduler(this, master)`创建TaskScheduler,创建过程是根据master进行match,standalone匹配的是`"""spark://(.*)""".r`,在这个case中:  

	val scheduler = new TaskSchedulerImpl(sc)  
	val masterUrls = sparkUrl.split(",").map("spark://" + _)  
    val backend = new SparkDeploySchedulerBackend(scheduler, sc, masterUrls)
        scheduler.initialize(backend)

然后又创建了`_dagScheduler = new DAGScheduler(this)`,继而启动`_taskScheduler.start()`
然后根据dynamicAllocationEnabled的boolean值true/false来创建_executorAllocationManager 为`Some(ExecutorAllocationManager)`/`None`并启动  

line 561
####2.其他过程
